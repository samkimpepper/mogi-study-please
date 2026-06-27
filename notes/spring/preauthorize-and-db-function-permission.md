---
tags:
  - backend
  - spring
  - spring-security
  - postgres
  - security
  - supabase
---

# PreAuthorize와 DB 함수 권한의 이중 방어

> [!summary]
> 한 줄 요약:
>
> - `@PreAuthorize`는 Spring Security에서 메서드 실행 전에 권한 조건을 검사하는 어노테이션이다.
> - PostgreSQL 함수 권한 문제는 Spring으로 치면 URL이나 RPC 입구는 열려 있고, 메서드 내부의 `@PreAuthorize`나 `if` 체크에만 의존하는 구조와 비슷하다.
> - 좋은 보안 구조는 기본 입구를 닫고, 필요한 역할만 열고, 내부에서도 한 번 더 검증하는 이중 방어다.

## 왜 공부했나

> [!question]
> 처음 헷갈렸던 질문:
>
> - Spring Security에서 메서드 실행 전에 권한을 검사하는 어노테이션 이름이 뭐였나?
> - PostgreSQL의 `PUBLIC EXECUTE` 문제를 Spring Security로 비유하면 어떤 느낌인가?
> - `permitAll`과 내부 권한 체크만으로 충분하지 않은 이유는 무엇인가?

Spring Security에서 메서드 실행 전에 권한을 검사하는 대표 어노테이션은 `@PreAuthorize`다.

이 이름의 `Pre`는 메서드 실행 전을 뜻한다.

## PreAuthorize 기본 느낌

`@PreAuthorize`는 메서드가 실행되기 전에 현재 인증 정보와 권한을 보고 실행 가능 여부를 판단한다.

```java
@PreAuthorize("hasRole('ADMIN')")
public void adminAction() {
    // admin만 실행 가능
}
```

로그인한 사용자만 허용하려면 이렇게 쓸 수 있다.

```java
@PreAuthorize("isAuthenticated()")
public void createPost() {
    // 로그인한 사용자만 실행 가능
}
```

파라미터를 이용해서 소유자 체크를 할 수도 있다.

```java
@PreAuthorize("#userId == authentication.principal.id")
public void updateProfile(Long userId) {
    // 본인 프로필만 수정 가능
}
```

짝꿍으로 메서드 실행 후 결과를 보고 검사하는 `@PostAuthorize`도 있다.

```java
@PostAuthorize("returnObject.ownerId == authentication.principal.id")
public Post getPost(Long postId) {
    return postRepository.findById(postId)
            .orElseThrow(() -> new BusinessException(ErrorCode.POST_NOT_FOUND));
}
```

하지만 실무에서 더 자주 떠올리는 것은 보통 실행 전 검사인 `@PreAuthorize`다.

## permitAll + 내부 체크만 믿는 구조

PostgreSQL 함수의 `PUBLIC EXECUTE` 문제를 Spring으로 비유하면, 감각적으로는 이런 구조에 가깝다.

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/rpc/**").permitAll()
);
```

입구는 모두에게 열어둔다.

그리고 각 메서드 내부에서 직접 막는다.

```java
@PostMapping("/rpc/admin-action")
public void adminAction() {
    if (!currentUser.isAdmin()) {
        throw new BusinessException(ErrorCode.FORBIDDEN);
    }

    // admin 작업
}
```

이 구조가 아예 보안이 없는 것은 아니다.

내부 `if` 체크가 정확히 들어가 있으면 막힌다.

하지만 문제는 매번 사람이 기억해야 한다는 점이다.

```text
새 admin action 추가
-> /rpc/** permitAll 아래에 들어감
-> 개발자가 if (!isAdmin) 체크를 깜빡함
-> 바로 열림
```

PostgreSQL 함수 권한도 비슷하다.

```text
CREATE FUNCTION
-> 기본적으로 PUBLIC EXECUTE 열림
-> 함수 내부에서 is_admin()으로 막음
```

즉 이런 상태다.

```text
입구는 열려 있음
방 안에서 경비원이 검사함
```

내부 경비원이 있으면 막히지만, 새 방을 만들 때 경비원 배치를 깜빡하면 바로 열린다.

## 더 좋은 구조: 기본 닫힘 + 내부 검증

Spring Security에서 더 좋은 감각은 기본을 닫고 필요한 것만 여는 것이다.

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/user/**").authenticated()
    .anyRequest().denyAll()
);
```

그리고 중요한 서비스 메서드에서도 한 번 더 검증할 수 있다.

```java
@PreAuthorize("hasRole('ADMIN')")
public void adminAction() {
    // admin 작업
}
```

또는 명시적인 도메인 검증을 둘 수도 있다.

```java
public void adminAction(CurrentUser currentUser) {
    if (!currentUser.isAdmin()) {
        throw new BusinessException(ErrorCode.FORBIDDEN);
    }

    // admin 작업
}
```

DB 함수로 치면 입구 권한부터 닫는다.

```sql
REVOKE EXECUTE ON FUNCTION admin_action(...) FROM PUBLIC, anon;
GRANT EXECUTE ON FUNCTION admin_action(...) TO authenticated;
```

그리고 함수 내부에서도 한 번 더 검증한다.

```sql
IF NOT is_admin() THEN
  RAISE EXCEPTION 'forbidden' USING ERRCODE = '42501';
END IF;
```

즉 좋은 구조는 이중 방어다.

```text
1. 입구 권한:
PUBLIC/anon 실행 권한을 닫고 필요한 role만 연다.

2. 내부 검증:
함수 내부에서도 is_admin(), owner check 같은 도메인 검증을 한다.
```

## Spring과 PostgreSQL 권한 매핑

| Spring Security | PostgreSQL/Supabase | 의미 |
| --- | --- | --- |
| `permitAll()` | `PUBLIC EXECUTE` 열림 | 입구가 모두에게 열림 |
| `authenticated()` | `GRANT EXECUTE TO authenticated` | 인증 사용자에게 허용 |
| `denyAll()` | `REVOKE EXECUTE FROM PUBLIC, anon` | 기본 입구 닫기 |
| `@PreAuthorize("hasRole('ADMIN')")` | 함수 내부 `is_admin()` 가드 | 실행 전/실행 중 권한 검증 |
| `BusinessException(FORBIDDEN)` | `RAISE EXCEPTION ... ERRCODE=42501` | 권한 없음 의미 전달 |

중요한 차이는 `GRANT TO authenticated`가 `PUBLIC` 권한을 자동으로 닫아주지 않는다는 점이다.

```text
GRANT TO authenticated
!=
anon 차단
```

따라서 PostgreSQL 함수는 보안 경계를 세울 때 다음 두 줄을 세트처럼 봐야 한다.

```sql
REVOKE EXECUTE ON FUNCTION my_write_rpc(...) FROM PUBLIC, anon;
GRANT EXECUTE ON FUNCTION my_write_rpc(...) TO authenticated;
```

## 실무에서 보는 포인트

| 관점 | 확인할 것 | 왜 중요한가 |
| --- | --- | --- |
| 기본 정책 | 기본이 deny인지 allow인지 | 기본 open은 새 코드가 실수로 열릴 위험이 크다 |
| 입구 권한 | URL/RPC/function 실행 권한이 닫혀 있는지 | 내부 체크 전에 아예 접근면을 줄인다 |
| 내부 검증 | admin/owner/auth 검증이 있는지 | 입구 권한이 뚫리거나 넓어도 한 번 더 막는다 |
| 자동화 | 새 함수 생성 시 REVOKE가 강제되는지 | 사람이 매번 기억하는 방식은 언젠가 샌다 |
| 테스트 | anon이 write RPC를 실행할 수 없는지 검사하는지 | 보안 경계를 회귀 테스트로 잡을 수 있다 |

## 헷갈리기 쉬운 부분

> [!warning]
> `GRANT TO authenticated`는 `anon` 차단이 아니다.
>
> - PostgreSQL에서 `PUBLIC`에 남아 있는 실행 권한은 별도로 닫아야 한다.
> - 필요한 역할에게 권한을 주는 것과 기본 공개 권한을 회수하는 것은 다른 작업이다.

> [!warning]
> 내부 체크가 있으니 괜찮다는 말은 단일 방어층이다.
>
> - 함수 내부의 `is_admin()` 체크는 중요하다.
> - 하지만 함수 실행 권한이 열려 있으면 모든 보안이 작성자의 가드 누락 여부에 의존하게 된다.
> - 좋은 구조는 입구 권한과 내부 검증이 모두 있는 것이다.

> [!danger]
> 위험한 오해:
>
> - `permitAll`을 해놓고 메서드마다 `if (!isAdmin)`을 쓰면 당장은 막힐 수 있다.
> - 하지만 새 메서드에서 체크를 빠뜨리는 순간 바로 열린다.
> - 보안은 "매번 기억하기"보다 "기본적으로 닫히게 만들기"가 더 안전하다.

## 면접 답변으로 말하면

> [!tip]
> 짧게 답변:
>
> Spring Security에서는 `@PreAuthorize`로 메서드 실행 전에 권한을 검사할 수 있습니다. 다만 좋은 보안 구조는 메서드 내부 검증만 믿는 것이 아니라, SecurityConfig에서 기본 접근 정책을 먼저 닫고 필요한 경로만 열어둔 뒤, 중요한 서비스 메서드에서도 `@PreAuthorize`나 도메인 검증으로 한 번 더 확인하는 것입니다. PostgreSQL 함수 권한도 비슷하게, `PUBLIC EXECUTE`를 명시적으로 `REVOKE`하고 필요한 role에만 `GRANT`한 뒤 함수 내부에서도 `is_admin()` 같은 검증을 두는 이중 방어가 좋습니다.

## 나중에 다시 볼 포인트

- `@PreAuthorize`는 메서드 실행 전 권한 조건을 검사한다.
- `@PostAuthorize`는 메서드 실행 후 반환값을 보고 권한을 검사한다.
- `permitAll`과 내부 if 검증만으로는 새 코드에서 가드를 빠뜨릴 위험이 있다.
- PostgreSQL 함수는 `CREATE FUNCTION` 후 `PUBLIC EXECUTE` 기본 권한을 의식해야 한다.
- `GRANT TO authenticated`는 `PUBLIC` 권한을 제거하지 않는다.
- 좋은 보안은 기본 닫힘 + 필요한 권한만 열기 + 내부 검증의 조합이다.

## 복습 질문

- [ ] `@PreAuthorize`는 언제 권한을 검사하는가?
- [ ] `@PreAuthorize`와 `@PostAuthorize`의 차이는 무엇인가?
- [ ] `permitAll` 후 내부 `if (!isAdmin)`만 믿는 구조가 왜 위험한가?
- [ ] PostgreSQL에서 `GRANT EXECUTE TO authenticated`만으로 `anon`이 차단되지 않는 이유는 무엇인가?
- [ ] DB 함수 권한에서 이중 방어는 어떤 두 층으로 구성되는가?

## 한 줄 회고

- 헷갈렸던 점: `@PreAuthorize` 같은 내부 권한 검증도 중요하지만, 보안의 기본은 입구를 먼저 닫고 필요한 곳만 여는 것이다.
