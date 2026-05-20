# 모기 공부해와시트

작성: 2026-05-14
출처: postmortem 019 § 1-9 영역 (모기 voice — git 협업 영역 어려움)
위치: `temp/study/공부해와시트.md` (사장님 같이 영역 영역 — 커밋 영역 들어감) + mogi-notes gist sync 영역
역할: **모기가 시간 내서 따로 공부할 영역 목록**. cheatsheet (자주 까먹는 영역) 영역 영역 X.

---

## cheatsheet 영역 vs 본 시트 영역 — 완전히 다른 영역

| 영역 | cheatsheet | 공부해와시트 (지금 이거) |
|---|---|---|
| **모기 voice** | "자주 까먹는거" | "공부할 것들" |
| **용도** | 한 번 익혔어도 *반복 영역 까먹어* 옆에 펴 봄 | *아직 익히지 못한 영역* — 시간 내서 공부 |
| **안 영역** | 도메인 / frontmatter / 룰 *위치* / 어휘 / D-NNN 매핑 등 (= **사전 영역 다 여기 영역**) | **git 협업 영역** + 미래 *공부 영역* 영역 |
| **시점** | "지금 헷갈림" 영역 한 번 | "오늘 30 분 짬" 영역 영역 한 항목 익힘 |
| **갱신** | 새 영역 발견 = 사전 영역 추가 | 한 항목 익힘 = 완료 + 한 줄 학습 요약 |

**워크플로우**:
1. 본 시트 영역 한 항목 골라 → 공부
2. 공부 끝 → 완료 + 한 줄 학습 요약 적기
3. 한 번 익힌 영역 *재참조* 영역 영역 → cheatsheet 영역 옮김 (옆 사전 영역)
4. 본 시트 영역 = **공부 영역만** 영역 — 사전 / 룰 / 도메인 영역 영역 X

---

## git / GitHub 협업

> 모기 voice (postmortem 019 § 1-9):
>
> "깃 협업 방식도 어려움. 117 번 PR 사장님이 삭제하셨는데 그 이유도 모름. 브랜치라던가 커밋이라던가 이런게 여기저기 얽히고 섥혀있는게 아직 머릿속으로 잘 시뮬레이션이 안된다능."

깃헙 안 써봐서 = 영역 영역 영역 *익혀야* 영역 핵심.

### 1. 브랜치 (branch)

- [ ] 브랜치 = 무엇 — 한 commit history 줄기. main 외 따로 영역 작업하는 갈래
- [ ] **현재 브랜치 확인** — `git branch --show-current`
- [ ] **새 브랜치 만들고 이동** — `git switch -c docs/postmortem-019-...`
- [ ] **다른 브랜치 이동** — `git switch main`
- [ ] **브랜치 목록** — `git branch` (로컬) / `git branch -r` (원격)
- [ ] **브랜치 = 머지 후 보통 삭제** — `git branch -d <name>`
- [ ] **브랜치 이름 룰 = `CONTRIBUTING.md § 브랜치 정책`** (룰 영역 *위치* — 내용 영역 cheatsheet)

> 학습 후 한 줄 요약:
>

### 2. 커밋 (commit)

- [ ] 커밋 = 무엇 — 한 변경 snapshot. git history 안 한 줄 점.
- [ ] **`git status`** — 지금 어떤 파일 변경 / staging 영역
- [ ] **`git diff`** — staging 안 한 변경
- [ ] **`git diff --staged`** — staging 영역 영역 변경
- [ ] **`git add <files>`** — staging 영역 영역 올림
- [ ] **`git commit -m "..."`** — staging 영역 영역 commit 만듦
- [ ] **`git log --oneline -10`** — 최근 10 commit 줄로
- [ ] **커밋 메시지 패턴 = `<type>(<scope>): <message>`** (Conventional Commits)
  - type: `docs` / `feat` / `chore` / `fix` / `refactor` 등
  - 예: `docs(postmortem): add 019 — chain 1~5 ...`

> 학습 후 한 줄 요약:
>

### 3. PR (Pull Request) 흐름

> 모기 voice: "117 번 PR 사장님이 삭제하셨는데 그 이유도 모름"

- [ ] PR = 무엇 — "내 브랜치 → main 영역 합쳐 주세요" 영역 사장님께 묻는 영역
- [ ] **`gh pr create --title "..." --body "..."`** — PR 만들기
- [ ] **`gh pr view <num>`** — PR 영역 상태 영역
- [ ] **`gh pr view <num> --comments`** — PR 코멘트 영역 (postmortem 019 § 3-1 영역 사용)
- [ ] **`gh pr view <num> --json closedAt,comments,reviews`** — 상세 영역 (closed 사유 진단)
- [ ] **PR 영역 3 결과**
  - **merged** 완료 — 합쳐짐 → main 영역 들어감
  - **closed** 닫힘 — 머지 X + 닫음 (이유 = 코멘트 / 외부 메시지)
  - **open** 대기 — 사장님 review 대기
- [ ] **closed 사유 모를 때** = 도리토에게 외부 메시지 (Slack / DM)

> 학습 후 한 줄 요약:
>

### 4. 외부 영역 (사장님 / 도리토 commit) 발견 시 진단

postmortem 019 § 3-2 영역 — 4744c4d 고아 commit 영역 진단 명령.

- [ ] **`git fetch origin`** — 사장님 영역 최신 영역 가져오기 (내 브랜치 영향 X)
- [ ] **`git log <branch> --oneline`** — 브랜치 영역 commit 줄로
- [ ] **`git rev-list --left-right --count origin/main...HEAD`** — main 영역 ↔ 내 영역 앞 / 뒤
- [ ] **`git show --stat <sha>`** — 특정 commit 영역 무엇 변경 (파일 + 줄 수)
- [ ] **`git log <branch1>..<branch2>`** — 두 브랜치 영역 차이 commit
- [ ] 영역 차이 모르면 → 도리토에게 외부 메시지

> 학습 후 한 줄 요약:
>

### 5. cherry-pick / rebase (천천히 영역)

당장 영역 X. 도리토가 cherry-pick / rebase 영역 언급 시 영역 한 번 영역 펴 봄.

- [ ] cherry-pick = 무엇 — 한 commit 영역 다른 브랜치 영역 *복사 이동*
- [ ] rebase = 무엇 — 내 브랜치 영역 base 영역 *옮기기*
- [ ] *주의*: rebase / push --force 영역 위험. 사장님 권한 영역.

> 학습 후 한 줄 요약:
>

---

## 미래 *공부 영역* 영역 발견 시 추가 영역

작업 중 "이건 시간 내서 공부 필요" 발견 → 본 § 에 추가.

### 6. dev ↔ main 통합 모델 (2026-05-16 M3 Phase 1 PR 중 발견)

> 모기 voice: "dev에 한거 맞지?" / "브랜치룰 지켰는지 확인좀" — PR 을 어디로 보내는지 모델이 머릿속에 없었음.

- [ ] **이 repo 흐름 = 모든 작업 → `dev` → (release 시점) `main`** — `CONTRIBUTING.md § 브랜치 정책` 이 source
- [ ] **PR base 는 항상 `dev`** (main 직접 PR 금지, release/hotfix 만 예외) — `gh pr create --base dev`
- [ ] **`main` 은 release 시점에만 갱신** — production 자동 deploy 트리거
- [ ] **`main-merge-guard` = CI 가 "main 으로 직접 머지 막기" 검사** (private repo 라 버튼은 안 막힘 → 사람이 fail 보면 머지 금지)
- [ ] 사고 사례: 2026-05-12 / 05-15 feat·docs 가 main 직접 머지 → 충돌 → postmortem 020

> 학습 후 한 줄 요약:
>

### 7. 로컬 브랜치는 자동 갱신 안 됨 (stale 개념) (2026-05-16 발견)

> 증상: 로컬 `main` 이 5주 전이라 도구가 "265 커밋 차이" 헛계산. 로컬 브랜치 ≠ 원격 최신.

- [ ] **로컬 `main` 은 내가 `git fetch`/`git pull` 안 하면 옛날 그대로** — 시간 지나면 stale
- [ ] **비교·기준은 `origin/<branch>` 로** — 예: `git log origin/dev..HEAD` (로컬 dev 아니라 origin/dev)
- [ ] **`git fetch origin` = 원격 최신 가져오되 내 작업 영향 0** (안전, 자주 해도 됨)
- [ ] **`git rev-list --left-right --count origin/dev...HEAD`** — 원격 dev 기준 내가 몇 개 앞/뒤
- [ ] 헷갈리면: "내가 로컬 main 보고 있나, origin/main 보고 있나?" 부터 self-check

> 학습 후 한 줄 요약:
>

### 8. PR 에 뭐가 들어가는지 = base 와의 diff (2026-05-16 발견)

> 증상: PR 브랜치에 파일 통째 가져오니 의도 외 변경까지 같이 들어감.

- [ ] **PR diff = `base..head`** — base(dev)에 없는 내 커밋 전부가 PR 에 보임
- [ ] **`git diff --stat origin/dev..HEAD`** — PR 올리기 전 "사장님이 볼 것" 미리 확인
- [ ] 파일 1개라도 그 파일의 모든 변경이 통째로 감 — "일부만" 은 깔끔하게 안 됨
- [ ] `.planning/` 같은 노이즈 빼고 싶으면 = 별 PR 브랜치 (gsd-pr-branch)

> 학습 후 한 줄 요약:
>

---

## 학습 로그

한 항목 익힘 = 적어두기. 미래 모기 자료.

| 날짜 | 영역 | 한 줄 요약 |
|---|---|---|
| 2026-05-14 | 본 시트 첫 작성 | postmortem 019 § 1-9 영역 git 협업 영역 공부 목록 정리. cheatsheet ↔ 본 시트 영역 영역 분리 |
| 2026-05-16 | § 6·7·8 추가 (미공부) | M3 Phase 1 PR 중 발견 — dev↔main 통합 모델 / 로컬 stale 개념 / PR base diff. 아직 익히기 전, 시간 내서 공부 대상 |
|  |  |  |

