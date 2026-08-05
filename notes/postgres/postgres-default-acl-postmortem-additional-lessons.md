---
tags:
  - backend
  - postgres
  - supabase
  - security
  - postmortem
  - permissions
review_answered: false
---

# Legacy default ACL 사고 회고에서 더 배울 점

## 🎯 왜 공부했나

사이드프로젝트의 legacy Supabase 원격 QA 프로젝트에서 새 PostgreSQL 함수를 만들 때마다 `anon` 역할에 `EXECUTE` 권한이 자동으로 붙는 문제가 발견됐다.

7월에 당시 존재하던 쓰기 RPC의 `anon EXECUTE`를 전부 회수했지만, 새 함수의 초기 권한을 만드는 default ACL은 남아 있었다. 그 결과 sweep 이후 새로 생성한 함수 두 개에 `anon EXECUTE`가 다시 생겼다.

```text
7월
기존 함수의 anon EXECUTE 전부 회수
→ 당시 상태는 깨끗함

하지만 legacy default ACL은 유지
→ 다음 함수 생성 순간 anon EXECUTE 다시 부여
```

원격 검증에서 로컬에 없던 `anon` 권한을 발견했고, 기존 함수들과 신규 함수들을 생성 시점별로 비교한 다음 `pg_default_acl`을 조회해 원인을 확정했다.

사건의 핵심 해결은 다음 두 가지였다.

```text
현재 오염된 신규 함수 2개의 anon EXECUTE 회수
+
앞으로 권한을 자동 부여하는 legacy default ACL에서 anon 제거
```

이 회고는 이미 가설 수립, 형제 객체 비교, 시스템 카탈로그 실측, 재발 방지까지 잘 정리되어 있었다. 여기서는 그 회고를 읽고 추가로 배운 실무 포인트를 정리한다.

관련 기본 개념은 [[postgres-function-default-acl-supabase-vs-mssql]]에서 먼저 볼 수 있다.

---

## 🧠 사건에서 바로 얻은 핵심 교훈

회고의 다음 문장이 사건 전체를 잘 요약한다.

> 사람이 준 권한이 아니면 기계가 준 것이다.

여기에 한 문장을 더 붙이면 권한 감사의 기준이 된다.

> 현재 객체가 깨끗하다는 사실은 미래 객체도 깨끗하다는 뜻이 아니다. 현재 ACL뿐 아니라 권한을 생산하는 규칙과 최종 실효 권한까지 함께 검증해야 한다.

완성된 권한 감사에는 세 층이 필요하다.

```text
1. 현재 상태
   기존 함수의 ACL은 깨끗한가?

2. 생성 규칙
   default ACL, seed, bootstrap은 새 객체에 무엇을 부여하는가?

3. 실제 효과
   여러 권한 경로를 모두 계산했을 때 anon이 정말 실행 가능한가?
```

---

## 🚨 가장 중요한 추가 함정: 전역 default privilege와 스키마별 default privilege

PostgreSQL 함수는 내장 기본값으로 `PUBLIC EXECUTE`를 받는다.

```text
새 PostgreSQL 함수
→ 전역 기본 PUBLIC EXECUTE
→ 모든 역할이 PUBLIC 경로로 실행 가능
```

한편 legacy Supabase 프로젝트에는 다음처럼 `public` 스키마에서 API 역할에 직접 권한을 부여하는 별도 default ACL이 존재할 수 있다.

```text
scope = public
objtype = f
anon = EXECUTE
authenticated = EXECUTE
service_role = EXECUTE
```

두 권한 생산 장치는 서로 다른 층이다.

```text
PostgreSQL 전역 기본값
└─ PUBLIC EXECUTE

legacy Supabase의 public 스키마 default ACL
├─ anon 직접 EXECUTE
├─ authenticated 직접 EXECUTE
└─ service_role 직접 EXECUTE
```

### 스키마별 REVOKE로 전역 권한을 뺄 수 없다

다음 SQL은 PostgreSQL의 전역 기본 `PUBLIC EXECUTE`를 제거할 것처럼 보인다.

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE postgres
IN SCHEMA public
REVOKE EXECUTE ON FUNCTIONS
FROM PUBLIC;
```

하지만 PostgreSQL 공식 문서에 따르면 스키마별 default privilege는 전역 설정에 권한을 **추가**하는 구조다. 전역에서 부여된 권한을 스키마별 `REVOKE`로 뺄 수 없다.

```text
전역 default privilege
+ 스키마별 default privilege
= 새 객체의 초기 권한
```

스키마별 `REVOKE`는 이전에 같은 스키마 범위에서 실행한 `GRANT`를 취소할 때만 의미가 있다.

전역의 함수 `PUBLIC EXECUTE` 기본값을 제거하려면 `IN SCHEMA`를 빼야 한다.

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE postgres
REVOKE EXECUTE ON FUNCTIONS
FROM PUBLIC;
```

- [PostgreSQL ALTER DEFAULT PRIVILEGES](https://www.postgresql.org/docs/current/sql-alterdefaultprivileges.html)

### 이번 legacy anon 권한에는 스키마 범위가 맞다

사건에서 실측한 `anon=X`는 `scope=public`에 직접 부여된 권한이었다.

따라서 그 부여를 되돌리는 다음 명령은 의미가 있다.

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE postgres
IN SCHEMA public
REVOKE EXECUTE ON FUNCTIONS
FROM anon;
```

이를 구분하면 다음과 같다.

| 제거할 권한 | 필요한 default privilege 변경 |
| --- | --- |
| PostgreSQL 전역 기본 `PUBLIC EXECUTE` | `IN SCHEMA` 없이 `FROM PUBLIC` |
| legacy Supabase의 `public` 스키마 `anon EXECUTE` | `IN SCHEMA public ... FROM anon` |

> [!warning]
> "default ACL에서 anon을 제거했다"는 사실만으로 신규 함수가 완전한 deny-by-default가 됐다고 결론 내리면 안 된다. PostgreSQL 전역 `PUBLIC EXECUTE`가 별도로 살아 있는지 확인해야 한다.

### 회고의 표현을 더 정확히 만들기

회고에서 "부여 기계 자체를 껐다"는 표현은 사건 범위에서는 맞다. 하지만 정확한 범위를 붙이면 더 안전하다.

> legacy 프로젝트의 `anon` 직접 부여 기계를 껐다. PostgreSQL의 전역 신규 함수 `PUBLIC EXECUTE` 기본값까지 제거됐는지는 별도로 확인한다.

현재 함수들이 각각 `REVOKE FROM PUBLIC`을 수행했다면 지금은 안전할 수 있다. 그러나 전역 기본값이 살아 있으면 다음 함수 작성자가 `REVOKE FROM PUBLIC`을 빠뜨릴 때 같은 계열의 문제가 다시 생길 수 있다.

---

## 🎯 ACL 행이 아니라 실효 권한을 검사해야 한다

회고에서는 `information_schema.routine_privileges`를 사용해 함수의 권한 목록을 확인했다.

```sql
SELECT grantee, privilege_type
FROM information_schema.routine_privileges
WHERE routine_name = '<함수명>'
ORDER BY grantee;
```

이 조회는 다음 질문에 답하기 좋다.

> 이 함수에 어떤 역할의 grant 행이 기록되어 있는가?

하지만 보안에서 최종적으로 필요한 질문은 조금 다르다.

> 직접 grant, `PUBLIC`, 역할 상속 등 모든 경로를 계산했을 때 `anon`이 실제로 실행할 수 있는가?

### 직접 grant가 없어도 권한이 남을 수 있다

```text
routine_privileges에 anon 직접 행 없음

하지만
PUBLIC EXECUTE가 남아 있거나
anon이 사용할 수 있는 다른 role에 EXECUTE가 있으면

→ anon의 최종 실행 권한은 true일 수 있음
```

따라서 최종 검증에는 `has_function_privilege`를 함께 사용한다.

```sql
SELECT
  p.oid::regprocedure AS function_identity,
  has_function_privilege('anon', p.oid, 'EXECUTE')
    AS anon_can_execute,
  p.prosecdef AS security_definer,
  pg_get_userbyid(p.proowner) AS owner
FROM pg_proc p
JOIN pg_namespace n
  ON n.oid = p.pronamespace
WHERE n.nspname = 'public'
ORDER BY 1;
```

이 쿼리는 네 가지를 동시에 확인한다.

| 확인값 | 답하는 질문 |
| --- | --- |
| `oid::regprocedure` | 정확히 어떤 시그니처의 함수인가? |
| `has_function_privilege` | anon이 최종적으로 실행 가능한가? |
| `prosecdef` | `SECURITY DEFINER`인가? |
| `proowner` | 함수 owner는 누구인가? |

- [PostgreSQL Access Privilege Inquiry Functions](https://www.postgresql.org/docs/current/functions-info.html)

검증 목적도 두 층으로 나눈다.

```text
원인 검사
→ pg_default_acl에 잘못된 권한 생산 규칙이 남았는가?

결과 검사
→ has_function_privilege 기준으로 anon이 실제 실행 가능한가?
```

`pg_default_acl`만 보면 설정을 잘못 해석할 수 있고, `has_function_privilege`만 보면 권한이 어디서 생겼는지 원인을 알기 어렵다. 둘을 함께 봐야 한다.

---

## 🧪 미래 함수를 실제로 만들어 보는 canary 검증

Default ACL의 목적은 미래 객체에 초기 권한을 부여하는 것이다. 그렇다면 가장 직접적인 검증은 테스트용 미래 객체를 실제로 만들어 보는 것이다.

```sql
BEGIN;

CREATE FUNCTION public.__default_acl_probe()
RETURNS void
LANGUAGE plpgsql
AS $$
BEGIN
  NULL;
END;
$$;

SELECT has_function_privilege(
  'anon',
  'public.__default_acl_probe()',
  'EXECUTE'
) AS anon_can_execute;

ROLLBACK;
```

원하는 결과가 `false`라면 다음을 실제 결과로 확인한 것이다.

```text
현재 default privilege 조합 적용
→ 새 함수 생성
→ anon의 최종 EXECUTE 권한 없음
```

트랜잭션을 롤백하면 probe 함수도 남지 않는다.

이 방식은 **canary object** 또는 권한 카나리아라고 생각할 수 있다.

```text
권한 설정을 사람이 해석만 하는 대신
실제 미래 객체 하나를 넣어 보고
어떤 권한이 찍히는지 관찰한다.
```

### 중요한 조건: 실제 마이그레이션과 같은 역할로 생성해야 한다

Default ACL은 객체를 생성한 현재 역할을 기준으로 적용된다.

```text
마이그레이션 함수 생성 역할 = postgres
probe 함수 생성 역할          = audit_admin

→ 서로 다른 default ACL이 적용될 수 있음
→ 테스트가 실제 마이그레이션을 재현하지 못함
```

따라서 probe도 실제 마이그레이션과 같은 역할로 생성해야 한다.

---

## 🪪 함수 이름이 아니라 함수 시그니처로 감사하기

PostgreSQL은 함수 overloading을 지원한다. 이름이 같아도 입력 인자 타입이 다르면 별개의 함수다.

```text
public.foo(bigint)
public.foo(text)
public.foo(bigint, boolean)
```

다음 조회는 세 함수를 모두 `foo`라는 이름으로 묶는다.

```sql
WHERE routine_name = 'foo'
```

권한을 집계해서 한 줄로 만들면 특정 overload에만 남은 위험한 권한을 놓칠 수 있다.

PostgreSQL 함수의 보안상 정확한 신원은 다음 세 요소로 보는 게 좋다.

```text
스키마
+ 함수명
+ 입력 인자 타입
```

`pg_proc.oid::regprocedure`는 이를 사람이 읽기 좋은 형태로 보여준다.

```text
public.foo(bigint)
public.foo(text)
```

권한 회수도 정확한 시그니처를 사용한다.

```sql
REVOKE EXECUTE
ON FUNCTION public.foo(bigint)
FROM anon;
```

- [PostgreSQL CREATE FUNCTION](https://www.postgresql.org/docs/current/sql-createfunction.html)

사건 당시 대상 함수명이 모두 유일했다면 기존 조회도 사건 해결에는 충분했을 수 있다. 하지만 재사용할 보안 하네스는 overload를 구분하도록 만드는 편이 안전하다.

---

## 🏭 default ACL은 백그라운드 데몬이 아니라 생성 시점의 도장이다

"부여 기계"라는 비유는 원인을 이해하기에 좋다. 다만 기계가 언제 동작하는지 정확히 알아야 한다.

```text
default ACL
→ 객체가 새로 생성되는 순간 초기 권한을 찍는 도장
```

이미 존재하는 함수에 주기적으로 권한을 다시 붙이는 백그라운드 작업은 아니다.

```text
기존 함수에서 anon REVOKE
→ 함수가 그대로 존재하는 동안
→ default ACL이 몰래 다시 anon을 붙이지 않음
```

Default ACL이 다시 적용되는 대표적인 경우는 다음과 같다.

- 완전히 새로운 함수 생성
- 기존 함수를 `DROP`한 뒤 재생성
- 같은 이름이지만 다른 입력 타입의 overload 생성

### CREATE OR REPLACE는 기존 ACL을 유지한다

정확히 같은 함수 시그니처에 `CREATE OR REPLACE FUNCTION`을 사용하면 기존 함수 객체의 owner와 권한은 바뀌지 않는다.

```text
CREATE OR REPLACE FUNCTION
→ 기존 함수 객체 유지
→ 기존 ACL 유지

DROP FUNCTION + CREATE FUNCTION
→ 새 함수 객체
→ 현재 default ACL 적용
```

함수의 인자 타입을 바꾸면 기존 함수를 교체하는 것이 아니라 별도 overload가 만들어질 수도 있다. 이 경우 새 객체이므로 default ACL이 적용된다.

- [PostgreSQL CREATE FUNCTION](https://www.postgresql.org/docs/current/sql-createfunction.html)

이 차이는 마이그레이션 리뷰에서 중요하다.

```text
함수 본문만 교체
→ 기존 권한 유지 여부 확인

함수 drop/recreate 또는 새 signature
→ default ACL 재적용 여부 확인
```

---

## 🕵️ "실제 뚫림 0건"과 증거의 범위

회고에는 다음 사실들이 기록되어 있다.

```text
anon EXECUTE 권한 노출은 확인
함수 내부에는 관리자 또는 소유자 검사 존재
호출 로그는 조회하지 않음
```

이 증거로 확정할 수 있는 범위와 확정할 수 없는 범위를 구분해야 한다.

### 확정할 수 있는 것

- 의도하지 않은 `anon EXECUTE` 권한이 존재했다.
- 확인한 함수 경로에서는 내부 가드가 무단 데이터 변경을 차단하도록 구현되어 있었다.
- 현재 ACL과 default ACL에서 사건의 원인을 실측했다.

### 확정하지 않은 것

- 실제로 누군가 `anon`으로 함수를 호출했는가?
- 호출 시도가 몇 번 있었는가?
- 반복 호출로 비용이나 부하가 발생했는가?
- 오류 메시지나 실행 시간으로 정보가 노출됐는가?

따라서 다음 표현이 더 엄밀하다.

> 의도하지 않은 실행 권한 노출은 확인됐지만, 확인된 함수 경로에서는 내부 가드가 무단 변경을 차단했다. 실제 호출 시도 여부는 로그를 조회하지 않아 알 수 없고, 무단 데이터 변경 증거는 확인되지 않았다.

두 개념을 분리해야 한다.

```text
권한 노출
≠ 데이터 침해 확정

내부 가드가 데이터 변경 차단
≠ 아무 보안 영향도 없음
```

내부 가드가 쓰기를 차단해도 다음 공격 표면은 남을 수 있다.

- 반복 호출에 따른 DB 비용과 부하
- 오류 메시지를 통한 내부 상태 노출
- 응답 시간 차이를 통한 상태 추측
- 나중에 함수 본문이 변경되면서 가드가 약해지는 회귀

사후 회고에서 severity를 평가할 때는 다음을 구분한다.

```text
노출된 권한
실제 호출 가능성
내부 인가 가드
데이터 영향
호출 이력과 공격 증거
```

---

## 🚧 GRANT와 RLS 중 무엇이 첫 번째 문인가

회고의 사전 지식에는 다음과 같이 적혀 있었다.

> grant는 "이 role이 이 객체를 만질 수 있나"를 판단하는 1차 문이며 RLS보다 앞에서 검사된다.

그런데 테이블 default ACL의 `anon DML` 검토를 보류한 이유에서는 "RLS가 1차 방어"라고 표현했다. 두 문장은 읽는 사람에게 충돌해 보일 수 있다.

각 역할을 정확히 분리하면 다음과 같다.

```text
GRANT
→ 이 역할이 테이블 작업 자체를 시도할 수 있는가?

RLS
→ 허용된 테이블 작업에서 어떤 행을 읽거나 변경할 수 있는가?
```

실행 흐름으로 보면 객체 권한이 입구고, RLS가 행 단위 인가다.

```text
테이블 SELECT/INSERT/UPDATE/DELETE 권한 검사
→ RLS policy로 대상 행 검사
→ 실제 데이터 작업
```

Supabase에서는 `anon`과 `authenticated`에 테이블 DML 권한을 부여하고 RLS로 행을 제한하는 구조가 의도된 설계일 수 있다. 따라서 테이블 default ACL을 함수 사건과 함께 성급히 회수하지 않고 별도 todo로 분리한 판단 자체는 합리적이다.

다만 보류 근거는 다음처럼 표현하는 편이 정확하다.

> 현재 RLS가 행 단위 영향을 완화하고 있으나, broad table grant가 의도된 API 설계인지와 모든 RLS 정책이 충분한지는 별도 범위에서 검증해야 하므로 분리한다.

RLS가 있다고 table grant 검토가 불필요한 것은 아니다. 반대로 `anon` table grant가 있다는 사실만으로 곧바로 취약점인 것도 아니다. Supabase의 의도된 접근 모델과 RLS 정책을 함께 봐야 한다.

---

## 🌍 로컬 PASS와 원격 PASS가 증명하는 것은 다르다

사건 당시 신형 CLI 로컬 환경에는 legacy `anon` 함수 default ACL이 없었다.

```text
로컬 초기 상태
→ 이미 anon default ACL 없음

원격 legacy 초기 상태
→ anon default ACL 있음
```

이 상태에서 로컬 fresh reset 검증이 통과한 것은 다음을 증명한다.

```text
깨끗한 초기 환경에서
마이그레이션 전체를 적용해도
원하는 최종 상태가 유지된다.
```

하지만 로컬에는 처음부터 오염이 없었으므로 다음 전환까지 직접 증명하지는 않는다.

```text
legacy anon default ACL이 있는 환경에
회수 마이그레이션을 적용하면
그 오염이 실제로 제거된다.
```

이번 사건에서는 실제 legacy 원격에 마이그레이션을 적용하고 확인했기 때문에 전환 근거가 생긴다.

검증 증거를 나누면 다음과 같다.

| 검증 환경 | 증명하는 것 |
| --- | --- |
| 로컬 fresh reset | 깨끗한 환경의 최종 상태 유지 |
| legacy 원격 적용 | 실제 오염 상태가 교정됨 |
| legacy fixture 기반 테스트 | 오염에서 정상으로 가는 전환을 반복 재현 |

장기적으로 migration transition 자체를 테스트하고 싶다면 legacy default ACL을 가진 fixture에서 회수 마이그레이션을 적용하는 테스트가 가장 강하다.

다만 회귀 방지 목적이라면 최종 상태를 계속 감시하는 verify도 가치가 있다.

```text
전환 테스트
→ 과거 오염을 제대로 고쳤는가?

상태 불변식 테스트
→ 앞으로 다시 오염되지 않았는가?
```

둘은 서로 다른 테스트다.

---

## 🧬 환경도 생성 이력을 가진다

로컬과 원격에 같은 migration 파일이 있다고 해서 현재 DB 상태가 완전히 같다는 보장은 없다.

```text
현재 migration 코드
+ 프로젝트가 처음 만들어질 때의 플랫폼 기본값
+ 과거 dashboard 설정
+ seed와 bootstrap
+ 관리 역할이 만든 객체
+ 수동 운영 변경
= 현재 데이터베이스 상태
```

이번 사건에서 원격은 legacy Supabase 프로젝트 생성 당시의 default ACL을 가지고 있었고, 새 로컬 스택에는 그 설정이 없었다.

즉 차이는 단순히 최신 migration 파일의 차이가 아니라 **환경의 생성 이력 차이**였다.

이런 차이를 configuration provenance 또는 환경 provenance라고 생각할 수 있다.

> 데이터뿐 아니라 권한과 기본 설정도 과거 사건의 흔적을 가진 상태다.

따라서 다음 정보도 회고와 재현 문서에 남기면 좋다.

- 원격 프로젝트 생성 시기
- Supabase CLI 버전
- 로컬 이미지 또는 Postgres 버전
- migration 실행 역할
- 함수 owner
- seed와 bootstrap 적용 여부
- dashboard에서 변경한 Data API 설정

"신형 CLI 로컬에는 없다"는 관찰을 모든 새 Supabase 프로젝트의 보편적 특성으로 일반화하기보다는, 확인한 버전과 환경을 함께 기록하는 편이 안전하다.

---

## 🔍 시스템 카탈로그는 인접 범위까지 훑기

8월 4일 시퀀스 권한 사건에서도 `pg_default_acl`을 조회했지만 `objtype = S`만 확인해서 같은 표에 있던 함수 `objtype = f`의 `anon=X`를 놓쳤다.

이 사건에서 얻는 감사 범위 교훈은 다음과 같다.

```text
특정 가설을 검증하기 위한 좁은 조회
→ 빠르고 정확함

하지만 같은 시스템 카탈로그의 인접 행에
같은 계열의 오염이 있을 수 있음
→ 한 번은 전체 분포도 확인
```

좋은 조사 순서는 다음과 같다.

```text
1. 좁은 필터로 현재 가설을 빠르게 검증
2. 원인을 찾은 뒤 필터를 풀어 같은 계열의 인접 상태 확인
3. objtype, owner, schema별로 그룹화해 전체 분포 확인
```

무조건 모든 카탈로그를 전수 조사하라는 뜻은 아니다. 특정 권한 생산 규칙을 발견했으면 그 규칙이 적용될 수 있는 **같은 표의 인접 범위**를 한 번 확인한다는 뜻이다.

```sql
SELECT
  pg_get_userbyid(d.defaclrole) AS acl_owner,
  CASE
    WHEN d.defaclnamespace = 0 THEN '(global)'
    ELSE d.defaclnamespace::regnamespace::text
  END AS scope,
  d.defaclobjtype AS objtype,
  d.defaclacl::text AS acl
FROM pg_default_acl d
ORDER BY 1, 2, 3;
```

여기서는 특히 세 축을 함께 본다.

```text
누가 만드는가?   → acl_owner
어디에 만드는가? → scope
무엇을 만드는가? → objtype
```

---

## 🧰 재사용 가능한 권한 디버깅 순서

로컬은 깨끗한데 원격에서만 이상한 권한 문제가 발견되면 다음 순서로 좁혀 간다.

### 1. 대상 객체의 현재 권한 실측

```sql
SELECT grantee, privilege_type
FROM information_schema.routine_privileges
WHERE specific_schema = 'public'
  AND routine_name = '<함수명>'
ORDER BY grantee;
```

먼저 증상을 숫자와 행으로 확인한다.

### 2. 같은 처지의 형제 객체와 비교

```text
오염된 함수와 깨끗한 함수의 차이
→ 생성 날짜
→ 생성 migration
→ owner
→ schema
→ CREATE와 CREATE OR REPLACE 여부
```

회고에서는 sweep 이전 함수 3종과 이후 신규 함수 2종을 비교해 "신설 시점 자동 부여" 패턴을 찾았다.

### 3. 자동 부여 장치 실측

```text
default ACL
seed blanket GRANT
bootstrap SQL
dashboard 설정
DDL event trigger
```

사람이 작성한 migration에 grant가 없다면 권한을 만드는 다른 기계를 찾는다.

### 4. 실효 권한 확인

```sql
SELECT has_function_privilege(
  'anon',
  'public.target_function(bigint)',
  'EXECUTE'
);
```

직접 ACL 행의 유무가 아니라 최종 결과를 확인한다.

### 5. 실제 역할로 부정 테스트

```sql
BEGIN;
SET LOCAL ROLE anon;
SELECT public.target_function(123);
ROLLBACK;
```

함수 경계에서 거부되어야 한다면 내부 비즈니스 가드의 오류가 아니라 `EXECUTE` 권한 부족으로 실패하는지도 구분한다.

### 6. 미래 객체 카나리아

실제 migration creator role로 probe 함수를 생성해서 새 함수의 최종 권한을 확인한다.

```text
현재 함수 확인
+ default ACL 구조 확인
+ 미래 함수 생성 결과 확인
```

이 세 가지를 모두 통과하면 현재와 미래를 함께 검증할 수 있다.

---

## 📊 회고의 severity를 평가하는 기준

`anon EXECUTE`가 있었다는 사실만으로 모든 사건의 위험도가 같지는 않다.

다음 질문을 순서대로 본다.

| 질문 | 위험 판단에 주는 정보 |
| --- | --- |
| 함수가 `SECURITY DEFINER`인가? | owner 권한으로 실행되는지 |
| 함수 owner가 누구인가? | 얻을 수 있는 최대 권한 범위 |
| 내부 인증·인가 가드가 있는가? | 호출 후 무단 작업을 막는지 |
| RLS를 우회하는가? | 행 단위 보호가 남는지 |
| 함수가 쓰기·삭제를 수행하는가? | 데이터 영향 범위 |
| 반복 호출 비용이 큰가? | 가용성 공격 가능성 |
| 오류와 반환값이 정보를 노출하는가? | 정보 노출 가능성 |
| 실제 호출 로그가 있는가? | 노출이 악용됐는지에 대한 증거 |

이번 사건은 다음처럼 분류할 수 있다.

```text
의도하지 않은 익명 실행 권한 노출: 확인
내부 인가 가드: 확인
알려진 무단 데이터 변경 경로: 차단
실제 익명 호출 이력: 미확인
무단 데이터 변경 증거: 확인되지 않음
```

이렇게 쓰면 위험을 축소하지도 않고, 증거보다 크게 침해를 단정하지도 않는다.

---

## 📝 최종 정리

### 1. Sweep은 현재 객체만 청소한다

기존 함수의 권한을 전부 회수해도 default ACL, seed 같은 생성 규칙이 남으면 다음 객체부터 같은 권한이 다시 생긴다.

### 2. 권한 생산 규칙도 층이 있다

PostgreSQL 전역 `PUBLIC EXECUTE`와 legacy Supabase의 스키마별 `anon EXECUTE`는 서로 다른 규칙이다. 각각 올바른 범위에서 제거해야 한다.

### 3. ACL 행의 부재는 최종 권한 부재와 다르다

`routine_privileges`로 직접 grant를 보고, `has_function_privilege`로 `PUBLIC`과 역할 상속까지 포함한 실효 권한을 확인한다.

### 4. 미래 동작은 미래 객체로 검증한다

실제 migration creator role로 probe 함수를 생성해 default ACL 적용 결과를 확인하면 카탈로그 해석 실수를 줄일 수 있다.

### 5. 함수는 이름이 아니라 시그니처로 식별한다

Overload 때문에 같은 이름의 여러 함수가 존재할 수 있으므로 `schema.name(argument types)` 또는 OID를 사용한다.

### 6. CREATE OR REPLACE는 기존 ACL을 유지한다

Default ACL은 새로운 객체 생성 시 적용된다. 정확히 같은 함수의 `CREATE OR REPLACE`는 기존 권한을 유지하고, drop/recreate나 새 overload에는 현재 default ACL이 적용된다.

### 7. 노출과 침해는 구분한다

의도하지 않은 권한 노출은 확정할 수 있지만 로그를 확인하지 않았다면 실제 호출 여부까지 단정할 수 없다. 내부 가드는 데이터 변경 위험을 낮추지만 호출 표면 자체를 없애지는 않는다.

### 8. 환경에는 생성 이력이 남는다

현재 migration 파일이 같아도 legacy 원격과 fresh 로컬은 bootstrap과 default ACL이 다를 수 있다. 권한 상태도 데이터처럼 환경별 이력을 가진다.

> [!summary]
> 권한 감사의 완결 조건은 "현재 객체 정리 + 권한 생산 규칙 교정 + 실제 역할의 실효 권한 검증"이다. 현재 ACL이 깨끗하다는 사실만으로 미래 객체의 안전성을 보장할 수 없으며, 실제 creator role로 카나리아 객체를 만들어 결과를 확인하는 것이 가장 직접적인 검증이다.

## 복습 질문

- [ ] PostgreSQL의 전역 함수 `PUBLIC EXECUTE`와 legacy Supabase의 스키마별 `anon EXECUTE`는 무엇이 다르고, 각각 어떻게 제거해야 하는가?
- [ ] `routine_privileges`에서 `anon` 행이 보이지 않아도 `anon`이 실제로 함수를 실행할 수 있는 이유는 무엇인가?
- [ ] Default ACL을 검증할 때 실제 migration creator role로 probe 함수를 만들어야 하는 이유는 무엇인가?
- [ ] `CREATE OR REPLACE FUNCTION`과 `DROP FUNCTION` 후 재생성은 ACL 적용 측면에서 어떻게 다른가?
- [ ] 권한 노출은 확인됐지만 호출 로그를 보지 않은 사건에서 "실제 침해 0건"이라고 단정하면 안 되는 이유는 무엇인가?

## 백지 복습 답변

### 1. 전역 PUBLIC과 스키마별 anon default privilege

내 답:

-

피드백:

-

### 2. 직접 ACL과 실효 권한

내 답:

-

피드백:

-

### 3. 미래 함수 카나리아 검증

내 답:

-

피드백:

-

### 4. CREATE OR REPLACE와 재생성

내 답:

-

피드백:

-

### 5. 권한 노출과 실제 침해 증거

내 답:

-

피드백:

-

## 한 줄 회고

- 헷갈렸던 점: 기존 함수의 `anon` 권한과 `pg_default_acl`의 `anon=X`만 제거하면 신규 함수 권한 문제가 완전히 끝난다고 생각하기 쉽다. 하지만 PostgreSQL 전역의 `PUBLIC EXECUTE`, 스키마별 직접 grant, creator role, 함수 overload가 서로 다른 축이며, grant 행이 아니라 `has_function_privilege`와 실제 probe 함수로 최종 결과까지 확인해야 한다.
