---
tags:
  - di
  - zustand
  - factory-pattern
  - spring
  - testing
---

# 🔍 zustand factory 패턴 — 왜 굳이? (순환 import & 테스트)

> [!abstract] 한 줄 요약 두 문제(순환 import, 테스트 어려움)의 뿌리는 **하나**다. **"의존성을 직접 import 해서 안에서 끌어다 쓰느냐(나쁨) vs 밖에서 인자로 받느냐(좋음)"** 👉 이게 바로 DI 가 풀려는 문제 그 자체. Spring `@Autowired` 가 자동화한 원리를 zustand 는 `get` 인자로 손수 적는 것.

---

## 🧠 먼저 전제: import 는 "복사"가 아니라 "실행"이다

> [!important] 자바와 다른 점 자바는 클래스 로딩 순서를 거의 신경 안 씀 (JVM 이 알아서). 하지만 **JS/TS 모듈은 import 되는 순간 그 파일이 위에서 아래로 한 번 실행된다.** `export const x = ...` → 파일이 처음 로드될 때 `= ...` 부분이 진짜로 실행되며 값이 채워짐.
> 
> 즉, **모듈은 "실행되는 순간"이 있다.** 자바 클래스처럼 가만히 있는 정적 정의가 아님. 이게 순환 import 이해의 핵심.

---

## 💥 1번 문제: 순환 import 가 왜 터지나

### 안 좋은 패턴 (서로를 import)

```typescript
// useStore.ts
import { createInventorySlice } from './inventorySlice'  // ①

export const useStore = create((set, get) => ({
  ...createInventorySlice(set, get),
}))
```

```typescript
// inventorySlice.ts  ← 안 좋은 패턴
import { useStore } from './useStore'  // ②

export const createInventorySlice = (set, get) => ({
  toggleInventory: () => {
    if (useStore.getState().auth.status !== 'authenticated') return
  }
})
```

### 실행 순서 따라가기 (useStore.ts 를 먼저 로드한다고 가정)

```
1. useStore.ts 실행 시작
2. ① 에서 inventorySlice.ts import
      → useStore.ts 실행 잠깐 멈춤 ⏸️, inventorySlice.ts 로 점프
3. inventorySlice.ts 실행 시작
4. ② 에서 useStore.ts import
      → 근데 useStore.ts 는 2번에서 멈춰있음
      → export const useStore 가 아직 실행 안 됨 ❌
5. JS: "순환이네? 일단 지금까지 만들어진 거라도 줄게"
      → useStore = undefined 상태로 넘겨줌 😱
6. inventorySlice.ts 는 그 undefined 받고 함수 정의 끝냄 → useStore.ts 로 복귀
7. 이제야 useStore.ts 가 export const useStore = create(...) 실행 → 진짜 store 생성
```

> [!warning] 함정의 정체 6번 시점에서 `inventorySlice` 가 받은 `useStore` 는 **`undefined`**.

### 왜 더 위험한가 (조용히 숨어있음)

- `toggleInventory` 함수 **안**에서 `useStore.getState()` 를 쓰면 → 함수는 정의만 되고 당장 실행 안 됨
- 나중에 버튼 클릭할 때쯤엔 store 가 다 만들어져 있어서 **우연히 동작** 함
- ➡️ 개발할 땐 멀쩡하다가, **import 순서가 살짝 바뀌거나 모듈 하나만 추가해도** 갑자기 터짐
    - `Cannot read property 'getState' of undefined` 💀

> [!note] 자바 비유 두 빈이 서로 **생성자 주입**으로 물려 순환 의존성 에러 나는 거랑 같은 상황. 차이: Spring 은 **시작할 때 딱 잡아서 에러를 던짐.** JS 모듈 순환은 **조용히 `undefined` 를 흘려보내** 런타임에 뜬금없이 터짐.

---

## ✅ factory 패턴은 왜 안 터지나

```typescript
// inventorySlice.ts  ← 좋은 패턴
import type { StoreState } from './useStore'  // ← type 키워드!

export const createInventorySlice = (set, get) => ({
  toggleInventory: () => {
    if (get().auth.status !== 'authenticated') return  // useStore 대신 get()
  }
})
```

차이는 두 군데:

> [!success] 1. `import type` → 컴파일하면 완전히 사라짐 TypeScript 의 `import type` 은 타입 검사할 때만 쓰이고 **빌드 결과물(.js)엔 그 import 라인 자체가 없음.** 즉 런타임에 `inventorySlice.ts` 는 `useStore.ts` 를 **아예 import 하지 않음** → 순환 고리의 한쪽이 끊김 ✂️

> [!success] 2. `get` 인자 → slice 가 useStore 를 직접 참조 안 함 `get` 은 `useStore.ts` 가 `createInventorySlice(set, get)` 호출 시 넘겨주는 값. `inventorySlice` 는 `useStore` 를 알 필요 0. "누가 나한테 get 을 줄 거다" 라는 **약속만** 믿으면 됨. 의존 방향이 **한쪽으로만** 흐름: `useStore → slice` (단방향) ➡️

---

## 🧪 2번 문제: 테스트가 왜 어렵나

### 안 좋은 패턴 — 진짜 useStore 가 항상 딸려옴

```typescript
import { useStore } from '../useStore'
// 이 함수는 "진짜 useStore" 없으면 동작 자체를 안 함
```

`inventorySlice` 만 따로 테스트하려 해도 → 항상 **진짜 `useStore` 전체**가 끌려옴. auth 슬라이스, toast 슬라이스, persist 미들웨어... 전부 같이. inventory 하나 테스트하는데 store 전체를 세팅해야 함. 😩

> [!note] 자바 비유 `InventoryService` 테스트하려는데 `new InventoryService()` 가 내부에서 `RealAuthService` 를 직접 `new` 해버려서 가짜를 못 끼워넣는 상황.

### 좋은 패턴 — 가짜 get 넣으면 끝

```typescript
// inventorySlice 단독 테스트
const fakeGet = () => ({
  auth: { status: 'authenticated' },   // 원하는 상태로 조작
  showToast: jest.fn(),
})
const fakeSet = jest.fn()

const slice = createInventorySlice(fakeSet, fakeGet)
slice.toggleInventory('item-1')   // 진짜 store 없이 단독 실행 ✅
```

> [!tip] Spring 비유 = `@MockBean` Spring 에서 `AuthService` 를 mock 으로 갈아끼워 `InventoryService` 만 테스트하듯, 여기선 `get` 을 가짜로 갈아끼워 slice 만 테스트. **의존성을 밖에서 주입**받기 때문에 가능 — 안에서 직접 import 하면 못 함.

```
 한눈에 — 두 파일 관계도

  ┌─ inventorySlice.ts ─────────────┐    ┌─ useStore.ts ──────────────────┐
  │                                  │    │                                  │
  │  createInventorySlice(set, get)  │ ◄──┤  ...createInventorySlice(       │
  │  ├ inventory: {}                 │    │       set,  ← zustand 가 제공    │
  │  ├ toggleInventory               │    │       get,  ← zustand 가 제공    │
  │  │   └─ get().auth ─────────┐    │    │     )                            │
  │  │   └─ get().showToast ──┐ │    │    │  auth: { ... }       ◄──┐       │
  │  └ retireInventory        │ │    │    │  showToast: () => {} ◄┐ │       │
  │                            │ │    │    │                       │ │       │
  └────────────────────────────┼─┼────┘    └───────────────────────┼─┼───────┘
                               │ │                                  │ │
                               │ └──────────────────────────────────┘ │
                               └──────────────────────────────────────┘
                                get() 호출 시 root state 에서 빌려옴

```


---

## 🎯 한 줄로 꿰기

> [!abstract] 두 문제는 뿌리가 같다
> 
> ||직접 import (나쁨)|인자로 받음 (좋음)|
> |---|---|---|
> |**결합**|런타임 결합 O|결합 끊김|
> |**순환 import**|터짐 💥|안 남 ✅|
> |**테스트**|가짜로 못 바꿈|가짜 주입 가능 ✅|
> 
> - 직접 import → 런타임 결합 → 순환 터짐 + 가짜로 못 바꿈
> - 인자로 받음 → 결합 끊김 → 순환 안 남 + 가짜 주입 가능
> 
> 이게 바로 **DI(의존성 주입)가 풀려는 문제** 그 자체. Spring 이 `@Autowired` 로 자동화한 원리를 zustand 에선 `get` 인자로 손수 적는 것뿐. 🤝

---

## 🔗 관련 노트

- [[Spring-DI-vs-zustand]] — 기본 멘탈 모델 & 최상위 부트스트랩
