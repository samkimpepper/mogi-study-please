---
tags:
  - backend
  - oop
  - polymorphism
---

# OOP와 선택문, 그리고 다형성

## 한 줄 정리

OOP는 `if`문 자체를 없애자는 말이 아니라, 상위 계층이 구체 타입을 보고 계속 분기하는 구조를 줄이고 객체의 다형성에 맡기자는 접근이다.

> [!summary]
> 구조적 프로그래밍이 `GOTO`를 줄여 흐름을 읽기 쉽게 만든 것처럼, OOP는 상위 계층의 타입 기반 `if`/`switch`를 줄여 변경에 강한 모듈 구조를 만들려는 시도라고 볼 수 있다.

## OOP가 구조적 프로그래밍 전체를 대체하는 것은 아니다

구조적 프로그래밍의 흐름 제어는 크게 세 가지다.

```text
1. 순서: 위에서 아래로 실행
2. 선택: if, switch, match
3. 반복: for, while
```

OOP를 써도 코드는 여전히 위에서 아래로 실행되고, `for`, `while`도 그대로 사용한다. 그래서 "OOP가 구조적 프로그래밍을 대체한다"는 말은 너무 넓다.

더 정확히는 다음처럼 이해하는 편이 좋다.

> [!note]
> OOP는 구조적 프로그래밍의 세 흐름 제어 중 `선택(selection)`의 일부를 서브타입 다형성으로 대체할 수 있다.

## if문에서의 선택

`if`문에서는 조건식의 boolean 값에 따라 이후 실행할 코드가 달라진다.

```java
if (paymentType == PaymentType.CARD) {
    cardPay();
} else if (paymentType == PaymentType.KAKAO) {
    kakaoPay();
} else if (paymentType == PaymentType.TOSS) {
    tossPay();
}
```

이 방식은 읽기 쉽지만, 결제 수단이 늘어날 때마다 상위 서비스 코드가 계속 바뀐다.

```text
CARD 추가
-> PaymentService 수정

KAKAO 추가
-> PaymentService 수정

TOSS 추가
-> PaymentService 수정
```

즉, `PaymentService`가 "결제를 시킨다"는 역할뿐 아니라 "각 결제 수단이 어떻게 처리되는지"까지 알고 있게 된다.

> [!warning]
> 문제가 되는 것은 `if` 사용 자체가 아니라, 상위 계층이 구체 타입을 검사하는 분기문을 여러 곳에서 반복하는 구조다.

## 다형성에서의 선택

다형성에서는 상위 코드가 구체 타입을 직접 검사하지 않는다. 공통 인터페이스에 메시지를 보내고, 실제 실행될 메서드는 런타임 인스턴스 타입에 따라 결정된다.

```java
interface PaymentMethod {
    void pay();
}

class CardPayment implements PaymentMethod {
    @Override
    public void pay() {
        // 카드 결제
    }
}

class KakaoPayment implements PaymentMethod {
    @Override
    public void pay() {
        // 카카오 결제
    }
}
```

상위 서비스는 구체 결제 수단을 몰라도 된다.

```java
class PaymentService {
    void pay(PaymentMethod paymentMethod) {
        paymentMethod.pay();
    }
}
```

이때 선택은 사라진 것이 아니라 위치가 바뀐 것이다.

```text
if문
-> 조건값이 실행 경로를 고른다.

다형성
-> 객체의 실제 타입이 실행 메서드를 고른다.
```

> [!tip]
> `if`는 "너 카드야? 그럼 카드 결제해"라고 묻는 방식이고, 다형성은 "너 결제수단이지? 그럼 네 방식대로 결제해"라고 맡기는 방식이다.

## 언제 if를 다형성으로 바꿔야 하나

모든 `if`를 없애려고 하면 오히려 코드가 이상해진다. 단순한 검증 조건은 그냥 `if`가 자연스럽다.

```java
if (amount <= 0) {
    throw new IllegalArgumentException();
}
```

이런 코드는 "타입별 행동"을 고르는 것이 아니라, 잘못된 값을 막는 검증이다. OOP로 억지로 바꿀 이유가 없다.

반대로 아래처럼 구체 타입별 행동 분기가 여러 곳에 반복되면 다형성을 고민할 만하다.

```java
if (user instanceof Admin) {
    // 관리자 처리
} else if (user instanceof Seller) {
    // 판매자 처리
} else if (user instanceof Buyer) {
    // 구매자 처리
}
```

새 역할이 추가될 때마다 여러 서비스의 분기문을 수정해야 한다면, 상위 계층이 너무 많은 구체 타입 지식을 갖고 있다는 신호다.

## 백엔드 관점에서 가져갈 점

실무에서 중요한 질문은 "`if`를 썼는가?"가 아니라 "변경이 어디까지 번지는가?"다.

```text
나쁜 냄새
-> 결제 수단, 유저 역할, 알림 채널 같은 타입이 늘어날 때
-> 여러 상위 서비스의 if/switch를 같이 수정해야 함

OOP로 풀기 좋은 방향
-> 공통 인터페이스를 만들고
-> 구체 구현체가 자기 행동을 갖게 하고
-> 상위 서비스는 인터페이스에만 메시지를 보냄
```

> [!summary]
> OOP가 선택문을 대체한다는 말은 `if`를 문법적으로 금지한다는 뜻이 아니다. 상위 정책 코드가 타입을 보고 직접 고르던 실행 경로를 객체의 다형성에 맡긴다는 뜻이다.

## 복습 질문

- [ ] OOP가 구조적 프로그래밍 전체를 대체한다고 말하면 왜 부정확한가?
- [ ] `if`문에서의 선택과 서브타입 다형성에서의 선택은 각각 무엇을 기준으로 일어나는가?
- [ ] 결제 수단 예시에서 `PaymentService`가 `if/switch`를 많이 가지면 어떤 변경 문제가 생기는가?
- [ ] 모든 `if`를 다형성으로 바꾸면 안 되는 이유는 무엇인가?
- [ ] 타입별 분기가 여러 곳에 반복될 때 다형성을 고려해야 하는 이유는 무엇인가?

## 한 줄 회고

- 헷갈렸던 점: OOP가 `if`문을 무조건 없애는 것이 아니라, 상위 계층이 구체 타입을 보고 직접 선택하던 구조를 객체의 다형성에 맡기는 것이라는 점.
