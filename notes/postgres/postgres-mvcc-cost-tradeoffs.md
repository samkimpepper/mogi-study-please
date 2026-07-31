---
tags:
  - backend
  - postgres
  - database
  - mvcc
  - transaction
review_answered: false
---

# MVCC 비용은 사라지지 않고 다른 곳으로 이동한다

## 한 줄 정리

MVCC에서 긴 트랜잭션에 일관된 과거 화면을 제공하면서 다른 트랜잭션의 쓰기도 허용하려면, 옛 데이터 버전을 어딘가에 보존해야 한다. PostgreSQL은 그 비용이 dead tuple과 VACUUM으로 눈에 잘 보이고, 다른 DBMS는 undo나 version store 같은 다른 영역에서 비용을 지불한다.

## 공부하게 된 계기

한 시니어 개발자가 다음과 같은 취지의 글을 썼다.

> 악명 높은 PostgreSQL MVCC 오버헤드가 다른 DB에 존재하지 않는 것은 아니다. 그 비용을 다른 곳으로 옮겼을 뿐이다.
>
> Long-running transaction의 비용은 논리적으로 제거할 수 없고, 어디에 배치해도 트레이드오프를 교환해야 한다. 각 DBMS는 사용 사례에 맞춰 의사결정을 했을 뿐이다.

이 글은 PostgreSQL의 dead tuple과 VACUUM 문제가 가짜라는 뜻이 아니다. **동일한 논리적 문제를 다른 DBMS도 해결해야 하며, 구현에 따라 비용이 나타나는 장소와 형태가 다르다**는 뜻이다.

이전에 정리한 [[postgres-mvcc-dead-tuples-vacuum]]에서 한 단계 더 나아간 내용이다.

---

## 긴 트랜잭션은 왜 과거 버전을 붙잡는가

A가 긴 조회 트랜잭션을 시작했을 때 `users.name`이 `mogi`였다고 해보자.

```text
A: SELECT 시작
   name = 'mogi'인 snapshot을 봄
```

그동안 B가 같은 데이터를 수정하고 커밋한다.

```text
B: name을 'senior-mogi'로 UPDATE
B: COMMIT
```

B가 커밋했더라도 A가 자기 트랜잭션의 일관된 snapshot을 계속 보려면, DB는 A에게 옛 값인 `mogi`를 보여줄 수 있어야 한다.

이를 처리하는 선택지는 대략 다음과 같다.

1. A가 끝날 때까지 B의 쓰기를 막는다.
2. B의 쓰기를 허용하고, A를 위해 옛 버전을 보존한다.
3. 옛 버전을 보존하지 않고 A를 실패시키거나 다시 시작시킨다.
4. 일관성을 완화하고 A에게 중간에 바뀐 값을 보여준다.

일반적인 MVCC DBMS는 읽기와 쓰기의 동시성을 얻기 위해 주로 2번을 선택한다.

> A에게 과거의 일관된 화면을 계속 제공하려면, 그 과거를 재현할 정보가 어딘가에는 남아 있어야 한다.

---

## PostgreSQL은 비용을 어디에 두는가

PostgreSQL은 `UPDATE`할 때 기존 tuple을 제자리에서 단순히 덮어쓰기보다 새 tuple 버전을 만들고, 옛 tuple도 테이블에 남겨둔다.

```text
테이블 내부

옛 tuple: name = 'mogi'         → A가 아직 볼 수 있음
새 tuple: name = 'senior-mogi'  → 현재 버전
```

A처럼 오래된 snapshot을 사용하는 트랜잭션이 끝나지 않았다면, VACUUM은 옛 tuple을 함부로 회수할 수 없다.

```text
long-running transaction
→ 오래된 snapshot 유지
→ 옛 tuple이 아직 필요할 수 있음
→ VACUUM이 공간을 회수하지 못함
→ dead tuple과 bloat 증가
```

그래서 PostgreSQL의 MVCC 비용은 다음과 같은 모습으로 드러난다.

- dead tuple 축적
- 테이블과 인덱스 bloat
- VACUUM 작업 부담
- 불필요한 디스크 I/O와 스캔 비용
- transaction ID wraparound 관리 부담

이 비용은 실제 PostgreSQL의 운영상 약점이 될 수 있다. 특히 `UPDATE`와 `DELETE`가 많거나, 오래 열린 트랜잭션 때문에 VACUUM이 계속 지연되는 워크로드에서는 더욱 그렇다.

---

## 다른 DBMS는 비용을 어디에 두는가

다른 DBMS가 PostgreSQL과 똑같은 방식으로 dead tuple을 테이블에 남기지 않는다고 해서 옛 버전 유지 비용이 사라지는 것은 아니다.

| DBMS 또는 방식 | 옛 버전 유지 비용이 주로 드러나는 곳 |
| --- | --- |
| PostgreSQL | 테이블의 옛 tuple, VACUUM, bloat |
| MySQL InnoDB | undo log, history list 증가, purge 지연 |
| Oracle | undo tablespace |
| SQL Server | `tempdb` version store 또는 별도 버전 저장소 |

예를 들어 MySQL InnoDB에서도 오래된 snapshot을 가진 트랜잭션이 있으면, purge가 그 트랜잭션에 필요할 수 있는 undo 정보를 제거하지 못한다.

```text
PostgreSQL
→ 테이블의 옛 tuple을 아직 치우지 못함

MySQL InnoDB
→ 과거 버전을 재구성할 undo 정보를 아직 치우지 못함
```

저장 장소와 청소 방식은 다르지만 근본 원인은 같다.

> 아직 옛날 화면을 보는 트랜잭션이 있으므로, 그 화면을 만드는 데 필요한 정보를 버릴 수 없다.

따라서 PostgreSQL의 테이블 bloat 대신 다른 DBMS에서는 undo 영역 증가, version store 증가, purge 지연, 로그나 임시 저장 공간 압박 등의 비용이 나타날 수 있다.

---

## PostgreSQL tuple 방식과 undo 방식의 차이

먼저 PostgreSQL이 옛 버전을 별도의 “옛 테이블”에 보관하는 것은 아니다. **원래 테이블 파일 안에 옛 tuple과 새 tuple이 함께 존재한다.**

`name = 'mogi'`를 `name = 'senior-mogi'`로 수정하는 상황을 비교해보자.

### PostgreSQL: 완성된 옛 row를 테이블에 남긴다

```text
테이블 내부

옛 tuple: name = 'mogi'
새 tuple: name = 'senior-mogi'
```

- 새 snapshot은 새 tuple을 본다.
- 오래된 snapshot은 옛 tuple을 직접 본다.
- 아무도 옛 tuple을 필요로 하지 않으면 VACUUM이 그 공간을 재사용 가능하게 만든다.

### Undo 방식: 과거를 복원할 정보를 별도 영역에 남긴다

```text
테이블

현재 row: name = 'senior-mogi'

undo 영역

변경 전 name은 'mogi'였음
```

- 새 snapshot은 테이블의 현재 row를 본다.
- 오래된 snapshot은 undo 기록을 따라가 과거 상태를 재구성한다.
- 아무도 그 기록을 필요로 하지 않으면 purge가 undo를 정리한다.

실제 구현은 DBMS마다 다르지만, 개념적으로 PostgreSQL은 **과거의 완성된 row 버전**을 남기고 undo 방식은 **과거를 복원할 변경 이력**을 남긴다고 이해할 수 있다.

### 최신 데이터와 과거 데이터 읽기

PostgreSQL에서는 원래 테이블 안에 여러 버전이 함께 존재하므로 각 tuple이 현재 snapshot에 보이는지 판단한다. VACUUM이 잘 따라오지 못하면 테이블 본체가 커지고 더 많은 페이지를 읽게 되어 최신 데이터 조회도 bloat의 영향을 받을 수 있다.

undo 방식은 테이블 본체를 비교적 최신 row 중심으로 유지할 수 있다. 현재 데이터를 읽는 일반적인 조회는 보통 undo를 따라가지 않지만, 오래된 snapshot은 현재 row에서 undo chain을 거슬러 과거 상태를 재구성해야 할 수 있다.

```text
현재 버전 v5
→ undo를 적용해 v4
→ undo를 적용해 v3
오래된 snapshot이 원하는 v2까지 이동
```

따라서 undo chain이 길면 오래된 snapshot을 읽는 비용이 커질 수 있다. 다만 실제 성능은 DBMS의 인덱스와 버전 관리 구현에 따라 달라지므로, PostgreSQL은 과거 조회가 빠르고 undo 방식은 느리다고 단정할 수는 없다.

### UPDATE와 정리 비용

```text
PostgreSQL: 새 tuple 생성
→ 기존 tuple은 옛 버전이 됨
→ 나중에 VACUUM이 공간 회수

Undo 방식: 변경 전 정보를 undo에 기록
→ 테이블 row를 최신 값으로 변경
→ 나중에 purge가 불필요한 undo 정리
```

PostgreSQL은 테이블 공간과 인덱스 갱신 비용이 생길 수 있다. `HOT update`가 인덱스 갱신을 줄여주는 경우도 있지만 항상 가능한 것은 아니다. undo 방식은 테이블에 옛 row를 계속 추가하는 대신 undo 저장 공간, 기록 I/O, chain 관리와 purge 비용을 지불한다.

### 긴 트랜잭션이 만드는 문제

```text
PostgreSQL: 오래된 snapshot이 옛 tuple을 붙잡음
→ VACUUM의 공간 회수 지연
→ 테이블·인덱스 bloat로 현재 워크로드에도 영향

Undo 방식: 오래된 snapshot이 undo 기록을 붙잡음
→ purge 지연
→ undo 영역 증가와 오래된 snapshot의 chain 탐색 비용
```

PostgreSQL에서는 문제가 테이블 본체에 직접 번지기 쉽다. undo 방식은 과거 버전의 부담을 별도 영역으로 격리할 수 있지만, undo 저장·탐색·정리 비용을 감당해야 한다.

### 롤백의 차이

PostgreSQL에서 트랜잭션이 새 tuple을 만들었다가 실패하면 해당 tuple은 다른 트랜잭션에 보이지 않게 되고, 나중에 VACUUM이 정리한다.

undo 방식은 변경 전에 남긴 undo 기록을 역순으로 적용해 이전 상태로 되돌릴 수 있다. 따라서 큰 트랜잭션의 롤백에서는 undo를 실제로 적용하는 작업이 오래 걸릴 수 있다.

| 관점 | PostgreSQL tuple 방식 | Undo 방식 |
| --- | --- | --- |
| 과거 버전 위치 | 원래 테이블 내부 | 별도 undo 영역 |
| 최신 row 조회 | dead tuple과 bloat의 영향을 받을 수 있음 | 보통 현재 row를 바로 읽기 쉬움 |
| 과거 row 조회 | 저장된 옛 tuple을 찾아 가시성 판단 | undo chain으로 과거 상태 재구성 |
| `UPDATE` 비용 | 새 tuple 생성, 인덱스 갱신 가능 | undo 생성, 현재 row 변경 |
| 정리 작업 | VACUUM | purge 또는 undo 정리 |
| 긴 트랜잭션 영향 | 테이블·인덱스 bloat로 번지기 쉬움 | undo 증가와 purge 지연으로 나타남 |
| 롤백 방향 | 실패한 tuple을 숨기고 나중에 청소 | undo를 적용하여 이전 상태 복구 |

> PostgreSQL은 완성된 옛 row를 원래 테이블에 보존하고, undo 방식은 현재 row와 과거를 재구성할 정보를 분리한다. 결국 테이블 본체의 공간과 청소에 비용을 낼지, 별도 이력 저장소의 관리와 과거 재구성에 비용을 낼지의 차이다.


---

## “논리적으로 제거할 수 없다”는 뜻

다음 세 조건을 동시에 원한다고 생각해보자.

1. 긴 트랜잭션에도 일관된 snapshot을 제공한다.
2. 그동안 다른 트랜잭션의 쓰기도 허용한다.
3. 과거 버전이나 이를 재구성할 정보는 전혀 보존하지 않는다.

이 세 가지는 동시에 만족시킬 수 없다.

과거 버전을 저장하지 않으려면 쓰기를 막거나, 오래된 조회를 취소하거나, 일관성을 낮추는 등 다른 대가를 지불해야 한다.

| 선택 | 지불하는 비용 |
| --- | --- |
| 과거 버전 보존 | 저장 공간과 정리 작업 |
| 쓰기 차단 | 동시성과 응답 시간 |
| 오래된 조회 취소·재시작 | 실패 처리와 재처리 |
| 일관성 완화 | 애플리케이션의 정확성 |

즉 비용을 완전히 없애는 문제가 아니라, **저장 공간·정리 작업·잠금·실패 가능성·일관성 중 어느 비용을 어디에서 감당할지 선택하는 문제**다.

---

## 이 주장을 오해하면 안 되는 지점

이 글이 “모든 DBMS의 MVCC 성능은 결국 똑같다”는 뜻은 아니다.

같은 논리적 문제를 처리하더라도 구현 방식과 워크로드에 따라 실제 운영 비용은 크게 달라진다. `UPDATE`와 `DELETE`가 매우 많은 환경에서는 PostgreSQL의 bloat와 VACUUM 부담이 실제 약점이 될 수 있다. 반대로 PostgreSQL이 선택한 구조가 구현의 단순성이나 읽기·쓰기 동시성 측면에서 유리한 경우도 있다.

따라서 DBMS를 평가할 때는 “이 비용이 존재하는가?”만 물으면 부족하다. 다음 질문을 함께 봐야 한다.

- 비용이 어느 저장 영역에 쌓이는가?
- 누가, 언제 과거 버전을 정리하는가?
- 긴 트랜잭션이 정리 작업을 어떻게 지연시키는가?
- 우리 워크로드에서는 테이블 bloat, undo 증가, version store 압박 중 무엇이 더 위험한가?

> [!summary]
> MVCC의 과거 버전 유지 비용은 사라지지 않는다. PostgreSQL은 주로 테이블과 VACUUM에서 비용을 내고, 다른 DBMS는 undo, version store, purge 또는 잠금 같은 다른 통장에서 비용을 낸다.

## 복습 질문

- [ ] 긴 트랜잭션에 일관된 snapshot을 제공하려면 왜 옛 버전이 필요한가?
- [ ] PostgreSQL에서 오래된 snapshot이 VACUUM과 bloat에 어떤 영향을 주는가?
- [ ] PostgreSQL과 MySQL InnoDB는 옛 버전 유지 비용을 각각 어디에 두는가?
- [ ] MVCC 비용을 “논리적으로 제거할 수 없다”는 말은 어떤 선택지 사이의 트레이드오프를 뜻하는가?
- [ ] 이 주장이 모든 DBMS의 MVCC 성능이 똑같다는 뜻이 아닌 이유는 무엇인가?

## 백지 복습 답변

### 1. 긴 트랜잭션에 일관된 snapshot을 제공하려면 왜 옛 버전이 필요한가?

내 답:

### 2. PostgreSQL에서 오래된 snapshot이 VACUUM과 bloat에 어떤 영향을 주는가?

내 답:

### 3. PostgreSQL과 MySQL InnoDB는 옛 버전 유지 비용을 각각 어디에 두는가?

내 답:

### 4. MVCC 비용을 “논리적으로 제거할 수 없다”는 말은 어떤 선택지 사이의 트레이드오프를 뜻하는가?

내 답:

### 5. 이 주장이 모든 DBMS의 MVCC 성능이 똑같다는 뜻이 아닌 이유는 무엇인가?

내 답:

## 한 줄 회고

- 헷갈렸던 점:
