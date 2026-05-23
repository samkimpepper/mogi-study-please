# `free -h`로 메모리 상태 확인하기

## 한 줄 정의

`free -h`는 리눅스 서버의 메모리 사용 상태를 사람이 읽기 쉬운 단위로 보여주는 명령어다.

> [!summary]
> `free -h`는 서버 전체 메모리 상태를 빠르게 확인하는 명령어다.
> 실무에서는 `free`보다 `available`을 더 중요하게 본다.

```bash
free -h
```

## 출력 예시

```text
              total        used        free      shared  buff/cache   available
Mem:           7.7Gi       2.1Gi       1.0Gi       120Mi       4.6Gi       5.2Gi
Swap:          2.0Gi          0B       2.0Gi
```

## 컬럼 의미

```text
total:
전체 메모리

used:
현재 사용 중인 메모리

free:
아무것도 사용하지 않는 완전히 빈 메모리

buff/cache:
리눅스가 성능을 위해 버퍼나 파일 캐시로 쓰는 메모리

available:
새 프로그램이 필요할 때 사용할 수 있는 메모리
```

## 제일 중요한 값

`free -h`에서는 `free`보다 `available`을 더 중요하게 본다.

> [!important]
> `free`가 낮다고 바로 메모리 부족이라고 판단하지 않는다.
> 리눅스는 남는 메모리를 캐시로 적극 사용하기 때문에, 실제 여유 메모리는 `available`을 중심으로 본다.

리눅스는 남는 메모리를 그냥 비워두지 않고 파일 캐시나 버퍼로 활용한다. 그래서 `free` 값은 작게 보일 수 있다.

하지만 캐시로 쓰는 메모리는 필요하면 회수해서 애플리케이션에 줄 수 있다. 그래서 실제 여유 메모리를 볼 때는 `available`을 보는 것이 더 실무적이다.

```text
free가 낮다:
완전히 비어 있는 메모리가 적다는 뜻

available이 높다:
필요하면 애플리케이션이 사용할 수 있는 메모리가 아직 충분하다는 뜻
```

## 헷갈리는 예시

> [!example]
> `free`는 낮지만 `available`이 높으면 아직 여유가 있는 상태일 수 있다.

```text
Mem: total 8Gi
used 2Gi
free 500Mi
buff/cache 5Gi
available 5.5Gi
```

이 경우 `free`가 500Mi라서 메모리가 부족해 보일 수 있다.

하지만 `available`이 5.5Gi이므로 실제로는 아직 여유가 많은 상태다.

## 위험하게 볼 상황

다음 상황이면 메모리 부족을 의심할 수 있다.

> [!warning]
> `available`이 낮고 `Swap used`가 계속 증가하면 메모리 압박 상태일 수 있다.
> 이때는 특정 프로세스의 메모리 사용량과 OOM 발생 여부를 같이 확인한다.

```text
available이 매우 낮다
Swap used가 계속 증가한다
서버 응답이 느려진다
OOM Killer로 프로세스가 죽는다
```

특히 `available`이 낮고 swap까지 많이 쓰고 있다면, 메모리 압박 때문에 서버가 느려질 수 있다.

## 백엔드 개발자 관점 확인 흐름

메모리 문제가 의심되면 다음 순서로 본다.

```text
1. free -h로 available과 swap 상태를 본다.
2. top으로 어떤 프로세스가 메모리를 많이 쓰는지 본다.
3. ps로 해당 PID가 어떤 애플리케이션인지 확인한다.
4. JVM heap, 캐시, 대량 조회, 메모리 누수 가능성을 확인한다.
```

프로세스별 메모리 사용량은 다음처럼 볼 수 있다.

```bash
ps aux --sort=-%mem | head
```

또는 실시간으로 보려면 다음 명령어를 쓴다.

```bash
top
```

## 기억할 문장

> [!tip]
> `free -h`를 볼 때는 `available`부터 확인한다.
> 그 다음 swap 사용량과 메모리를 많이 쓰는 프로세스를 이어서 본다.

```text
free -h에서 제일 중요한 값은 available이다.
free가 낮아도 available이 높으면 괜찮은 경우가 많다.
available이 낮고 swap까지 쓰면 메모리 부족을 의심한다.
```

## 실전 문제 1

다음 `free -h` 출력이 있다고 하자.

```text
              total        used        free      shared  buff/cache   available
Mem:           7.7Gi       2.2Gi       420Mi      180Mi       5.1Gi       5.0Gi
Swap:          2.0Gi          0B       2.0Gi
```

### 질문

- 이 서버는 메모리가 부족한 상태일까?
- `free`가 420Mi밖에 없는데 왜 괜찮거나 위험하다고 판단할 수 있을까?
- 이 상황에서 백엔드 개발자가 추가로 확인할 명령어는 무엇일까?

### 풀이

> [!check]
> 이 서버는 메모리 부족 상태가 아니다.
> `available`이 5.0Gi로 충분하고, swap도 사용하지 않고 있기 때문이다.

`free`는 420Mi로 낮아 보인다.

하지만 `free`는 완전히 비어 있는 메모리만 의미한다. 리눅스는 남는 메모리를 `buff/cache`로 적극 사용하기 때문에 `free`가 낮게 보일 수 있다.

이때 봐야 할 값은 `available`이다.

```text
free: 420Mi
available: 5.0Gi
swap used: 0B
```

`available`이 5.0Gi이고 swap을 쓰지 않으므로, 현재는 메모리 여유가 충분한 상태로 판단할 수 있다.

### 다음 확인 명령어

전체 메모리 상태는 괜찮지만, 어떤 프로세스가 메모리를 쓰는지 보고 싶다면 다음 명령어를 사용할 수 있다.

```bash
top
```

또는 메모리를 많이 쓰는 프로세스 순서로 바로 보고 싶다면 다음 명령어를 쓴다.

```bash
ps aux --sort=-%mem | head
```

> [!note]
> 실전 흐름은 `free -h`로 전체 상태를 보고, 문제가 의심되면 `top`이나 `ps`로 어떤 프로세스가 메모리를 많이 쓰는지 좁혀가는 것이다.
