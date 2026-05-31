---
tags:
  - backend
  - mysql
  - redis
  - concurrency
---

# Shopify 재고 예약 시스템과 MySQL SKIP LOCKED

## 한 줄 정리

Shopify의 재고 예약 시스템 전환은 "Redis가 느려서 MySQL로 바꿨다"가 아니라, 예약 상태와 최종 재고 원장을 같은 트랜잭션 경계 안에서 더 정확하게 다루기 위한 설계 변경이다.

## 기존 Redis 시스템이 하던 일

기존 구조에서 Redis는 단순 조회 캐시라기보다 결제 중 재고를 잠깐 잡아두는 예약 계층에 가까웠다.

```text
MySQL
→ 최종 재고 원장
→ 결제 완료 후 실제 재고를 확정 차감하는 곳

Redis
→ 결제 시작 시 몇 분 동안 재고를 임시 선점하는 곳
→ DECR / INCR 같은 빠른 카운터 연산으로 예약 수량을 관리
```

흐름은 대략 다음과 같다.

```text
상품 재고 10개

사용자 A 결제 시작
→ Redis 수량 10에서 9로 감소
→ A가 1개 예약 중

사용자 B 결제 시작
→ Redis 수량 9에서 8로 감소
→ B가 1개 예약 중

A 결제 완료
→ MySQL 최종 재고 원장 차감
→ Redis 예약 상태 정리

B 결제 실패 또는 시간 만료
→ Redis 수량 복구
```

> [!note]
> Redis를 "캐시"라고만 보면 헷갈린다. 여기서는 MySQL 데이터를 빠르게 읽기 위한 복사본이라기보다, 결제 가능 여부에 직접 영향을 주는 쓰기 경로의 핵심 상태 저장소였다.

## 왜 최종 재고 원장을 바로 잠그지 않았을까

결제는 짧은 DB 작업 하나로 끝나지 않는다.

사용자는 배송지와 결제 수단을 확인하고, 외부 결제 서비스 응답을 기다리고, 중간에 이탈할 수도 있다. 이 시간 동안 MySQL 트랜잭션을 열고 최종 재고 원장을 잠그면 문제가 커진다.

```text
나쁜 방식

BEGIN;
재고 row 잠금;
사용자 결제 진행 대기;
외부 결제 서비스 응답 대기;
성공하면 재고 차감;
COMMIT;
```

이 방식은 다음 부담을 만든다.

- 재고 row 락을 오래 잡는다.
- DB 커넥션을 오래 점유한다.
- 같은 상품을 사려는 다른 요청이 줄줄이 기다릴 수 있다.
- 결제 실패, 타임아웃, 장애 상황에서 복구가 복잡해진다.

그래서 기존 Redis 방식은 이런 타협이었다.

```text
결제 시작
→ Redis에서 빠르게 예약
→ MySQL 트랜잭션은 오래 열지 않음

결제 완료
→ 짧은 MySQL 트랜잭션으로 최종 재고 차감

결제 실패 또는 만료
→ Redis 예약 해제
```

> [!summary]
> 최종 재고 원장을 결제 시간 내내 잠글 수 없으니, 앞단에 빠른 예약 계층을 두고 MySQL에는 결제 완료 시점의 확정 변경만 반영하려는 설계였다.

## Redis 구조의 핵심 한계

문제는 Redis의 예약 상태와 MySQL의 최종 재고 원장이 서로 다른 시스템에 있다는 점이다.

```text
Redis 예약 성공
MySQL 원장 차감 실패

또는

MySQL 원장 차감 성공
Redis 예약 정리 실패
```

이런 중간 실패가 생기면 두 시스템의 상태가 어긋날 수 있다.

- 오버셀: 실제로는 같은 재고가 두 번 팔리는 문제
- 언더셀: 재고가 있는데도 예약 상태가 꼬여 판매하지 못하는 문제

> [!warning]
> Redis와 MySQL을 완전히 하나의 ACID 트랜잭션처럼 묶기 어렵기 때문에, 정확성이 중요한 재고 시스템에서는 이 경계가 위험해진다.

## MySQL로 옮긴 방향

Shopify가 선택한 방향은 재고를 수량 컬럼 하나로만 보지 않고, 판매 가능한 단위 하나를 하나의 행으로 표현하는 방식이다.

```text
수량 컬럼 방식

item_id = 100
available_quantity = 5
```

```text
판매 단위별 row 방식

item_id = 100, unit_id = 1, status = available
item_id = 100, unit_id = 2, status = available
item_id = 100, unit_id = 3, status = available
item_id = 100, unit_id = 4, status = available
item_id = 100, unit_id = 5, status = available
```

이렇게 하면 한 상품의 수량 숫자 하나를 모두가 같이 두드리는 구조가 아니라, 아직 예약되지 않은 행을 각 요청이 하나씩 가져가는 구조로 만들 수 있다.

## SKIP LOCKED의 의미

`SKIP LOCKED`는 이미 다른 트랜잭션이 잠근 행을 기다리지 않고 건너뛰게 해주는 기능이다.

```sql
SELECT *
FROM inventory_units
WHERE item_id = 100
  AND status = 'available'
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

의미는 다음과 같다.

```text
예약 가능한 재고 행을 찾는다.
내가 선택한 행은 수정할 것이므로 잠근다.
이미 다른 트랜잭션이 잠근 행은 기다리지 않고 건너뛴다.
```

동시에 두 사용자가 같은 상품을 결제하려고 하면 이런 식으로 동작한다.

```text
사용자 A
→ unit 1을 FOR UPDATE로 잠금

사용자 B
→ unit 1은 이미 잠겨 있으므로 SKIP
→ unit 2를 잠금
```

> [!tip]
> `SKIP LOCKED`는 락을 무시하는 기능이 아니다. 락을 제대로 사용하되, 이미 남이 잡은 행 때문에 기다리지 않고 다른 가능한 행을 찾게 해주는 기능이다.

## 예약 pool과 보충

단위당 1행 방식은 이해하기 쉽지만, 실제 재고를 전부 row로 펼치면 테이블이 너무 커질 수 있다.

```text
재고 50,000개
위치 10개
= 500,000행
```

그래서 실제 재고가 많더라도 예약에 바로 사용할 수 있는 row는 item/location 조합당 일정 크기로 제한할 수 있다.

```text
실제 재고: 50,000개
예약용 available pool: 최대 1,000행
나머지 재고: 원장에는 있지만 아직 예약용 row로 펼치지 않음
```

즉 창고에 상품이 50,000개 있어도 계산대 앞 진열대에는 1,000개만 꺼내두는 것과 비슷하다. 진열대가 비면 창고에서 다시 꺼내온다.

> [!note]
> pool 크기 제한은 "1,000개까지만 팔 수 있다"는 뜻이 아니다. 예약에 바로 사용할 row를 최대 1,000개 정도로 유지해서 테이블 크기와 스캔 비용을 제어한다는 뜻이다.

## Thundering herd 문제

pool이 비었을 때 여러 결제 요청이 동시에 들어오면 모두가 "내가 보충해야겠다"고 판단할 수 있다.

```text
현재 pool: 0개
필요한 보충: 1,000개
동시에 들어온 요청: 100개
```

제어가 없으면 다음처럼 중복 보충이 발생할 수 있다.

```text
A 요청: pool 0개네. 1,000개 넣자.
B 요청: pool 0개네. 1,000개 넣자.
C 요청: pool 0개네. 1,000개 넣자.
...
```

원래는 1,000행만 추가하면 되는데, 많은 요청이 동시에 깨어나 같은 일을 하려고 몰리는 것이다. 이런 현상을 thundering herd라고 볼 수 있다.

> [!warning]
> thundering herd는 하나의 조건 변화나 이벤트를 보고 너무 많은 작업자가 동시에 깨어나 같은 작업을 중복으로 수행하려는 문제다.

## 보충은 대표 row를 잠가 한 트랜잭션만 수행

보충을 한 트랜잭션만 하게 하려면, 예약 unit 1,000개 중 아무 행이나 잠그는 것이 아니다. 보통은 해당 item/location의 pool 상태를 대표하는 별도 row를 두고 그 row를 잠근다.

```text
inventory_units
→ 실제로 예약되는 재고 단위들

inventory_pool_state
→ 특정 item/location pool의 상태와 보충 권한을 조율하는 대표 row
```

예를 들면 다음과 같은 대표 테이블을 둘 수 있다.

```text
inventory_pool_state

item_id | location_id | pool_size | max_pool_size
100     | 1           | 1000      | 1000
```

보충할 때는 이 대표 row를 `FOR UPDATE`로 잠근다.

```sql
BEGIN;

SELECT *
FROM inventory_pool_state
WHERE item_id = 100
  AND location_id = 1
FOR UPDATE;

-- lock을 잡은 뒤 pool 상태를 다시 확인
-- 아직 부족하면 inventory_units에 available row를 추가

COMMIT;
```

이 row를 잠그는 의미는 "상품 100번, 위치 1번의 pool 보충 담당권을 내가 잠깐 가져간다"에 가깝다.

```text
A 요청
→ inventory_pool_state(100, 1) row 잠금 성공
→ A만 보충 수행

B 요청
→ 같은 대표 row 잠금 시도
→ A가 잡고 있으므로 기다림

C 요청
→ 같은 대표 row 잠금 시도
→ A가 잡고 있으므로 기다림
```

A가 보충을 끝내고 커밋하면, B나 C는 락을 얻은 뒤 pool 상태를 다시 확인한다. 이미 A가 채웠다면 중복 보충을 하지 않는다.

> [!summary]
> `SKIP LOCKED`는 여러 요청이 서로 다른 재고 단위를 동시에 예약하게 해주는 장치다. 반면 pool 보충용 대표 row lock은 보충 작업만 한 트랜잭션이 수행하게 만드는 장치다.

## 복합 기본 키와 락 수

InnoDB에서는 기본 키가 단순한 식별자 역할만 하지 않는다. 테이블의 실제 데이터는 기본 키 기준의 클러스터드 인덱스에 저장된다.

초기 구조가 다음처럼 `id`만 기본 키로 가진 형태였다고 생각해볼 수 있다.

```sql
CREATE TABLE inventory_units (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  shop_id BIGINT,
  inventory_item_id BIGINT,
  inventory_group_id BIGINT,
  status VARCHAR(20)
);

CREATE INDEX idx_reserve_lookup
ON inventory_units (shop_id, inventory_item_id, inventory_group_id, status);
```

예약 쿼리는 보통 `shop_id`, `inventory_item_id`, `inventory_group_id` 같은 조건으로 가용 row를 찾는다.

```sql
SELECT *
FROM inventory_units
WHERE shop_id = ?
  AND inventory_item_id = ?
  AND inventory_group_id = ?
  AND status = 'available'
LIMIT 3
FOR UPDATE SKIP LOCKED;
```

이때 기본 키가 `id` 하나뿐이면 MySQL은 보조 인덱스에서 조건에 맞는 row를 찾고, 다시 기본 키를 통해 실제 row 위치로 찾아간다.

```text
보조 인덱스
→ (shop_id, inventory_item_id, inventory_group_id, status)로 검색
→ primary key id를 얻음

클러스터드 인덱스
→ id로 실제 row를 찾음
```

`FOR UPDATE`는 찾은 row를 수정할 예정이라고 보고 잠근다. 이 과정에서 보조 인덱스와 기본 키 쪽에 모두 락이 걸릴 수 있다.

```text
예약 row 1개
→ 보조 인덱스 엔트리 락
→ 기본 키 엔트리 락
→ 총 2개 락
```

Shopify는 기본 키를 예약 조회 조건에 맞게 복합 키로 구성했다.

```sql
PRIMARY KEY (
  shop_id,
  inventory_item_id,
  inventory_group_id,
  id
)
```

이렇게 하면 예약 쿼리의 핵심 필터 컬럼이 기본 키 경로에 포함된다. 보조 인덱스를 거쳐 기본 키로 다시 찾아가는 부담과 락 수를 줄일 수 있다.

> [!note]
> InnoDB에서 기본 키 설계는 단순 조회 성능뿐 아니라 어떤 인덱스 엔트리를 잠그는지에도 영향을 준다. 초당 수천 건의 예약에서는 row 하나당 락이 2개인지 1개인지가 처리량에 직접 영향을 줄 수 있다.

## READ COMMITTED와 갭 락

MySQL InnoDB의 기본 격리 수준은 보통 `REPEATABLE READ`다. 이 격리 수준에서는 `SELECT ... FOR UPDATE`가 실제 row뿐 아니라 row와 row 사이의 빈 공간인 gap까지 잠글 수 있다.

문제는 pool이 비어 있을 때 더 잘 드러난다.

```sql
SELECT *
FROM inventory_units
WHERE item_id = 100
  AND status = 'available'
FOR UPDATE SKIP LOCKED;
```

available row가 하나도 없으면 아무 row도 안 잡은 것처럼 보일 수 있다. 하지만 `REPEATABLE READ`에서는 해당 조건 범위에 새 row가 끼어들지 못하게 gap lock이 잡힐 수 있다.

```text
예약 트랜잭션
→ available row를 찾음
→ 없음
→ 그런데 빈 범위에 gap lock을 잡음

보충 트랜잭션
→ available row를 INSERT해서 pool을 채우려 함
→ gap lock 때문에 INSERT가 막힘
```

즉 pool이 비어서 보충해야 하는데, pool이 비었는지 확인한 트랜잭션이 오히려 보충 INSERT를 막는 상황이 생길 수 있다.

> [!warning]
> `SELECT ... FOR UPDATE`는 읽기처럼 보이지만 잠금 읽기다. 특히 `REPEATABLE READ`에서는 존재하는 row뿐 아니라 "여기에 새 row가 들어올 수 있는 빈 공간"까지 잠글 수 있다.

이 문제를 줄이기 위해 해당 트랜잭션의 격리 수준을 `READ COMMITTED`로 바꿀 수 있다.

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

`READ COMMITTED`에서는 매번 커밋된 데이터를 기준으로 읽고, InnoDB의 gap lock 사용 방식도 줄어든다. 그래서 빈 pool을 확인하는 트랜잭션이 보충 트랜잭션의 INSERT를 덜 막게 된다.

```text
REPEATABLE READ
→ 일관된 반복 읽기를 위해 gap까지 잠그는 경우가 있음
→ 빈 pool 확인이 INSERT 보충을 막을 수 있음

READ COMMITTED
→ 커밋된 데이터 기준으로 읽음
→ gap lock 부담이 줄어듦
→ 보충 INSERT가 진행되기 쉬움
```

> [!summary]
> 여기서 격리 수준 변경은 단순 성능 튜닝이 아니다. `SELECT ... FOR UPDATE SKIP LOCKED`와 pool 보충 INSERT가 동시에 일어날 때 어떤 범위를 잠글지 바꾸는 동시성 설계 결정이다.

## 일관된 락 순서와 데드락

데드락은 서로가 서로의 락을 기다리면서 아무 트랜잭션도 진행하지 못하는 상태다.

예를 들어 다음 두 테이블을 함께 다룬다고 생각할 수 있다.

```text
reservation_units
→ 예약 가능한 unit row들이 있는 테이블

reserved_quantities
→ 특정 예약이나 주문이 몇 개를 잡았는지 기록하는 테이블
```

문제는 `reserve`와 `claim`이 같은 테이블들을 서로 다른 순서로 접근할 때 생길 수 있다.

```text
reserve
→ reserved_quantities INSERT
→ reservation_units DELETE

claim
→ reservation_units 관련 처리
→ reserved_quantities DELETE
```

이런 순서 차이는 다음과 같은 순환 대기를 만들 수 있다.

```text
트랜잭션 A reserve
→ reserved_quantities 쪽 락을 잡음
→ reservation_units 쪽 락이 필요함

트랜잭션 B claim
→ reservation_units 쪽 락을 잡음
→ reserved_quantities 쪽 락이 필요함

A는 B가 가진 락을 기다림
B는 A가 가진 락을 기다림
```

해결책은 모든 트랜잭션이 테이블을 같은 순서로 만지게 하는 것이다.

```text
표준 순서

1. reservation_units 먼저 처리
2. reserved_quantities 나중에 처리
```

Shopify 사례에서는 `reserve`도 항상 units 테이블을 먼저 `DELETE`하고, 그다음 `reserved_quantities`에 `INSERT`하도록 순서를 맞췄다.

> [!tip]
> 데드락을 줄이는 기본 방법 중 하나는 "항상 같은 순서로 락을 잡기"다. 기다림 자체를 완전히 없애지는 못해도, 서로 반대 순서로 물고 늘어지는 순환 대기를 줄일 수 있다.

백엔드 코드의 락 순서 문제와도 비슷하다.

```java
// 위험한 패턴
void methodA() {
    lock(user);
    lock(order);
}

void methodB() {
    lock(order);
    lock(user);
}
```

두 메서드가 동시에 실행되면 한쪽은 `user`를 잡고 `order`를 기다리고, 다른 쪽은 `order`를 잡고 `user`를 기다릴 수 있다.

```java
// 더 나은 패턴
void methodA() {
    lock(user);
    lock(order);
}

void methodB() {
    lock(user);
    lock(order);
}
```

DB 트랜잭션에서도 원리는 같다. 여러 테이블이나 여러 row를 함께 잠글 수 있다면, 가능한 한 접근 순서를 표준화해야 한다.

## UNION ALL 배치와 라운드트립 감소

장바구니에는 보통 여러 라인 아이템이 들어간다.

```text
상품 A 2개
상품 B 1개
상품 C 3개
```

각 상품을 따로 예약하면 애플리케이션 서버와 DB 사이의 왕복이 늘어난다.

```text
앱 → DB: 상품 A 예약
DB → 앱

앱 → DB: 상품 B 예약
DB → 앱

앱 → DB: 상품 C 예약
DB → 앱
```

쿼리 하나하나가 빠르더라도, 부하 상황에서는 이 라운드트립 수가 전체 레이턴시에 영향을 준다.

그래서 여러 라인 아이템의 예약 쿼리를 `UNION ALL`로 묶어 한 번에 보낼 수 있다.

```sql
(
  SELECT *
  FROM reservation_units
  WHERE item_id = 100
  LIMIT 2
  FOR UPDATE SKIP LOCKED
)
UNION ALL
(
  SELECT *
  FROM reservation_units
  WHERE item_id = 200
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
UNION ALL
(
  SELECT *
  FROM reservation_units
  WHERE item_id = 300
  LIMIT 3
  FOR UPDATE SKIP LOCKED
);
```

실제 SQL은 MySQL 문법과 최적화에 맞춰 더 복잡할 수 있지만, 의도는 단순하다.

```text
장바구니의 여러 상품 예약을
DB에 여러 번 요청하지 않고
한 번의 요청으로 묶어 처리한다.
```

여기서 `UNION`이 아니라 `UNION ALL`을 쓰는 이유도 중요하다.

```text
UNION
→ 중복 제거를 수행할 수 있음
→ 정렬이나 임시 작업 비용이 생길 수 있음

UNION ALL
→ 결과를 그대로 이어 붙임
→ 중복 제거 없음
→ 더 가볍다
```

예약 대상은 라인 아이템별로 분리되어 있고 중복 제거가 목적이 아니므로 `UNION ALL`이 더 적합하다.

> [!summary]
> 일관된 락 순서는 데드락 가능성을 줄이고, `UNION ALL` 배치는 DB 라운드트립을 줄인다. 둘 다 특별한 마법이라기보다 대규모 트래픽에서 기본기를 세밀하게 지킨 최적화에 가깝다.

## 실제 병목은 쿼리가 아니라 커넥션 점유

운영 환경에서는 이상한 신호가 보일 수 있다.

```text
목표 처리량까지 못 감
P90 레이턴시는 괜찮음
CPU도 아직 여유 있음
예약 쿼리도 이미 최적화됨
```

이런 상황에서는 느린 쿼리 하나만 보면 원인을 놓칠 수 있다. Shopify 사례에서 실제 병목은 쿼리 실행 시간보다 DB 커넥션을 오래 붙잡는 코드 쪽에 가까웠다.

```text
커넥션 획득
BEGIN
쿼리 10ms
애플리케이션 로직 200ms
다른 작업 대기
또 쿼리 10ms
COMMIT
커넥션 반납
```

DB가 실제로 일한 시간은 짧아도, 트랜잭션이 열려 있는 동안 커넥션은 계속 점유된다. 커넥션 풀이 고갈되면 다른 요청은 DB가 놀고 있어도 커넥션을 얻지 못해 기다린다.

> [!note]
> 쿼리가 빠르더라도 트랜잭션 범위가 크거나 커넥션을 잡은 채 애플리케이션 로직을 오래 수행하면 처리량의 천장이 생길 수 있다.

비유하면 계산대 직원은 한가한데, 손님들이 계산대 앞에서 지갑과 쿠폰을 찾느라 계산대를 오래 점유하는 상황과 비슷하다.

## 커넥션 가시성 확보

문제를 찾기 위해 SQL에 비즈니스 프로세스 식별 태그를 붙였다.

```sql
SELECT ...
FROM ...
WHERE ...
/* conn_tag:checkout_completion */
```

이 태그를 ProxySQL 레이어에서 파싱하고, 호출자별 커넥션 점유 시간을 집계하면 다음을 볼 수 있다.

```text
checkout_completion이 커넥션을 얼마나 오래 잡는가?
inventory_reserve가 커넥션을 얼마나 오래 잡는가?
payment_finalize가 커넥션을 얼마나 오래 잡는가?
order_create가 커넥션을 얼마나 오래 잡는가?
```

이전에는 "MySQL 커넥션이 부족하다" 정도로만 보이던 문제가, 태그를 붙인 뒤에는 어떤 비즈니스 흐름이 커넥션을 오래 잡는지 보이게 된다.

발견된 문제는 예약 쿼리 자체가 아니라 체크아웃 경로의 다른 코드들이 커넥션을 필요 이상으로 오래 점유하고 있었다는 점이었다.

가능한 개선 방향은 다음과 같다.

- 불필요한 DB read 제거
- 프라이머리 DB가 아니어도 되는 read 분리
- 트랜잭션 범위 축소
- 트랜잭션 안에서 하지 않아도 되는 작업을 밖으로 이동
- 중복 쿼리 제거
- 오래된 InnoDB 동시성 설정 재검토

> [!summary]
> DB 병목은 항상 느린 쿼리로만 나타나지 않는다. 커넥션 점유 시간, 트랜잭션 범위, 프라이머리 DB read, 오래된 DB 설정이 처리량의 한계가 될 수 있다.

## Shadow Mode 전환

정확성이 중요한 시스템은 한 번에 교체하지 않는 편이 안전하다.

Shopify는 Redis에서 MySQL로 즉시 source of truth를 바꾸지 않고, 한동안 Shadow Mode로 두 시스템을 병렬 운영했다.

```text
전환 전

Redis
→ source of truth
→ 실제 고객 요청의 성공/실패 판단 기준

MySQL
→ shadow system
→ 같은 예약을 기록하지만 아직 고객 결과를 결정하지 않음
```

예약 요청이 들어오면 양쪽에 모두 기록한다.

```text
사용자 결제 시작
→ Redis에 예약 기록
→ MySQL에도 같은 예약 기록
→ 고객에게는 Redis 결과 기준으로 응답
```

이렇게 하면 실제 운영 트래픽에서 MySQL의 정확성과 성능을 검증할 수 있다.

```text
Redis 결과: 예약 성공
MySQL 결과: 예약 성공
→ 정상

Redis 결과: 예약 성공
MySQL 결과: 예약 실패
→ 고객에게는 Redis 기준으로 성공 처리
→ 내부적으로 불일치 기록 및 원인 분석
```

> [!tip]
> Shadow Mode는 새 시스템에 실제 트래픽을 흘리지만, 고객에게 영향을 주는 최종 판단은 기존 시스템이 계속 담당하게 하는 전환 방식이다.

## 인플라이트 예약과 킬 스위치

인플라이트 예약은 이미 결제 진행 중이라 몇 분 동안 살아 있는 예약이다.

만약 어느 순간 Redis를 바로 끄고 MySQL로 전환하면 다음 문제가 생긴다.

```text
Redis에만 존재하는 기존 예약
MySQL에는 아직 없는 예약
```

이 상태에서는 결제 중인 예약을 MySQL로 따로 옮기는 마이그레이션이 필요하다. 재고 예약 마이그레이션은 오버셀과 언더셀 위험이 있어서 까다롭다.

Shadow Mode로 한동안 양쪽에 동시에 쓰고 있었다면 상황이 달라진다.

```text
Redis에도 예약 있음
MySQL에도 같은 예약 있음
```

그래서 source of truth를 MySQL로 바꾸는 순간에도 인플라이트 예약을 별도로 대규모 마이그레이션할 필요가 줄어든다.

전환 후에도 Redis를 바로 버리지 않고 이중 쓰기를 유지하면 킬 스위치를 둘 수 있다.

```text
전환 전
→ Redis 기준, MySQL shadow

전환 후
→ MySQL 기준, Redis도 계속 최신 상태 유지

문제 발생
→ 킬 스위치로 Redis 기준으로 복귀
```

Redis가 계속 최신 상태를 유지해야만 문제가 생겼을 때 되돌아갈 수 있다.

## 점진 롤아웃

전환은 전체 트래픽에 한 번에 적용하지 않고 파드 단위로 점진적으로 진행할 수 있다.

```text
1단계: 낮은 트래픽 파드
2단계: 일반 트래픽 파드
3단계: 높은 트래픽 머천트
4단계: 최고 볼륨 머천트
```

영향이 작은 구간부터 켜고, 지표와 불일치를 관찰한 뒤 점점 범위를 넓히는 방식이다.

> [!summary]
> 큰 저장소 전환의 핵심은 새 시스템을 바로 믿는 것이 아니라, 기존 시스템을 기준으로 둔 채 실제 트래픽에서 그림자처럼 검증하고, 전환 후에도 되돌아갈 길을 남기는 것이다.

## PostgreSQL보다 MySQL이 유리할 수 있다는 댓글의 의미

"Redis의 대체라면 MySQL이 PostgreSQL보다 유리할 수 있다"는 말은 MySQL이 항상 PostgreSQL보다 좋다는 뜻은 아니다.

이 사례의 workload가 짧게 살아 있는 row를 자주 만들고 지우는 패턴에 가깝다는 점을 말한 것이다.

```text
예약용 unit row 생성
→ 예약됨
→ claim 또는 만료
→ row 삭제 또는 이동
→ pool 보충으로 다시 row 생성
```

이런 식으로 행이 계속 생기고 사라지는 정도를 row churn이라고 볼 수 있다.

PostgreSQL은 MVCC 구조상 `DELETE`나 `UPDATE`가 발생해도 기존 row가 즉시 물리적으로 사라지는 것이 아니라 dead tuple로 남고, 나중에 `VACUUM`이 정리한다.

```text
PostgreSQL

INSERT row
DELETE row
→ row가 바로 파일에서 사라지는 것이 아님
→ dead tuple로 남음
→ VACUUM이 나중에 청소
```

row churn이 심하면 다음 운영 부담이 커질 수 있다.

- dead tuple 증가
- autovacuum 부하 증가
- table bloat 증가
- index bloat 증가
- 쿼리 성능 흔들림
- vacuum 튜닝 필요

MySQL InnoDB도 MVCC가 있고 undo log, purge lag, secondary index maintenance 같은 비용이 있다. 따라서 MySQL이면 이 문제가 사라지는 것은 아니다.

다만 이 사례처럼 Redis 대체에 가까운 임시 예약 슬롯 저장소라면, PostgreSQL의 vacuum/bloat 관리 부담을 의식할 수 있다. Shopify처럼 MySQL 운영 경험과 도구가 깊게 쌓인 조직에서는 MySQL이 더 자연스러운 선택일 수 있다.

> [!summary]
> 댓글의 요지는 "PostgreSQL이 나쁘다"가 아니라, 고빈도 INSERT/DELETE가 많은 예약 슬롯 테이블에서는 PostgreSQL의 dead tuple과 VACUUM 운영 부담이 커질 수 있고, Shopify의 기존 MySQL 운영 맥락에서는 MySQL이 더 맞았을 수 있다는 뜻이다.

## SQL 프론트엔드가 있는 키-밸류 스토어라는 말

"MySQL을 SQL 프론트엔드가 있는 키-밸류 스토어라고 한다"는 말은 농담 섞인 표현이지만, InnoDB의 사용 패턴을 잘 설명한다.

키-밸류 스토어는 보통 다음 모델이다.

```text
key → value

user:1 → { name: "mogi" }
inventory:100:1 → { status: "available" }
```

MySQL도 primary key 중심으로 접근하면 비슷한 방식으로 볼 수 있다.

```sql
SELECT *
FROM inventory_units
WHERE shop_id = 1
  AND inventory_item_id = 100
  AND inventory_group_id = 3
  AND id = 55;
```

여기서 복합 기본 키가 key 역할을 한다.

```text
key
→ (shop_id, inventory_item_id, inventory_group_id, id)

value
→ status, expires_at, reservation_id 같은 나머지 컬럼
```

InnoDB는 기본 키 기준으로 데이터를 클러스터링해서 저장한다. 그래서 primary key나 그 prefix range로 row를 빠르게 찾고, 짧은 트랜잭션으로 상태를 바꾸는 패턴에 잘 맞는다.

Shopify 사례에서도 복잡한 분석 쿼리보다 다음에 가깝다.

```text
특정 shop/item/group 범위에서
available unit row 몇 개를 찾고
그 row를 reserved 또는 claimed 상태로 바꾼다
```

즉 MySQL을 복잡한 관계형 질의 엔진이라기보다, SQL과 트랜잭션이 붙어 있는 고성능 primary key/range 기반 row store처럼 운용한 것이다.

> [!note]
> 이 말이 MySQL이 진짜 Redis 같은 단순 키-밸류 저장소라는 뜻은 아니다. MySQL은 여전히 SQL, 트랜잭션, 인덱스, 락, 격리 수준, JOIN, 제약 조건을 제공한다. 다만 접근 패턴을 단순하게 제한하면 키-밸류 스토어처럼 예측 가능한 고성능 read/write 저장소로 활용할 수 있다는 뜻이다.

## WITH (NOLOCK)과 다른 점

MSSQL의 `WITH (NOLOCK)`과 `SKIP LOCKED`는 둘 다 락 대기를 줄이는 것처럼 보이지만 목적이 다르다.

| 기능 | 의미 | 재고 예약에 적합한가 |
|---|---|---|
| `SKIP LOCKED` | 이미 잠긴 행은 건너뛰고, 내가 가져간 행은 잠근다 | 적합 |
| `WITH (NOLOCK)` | 잠금 대기를 줄이기 위해 커밋되지 않은 데이터까지 읽을 수 있다 | 위험 |

`WITH (NOLOCK)`은 정확하지 않은 중간 상태를 읽을 수 있다. 그래서 "재고가 남았는가?"처럼 돈과 주문에 직접 연결되는 판단에는 위험하다.

반대로 `SKIP LOCKED`는 다음에 가깝다.

```text
누가 이미 집어 든 물건은 건너뛰고,
아직 선반에 남은 물건 중 하나를 내가 집어 들어 이름표를 붙인다.
```

즉 재고 예약에서 필요한 것은 "락을 대충 무시하고 읽기"가 아니라 "같은 재고 단위를 두 요청이 동시에 잡지 못하게 하기"다.

## 실무에서 가져갈 점

`Redis = 빠름`, `MySQL = 느림`으로 단순 비교하면 설계를 잘못 이해하기 쉽다.

중요한 질문은 다음에 가깝다.

- 핵심 불변식은 무엇인가?
- 그 불변식은 어느 저장소와 어느 트랜잭션 경계에서 보장되는가?
- 결제처럼 오래 걸리는 흐름에서 DB 락과 커넥션을 얼마나 오래 잡는가?
- 성능 병목이 정말 쿼리 자체인지, 아니면 커넥션 점유 시간이나 전체 호출 경로인지 확인했는가?

재고 시스템의 핵심 불변식은 "같은 재고 단위가 두 번 팔리면 안 된다"이다. 이 불변식이 Redis와 MySQL 사이에 걸쳐 있으면 중간 실패를 다루기 어려워진다.

> [!summary]
> Shopify 사례의 핵심은 Redis를 없애고 MySQL만 쓰자는 단순한 결론이 아니다. 정확성이 중요한 예약 상태를 최종 원장과 더 가까운 트랜잭션 모델 안으로 가져오고, `SKIP LOCKED` 같은 기능으로 동시성 비용을 제어한 사례로 보는 편이 좋다.

## 복습 질문

- [ ] 기존 시스템에서 Redis는 단순 조회 캐시가 아니라 어떤 역할을 했는가?
- [ ] 결제 시간 동안 MySQL 최종 재고 원장을 계속 잠그면 어떤 문제가 생기는가?
- [ ] Redis 예약 상태와 MySQL 최종 원장이 서로 다른 시스템에 있으면 왜 정합성 문제가 생기는가?
- [ ] `SKIP LOCKED`는 이미 잠긴 행을 어떻게 처리하는가?
- [ ] `SKIP LOCKED`와 MSSQL의 `WITH (NOLOCK)`은 정확성 관점에서 무엇이 다른가?
- [ ] 예약 pool 크기를 제한하는 이유는 무엇인가?
- [ ] thundering herd는 pool 보충 상황에서 어떻게 발생하는가?
- [ ] 보충을 한 트랜잭션만 하게 할 때 왜 실제 unit row가 아니라 대표 row를 잠그는가?
- [ ] InnoDB에서 `id`만 기본 키로 두면 예약 row 하나를 잡을 때 왜 보조 인덱스와 기본 키 양쪽 락이 생길 수 있는가?
- [ ] `REPEATABLE READ`의 gap lock은 pool 보충 INSERT를 어떻게 막을 수 있는가?
- [ ] `READ COMMITTED`로 격리 수준을 바꾸면 이 문제를 왜 줄일 수 있는가?
- [ ] `reserve`와 `claim`이 테이블을 서로 다른 순서로 잠그면 왜 데드락이 생길 수 있는가?
- [ ] 여러 라인 아이템 예약을 `UNION ALL`로 묶으면 어떤 비용을 줄일 수 있는가?
- [ ] 쿼리 자체가 빠른데도 커넥션 점유 시간이 병목이 될 수 있는 이유는 무엇인가?
- [ ] SQL에 비즈니스 프로세스 태그를 붙이면 어떤 관측이 가능해지는가?
- [ ] Shadow Mode에서 새 시스템은 실제 운영 트래픽을 받으면서도 왜 고객 결과를 직접 결정하지 않는가?
- [ ] MySQL 전환 후에도 Redis 이중 쓰기를 유지하면 어떤 롤백 이점이 생기는가?
- [ ] 고빈도 INSERT/DELETE workload에서 PostgreSQL의 VACUUM과 bloat가 왜 운영 부담이 될 수 있는가?
- [ ] MySQL을 "SQL 프론트엔드가 있는 키-밸류 스토어"라고 부르는 말은 어떤 접근 패턴을 가리키는가?

## 한 줄 회고

- 헷갈렸던 점: Redis를 단순 캐시로 본 점, `SKIP LOCKED`를 `WITH (NOLOCK)`처럼 락을 대충 무시하는 기능으로 오해할 수 있다는 점, pool 보충 락이 실제 재고 unit이 아니라 대표 row를 잠그는 방식이라는 점, 기본 키와 격리 수준이 InnoDB의 락 범위에 직접 영향을 준다는 점, 데드락은 쿼리 하나가 아니라 여러 트랜잭션의 락 순서 차이에서 생길 수 있다는 점, 병목은 느린 쿼리가 아니라 커넥션 점유 시간과 전환 운영 방식에서 드러날 수 있다는 점, 그리고 MySQL 선택은 단순 선호가 아니라 row churn과 기존 운영 역량까지 포함한 판단이라는 점.
