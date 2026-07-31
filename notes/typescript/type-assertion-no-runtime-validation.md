---
tags:
  - typescript
  - javascript
  - type-safety
  - backend
review_answered: false
---

# TypeScript의 `as`는 왜 런타임에 검사하지 않을까

> [!summary]
> TypeScript는 JavaScript에 정적 타입 검사만 더한 언어다. 컴파일되면 타입 정보와 `as`가 사라지므로 런타임 검사는 일어나지 않는다.

## `as Texture`의 실제 의미

```ts
type Texture = 'creamy' | 'glowy'

const texture = slugFromDb as Texture
```

컴파일된 JavaScript에는 사실상 다음 코드만 남는다.

```js
const texture = slugFromDb
```

`as Texture`는 값을 변환하거나 검증하지 않고, 컴파일러에게 “개발자가 확인했으니 믿어도 된다”고 알려주는 탈출구다.

## Java와 다른 이유

Java enum은 JVM에도 타입과 값의 목록이 남으므로 `Texture.valueOf()`로 런타임 검사가 가능하다. 반면 TypeScript의 union 타입은 컴파일러만 사용하는 정보라서 JavaScript가 실행될 때는 존재하지 않는다.

| 구분 | Java enum | TypeScript union과 `as` |
| --- | --- | --- |
| 런타임에 타입 정보가 남는가 | 남는다 | 사라진다 |
| 잘못된 외부 값 검사 | `valueOf()` 등으로 가능 | 별도 parser가 필요 |
| 단언 자체가 검사하는가 | 캐스팅에 따라 예외 가능 | 전혀 검사하지 않음 |

TypeScript가 자동 검사 코드를 만들지 않는 이유는 기존 JavaScript와 같은 동작을 유지하고, 모든 대입과 함수 호출에 검사 비용을 추가하지 않기 위해서다.

## 외부 값은 직접 검사한다

```ts
const TEXTURES = ['creamy', 'glowy'] as const
type Texture = (typeof TEXTURES)[number]

function isTexture(value: string): value is Texture {
  return TEXTURES.some(texture => texture === value)
}

if (!isTexture(slugFromDb)) {
  throw new Error(`Unknown texture: ${slugFromDb}`)
}

// 이 지점부터 slugFromDb는 안전한 Texture다.
const texture = slugFromDb
```

> DB, API 응답, 사용자 입력처럼 프로그램 밖에서 들어오는 값은 TypeScript 타입만 믿지 말고 런타임 경계에서 검증해야 한다.

## 복습 질문

- [ ] TypeScript의 `as`가 값을 검사하거나 변환하지 않는 이유는 무엇인가?
- [ ] Java enum과 TypeScript union은 런타임에 어떤 차이가 있는가?
- [ ] DB나 API에서 받은 값을 `as`로 단언하면 왜 위험한가?
- [ ] TypeScript에서 Java의 `Enum.valueOf()` 역할은 무엇이 해야 하는가?

## 백지 복습 답변

### 1. TypeScript의 `as`가 값을 검사하거나 변환하지 않는 이유는 무엇인가?

내 답:

-

피드백:

-

## 한 줄 회고

- 헷갈렸던 점: TypeScript 타입은 런타임 검증 기능이 아니라 컴파일 전에 코드의 논리를 확인하는 정보다.
