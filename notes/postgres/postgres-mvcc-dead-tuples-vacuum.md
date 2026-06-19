---
tags:
  - backend
  - postgres
  - database
  - mvcc
  - vacuum
---

# Postgres MVCC와 dead tuple, VACUUM

## 한 줄 정리

Postgres는 `UPDATE`/`DELETE` 때 기존 row를 바로 물리적으로 덮어쓰거나 지우지 않고, 옛 버전을 dead tuple로 남겨 읽기와 쓰기가 서로 덜 막히게 만든다. 대신 나중에 `VACUUM`으로 청소해야 한다.

## 왜 공부했나

다른 사람이 이런 말을 했다.

```text
PostgreSQL은 insert보다 update/delete에서,
MVCC 구현상 dead tuple이 쌓이고 vacuum으로 정리해야 해서 불리하다.
예전에 대량 삭제했다가 이 이유로 쿼리 캔슬이 일어난 인시던트가 있었다.
```

처음에는 "MSSQL도 삭제하면 트랜잭션 로그를 남기는데 같은 얘기 아닌가?"라고 헷갈렸다.

결론부터 말하면, Postgres의 이 이야기는 로그 파일보다 **테이블 안에 남는 옛 row 버전**이 핵심이다.

## DELETE는 바로 파일에서 사라지는 게 아니다

Postgres에서 다음 삭제를 실행한다고 하자.

```sql
DELETE FROM users
WHERE id = 1;
```

직관적으로는 row가 테이블에서 즉시 사라질 것 같지만, Postgres MVCC 관점에서는 대략 이렇게 된다.

```text
users 테이블 내부

[id=1, name='mogi']  -> 삭제됨 표시가 붙은 dead tuple
```

즉 row 조각이 테이블 파일 안에 당분간 남는다.

나중에 `VACUUM`이 와서 "이제 어떤 트랜잭션도 이 옛 row를 볼 필요가 없다"고 판단하면, 그 공간을 재사용 가능하게 정리한다.

> [!summary]
> Postgres의 `DELETE`는 곧바로 물리 삭제라기보다, 우선 "이 row 버전은 더 이상 현재 버전이 아니다"라고 표시하는 쪽에 가깝다.

## UPDATE도 새 버전을 만든다

`UPDATE`도 기존 row를 제자리에서 덮어쓰기만 하는 것으로 보면 헷갈린다.

```sql
UPDATE users
SET name = 'mogi2'
WHERE id = 1;
```

Postgres MVCC 감각으로는 다음에 가깝다.

```text
옛 row 버전: id=1, name='mogi'   -> dead tuple 후보
새 row 버전: id=1, name='mogi2'  -> 현재 버전
```

그래서 `UPDATE`는 내부적으로 "새 버전 추가 + 옛 버전 dead 처리"에 가깝다.

`INSERT`는 새 row를 추가하면 끝나는 편이지만, `UPDATE`/`DELETE`는 나중에 청소해야 할 옛 버전을 만든다.

## 왜 이런 구조를 쓰나

장점은 읽기와 쓰기가 서로 덜 막힌다는 것이다.

예를 들어 A 트랜잭션이 먼저 긴 조회를 시작했다.

```text
A: SELECT 시작
```

그 사이 B 트랜잭션이 같은 row를 수정하거나 삭제한다.

```text
B: UPDATE 또는 DELETE 실행
```

만약 DB가 기존 row를 즉시 물리적으로 덮어쓰거나 지워버리면, A가 자기 조회 시작 시점의 데이터를 일관되게 보기가 어렵다.

선택지는 크게 둘이다.

```text
1. A가 끝날 때까지 B의 쓰기를 막는다.
2. B는 새 버전/삭제 표시를 만들고, A는 옛 버전을 계속 보게 한다.
```

Postgres는 MVCC로 2번에 가까운 방식을 쓴다.

```text
A는 자기 시점의 옛 버전을 봄
B는 새 버전을 만들거나 삭제 표시를 남김
```

그래서 읽기 때문에 쓰기가 덜 막히고, 쓰기 때문에 읽기가 덜 막힌다.

> [!info]
> MVCC는 "너는 네 트랜잭션 시점의 버전을 보고, 나는 새 버전을 만들게"라는 방식으로 동시성을 높이는 구조다.

## 대신 VACUUM이 필요하다

옛 row 버전을 남기는 방식은 공짜가 아니다.

대량 `UPDATE`/`DELETE`를 하면 dead tuple이 많이 생긴다.

```text
[live tuple][dead tuple][dead tuple][live tuple][dead tuple]
```

이 상태가 오래 가면 다음 문제가 생길 수 있다.

- 테이블 내부에 죽은 row가 많아진다.
- 인덱스에도 불필요한 흔적이 많아진다.
- 쿼리가 더 많은 페이지를 훑게 된다.
- 디스크 I/O가 늘어난다.
- autovacuum이 따라가지 못하면 테이블 bloat가 커진다.

`VACUUM`은 이런 dead tuple을 정리해서 공간을 재사용 가능하게 만든다.

```text
dead tuple
-> 이제 아무도 필요로 하지 않음
-> VACUUM이 공간 재사용 가능 처리
```

일반 `VACUUM`은 보통 테이블 파일 크기를 바로 줄이는 작업이 아니라, 테이블 내부 공간을 다시 쓸 수 있게 만드는 작업에 가깝다.

## MSSQL 트랜잭션 로그와 다른 점

MSSQL도 `DELETE`를 하면 트랜잭션 로그를 남긴다.

```text
MSSQL DELETE
-> 어떤 row를 삭제했는지 로그 파일에 기록
-> 롤백/장애 복구에 사용
```

Postgres도 WAL이라는 로그를 남긴다.

하지만 지금 말하는 Postgres의 불리함은 로그 파일 자체보다, 테이블 안에 남는 dead tuple과 vacuum 부담이다.

| 구분 | MSSQL 트랜잭션 로그 | Postgres dead tuple |
| --- | --- | --- |
| 남는 위치 | 로그 파일 | 테이블/인덱스 내부 |
| 주된 목적 | 롤백, 장애 복구 | MVCC에서 옛 버전 제공 |
| 문제가 되는 상황 | 로그 파일 full, 로그 재사용 불가 | bloat, vacuum 지연, 스캔 비용 증가 |
| 정리 방식 | 커밋, 체크포인트, 로그 백업/재사용 | VACUUM/autovacuum |

> [!warning]
> 둘 다 "삭제했는데 뭔가 남는다"는 점은 비슷하지만, MSSQL 로그와 Postgres dead tuple은 남는 위치와 목적이 다르다.

## 대량 삭제 인시던트가 생기는 흐름

대량 삭제에서 문제가 생기는 흐름은 보통 다음과 비슷하다.

```text
1. 큰 테이블에서 대량 DELETE 실행
2. dead tuple이 대량으로 생김
3. autovacuum이 정리 속도를 따라가지 못함
4. 테이블/인덱스 bloat가 커짐
5. 관련 쿼리가 더 많은 페이지를 읽게 됨
6. 쿼리 지연, statement_timeout, lock_timeout, 수동 cancel 같은 문제가 발생
```

또는 대량 삭제 트랜잭션 자체가 너무 오래 걸려서 취소될 수도 있다.

정확한 원인은 당시 로그와 실행 계획을 봐야 하지만, "대량 삭제 + dead tuple + vacuum 지연"은 Postgres 운영에서 실제로 조심해야 하는 조합이다.

## 실무에서 가져갈 점

Postgres에서 대량 `UPDATE`/`DELETE`를 할 때는 한 번에 너무 크게 처리하지 않는 것이 좋다.

보통은 다음을 함께 본다.

- 배치 크기를 나눌 수 있는가?
- 트랜잭션을 너무 오래 열어두고 있지 않은가?
- autovacuum이 따라오고 있는가?
- dead tuple과 bloat가 쌓이고 있지 않은가?
- 삭제 대신 파티션 drop 같은 방식이 가능한가?

특히 오래 열린 트랜잭션은 VACUUM이 dead tuple을 치우지 못하게 만들 수 있다.

```text
오래 열린 트랜잭션
-> 옛 row 버전을 아직 볼 수 있어야 함
-> VACUUM이 dead tuple을 치우기 어려움
```

> [!summary]
> Postgres의 MVCC는 동시성을 얻기 위해 옛 row 버전을 남긴다. 이 설계 덕분에 읽기와 쓰기가 덜 막히지만, update/delete가 많은 workload에서는 dead tuple과 vacuum 운영을 반드시 신경 써야 한다.

## 복습 질문

- [ ] Postgres에서 `DELETE`한 row가 바로 물리적으로 사라지지 않는 이유는 무엇인가?
- [ ] `UPDATE`가 "새 버전 추가 + 옛 버전 dead 처리"에 가깝다는 말은 무슨 뜻인가?
- [ ] MVCC가 읽기와 쓰기의 충돌을 줄이는 방식은 무엇인가?
- [ ] MSSQL 트랜잭션 로그와 Postgres dead tuple은 무엇이 다른가?
- [ ] 대량 삭제 시 dead tuple과 vacuum 지연이 쿼리 캔슬로 이어질 수 있는 흐름은 무엇인가?

## 한 줄 회고

- 헷갈렸던 점: MSSQL처럼 삭제 로그를 남긴다는 이야기가 아니라, Postgres는 MVCC 때문에 테이블 안에 옛 row 버전이 dead tuple로 남고 이것을 VACUUM이 나중에 청소해야 한다는 점.
