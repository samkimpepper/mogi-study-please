# React ref / stale closure / write-through 캐시 공부 노트

> 백엔드(자바/스프링) 관점에서 React의 한글 IME 저장 버그를 이해한 정리. "타블로"를 입력하고 저장했는데 "타블"만 저장되던 버그가 출발점.

---

## 0. 한 줄 요약

저장 핸들러(`handleSubmit`)가 화면 상태(state) 대신 `ref`라는 별도 박스에서 값을 읽는다. 그 ref를 **언제 갱신하느냐**가 핵심이고, 갱신 경로를 하나라도 빠뜨리면 ref가 옛날 값으로 남아 저장이 깨진다. 리팩터의 본질은 ref 갱신 방식을 **"읽기 타이밍 의존(render-time sync)"**에서 **"쓰기 시점 write-through(setter wrapper)"**로 바꾼 것.

---

## 1. 문제 상황 — 왜 한글에서만 터지나

화장품 신상품 등록 화면. 텍스트 입력칸 4개(한글/영어 제품명, 한글/영어 색상명) 입력 후 저장 버튼.

버그: "타블로" 입력 후 저장 → "타블"만 저장됨. 마지막 "로"가 잘림.

### 한글의 특수성 (IME)

- 영어: 키 누르면 글자 **즉시 확정**.
- 한글: 자모를 **조합**해서 한 글자 완성. `ㄹ → 로 → (롤? 롱? 아직 모름)`
- 브라우저는 글자가 확정되기 전까지 **"조합 중(composition)"** 상태로 들고 있음. 확정 전이라 데이터(state)에 반영 안 됨.
- 사용자가 마지막 글자 조합 도중에 저장 버튼을 누르면, 그 글자는 아직 조합 중이라 저장 로직이 못 읽음.

→ 영어였으면 안 났을 버그. **조합형 입력기(IME) = 한글/일본어/중국어**만의 함정.

---

## 2. React 기본기 — 컴포넌트 함수는 계속 재실행된다

백엔드 직관과 가장 다른 부분.

- 백엔드 함수: `process()` 한 번 부르고 끝.
- React 컴포넌트 함수: **state가 바뀔 때마다 React가 함수를 처음부터 통째로 재실행.** 이 한 번의 재실행 = **render**.

```js
function Component() {
  const [shadeName, setShadeName] = useState("");
  console.log("실행됨!", shadeName);
  return <input ... />;
}
// "타" 입력 → 실행 / "타블" → 실행 / "타블로" → 실행 ... 매번 재실행
```

자바 비유: `Component()` 메서드를 React가 여러 번 호출하는 것. 호출할 때마다 내부 지역변수도 새로 생김.

```java
Component();  // 1회차: 지역변수 shadeName="타",   handleSubmit 새로 생성
Component();  // 2회차: 별개의 shadeName="타블",   handleSubmit 또 새로 생성
```

→ render마다 `shadeName`도, `handleSubmit`도 **새로 태어난다.** 1회차 변수와 2회차 변수는 이름만 같지 남남.

---

## 3. Stale Closure — 핵심 메커니즘

### 자바 람다 캡처와 같은 메커니즘 (직감 맞았음)

```java
String name = "타블";
Runnable r = () -> System.out.println(name);  // 정의 시점의 name 캡처
```

JS 클로저도 동일: 함수는 **정의되는 순간의 바깥 변수를 캡처**한다.

### 자바 final 강제 vs React stale의 차이

- **자바**: 변수 바뀌면 람다가 보는 값이 모호 → 아예 캡처 금지(`final`/effectively final 강제). 컴파일 타임에 차단.
- **React**: 막지 않음. 대신 `shadeName`이 **render마다 새로 태어나는 지역 상수**라서, 한번 캡처되면 사실상 final처럼 굳음.

```js
// render #3 — 이 실행의 shadeName
const shadeName = "타블";
const handleSubmit = () => { ...shadeName... };  // "타블" 캡처

// render #4 — 완전 별개 실행, 별개 상수
const shadeName = "타블로";                       // 다른 변수
const handleSubmit = () => { ...shadeName... };  // "타블로" 캡처한 새 함수
```

> render #3의 `shadeName`과 render #4의 `shadeName`은 **이름만 같은 남남.** render #3에서 태어난 `handleSubmit`이 캡처한 "타블"은 영원히 안 바뀜.

### 결론

**한 번 실행에 들어간 함수는 자기가 태어난 render의 값에 영원히 묶인다.** 도중에 state가 바뀌어 새 render가 생겨도, 이미 달리고 있는 옛날 함수는 그 미래 값을 못 본다.

자바 비유: 핸들러 진입 시점에 `String name = this.shadeName`로 복사해두면, 처리 도중 원본이 바뀌어도 내 지역 복사본은 안 바뀜.

---

## 4. ref — stale 탈출 장치

`ref`는 closure를 우회하는 장치. state와 성질이 정반대.

- **state**: render마다 새로 캡처되는 복사본 → stale 위험.
- **ref**: 모든 render가 공유하는 **단 하나의 가변 박스**. render 몇 번이든 같은 박스를 가리킴 → 항상 최신 가능.

자바 비유 (동작 원리에 가장 정확):

```java
final AtomicReference<String> ref = new AtomicReference<>("타블");
Runnable r = () -> System.out.println(ref.get());  // 박스를 캡처
ref.set("타블로");
r();  // "타블로" — 박스(참조)는 고정, 내용물(.current)은 최신
```

`final List`처럼 **참조는 고정인데 내용물은 바꿀 수 있는 박스.** `ref.current`가 그 내용물.

> 비교 메모: `ThreadLocal`이 떠오를 수 있는데 결이 다름.
> 
> - ThreadLocal 목적 = 스레드별 **격리**
> - useRef 목적 = render 간 **공유**(stale 회피) "여러 지점에서 같은 저장소에 접근" 느낌만 비슷하고 메커니즘은 다름. 동작 비유는 `AtomicReference`가 정확.

**단, 전제**: ref가 "항상 최신"이려면 그 박스가 **읽기 전에 갱신**돼 있어야 함. 이 전제가 깨지는 게 원래 버그.

---

## 5. 원래 버그 — render-time sync의 타이밍 함정

### blur + RAF: "로"를 살리려는 응급처치

```js
async function handleSubmit() {
  document.activeElement.blur();   // ① 포커스 뗌 → IME가 "로" 강제 확정 → onChange 유발
  await requestAnimationFrame();   // ② 한 틱 기다림
  const v = shadeName;             // ③ 값 읽음
  save(v);
}
```

의도: "로가 조합 중이라 안 들어갔네? blur로 강제 확정시키고, 한 틱 기다렸다가 읽으면 타블로 나오겠지." ※ blur가 **스스로 onChange를 유발**한다는 점이 포인트. 입력이 우연히 끼어드는 게 아니라 handleSubmit이 직접 부름.

### 옛날 ref 갱신 방식 (버그)

```js
function Component() {
  const shadeNameRef = useRef("");
  shadeNameRef.current = shadeName;   // ← render 본문에서 매 render마다 실행
}
```

발상: "render가 일어나면 이 줄이 돌아서 박스에 최신 state를 넣는다." → **render #4가 실제로 실행돼야만** 이 줄이 돈다는 게 함정.

### React 18 automatic batching이 깨뜨림

`setState`는 즉시 render를 일으키지 않음. **나중에 모아서(batch) 처리.**

```java
// setState는 즉시 반영(X)
this.shadeName = "타블로";
// 실제로는 "나중에 다시 그려줘" 큐에 등록만(O)
renderQueue.add(() -> rerender());
```

타임라인:

```
① blur() → setShadeName("타블로")
   → render #4를 큐에 "예약"만 함. 아직 실행 X. 박스 아직 "타블".
② await RAF → 한 틱
   → 이 틱이 끝나고 ③이 도는 게, 큐의 render #4가 도는 것보다 먼저일 수 있음 ⚠
③ const v = shadeNameRef.current
   → render #4가 안 돌아서 'ref.current = shadeName' 줄이 실행된 적 없음
   → 박스 여전히 "타블" ❌
```

> 핵심: ref 박스는 제대로 뒀는데, **박스 채우는 시점이 "render 실행"에 묶여 있었음.** 그 render 실행이 batching으로 미뤄지고, 읽기가 추월 → 빈/옛날 박스 읽음. render-time sync = "읽을 때마다 캐시 새로고침" — 읽는 타이밍에 의존.

---

## 6. 고친 방식 — setter wrapper (write-through 캐시)

### 변경 1: 대입문을 render 본문 → setter wrapper로 (실행 트리거 교체)

```js
const shadeNameRef = useRef('');

function setShadeNameSync(v) {
  shadeNameRef.current = v;   // ① 박스 즉시 대입 — render와 무관한 그냥 대입문
  setShadeName(v);            // ② state 갱신 예약 (render는 알아서 나중에)
}
```

핵심은 "위치 이동"이 아니라 **실행 트리거 교체**:

- 옛날: 대입 트리거 = "render가 일어남" → batching 타이밍에 의존
- 새것: 대입 트리거 = "값을 씀(onChange 등)" → 동기 즉시 실행, render 안 기다림

```java
void setShadeNameSync(String v) {
    this.shadeNameRef = v;   // ① 그냥 필드 대입. 즉시. render 무관.
    queueRerender(v);        // ② 화면 다시 그리기는 큐 등록(나중)
}
```

`this.shadeNameRef = v`는 render를 실행하는 게 아니라 **그냥 대입.** render #3 멈추거나 #4 당기는 거 아님.

### 변경 2: 모든 쓰기 경로를 wrapper로 교체

대입문을 wrapper에 넣기만 하면 끝이 아님. **모든 값 쓰기 경로가 wrapper를 거쳐야** 효과.

```js
setShadeName(v)       // raw 직접 호출 (X)
setShadeNameSync(v)   // wrapper 경유 (O)
```

쓰기 경로는 onChange만이 아님: 폼 열 때 초기화, 브랜드 변경 시 리셋, 라인 picker 자동채움, "변경"/"뒤로" 버튼 등. 한 곳이라도 raw setter로 남으면 그 경로는 박스를 안 채움 → 거기서만 stale 버그.

### 고친 코드 타임라인

```
[입력 onChange] → setShadeNameSync("타블로")
   ① ref.current = "타블로"   ← 이 순간 박스 채워짐. 끝.
   ② setShadeName("타블로")   ← render는 알아서 나중에. 신경 안 씀.

[저장] → handleSubmit
   → ① blur() → IME "로" 확정 유발 → onChange → setShadeNameSync → ref.current="타블로" 즉시
   → ③ ref.current 읽음 → "타블로" ✅
   (render #4가 돌았든 안 돌았든 무관. render 안 보고 박스만 봄)
```

> 새 코드에선 "render #3의 값" 개념 자체가 사라짐. 옛날엔 "어느 render의 값을 읽나"가 문제였는데, 새 코드는 그 질문을 없앰 → **render 안 보고 박스만 본다.**

### write-through 캐시 비유 (백엔드 그대로)

```java
void save(String v) {
    cache.put(key, v);   // 쓸 때 캐시도 같이  (= shadeNameRef.current = v)
    db.update(v);        // 쓸 때 DB도 같이    (= setShadeName(v))
}
// 읽을 때 cache.get(key) → 항상 최신. 타이밍 걱정 0.
```

- render-time sync = "읽을 때 캐시 새로고침" → 읽기 타이밍 의존
- setter wrapper = write-through → 쓸 때 캐시(ref)+저장소(state) 한 묶음 갱신 → 읽는 쪽 타이밍 무관

※ blur + RAF는 그대로 유지. wrapper가 주 방어선, blur/RAF는 다른 IME 케이스 보조 안전망(이중 방어).

---

## 7. 회귀 위험 & 검증 — 우회 경로 없나 전수조사

write-through 캐시의 약점은 백엔드와 동일: **누가 캐시 안 거치고 저장소에 직접 쓰면 캐시가 썩는다.**

옛날 `ref.current = state` 줄은 모든 state 변경 후 render에서 ref를 갱신해줬음(좋든 나쁘든 전부 커버). 이 줄을 지우는 순간, raw setter를 직접 부르는 코드는 ref를 갱신 안 함 → state만 바뀌고 ref는 옛날 값.

예: 라인 picker로 기존 라인 선택 → `setProductNameKr("젤핏 틴트")`가 raw면 → state는 "젤핏 틴트", ref는 "" → handleSubmit이 `ref.current` 읽으면 "" → `!finalProductNameKr` 가드가 저장을 조용히 거부.

### 검증 방법: setter 호출 전수 grep

```bash
git show pr-162:.../NewProductSheet.tsx \
  | grep -nE 'setProductNameKr|setProductNameEn|setShadeName|setShadeNameEn'
```

분류:

- useState 선언 줄 → setter 정의. 정상.
- `...Sync` wrapper 본문 → wrapper 안의 raw 호출. 정상(유일하게 허용되는 raw 호출 지점).
- 직접 setState 경로(폼 초기화/리셋/자동채움 등) → 전부 `...Sync`인지 확인.
- Field의 onChange prop → 전부 `...Sync`인지 확인.

→ useState 선언과 wrapper 본문 빼면 **raw 직접 호출 0건** = 모든 쓰기가 write-through 통과 = ref가 stale로 남을 구멍 없음 → 회귀 없음.

---

## 8. 최종 정리 (백엔드 한 줄)

> 캐시(ref)와 저장소(state)를 가진 시스템에서, 갱신 로직을 **"읽기 타이밍 의존(render-time sync)"**에서 **"쓰기 시점 write-through(setter wrapper)"**로 바꾼 리팩터. 이런 변경의 단골 회귀는 **write-through 레이어를 우회하는 직접 쓰기.** 그래서 리뷰는 "모든 쓰기가 wrapper를 통과하는가"를 **전수로** 확인하는 것 — 한 곳만 빠져도 조용히 깨지니까.

### 개념 대응표

|React|하는 일|백엔드 비유|
|---|---|---|
|`useState`|값 보관 + 바뀌면 re-render 트리거|화면에 바인딩된 모델 필드 / DB|
|`useRef`|값 보관, 바뀌어도 re-render 안 함. render 간 공유되는 단일 박스|살아남는 가변 캐시 박스 (`AtomicReference`)|
|setter (`setShadeName`)|state를 바꾸는 유일한 합법 통로|`repository.save()`|
|`handleSubmit`|저장 버튼 핸들러|Controller의 `@PostMapping`|
|stale closure|옛 render의 함수가 옛 값에 묶임|진입 시 지역변수로 복사해둔 값 / final 캡처|
|render-time sync|render 실행 때 ref 갱신|read-time 캐시 새로고침|
|setter wrapper|쓸 때 ref+state 같이 갱신|write-through 캐시|
|batching|setState가 즉시 render 안 하고 큐에 모음|비동기 큐잉, 지연 처리|

### 학습 흐름 한 줄

closure(왜 state 못 믿나) → 그래서 ref → render-time sync의 타이밍 함정(batching) → write-through로 전환 → 우회 경로 없나 전수조사. **백엔드 캐시 일관성 문제와 구조가 동일.**