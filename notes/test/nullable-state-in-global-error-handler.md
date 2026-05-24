---
tags:
  - test
  - error-handling
  - nullable
  - global-error-handler
  - frontend
---

# 전역 에러 핸들러와 Nullable 상태 처리

## 한 줄 정의

전역 에러 핸들러처럼 앱 생명주기 초기에 붙는 코드는 아직 준비되지 않았을 수 있는 전역 상태를 nullable하게 읽어야 한다.

> [!summary]
> 전역 에러 핸들러에서는 사용자 세션, 인증 정보, request context 같은 값이 없을 수 있다고 봐야 한다.
> 로깅용 부가 정보는 nullable하게 읽고, 없으면 생략한다.

## 왜 헷갈렸나

전역 에러 핸들러는 보통 앱 시작 시점에 설치된다.

이때 핸들러 내부에서 사용자 세션, 인증 정보, request context 같은 전역 상태를 참조하면 다음 질문이 생긴다.

- 에러 핸들러가 너무 일찍 설치되는 것은 아닌가?
- 세션이나 인증 상태가 아직 없으면 터지는 것은 아닌가?
- 에러를 처리하려다가 에러 핸들러 자체가 또 에러를 내는 것은 아닌가?

핵심은 "그 값을 반드시 있다고 가정하는가?"이다.

> [!question]
> 리뷰 포인트는 "에러 핸들러가 너무 일찍 실행될 때 아직 준비 안 된 상태를 강하게 믿고 있지 않은가?"이다.

## 예시

프론트 코드에서 다음과 같은 함수가 있다고 하자.

```ts
function currentUserId(): string | undefined {
  return useStore.getState().auth.session?.user.id
}
```

이 코드는 `auth.session`이 없을 때 에러를 던지지 않는다.

`?.` 때문에 세션이 없으면 그냥 `undefined`가 된다. 즉, 로그에 `userId`가 붙지 않을 뿐 에러 처리 흐름 자체가 깨지지는 않는다.

## 안전한 이유

이 구조가 괜찮은 이유는 다음과 같다.

> [!tip]
> `session?.user.id`처럼 optional하게 읽으면 세션이 없을 때 에러가 아니라 `undefined`가 된다.
> 이 경우 로그에 `userId`가 빠질 뿐, 에러 처리 흐름은 계속된다.

- `useStore` 자체는 import 시점에 이미 만들어진다.
- `currentUserId()`는 interceptor 설치 시점에 바로 실행되는 것이 아니라, 실제 에러가 발생했을 때 실행된다.
- `auth.session`은 optional하게 처리된다.
- 세션이 없으면 `undefined`가 반환될 뿐이다.

즉 "세션은 반드시 있어야 한다"가 아니라 "세션이 있으면 userId를 붙이고, 없으면 생략한다"는 구조다.

## Spring으로 비유

Java/Spring으로 보면 다음 코드와 비슷하다.

```java
String userId = Optional.ofNullable(session)
        .map(Session::getUserId)
        .orElse(null);
```

Spring Security를 쓴다면 다음처럼 볼 수도 있다.

```java
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

String userId = authentication != null ? authentication.getName() : null;
```

여기서 `userId`는 핵심 비즈니스 로직을 수행하기 위한 필수 값이 아니라, 에러 로그에 붙이는 부가 정보다.

그래서 없을 수 있다고 보고 nullable하게 처리하는 것이 자연스럽다.

## 기억할 기준

값의 성격에 따라 다르게 처리해야 한다.

> [!important]
> 필수 비즈니스 값이면 없을 때 실패시키고, 로깅/추적용 부가 정보면 nullable하게 처리한다.

- 필수 로직에 필요한 값이면 없을 때 명확히 실패시킨다.
- 로깅, 추적, 디버깅용 부가 정보면 없을 수 있다고 보고 nullable하게 처리한다.

이번 케이스의 `userId`는 에러 로그에 붙이는 부가 정보다.

따라서 세션이 없다고 에러 핸들러가 실패하면 안 되고, userId 없이 로그를 남기는 쪽이 맞다.

## 실무 포인트

전역 interceptor, error handler, filter, `@ControllerAdvice`, 로깅 AOP 같은 코드는 앱 생명주기에서 이른 시점이나 예외 상황에서 실행될 수 있다.

> [!warning]
> 전역 에러 처리 코드가 2차 장애를 만들면 원래 에러보다 디버깅이 더 어려워진다.
> 그래서 부가 정보 조회는 실패해도 본래 에러 처리 흐름을 망치지 않게 작성한다.

이런 곳에서는 아직 준비되지 않았을 수 있는 상태를 강하게 가정하면 2차 장애가 날 수 있다.

특히 다음 값들은 항상 있다고 가정하지 않는 것이 좋다.

- session
- authentication
- request context
- tenant context
- trace id
- current user

핵심은 다음 문장으로 정리할 수 있다.

```text
전역 에러 처리 코드에서는 부가 정보가 없을 수 있다고 보고 nullable하게 읽는다.
없으면 생략하고, 본래 에러 처리 흐름은 계속 진행되게 만든다.
```
