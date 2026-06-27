---
tags:
  - backend
  - spring
  - jpa
  - error-handling
  - transaction
  - concurrency
---

# 비즈니스 검증, BusinessException, 그리고 DB 최종 방어선

> [!summary]
> 한 줄 요약:
>
> - Spring 서비스 코드에서 예상 가능한 비즈니스 실패는 `try-catch`로 흐름 전체를 감싸기보다, `if`로 명시적으로 검증하고 의미 있는 `BusinessException`을 던지는 편이 자연스럽다.
> - 하지만 중복 이메일 같은 데이터 무결성은 `if` 검증만으로는 동시성 상황을 완전히 막을 수 없으므로 DB unique constraint가 최종 방어선이 되어야 한다.
> - 따라서 실무에서는 `if 검증`, `DB 제약조건`, `DataIntegrityViolationException` 번역이 함께 필요하다.

## 왜 공부했나

> [!question]
> 처음 헷갈렸던 질문:
>
> - Spring 백엔드에서 비즈니스 실패를 처리할 때 `try-catch`를 많이 쓰는 게 맞을까?
> - `if`로 값을 검증하고 `throw new BusinessException(...)` 하는 방식은 괜찮은 방식일까?
> - 중복 이메일처럼 미리 검사할 수 있는 값은 DB 에러 처리 없이 `existsByEmail`만 믿어도 될까?

Spring 백엔드를 구현할 때 보통 서비스 메서드 전체를 `try-catch`로 감싸기보다, 조건을 명시적으로 확인하고 의미 있는 비즈니스 예외를 던지는 흐름을 자주 쓴다.

이 감각은 대체로 맞다.

서비스 코드는 "실패했다"가 아니라 "무엇 때문에 실패했는지"를 말해야 한다.

## 좋은 흐름: if 검증 후 BusinessException

예를 들어 닉네임 변경 로직은 이렇게 쓸 수 있다.

```java
@Transactional
public void changeNickname(Long userId, String nickname) {
    User user = userRepository.findById(userId)
            .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));

    if (userRepository.existsByNickname(nickname)) {
        throw new BusinessException(ErrorCode.DUPLICATE_NICKNAME);
    }

    user.changeNickname(nickname);
}
```

이 코드는 흐름이 분명하다.

```text
유저 없으면 USER_NOT_FOUND
닉네임 중복이면 DUPLICATE_NICKNAME
문제 없으면 변경
```

서비스 코드가 실패 의미를 직접 드러낸다.

이런 코드는 전역 핸들러와도 잘 맞는다.

```text
BusinessException(ErrorCode.USER_NOT_FOUND)
-> @ControllerAdvice
-> 404 ErrorResponse

BusinessException(ErrorCode.DUPLICATE_NICKNAME)
-> @ControllerAdvice
-> 409 ErrorResponse
```

> [!info]
> 백엔드 관점에서 보면:
>
> - 예상 가능한 비즈니스 실패는 조건을 명시적으로 검사한다.
> - 실패 이유에 맞는 `ErrorCode`를 선택한다.
> - 응답 변환은 서비스가 아니라 전역 예외 핸들러가 맡는다.

## 나쁜 흐름: 모든 실패를 try-catch로 뭉개기

반대로 이런 코드는 좋지 않다.

```java
public void changeNickname(Long userId, String nickname) {
    try {
        User user = userRepository.findById(userId).get();

        if (userRepository.existsByNickname(nickname)) {
            throw new RuntimeException("중복 닉네임입니다");
        }

        user.changeNickname(nickname);

    } catch (Exception e) {
        throw new BusinessException(ErrorCode.CHANGE_NICKNAME_FAILED);
    }
}
```

문제는 여러 실패가 하나로 뭉개진다는 점이다.

- 유저가 없는 실패
- 닉네임이 중복된 실패
- DB 장애
- 예상하지 못한 버그

이 모든 것이 `CHANGE_NICKNAME_FAILED` 하나가 되어버린다.

그러면 전역 핸들러도 정확한 응답을 만들기 어렵다.

```text
USER_NOT_FOUND여야 할 실패가 CHANGE_NICKNAME_FAILED가 됨
DUPLICATE_NICKNAME이어야 할 실패가 CHANGE_NICKNAME_FAILED가 됨
DB 장애와 사용자 입력 문제가 같은 에러로 보임
```

> [!danger]
> 위험한 오해:
>
> - `catch (Exception e)`로 감싸서 하나의 비즈니스 예외로 바꾸면 깔끔해 보일 수 있다.
> - 하지만 실제로는 실패 의미를 잃어버리는 경우가 많다.
> - 예외를 잡을 때는 "복구할 수 있는가" 또는 "더 의미 있는 예외로 정확히 번역할 수 있는가"가 분명해야 한다.

## 단, if 검증만 믿으면 안 되는 경우

`if` 검증 후 `BusinessException`을 던지는 방식은 좋다.

하지만 모든 문제를 `if`로 미리 막을 수는 없다.

대표적인 예가 unique constraint다.

```java
if (userRepository.existsByEmail(email)) {
    throw new BusinessException(ErrorCode.DUPLICATE_EMAIL);
}

userRepository.save(new User(email));
```

단일 요청만 보면 괜찮아 보인다.

하지만 동시에 같은 이메일로 회원가입 요청이 두 개 들어오면 문제가 생길 수 있다.

```text
요청 A: existsByEmail(email) -> false
요청 B: existsByEmail(email) -> false

요청 A: insert 성공
요청 B: insert 시도
요청 B: DB unique constraint 위반
```

두 요청 모두 `existsByEmail` 검사 시점에는 아직 데이터가 없다고 볼 수 있다.

그래서 둘 다 검증을 통과한다.

하지만 실제 insert 시점에는 하나만 성공하고, 나머지는 DB unique constraint에 막힌다.

이것이 DB 제약조건이 필요한 이유다.

> [!warning]
> 주의할 점:
>
> - `existsByEmail` 같은 사전 검증은 사용자에게 빠르고 명확한 에러를 주는 데 도움이 된다.
> - 하지만 동시성 상황에서 데이터 무결성을 최종 보장하지는 못한다.
> - 최종 무결성은 DB unique constraint가 보장해야 한다.

## 실무에서 필요한 3겹

중복 이메일 같은 문제는 보통 세 겹으로 본다.

```text
1. if 검증
사용자에게 빠르고 명확한 비즈니스 에러를 주기 위함

2. DB 제약조건
동시성 상황에서도 데이터 무결성을 최종 보장하기 위함

3. DataIntegrityViolationException 처리
if 검증을 통과했지만 DB에서 최종적으로 막힌 실패를 의미 있는 에러로 바꾸기 위함
```

코드로 쓰면 이런 느낌이다.

```java
@Transactional
public void createUser(CreateUserRequest request) {
    if (userRepository.existsByEmail(request.email())) {
        throw new BusinessException(ErrorCode.DUPLICATE_EMAIL);
    }

    try {
        userRepository.save(new User(request.email()));
    } catch (DataIntegrityViolationException e) {
        throw new BusinessException(ErrorCode.DUPLICATE_EMAIL);
    }
}
```

여기서 `try-catch`가 아예 나쁜 것은 아니다.

중요한 것은 어떤 목적으로 잡느냐다.

이 코드는 서비스 흐름 전체를 대충 실패 처리로 감싸는 `try-catch`가 아니다.

DB race condition 때문에 마지막에 터질 수 있는 특정 예외를, 의미 있는 비즈니스 에러로 번역하는 `try-catch`다.

## 더 깔끔한 방향

서비스마다 `DataIntegrityViolationException`을 잡기 시작하면 다시 중복이 생길 수 있다.

그래서 더 깔끔하게는 이 변환을 공통 핸들러나 별도 예외 변환 계층으로 빼기도 한다.

```text
DB unique constraint 위반
-> DataIntegrityViolationException
-> GlobalExceptionHandler 또는 Translator
-> ErrorCode.DUPLICATE_EMAIL
-> ErrorResponse
```

다만 이때 주의할 점도 있다.

모든 `DataIntegrityViolationException`이 이메일 중복은 아니다.

어떤 제약조건이 깨졌는지 확인해야 정확히 `DUPLICATE_EMAIL`, `DUPLICATE_NICKNAME`, `INVALID_RELATION`처럼 매핑할 수 있다.

프로젝트 규모가 작다면 서비스에서 특정 케이스만 잡는 편이 더 명확할 수 있고, 규모가 커지면 공통 변환 계층이 필요해질 수 있다.

## Supabase 프로젝트와 연결

React + Supabase 프로젝트에서도 같은 감각이 적용된다.

사전 검증은 UI나 클라이언트에서 할 수 있다.

```text
필수 입력값이 비어 있음
-> inline validation
```

하지만 최종 무결성은 DB가 막아야 한다.

```text
동시에 같은 데이터 insert
-> Postgres unique constraint
-> Supabase error.code = "23505"
-> classifyDataError
-> DATA_ERROR.DUPLICATE
```

Spring으로 치면 이 흐름이다.

```text
DataIntegrityViolationException
-> ErrorCode.DUPLICATE
-> BusinessException 또는 ErrorResponse
```

즉, `classifyDataError`는 단순히 "에러 메시지를 예쁘게 바꾸는 함수"가 아니다.

DB가 최종 방어선으로 막은 실패를, 애플리케이션이 이해하는 실패 의미로 번역하는 계층이다.

## 실무에서 보는 포인트

| 관점 | 확인할 것 | 왜 중요한가 |
| --- | --- | --- |
| 서비스 로직 | 예상 가능한 실패를 명시적으로 검사하는가 | 코드만 읽어도 비즈니스 규칙이 보인다 |
| 예외 처리 | `catch (Exception)`으로 실패 의미를 뭉개지 않는가 | 원인별 응답과 로그 분석이 가능해야 한다 |
| 데이터 무결성 | DB 제약조건이 최종 방어선으로 있는가 | 동시성 상황에서 사전 검증만으로는 부족하다 |
| 에러 변환 | DB 제약조건 실패를 의미 있는 에러로 번역하는가 | 사용자에게 정확한 409, 400, 403 등을 줄 수 있다 |
| 중복 제거 | 같은 예외 변환이 여러 서비스에 반복되는가 | 반복되면 공통 translator나 전역 핸들러를 고려한다 |

## 헷갈리기 쉬운 부분

> [!warning]
> `try-catch`를 쓰지 말라는 뜻이 아니다.
>
> - 비즈니스 흐름 전체를 감싸서 모든 예외를 하나로 뭉개는 `try-catch`가 문제다.
> - 특정 저수준 예외를 의미 있는 비즈니스 예외로 정확히 번역하는 `try-catch`는 필요할 수 있다.

> [!warning]
> `if` 검증과 DB 제약조건은 경쟁 관계가 아니다.
>
> - `if` 검증은 사용자 경험과 명확한 에러 메시지를 위해 필요하다.
> - DB 제약조건은 데이터 무결성을 최종 보장하기 위해 필요하다.
> - 둘 중 하나만 고르는 문제가 아니라 역할이 다르다.

> [!danger]
> 위험한 오해:
>
> - `existsByEmail`을 했으니 unique constraint는 없어도 된다고 생각하면 안 된다.
> - 동시 요청에서는 사전 검증을 둘 다 통과할 수 있다.
> - 중복을 절대 허용하면 안 되는 값은 반드시 DB 제약조건으로 막아야 한다.

## 면접 답변으로 말하면

> [!tip]
> 짧게 답변:
>
> 예상 가능한 비즈니스 실패는 서비스에서 `if`로 명시적으로 검증하고, `BusinessException(ErrorCode...)`처럼 의미 있는 예외를 던지는 편이 좋다고 생각합니다. 다만 중복 이메일처럼 데이터 무결성과 관련된 규칙은 사전 검증만으로는 동시성 상황을 완전히 막을 수 없기 때문에 DB unique constraint가 최종 방어선이 되어야 합니다. 그리고 DB에서 최종적으로 막힌 `DataIntegrityViolationException` 같은 예외도 전역 핸들러나 변환 계층에서 다시 의미 있는 에러로 바꿔줘야 합니다.

> [!note]-
> 긴 설명
>
> 서비스 메서드 전체를 `try-catch`로 감싸서 모든 실패를 `CHANGE_FAILED` 같은 하나의 예외로 바꾸면, 유저 없음, 중복, DB 장애 같은 원인이 모두 뭉개집니다. 그래서 예상 가능한 비즈니스 실패는 조건을 명시적으로 검사하고 그에 맞는 `ErrorCode`를 던지는 편이 좋습니다.
>
> 하지만 `existsByEmail` 같은 사전 검증은 동시성 상황에서 최종 보장이 될 수 없습니다. 두 요청이 동시에 들어오면 둘 다 존재 여부 검사에서는 false를 보고, 이후 하나만 insert에 성공하고 나머지는 unique constraint에 걸릴 수 있습니다. 따라서 DB 제약조건이 최종 무결성을 보장해야 하고, 이때 발생한 `DataIntegrityViolationException`도 의미 있는 비즈니스 에러로 번역해야 합니다.

## 나중에 다시 볼 포인트

- 예상 가능한 비즈니스 실패는 `if` 검증 후 `BusinessException`으로 명확히 표현한다.
- 서비스 메서드 전체를 `catch (Exception)`으로 감싸면 실패 의미가 뭉개질 수 있다.
- `try-catch` 자체가 나쁜 것이 아니라, 무엇을 어떤 의미로 번역하는지가 중요하다.
- `existsByEmail` 같은 사전 검증은 동시성 상황에서 최종 보장이 아니다.
- unique constraint 같은 DB 제약조건은 데이터 무결성의 최종 방어선이다.
- DB 제약조건 위반은 다시 `ErrorCode`나 `BusinessException`으로 번역되어야 사용자에게 의미 있게 전달된다.

## 복습 질문

- [ ] 서비스 코드에서 예상 가능한 비즈니스 실패를 `if`로 검증하고 `BusinessException`을 던지는 방식이 왜 자연스러운가?
- [ ] `catch (Exception)`으로 모든 실패를 하나의 `BusinessException`으로 바꾸면 어떤 문제가 생기는가?
- [ ] `existsByEmail` 검증을 했는데도 unique constraint가 필요한 이유는 무엇인가?
- [ ] `DataIntegrityViolationException`을 잡는 `try-catch`는 어떤 경우에 의미가 있는가?
- [ ] `if 검증`, `DB 제약조건`, `예외 변환`은 각각 어떤 역할을 하는가?

## 한 줄 회고

- 헷갈렸던 점: `try-catch`를 안 쓰는 것이 목표가 아니라, 예상 가능한 비즈니스 실패는 명시적으로 던지고 DB가 최종적으로 막은 실패는 정확히 번역하는 것이 핵심이다.
