---
tags:
  - github-actions
  - pull-request
  - labeler
  - migration
  - supabase
  - collaboration
---

# PR 변경 파일 기준 자동 라벨링

> [!summary]
> PR 라벨러는 Pull Request에 포함된 변경 파일 경로를 보고 자동으로 라벨을 붙이는 도구다.
> 예를 들어 SQL 마이그레이션 파일이 추가되면 `needs-migration-applied` 같은 라벨을 붙여서 리뷰어가 놓치지 않게 만들 수 있다.

## 본 설정

```yaml
needs-migration-applied:
  - changed-files:
      - any-glob-to-any-file: "supabase/migrations/**.sql"
```

이 설정은 대략 이렇게 읽으면 된다.

```text
PR 안에 supabase/migrations/ 아래의 .sql 파일이 하나라도 있으면
needs-migration-applied 라벨을 붙여라.
```

## 각 줄의 의미

```yaml
needs-migration-applied:
```

붙일 라벨 이름이다.

이 라벨은 보통 "이 PR은 DB 마이그레이션 적용 여부를 확인해야 한다"는 신호로 쓴다.

```yaml
changed-files:
```

PR에 포함된 변경 파일 목록을 조건으로 보겠다는 뜻이다.

```yaml
any-glob-to-any-file: "supabase/migrations/**.sql"
```

변경된 파일 중 하나라도 이 glob 패턴에 걸리면 조건을 만족한다.

```text
supabase/migrations/**.sql
```

는 `supabase/migrations` 아래의 SQL 파일을 의미한다.

## 왜 쓰는가

DB 마이그레이션은 코드 변경보다 놓치기 쉽다.

애플리케이션 코드는 PR 리뷰에서 눈에 잘 보이지만, 마이그레이션 파일은 "배포 전에 적용해야 하는 작업"을 동반한다.

예를 들어 이런 상황이 생길 수 있다.

```text
1. PR에 users 테이블 컬럼 추가 마이그레이션이 들어감
2. 애플리케이션 코드는 새 컬럼을 읽기 시작함
3. 배포 환경 DB에 마이그레이션이 적용되지 않음
4. 런타임에서 column does not exist 같은 에러 발생
```

그래서 라벨을 자동으로 붙여두면 리뷰나 배포 체크리스트에서 바로 눈에 띈다.

> [!important]
> 이 라벨은 마이그레이션을 자동으로 실행하는 기능이 아니다.
> "이 PR에는 마이그레이션 파일이 있으니 확인해라"는 표시를 자동으로 붙이는 장치다.

## 백엔드 관점에서 보면

이런 설정은 코드 품질보다 **협업 프로세스 안전장치**에 가깝다.

테스트가 로직 버그를 잡는다면, PR 라벨러는 리뷰 과정에서 놓치기 쉬운 운영 작업을 드러낸다.

```text
테스트:
코드가 의도대로 동작하는지 확인

PR 라벨러:
이 PR에 어떤 종류의 주의사항이 있는지 표시
```

마이그레이션 외에도 비슷하게 쓸 수 있다.

```yaml
needs-env-check:
  - changed-files:
      - any-glob-to-any-file: ".env.example"

frontend:
  - changed-files:
      - any-glob-to-any-file: "src/**/*.tsx"

backend:
  - changed-files:
      - any-glob-to-any-file: "server/**"
```

## 한 줄 정리

> [!abstract]
> `changed-files` 기반 PR 라벨러는 "이 PR에 어떤 종류의 변경이 들어있는지"를 파일 경로로 감지해서, 사람이 리뷰 전에 알아야 할 신호를 자동으로 붙여주는 협업 장치다.

---

## 복습 질문

- [ ] `needs-migration-applied`는 무엇을 실행하는 기능이 아니라 무엇을 붙이는 기능인가?
- [ ] `changed-files` 조건은 PR의 어떤 정보를 기준으로 판단하는가?
- [ ] `any-glob-to-any-file: "supabase/migrations/**.sql"`은 어떤 경우에 true가 되는가?
- [ ] SQL 마이그레이션 파일이 있는 PR에 자동 라벨을 붙이면 어떤 운영 실수를 줄일 수 있는가?
- [ ] 테스트와 PR 라벨러는 각각 어떤 종류의 문제를 줄이는가?

## 한 줄 회고

- 헷갈렸던 점: SQL 마이그레이션 파일을 자동 적용하는 워크플로우가 아니라, 마이그레이션 확인이 필요하다는 PR 라벨을 자동으로 붙이는 설정이다.
