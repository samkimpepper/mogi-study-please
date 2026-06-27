---
tags:
  - backend
  - spring
  - architecture
  - error-handling
  - supabase
  - typescript
---

# Spring 베스트프랙티스로 보는 Supabase 직접 호출 구조의 경계 문제

> [!summary]
> 한 줄 요약:
>
> - Spring에서는 Controller, Service, Repository, Exception Handler 같은 경계를 두고 각 계층의 책임을 분리한다.
> - React + Supabase 직접 호출 구조에서는 `supabase` 클라이언트를 아무 파일에서나 import할 수 있어서, DB 접근 경계가 관례로만 존재한다.
> - 경계가 강제되지 않으면 DB 접근, 에러 분류, 타입 정의, 응답 정책이 각 파일로 새고, 같은 문제가 여러 증상으로 흩어진다.

## 왜 공부했나

> [!question]
> 처음 헷갈렸던 질문:
>
> - 왜 프론트 컴포넌트가 Supabase를 직접 호출하면 위험할까?
> - Spring으로 치면 이 구조는 어떤 안티패턴에 가까울까?
> - 에러 처리 문제, 타입 문제, 모듈화 문제가 왜 한 뿌리라고 볼 수 있을까?

이번에 본 프로젝트는 전용 백엔드 서버 없이 React 클라이언트가 Supabase를 직접 호출한다.

즉, 프론트 코드가 사실상 DB 클라이언트를 들고 있다.

```ts
import { supabase } from "@/data/supabaseClient";
```

이 한 줄을 아무 파일에서나 쓸 수 있으면, 컴포넌트든 admin 화면이든 repo든 어디서나 DB에 접근할 수 있다.

겉으로는 `app/src/data/*Repo.ts` 같은 repository 레이어가 있지만, 그 레이어를 반드시 거치도록 강제하는 장치가 없다.

Spring으로 치면 이런 상태다.

```text
Repository 레이어는 만들어놨다.
하지만 Controller에서도 DataSource나 JdbcTemplate을 직접 꺼내 raw SQL을 날릴 수 있다.
그리고 아무도 빌드 시점에 막지 않는다.
```

## Spring 베스트프랙티스의 기본 경계

일반적인 Spring 백엔드에서는 계층을 나눈다.

```text
Controller
-> Service
-> Repository
-> DB
```

각 계층은 책임이 다르다.

| 계층 | 책임 | 하지 말아야 할 일 |
| --- | --- | --- |
| Controller | HTTP 요청/응답, DTO 변환 | DB 직접 접근, 비즈니스 규칙 처리 |
| Service | 비즈니스 흐름, 트랜잭션, 도메인 규칙 | HTTP 응답 직접 생성, SQL 상세 의존 |
| Repository | DB 접근, 쿼리 실행 | 사용자 메시지 결정, 화면 정책 결정 |
| Exception Handler | 예외를 공통 응답으로 변환 | 비즈니스 로직 수행 |

예를 들어 좋은 흐름은 이렇다.

```java
@RestController
public class UserController {

    private final UserService userService;

    @PostMapping("/users")
    public ResponseEntity<Void> createUser(@RequestBody CreateUserRequest request) {
        userService.createUser(request);
        return ResponseEntity.ok().build();
    }
}
```

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    @Transactional
    public void createUser(CreateUserRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new BusinessException(ErrorCode.DUPLICATE_EMAIL);
        }

        userRepository.save(new User(request.email()));
    }
}
```

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    boolean existsByEmail(String email);
}
```

Controller는 HTTP만 알고, Service는 비즈니스 흐름을 알고, Repository는 DB 접근만 안다.

이 경계가 깨지면 코드가 빠르게 섞인다.

## Spring으로 치면 나쁜 구조

Supabase 클라이언트를 아무 컴포넌트에서나 import하는 구조는 Spring으로 치면 Controller에서 `JdbcTemplate`이나 `DataSource`를 직접 쓰는 것과 비슷하다.

```java
@RestController
public class UserController {

    private final JdbcTemplate jdbcTemplate;

    @GetMapping("/users")
    public List<UserResponse> users() {
        return jdbcTemplate.query(
                "select id, email from users",
                (rs, rowNum) -> new UserResponse(
                        rs.getLong("id"),
                        rs.getString("email")
                )
        );
    }
}
```

동작은 한다.

하지만 구조적으로는 좋지 않다.

- Controller가 DB 접근을 안다.
- SQL 결과 매핑을 Controller가 안다.
- DB 예외가 Controller까지 직접 올라온다.
- 같은 쿼리나 에러 처리가 다른 Controller에 복붙될 수 있다.
- 나중에 Repository 정책을 만들어도 이미 우회 코드가 생긴다.

프론트 코드로 치면 이런 상태다.

```tsx
function AdminCategoryTree() {
  const load = async () => {
    const { data, error } = await supabase
      .from("categories")
      .select("*");

    if (error) {
      setError(error.message);
      return;
    }

    setCategories(data);
  };
}
```

여기서는 컴포넌트가 너무 많은 것을 안다.

- 어떤 테이블을 조회하는지 안다.
- Supabase 에러 형태를 안다.
- DB 에러 메시지를 화면에 그대로 보여준다.
- repository 레이어를 우회한다.

> [!warning]
> 주의할 점:
>
> - 레이어가 파일 구조로 존재하는 것과, 실제로 경계가 강제되는 것은 다르다.
> - `repo를 쓰자`는 약속만 있고 `supabase 직접 import 금지`가 없으면 급한 코드가 경계를 우회한다.

## 비교 1: DB 접근 경계

Spring 베스트프랙티스에서는 DB 접근을 Repository로 모은다.

```text
Controller -> Service -> Repository -> DB
```

Supabase 직접 호출 구조에서 기대하는 이상적인 경계도 비슷하다.

```text
React Component -> data repo -> Supabase -> DB
```

문제는 이 경계가 강제되지 않는다는 점이다.

```text
React Component -> Supabase -> DB
```

백엔드로 치면 다음과 같은 우회다.

```text
Controller -> DataSource/JdbcTemplate -> DB
```

이 우회가 허용되면 repository 레이어가 있어도 점점 힘을 잃는다.

> [!tip]
> Spring식 교훈:
>
> Repository 레이어를 만들었다면, 상위 계층이 DB 클라이언트를 직접 만지지 못하게 의존성 방향을 제한해야 한다.

## 비교 2: 에러 처리 경계

Spring에서는 DB 저수준 예외를 애플리케이션 에러로 바꾸고, 전역 핸들러에서 응답을 만든다.

```text
SQLException
-> DataAccessException
-> BusinessException 또는 ErrorCode
-> @RestControllerAdvice
-> ErrorResponse
```

좋은 구조에서는 Controller마다 DB 에러코드를 직접 분기하지 않는다.

```java
@ExceptionHandler(BusinessException.class)
public ResponseEntity<ErrorResponse> handle(BusinessException e) {
    ErrorCode code = e.getErrorCode();
    return ResponseEntity.status(code.getStatus())
            .body(ErrorResponse.from(code));
}
```

반면 현재 Supabase 구조에서는 PostgreSQL 에러코드가 여러 파일에 흩어진다.

```ts
if (error.code === "23505") {
  // 중복
}

if (error.code === "42501") {
  // 권한 없음
}
```

이것은 Spring으로 치면 서비스마다 `SQLException`을 직접 열어보는 느낌이다.

```java
catch (SQLException e) {
    if ("23505".equals(e.getSQLState())) {
        throw new BusinessException(ErrorCode.DUPLICATE);
    }
}
```

한두 곳이면 참을 수 있어도, 여러 파일에 퍼지면 정책이 갈라진다.

```text
같은 23505인데 A 파일은 duplicate
같은 23505인데 B 파일은 generic error
같은 42501인데 C 파일은 auth_required
같은 42501인데 D 파일은 forbidden
```

Spring식으로 보면 `SQLExceptionTranslator`와 `@RestControllerAdvice`가 해주던 일을 각 파일이 손으로 나눠 하고 있는 셈이다.

## 비교 3: 메시지 문자열 분기

가장 위험한 패턴은 메시지 문자열로 에러 의미를 나누는 것이다.

```ts
if (error.code === "42501") {
  return message.includes("not assessment author")
    ? "forbidden"
    : "auth_required";
}
```

이 코드는 `42501` 하나를 두 의미로 나누기 위해 메시지를 본다.

- 로그인 안 함
- 로그인은 했지만 남의 글이라 권한 없음

하지만 메시지는 사람에게 보여주기 위한 문장이다.

SQL 함수의 `RAISE EXCEPTION` 문구가 바뀌면 분기가 조용히 깨진다.

Spring으로 치면 이런 코드다.

```java
if (e.getMessage().contains("not owner")) {
    throw new BusinessException(ErrorCode.FORBIDDEN);
}

throw new BusinessException(ErrorCode.AUTH_REQUIRED);
```

이것은 좋은 방식이 아니다.

Spring에서는 보통 서로 다른 의미를 서로 다른 예외나 `ErrorCode`로 표현한다.

```java
throw new BusinessException(ErrorCode.AUTH_REQUIRED);
throw new BusinessException(ErrorCode.FORBIDDEN);
```

DB 함수에서 던지는 에러라면, 메시지가 아니라 안정적인 코드, hint, detail, 별도 SQLSTATE 같은 값으로 구분할 수 있어야 한다.

> [!danger]
> 위험한 오해:
>
> - 메시지 문자열은 분기 기준이 아니다.
> - 메시지는 사용자나 개발자가 읽는 설명이고, 언제든 바뀔 수 있다.
> - 프로그램 분기는 안정적인 code, enum, exception type으로 해야 한다.

## 비교 4: 타입 경계

Spring에서는 Entity, DTO, Repository 반환 타입을 통해 데이터 모양을 관리한다.

예를 들어 Controller가 DB row를 직접 다루지 않고 응답 DTO로 변환한다.

```java
public record UserResponse(Long id, String email) {
    public static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getEmail());
    }
}
```

Supabase는 DB 스키마에서 TypeScript 타입을 자동 생성할 수 있다.

```text
database.types.ts = DB 스키마의 자동 생성 타입
```

이 타입이 DB 모양의 단일 소스에 가깝다.

그런데 각 repo가 손으로 row 타입을 만들고 `as`로 단언하면 문제가 생긴다.

```ts
const row = rpcData[0] as {
  swatch_id?: number;
  swatch_row?: unknown;
};
```

`as`는 컴파일러에게 "내가 맞으니 믿어라"라고 말하는 것이다.

DB 스키마가 바뀌어도 손으로 만든 타입은 자동으로 따라오지 않는다.

Spring으로 치면 DB 스키마나 Entity는 바뀌었는데, 낡은 RowMapper나 DTO를 계속 믿는 상황과 비슷하다.

> [!warning]
> 타입도 경계다.
>
> - 자동 생성 타입이라는 단일 소스를 두고 손 타입을 계속 만들면 둘이 표류한다.
> - `as`가 많아질수록 컴파일러가 잡아줄 수 있는 문제가 줄어든다.
> - 타입 단언이 반복되는 곳은 경계가 약하거나 데이터 모양이 정리되지 않았다는 신호일 수 있다.

## 비교 5: 전역 클라이언트와 의존성 방향

Spring에서도 어디서나 `DataSource`를 주입받을 수 있게 열어두면 경계가 약해진다.

```java
@Service
public class SomeService {
    private final DataSource dataSource;
}

@RestController
public class SomeController {
    private final DataSource dataSource;
}
```

기술적으로는 가능해도, 모든 계층이 DB 클라이언트를 직접 알게 된다.

Supabase 전역 클라이언트도 비슷하다.

```ts
import { supabase } from "@/data/supabaseClient";
```

이 import가 어디서나 가능하면 의존성 방향이 무너진다.

좋은 방향은 이런 식이다.

```text
features/*
-> data repositories
-> supabaseClient
```

피해야 할 방향은 이렇다.

```text
features/*
-> supabaseClient
```

경계를 강제하려면 단순 컨벤션보다 도구가 필요하다.

- ESLint import 제한
- feature 디렉토리에서 `supabaseClient` 직접 import 금지
- data repo 밖에서 `supabase.from` 사용 금지
- 아키텍처 테스트
- 모듈 경계 규칙
- 코드리뷰 체크리스트

Spring 쪽으로 대응하면 이런 도구와 비슷하다.

- ArchUnit 테스트
- 패키지 의존성 규칙
- 모듈 분리
- `Controller`에서 `Repository` 직접 주입 금지
- `Controller`에서 `DataSource`/`JdbcTemplate` 직접 사용 금지

## 세 문제가 한 뿌리인 이유

겉으로 보면 문제가 여러 개다.

```text
#276 DTO/타입 문제
#277 에러 처리 문제
#278 모듈화/경계 문제
```

하지만 뿌리는 하나에 가깝다.

```text
supabase 전역 호출에 강제 경계가 없음
```

이 뿌리에서 여러 증상이 나온다.

```text
DB 접근 경계 없음
-> 컴포넌트가 직접 DB 호출

에러 처리 경계 없음
-> PG 코드와 메시지 분기가 파일마다 흩어짐

타입 경계 없음
-> 자동 생성 타입 대신 손 타입 + as 단언
```

Spring으로 번역하면 이렇다.

```text
DataSource 아무 데서나 주입 가능
-> raw SQL 산재
-> SQLException 직접 처리 산재
-> DTO/RowMapper 산재
```

따라서 이 문제를 각각 따로만 고치면 임시 처방이 된다.

```text
에러만 살짝 고침
타입만 살짝 고침
admin만 살짝 고침
```

공통 원인은 경계다.

> [!summary]
> 에러 처리 문제는 단독 문제가 아니라, DB 접근 경계가 약해서 생긴 증상 중 하나다.

## Spring 베스트프랙티스와 현재 구조 비교

| 주제 | Spring 베스트프랙티스 | 현재 Supabase 직접 호출 구조의 문제 | 개선 방향 |
| --- | --- | --- | --- |
| DB 접근 | Repository만 DB 접근 | 컴포넌트/admin이 Supabase 직접 호출 가능 | data repo를 통과하도록 import 제한 |
| 비즈니스 흐름 | Service에서 처리 | DB 함수, repo, 컴포넌트에 분산 | 책임 경계 명확화 |
| 에러 분류 | `SQLExceptionTranslator`, `ErrorCode` | PG 코드 매핑이 파일마다 산재 | `classifyDataError` + `DATA_ERROR` |
| 에러 응답 | `@RestControllerAdvice` 한 곳 | repo별 reason, UI별 처리 제각각 | 공통 AppError와 UI 정책 정리 |
| 분기 기준 | exception type, enum, code | 일부 메시지 문자열 분기 | 안정적인 code/detail/hint로 분기 |
| 타입 | Entity/DTO/Repository 타입 | 손 Row 타입 + `as` 단언 | 자동 생성 타입과 DTO 단일화 |
| 경계 강제 | ArchUnit, 모듈, 패키지 규칙 | 컨벤션 중심 | ESLint boundary, import restriction |

## 출시 안정화 기간의 현실적 접근

뿌리는 경계 문제지만, 항상 큰 수술을 바로 할 수는 없다.

출시 안정화 기간이라면 최소판이 현실적일 수 있다.

```text
1. data 에러 분류만 우선 한 곳으로 모으기
2. 메시지 문자열 분기처럼 위험한 부분 표시하기
3. feature에서 supabase 직접 import하는 새 코드만 막기
4. admin 내부 도구는 당장 전면 정리하지 않고 범위 밖으로 명시하기
5. 코프링 이전 또는 모듈화 작업 때 경계 강제를 본격 처리하기
```

중요한 것은 지금 모든 것을 완벽하게 고치는 것이 아니라, 어떤 문제가 임시 처방이고 어떤 문제가 구조적 원인인지 구분하는 것이다.

## 실무에서 보는 포인트

| 관점 | 확인할 것 | 왜 중요한가 |
| --- | --- | --- |
| 계층 | 상위 계층이 하위 기술을 직접 아는가 | 경계가 새면 변경에 약해진다 |
| 강제 | 규칙이 빌드나 lint로 막히는가 | 관례만으로는 급한 코드가 우회한다 |
| 에러 | 외부/DB 에러가 한 곳에서 번역되는가 | 같은 실패가 같은 의미로 처리되어야 한다 |
| 타입 | 데이터 모양의 단일 소스가 있는가 | 손 타입과 실제 DB 스키마가 표류할 수 있다 |
| 운영 | 메시지 문자열로 분기하지 않는가 | 문구 변경이 로직 버그로 이어질 수 있다 |
| 설계 | 여러 증상의 공통 원인을 보는가 | 증상별 패치보다 구조적 해결이 중요하다 |

## 헷갈리기 쉬운 부분

> [!warning]
> 서버리스 구조가 무조건 나쁘다는 뜻은 아니다.
>
> - Supabase 직접 호출 구조는 빠르게 만들 수 있고, RLS와 RPC를 잘 쓰면 강력하다.
> - 다만 전용 백엔드 서버가 해주던 경계 강제, 에러 변환, DTO 확정 책임이 사라지거나 클라이언트로 이동한다.
> - 그래서 그 책임을 프론트 구조 안에서 의식적으로 만들어야 한다.

> [!warning]
> Repository 파일이 있다고 경계가 생기는 것은 아니다.
>
> - `app/src/data/*Repo.ts`가 있어도 컴포넌트에서 `supabase`를 직접 import할 수 있으면 우회가 가능하다.
> - 경계는 파일 이름이 아니라 의존성 방향과 강제 규칙으로 만들어진다.

> [!danger]
> 위험한 오해:
>
> - "나중에 코드리뷰로 보면 된다"는 말만으로는 부족하다.
> - 급한 작업, admin 도구, 임시 패치가 항상 경계를 우회한다.
> - 반복되는 우회는 결국 새로운 관성이 된다.

## 면접 답변으로 말하면

> [!tip]
> 짧게 답변:
>
> Spring에서는 Controller, Service, Repository, Exception Handler처럼 계층별 책임을 나누고, DB 접근과 예외 변환을 특정 계층에 모읍니다. 이 경계가 깨져서 Controller가 DataSource를 직접 쓰거나 SQLException을 직접 분기하면 유지보수가 어려워집니다. React + Supabase 직접 호출 구조에서도 비슷하게, 컴포넌트가 Supabase 클라이언트를 직접 import하면 DB 접근, 에러 처리, 타입 정의가 각 화면으로 흩어질 수 있습니다. 그래서 repository를 통과하도록 import 경계를 강제하고, 에러 분류와 타입의 단일 소스를 두는 것이 중요합니다.

> [!note]-
> 긴 설명
>
> 서버리스나 BaaS 구조에서는 전용 백엔드 서버가 없기 때문에, 프론트가 DB에 가까운 책임을 일부 떠안게 됩니다. 이때 단순히 `data repo를 쓰자`는 컨벤션만으로는 부족합니다. 아무 컴포넌트에서나 Supabase 클라이언트를 import할 수 있다면, 급한 기능이나 admin 도구가 repository를 우회할 가능성이 높습니다.
>
> Spring으로 치면 Repository 레이어를 만들어놓고도 Controller에서 JdbcTemplate이나 DataSource를 직접 사용하는 것을 막지 않는 상황입니다. 이렇게 되면 SQL, DB 에러코드, 응답 메시지, 타입 매핑이 여러 계층으로 퍼집니다. 결과적으로 같은 에러가 파일마다 다르게 처리되고, 메시지 문자열로 분기하는 위험한 코드가 생기며, 자동 생성 타입 대신 손 타입과 `as` 단언이 늘어납니다.
>
> 따라서 핵심은 경계를 강제하는 것입니다. Spring에서는 ArchUnit, 패키지 규칙, 모듈 분리로 의존성 방향을 제한할 수 있고, 프론트에서는 ESLint import rule이나 boundary rule로 feature 코드가 Supabase 클라이언트를 직접 import하지 못하게 할 수 있습니다. 에러는 `classifyDataError`와 `DATA_ERROR` 같은 단일 분류 계층으로 모으고, 타입은 자동 생성 타입이나 DTO 단일 소스를 기준으로 관리해야 합니다.

## 나중에 다시 볼 포인트

- `repo를 쓰자`는 관례와 `repo를 거치지 않으면 빌드가 깨진다`는 강제는 다르다.
- Supabase 클라이언트를 아무 파일에서나 import할 수 있으면 DB 접근 경계가 약해진다.
- Spring으로 치면 Controller가 Repository를 우회해서 DataSource/JdbcTemplate을 직접 쓰는 느낌이다.
- 에러 처리, 타입 단언, admin 우회는 각각 다른 문제가 아니라 경계 부재의 다른 증상일 수 있다.
- 메시지는 사람용이고, 분기는 code/enum/exception type 같은 안정적인 값으로 해야 한다.
- 자동 생성 타입이라는 단일 소스가 있으면 손 타입과 `as` 단언을 줄여야 한다.
- 여러 증상이 보이면 바로 각각 패치하기 전에 공통 원인을 찾아야 한다.

## 복습 질문

- [ ] Supabase 클라이언트를 컴포넌트에서 직접 import하는 것은 Spring으로 치면 어떤 구조와 비슷한가?
- [ ] Repository 레이어가 파일로 존재하는 것과 경계가 강제되는 것은 어떻게 다른가?
- [ ] 메시지 문자열로 에러를 분기하면 왜 위험한가?
- [ ] 에러 처리 문제, 타입 문제, 모듈화 문제가 왜 같은 뿌리에서 나왔다고 볼 수 있는가?
- [ ] Spring에서는 계층 경계를 강제하기 위해 어떤 방법을 쓸 수 있고, 프론트에서는 어떤 방법을 쓸 수 있는가?

## 한 줄 회고

- 헷갈렸던 점: 문제의 핵심은 React나 TypeScript 자체가 아니라, 전용 백엔드가 하던 경계 강제와 예외/DTO 변환 책임이 Supabase 직접 호출 구조에서는 약해졌다는 점이다.
