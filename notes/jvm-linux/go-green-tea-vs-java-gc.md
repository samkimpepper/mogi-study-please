---
tags:
  - backend
  - java
  - jvm
  - go
  - gc
  - performance
review_answered: false
---

# Go Green Tea GC와 Java GC 비교

## 한 줄 정리

Go Green Tea, Java G1, Java ZGC는 모두 heap을 일정한 메모리 구역으로 나누어 다루지만 목적이 다르다. **Green Tea는 mark 순회를 페이지별로 묶어 CPU cache 효율을 높이고, G1은 garbage가 많은 region부터 골라 live object를 다른 region으로 옮기며, ZGC는 marking과 relocation을 애플리케이션 실행 중에 최대한 동시 수행해 정지 시간을 줄인다.**

Go Green Tea의 동작 원리는 [[green-tea-gc-page-based-tracing]]에서 이어진다.

## 먼저 기억할 것

“Java GC”라는 하나의 알고리즘이 존재하는 것은 아니다.

HotSpot JVM에는 목적이 다른 여러 garbage collector가 있다.

| Collector | 주된 목표 |
| --- | --- |
| Serial GC | 작은 heap과 단순한 환경 |
| Parallel GC | 처리량 우선 |
| G1 GC | 처리량과 예측 가능한 pause time의 균형 |
| ZGC | 매우 짧은 pause time |

서버급 환경에서는 G1이 기본으로 선택되는 경우가 많다. 매우 짧은 응답 지연이 중요하면 ZGC를 선택할 수 있다.

이 노트에서는 Java 백엔드에서 접할 가능성이 큰 **G1**을 중심으로 비교하고, 저지연 collector인 **ZGC**도 함께 본다.

---

## 세 방식의 공통점

Go Green Tea, Java G1, Java ZGC는 세부 구현은 다르지만 tracing GC라는 공통 기반을 가진다.

```text
GC root에서 출발
→ 객체의 reference를 따라감
→ 도달 가능한 객체를 live로 판단
→ 도달하지 못한 객체의 메모리를 회수
```

세 방식 모두 heap을 page나 region 같은 메모리 구역으로 나누어 관리한다. 그렇다고 객체 생존 여부를 구역 전체에 대해 하나로 판단하는 것은 아니다.

```text
같은 page 또는 region

[live object][dead object][live object][dead object]
```

구역 안에서도 어느 객체가 살아 있는지는 구분해야 한다.

> 메모리를 구역으로 나눈다는 공통점만 보고 같은 GC 알고리즘이라고 생각하면 안 된다. 구역을 무엇을 위해 사용하는지가 중요하다.

---

## Go Green Tea가 해결하려는 문제

기존 Go GC의 mark 단계는 객체를 작업 목록에 넣고 포인터를 따라가는 graph flood 방식에 가깝다.

```text
객체 A
→ 멀리 있는 객체 B
→ 멀리 있는 객체 C
→ 다시 A와 가까운 객체 D
```

논리적으로 연결된 객체들이 물리 메모리에서 가까이 있다는 보장은 없다. 따라서 GC worker가 heap 여기저기로 이동하면서 CPU cache miss와 메모리 대기를 많이 겪을 수 있다.

Green Tea는 작업 목록의 단위를 객체에서 8KiB Go page로 바꾼다.

```text
객체 작업 목록
→ 페이지 작업 목록
```

페이지마다 객체별 `seen`과 `scanned` bitmap을 유지한다.

```text
seen = 1
→ 살아 있는 객체로 발견함

scanned = 1
→ 객체 내부 reference까지 조사함
```

페이지를 FIFO queue에 잠시 두어 같은 페이지에서 발견되는 여러 객체를 모은 뒤 연속해서 scan한다.

```text
페이지 A를 queue에 넣음
→ A1 발견
→ 기다리는 동안 A2와 A4도 발견
→ 페이지 A 차례에 A1, A2, A4를 한꺼번에 scan
```

Green Tea의 핵심 목표는 다음과 같다.

- 같은 페이지의 가까운 객체를 연속해서 읽는다.
- CPU cache와 prefetch를 더 잘 활용한다.
- 불규칙한 pointer chasing을 줄인다.
- 객체보다 적은 수의 페이지를 작업 목록에 넣어 queue 경쟁을 줄인다.
- page bitmap을 이용해 vector 연산을 활용하기 쉽게 만든다.

Green Tea는 기존 Go의 mark-sweep 구조를 바탕으로 **mark 단계의 CPU 효율과 throughput을 개선하는 최적화**다.

---

## Java G1의 region은 무엇을 위한 것인가

G1도 Java heap을 동일한 크기의 여러 region으로 나눈다.

```text
Java heap

[Eden region][Old region][Eden region][Survivor region][Old region]
```

Eden, Survivor, Old는 반드시 물리적으로 연속된 하나의 공간이 아니라 region들의 논리적인 집합이다.

G1은 concurrent marking을 통해 각 region에 live object가 얼마나 있는지 파악한다. 이후 garbage가 많아서 회수 효율이 좋은 region들을 collection set에 포함한다.

```text
Region A: live 10%, garbage 90%
Region B: live 80%, garbage 20%

→ 같은 시간에 더 많은 공간을 회수할 수 있는 Region A를 우선 고려
```

이것이 Garbage-First라는 이름의 감각이다.

선택된 region을 수거할 때는 보통 live object를 다른 region으로 복사한다.

```text
수거 대상 region

[live A][dead][dead][live B][dead]

→ live A와 live B를 다른 region으로 이동
→ 기존 region 전체를 빈 공간으로 재사용
```

이 작업을 evacuation이라고 한다. 살아 있는 객체를 모아서 옮기므로 heap compaction 효과도 얻는다.

G1의 region은 주로 다음 목적에 사용된다.

- heap을 작은 수거 단위로 나눈다.
- garbage가 많은 region부터 선택한다.
- 한 번의 pause에 처리할 region 수를 조정한다.
- live object를 다른 region으로 옮겨 공간을 압축한다.
- pause time 목표에 맞춰 collection set 크기를 조절한다.

> G1의 핵심은 region을 queue에 오래 두어 같은 위치의 mark 작업을 모으는 것이 아니라, 어느 region을 이번에 수거하고 evacuate할지 선택하는 것이다.

---

## Green Tea page와 G1 region은 무엇이 다른가

둘 다 연속된 메모리 구역이지만 크기와 역할이 다르다.

| 비교 | Go Green Tea page | Java G1 region |
| --- | --- | --- |
| 대표 크기 | 8KiB Go heap page | 보통 MiB 단위로 결정되는 큰 구역 |
| 핵심 역할 | mark scan 작업을 가까운 메모리끼리 묶음 | allocation과 reclamation, evacuation 단위 |
| 주요 목표 | cache locality와 marking CPU 비용 개선 | garbage가 많은 구역 우선 회수, pause 예측 |
| 객체 이동 | 기존 Go mark-sweep 구조이므로 일반 객체를 compact하기 위한 이동이 핵심이 아님 | 선택된 region의 live object를 다른 region으로 복사 |
| 주요 metadata | 객체별 `seen`·`scanned` bitmap | marking 정보, region 통계, remembered set 등 |
| 작업 묶기 | FIFO page queue에서 scan 대상 축적 | collection set에 수거할 region 선택 |

같은 “구역 단위 GC”라는 표현을 쓰더라도 다음처럼 목적이 다르다.

```text
Green Tea
→ 이 페이지에서 scan할 객체를 모아 한 번에 읽자

G1
→ garbage가 많은 region을 골라 live object만 옮기고 region을 비우자
```

---

## Green Tea bitmap과 G1 remembered set은 다르다

둘 다 bit나 작은 메모리 구역의 metadata를 사용하므로 비슷하게 느껴질 수 있지만 추적하는 정보가 다르다.

### Green Tea의 `seen`·`scanned`

페이지 내부의 객체별 marking 진행 상태를 나타낸다.

```text
이 객체를 발견했는가?
이 객체 내부 reference를 조사했는가?
```

### G1의 remembered set과 card

다른 region에서 이 region을 가리키는 reference가 있을 가능성이 있는 위치를 추적한다.

```text
Region A의 객체
→ Region B의 객체를 가리킴

Region B를 수거할 때
→ Region A 전체를 다시 scan하지 않음
→ remembered set이 가리키는 card를 우선 확인
```

G1은 heap을 작은 card 영역으로도 나누고, cross-region reference가 생긴 위치를 remembered set에 기록한다. 이를 통해 일부 region만 수거할 때 heap 전체를 다시 훑지 않아도 된다.

| Metadata | 답하려는 질문 |
| --- | --- |
| Green Tea `seen` | 이 객체가 live object로 발견됐는가? |
| Green Tea `scanned` | 이 객체 내부 reference까지 조사했는가? |
| G1 remembered set | 수거 대상 region을 외부에서 가리킬 수 있는 reference는 어디에 있는가? |

> Green Tea bitmap은 객체 그래프 순회의 진행 상태이고, G1 remembered set은 region 경계를 넘는 reference의 위치를 찾기 위한 색인에 가깝다.

---

## 세대별 GC 여부도 다르다

Java의 대표 collector들은 객체 대부분이 젊을 때 죽는다는 generational hypothesis를 활용한다.

```text
새 객체
→ Young generation에 할당
→ 대부분 빠르게 죽음
→ 살아남은 객체만 Old generation으로 이동
```

G1은 Eden, Survivor, Old region을 사용한다. 최신 ZGC도 young과 old generation을 나누는 generational collector다.

Go Green Tea가 해결하려는 핵심 문제는 세대 분리가 아니다. 기존 Go의 tracing mark-sweep에서 객체 graph를 scan하는 순서를 page 중심으로 바꾸는 것이 핵심이다.

```text
Java generational GC
→ 자주 죽는 young object를 중심으로 더 자주 수거

Go Green Tea
→ mark해야 할 live object를 page별로 묶어 더 효율적으로 scan
```

서로 경쟁하는 하나의 선택지가 아니라, GC의 서로 다른 측면을 최적화하는 아이디어다.

---

## Java ZGC와 비교하면

ZGC도 heap을 page 단위로 관리하므로 이름만 보면 Green Tea와 더 비슷해 보일 수 있다. 하지만 주된 목표는 다르다.

ZGC는 marking, relocation 같은 비싼 작업 대부분을 애플리케이션 thread와 동시에 수행한다. 객체가 이동하는 동안에도 애플리케이션이 올바른 객체를 참조할 수 있도록 colored pointer와 load/store barrier를 사용한다.

```text
객체가 다른 위치로 이동
→ 기존 reference가 남아 있을 수 있음
→ barrier가 pointer metadata를 확인
→ 필요하면 올바른 새 위치를 사용하도록 처리
```

ZGC의 중심 목표는 heap이 매우 커도 애플리케이션 정지 시간을 짧게 유지하는 것이다.

| 비교 | Go Green Tea | Java ZGC |
| --- | --- | --- |
| 핵심 목표 | marking CPU 비용과 cache locality 개선 | 매우 짧은 pause time |
| 주요 아이디어 | page FIFO queue, `seen`·`scanned` bitmap | concurrent marking·relocation, colored pointer, barrier |
| 객체 이동 | compaction을 위한 객체 이동이 중심이 아님 | 객체 relocation을 애플리케이션과 동시에 수행 |
| 주된 비용 교환 | page별 작업 축적과 metadata 관리 | 애플리케이션의 pointer load/store에 barrier 비용 |

둘 다 현대 CPU와 큰 heap을 고려하지만 Green Tea는 **어떻게 더 cache 친화적으로 scan할까**, ZGC는 **객체를 옮기는 동안에도 어떻게 애플리케이션을 거의 멈추지 않을까**에 더 가깝다.

---

## 전체 비교

| 관점 | Go Green Tea | Java G1 | Java ZGC |
| --- | --- | --- | --- |
| 기반 | tracing mark-sweep | generational, regional, evacuation | generational, concurrent compacting |
| 대표 목표 | GC CPU와 cache locality | 처리량과 pause 목표의 균형 | 매우 짧은 pause |
| 메모리 구역 | 8KiB Go page | MiB 단위 G1 region | ZGC heap page |
| mark 작업 특징 | page FIFO queue에서 대상 객체 축적 | concurrent marking으로 region별 live 상태 파악 | marking 대부분을 concurrent 수행 |
| 공간 회수 | unreachable object의 slot 재사용 | collection set의 live object를 evacuate한 뒤 region 재사용 | live object를 concurrent relocation |
| 주요 metadata | `seen`·`scanned` bitmap | mark 정보, remembered set, card | colored pointer, mark·relocation metadata, remembered set |
| 대표 트레이드오프 | page에 scan 대상이 잘 모이지 않으면 이득 감소 | evacuation pause와 remembered set 관리 비용 | barrier와 concurrent GC thread의 CPU 비용 |

---

## Java 백엔드 개발자 관점에서 가져갈 점

### 1. Green Tea 옵션을 Java에 적용하는 것은 아니다

Green Tea는 Go runtime의 GC 구현이다. Java 애플리케이션에서 켜는 JVM 옵션이 아니다.

가져갈 것은 특정 설정이 아니라 다음 관점이다.

> 같은 수의 객체를 mark해도 메모리를 어떤 순서로 읽느냐에 따라 GC CPU 비용이 크게 달라질 수 있다.

### 2. Java에서 region이라는 단어를 봐도 Green Tea page로 이해하지 않는다

G1 로그에서 `Eden regions`, `Old regions`, `Humongous regions`가 나오는 것은 Green Tea의 page queue와 다른 이야기다.

```text
Green Tea page
→ mark scan locality

G1 region
→ generation 구성, collection set 선택, evacuation
```

### 3. G1에서는 allocation rate와 live set이 중요하다

애플리케이션이 객체를 매우 빠르게 만들면 young GC가 자주 발생한다. GC 이후에도 살아남는 객체가 많으면 옮겨야 할 live object가 많아져 evacuation 비용이 커질 수 있다.

```text
객체 생성 속도 증가
→ Eden이 빠르게 참
→ young GC 빈도 증가

GC 후 생존 객체 증가
→ 복사할 객체 증가
→ pause 중 evacuation 작업 증가
```

### 4. ZGC는 낮은 pause의 대가로 공짜가 아니다

ZGC는 많은 작업을 concurrent하게 수행하지만, 애플리케이션의 reference 접근에 barrier가 들어가고 GC thread도 CPU를 사용한다.

```text
pause 감소
↔ barrier와 concurrent 작업의 throughput 비용
```

GC 선택은 “최신 GC가 무조건 좋다”가 아니라 응답 지연, 처리량, heap 크기와 CPU 여유 사이의 선택이다.

### 5. GC 문제는 pause만 보지 않는다

Java 백엔드에서는 다음을 함께 봐야 한다.

- 객체 allocation rate
- GC 이후 남는 live set 크기
- young·mixed GC 빈도
- pause time과 그 원인
- old generation 증가 속도
- humongous object 발생
- GC thread의 CPU 사용량
- container memory limit과 `-Xmx`

> Green Tea 트윗이 주는 실무적 교훈은 “페이지 단위가 더 좋다”가 아니라, GC 성능은 살아 있는 객체 수뿐 아니라 객체 배치, 참조 구조, 순회 순서와 CPU cache에도 영향을 받는다는 점이다.

---

## 쉬운 비유

### Go Green Tea

창고의 물건을 확인할 때 물건의 연결 관계만 따라 창고를 뛰어다니지 않고, 같은 선반에서 확인할 물건을 모아 한꺼번에 조사한다.

```text
목표
→ 창고 안을 덜 뛰어다니기
```

### Java G1

쓰레기가 가장 많이 쌓인 창고 구역을 골라, 쓸 만한 물건만 다른 구역으로 옮긴 뒤 기존 구역 전체를 비운다.

```text
목표
→ 제한된 시간 안에 많은 빈 공간 확보
```

### Java ZGC

직원들이 계속 물건을 사용하는 중에도 이사 담당자가 물건을 옮긴다. 직원이 옛 위치를 찾아가면 안내 표지가 새 위치로 연결해준다.

```text
목표
→ 창고 영업을 거의 멈추지 않고 정리
```

---

## 참고 자료

- [Go 공식 블로그: The Green Tea Garbage Collector](https://go.dev/blog/greenteagc)
- [Oracle JDK 26 GC Tuning Guide: Available Collectors](https://docs.oracle.com/en/java/javase/26/gctuning/available-collectors.html)
- [Oracle JDK 26 GC Tuning Guide: G1 Garbage Collector](https://docs.oracle.com/en/java/javase/26/gctuning/garbage-first-g1-garbage-collector1.html)
- [Oracle JDK 26 GC Tuning Guide](https://docs.oracle.com/en/java/javase/26/gctuning/)
- [OpenJDK JEP 439: Generational ZGC](https://openjdk.org/jeps/439)

## 복습 질문

- [ ] Go Green Tea의 page와 Java G1의 region은 각각 무엇을 위한 작업 단위인가?
- [ ] Green Tea의 `seen`·`scanned` bitmap과 G1의 remembered set은 어떤 질문에 답하는 metadata인가?
- [ ] Green Tea가 mark-sweep 구조에서 cache locality를 개선하는 방식과 G1이 region 공간을 회수하는 방식은 어떻게 다른가?
- [ ] ZGC가 매우 짧은 pause를 만들기 위해 colored pointer와 barrier를 사용하는 이유는 무엇인가?
- [ ] Java 백엔드에서 GC 문제를 볼 때 pause time 외에 어떤 지표와 애플리케이션 동작을 함께 확인해야 하는가?

## 백지 복습 답변

### 1. Go Green Tea의 page와 Java G1의 region은 각각 무엇을 위한 작업 단위인가?

내 답:

### 2. Green Tea의 `seen`·`scanned` bitmap과 G1의 remembered set은 어떤 질문에 답하는 metadata인가?

내 답:

### 3. Green Tea가 mark-sweep 구조에서 cache locality를 개선하는 방식과 G1이 region 공간을 회수하는 방식은 어떻게 다른가?

내 답:

### 4. ZGC가 매우 짧은 pause를 만들기 위해 colored pointer와 barrier를 사용하는 이유는 무엇인가?

내 답:

### 5. Java 백엔드에서 GC 문제를 볼 때 pause time 외에 어떤 지표와 애플리케이션 동작을 함께 확인해야 하는가?

내 답:

## 한 줄 회고

- 헷갈렸던 점:
