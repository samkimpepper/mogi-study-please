---
tags:
  - ai
  - llm
  - index
---

# AI 노트 목차

> [!summary]
> LLM의 동작 원리, 한계, 에이전트 활용, Claude Code 이슈를 모아둔 노트 목록이다.
>
> 아직 파일 수가 많지는 않아서 하위 디렉토리로 쪼개기보다, 이 목차에서 주제별로 묶어 관리한다.

## LLM 동작 원리와 한계

LLM이 왜 자기 사고과정을 정확히 설명하기 어렵고, 왜 계산이나 상태 추적에서 한계가 생기는지 정리한 노트들이다.

| 노트 | 핵심 질문 |
| --- | --- |
| [왜 LLM은 자기 사고과정을 정확히 설명할 수 없는가](llm-cannot-explain-own-reasoning.md) | LLM이 말하는 이유 설명을 그대로 믿기 어려운 이유는 무엇인가? |
| [왜 LLM 사고과정은 스택트레이스처럼 찍기 어려운가](why-llm-has-no-stack-trace.md) | 프로그램의 stack trace와 LLM 내부 계산은 무엇이 다른가? |
| [Chain-of-Thought와 Console Log는 무엇이 다른가](chain-of-thought-vs-console-log.md) | CoT는 실제 실행 로그인가, 설명용 텍스트인가? |
| [LLM 컨텍스트와 튜링 머신 메모리 차이](llm-context-vs-turing-machine-memory.md) | 컨텍스트, self-attention, 명시적 메모리는 어떻게 다른가? |
| [LLM이 스도쿠 같은 백트래킹 문제에 약한 이유](llm-sudoku-backtracking-limit.md) | LLM은 왜 솔버를 짜는 건 잘해도 솔버 자체가 되는 건 약한가? |

## LLM 실수와 검증

LLM이 같은 실수를 반복하는 이유와, 프롬프트만 믿지 않고 검증 레이어를 두는 방법을 다룬다.

| 노트 | 핵심 질문 |
| --- | --- |
| [LLM이 같은 실수를 반복하는 이유](llm-repeated-mistakes-context-vs-weights.md) | 지적을 해도 왜 같은 실수가 반복될 수 있고, 어떻게 막아야 하는가? |

## 에이전트와 문서화

AI 에이전트를 코드베이스나 조직 지식과 함께 쓸 때 생기는 문제를 정리한다.

| 노트 | 핵심 질문 |
| --- | --- |
| [AI 에이전트와 의도 부채](intent-debt-and-agents.md) | 코드에 남지 않은 의도와 맥락은 에이전트 시대에 왜 부채가 되는가? |

## Claude Code와 디버깅 사례

Claude Code를 쓰면서 겪은 구체적인 버그나 디버깅 사례를 모은다.

| 노트 | 핵심 질문 |
| --- | --- |
| [외톨이 surrogate `\uD83A` 가 세션을 brick 시킨 버그](claude-code-prohan-surrogate-brick-bug.md) | 유효하지 않은 surrogate가 JSON 처리와 세션 상태에 어떤 문제를 만들었는가? |

## 나중에 분리할 기준

파일이 더 쌓이면 다음 기준으로 하위 디렉토리를 나눌 수 있다.

```text
notes/ai/llm-mechanics/
notes/ai/agents/
notes/ai/claude-code/
```

다만 지금은 파일 수가 적어서 경로를 깊게 만들기보다 이 목차로 관리한다.

## 복습 질문

- [ ] LLM 동작 원리와 한계 관련 노트들을 찾을 수 있다.
- [ ] LLM 실수와 검증 관련 노트를 찾을 수 있다.
- [ ] 에이전트와 문서화 관련 노트를 찾을 수 있다.
- [ ] Claude Code 디버깅 사례 노트를 찾을 수 있다.
- [ ] 스도쿠와 백트래킹 사례 노트를 찾을 수 있다.

## 한 줄 회고

- 헷갈렸던 점: 파일이 조금 늘었다고 바로 디렉토리를 나누기보다, 먼저 README 목차로 묶어도 충분하다는 점.
