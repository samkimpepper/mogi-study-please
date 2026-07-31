---
tags:
  - backend
  - postgres
  - database
  - mvcc
  - transaction
  - batch
review_answered: false
---

# 주니어 백엔드 개발자를 위한 MVCC 실무 감각

## 한 줄 정리

주니어 백엔드 개발자에게 MVCC 구현 차이는 DB 엔진을 선택하는 이론보다, **트랜잭션을 짧게 설계하고 대량 작업을 나누며 장애가 났을 때 과거 버전이 쌓이는 곳을 올바르게 확인하기 위한 실무 지식**에 가깝다.

MVCC 구현 자체와 PostgreSQL tuple 방식·undo 방식의 차이는 [[postgres-mvcc-cost-tradeoffs]]에서 이어진다.

---

## 실무에서는 어떤 모습으로 만나는가

애플리케이션 개발자가 tuple이나 undo를 직접 구현할 일은 거의 없다. 대신 MVCC 비용은 다음과 같은 증상으로 나타난다.

- 평범한 API가 갑자기 느려진다.
- DB 커넥션 풀이 부족해진다.
- 배치 이후 테이블이나 undo 영역이 커진다.
- 대량 작업을 취소했는데 rollback이 끝나지 않는다.
- 삭제를 완료했는데 PostgreSQL 테이블 파일 크기가 바로 줄지 않는다.
- 특별히 느린 쿼리가 없어 보이는데 VACUUM이나 purge가 따라오지 못한다.

코드에서는 단순한 `@Transactional`이나 반복문으로 보이지만, DB 내부에서는 row 버전 생성과 정리 작업이 계속 일어난다.

---

## 1. `@Transactional` 안에서 너무 많은 일을 하는 경우

다음 코드는 자연스러워 보이지만 외부 API 호출까지 DB 트랜잭션 안에 포함한다.

```java
@Transactional
public void processOrder(Long orderId) {
    Order order = orderRepository.findById(orderId)
        .orElseThrow();

    paymentClient.requestPayment(order);
    emailClient.sendEmail(order);

    order.complete();
}
```

실제 흐름은 다음과 같다.

```text
트랜잭션 시작
→ DB 조회
→ 외부 결제 API 응답 대기
→ 이메일 API 응답 대기
→ DB 수정
→ COMMIT
```

외부 API가 10초 걸리면 DB 트랜잭션과 커넥션도 그동안 열려 있을 수 있다.

- DB 커넥션을 오래 점유한다.
- row lock을 오래 보유할 수 있다.
- 다른 요청의 대기 시간이 늘어난다.
- timeout과 deadlock 가능성이 커진다.
- rollback 범위가 커진다.
- 오래된 snapshot이나 버전 정리에 악영향을 줄 수 있다.

DBMS에 따라 추가 증상의 위치가 달라진다.

```text
PostgreSQL
→ 옛 tuple 정리 지연
→ dead tuple과 bloat 증가 가능

Undo 방식 DBMS
→ undo 기록 정리 지연
→ undo 영역과 history 증가 가능
```

외부 통신을 항상 트랜잭션 밖으로 빼기만 하면 된다는 뜻은 아니다. 결제처럼 여러 시스템 사이의 일관성이 필요한 작업에는 멱등성, outbox, 재시도, 보상 트랜잭션 등의 설계가 필요하다.

실무에서 먼저 가져갈 원칙은 단순하다.

> DB 트랜잭션 안에는 DB 일관성을 위해 꼭 함께 묶여야 하는 작업만 넣고, 네트워크 대기나 긴 연산을 무심코 포함하지 않는다.

---

## 2. 대량 `UPDATE`·`DELETE` 배치를 실행하는 경우

휴면 회원을 한 번에 삭제한다고 해보자.

```sql
DELETE FROM users
WHERE status = 'INACTIVE';
```

애플리케이션에서는 SQL 한 줄이지만 대상이 천만 건이라면 DB 내부 비용은 매우 크다.

### PostgreSQL에서 보이는 증상

```text
대량 DELETE
→ dead tuple 대량 생성
→ VACUUM이 정리해야 함
→ 테이블·인덱스 bloat
→ 다른 쿼리의 디스크 I/O 증가
```

삭제를 커밋했는데도 테이블 파일 크기가 바로 줄지 않을 수 있다. 일반 `VACUUM`은 공간을 OS에 즉시 반환하기보다 테이블 내부에서 재사용할 수 있게 만드는 작업에 가깝기 때문이다.

### Undo 방식 DBMS에서 보이는 증상

```text
대량 DELETE
→ undo 대량 생성
→ undo 영역과 history 증가
→ purge가 뒤처질 수 있음
→ 실패하면 큰 rollback 실행
```

PostgreSQL과 같은 bloat가 보이지 않더라도 undo 공간, purge 지연, rollback 시간이 문제가 될 수 있다.

그래서 대량 작업은 보통 작은 단위로 나눈다.

```text
1,000건 처리 → COMMIT
1,000건 처리 → COMMIT
1,000건 처리 → COMMIT
```

작은 배치로 나누면 다음 장점이 있다.

- 한 번에 생성되는 과거 버전이 줄어든다.
- lock 보유 시간이 짧아진다.
- 실패했을 때 재처리할 범위가 작아진다.
- VACUUM이나 purge가 따라갈 기회를 얻는다.

다만 커밋을 너무 자주 하면 커밋 자체의 비용이 커진다. `1,000건`은 정답이 아니라 예시이며, 배치 크기는 운영 지표와 실제 부하 테스트를 보고 정해야 한다.

---

## 3. 긴 조회 배치가 정리 작업을 막는 경우

쓰기 작업만 MVCC 문제를 만드는 것은 아니다. 긴 조회가 오래된 snapshot을 계속 유지하면 다른 트랜잭션이 만든 과거 버전을 정리하지 못할 수 있다.

```text
대용량 조회 시작
→ 오랫동안 같은 snapshot으로 데이터 처리
→ 그동안 다른 요청이 UPDATE·DELETE
→ 긴 조회가 과거 버전을 계속 필요로 함
→ VACUUM 또는 purge가 과거 버전을 정리하지 못함
```

PostgreSQL에서는 오래 실행되는 쿼리, 오래된 snapshot을 유지하는 격리 수준, 서버 측 cursor 등을 주의해야 한다. undo 방식에서도 같은 논리로 오래된 undo 기록의 purge가 지연될 수 있다.

백만 건을 하나의 트랜잭션에서 계속 읽고 처리하기보다 작은 페이지로 나누는 방법을 고려할 수 있다.

```text
id 1~1,000 조회 → 처리 → 트랜잭션 종료
id 1,001~2,000 조회 → 처리 → 트랜잭션 종료
```

데이터가 처리 중에도 계속 추가되거나 변경된다면 offset 방식보다 keyset pagination이 적합한지도 함께 검토한다.

---

## 4. 같은 API 지연이라도 확인할 곳이 다르다

사용자에게는 모두 “API가 느리다”로 보이지만 MVCC 비용이 나타나는 위치는 DBMS마다 다르다.

### PostgreSQL에서 확인할 것

- 오래 실행 중인 쿼리와 트랜잭션이 있는가?
- `idle in transaction` 세션이 있는가?
- 오래된 snapshot을 유지하는 작업이 있는가?
- dead tuple이 많이 쌓였는가?
- autovacuum이 동작 중이며 변경량을 따라가고 있는가?
- 테이블과 인덱스 bloat가 커졌는가?
- 직전에 대량 `UPDATE`나 `DELETE`가 있었는가?

### MySQL InnoDB 같은 undo 방식에서 확인할 것

- 오래 열린 트랜잭션이 있는가?
- undo history가 계속 증가하는가?
- purge가 변경 속도를 따라가지 못하는가?
- undo tablespace가 커지고 있는가?
- 큰 트랜잭션이 rollback 중인가?

> PostgreSQL에서 dead tuple이 보이지 않는다고 다른 DBMS의 버전 관리 비용이 없는 것은 아니다. 그 DBMS가 과거 버전을 어디에 보존하고 어떻게 정리하는지 확인해야 한다.

---

## 5. 큰 트랜잭션을 취소한 경우

대량 수정 작업이 잘못되어 취소했더라도 DB의 후속 작업은 즉시 끝나지 않을 수 있다.

### Undo 방식

이미 변경한 내용이 많다면 undo를 역순으로 적용하여 이전 상태로 되돌리는 데 시간이 걸릴 수 있다.

```text
대량 변경 수행
→ 취소 요청
→ undo를 이용한 rollback
→ rollback이 장시간 계속될 수 있음
```

DB 세션을 종료했더라도 복구 작업과 리소스 사용이 바로 사라지지 않을 수 있다.

### PostgreSQL

실패한 트랜잭션이 만든 tuple은 가시성 규칙에 따라 다른 트랜잭션에서 보이지 않게 할 수 있다. 하지만 실패한 tuple이 사용한 물리적 공간은 나중에 VACUUM이 정리해야 한다.

```text
트랜잭션 취소
→ 변경 내용은 논리적으로 보이지 않음
→ 실패한 tuple은 테이블에 남을 수 있음
→ VACUUM이 나중에 공간을 정리
```

따라서 undo 방식은 취소 시 되돌리는 시간이 크게 보일 수 있고, PostgreSQL은 취소 후 남은 공간을 청소하는 비용이 크게 보일 수 있다.

---

## ORM 코드 한 줄 뒤에서 일어나는 일

JPA를 사용하면 애플리케이션 코드가 간단해서 DB 비용을 놓치기 쉽다.

```java
users.forEach(User::deactivate);
```

코드에서는 한 줄이지만 DB 내부에서는 다음 작업이 일어날 수 있다.

```text
수십만 건 UPDATE
→ row 버전 수십만 개 생성
→ 인덱스 변경
→ WAL 또는 redo 기록
→ dead tuple 또는 undo 생성
→ 나중에 VACUUM 또는 purge
```

ORM은 객체와 SQL 사이의 작업을 편하게 해주지만, SQL 실행 횟수와 DB 내부의 MVCC 비용까지 없애주지는 않는다.

---

## 주니어가 가져갈 실무 습관

- 트랜잭션 범위를 가능한 한 짧게 유지한다.
- `@Transactional` 안에 외부 API 호출이나 긴 연산을 무심코 넣지 않는다.
- 대량 `UPDATE`와 `DELETE`는 작은 배치로 나눈다.
- 많은 데이터를 하나의 트랜잭션에서 계속 읽고 처리하지 않는다.
- 장애가 나면 느린 SQL만 보지 말고 오래 열린 트랜잭션도 확인한다.
- PostgreSQL을 사용한다면 dead tuple, autovacuum, bloat의 의미를 이해한다.
- MySQL InnoDB를 사용한다면 undo와 purge 지연의 의미를 이해한다.
- 데이터 보정 작업을 시작하기 전에 실패 시 rollback 비용을 생각한다.
- ORM이 DB 내부 비용을 없애준다고 생각하지 않는다.

## 실무에서 던질 질문 5개

배치 작업을 설계하거나 DB 장애를 조사할 때 다음 질문부터 확인한다.

1. 이 트랜잭션은 얼마나 오래 열려 있는가?
2. 한 번에 몇 건을 읽거나 변경하는가?
3. 실패하면 rollback 범위와 시간은 얼마나 되는가?
4. 이 DBMS는 과거 버전을 어디에 쌓고 누가 정리하는가?
5. 지금 VACUUM 또는 purge가 긴 트랜잭션 때문에 지연되고 있는가?

> [!tip] 면접에서 말한다면
> MVCC 구현 세부사항을 외우는 데서 끝내지 않고, 긴 트랜잭션이 PostgreSQL에서는 VACUUM과 bloat에, undo 방식에서는 undo 증가와 purge 지연에 영향을 줄 수 있다고 설명한다. 그래서 트랜잭션 범위를 짧게 유지하고 대량 작업을 작은 단위로 나누며, 장애 시 DBMS에 맞는 버전 정리 상태를 확인한다고 연결하면 좋다.

## 복습 질문

- [ ] 외부 API 호출을 긴 DB 트랜잭션 안에 넣으면 어떤 문제가 생길 수 있는가?
- [ ] 대량 `UPDATE`나 `DELETE`를 작은 배치로 나누면 어떤 장점과 비용이 있는가?
- [ ] 쓰기 작업이 아닌 긴 조회도 VACUUM이나 purge를 지연시킬 수 있는 이유는 무엇인가?
- [ ] PostgreSQL과 undo 방식 DBMS에서 API 지연이 발생했을 때 각각 어떤 버전 정리 상태를 확인해야 하는가?
- [ ] ORM 코드가 간단해 보여도 DB 내부의 MVCC 비용을 확인해야 하는 이유는 무엇인가?

## 백지 복습 답변

### 1. 외부 API 호출을 긴 DB 트랜잭션 안에 넣으면 어떤 문제가 생길 수 있는가?

내 답:

### 2. 대량 `UPDATE`나 `DELETE`를 작은 배치로 나누면 어떤 장점과 비용이 있는가?

내 답:

### 3. 쓰기 작업이 아닌 긴 조회도 VACUUM이나 purge를 지연시킬 수 있는 이유는 무엇인가?

내 답:

### 4. PostgreSQL과 undo 방식 DBMS에서 API 지연이 발생했을 때 각각 어떤 버전 정리 상태를 확인해야 하는가?

내 답:

### 5. ORM 코드가 간단해 보여도 DB 내부의 MVCC 비용을 확인해야 하는 이유는 무엇인가?

내 답:

## 한 줄 회고

- 헷갈렸던 점:
