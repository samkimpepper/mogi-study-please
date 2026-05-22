# 리눅스 트러블슈팅 오답노트

문제를 풀다가 헷갈린 부분을 다시 보기 좋게 정리한다.

## top 문제 1: CPU가 바쁜 서버 해석

### 문제 상황

서버는 2코어이고, API 서버가 갑자기 느려졌다.

```text
load average: 4.80, 4.20, 3.90
%Cpu(s): 92.0 us,  5.0 sy,  0.0 ni,  1.0 id,  2.0 wa

PID   USER   PR  NI  VIRT   RES   SHR  S  %CPU  %MEM  TIME+   COMMAND
1234  app    20   0  2.1g   900m  80m   R  180.0 22.0  10:20   java
2211  mysql  20   0  1.0g   400m  40m   S   15.0  9.8   2:11   mysqld
3010  root   20   0  100m    30m  10m   S    2.0  0.7   0:20   sshd
```

### 내가 맞게 본 것

CPU가 바쁜 상태라고 본 것은 맞다.

근거:

```text
us 92.0:
사용자 프로세스가 CPU를 거의 다 쓰는 중

id 1.0:
CPU가 노는 시간이 거의 없음

wa 2.0:
디스크나 네트워크 I/O 대기보다는 CPU 사용 문제가 더 의심됨
```

따라서 이 상황은 "디스크를 기다리느라 느림"보다는 "애플리케이션 같은 사용자 프로세스가 CPU를 많이 쓰는 상황"에 가깝다.

### 헷갈린 것 1: load average 해석

`load average: 4.80`은 단독으로 보면 감이 잘 안 온다.

반드시 CPU 코어 수와 비교해야 한다.

```text
2코어 서버에서 load average 2.0:
CPU가 거의 꽉 찬 수준

2코어 서버에서 load average 4.8:
처리 가능한 양보다 일이 더 많이 쌓인 상태
```

비유:

```text
계산대 2개가 있는데
계속 4~5명 정도가 줄 서 있는 상태
```

기억할 것:

```text
load average <= CPU 코어 수
-> 대체로 감당 가능

load average > CPU 코어 수
-> 처리 대기 작업이 쌓이는 중
```

### 헷갈린 것 2: TIME+보다 %CPU가 더 중요함

`PID 1234 java`를 의심한 것은 맞다.

다만 이유는 `TIME+ 10:20` 때문이라기보다 `%CPU 180.0` 때문이다.

```text
%CPU:
지금 이 프로세스가 CPU를 얼마나 쓰고 있는지

TIME+:
프로세스가 지금까지 누적해서 CPU를 쓴 시간
```

2코어 서버에서 `java`가 `180%`를 쓰고 있다는 것은, CPU 2개 중 1.8개 정도를 혼자 쓰는 느낌이다.

장애 순간에는 보통 다음 값이 더 중요하다.

```text
- %CPU
- %MEM
- PID
- COMMAND
```

### 다음에 쳐야 할 명령어

`top`에서 수상한 PID를 찾았으면, 다음으로 그 PID의 정체를 확인한다.

```bash
ps -fp 1234
```

확인하고 싶은 것:

```text
java -jar api-server.jar 인지
java -jar batch-worker.jar 인지
java -jar old-test-server.jar 인지
```

그 다음은 로그와 연결 상태를 같이 본다.

```bash
tail -f app.log
journalctl -u <service-name> -f
ss -ant | grep ':8080' | awk '{print $1}' | sort | uniq -c
```

### 원인 후보

가능한 원인:

```text
- 요청이 몰림
- 특정 API에 트래픽 집중
- 최근 배포로 비효율적인 코드가 들어감
- 무거운 반복문이나 대량 데이터 처리
- 캐시 미스로 매 요청마다 무거운 계산 수행
- DB 조회 결과를 너무 많이 가져와서 애플리케이션에서 처리 중
- 배치 작업이 API 서버에서 같이 돌고 있음
```

### 최종 정리

```text
top에서 범인 찾는 순서:

1. load average로 서버 전체 압박을 본다.
2. CPU 코어 수와 load average를 비교한다.
3. %Cpu(s)의 us, id, wa를 본다.
4. CPU 문제인지 I/O 대기 문제인지 구분한다.
5. 프로세스 목록에서 %CPU가 높은 PID를 찾는다.
6. ps -fp <PID>로 정체를 확인한다.
7. 로그와 포트 연결 상태를 이어서 확인한다.
```

## top 문제 2: I/O wait가 높은 서버 해석

### 문제 상황

서버는 4코어이고, API 응답이 갑자기 느려졌다.

```text
load average: 5.20, 4.90, 4.30
Tasks: 180 total, 1 running, 179 sleeping, 0 stopped, 0 zombie
%Cpu(s): 8.0 us, 3.0 sy, 0.0 ni, 45.0 id, 44.0 wa

PID   USER   PR  NI  VIRT   RES   SHR  S  %CPU  %MEM  TIME+   COMMAND
1234  app    20   0  2.5g   900m  80m   S   12.0 11.2  20:11   java
2222  mysql  20   0  3.0g   1.8g 120m   S    8.0 23.0  55:30   mysqld
3100  root   20   0  200m    60m  20m   S    3.0  0.7   1:20   journald
```

### 내가 맞게 본 것

CPU 계산이 바쁜 것보다는 I/O를 기다리느라 느린 상황으로 본 것은 맞다.

근거:

```text
us 8.0:
앱 코드가 CPU 계산을 많이 하는 상황은 아님

id 45.0:
CPU가 아예 꽉 막힌 상태도 아님

wa 44.0:
CPU가 일을 못 해서가 아니라, 디스크/네트워크 I/O 응답을 기다리는 시간이 큼
```

이 상황은 "자바 서버가 CPU를 태운다"보다는 "읽기/쓰기나 외부 I/O가 느려서 기다리는 상황"에 가깝다.

### 헷갈린 것 1: sleeping 프로세스가 많다고 바로 문제는 아님

`Tasks: 179 sleeping`은 자체로 나쁜 것은 아니다.

리눅스 프로세스는 평소에도 대부분 sleeping 상태일 수 있다.

```text
요청 기다림
I/O 기다림
타이머 기다림
이벤트 기다림
```

이런 이유로 sleeping 상태가 많을 수 있다.

이 문제에서는 `sleeping` 개수보다 `%Cpu(s)`의 `wa 44.0`이 훨씬 중요한 신호다.

### 헷갈린 것 2: load average는 I/O 대기도 포함될 수 있음

`load average: 5.20`은 4코어 기준으로 조금 높은 편이다.

```text
4코어 서버에서 load average 4.0:
거의 꽉 찬 기준선

4코어 서버에서 load average 5.2:
처리 가능한 수준을 살짝 넘어서 대기가 생기는 중
```

하지만 여기서 중요한 점은 `load average`가 CPU 계산 대기만 의미하지 않는다는 것이다.

I/O 대기 중인 작업도 load average에 포함될 수 있다.

이 문제에서는 다음 조합으로 봐야 한다.

```text
load average가 5.2라서 조금 높음
그런데 us는 낮고 wa가 높음
-> CPU 계산이 밀린 게 아니라 I/O 때문에 작업들이 기다리는 쪽
```

### 제일 중요한 신호: wa가 높음

이 문제에서 제일 눈에 띄는 값은 `wa 44.0`이다.

그다음으로 `id 45.0`도 같이 봐야 한다.

```text
id도 높은데 서버가 느림
wa도 높음
```

이 조합은 다음처럼 해석한다.

```text
CPU는 어느 정도 놀 여유가 있는데,
작업들이 디스크/네트워크 응답을 기다려서 진행이 안 됨
```

### 원인 후보

가능한 원인:

```text
- DB 쿼리가 느려서 애플리케이션 요청들이 기다림
- MySQL이 디스크 읽기/쓰기를 많이 함
- 로그를 너무 많이 써서 디스크 I/O가 밀림
- 디스크 용량 부족 또는 디스크 성능 저하
- 외부 API, Redis, DB 같은 네트워크 I/O가 느림
- 파일 업로드/다운로드 작업이 몰림
```

단, `top`만 보고 "슬로우쿼리다"라고 확정하면 안 된다.

`wa`는 "I/O 기다림이 많다"까지만 말해준다.

그 I/O가 디스크인지, DB인지, 로그인지, 네트워크인지는 다음 명령어로 좁혀야 한다.

### 다음에 확인할 명령어

먼저 기본 리소스 상태를 확인한다.

```bash
df -h
free -h
```

확인하는 것:

```text
df -h:
디스크가 꽉 찼는지 확인

free -h:
메모리 부족으로 swap을 많이 쓰는지 확인
```

MySQL이 의심되면 프로세스 정체와 로그도 이어서 본다.

```bash
ps -fp 2222
tail -f app.log
journalctl -u <service-name> -f
```

나중에 I/O를 더 깊게 볼 때는 이런 명령어도 쓴다.

```bash
iostat -xz 1
iotop
```

지금 단계에서는 다음 흐름만 기억해도 충분하다.

```text
top에서 wa가 높다
-> CPU 계산 문제가 아니라 I/O 대기 가능성
-> df -h로 디스크 용량 확인
-> free -h로 메모리/swap 확인
-> 앱 로그/DB 상태를 이어서 확인
```

### 최종 정리

```text
sleeping 프로세스가 많다고 바로 문제는 아니다.

load average는 CPU 코어 수와 비교하되,
I/O 대기 작업도 섞일 수 있다.

wa가 높으면 CPU가 바쁜 게 아니라
I/O를 기다리는 상황일 수 있다.
```
