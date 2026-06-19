# mogi-study-please

모기 공부해와.

## 공부 목록

- [git / GitHub 협업](study/git-github-collaboration.md)

## Obsidian 노트 시스템

이 레포는 백엔드 이직 공부 내용을 Obsidian에서 다시 읽기 쉽게 정리하는 용도로 쓴다.

### 폴더 역할

| 폴더/파일 | 용도 |
| --- | --- |
| `TIL.md` | 날짜별로 공부한 주제를 한 줄 체크리스트로 기록 |
| `notes/` | 문서나 글을 읽고 정리한 심층 노트 |
| `study/` | 큰 공부 주제별 정리 |
| `references/` | 원문, 참고 자료, 링크 정리 |
| `templates/` | 새 노트를 만들 때 복사해서 쓰는 템플릿 |
| `.obsidian/snippets/study-note.css` | Obsidian에서 노트를 보기 좋게 꾸미는 CSS snippet |

### 노트 작성 규칙

- 중요한 요약은 `> [!summary]`에 적는다.
- 개념 설명은 `> [!info]`에 적는다.
- 주의할 점은 `> [!warning]`에 적는다.
- 위험하거나 절대 하면 안 되는 것은 `> [!danger]`에 적는다.
- 면접 답변은 `> [!tip]`에 적는다.
- 헷갈리는 질문은 `> [!question]`에 적는다.
- 예시는 `> [!example]`에 적는다.
- 긴 설명은 접히는 callout인 `> [!note]-`에 넣는다.
- 코드블록은 실제 코드, 명령어, 출력 예시처럼 그대로 읽거나 복사할 필요가 있는 경우에만 쓴다.
- 개념 흐름, 비유, 단계 설명은 코드블록보다 일반 문단이나 인용문 `>`를 우선 사용한다.
- 핵심 문장이나 원문 맥락은 필요할 때 인용문 `>`로 정리하되, 너무 많이 남발하지 않는다.
- 필요하면 Mermaid 다이어그램을 추가한다.
- 마지막에는 반드시 복습 체크리스트를 둔다.

### 템플릿 사용법

- 일반 개념 정리: `templates/backend-note-template.md`
- 질문에서 출발한 정리: `templates/question-note-template.md`

새 노트를 만들 때 템플릿을 복사한 뒤 `{{title}}` 또는 `{{question}}`을 실제 제목으로 바꾼다.

### Obsidian CSS snippet 켜기

Obsidian에서 다음 경로로 이동한다.

```text
Settings -> Appearance -> CSS snippets
```

그다음 `study-note.css`를 활성화한다.
