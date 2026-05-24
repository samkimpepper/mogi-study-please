---
tags:
  - di
  - spring
  - zustand
  - dependency-injection
  - factory-pattern
---

# 🔌 Spring DI vs zustand — 의존성 주입 멘탈 모델

> [!abstract] 한 줄 요약 **Spring `@Autowired`** = 컨테이너(비서)가 알아서 챙겨줌 🤖 **zustand factory** = 비서가 없어서 내가 직접 `(set, get)` 손으로 넘김 ✋ 본질은 똑같다 — 옛날 `new InventoryService(authService, ...)` 로 생성자 주입하던 그 결.

---

## 🏢 비유부터: 비서가 있냐 없냐

||비유|의미|
|---|---|---|
|**Spring**|자동문 + 비서 있는 회사|출근하면 필요한 서류(의존성)가 책상에 이미 올라와 있음. 누가 가져다 놨는지 몰라도 됨|
|**zustand**|비서 없는 1인 사무실|서류 필요하면 내가 직접 "이거 줘" 하고 받아와야 함|

---

## ☕ Spring 에서 익숙한 그림

```java
@Service
class InventoryService {
    @Autowired AuthService auth;   // ← 컨테이너가 알아서 넣어줌

    public void toggle(String id) {
        if (auth.getStatus() != AUTHENTICATED) return;
    }
}
```

> [!note] 핵심 `InventoryService` 는 `AuthService` 가 **어디서 만들어졌는지 / 언제 생성되는지 전혀 신경 안 써도 됨.** 그냥 "나 AuthService 필요해" 라고 **선언만** 하면 끝. 이게 DI 컨테이너의 마법 ✨

컨테이너가 내부적으로 하는 일:

1. **스캔** — `@Component`, `@Service`, `@Repository` 붙은 클래스 전부 찾기
2. **그래프 분석** — "A는 B 필요, B는 C 필요" 순서 정렬 (위상정렬 / topological sort)
3. **순서대로 생성** — 의존성 없는 leaf(`C`)부터 만들고 → 위로 차곡차곡 주입

---

## 🧩 zustand 에는 컨테이너가 없다

zustand 는 그냥 **순수 함수 모음**. `@Autowired` 같은 데코레이터 0개, 런타임 컨테이너 0개. 그래서 `InventorySlice` 가 `auth` 를 쓰려면 → **누군가 손으로 넣어줘야 함.**

그 "넣어주는 도구" 가 바로 `get()` 함수.

```typescript
// inventorySlice.ts
export const createInventorySlice = (
  set,   // ← "상태 바꾸는 함수, 외부에서 줘"
  get,   // ← "현재 상태 읽는 함수, 외부에서 줘"
) => ({
  toggleInventory: async (id) => {
    if (get().auth.status !== 'authenticated') return  // get() 으로 auth 빌려옴
    get().showToast('...')                             // get() 으로 toast 빌려옴
  }
})
```

> [!tip] 포인트 `createInventorySlice` 가 자기 의존성을 **직접 import 안 함.** 대신 "`get()` 호출하면 전체 root state 가 나온다" 는 **약속만** 받음.

그리고 진짜 조립은 `useStore.ts` 에서 **손으로**:

```typescript
// useStore.ts
export const useStore = create()(
  persist(
    (set, get) => ({
      auth: { status: 'unknown', ... },          // auth 슬라이스
      showToast: (msg) => { ... },                // toast 슬라이스
      ...createInventorySlice(set, get),          // ← 여기서 set, get 손으로 주입
    })
  )
)
```

➡️ Spring 이 자동으로 해주는 일을 사람이 직접 함.

---

## 📊 Spring vs zustand 차이 정리

||Spring|zustand|
|---|---|---|
|**의존성 선언**|`@Autowired AuthService`|`get` 을 인자로 받음|
|**누가 넣어주나**|컨테이너가 런타임에 자동|사람이 `createInventorySlice(set, get)` 호출 시|
|**검증 시점**|컨테이너가 앱 시작 시|TypeScript 가 컴파일 시|
|**인스턴스 1개 보장**|싱글톤 scope|`create()` 1번 호출로 store 1개|

---

## ❓ 왜 굳이 이 패턴인가 (직접 import 하면 안 돼?)

```typescript
// 😈 안 좋은 패턴
import { useStore } from '../useStore'
const toggleInventory = () => {
  if (useStore.getState().auth.status !== 'authenticated') return
}
```

> [!warning] 문제 2가지 **1. 순환 import (circular dependency)**
> 
> - `useStore` 가 동작하려면 → `createInventorySlice` 를 import 해야 함
> - 근데 `createInventorySlice` 가 `useStore` 를 import 하면 → 서로 물고 물림 🔁
> - 빌드 시 한쪽이 아직 `undefined` 인 시점 발생 → 터짐
> - 👉 Spring 에서 빈 두 개가 `@Autowired` 로 서로 물려 순환 의존성 에러 나는 거랑 똑같은 상황
> 
> **2. 테스트 어려움**
> 
> - slice 단독으로 mock store 주고 못 굴림. 진짜 `useStore` 가 항상 따라옴

> [!success] factory + get() 인자 주입 = 둘 다 해결
> 
> - import 는 **타입만** 가져옴 (런타임 의존성 0)
> - 테스트할 땐 **가짜 `get`/`set`** 넣어서 단독 호출 가능 → Spring 의 `@MockBean` 같은 효과 🧪

---

## 🔝 "계속 거슬러 올라가면 최상위에서 주입하는 놈은 누구?"

> [!question] 핵심 질문 A는 B 주입받고, B는 C 주입받고... 끝까지 가면 **결국 누가 맨 처음 손으로 `new` 하나?** 컨테이너 자신은 컨테이너가 안 만들어주잖아!

### Spring 의 답: `main()` 에서 시작된다

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);  // ← 진짜 시작점
    }
}
```

`SpringApplication.run()` 내부 (엄청 단순화):

```java
ApplicationContext context = new AnnotationConfigApplicationContext();
// ↑ 이 new 가 "최상위에서 손으로 만드는" 그 한 줄. 여기가 끝점.
```

> [!important] 결론 **`ApplicationContext`(컨테이너 본체)는 `@Autowired` 로 주입받는 게 아니라 그냥 `new` 로 만들어진다.** 비서를 고용하는 행위 자체는 비서가 못 함. 사장(`main`)이 직접 해야 함.

### 닭-달걀처럼 보이지만 → 계층이 다름

> [!note] 공장 vs 제품
> 
> - 컨테이너 = **공장** 🏭
> - 빈(Bean) = 공장이 찍어내는 **제품** 📦
> - 공장은 제품 라인에서 안 나옴. 공장은 **사람이 짓는 것**. `main()` 이 그 사람.

### zustand 도 똑같은 "최상위 한 지점" 이 있다

```typescript
export const useStore = create()(...)  // ← 이 create() 호출이 zustand 의 main()
```

차이는 단지:

- **Spring** → 부트스트랩 한 줄을 `SpringApplication.run()` 안에 **숨겨놓음**
- **zustand** → `create()` 한 줄로 **눈앞에 그냥 보여줌**

본질은 같다 → **누군가는 맨 위에서 손으로 시작 버튼을 눌러야 함.** 🔘

---

## ☕ `create((set, get) => ...)` 를 자바로 보면

> [!summary] 한 줄 감각
> zustand 의 `create((set, get) => ({ ... }))` 는 자바로 치면 **Store 를 하나 만들고, 그 Store 의 `setState/getState` 메서드를 각 Slice 생성자에 수동으로 꽂아주는 설정 클래스**에 가깝다.

zustand 코드:

```typescript
export const useStore = create((set, get) => ({
  ...createAuthSlice(set, get),
  ...createInventorySlice(set, get),
  ...createToastSlice(set, get),
}))
```

자바로 억지 비유하면:

```java
public class StoreBootstrap {

    public static Store createStore() {
        Store store = new Store();

        SetFunction set = store::setState;
        GetFunction get = store::getState;

        AuthSlice authSlice = AuthSliceFactory.create(set, get);
        InventorySlice inventorySlice = InventorySliceFactory.create(set, get);
        ToastSlice toastSlice = ToastSliceFactory.create(set, get);

        store.add(authSlice);
        store.add(inventorySlice);
        store.add(toastSlice);

        return store;
    }
}
```

> [!note] 역할 분리
> 
> - `create()` 가 하는 일: store 를 만들고 `set`, `get` 을 준비함
> - `useStore.ts` 가 하는 일: 준비된 `set`, `get` 을 각 slice factory 에 손으로 넘김
> - `createInventorySlice(set, get)` 이 하는 일: 받은 함수들로 inventory 기능 묶음을 만들어냄

Spring 생성자 주입 느낌으로 더 줄이면:

```java
public class InventorySlice {

    private final SetFunction set;
    private final GetFunction get;

    public InventorySlice(SetFunction set, GetFunction get) {
        this.set = set;
        this.get = get;
    }

    public void toggleInventory(String id) {
        StoreState state = get.call();

        if (!state.auth().isAuthenticated()) {
            return;
        }

        // set.call(...) 로 상태 변경
    }
}
```

그리고 조립하는 쪽:

```java
InventorySlice inventorySlice = new InventorySlice(set, get);
```

zustand 에서는 이 `new InventorySlice(set, get)` 이 함수 호출로 바뀐 것뿐이다.

```typescript
createInventorySlice(set, get)
```

> [!important] 그래서 누가 주입하나?
> 
> - `set`, `get` 을 **제공**하는 건 zustand 의 `create()`
> - 그 `set`, `get` 을 각 slice 에 **꽂아주는** 건 `useStore.ts` 의 조립 코드
> 
> Spring 에서는 이 조립을 `ApplicationContext` 가 자동으로 해준다. zustand 에서는 `useStore.ts` 에서 사람이 직접 한다.

---

## 🎯 최종 정리

> [!abstract] 기억할 것
> 
> - 자동 주입은 **무한히 자동인 게 아님.** "`main` 에서 컨테이너를 `new` 하는 단 한 번의 수동 작업" 위에 쌓인 자동화일 뿐.
> - zustand 는 프레임워크가 작아서 그 컨테이너가 없음 → "의존성을 인자로 받는 함수" 라는 가장 원시적인 DI (= constructor injection 의 함수 버전) 를 직접 적어줌.
> - Spring 으로 치면 `@Autowired` 대신 옛날 방식 `new InventoryService(authService, toastService)` 호출하는 거랑 똑같은 결. 🤝

### 🔖 더 파볼 키워드

- `BeanFactory` vs `ApplicationContext` 차이
- 빈 생명주기 (`@PostConstruct`, `@PreDestroy`)
- 순환 의존성 해결 방식 (Spring 의 3단계 캐시 / setter 주입 우회)
