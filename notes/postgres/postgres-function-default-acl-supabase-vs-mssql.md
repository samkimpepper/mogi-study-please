---
tags:
  - backend
  - postgres
  - supabase
  - mssql
  - security
  - database
review_answered: false
---

# PostgreSQL 함수 default ACL과 Supabase 권한 함정

## 🎯 왜 공부했나

사이드프로젝트에서 PostgreSQL 함수를 새로 만들 때마다 `anon` 역할에 실행 권한이 생길 수 있다는 사실을 발견했다.

기존 함수에서 권한을 회수해도 새 함수를 만들면 같은 문제가 반복될 수 있었고, 이를 막으려면 개별 함수의 `REVOKE`뿐만 아니라 `ALTER DEFAULT PRIVILEGES`까지 설정해야 했다.

MSSQL을 사용할 때는 이런 문제를 크게 느끼지 못했기 때문에 다음 의문이 생겼다.

> 이런 권한 함정은 PostgreSQL에만 있는 것인가? 왜 Supabase에서는 함수나 테이블을 추가할 때마다 권한 문제가 계속 나타나는가?

---

## 🧠 먼저 결론

> 권한 문제는 모든 DB에 있지만, **새 함수에 `PUBLIC EXECUTE`가 기본으로 주어지는 이번 함정은 PostgreSQL의 기본 권한 특성**이다. 그리고 Supabase에서는 PostgreSQL 함수가 외부 RPC가 될 수 있어서, DB 내부의 기본 권한이 인터넷 API의 공격 표면으로 이어진다.

MSSQL에도 `public` 역할, ownership chaining, `EXECUTE AS` 같은 권한 함정이 있다. 다만 전통적인 Spring + MSSQL 구조에서는 DB가 백엔드 서버 뒤에 숨고 모든 앱 사용자가 하나의 DB 서비스 계정으로 접근하는 경우가 많아서, 애플리케이션 개발자가 사용자별 DB 권한 문제를 직접 마주칠 일이 상대적으로 적었다.

---

## 🆚 PostgreSQL과 MSSQL의 함수 기본 실행 권한 차이

이번 문제에서는 두 DB의 기본값이 실제로 다르다.

### PostgreSQL

PostgreSQL은 새 함수와 프로시저를 만들 때 기본적으로 `PUBLIC`에 `EXECUTE` 권한을 부여한다.

```text
새 PostgreSQL 함수 생성
→ PUBLIC EXECUTE 기본 부여
→ 모든 DB 역할이 PUBLIC을 통해 실행할 수 있음
```

PostgreSQL 공식 문서의 기본 권한 표에서도 함수와 프로시저의 기본 `PUBLIC` 권한을 `EXECUTE`로 설명한다.

- [PostgreSQL Privileges](https://www.postgresql.org/docs/current/ddl-priv.html)

### MSSQL

SQL Server에서는 사용자 정의 모듈의 `EXECUTE` 권한이 기본적으로 모듈 소유자에게 있으며, 다른 사용자나 역할이 실행하려면 별도의 권한 부여가 필요하다.

```text
새 MSSQL 프로시저 생성
→ 소유자가 실행 가능
→ 다른 사용자에게는 명시적인 GRANT EXECUTE 필요
```

```sql
GRANT EXECUTE
ON OBJECT::HumanResources.uspUpdateEmployeeHireInfo
TO RecruitingRole;
```

- [Microsoft EXECUTE 문서](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/execute-transact-sql)
- [Microsoft GRANT Object Permissions 문서](https://learn.microsoft.com/en-us/sql/t-sql/statements/grant-object-permissions-transact-sql)

따라서 MSSQL을 사용할 때 "프로시저를 만들 때마다 모든 사용자가 실행할 수 있는지 확인했던 기억이 없다"는 감각은 틀리지 않았다. 적어도 이번에 발견한 **신규 함수의 기본 `PUBLIC EXECUTE`** 문제에서는 PostgreSQL과 MSSQL의 기본값이 다르다.

---

## 🌍 PUBLIC과 public 스키마는 다르다

PostgreSQL에서 다음 두 단어는 철자가 같아 보여도 완전히 다른 개념이다.

| 이름 | 정체 | 의미 |
| --- | --- | --- |
| `PUBLIC` | 특수한 의사 역할 | 현재와 미래의 모든 DB 역할을 포함 |
| `public` | 스키마 | 테이블과 함수 같은 DB 객체를 담는 namespace |

다음 SQL에서 첫 번째 `public`은 스키마이고, 두 번째 `PUBLIC`은 모든 역할을 뜻한다.

```sql
REVOKE EXECUTE
ON FUNCTION public.delete_everything()
FROM PUBLIC;
```

```text
public.delete_everything()
└─ public 스키마 안에 있는 함수

FROM PUBLIC
└─ 모든 DB 역할로부터 권한 회수
```

`anon`과 `authenticated`도 `PUBLIC`에 포함되므로, 함수가 `PUBLIC EXECUTE`를 가지고 있으면 이 역할들도 그 경로를 통해 실행 권한을 얻을 수 있다.

> `PUBLIC`은 명시적으로 가입시키는 일반 role이 아니다. PostgreSQL의 모든 역할에 공통으로 적용되는 권한 대상을 뜻한다.

---

## 🔥 PostgreSQL의 기본값이 Supabase에서 더 위험하게 느껴지는 이유

PostgreSQL만 놓고 보면 `PUBLIC EXECUTE`는 **DB에 접속한 역할들이 함수를 실행할 수 있다**는 뜻이다. 이것만으로 인터넷 사용자가 DB에 바로 연결되는 것은 아니다.

전통적인 DB 환경에서는 먼저 DB 접속 자체가 통제된다.

```text
허가받은 서버 또는 사용자
→ DB 연결 성공
→ DB 내부 함수 호출
```

그런데 Supabase는 PostgreSQL 앞에 Data API를 제공하고 JWT의 역할을 실제 PostgreSQL 역할과 연결한다.

```text
외부 HTTP 요청
→ Supabase Data API / PostgREST
→ JWT에 따라 anon 또는 authenticated 역할
→ public 스키마의 PostgreSQL 함수 호출
```

따라서 `public`처럼 Data API에 노출된 스키마에 함수가 있고 `anon`이 그 함수를 실행할 수 있다면, DB 내부 함수가 외부 RPC 호출 표면으로 연결될 수 있다.

```text
public 스키마에 함수 생성
+ anon EXECUTE
+ Data API 노출
≈ 익명 사용자가 호출할 수 있는 RPC 후보
```

Supabase 공식 문서도 기존 프로젝트에서 새 함수의 `EXECUTE` 권한이 Data API 역할에 자동 부여될 수 있으며, 원하지 않는 객체가 API를 통해 도달 가능해질 수 있다고 설명한다.

- [Supabase Securing your API](https://supabase.com/docs/guides/api/securing-your-api)
- [Supabase Database Functions](https://supabase.com/docs/guides/database/functions)

> PostgreSQL의 DB 내부 기본 권한과 Supabase의 DB 직접 API 구조가 만나는 지점에서 위험이 커진다.

---

## 📋 ACL과 default ACL은 무엇인가

### ACL

ACL은 Access Control List의 약자다. 특정 DB 객체에 어떤 역할이 어떤 권한을 가지고 있는지 기록한 목록이다.

예를 들면 함수 하나의 ACL에는 다음과 같은 정보가 들어갈 수 있다.

```text
함수: public.replace_shade_color_families(...)

postgres       → EXECUTE
authenticated  → EXECUTE
anon           → EXECUTE
PUBLIC         → EXECUTE
```

개별 함수에 실행하는 `GRANT`와 `REVOKE`는 이 함수의 현재 ACL을 바꾼다.

```sql
REVOKE EXECUTE
ON FUNCTION public.replace_shade_color_families(bigint, bigint[])
FROM PUBLIC, anon;
```

### default ACL

Default ACL은 **앞으로 생성될 객체가 처음 받을 권한의 템플릿**이다.

```text
현재 함수의 ACL
→ 이미 존재하는 그 함수의 권한

default ACL
→ 앞으로 새로 생성될 함수의 초기 권한
```

기존 함수만 다음처럼 막았다고 하자.

```sql
REVOKE EXECUTE
ON FUNCTION public.dangerous_operation()
FROM PUBLIC, anon;
```

이 조치는 현재의 `dangerous_operation()`만 닫는다. 이후 새로운 함수를 만들면 기본 권한 템플릿이 다시 적용된다.

```text
기존 함수 REVOKE
→ 현재 함수는 닫힘

다음 마이그레이션에서 새 함수 생성
→ default ACL 적용
→ PUBLIC 또는 anon EXECUTE가 다시 생길 수 있음
```

이 재발을 막는 장치가 `ALTER DEFAULT PRIVILEGES`다.

---

## 🧱 기존 함수와 앞으로 생길 함수는 따로 막아야 한다

### 이미 존재하는 함수

```sql
REVOKE EXECUTE
ON ALL FUNCTIONS IN SCHEMA public
FROM PUBLIC, anon, authenticated;
```

이 SQL은 현재 `public` 스키마에 존재하는 함수의 실행 권한을 회수한다.

하지만 앞으로 새로 생기는 함수의 기본 권한까지 바꾸지는 않는다.

### 앞으로 생성될 함수

Supabase 마이그레이션에서 함수를 생성하는 역할이 `postgres`라면 다음과 같이 미래 기본값을 닫을 수 있다.

PostgreSQL에 내장된 함수의 전역 `PUBLIC EXECUTE` 기본값을 제거할 때는 `IN SCHEMA`를 붙이면 안 된다. 스키마별 default privilege는 전역 기본값에 권한을 추가하는 구조이므로, 전역에서 부여된 권한을 스키마 범위의 `REVOKE`로 뺄 수 없기 때문이다.

```sql
-- PostgreSQL 전역 기본값인 PUBLIC EXECUTE 제거
ALTER DEFAULT PRIVILEGES
FOR ROLE postgres
REVOKE EXECUTE ON FUNCTIONS
FROM PUBLIC;

-- legacy Supabase가 public 스키마에서 API 역할에 직접 부여한 권한 제거
ALTER DEFAULT PRIVILEGES
FOR ROLE postgres
IN SCHEMA public
REVOKE EXECUTE ON FUNCTIONS
FROM anon, authenticated, service_role;
```

그 뒤 필요한 RPC만 명시적으로 다시 연다.

```sql
GRANT EXECUTE
ON FUNCTION public.get_my_profile()
TO authenticated;
```

원하는 정책은 다음과 같다.

```text
기본값은 실행 불가
→ 필요한 역할과 함수 조합만 검토
→ 명시적인 GRANT로 허용
```

> [!warning]
> 기존 함수 전체의 실행 권한을 한꺼번에 회수하면 현재 앱이 사용하는 RPC도 막힐 수 있다. 실제 적용할 때는 필요한 함수의 allowlist와 검증을 같은 마이그레이션에 포함해야 한다.

---

## 🕳️ PUBLIC만 막거나 anon만 막으면 생기는 구멍

Supabase 프로젝트의 실제 default ACL 상태에 따라 함수 실행 권한이 두 경로로 올 수 있다.

```text
경로 1: PostgreSQL의 PUBLIC EXECUTE
경로 2: Supabase default ACL의 anon 직접 EXECUTE
```

### anon만 REVOKE한 경우

```sql
REVOKE EXECUTE
ON FUNCTION public.foo()
FROM anon;
```

함수에 `PUBLIC EXECUTE`가 남아 있다면 `anon`은 모든 역할에 적용되는 `PUBLIC` 경로를 통해 여전히 실행할 수 있다.

```text
anon 직접 권한 제거
하지만 PUBLIC EXECUTE 유지
→ anon도 PUBLIC에 포함
→ 실행 가능
```

### PUBLIC만 REVOKE한 경우

```sql
REVOKE EXECUTE
ON FUNCTION public.foo()
FROM PUBLIC;
```

프로젝트의 default ACL이 `anon` 역할에 직접 `EXECUTE`를 부여했다면 그 직접 권한은 남을 수 있다.

```text
PUBLIC 경로 제거
하지만 anon 직접 GRANT 유지
→ 실행 가능
```

따라서 현재 권한을 확인할 때는 `PUBLIC` 경로와 `anon`, `authenticated`, `service_role`에 대한 직접 권한을 함께 봐야 한다.

Supabase 문서에서도 함수 실행을 제한할 때 `PUBLIC`과 제한할 개별 API 역할의 권한을 모두 확인하도록 안내한다.

---

## 👤 default privilege는 객체 생성 역할별 설정이다

Default ACL의 가장 놓치기 쉬운 특징은 **객체를 만드는 역할마다 따로 적용된다**는 점이다.

다음 설정은 `postgres`가 앞으로 만드는 함수에만 적용된다. `IN SCHEMA`가 없으므로 현재 데이터베이스에서 `postgres`가 만드는 함수의 전역 기본 `PUBLIC EXECUTE`를 제거한다.

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE postgres
REVOKE EXECUTE ON FUNCTIONS
FROM PUBLIC;
```

그런데 실제 함수를 다른 역할이 만들었다고 하자.

```text
default ACL 설정 대상: postgres
실제 함수 생성 역할:   migration_admin
```

그러면 `postgres`용 default ACL은 `migration_admin`이 만든 새 함수에 적용되지 않는다.

```text
함수 생성 역할이 다름
→ 다른 역할의 default ACL 사용
→ 닫았다고 생각한 기본 권한이 다시 등장할 수 있음
```

따라서 다음 질문을 함께 확인해야 한다.

- 마이그레이션은 어떤 DB 역할로 실행되는가?
- 새 함수의 owner는 누구인가?
- `ALTER DEFAULT PRIVILEGES FOR ROLE ...`의 대상과 실제 생성 역할이 같은가?
- 설정이 필요한 스키마에 적용됐는가?
- 기존 객체의 권한도 별도로 정리했는가?

> default ACL은 데이터베이스 전체에 무조건 적용되는 전역 보안 스위치가 아니다. "어떤 역할이, 어떤 스키마에, 앞으로 만들 어떤 종류의 객체인가"에 따라 적용 범위가 나뉜다.

---

## 🛡️ 함수가 호출 가능하다고 곧바로 모든 데이터가 뚫리는 것은 아니다

함수의 `EXECUTE` 권한은 함수에 들어갈 수 있는 첫 번째 문이다. 함수가 내부에서 테이블을 읽거나 수정할 때의 권한은 함수의 실행 모드에 따라 달라진다.

### SECURITY INVOKER

PostgreSQL 함수의 기본 실행 방식이다. 호출자의 권한으로 함수 본문을 실행한다.

```text
anon이 함수 호출
→ anon 권한으로 함수 실행
→ 내부 테이블 권한 검사
→ RLS 정책 검사
```

따라서 `anon`이 함수를 호출할 수 있다는 사실만으로 그 함수가 접근하는 모든 데이터가 자동으로 공개되는 것은 아니다.

하지만 다음 문제는 여전히 남는다.

- 외부에 노출할 의도가 없었던 비즈니스 기능이 호출 가능하다.
- 비용이 큰 작업이나 상태 변경 함수를 반복 호출할 수 있다.
- 함수 내부에 별도 인가 검사가 없다면 허용 범위가 예상보다 넓을 수 있다.
- 앞으로 함수 구현이 바뀌면서 위험해질 수 있다.

### SECURITY DEFINER

`SECURITY DEFINER` 함수는 호출자가 아니라 함수 소유자의 권한으로 실행된다.

```text
anon이 함수 호출
→ 함수 owner 권한으로 실행
→ anon이 직접 할 수 없는 작업까지 가능할 수 있음
```

특히 다음 조합은 고위험이다.

```text
SECURITY DEFINER
+ PUBLIC 또는 anon EXECUTE
+ 함수 내부 인증·인가 검사 부족
+ 과도한 owner 권한
+ 안전하지 않은 search_path
```

Supabase는 `SECURITY DEFINER`를 사용할 때 `search_path`를 제한하고 본문의 객체를 스키마까지 명시하도록 안내한다.

```sql
CREATE FUNCTION public.admin_operation()
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = ''
AS $$
BEGIN
  -- public.table_name처럼 스키마를 명시한다.
END;
$$;
```

`EXECUTE` 권한, 함수 실행 주체, 내부 테이블 권한과 RLS는 같은 것이 아니다.

```text
함수에 들어갈 수 있는가?       → EXECUTE
누구의 권한으로 실행되는가?    → INVOKER / DEFINER
어떤 행을 읽거나 쓸 수 있는가? → table grant + RLS
```

---

## 🌱 PostgreSQL이 왜 이런 기본값을 가지고 있을까

PostgreSQL 공식 권한 문서는 함수가 `PUBLIC EXECUTE`를 받는다는 사실과, 최대 보안이 필요하면 함수 생성과 `REVOKE`를 같은 트랜잭션 안에서 수행하라고 설명한다. 다만 왜 이런 역사적 기본값을 선택했는지를 길게 설명하지는 않는다.

구조적으로 보면 PostgreSQL 함수는 단순한 외부 RPC만을 의미하지 않는다.

- 일반 SQL 쿼리의 함수 호출
- 연산자 구현
- 타입 변환
- 표현식과 인덱스
- 트리거 함수
- DB 내부 유틸리티

전통적인 PostgreSQL 사용 환경에서는 먼저 DB 연결 자체가 통제되고, 연결이 허용된 여러 역할이 DB 함수를 널리 사용하는 상황이 자연스러웠다. 이런 환경에서는 함수의 기본 실행 가능성이 사용성 중심의 선택으로 작동할 수 있다.

하지만 Supabase는 데이터베이스 역할을 외부 HTTP 요청과 직접 연결한다.

```text
전통적 전제
DB에 연결된 역할들 사이의 함수 사용

Supabase에서 생긴 효과
인터넷 요청을 대표하는 anon 역할도 DB 역할
→ 기존 기본값이 외부 API 보안 문제로 확대
```

따라서 이것을 단순히 "PostgreSQL이 아무 보안 없이 인터넷에 함수를 공개한다"고 이해하면 정확하지 않다. 외부 API 연결은 Supabase가 제공하고, 그 경로에서 PostgreSQL의 기본 함수 권한이 예상치 못한 효과를 내는 것이다.

---

## ☕ Spring + MSSQL에서는 왜 덜 느꼈나

전통적인 Spring 백엔드 구조에서는 브라우저가 DB에 직접 연결하지 않는다.

```text
브라우저
→ Spring Controller
→ Spring Security
→ Service
→ Repository
→ 애플리케이션 DB 계정
→ MSSQL
```

실제 앱 사용자는 MSSQL의 개별 DB user가 아닌 경우가 많다.

```text
회원 A ─┐
회원 B ─┼─→ backend_app이라는 하나의 DB 계정
회원 C ─┘
```

따라서 사용자별 인증과 인가는 주로 애플리케이션 계층에서 처리한다.

```java
@PreAuthorize("hasRole('ADMIN')")
```

DB에서는 다음처럼 서비스 종류별 권한만 관리했을 수 있다.

```text
backend_app 계정 → 앱에 필요한 테이블과 프로시저
batch 계정       → 배치 작업 권한
개발자 계정      → 읽기 또는 제한된 쓰기
```

그래서 SQL 객체를 하나 만들 때마다 `anon`과 `authenticated`의 접근 가능성을 생각할 필요가 없었다.

반면 Supabase에서는 Spring이 담당하던 여러 경계가 DB 쪽으로 이동한다.

| Spring 중심 구조 | Supabase 중심 구조 |
| --- | --- |
| Controller endpoint | PostgreSQL 함수와 Data API |
| Spring Security 사용자 | `anon`, `authenticated` DB 역할 |
| 메서드 인가 | 함수 `EXECUTE` 권한과 내부 검사 |
| 서비스 계층 데이터 필터 | RLS policy |
| 관리자 서비스 코드 | `SECURITY DEFINER` 함수 |
| 애플리케이션 배포 설정 | migration의 GRANT, REVOKE, default ACL |

> Supabase에서는 DB 자체가 API이자 인가 계층의 일부이므로, SQL 함수 하나를 만드는 일도 API 엔드포인트와 권한 표면을 만드는 일이 될 수 있다.

---

## 🪤 MSSQL에도 권한 함정은 있다

MSSQL이 권한 면에서 단순하거나 자동으로 안전한 것은 아니다. 함정이 다른 위치에 있고, 애플리케이션 개발자 눈에 덜 보였을 가능성이 크다.

### Ownership chaining

사용자에게 테이블 직접 접근권한은 없지만 프로시저 실행권한은 있다고 하자.

```text
사용자
→ 프로시저 EXECUTE 권한 있음
→ 프로시저와 내부 테이블의 owner가 같음
→ 내부 객체의 권한 검사가 ownership chain에 의해 생략될 수 있음
```

이 기능을 이용하면 프로시저만 안전한 접근 창구로 제공할 수 있다. 반대로 프로시저가 과도한 작업을 제공하면 예상하지 못한 권한 우회가 될 수도 있다.

### EXECUTE AS

SQL Server 모듈도 실행 주체를 바꿀 수 있다.

```sql
CREATE PROCEDURE dbo.admin_operation
WITH EXECUTE AS OWNER
AS
BEGIN
  -- owner 권한으로 실행
END;
```

이는 PostgreSQL의 `SECURITY DEFINER`와 비슷한 위험을 가질 수 있다.

### 그 밖의 MSSQL 권한 함정

- 스키마 전체에 부여한 `GRANT EXECUTE`
- 모든 DB 사용자가 속하는 `public` 데이터베이스 역할
- 동적 SQL에서 ownership chain이 끊기는 문제
- 애플리케이션 계정에 `db_owner`를 부여하는 과도한 권한
- cross-database ownership chaining
- 스키마의 `ALTER` 권한을 통한 예상 밖의 접근

Microsoft도 같은 owner의 객체 연결에서는 ownership chaining 때문에 내부 객체 권한 검사가 생략될 수 있다고 설명하고, 스키마 권한 부여가 권한 상승으로 이어질 수 있음을 경고한다.

- [Microsoft EXECUTE AS 문서](https://learn.microsoft.com/en-us/sql/t-sql/statements/execute-as-clause-transact-sql)
- [Microsoft GRANT Schema Permissions 문서](https://learn.microsoft.com/en-us/sql/t-sql/statements/grant-schema-permissions-transact-sql)

MSSQL에서 권한 문제가 없었던 것이 아니라 다음 중 하나였을 수 있다.

```text
DBA나 인프라 담당자가 미리 설정해서 보이지 않았거나
백엔드 서버 뒤에 숨어 사용자별 DB 권한을 다루지 않았거나
애플리케이션 계정 권한이 너무 넓어서 권한 오류가 발생하지 않았거나
```

권한 오류가 나지 않는 것과 최소 권한으로 안전하게 설계된 것은 다르다.

---

## 😵 Supabase에서 권한 문제가 계속 나타나는 이유

Supabase 요청이 데이터에 도달하기까지 여러 보안 문을 통과한다.

```text
① 이 스키마가 Data API에 노출됐는가?
↓
② anon/authenticated가 스키마를 사용할 수 있는가?
↓
③ 함수 EXECUTE 권한이 있는가?
↓
④ SECURITY INVOKER인가, DEFINER인가?
↓
⑤ 내부 테이블 권한이 있는가?
↓
⑥ RLS 정책상 해당 행에 접근 가능한가?
↓
⑦ 다음에 생길 함수도 default ACL로 같은 권한을 받는가?
```

각 단계는 서로 다른 질문에 답한다.

```text
RLS를 잘 설정했다
≠ 함수 호출 권한도 안전하다

함수에서 anon을 REVOKE했다
≠ PUBLIC 경로도 사라졌다

기존 함수를 전부 막았다
≠ 다음에 만들 함수도 자동으로 막힌다

default ACL을 바꿨다
≠ 다른 owner가 만드는 함수에도 적용된다

SECURITY INVOKER다
≠ 의도하지 않은 함수 노출이 괜찮다
```

보안 장치 하나가 다른 장치를 대신하지 않는다. 이 때문에 한 문제를 고쳤는데 다른 층의 권한 구멍이 다시 발견되는 느낌이 든다.

---

## 😭 그래서 Spring으로 돌아가고 싶은 마음이 드는 이유

익숙했던 구조는 이랬다.

```text
DB는 백엔드 서버 뒤에 숨는다
→ 인증·인가는 Spring Security에서 먼저 처리한다
→ DB는 제한된 서비스 계정만 상대한다
```

Supabase에서는 다음처럼 생각해야 한다.

```text
DB 자체가 외부 API 경계의 일부다
→ SQL 함수 하나도 RPC가 될 수 있다
→ GRANT, RLS, SECURITY DEFINER가 애플리케이션 보안 코드다
```

따라서 Spring에서 Supabase로 이동하면서 단순히 DB 제품만 PostgreSQL로 바뀐 것이 아니다. **보안 경계가 애플리케이션에서 데이터베이스로 크게 이동한 것**이다.

Spring으로 전환하면 사용자별 인증과 인가를 다시 익숙한 Controller와 Service 계층에서 통제할 수 있고, PostgreSQL를 외부 Data API에 직접 노출하지 않을 수 있다.

```text
브라우저
→ Spring API
→ 인증·인가와 비즈니스 검증
→ 제한된 DB 서비스 계정
→ PostgreSQL
```

그러면 `anon` 역할, 함수별 RPC 노출, RLS와 함수 실행권한의 결합을 매 기능마다 직접 다루는 부담은 줄어든다.

하지만 Spring으로 옮긴다고 보안 문제가 사라지는 것은 아니다. 위치가 바뀐다.

- Controller endpoint 인가 누락
- 서비스 메서드의 소유권 검증 누락
- 애플리케이션 DB 계정의 과도한 권한
- 관리자 API의 노출
- SQL injection
- 내부용 기능을 외부 endpoint로 연결하는 실수

차이는 **모기가 더 익숙하고 관찰하기 쉬운 계층에서 문제를 다룰 수 있다**는 점이다.

> Spring은 보안을 없애 주는 것이 아니라, DB 전체 기능을 외부 역할에 직접 연결하는 대신 명시적인 HTTP API 경계 뒤로 옮겨 준다.

---

## ✅ Supabase를 사용하는 동안의 안전한 기본 원칙

### 1. 함수 실행권한은 기본 거부

```text
새 함수는 닫힌 상태
→ 외부 RPC가 필요한 함수만 명시적으로 GRANT
```

### 2. 기존 객체와 미래 객체를 모두 관리

```text
REVOKE ON ALL FUNCTIONS
+ ALTER DEFAULT PRIVILEGES
```

둘 중 하나만 하면 현재 또는 미래에 구멍이 남는다.

### 3. PUBLIC과 API 역할을 함께 확인

```text
PUBLIC
anon
authenticated
service_role
```

권한이 어느 경로에서 오는지 모두 확인한다.

### 4. 함수 생성 owner를 확인

`ALTER DEFAULT PRIVILEGES FOR ROLE ...`의 역할과 실제 마이그레이션 생성 역할이 일치해야 한다.

### 5. SECURITY DEFINER는 고위험 기능으로 취급

- 정말 필요한지 먼저 확인한다.
- 함수 내부에서 인증과 인가를 검증한다.
- owner 권한을 최소화한다.
- `search_path`를 제한한다.
- 모든 객체 이름에 스키마를 명시한다.
- `anon`과 `PUBLIC`의 실행 가능 여부를 검증한다.

### 6. 권한을 추측하지 말고 실제 역할로 테스트

```sql
BEGIN;
SET LOCAL ROLE anon;
SELECT public.target_function();
ROLLBACK;
```

마이그레이션에 원하는 성공과 실패 케이스를 함께 넣으면 새로운 함수가 생길 때 같은 문제가 재발하는 것을 더 빨리 발견할 수 있다.

---

## 📝 최종 정리

### PostgreSQL만의 문제인가

모든 DB에 권한 문제는 있지만, 새 함수에 기본적으로 `PUBLIC EXECUTE`가 생기는 이번 함정은 PostgreSQL의 특성이다. MSSQL 사용자 정의 모듈은 다른 사용자가 실행하려면 일반적으로 명시적인 권한 부여가 필요하다.

### Supabase에서는 왜 더 심각한가

Supabase가 외부 요청의 `anon`, `authenticated` 역할을 PostgreSQL 권한 시스템과 직접 연결하고, 노출된 스키마의 함수를 Data API의 RPC로 제공하기 때문이다.

### MSSQL에서는 왜 잘 못 느꼈나

전통적인 Spring + MSSQL 구조에서는 DB가 백엔드 뒤에 있었고, 실제 앱 사용자는 하나의 DB 서비스 계정으로 간접 접근했다. 사용자별 인가 문제는 주로 Spring에서 처리되었으므로 DB 객체 권한이 기능 개발 과정에 덜 드러났다.

### 왜 권한 문제가 계속 생기는가

Supabase에서는 스키마 노출, 역할 권한, 함수 `EXECUTE`, 함수 실행 주체, 테이블 권한, RLS, default ACL이 서로 다른 보안 층이다. 하나를 고쳐도 다른 층의 구멍이 남을 수 있다.

> [!summary]
> MSSQL에서도 권한 함정은 존재하지만 백엔드와 DBA 뒤에 가려졌을 수 있다. Supabase에서는 PostgreSQL 역할·함수·RLS가 외부 요청을 직접 상대하므로 모든 권한 결정이 애플리케이션 기능 개발 과정에 드러난다. 이번 문제의 핵심은 PostgreSQL의 함수 기본 `PUBLIC EXECUTE`와 Supabase의 Data API 구조가 결합한다는 점이다.

## 복습 질문

- [ ] PostgreSQL의 `PUBLIC`과 `public` 스키마는 무엇이 다르며, 함수 권한에서 이 구분이 왜 중요한가?
- [ ] 기존 함수의 `EXECUTE` 권한을 모두 회수해도 `ALTER DEFAULT PRIVILEGES`가 필요한 이유는 무엇인가?
- [ ] 함수에서 `anon` 권한만 회수하거나 `PUBLIC` 권한만 회수하면 실행 권한이 남을 수 있는 이유는 무엇인가?
- [ ] Spring + MSSQL 구조에서는 사용자별 DB 권한 문제를 덜 느끼고, Supabase에서는 더 자주 마주치는 이유는 무엇인가?
- [ ] `SECURITY INVOKER`와 `SECURITY DEFINER` 함수가 `anon`에 노출됐을 때 위험도가 다른 이유는 무엇인가?

## 백지 복습 답변

### 1. PUBLIC과 public 스키마

내 답:

-

피드백:

-

### 2. 기존 ACL과 default ACL

내 답:

-

피드백:

-

### 3. PUBLIC과 anon 권한 경로

내 답:

-

피드백:

-

### 4. Spring + MSSQL과 Supabase의 구조 차이

내 답:

-

피드백:

-

### 5. INVOKER와 DEFINER의 위험도

내 답:

-

피드백:

-

## 한 줄 회고

- 헷갈렸던 점: MSSQL에서는 잘 느끼지 못했던 권한 문제가 Supabase에서 계속 나타나 PostgreSQL만 유독 이상한 것처럼 느껴졌다. 이번 함수 기본 권한은 실제로 PostgreSQL과 MSSQL의 기본값이 다르지만, 더 큰 원인은 Supabase에서 DB가 백엔드 뒤에 숨지 않고 외부 API와 인가 계층의 일부로 직접 작동한다는 구조 차이였다.
