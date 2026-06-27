---
tags:
  - backend
  - spring
  - error-handling
  - exception
---

# ErrorCode와 BusinessException으로 에러 의미를 한 곳에서 정의하기

> [!summary]
> 한 줄 요약:
>
> - 에러 의미를 한 곳에서 정의한다는 것은, 실패를 문자열이나 DB 코드 그대로 흘려보내지 않고 우리 서비스가 이해하는 에러 이름으로 바꿔 관리한다는 뜻이다.
> - Spring에서는 보통 `ErrorCode enum`과 `BusinessException`으로 이 구조를 만든다.
> - React + Supabase 프로젝트의 `AUTH_ERROR`, `DATA_ERROR`, `AppError`, `classifyDataError`도 같은 문제를 해결하려는 구조다.

## 왜 공부했나

> [!question]
> 처음 헷갈렸던 질문:
>
> - 왜 그냥 `throw new RuntimeException("이미 존재합니다")` 하면 안 될까?
> - 왜 굳이 `ErrorCode.DUPLICATE_EMAIL` 같은 이름을 만들고, 그걸 예외 안에 넣을까?
> - 지금 프로젝트의 `AUTH_ERROR`, `DATA_ERROR`, `classifyDataError`를 Spring식으로 치면 무엇일까?

앞에서 DB 에러코드를 여러 곳에서 직접 분기하면 안 된다는 것을 봤다.

예를 들어 PostgreSQL의 `23505`, `42501` 같은 코드를 repository마다 직접 보면, 백엔드로 치면 컨트롤러나 서비스가 DB 벤더 코드를 직접 보는 느낌이 된다.

그럼 다음 질문은 자연스럽게 이것이다.

> DB 에러코드를 직접 들고 다니면 안 된다면, 무엇으로 바꿔서 들고 다녀야 할까?

그 답이 `ErrorCode`와 `BusinessException`이다.

## 문자열 예외의 문제

가장 단순하게는 이렇게 예외를 던질 수 있다.

```java
throw new RuntimeException("이미 존재하는 이메일입니다");
```

또는 TypeScript라면 이렇게 쓸 수 있다.

```ts
throw new Error("이미 등록돼 있어요");
```

처음에는 편하다.

하지만 실무에서는 금방 문제가 생긴다.

- HTTP status가 어디에 있는지 알기 어렵다.
- 사용자 메시지가 여러 파일에 흩어진다.
- 로그나 프론트 분기에서 사용할 안정적인 코드가 없다.
- 같은 에러인데 화면마다 문구가 달라질 수 있다.
- 프론트가 에러를 문자열로 비교하게 될 수 있다.
- 메시지를 바꾸면 로직이 깨질 수 있다.
- 다국어 처리나 메시지 정책 변경이 어렵다.

문자열은 사람에게 보여주기 위한 문장이지, 프로그램이 안정적으로 판단하기 좋은 식별자가 아니다.

> [!warning]
> 주의할 점:
>
> - `"이미 존재하는 이메일입니다"` 같은 메시지는 사용자에게 보여주는 용도다.
> - 프로그램이 에러 종류를 판단할 때는 메시지가 아니라 `DUPLICATE_EMAIL` 같은 안정적인 코드를 보는 편이 좋다.

## Spring의 ErrorCode enum

Spring에서는 보통 에러 의미를 enum으로 모은다.

```java
public enum ErrorCode {
    DUPLICATE_EMAIL(409, "이미 사용 중인 이메일입니다"),
    USER_NOT_FOUND(404, "사용자를 찾을 수 없습니다"),
    INVALID_PASSWORD(400, "비밀번호가 올바르지 않습니다"),
    FORBIDDEN(403, "권한이 없습니다");

    private final int status;
    private final String message;

    ErrorCode(int status, String message) {
        this.status = status;
        this.message = message;
    }

    public int getStatus() {
        return status;
    }

    public String getMessage() {
        return message;
    }
}
```

이렇게 하면 에러와 관련된 중요한 의미가 한 곳에 모인다.

- `DUPLICATE_EMAIL`: 프로그램이 판단할 에러 이름
- `409`: HTTP status
- `"이미 사용 중인 이메일입니다"`: 사용자에게 보여줄 메시지

즉, 에러가 단순 문자열이 아니라 구조화된 값이 된다.

## BusinessException

그 다음 예외에는 문자열을 직접 넣지 않고 `ErrorCode`를 넣는다.

```java
public class BusinessException extends RuntimeException {

    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() {
        return errorCode;
    }
}
```

서비스에서는 이렇게 사용한다.

```java
if (userRepository.existsByEmail(email)) {
    throw new BusinessException(ErrorCode.DUPLICATE_EMAIL);
}
```

이 코드는 문자열을 던지는 코드보다 의미가 뚜렷하다.

```text
나쁜 느낌:
"이미 존재하는 이메일입니다"라는 문장을 던진다.

좋은 느낌:
DUPLICATE_EMAIL이라는 비즈니스 에러가 발생했다고 선언한다.
```

서비스 코드가 사람 문장이 아니라 도메인 의미를 말하게 되는 것이다.

> [!info]
> 백엔드 관점에서 보면:
>
> - `ErrorCode`는 우리 서비스가 이해하는 실패 목록이다.
> - `BusinessException`은 그 실패를 예외 흐름에 태워 올리는 포장지다.
> - `@ControllerAdvice`는 나중에 이 `ErrorCode`를 읽어서 HTTP 응답으로 바꾼다.

## ErrorResponse로 변환되는 흐름

전역 핸들러에서는 `BusinessException` 안의 `ErrorCode`를 꺼내 응답을 만든다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(
            BusinessException e
    ) {
        ErrorCode errorCode = e.getErrorCode();

        return ResponseEntity.status(errorCode.getStatus())
                .body(new ErrorResponse(
                        errorCode.name(),
                        errorCode.getMessage()
                ));
    }
}
```

클라이언트가 받는 응답은 이런 식이 될 수 있다.

```json
{
  "code": "DUPLICATE_EMAIL",
  "message": "이미 사용 중인 이메일입니다"
}
```

여기서 `code`와 `message`는 역할이 다르다.

| 값 | 역할 |
| --- | --- |
| `code` | 개발자, 프론트, 로그, 분기에서 사용할 안정적인 식별자 |
| `message` | 사용자에게 보여줄 문장 |
| `status` | HTTP 의미 |
| `cause` | 실제 원인 추적용. 보통 사용자에게 직접 보여주지 않음 |

프론트는 메시지가 아니라 코드를 기준으로 안정적으로 분기할 수 있다.

```ts
if (error.code === "DUPLICATE_EMAIL") {
  showToast("이미 사용 중인 이메일입니다");
}
```

반대로 메시지로 분기하면 위험하다.

```ts
if (error.message === "이미 사용 중인 이메일입니다") {
  // 문구가 바뀌면 로직이 깨질 수 있음
}
```

## 현재 프로젝트와 연결

현재 프로젝트의 auth 쪽에는 이미 비슷한 구조가 있다.

```text
classifyAuthError
AUTH_ERROR
AppError
```

이 구조를 Spring으로 번역하면 이렇게 볼 수 있다.

```text
AUTH_ERROR = ErrorCode enum
AppError = BusinessException
classifyAuthError = 외부 auth 에러를 우리 서비스 에러로 번역하는 계층
```

대략 이런 모양이다.

```ts
export const AUTH_ERROR = {
  REQUIRED_LOGIN: {
    code: "AUTH_REQUIRED_LOGIN",
    message: "로그인이 필요해요",
    status: 401,
  },
  EXPIRED_SESSION: {
    code: "AUTH_EXPIRED_SESSION",
    message: "세션이 만료됐어요",
    status: 401,
  },
};
```

`classifyAuthError`는 Supabase Auth나 외부 라이브러리가 준 에러를 우리 서비스가 이해하는 에러로 바꾼다.

```text
Supabase Auth error
-> classifyAuthError
-> AUTH_ERROR.EXPIRED_SESSION
-> AppError
```

Spring으로 치면 이런 흐름이다.

```text
외부/저수준 예외
-> ErrorCode로 분류
-> BusinessException
-> GlobalExceptionHandler에서 응답
```

## data 쪽에 빠진 구조

문제는 data 쪽이다.

auth 쪽은 이미 외부 에러를 우리 서비스 언어로 바꾸는 구조가 있는데, data 쪽에는 아직 비슷한 계층이 없다.

그래서 여러 repository가 PostgreSQL 에러코드를 직접 분기한다.

```text
auth 에러:
외부 에러 -> 우리 서비스 에러 의미로 번역됨

data 에러:
PostgreSQL 에러코드 -> repository들이 직접 해석함
```

data 쪽에도 이런 구조가 필요하다.

```ts
export const DATA_ERROR = {
  DUPLICATE: {
    code: "DATA_DUPLICATE",
    message: "이미 등록된 값이에요",
    status: 409,
  },
  FORBIDDEN: {
    code: "DATA_FORBIDDEN",
    message: "권한이 없어요",
    status: 403,
  },
  NOT_FOUND: {
    code: "DATA_NOT_FOUND",
    message: "대상을 찾지 못했어요",
    status: 404,
  },
  INVALID_INPUT: {
    code: "DATA_INVALID_INPUT",
    message: "입력값이 올바르지 않아요",
    status: 400,
  },
};
```

그리고 PostgreSQL 에러코드는 한 곳에서 분류한다.

```ts
function classifyDataError(error: SupabaseError) {
  if (error.code === "23505") {
    return DATA_ERROR.DUPLICATE;
  }

  if (error.code === "42501") {
    return DATA_ERROR.FORBIDDEN;
  }

  if (error.code === "P0002") {
    return DATA_ERROR.NOT_FOUND;
  }

  if (error.code === "22023") {
    return DATA_ERROR.INVALID_INPUT;
  }

  return DATA_ERROR.UNKNOWN;
}
```

그러면 흐름이 이렇게 바뀐다.

```text
Supabase error.code = "23505"
-> classifyDataError(error)
-> DATA_ERROR.DUPLICATE
-> AppError
-> UI에서는 중복 에러로 표시
```

```text
Supabase error.code = "42501"
-> classifyDataError(error)
-> DATA_ERROR.FORBIDDEN
-> AppError
-> UI에서는 권한 없음으로 표시
```

핵심은 PostgreSQL 에러코드를 없애는 것이 아니다.

PostgreSQL 에러코드를 아는 위치를 한 곳으로 모으는 것이다.

## Spring과 현재 프로젝트 매핑

| 현재 프로젝트 | Spring식 표현 | 역할 |
| --- | --- | --- |
| `AUTH_ERROR` | `ErrorCode enum` | 인증 에러 의미 정의 |
| `DATA_ERROR` | `ErrorCode enum` | 데이터 에러 의미 정의 |
| `AppError` | `BusinessException` | 의미 있는 에러를 예외처럼 전달 |
| `classifyAuthError` | 외부 auth 예외 번역 계층 | Supabase Auth 에러를 서비스 에러로 변환 |
| `classifyDataError` | `SQLExceptionTranslator` 비슷한 계층 | PostgreSQL/Supabase 에러를 서비스 에러로 변환 |
| `error.code === "23505"` 복붙 | 컨트롤러/서비스에서 SQLState 직접 분기 | 피해야 할 구조 |

## 실무에서 보는 포인트

| 관점 | 확인할 것 | 왜 중요한가 |
| --- | --- | --- |
| 에러 정의 | 코드, 메시지, status가 한 곳에 있는가 | 에러 정책이 흩어지지 않는다 |
| 분기 기준 | 메시지가 아니라 code로 분기하는가 | 문구 변경이 로직 변경으로 번지지 않는다 |
| 계층 분리 | 외부/DB 에러코드를 어느 계층에서 아는가 | 저수준 정보가 UI나 서비스 전반에 퍼지는 것을 막는다 |
| 유지보수 | 같은 실패가 항상 같은 에러 코드로 바뀌는가 | 화면마다 다른 처리를 하는 문제를 줄인다 |
| 마이그레이션 | 코프링으로 가면 어느 부분이 흡수되는가 | `DATA_ERROR` 일부는 서버의 `ErrorCode`와 `BusinessException`으로 이동할 수 있다 |

## 헷갈리기 쉬운 부분

> [!warning]
> `ErrorCode`는 단순 메시지 모음이 아니다.
>
> - 메시지만 모아두는 상수가 아니다.
> - 프로그램이 에러를 식별할 수 있는 이름, HTTP status, 사용자 메시지를 함께 묶는 구조다.
> - 메시지는 바뀔 수 있지만 코드는 최대한 안정적으로 유지해야 한다.

> [!warning]
> 모든 예외를 `BusinessException`으로 만들 필요는 없다.
>
> - 사용자가 잘못 입력한 값, 권한 없음, 이미 존재함, 찾을 수 없음 같은 예상 가능한 실패는 비즈니스 에러로 다룰 수 있다.
> - NullPointerException, DB 연결 장애, 예상하지 못한 버그는 별도의 예상 밖 예외로 보고 로깅과 장애 대응 대상으로 다루는 편이 좋다.

> [!danger]
> 위험한 오해:
>
> - `throw new RuntimeException("메시지")`도 동작은 한다.
> - 하지만 동작한다고 해서 운영하기 좋은 구조는 아니다.
> - 에러를 문자열로만 흘려보내면 프론트 분기, 로그 분석, 메시지 변경, 다국어 처리, 공통 응답 형식 관리가 모두 어려워진다.

## 면접 답변으로 말하면

> [!tip]
> 짧게 답변:
>
> 저는 비즈니스 예외를 단순 문자열 예외로 던지기보다 `ErrorCode`와 커스텀 예외로 구조화하는 편이 좋다고 생각합니다. `ErrorCode`에 에러 식별자, HTTP status, 사용자 메시지를 모아두고, 서비스에서는 `BusinessException(ErrorCode.DUPLICATE_EMAIL)`처럼 의미 있는 예외를 던집니다. 그러면 전역 예외 핸들러에서 일관된 `ErrorResponse`로 변환할 수 있고, 프론트도 메시지가 아니라 안정적인 code를 기준으로 분기할 수 있습니다.

> [!note]-
> 긴 설명
>
> 단순히 `RuntimeException("이미 존재합니다")`처럼 문자열을 던지면 처음에는 편하지만, 에러 정책이 여러 계층에 흩어집니다. HTTP status는 어디서 정할지, 프론트는 어떤 기준으로 분기할지, 메시지를 바꿨을 때 로직이 깨지지 않을지 같은 문제가 생깁니다.
>
> 그래서 Spring에서는 `ErrorCode enum`을 두고 각 에러의 code, status, message를 한 곳에서 정의한 뒤, `BusinessException`이 이 `ErrorCode`를 들고 올라가게 만들 수 있습니다. 이후 `@ControllerAdvice`에서 `BusinessException`을 잡아 공통 `ErrorResponse`로 변환하면 API 에러 응답 형식도 통일됩니다.
>
> React + Supabase 구조에서도 같은 원리가 적용됩니다. Supabase나 PostgreSQL이 내려준 `23505`, `42501` 같은 저수준 에러를 여러 repository에서 직접 분기하지 않고, `classifyDataError`에서 `DATA_DUPLICATE`, `DATA_FORBIDDEN` 같은 서비스 에러로 바꾸면 유지보수성이 좋아집니다.

## 나중에 다시 볼 포인트

- 에러 메시지는 사용자에게 보여주는 문장이고, 에러 코드는 프로그램이 판단하는 식별자다.
- `ErrorCode`는 code, status, message를 한 곳에 모아 에러 정책을 관리한다.
- `BusinessException`은 `ErrorCode`를 예외 흐름에 실어 올리는 역할을 한다.
- `AUTH_ERROR`는 Spring의 `ErrorCode enum`과 비슷한 역할을 한다.
- `AppError`는 Spring의 `BusinessException`과 비슷한 역할을 한다.
- `classifyDataError`는 Supabase/PostgreSQL 에러를 우리 서비스 에러로 번역하는 계층이다.
- 메시지 문자열로 분기하지 말고 안정적인 code로 분기해야 한다.

## 복습 질문

- [ ] 왜 `throw new RuntimeException("이미 존재합니다")`보다 `throw new BusinessException(ErrorCode.DUPLICATE_EMAIL)`이 더 나은가?
- [ ] `ErrorCode` 안에는 어떤 정보들이 들어갈 수 있는가?
- [ ] 에러의 `code`, `message`, `status`는 각각 어떤 역할을 하는가?
- [ ] 현재 프로젝트의 `AUTH_ERROR`, `AppError`, `classifyAuthError`는 Spring으로 치면 각각 무엇과 비슷한가?
- [ ] 왜 프론트에서 `error.message`가 아니라 `error.code`로 분기하는 편이 좋은가?

## 한 줄 회고

- 헷갈렸던 점: 에러 메시지를 던지는 것과 에러 의미를 던지는 것은 다르다. `ErrorCode`와 `BusinessException`은 실패를 사람 문장이 아니라 서비스가 이해하는 안정적인 의미로 다루기 위한 구조다.
