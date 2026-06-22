---
tags:
  - ai
  - llm
  - transformer
---

# Transformer와 Attention 기본 개념

> [!summary]
> Transformer는 문장을 토큰 단위로 쪼갠 뒤, 각 토큰이 다른 토큰들을 얼마나 참고해야 하는지 attention으로 계산하면서 의미를 업데이트하고 다음 토큰을 예측하는 신경망 구조다.
>
> GPT 같은 LLM은 이 Transformer 구조를 여러 층 쌓아 만든 모델이라고 보면 된다.

## 왜 공부했나

> [!question]
> 처음 헷갈렸던 질문:
>
> - Transformer 기반 모델이라는 말은 정확히 무슨 뜻인가?
> - attention은 토큰이 다른 토큰을 "참고한다"는 말과 어떻게 연결되는가?
> - LLM이 내부 사고과정을 로그처럼 설명하기 어려운 이유와 Transformer 구조는 어떤 관련이 있는가?

## 한 줄로 말하면

Transformer의 핵심은 self-attention이다.

> 각 토큰이 문장 안의 다른 토큰들을 보고, "내 의미를 업데이트하려면 누구를 얼마나 참고해야 하지?"를 계산하는 구조다.

예를 들어 이런 문장이 있다고 하자.

> 철수는 사과를 먹었다. 그는 배가 고팠다.

여기서 "그는"이 누구를 가리키는지 알려면 앞의 "철수"를 봐야 한다. Transformer는 이런 관계를 사람이 직접 규칙으로 박아두는 대신, attention 계산으로 각 토큰 사이의 관련도를 학습한다.

## 토큰이 서로를 참고한다는 뜻

LLM은 문장을 글자 그대로 이해하지 않는다. 먼저 입력을 토큰으로 쪼갠다.

```text
"나는 백엔드 개발자야"
-> ["나는", "백", "엔드", "개발", "자", "야"]
```

실제로는 모델의 tokenizer에 따라 더 복잡하게 쪼개질 수 있다.

각 토큰은 숫자 벡터로 바뀐다.

```text
"개발자" -> [0.12, -0.03, 0.88, ...]
```

그다음 Transformer는 각 토큰마다 문장 안의 다른 토큰들을 얼마나 봐야 하는지 계산한다.

```text
그는 -> 철수 0.8
그는 -> 사과 0.1
그는 -> 먹었다 0.1
```

이 숫자는 직관적으로 "현재 토큰이 다른 토큰을 얼마나 참고하는가"에 가깝다. 이게 attention의 기본 감각이다.

> [!info]
> self-attention은 이미 주어진 입력 토큰들 사이에서 서로의 관련도를 계산하는 방식이다.
>
> 외부 DB를 검색하는 것이 아니라, 현재 컨텍스트 안의 토큰들이 서로를 보는 방식에 가깝다.

## 백엔드 개발자 관점의 비유

Transformer의 attention을 백엔드 코드에 억지로 비유하면, 현재 토큰을 처리할 때 전체 context에서 관련 있는 토큰들을 찾아 가중합하는 함수처럼 볼 수 있다.

```python
def process(current_token, context):
    relevant_parts = search_relevant_tokens(current_token, context)
    return transform(current_token, relevant_parts)
```

토큰마다 이런 일이 일어난다고 보면 된다.

```python
for token in sentence:
    for other_token in sentence:
        score = relevance(token, other_token)

    token_state = weighted_sum(other_tokens, scores)
```

핵심은 모든 토큰이 다른 토큰과의 관계를 보면서 자기 상태를 업데이트한다는 점이다.

> [!warning]
> 이 비유는 이해를 돕기 위한 단순화다.
>
> 실제 Transformer는 검색 함수처럼 사람이 읽기 쉬운 로직으로 동작하는 것이 아니라, 거대한 행렬 연산과 학습된 가중치로 관련도를 계산한다.

## Transformer 이전 구조와 차이

예전에는 RNN, LSTM 같은 구조를 많이 썼다. 이 구조는 문장을 대체로 왼쪽에서 오른쪽으로 순차 처리한다.

```text
나는 -> 백엔드 -> 개발자 -> 야
```

문장이 길어지면 앞부분 정보가 뒤로 갈수록 희미해지기 쉬웠다.

Transformer는 입력 토큰들이 서로를 한꺼번에 참고할 수 있다.

```text
나는    백엔드    개발자    야
 ↓        ↓        ↓       ↓
서로 전체를 attention으로 참고
```

그래서 긴 문맥 안의 관계를 잡기 좋아졌고, 병렬 처리에도 유리해서 대규모 학습에 잘 맞았다.

## GPT는 decoder-only Transformer다

Transformer는 원래 encoder와 decoder 구조로 제안되었다. GPT류 모델은 보통 그중 decoder 쪽 구조를 중심으로 한 decoder-only Transformer다.

GPT가 하는 일을 아주 단순하게 말하면 이렇다.

> 지금까지의 토큰들을 보고 다음 토큰을 예측한다.

예를 들어 다음 입력이 있다.

```text
입력: "오늘 점심은 김치"
예측: "찌개"
```

그다음에는 새로 생성한 토큰까지 포함해서 다시 다음 토큰을 예측한다.

```text
입력: "오늘 점심은 김치찌개"
예측: "를"
```

이 과정을 반복하면서 문장이 생성된다.

> [!info]
> GPT는 이전 토큰들을 보고 다음 토큰의 확률분포를 계산하는 거대한 Transformer라고 볼 수 있다.

## 그럼 생각은 어디에 있나

Transformer 내부에는 사람이 보는 로그처럼 명시적인 절차가 저장되어 있지 않다.

```text
1단계: 문제를 읽는다
2단계: 변수 x를 둔다
3단계: 계산한다
4단계: 답을 낸다
```

이런 식의 실행 로그가 있는 것이 아니다.

내부에서는 대략 이런 수치 변환이 반복된다.

```text
토큰 벡터 -> attention -> MLP -> 다음 층 -> attention -> MLP -> ...
```

그래서 "모델이 내부에서 무슨 생각을 했는지 로그로 찍자"가 어렵다. 찍을 수 있는 것은 주로 입력 토큰, 출력 토큰, 확률값, attention weight 일부, 중간 activation 일부 같은 숫자들이다.

> [!warning]
> 모델이 말로 출력하는 "추론 과정"은 내부 계산의 원본 로그라기보다, 모델이 생성한 설명 텍스트에 가깝다.
>
> 이 점은 chain-of-thought가 console log와 다르다는 내용과 연결된다.

## 사람의 지능을 모델 설계에 반영한다는 뜻

"우리의 지능을 더 많이 반영할수록, 최종 출력물이 필요한 변환을 더 잘 모방할 수 있다"는 말은 모델 설계에 사람이 알고 있는 좋은 구조를 넣으면 성능이 좋아질 수 있다는 뜻에 가깝다.

예를 들면 다음과 같다.

| 사람이 아는 구조 | 모델 설계에 넣은 것 |
| --- | --- |
| 문장 안의 단어들은 서로 관계가 있다 | attention |
| 토큰 순서는 중요하다 | positional encoding |
| 중간 결과를 적으면 추론이 좋아질 수 있다 | chain-of-thought, scratchpad |
| 모델 혼자 못하는 계산은 도구가 잘한다 | tool use, code execution |

이런 설계상의 힌트를 귀납적 편향이라고 볼 수 있다. 모델이 아무 구조 없이 모든 것을 처음부터 배우게 하기보다, 잘 맞는 계산 구조를 미리 넣어주는 것이다.

## Attention의 경직성

Transformer의 attention은 강력하지만 일반적인 프로그램 실행기처럼 자유롭지는 않다.

백엔드 코드에서는 입력에 따라 반복 횟수를 바꾸고, 중간 상태를 저장하고, 조건 분기를 할 수 있다.

```python
while not solved:
    try_strategy()
    update_state()
```

하지만 기본 Transformer는 대체로 다음 제약 안에서 답을 만들어야 한다.

- 정해진 층 수
- 정해진 계산 패턴
- 정해진 context window
- 각 토큰마다 고정된 양의 계산

그래서 Transformer는 진짜 알고리즘 실행기라기보다, 학습된 패턴으로 알고리즘처럼 보이는 변환을 수행하는 경우가 많다.

> [!danger]
> Transformer가 "그냥 암기만 한다"는 뜻은 아니다.
>
> 새로운 문장 이해, 처음 보는 코드 수정, 낯선 에러 메시지 분석처럼 일반화도 한다. 다만 정확한 산술, 긴 단계 추론, 중간 상태가 많은 알고리즘, 훈련 분포와 크게 다른 입력에서는 불안정해질 수 있다.

## 실무에서 LLM 시스템을 설계할 때

Transformer 기반 LLM은 언어 패턴, 설명, 요약, 코드 작성, 유사 사례 적용에 강하다. 하지만 정확한 상태 계산이나 검증이 필요한 작업은 모델 하나만 믿기보다 외부 도구와 묶는 편이 낫다.

| 필요한 일 | 더 안정적인 구조 |
| --- | --- |
| 최신 정보 확인 | LLM + 검색 |
| 정확한 계산 | LLM + 계산기 또는 코드 실행 |
| 코드 변경 검증 | LLM + 테스트 + 린터 |
| JSON 형식 보장 | LLM + schema validator |
| 복잡한 추론 검증 | LLM + verifier 또는 다중 샘플링 |

> [!tip]
> 실무 감각으로 말하면, LLM은 판단과 설명을 잘하고 외부 도구는 정확한 실행과 검증을 잘한다.
>
> 그래서 좋은 LLM 시스템은 모델을 똑똑한 함수 하나로 보는 것이 아니라, 검색기, 실행기, 검증기와 함께 묶인 애플리케이션으로 설계한다.

## 관련 노트

- [Chain-of-Thought와 Console Log는 무엇이 다른가](chain-of-thought-vs-console-log.md)
- [LLM 컨텍스트와 튜링 머신 메모리 차이](llm-context-vs-turing-machine-memory.md)
- [왜 LLM 사고과정은 스택트레이스처럼 찍기 어려운가](why-llm-has-no-stack-trace.md)
- [LLM이 스도쿠 같은 백트래킹 문제에 약한 이유](llm-sudoku-backtracking-limit.md)

## 복습 질문

- [ ] Transformer를 한 문장으로 설명할 수 있는가?
- [ ] self-attention이 "토큰이 서로를 참고한다"는 말과 어떻게 연결되는지 설명할 수 있는가?
- [ ] "그는"이라는 토큰이 앞의 "철수"를 참고한다는 예시로 attention을 설명할 수 있는가?
- [ ] 토큰화, 벡터화, attention, 다음 토큰 예측의 흐름을 순서대로 말할 수 있는가?
- [ ] RNN/LSTM의 순차 처리와 Transformer의 병렬적인 attention 구조 차이를 설명할 수 있는가?
- [ ] GPT가 decoder-only Transformer라는 말의 의미를 설명할 수 있는가?
- [ ] "지금까지의 토큰들을 보고 다음 토큰을 예측한다"는 말로 GPT의 생성 방식을 설명할 수 있는가?
- [ ] chain-of-thought가 내부 계산 로그가 아닌 이유를 Transformer 구조와 연결해 설명할 수 있는가?
- [ ] attention weight나 activation을 볼 수 있어도 사람이 이해하는 사고과정 로그가 바로 되지 않는 이유를 설명할 수 있는가?
- [ ] "사람의 지능을 모델 설계에 반영한다"는 말을 attention, positional encoding, tool use 예시로 설명할 수 있는가?
- [ ] attention이 강력하지만 일반 프로그램의 while문이나 mutable state와 다른 이유를 말할 수 있는가?
- [ ] LLM에 검색, 계산기, 검증기를 붙이는 이유를 attention의 장점과 한계로 설명할 수 있는가?

## 한 줄 회고

- 헷갈렸던 점: Transformer는 문장을 이해하는 규칙 모음이라기보다, 토큰 벡터들이 attention과 MLP를 여러 층 거치며 다음 토큰 확률분포로 변환되는 구조라는 점.
