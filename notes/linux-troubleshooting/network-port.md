# 포트와 TCP 연결 확인

포트와 TCP 연결 확인은 "애플리케이션이 요청을 받을 준비가 되어 있는지", "연결이 어떤 상태로 쌓였는지"를 보는 작업이다.

## ss

`ss`는 소켓과 TCP 연결 상태를 확인하는 명령어다.

> [!tip]
> 일단 이것부터 외워두면 좋다.
>
> ```bash
> ss -ltnp | grep ':8080'
> ss -ant | grep ':8080'
> ss -ant | grep ':8080' | awk '{print $1}' | sort | uniq -c
> ```

`LISTEN`, `ESTABLISHED`, `TIME_WAIT`, `CLOSE_WAIT` 같은 상태는 `ps`가 아니라 보통 `ss`로 본다.

```bash
ss -ltnp
ss -ant
ss -ant | grep ':8080'
```

자주 보는 옵션:

```text
-l: listening 상태만 보기
-t: TCP 보기
-n: 포트와 주소를 숫자로 보기
-p: 어떤 프로세스가 잡고 있는지 보기
-a: listening 포함 전체 연결 보기
```

예를 들어 8080 포트가 열려 있는지 보려면:

```bash
ss -ltnp | grep ':8080'
```

8080 포트에 붙은 연결 상태를 보려면:

```bash
ss -ant | grep ':8080'
```

상태별 개수를 대충 보려면:

```bash
ss -ant | grep ':8080' | awk '{print $1}' | sort | uniq -c
```

## lsof

`lsof`는 열린 파일을 보는 명령어다.

리눅스에서는 소켓도 파일처럼 다뤄지기 때문에, 특정 포트를 어떤 프로세스가 잡고 있는지 볼 때 유용하다.

```bash
lsof -i :8080
```

## 특정 포트 연결이 쌓였다는 뜻

"특정 포트 연결이 쌓였다"는 말은 어떤 서버 포트에 TCP 연결들이 많이 붙어 있고, 그 연결들이 특정 상태에서 빨리 정리되지 않고 남아 있다는 뜻이다.

> [!question]
> "연결 처리되고 반환 못하고 있다는 뜻인가?"
>
> `ESTABLISHED`나 `CLOSE_WAIT`가 많이 쌓인 경우에는 꽤 맞는 감각이다.
>
> 다만 상태마다 의미가 다르기 때문에, 연결 개수보다 어떤 TCP 상태로 쌓였는지를 먼저 봐야 한다.

예를 들어 서버가 8080 포트에서 HTTP 요청을 받고 있다고 하자.

```text
클라이언트들 -> 서버:8080
```

정상이라면 요청 처리 후 연결이 적당히 재사용되거나 닫힌다.

하지만 장애 상황에서는 다음처럼 연결이 많이 쌓일 수 있다.

```text
server:8080
- ESTABLISHED 500개
- TIME_WAIT 3000개
- CLOSE_WAIT 800개
- SYN_RECV 100개
```

여기서 중요한 것은 "연결이 많다" 자체보다 어떤 상태로 많이 쌓였는지다.

## TCP 연결 상태 해석

### ESTABLISHED가 많이 쌓임

`ESTABLISHED`는 TCP 연결이 정상적으로 맺어진 상태다.

> [!warning]
> `ESTABLISHED`가 비정상적으로 많으면 요청 처리가 오래 걸리거나, 뒤쪽 DB/API 응답을 기다리느라 클라이언트 응답을 못 주는 상황일 수 있다.

많이 쌓여 있다면 요청이나 응답 처리가 오래 걸리고 있을 수 있다.

```text
클라이언트 -> 서버: 요청
서버 -> DB: 조회
DB 응답 느림
서버가 클라이언트에게 응답을 못 줌
연결이 ESTABLISHED 상태로 오래 남음
```

가능한 원인:

```text
- 요청 처리 시간이 너무 김
- DB나 외부 API 응답이 느림
- 클라이언트가 연결을 오래 유지함
- WebSocket, SSE, 롱폴링처럼 장기 연결을 쓰고 있음
- 서버 스레드나 커넥션 풀이 막힘
```

### CLOSE_WAIT가 많이 쌓임

`CLOSE_WAIT`는 상대방은 연결을 닫겠다고 했는데, 내 애플리케이션이 아직 소켓을 닫지 않은 상태다.

> [!danger]
> `CLOSE_WAIT`가 많이 쌓이면 내 애플리케이션이 연결을 제대로 닫지 못하고 있을 가능성이 있다.
>
> response body, stream, socket, DB connection cleanup 누락을 의심한다.

```text
클라이언트: 나 연결 닫을게
서버 OS: 알겠음. 앱아, 너도 닫아
애플리케이션: ...
서버에 CLOSE_WAIT 누적
```

가능한 원인:

```text
- HTTP response body를 끝까지 안 읽거나 안 닫음
- DB connection, socket, stream을 제대로 close하지 않음
- finally/resource cleanup 누락
- 라이브러리 버그 또는 잘못된 사용
```

### TIME_WAIT가 많이 쌓임

`TIME_WAIT`는 연결이 닫힌 뒤 OS가 잠깐 보관하는 상태다.

연결 종료 후 늦게 도착할 수 있는 패킷을 처리하기 위해 일정 시간 남아 있는 정상 상태다.

하지만 너무 많으면 연결을 너무 자주 만들고 닫고 있다는 신호일 수 있다.

가능한 원인:

```text
- HTTP keep-alive를 못 씀
- 외부 API 호출마다 새 TCP 연결을 만듦
- DB connection pool 없이 매번 새 연결을 만듦
- 짧은 polling 요청이 너무 많음
```

### SYN_RECV가 많이 쌓임

`SYN_RECV`는 TCP 연결을 맺는 중간 단계다.

```text
클라이언트 -> 서버: SYN
서버 -> 클라이언트: SYN-ACK
클라이언트 -> 서버: ACK
```

`SYN_RECV`가 많이 쌓인다는 것은 연결 시도는 많은데 연결 수립이 끝나지 않는 상황일 수 있다.

가능한 원인:

```text
- 트래픽 폭증
- 네트워크 문제
- 서버가 연결 수립을 감당하지 못함
- SYN flood 같은 공격
```

## 포트 문제를 볼 때 질문 순서

포트 연결이 쌓였다는 말을 들으면 다음 순서로 보면 좋다.

```text
1. 어떤 포트인가?
2. 그 포트는 어떤 프로세스가 잡고 있는가?
3. 연결 상태는 LISTEN, ESTABLISHED, CLOSE_WAIT, TIME_WAIT 중 무엇이 많은가?
4. 언제부터 많아졌는가?
5. 앱 로그에는 에러가 있는가?
6. DB, Redis, 외부 API 같은 뒤쪽 의존성이 느린가?
7. 서버 CPU, 메모리, 디스크는 괜찮은가?
```

명령어 흐름으로 보면:

```bash
ss -ltnp | grep ':8080'
ss -ant | grep ':8080'
ss -ant | grep ':8080' | awk '{print $1}' | sort | uniq -c
ps -fp <PID>
top
```
