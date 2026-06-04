---
tags:
  - backend
  - postgres
  - database
  - concurrency
---

# Postgres 락과 COUNT trigger 동시성 문제

## 한 줄 정리

`COUNT(*)`로 개수를 확인하는 trigger는 단일 요청에서는 제약처럼 보이지만, 같은 대상에 동시 요청이 들어오면 둘 다 같은 개수를 보고 통과할 수 있으므로 검사 전에 같은 비즈니스 단위를 잠가야 한다.

## 왜 공부했나

사이드프로젝트 코드리뷰 중 `shade` 하나에 연결할 수 있는 `color_family`를 최대 2개로 제한하는 DB trigger를 봤다.

의도는 UI 검증이 아니라 DB가 최종 방어선으로 제약을 보장하는 것이었다. 그런데 trigger 내부가 대략 다음 흐름이었다.

```sql
SELECT COUNT(*) INTO v_count
FROM shade_color_families
WHERE shade_id = NEW.shade_id;

IF v_count >= 2 THEN
  RAISE EXCEPTION '...';
END IF;
```

이 코드는 혼자 한 명씩 저장할 때는 잘 막는다. 하지만 동시에 같은 `shade_id`에 다른 `color_family`를 추가하면 제약이 깨질 수 있다.

> [!summary]
> 문제는 trigger가 없다는 것이 아니라, trigger 안의 "검사하고 넣기" 흐름이 동시성 상황에서 원자적으로 보호되지 않는다는 점이다.

## COUNT만으로 부족한 이유

최대 2개 제한이고, 현재 한 `shade`에 이미 `color_family`가 1개 있다고 하자.

```text
현재 상태

shade_id = 10
color_family 개수 = 1
```

이때 요청 A와 요청 B가 거의 동시에 들어오면 다음 일이 가능하다.

```text
1. 요청 A가 color_family 추가 시작
2. 요청 B도 color_family 추가 시작
3. A가 COUNT(*) 조회 -> 현재 1개, 추가 가능
4. B도 COUNT(*) 조회 -> 아직 A가 커밋 전이라 1개로 보일 수 있음
5. A insert 성공
6. B insert 성공
7. 결과적으로 3개가 됨
```

`COUNT(*)`는 읽기만 한다. "내가 확인하는 동안 같은 `shade_id`를 건드리는 다른 요청은 잠깐 기다려"라는 의미가 없다.

> [!warning]
> DB trigger가 있어도 동시성 제어 없이 `COUNT -> INSERT` 흐름만 있으면, 두 트랜잭션이 같은 과거 상태를 보고 동시에 통과할 수 있다.

## 왜 머지 전에 고쳐야 하나

MVP 트래픽에서는 실제로 터질 확률이 낮을 수 있다. 하지만 이 작업의 목적이 "UI 약속"이 아니라 "DB 제약"이라면 기준이 달라진다.

```text
UI 검증
-> 사용자가 화면에서 실수하지 않게 도와주는 장치
-> 우회 가능
-> 동시성까지 완전히 책임지기 어려움

DB 제약
-> 어떤 경로로 쓰기가 들어와도 최종적으로 데이터 불변식을 지키는 장치
-> 관리자 도구, 배치, API, 동시 요청까지 고려해야 함
```

`shade당 color_family 최대 2개`가 정말 DB 불변식이라면, 낮은 확률의 동시 요청에서도 3개가 들어가면 안 된다.

> [!note]
> 코드리뷰에서 중요한 포인트는 "지금 당장 자주 터지냐"보다 "이 PR이 보장한다고 말하는 불변식이 실제로 DB에서 보장되냐"이다.

## SELECT FOR UPDATE로 고치는 방식

가장 읽기 쉬운 방식은 부모 row인 `shades`를 잠그는 것이다.

```sql
PERFORM 1
FROM shades
WHERE id = NEW.shade_id
FOR UPDATE;
```

이 코드를 `COUNT(*)` 전에 둔다.

```sql
IF TG_OP = 'UPDATE' AND NEW.shade_id = OLD.shade_id THEN
  RETURN NEW;
END IF;

PERFORM 1
FROM shades
WHERE id = NEW.shade_id
FOR UPDATE;

SELECT COUNT(*) INTO v_count
FROM shade_color_families
WHERE shade_id = NEW.shade_id;
```

의미는 다음과 같다.

```text
같은 shade_id를 다루는 요청들
-> 같은 부모 shades row를 FOR UPDATE로 잠그려고 함
-> 먼저 잡은 트랜잭션만 진행
-> 나머지는 기다림
-> 먼저 잡은 트랜잭션이 커밋한 뒤 다음 트랜잭션이 다시 COUNT
```

그러면 앞의 예시는 이렇게 바뀐다.

```text
현재 color_family 1개

A가 shades(id=10) row lock 획득
A가 COUNT -> 1개
A가 insert
A가 commit

B가 기다리다가 lock 획득
B가 COUNT -> 이제 2개
B는 trigger에서 차단
```

> [!tip]
> `SELECT ... FOR UPDATE`는 실제로 존재하는 row를 잠근다. 이번 케이스에서는 `shade_id`에 해당하는 부모 row가 있으므로 주니어가 읽기에도 가장 직관적이다.

## advisory lock으로 고치는 방식

Postgres에는 advisory lock도 있다.

```sql
PERFORM pg_advisory_xact_lock(hashtext('shade_color_families'), NEW.shade_id);
```

이 방식은 실제 row를 잠그는 것이 아니라, 개발자가 정한 숫자 key를 기준으로 잠근다.

```text
hashtext('shade_color_families'), NEW.shade_id
-> "shade_color_families 작업 중 shade_id별 잠금"이라는 의미로 개발자가 약속한 key
```

DB는 이 key가 `shade_id`인지, 주문 id인지, 유저 id인지 모른다. 같은 key를 잡으려는 트랜잭션끼리 줄 세워줄 뿐이다.

> [!summary]
> advisory lock은 실제 데이터 row에 직접 묶인 잠금이 아니라, 애플리케이션과 DB 함수 호출이 함께 약속한 이름표 같은 잠금이다.

## 논리적 잠금과 실제 row 잠금

"논리적 잠금"이라는 말은 Postgres 공식 분류라기보다 이해를 돕기 위한 표현이다.

실제 row 잠금은 DB 데이터와 직접 연결된다.

```sql
SELECT *
FROM shades
WHERE id = 10
FOR UPDATE;
```

이 쿼리는 `shades.id = 10` row를 잠근다. DB도 이 잠금이 어떤 row에 걸린 것인지 안다.

반면 advisory lock은 숫자 key에 걸린다.

```sql
SELECT pg_advisory_xact_lock(10);
```

이 잠금의 의미는 DB가 모른다. 개발자가 "10은 shade_id다"라고 약속해서 쓰는 것이다.

| 방식 | 잠그는 대상 | DB가 의미를 아는가 | 예시 |
| --- | --- | --- | --- |
| `SELECT ... FOR UPDATE` | 실제 row | 안다 | `shades.id = 10` |
| table lock | 실제 table | 안다 | `shade_color_families` table |
| advisory lock | 숫자 key | 모른다 | `shade_id = 10`이라고 개발자가 약속 |

> [!warning]
> advisory lock은 유연하지만, key 설계와 사용 위치가 코드에서 명확하지 않으면 나중에 "무엇을 잠그는지" 추적하기 어려워질 수 있다.

## MySQL SELECT FOR UPDATE와 비교

MySQL의 `SELECT ... FOR UPDATE`와 목적은 비슷하다.

```text
내가 확인하고 수정하는 동안
같은 대상에 대한 다른 트랜잭션은 기다리게 한다.
```

Postgres에서도 실제 row를 기준으로 줄 세울 수 있으면 `SELECT ... FOR UPDATE`가 자연스럽다.

```sql
SELECT 1
FROM shades
WHERE id = NEW.shade_id
FOR UPDATE;
```

다만 잠글 row가 없는 경우에는 주의해야 한다. 예를 들어 자식 테이블인 `shade_color_families`에서 현재 0개인 상태라면 잠글 자식 row가 없다. 그래서 이런 경우에는 보통 다음 둘 중 하나를 선택한다.

- 부모 row인 `shades`를 `FOR UPDATE`로 잠근다.
- advisory lock을 `shade_id` 기준으로 잡는다.

이번 케이스는 부모 row가 있으므로 `shades` row를 잠그는 방식이 더 읽기 쉽다.

## 코드리뷰 코멘트로 남기면

```text
현재 trigger는 COUNT(*)로 개수를 확인한 뒤 insert/update를 허용하는 구조라서,
동시에 같은 shade_id에 color_family를 추가하는 트랜잭션이 들어오면 둘 다 기존 개수를 보고 통과할 수 있습니다.

이 PR의 목적이 UI 레벨 검증이 아니라 DB에서 "shade당 color_family 최대 2개"를 보장하는 것이라면,
COUNT 전에 같은 shade_id 단위로 작업을 직렬화하는 lock이 필요해 보입니다.

가독성 측면에서는 부모 shades row를 SELECT ... FOR UPDATE로 잠그는 방식이 이해하기 쉬울 것 같습니다.
대안으로는 pg_advisory_xact_lock을 shade_id 기준으로 사용하는 방법도 있습니다.
```

## 실무에서 가져갈 점

DB 제약을 설계할 때는 "한 요청이 혼자 실행될 때 맞는가"만 보면 부족하다.

```text
검사
-> 쓰기
```

이 두 단계 사이에 다른 트랜잭션이 끼어들 수 있다면 동시성 버그가 생긴다.

그래서 다음 질문을 같이 해야 한다.

- 이 불변식은 DB가 최종적으로 보장해야 하는가?
- 같은 비즈니스 단위는 무엇인가?
- 그 단위를 기준으로 줄 세울 실제 row가 있는가?
- 없다면 advisory lock 같은 별도 직렬화 장치가 필요한가?
- lock을 잡은 뒤 다시 조건을 확인하고 있는가?

> [!summary]
> "검사하고 넣기"는 동시성에서 위험하다. 검사 대상 단위를 먼저 잠그고, 그 다음 검사해야 DB 제약이라고 말할 수 있다.

## 복습 질문

- [ ] `COUNT(*)` trigger가 단일 요청에서는 잘 동작해도 동시 요청에서 깨질 수 있는 이유는 무엇인가?
- [ ] `SELECT ... FOR UPDATE`는 어떤 대상을 잠그고, 이번 케이스에서는 왜 부모 `shades` row를 잠그는 것이 자연스러운가?
- [ ] advisory lock은 실제 row lock과 무엇이 다른가?
- [ ] "논리적 잠금"이라는 표현은 왜 advisory lock을 설명할 때 쓰기 좋은가?
- [ ] UI 검증과 DB 제약은 동시성 책임 관점에서 무엇이 다른가?

## 한 줄 회고

- 헷갈렸던 점: trigger가 있으면 DB 제약이 완성된다고 생각할 수 있지만, `COUNT -> INSERT` 사이에 다른 트랜잭션이 끼어들 수 있어서 같은 `shade_id` 단위로 먼저 잠그고 다시 검사해야 한다는 점. 그리고 advisory lock은 DB가 row 의미를 아는 잠금이 아니라 개발자가 key 의미를 약속해서 쓰는 잠금이라는 점.
