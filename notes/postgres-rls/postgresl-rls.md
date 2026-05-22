# Postgres RLS의 "조용한 0 rows" 함정 정리 🪤

> 스프링 개발자가 Supabase/Postgres 처음 쓸 때 제일 먼저 데이는 곳. 핵심만 한 줄로: **권한 없으면 에러가 터질 거라는 직관이 RLS에선 깨진다.**

---

## 0. 한 줄 결론 (이것만 기억해도 됨) ⭐

```
Postgres RLS에서 UPDATE/DELETE 권한 거부는
예외(Exception)가 아니라 조용한 "0 rows"로 온다.
```

스프링에서 권한 없으면 `AccessDeniedException` 터지는 게 당연하지? RLS는 **안 터진다.** 그냥 "0개 바꿨음" 하고 조용히 넘어간다. 이게 전부의 시작이다.

---

## 1. 먼저 알아야 할 배경 — soft delete

문제의 함수는 진짜 `DELETE`가 아니라 `is_deleted` 플래그를 켜는 **soft delete**다.

```js
.update({ is_deleted: true })
```

스프링에서도 흔히 쓰는 패턴이라 익숙할 거다. 행을 실제로 지우지 않고 "지워진 셈 침" 플래그만 켠다. 조회는 전부 `.eq('is_deleted', false)`로 살아있는 것만 가져온다.

JPA로 치면 `@SQLDelete`로 `UPDATE ... SET deleted = true` 거는 거랑 같은 개념.

---

## 2. 직관이 깨지는 지점 — "권한 체크"는 사실 두 종류다 🧠

이걸 분리 못 하면 평생 헷갈린다. 권한 체크에는 **레이어가 두 개** 있다.

### (1) 테이블 단위 권한 (`GRANT`) — 네 직관이 맞는 곳 ✅

> "너 이 _테이블_에 UPDATE 할 수 있어 없어?"

행 내용이랑 무관하게 테이블 통째로 보는 권한. **이건 Postgres도 에러 뱉는다.** 에러코드 `42501 insufficient_privilege`. MySQL, Oracle 다 똑같이 에러 낸다. 네 직관(권한 없으면 터진다)이 정확히 맞는 영역.

스프링 비유: **메서드에 `@PreAuthorize` 걸어둔 거.** 아예 못 들어옴 → 예외.

### (2) 행 단위 필터 (RLS) — 직관이 깨지는 곳 ⚠️

> "테이블 접근은 되는데, 그 안에서 _어떤 행_까지 보이고 건드릴 수 있냐?"

이게 **Row Level Security**고, 얘는 권한 체크가 아니라 **사실상 자동으로 붙는 `WHERE` 필터**로 설계됐다.

---

## 3. RLS = 보이지 않는 WHERE 한 줄 👻

RLS 정책을 켜면 Postgres가 네 모든 쿼리에 `WHERE`를 몰래 한 줄 붙인다고 보면 된다.

```sql
-- 네가 쓴 쿼리
UPDATE comparison_pairs SET is_deleted = true WHERE id = 5;

-- Postgres가 RLS 붙여서 실제로 실행하는 것
UPDATE comparison_pairs SET is_deleted = true
WHERE id = 5 AND (RLS_정책_조건);   -- ← 이 줄이 자동으로 붙음
```

자, 여기가 핵심이다. 만약 `id=5` 행이 RLS 조건을 통과 못 하면? 저 합쳐진 `WHERE`는 **그냥 매칭되는 행이 없는 거**가 된다.

`UPDATE ... WHERE id = 999` (없는 ID) 날렸을 때 에러 안 나고 `0 rows` 나오지? 그거랑 **문법적으로 완전히 똑같은 상황**이 돼버린다.

> Postgres 입장: "권한을 거부"한 게 아니라 "조건에 맞는 행이 없었다." → 에러를 안 내는 게 아니라, **에러를 낼 자리 자체가 없는 거다.** 🤯

### JDBC로 치면

```java
int rowsAffected = stmt.executeUpdate();  // 0 리턴
// SQLException? 안 터짐. 그냥 0.
```

너도 `rowsAffected == 0`이 원래 애매한 거 알잖아:

- "매칭되는 행이 없음" or
- "이미 그 값이라 안 바뀜"

**RLS는 여기에 세 번째 원인을 끼워넣는다 → "매칭은 됐는데 너는 못 건드림."** 근데 셋 다 똑같이 `0`으로 온다. 구분 불가. 😵

---

## 4. 왜 이렇게 설계했냐? — 일부러 그런 거다 (정보 유출 방지) 🔐

"아니 권한 없으면 그냥 에러 내면 되잖아?" 싶지? 근데 RLS가 "너 이 행 권한 없어!"라고 에러를 뱉으면 이런 공격이 가능해진다.

공격자가 `UPDATE ... WHERE email = 'ceo@company.com'`를 날려보고:

|응답|공격자가 알아내는 것|
|---|---|
|에러 남|"아, 그 행이 _존재하긴_ 하는구나" (권한만 없는 거)|
|0 rows|"그런 행 자체가 없구나"|

**에러 vs 0건의 차이만으로 "그 데이터가 존재하는지"를 알아낼 수 있게 된다.** 멀티테넌트 SaaS에서 이건 치명적인 정보 유출이다.

그래서 RLS는 일부러 **"넌 권한 없어"와 "그런 거 없어"를 똑같이 0건으로 뭉뚱그려서** 구분 못 하게 만들었다. 보이면 안 되는 행은 _애초에 존재하지 않는 것처럼_ 행동하게.

> 즉, 코드에서 겪은 "둘이 구분 안 됨" 문제는 **버그가 아니라 RLS의 핵심 기능**이다. 보안 기능이 앱 로직 입장에선 함정이 된 것뿐. 😇

---

## 5. 그래서 원래 뭐가 버그였냐 🐛

옛날 코드:

```js
if (!updated || updated.length === 0) {
  return { ok: false, reason: 'not_found' }   // 0 rows = 무조건 '없음'
}
```

`0 rows`를 **무조건 "없음(not_found)"**으로 단정했다. 근데 0 rows의 진짜 원인은 둘:

- **정말 없음** — 그 페어가 원래 없거나 이미 정리됨
- **RLS 차단** — 페어는 멀쩡히 살아있는데 정책이 이 유저의 UPDATE를 막음

### 왜 위험했냐 — 거짓 안심 😱

화면 메시지가 이렇게 분기돼 있었다:

- `not_found` → **"이미 정리된 링크예요"** (안심시키는 문구)
- 그 외 → "정리가 안 됐어요"

RLS가 막아서 0 rows가 났는데 → `not_found`로 분류 → 사용자한테 "이미 정리됐어요"라고 말함.

근데 **실제로는 링크가 그대로 살아있다.** 사용자는 "정리됐구나" 하고 넘어가는데 새로고침하면 링크가 멀쩡히 다시 보임. **데이터를 못 믿게 되는 순간.** 💀

---

## 6. 고친 방법 — UPDATE 직전에 진단용 SELECT 한 방 🔍

핵심 아이디어: `UPDATE`가 0을 돌려준 뒤 **"그럼 너 이 행 _보이긴_ 보여?"라고 SELECT로 되묻는다.**

```js
// prefetch SELECT — UPDATE가 0-rows일 때 "정말 없음"과 "RLS 차단"을 가른다
const { data: existing, error: existingError } = await supabase
  .from('comparison_pairs')
  .select('id')
  .or(pairFilter)
  .eq('is_deleted', false)
  .limit(1)
```

### 왜 이게 통하냐? 🤔

**RLS 정책은 명령(SELECT / UPDATE / DELETE)마다 따로 적는다.** 공유 데이터라서 "SELECT는 누구나 허용, UPDATE만 제한" 같은 구성이 가능하다.

그래서 두 쿼리 결과 조합으로 원인이 갈린다:

|prefetch SELECT|UPDATE|결론|origin|
|---|---|---|---|
|행 있음 (1+)|0 rows|보이는데 못 바꿈 = RLS가 UPDATE만 차단|`rls_blocked`|
|행 없음 (0)|0 rows|진짜로 없음|`no_row`|

코드:

```js
if (!updated || updated.length === 0) {
  let origin
  if (!existingError) {
    origin = existing == null || existing.length === 0 ? 'no_row' : 'rls_blocked'
  }
  return { ok: false, reason: 'not_found', origin }
}
```

> 비유하면 **진단 probe**다. UPDATE가 0을 돌려준 뒤 "그럼 너 이 행 보이긴 보여?"라고 SELECT로 되묻는 것. 🩺

---

## 7. 보너스 ① — 42501이라는 또 다른 권한 거부 🚪

위에서 "RLS 차단은 0 rows"라고 했는데, 권한 문제가 **진짜 에러로 터지는** 경우도 있다.

```js
if (updateError.code === '42501') {
  return { ok: false, reason: 'error', origin: 'permission_denied' }
}
```

`42501` = Postgres 에러 코드 `insufficient_privilege`. 정리하면 권한 거부가 **두 형태**로 온다:

|상황|결과|origin|
|---|---|---|
|테이블 권한(GRANT) 없음 / `WITH CHECK` 위반|`42501` 에러로 **터짐**|`permission_denied`|
|RLS `USING` 정책이 행을 가림|**에러 없이 0 rows**|`rls_blocked`|

같은 "권한 문제"라도 운영자한텐 다른 신호라서 코드가 두 군데서 따로 잡는다.

> 한 줄 비유: **`GRANT`은 "문 잠금"**(못 들어오면 쾅 막힘 = 에러), **RLS는 "투명 망토"**(들어와도 어떤 물건은 네 눈에 안 보임 = 0건). 🥷

---

## 8. 보너스 ② — 정직한 telemetry: 모르면 모른다고 한다 🤷

이게 사실 백엔드 개발자가 꼭 새겨야 할 태도다.

### 진단 SELECT 자체가 실패하면?

```js
if (!existingError) {   // ← prefetch가 성공했을 때만 분류
  origin = ...
}
// existingError면 origin은 undefined 그대로
```

진단 쿼리가 실패하면 행이 있는지 없는지 **모른다.** 그러면 `no_row`로 찍지 않고 `undefined`로 둔다.

### 핵심 원칙 ⭐

```
거짓말하는 telemetry는 비어있는 telemetry보다 나쁘다.
```

주니어 본능은 "대시보드 빈칸 보기 싫으니 뭐라도 값 채우자"인데, 틀린 값을 채우면 운영자가 그 거짓 데이터로 의사결정을 한다.

네트워크 끊김을 `rls_blocked`로 적어버리면 → 운영자가 대시보드 보고 "RLS 정책에 문제 있나?" 하고 **엉뚱한 데를 판다.**

`undefined`(= "모름")는 부끄러운 게 아니라 **정직한 값**이다. 😌

---

## 9. 다른 DB는 어떤데? (참고) 🗄️

네 직관 "다른 DB는 다 에러 뱉지 않냐?"에 대한 답:

|DB|행 수준 보안|거동|
|---|---|---|
|**Postgres**|RLS (네이티브)|권한 없는 행 → 조용한 0건|
|**Oracle**|VPD (Virtual Private Database)|똑같이 필터 방식 → 0건|
|**SQL Server**|RLS (있음)|똑같이 필터 방식|
|**MySQL**|❌ 네이티브 없음|VIEW나 앱 `WHERE`로 흉내 → 결국 0건|

요점: **"행 수준 보안 = WHERE 필터"라는 본질** 때문에 어디서든 "권한 없음"이 "행 없음"과 구분 안 되는 건 똑같다. 이건 Supabase 변덕도, Postgres 변덕도 아니고 행 수준 보안 개념 자체의 성질이다.

(MySQL은 RLS 자체가 없어서, 보통 앱 코드에서 `WHERE tenant_id = ?`를 손으로 붙인다. 차이는 **"누가 필터를 강제하냐"** — RLS는 DB가 강제해서 개발자가 깜빡해도 못 빠져나가고, MySQL 앱-WHERE 방식은 한 줄 빠뜨리면 사고 난다.)

---

## 10. 최종 요약 📌

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Postgres RLS에서 UPDATE/DELETE 권한 거부는                 │
│    예외가 아니라 조용한 "0 rows"로 온다.                      │
│    → try/catch로 인증 실패 다 잡았다고 착각하면 안 됨.        │
│                                                              │
│ 2. RLS는 "권한 체크"가 아니라 "자동 WHERE 필터"다.           │
│    그래서 권한 없는 행 = 매칭 안 되는 행 = 0건.              │
│                                                              │
│ 3. 일부러 그렇게 설계됨 (존재 여부 유출 방지).               │
│                                                              │
│ 4. 0 rows는 "없음 / 이미 그 상태 / 권한 막힘"이              │
│    전부 겹치는 애매한 신호다.                                │
│    → 원인 가르려면 같은 조건의 진단 SELECT가 필요.          │
│                                                              │
│ 5. 진단조차 실패하면? 추측해서 라벨 붙이지 말고             │
│    undefined로 정직하게 둔다.                                │
│    (거짓 telemetry > 빈 telemetry, 아님 그 반대 ㅋ)          │
└─────────────────────────────────────────────────────────────┘
```

> 스프링 직관 한 줄 교정: **`@PreAuthorize`(=GRANT)는 예외 던지지만, RLS(=행 필터)는 조용히 0건 준다.** 레이어가 다르다. 🎯