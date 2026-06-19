---
tags:
  - backend
  - mssql
  - database
  - concurrency
  - transaction
---

# MSSQL 기본 락과 row versioning 동시성

## 한 줄 정리

MSSQL의 기본 `READ COMMITTED`는 락 기반 동시성에 가깝기 때문에 같은 row를 읽고 쓰면 서로 기다릴 수 있다. `READ_COMMITTED_SNAPSHOT`이나 `SNAPSHOT`을 쓰면 이전 row version을 읽어 읽기/쓰기 blocking을 줄일 수 있다.

## 왜 공부했나

Postgres MVCC를 공부하다가 이런 질문이 생겼다.

```text
Postgres는 조회 중에 삭제가 일어나도 옛 row 버전을 읽는다.
그럼 MSSQL은 조회 도중 row 하나가 삭제되면 어떻게 하지?
MSSQL 기본은 좀 느린 거 아닌가?
```

결론은 다음과 같다.

```text
MSSQL 기본
-> 락으로 정합성을 지킴
-> 충돌하면 기다릴 수 있음

MSSQL row versioning 옵션
-> tempdb version store의 이전 버전을 읽음
-> 읽기/쓰기 blocking을 줄임
```

## 기본 READ COMMITTED는 락 기반에 가깝다

MSSQL 기본 `READ COMMITTED`에서는 조회가 읽는 동안 공유 락, 쓰기가 변경하는 동안 배타 락을 사용한다고 보면 된다.

```text
Shared Lock(S lock)
-> 읽을 때 사용

Exclusive Lock(X lock)
-> update/delete처럼 변경할 때 사용
```

S lock과 X lock은 서로 충돌한다.

```text
읽는 중인 row를 삭제하려면 기다려야 할 수 있음
삭제 중인 row를 읽으려면 기다려야 할 수 있음
```

## 조회가 먼저인 경우

A가 먼저 row를 조회하고 있다.

```sql
-- A
SELECT *
FROM users
WHERE id = 1;
```

B가 같은 row를 삭제하려 한다.

```sql
-- B
DELETE FROM users
WHERE id = 1;
```

기본 락 기반 감각에서는 다음처럼 된다.

```text
A가 row를 읽음
-> S lock 사용

B가 같은 row를 삭제하려 함
-> X lock 필요

S lock과 X lock이 충돌
-> B는 A의 읽기가 끝날 때까지 기다릴 수 있음
```

즉 삭제가 바로 들어가서 A의 읽기 결과를 깨뜨리는 것이 아니라, 락 충돌로 줄을 선다.

## 삭제가 먼저인 경우

반대로 B가 먼저 삭제 트랜잭션을 열고 아직 커밋하지 않았다고 하자.

```sql
-- B
BEGIN TRAN;

DELETE FROM users
WHERE id = 1;

-- 아직 COMMIT 안 함
```

이때 A가 같은 row를 조회한다.

```sql
-- A
SELECT *
FROM users
WHERE id = 1;
```

기본 `READ COMMITTED`에서는 A가 B의 미커밋 삭제를 읽으면 안 된다.

```text
B가 X lock 보유
A는 읽기 위해 S lock 필요
X lock과 S lock 충돌
-> A는 B가 COMMIT/ROLLBACK 할 때까지 기다릴 수 있음
```

B가 커밋하면 A는 0건을 볼 수 있고, B가 롤백하면 기존 row를 볼 수 있다.

> [!summary]
> MSSQL 기본 락 기반 동시성에서는 "읽는 동안 삭제되면 어떻게 되지?"의 답이 보통 "한쪽이 기다린다"에 가깝다.

## Postgres MVCC와의 차이

Postgres MVCC는 기본 감각이 다르다.

```text
A가 조회 시작
B가 같은 row 삭제

A는 자기 트랜잭션 시점의 옛 row 버전을 계속 봄
B는 삭제 표시를 남김
dead tuple은 나중에 VACUUM이 정리
```

MSSQL 기본 락 기반 감각은 다음에 가깝다.

```text
A가 조회 중
B가 같은 row 삭제 시도

S lock과 X lock이 충돌하면 한쪽이 기다림
```

즉 Postgres는 "옛 버전을 남겨서 둘 다 진행"에 가깝고, MSSQL 기본은 "충돌하면 락으로 줄 세움"에 가깝다.

관련 노트: [[Postgres MVCC와 dead tuple, VACUUM]]

## MSSQL도 row versioning을 쓸 수 있다

MSSQL에도 row versioning 기반 옵션이 있다.

대표적으로 다음 두 가지를 먼저 보면 된다.

```text
READ_COMMITTED_SNAPSHOT (RCSI)
SNAPSHOT isolation
```

`READ_COMMITTED_SNAPSHOT`을 켜면, 일반적인 `READ COMMITTED` 조회도 락을 기다리는 대신 커밋된 이전 버전을 읽는 방식에 가까워진다.

```sql
ALTER DATABASE YourDb
SET READ_COMMITTED_SNAPSHOT ON;
```

감각은 다음과 같다.

```text
B가 row 수정/삭제
-> 이전 버전을 tempdb version store에 보관

A가 조회
-> 락을 기다리지 않고 커밋된 이전 버전을 읽을 수 있음
```

이러면 읽기 API나 리포트성 조회가 쓰기 트랜잭션 때문에 오래 막히는 상황을 줄일 수 있다.

## 대신 tempdb 비용이 생긴다

row versioning도 공짜는 아니다.

MSSQL에서 row versioning을 쓰면 이전 row version을 `tempdb`의 version store에 둔다.

```text
MSSQL row versioning
-> 이전 버전 저장 위치: tempdb version store
-> 관리 포인트: tempdb 용량, I/O, 오래 열린 트랜잭션
```

오래 열린 트랜잭션이 있으면 version store의 오래된 row version을 빨리 정리하지 못할 수 있다.

이 점은 Postgres에서 오래 열린 트랜잭션이 VACUUM을 방해하는 것과 비슷한 운영 리스크가 있다.

```text
Postgres
-> 테이블 안 dead tuple
-> VACUUM 관리

MSSQL row versioning
-> tempdb version store
-> tempdb 관리
```

## MSSQL 기본은 항상 느린가

항상 느리다고 보기는 어렵다.

더 정확히는 다음과 같다.

```text
읽기와 쓰기가 같은 row나 범위를 자주 건드리는 workload
-> 기본 락 기반 READ COMMITTED에서 blocking이 잘 보일 수 있음
-> 느리다기보다 기다림이 생긴다
```

반대로 충돌이 적거나, 짧은 트랜잭션 위주거나, 락으로 줄 세우는 정합성을 명확히 선호하는 업무에서는 기본 락 기반 동작이 이해하기 쉬울 수 있다.

금융, ERP 같은 업무에서는 "읽는 중인 데이터와 쓰는 중인 데이터가 어설프게 엇갈리지 않도록 줄 세우기"가 중요한 경우도 있다.

> [!warning]
> `READ_COMMITTED_SNAPSHOT`은 성능 버튼처럼 무조건 켜는 옵션이 아니다. 읽기/쓰기 blocking은 줄지만, tempdb version store 비용과 애플리케이션이 기대하던 락 기반 동작 변화까지 같이 봐야 한다.

## 비교 정리

| 구분 | MSSQL 기본 READ COMMITTED | MSSQL RCSI/SNAPSHOT | Postgres MVCC |
| --- | --- | --- | --- |
| 기본 방식 | 락 기반 | row versioning | MVCC |
| 조회 중 삭제 | 락 충돌 시 한쪽 대기 | 이전 커밋 버전 읽기 가능 | 트랜잭션 시점의 옛 tuple 읽기 |
| 이전 버전 위치 | 기본적으로 사용하지 않음 | tempdb version store | 테이블 내부 tuple |
| 주요 비용 | blocking, lock wait | tempdb 사용량, version store 관리 | dead tuple, VACUUM, bloat |
| 운영 포인트 | 트랜잭션 짧게, 인덱스, 락 대기 확인 | tempdb와 오래 열린 트랜잭션 확인 | autovacuum과 bloat 확인 |

## 실무에서 가져갈 점

MSSQL에서 조회 지연이 보일 때는 쿼리 자체만 느린지, 락 대기 때문에 느린지 구분해야 한다.

```text
쿼리 계산이 느림
-> 실행 계획, 인덱스, 통계 확인

락 대기 때문에 느림
-> 누가 S/X lock을 잡고 있는지
-> 트랜잭션이 오래 열려 있는지
-> 같은 row/range를 읽고 쓰는지 확인
```

읽기/쓰기 blocking이 자주 문제라면 `READ_COMMITTED_SNAPSHOT` 같은 row versioning 옵션을 검토할 수 있다. 하지만 그때는 tempdb 용량과 I/O, 오래 열린 트랜잭션까지 같이 봐야 한다.

> [!summary]
> MSSQL 기본은 충돌을 락으로 줄 세워 정합성을 지키고, RCSI/SNAPSHOT은 이전 버전을 읽어 blocking을 줄인다. Postgres는 기본 MVCC로 비슷한 장점을 얻지만, dead tuple과 VACUUM이라는 다른 비용을 치른다.

## 복습 질문

- [ ] MSSQL 기본 `READ COMMITTED`에서 S lock과 X lock은 각각 언제 쓰이는가?
- [ ] 조회가 먼저이고 같은 row를 삭제하려는 트랜잭션이 들어오면 어떤 일이 생길 수 있는가?
- [ ] 삭제 트랜잭션이 먼저 X lock을 잡고 있으면 조회는 왜 기다릴 수 있는가?
- [ ] `READ_COMMITTED_SNAPSHOT`은 기본 락 기반 읽기와 무엇이 다른가?
- [ ] MSSQL row versioning의 운영 비용이 tempdb와 연결되는 이유는 무엇인가?
- [ ] Postgres MVCC와 MSSQL RCSI/SNAPSHOT은 이전 버전을 저장하는 위치가 어떻게 다른가?

## 한 줄 회고

- 헷갈렸던 점: MSSQL도 동시성을 처리하지만 기본은 Postgres처럼 옛 row를 테이블 안에 남겨 읽는 방식이 아니라, S lock과 X lock 충돌을 통해 한쪽을 기다리게 하는 락 기반 동작에 가깝다는 점. 다만 RCSI/SNAPSHOT을 쓰면 tempdb version store를 통해 이전 버전을 읽을 수 있다.
