---
tags:
  - frontend
  - react
  - error-handling
  - observability
  - backend
review_answered: false
---

# React ErrorBoundary가 비동기 오류를 잡지 못하는 이유

> [!summary]
> ErrorBoundary는 React가 컴포넌트 트리를 그리는 도중 발생한 예외를 다른 화면으로 교체하는 장치다. 렌더가 끝난 뒤 실행되는 클릭 이벤트나 비동기 데이터 요청의 실패는 각 작업의 `catch`에서 별도로 처리해야 한다.

## 처음 헷갈렸던 질문

최상위에 ErrorBoundary를 두었다면 왜 각 화면마다 데이터 로딩 실패를 잡는 `catch`와 inline 재시도 UI가 또 필요할까?

핵심은 오류가 **언제, 어떤 실행 흐름에서 발생했는가**다.

---

## 🎭 렌더 오류는 공연 도중의 사고다

React가 컴포넌트를 렌더링하는 동안 예외가 발생하면 현재 컴포넌트 트리를 정상적으로 완성할 수 없다.

```tsx
function Profile() {
  throw new Error('render failed')
}
```

ErrorBoundary는 React의 렌더 흐름 안에서 이 예외를 감지하고, 망가진 트리 대신 fallback 화면을 그린다.

```text
컴포넌트 렌더 시작
  ↓
렌더 도중 예외
  ↓
ErrorBoundary가 감지
  ↓
문제가 생겼어요 + 새로고침 화면으로 교체
```

무대 공연 중 사고가 나면 공연장 안전요원이 무대를 정리하고 대체 안내를 보여주는 것과 비슷하다.

## ⏰ 이벤트와 비동기 실패는 렌더가 끝난 뒤 도착한다

버튼 클릭, 저장 요청, `fetch()` 결과는 화면을 다 그린 뒤 나중에 발생한다.

```tsx
async function handleSave() {
  await savePost()
}
```

`savePost()`가 거부되는 순간 React는 컴포넌트를 렌더링하고 있지 않다. 따라서 렌더 트리를 fallback으로 교체하는 ErrorBoundary가 개입할 실행 흐름이 없다.

```text
렌더 완료
  ↓
사용자가 버튼 클릭
  ↓
비동기 저장 요청
  ↓
나중에 Promise 실패
  ↓
버튼 핸들러의 catch가 처리
```

공연이 끝난 뒤 관객이 로비에서 넘어진 사건까지 무대 안전요원이 처리하지 않는 것과 같다.

> [!warning]
> ErrorBoundary는 애플리케이션의 모든 오류를 잡는 전역 `try-catch`가 아니다. 렌더 오류 복구라는 좁고 명확한 책임을 가진다.

---

## 렌더가 끝나면 ErrorBoundary의 호출 경로도 끝난다

ErrorBoundary는 과거에 끝난 렌더를 계속 감시하는 장치가 아니다. React가 컴포넌트 트리를 렌더하는 동안 그 호출 경로에서 발생한 예외를 복구하는 장치다.

개념적으로 단순화하면 React는 다음과 비슷한 실행 흐름을 가진다.

```js
try {
  renderComponentTree()
} catch (error) {
  findNearestErrorBoundary()
  renderFallback()
}
```

실제 React 구현이 이 코드 그대로라는 뜻은 아니다. 중요한 것은 렌더 중 발생한 예외가 같은 호출 스택을 타고 React의 렌더 실행부까지 올라온다는 점이다.

```text
React 렌더 실행부
└─ 부모 컴포넌트 렌더
   └─ 자식 컴포넌트 렌더
      └─ 예외 발생
```

이때 React는 어느 컴포넌트를 그리다 실패했는지와 가장 가까운 ErrorBoundary가 어디인지 알 수 있다. 그래서 망가진 트리 부분을 버리고 해당 boundary의 fallback으로 교체할 수 있다.

렌더가 성공적으로 끝나면 이 호출 스택도 종료된다.

```text
React 렌더 시작
  ↓
컴포넌트 함수 호출
  ↓
렌더 완료
  ↓
브라우저에 화면 표시
  ↓
렌더 호출 스택 종료
```

나중에 사용자가 버튼을 클릭하면 브라우저가 새로운 호출 스택에서 이벤트 핸들러를 실행한다.

```text
렌더할 때

React
└─ SaveButton 렌더
   └─ 성공하고 종료


나중에 클릭할 때

브라우저
└─ click 이벤트
   └─ handleSave
      └─ save
         └─ 예외 발생
```

클릭 이벤트의 호출 경로에는 이미 종료된 React 렌더 실행부와 ErrorBoundary가 없다. 예외가 시간을 거슬러 과거 호출 스택으로 돌아갈 수도 없다.

> ErrorBoundary가 렌더 후에 감시를 중단한 것이 아니라, 이벤트 핸들러의 호출 경로에는 처음부터 ErrorBoundary가 없다.

### `await` 뒤의 실패는 더 분명하게 실행 경계가 갈린다

```tsx
async function handleSave() {
  await save()
}
```

`await`에서 함수는 실행을 잠시 양보하고 현재 호출 스택은 종료된다. Promise의 결과가 나중에 도착하면 microtask에서 이어서 실행된다.

```text
클릭 호출 스택
  ↓
handleSave 실행
  ↓
save Promise 시작
  ↓
await에서 현재 호출 스택 종료

──── 시간이 흐름 ────

Promise 실패
  ↓
microtask에서 실패 처리
```

Promise가 실패한 시점에는 렌더 호출 스택도, 처음 클릭했을 때의 호출 스택도 끝난 상태다. 따라서 비동기 작업을 시작한 코드가 직접 오류를 처리해야 한다.

```tsx
async function handleSave() {
  try {
    await save()
  } catch (error) {
    setSaveError(error)
  }
}
```

### Java도 비동기 실행 경계가 갈리면 똑같다

일반적인 Spring MVC 요청은 같은 요청 스레드의 동기 호출 체인에서 실행된다.

```text
DispatcherServlet
└─ Controller
   └─ Service
      └─ Repository
         └─ 예외 발생
```

Repository 예외가 같은 호출 스택을 타고 올라오기 때문에 `@RestControllerAdvice` 같은 예외 처리기가 받을 수 있다.

하지만 별도 실행기에 작업을 넘기면 바깥 `try-catch`는 그 작업의 예외를 잡지 못한다.

```java
try {
    executor.submit(() -> {
        throw new RuntimeException("비동기 실패");
    });
} catch (Exception e) {
    // 작업 스레드에서 나중에 발생한 예외는 여기서 잡히지 않는다.
}
```

```text
요청 스레드
try
└─ executor.submit
   └─ 작업 등록 후 반환
try 종료


작업 스레드
└─ Runnable 실행
   └─ 예외 발생
```

작업 스레드의 호출 스택에는 요청 스레드의 `try-catch`가 없기 때문이다. `CompletableFuture.exceptionally()`, 작업 내부의 `try-catch`, 비동기 실행기의 오류 처리처럼 새 실행 경계에 맞는 처리 방식이 필요하다.

Spring이 Controller가 반환한 `CompletableFuture`의 실패를 MVC 예외 처리와 연결해주는 경우도 있다. 이는 예외가 자동으로 과거 스택을 찾아간 것이 아니라, Spring이 Future의 완료 결과를 구독하여 자신의 예외 처리 파이프라인에 명시적으로 다시 연결한 것이다.

### 전역 오류 감지와 화면 복구도 구분한다

브라우저의 전역 `error`나 `unhandledrejection` 이벤트로 비동기 오류를 관측할 수는 있다. 하지만 이것만으로는 어느 ErrorBoundary의 화면을 교체해야 하는지 알기 어렵다.

- 작업을 시작한 컴포넌트가 이미 사라졌을 수 있다.
- 사용자가 다른 페이지로 이동했을 수 있다.
- 화면과 무관한 백그라운드 작업일 수 있다.
- 저장 실패 하나 때문에 전체 화면을 없애는 것은 복구 범위가 지나치게 클 수 있다.

따라서 전역 오류 감지는 주로 로그 보고에 사용하고, 사용자 화면의 복구는 작업을 소유한 컴포넌트가 담당한다.

```text
저장 실패
  ├─ 화면: 기존 폼 유지 + 저장 실패 표시
  └─ 운영: 오류 로그 보고
```

---

## stale closure와 같은 뿌리에서 나온다

React의 각 렌더는 자기만의 값과 함수를 가진 하나의 스냅샷이다. 렌더에서 만들어진 함수가 나중에 실행되면 서로 다른 두 문제가 나타날 수 있다.

- 함수가 자신이 태어난 렌더의 옛 값을 읽는다 → stale closure
- 함수에서 오류가 발생하지만 이미 렌더가 끝났다 → ErrorBoundary가 잡지 못함

```tsx
function Editor() {
  const [name, setName] = useState('')

  async function handleSave() {
    await waitForSomething()
    await save(name)
  }

  return <button onClick={handleSave}>저장</button>
}
```

`handleSave`가 기다리는 사이 새 렌더가 생겨도, 이미 실행 중인 함수는 자신이 태어난 렌더의 `name`을 계속 읽을 수 있다. 이것이 stale closure 문제다. 그 뒤 `save()`가 실패하면 이번에는 React가 렌더 중이 아니므로 ErrorBoundary가 아니라 `handleSave`의 `try-catch`가 처리해야 한다.

| 구분 | stale closure | ErrorBoundary 범위 |
| --- | --- | --- |
| 핵심 질문 | 함수가 어떤 값을 읽는가? | 발생한 오류를 누가 잡는가? |
| 문제 | 자신이 태어난 렌더의 옛 값을 읽음 | 렌더 밖 오류라 boundary가 잡지 못함 |
| 해결 | dependency, 함수형 업데이트, `ref`, 구조 변경 | 해당 작업의 `try-catch`와 에러 상태 |

`ref`를 사용하면 비동기 함수가 최신 값을 읽게 만들 수는 있다. 하지만 비동기 오류를 ErrorBoundary가 잡게 만들지는 못한다.

```tsx
async function handleSave() {
  try {
    await save(nameRef.current) // 최신 값 읽기
  } catch {
    setSaveError(true)          // 비동기 오류 처리
  }
}
```

즉 최신 값 문제와 오류 처리 문제는 각각 해결해야 한다.

> 나중에 실행되는 함수는 최신 렌더의 값에도, 렌더용 오류 안전망에도 자동으로 연결되지 않는다.

관련 노트:

- [[react-stale-closure/react-stale-closure|React ref / stale closure / write-through 캐시]]
- [[react-async-request-race-generation-guard|오래된 비동기 응답을 막는 요청 세대 가드]]

---

## 오류 종류마다 복구 UI가 다르다

| 실패 | 화면 상태 | 담당 안전망 | 일반적인 복구 |
| --- | --- | --- | --- |
| 컴포넌트 렌더 예외 | 해당 트리를 그릴 수 없음 | ErrorBoundary | fallback 전체 화면, 새로고침 |
| 목록 fetch 실패 | 화면은 정상이고 데이터만 없음 | 화면의 `catch`와 에러 상태 | 목록 자리에 inline 재시도 |
| 버튼 저장 실패 | 기존 화면은 그대로 사용 가능 | 이벤트 핸들러의 `try-catch` | 버튼 상태 복구, toast 또는 inline 문구 |
| 예상한 빈 결과 | 요청은 성공했고 결과가 0건 | 정상 렌더 분기 | “아직 없어요” 빈 상태 |

데이터 로딩 실패 때문에 헤더와 탭까지 전부 없앨 필요는 없다. 화면 구조가 살아 있다면 실패한 목록 자리에만 `InlineLoadError`를 보여주는 편이 복구 범위에 맞다.

```tsx
try {
  const products = await getProducts()
  setProducts(products)
} catch {
  setProductsError(true)
}
```

```tsx
return productsError
  ? <InlineLoadError onRetry={retryProducts} />
  : <ProductList products={products} />
```

## 재시도가 같은 함수 재호출로 충분했던 이유

사이드 프로젝트의 데이터 로더는 성공한 결과만 메모리 캐시에 저장한다. 요청이 실패하면 에러를 보고한 뒤 다시 던지고 캐시는 빈 상태로 남는다.

```text
첫 요청 실패
  ↓
에러 분류 및 보고
  ↓
throw
  ↓
캐시는 여전히 비어 있음
  ↓
다시 시도 버튼
  ↓
같은 로드 함수 호출 → 서버에 다시 요청
```

따라서 별도의 캐시 초기화 절차 없이 같은 함수를 다시 호출하는 것만으로 재시도가 된다. 단, 실패 결과나 빈 배열을 캐시에 저장했다면 재호출이 서버로 가지 않을 수 있으므로 이 전제가 중요하다.

---

## 안쪽 안전망이 잡으면 바깥 안전망은 모를 수 있다

사이드 프로젝트에는 라우터의 페이지 단위 오류 화면과 앱 최상위 ErrorBoundary가 함께 있었다. 페이지 렌더 오류는 가까운 라우터 안전망이 먼저 잡기 때문에 바깥 ErrorBoundary까지 전파되지 않았다.

```text
페이지 렌더 오류
  ↓
라우터 안전망이 먼저 처리
  ├─ fallback 화면 표시
  └─ 바깥 ErrorBoundary에는 도착하지 않음
```

서버 오류 보고가 바깥 ErrorBoundary에만 연결되어 있으면 사용자는 오류 화면을 보지만 서버 로그에는 아무것도 남지 않는다. 그래서 라우터가 오류를 잡는 콜백에도 보고 함수를 연결해야 했다.

> 오류 화면을 보여주는 것과 오류를 기록하는 것은 별개의 책임이다.

## Spring 백엔드와 비교하면

ErrorBoundary를 Spring의 `@RestControllerAdvice`와 완전히 같은 것으로 보면 안 된다.

- `@RestControllerAdvice`는 요청 처리 중 컨트롤러 밖으로 전파된 예외를 HTTP 응답으로 바꾼다.
- ErrorBoundary는 React 렌더 트리에서 발생한 예외를 fallback UI로 바꾼다.
- 비동기 작업이 각자의 실행 경계 밖으로 빠져나가면 어느 쪽이든 적절한 전파와 처리 지점이 필요하다.

공통 원칙은 **안전망까지 오류가 실제로 전파되어야 안전망이 일할 수 있다**는 것이다.

## 복습 질문

- [ ] ErrorBoundary가 렌더 오류는 잡지만 클릭 핸들러나 비동기 요청 실패는 잡지 못하는 이유는 무엇인가?
- [ ] 데이터 로딩 실패에 전체 fallback보다 inline 재시도가 더 적절할 수 있는 이유는 무엇인가?
- [ ] 이 프로젝트에서 같은 로드 함수를 다시 호출하는 것만으로 재시도가 가능했던 전제는 무엇인가?
- [ ] 안쪽 라우터 안전망이 오류를 처리하면 바깥 ErrorBoundary의 서버 보고가 실행되지 않을 수 있는 이유는 무엇인가?
- [ ] ErrorBoundary와 Spring `@RestControllerAdvice`의 책임은 어떻게 다른가?

## 백지 복습 답변

### 1. 렌더 오류와 비동기 오류

내 답:

### 2. inline 재시도

내 답:

### 3. 캐시와 재호출

내 답:

### 4. 중첩 안전망의 관측 구멍

내 답:

### 5. Spring 예외 처리와의 차이

내 답:

## 한 줄 회고

- 헷갈렸던 점:
