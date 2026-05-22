# 리눅스 장애 대응 기본기

> [!summary]
> 장애 대응의 첫 단계는 "무슨 일이 터졌는지"를 서버에서 직접 확인하는 것이다.
>
> 대시보드는 현상을 보여주고, 터미널 명령어는 원인을 좁히게 해준다.

백엔드 개발자 입장에서는 코드만 보는 것이 아니라, 코드가 실제 서버에서 어떤 프로세스로 떠 있고, 어떤 포트를 열고 있고, 어떤 연결 상태로 막혀 있는지도 볼 수 있어야 한다.

```text
대시보드:
응답시간 증가, 5xx 증가, CPU 증가 같은 현상을 보여줌

터미널 명령어:
어떤 프로세스가 문제인지,
어떤 포트에 연결이 쌓였는지,
서버 리소스가 부족한지,
네트워크 연결이 실제로 되는지 확인함
```

## 문서 목록

| 문서 | 내용 |
| --- | --- |
| [process.md](process.md) | `ps`, `top`, `kill`, PID 확인 |
| [resource.md](resource.md) | CPU, 메모리, 디스크 확인 |
| [network-port.md](network-port.md) | `ss`, `lsof`, TCP 상태 해석 |
| [dns.md](dns.md) | DNS 문제와 확인 명령어 |
| [incident-flow.md](incident-flow.md) | 서버가 느릴 때 실전 확인 순서 |
| [mistake-notes.md](mistake-notes.md) | 문제 풀이 오답노트 |

## 먼저 외울 명령어

| 보고 싶은 것 | 명령어 |
| --- | --- |
| 프로세스 | `ps`, `top` |
| 리소스 | `top`, `free`, `df` |
| 포트/소켓 | `ss`, `lsof` |
| DNS | `nslookup`, `dig`, `curl -v` |
| 종료 | `kill` |

## 빠른 확인 흐름

```bash
top
free -h
df -h
ss -ltnp | grep ':8080'
ss -ant | grep ':8080'
ss -ant | grep ':8080' | awk '{print $1}' | sort | uniq -c
ps -fp <PID>
tail -f app.log
```

## 오늘 기억할 핵심

```text
ps:
프로세스 목록과 PID 확인

top:
실시간 CPU/메모리/프로세스 사용량 확인

free:
메모리 사용량 확인

df:
디스크 사용량 확인

ss:
포트와 TCP 연결 상태 확인

lsof:
특정 포트를 어떤 프로세스가 잡고 있는지 확인

kill:
프로세스 종료
```

## 복습 체크리스트

- [ ] `ps`와 `ss`의 역할 차이를 설명할 수 있다.
- [ ] `top`으로 CPU/메모리 문제를 확인할 수 있다.
- [ ] `free -h`로 메모리 부족 여부를 확인할 수 있다.
- [ ] `df -h`로 디스크 부족 여부를 확인할 수 있다.
- [ ] `ss -ltnp`와 `ss -ant`의 차이를 말할 수 있다.
- [ ] `ESTABLISHED`, `CLOSE_WAIT`, `TIME_WAIT`, `SYN_RECV`의 의미를 구분할 수 있다.
- [ ] "특정 포트 연결이 쌓였다"는 말을 TCP 상태 기준으로 해석할 수 있다.
- [ ] "서버가 느리다"는 상황에서 리소스, 프로세스, 포트, 연결, 로그 순서로 확인할 수 있다.
