---
tags:
  - test
  - smoke-test
  - spring
  - backend
---

# Smoke Test

## 한 줄 정의

Smoke test는 배포나 실행 직후에 서비스의 핵심 기능이 완전히 망가지지 않았는지 빠르게 확인하는 최소한의 테스트다.

> [!summary]
> Smoke test는 "이 빌드가 최소한 살아있는가?"를 빠르게 확인하는 테스트다.
> 깊은 검증보다 서버 실행, health check, 핵심 API 응답 여부를 먼저 본다.

## 왜 하는가

모든 기능을 자세히 검증하기 전에, 애플리케이션이 기본적으로 살아있는지 먼저 확인하기 위해 한다.

예를 들어 백엔드 서비스라면 다음 정도를 확인할 수 있다.

- 서버가 정상적으로 실행되는가?
- health check API가 200 응답을 주는가?
- 메인 API 한두 개가 기본 응답을 주는가?
- DB 연결이나 인증 같은 핵심 의존성이 완전히 깨지지 않았는가?

## 다른 테스트와의 차이

Smoke test는 깊게 검증하는 테스트가 아니다.

> [!note]
> Smoke test는 세부 비즈니스 규칙을 검증하는 테스트가 아니다.
> 세부 케이스는 통합 테스트나 E2E 테스트에서 다루고, smoke test는 큰 고장 여부만 빠르게 본다.

예를 들어 회원가입 기능을 테스트한다고 할 때, 비밀번호 정책, 이메일 중복, 인증 메일, 예외 케이스를 전부 확인하는 것은 smoke test보다 통합 테스트나 E2E 테스트에 가깝다.

Smoke test에서는 보통 "회원가입 API가 요청을 받고 기본적으로 응답하는가?"처럼 큰 고장 여부만 빠르게 본다.

## 실무 감각

Smoke test는 "이 빌드나 배포본을 더 자세히 검사해도 되는 상태인가?"를 판단하는 첫 관문에 가깝다.

배포 직후 smoke test가 실패하면, 세부 기능을 볼 필요 없이 서버 실행, 환경 변수, DB 연결, 라우팅, 인증 설정 같은 기본 문제부터 확인하면 된다.

## 예시

```bash
curl -f http://localhost:8080/health
```

위 요청이 성공하면 최소한 서버가 떠 있고 health check 엔드포인트가 정상 응답한다는 것을 알 수 있다.

## Spring에서는 어떻게 만들까

Spring Boot에서는 보통 두 가지 방식으로 smoke test를 만든다.

### 1. Actuator health check로 확인하기

운영이나 배포 직후 확인용 smoke test로는 Actuator의 health endpoint를 많이 쓴다.

> [!tip]
> Spring Boot에서는 `/actuator/health`를 smoke test의 첫 관문으로 두기 좋다.
> 앱 실행, HTTP 서버, 기본 의존성 상태를 빠르게 확인할 수 있다.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health
```

애플리케이션을 실행한 뒤 다음 요청을 보낸다.

```bash
curl -f http://localhost:8080/actuator/health
```

정상 응답 예시는 다음과 같다.

```json
{
  "status": "UP"
}
```

이 요청이 성공하면 최소한 다음을 확인할 수 있다.

- Spring 애플리케이션이 정상 실행되었다.
- HTTP 서버가 요청을 받을 수 있다.
- health check endpoint가 정상 응답한다.
- 설정에 따라 DB 같은 핵심 의존성 상태도 함께 확인할 수 있다.

### 2. 테스트 코드로 확인하기

CI나 로컬 테스트에서는 `@SpringBootTest`로 애플리케이션 컨텍스트가 정상적으로 뜨는지 확인할 수 있다.

> [!note]
> `contextLoads()`는 비어 있어도 의미가 있다.
> Spring 컨텍스트를 띄우는 과정에서 빈 설정이나 의존성 주입 문제가 있으면 실패한다.

```java
@SpringBootTest
class ApplicationSmokeTest {

    @Test
    void contextLoads() {
    }
}
```

테스트 내용이 비어 있어도 의미가 있다. `@SpringBootTest`가 Spring 애플리케이션 컨텍스트를 띄우기 때문에, 빈 설정, 의존성 주입, 환경 설정이 깨져 있으면 테스트가 실패한다.

조금 더 실전적으로는 랜덤 포트로 앱을 띄우고 health endpoint를 직접 호출할 수 있다.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class SmokeTest {

    @LocalServerPort
    int port;

    @Test
    void healthCheckReturnsOk() {
        RestTemplate restTemplate = new RestTemplate();

        ResponseEntity<String> response =
                restTemplate.getForEntity("http://localhost:" + port + "/actuator/health", String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).contains("UP");
    }
}
```

필요한 테스트 의존성은 보통 다음 starter에 포함되어 있다.

```gradle
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

## Spring 프로젝트에서의 최소 구성

Spring 백엔드 프로젝트라면 smoke test는 다음 정도로 시작하면 충분하다.

> [!check]
> 최소 구성은 `contextLoads`, `/actuator/health`, 핵심 read-only API 하나 정도면 충분하다.
> 인증이나 복잡한 예외 케이스까지 넣으면 smoke test가 무거워진다.

- `@SpringBootTest`로 애플리케이션 컨텍스트가 뜨는지 확인한다.
- `/actuator/health`가 200 응답과 `UP` 상태를 주는지 확인한다.
- 정말 중요한 read-only API 하나가 기본 응답을 주는지 확인한다.

인증, 예외 케이스, 복잡한 비즈니스 검증까지 smoke test에 넣으면 테스트가 무거워진다. smoke test는 "이 배포본이 최소한 살아있는가?"를 빠르게 확인하는 용도로 두는 것이 좋다.
