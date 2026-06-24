---
name: dct-job
description: Jira DCTC 카드 기반 작업을 완전 자동화 — 플랜→구현→검증→PR까지 논스톱 실행
argument-hint: <DCTC-번호> <feat|fix|refactor|chore|docs|test> "<작업 설명 또는 파일경로>"
---

# /dct-job — 완전 자동화 파이프라인

Jira 카드 작업을 **처음부터 끝까지 자동으로** 수행하는 커맨드. 플랜과 마무리를 수동으로 나누고 싶다면 `/dct-plan` + `/dct-complete` 조합을 사용하라.

## 사용법

```
/dct-job <DCTC-번호> <작업종류> <작업명령|파일경로>
```

**예시**
- `/dct-job DCTC-1808 feat "프론트 사이드바 애니메이션 개선"`
- `/dct-job DCTC-1234 fix ./docs/job.md`
- `/dct-job DCTC-1500 refactor "auth 미들웨어 정리"`

## 인자

| 인자 | 설명 | 예시 |
|------|------|------|
| 스토리번호 | Jira 카드 번호 | `DCTC-1808` |
| 작업종류 | 커밋/브랜치 타입 | `feat`, `fix`, `refactor`, `chore`, `docs`, `test` |
| 작업명령 | 작업 설명 문자열 또는 명세 파일 경로 | `"사이드바 개선"` 또는 `./docs/job.md` |

## 실행 파이프라인

### Phase A — 플랜 & 진입 (🧠 **planmode** 사용)
이 단계는 **반드시 Claude Code 의 내장 plan mode 를 활성화한 상태**에서 진행한다. plan mode 동안은 파일 수정/커맨드 실행이 금지되고 읽기·분석만 허용되므로, 카드 조회 → 범위 분석 → 단계 설계 → 영향 파일 예측을 **증거 기반으로 깊이 있게** 수행한 뒤 `ExitPlanMode` 로 사용자 승인을 받는다.

> **🚨 첫 액션 필수**: Phase A 의 **맨 처음 단계는 반드시 `EnterPlanMode` 툴 호출**이다. Claude Code frontmatter 에는 plan mode 자동진입 필드가 없으므로 Claude 가 명시적으로 호출해야 한다. 이전에 어떤 mutation 도 발생하지 않아야 함.

1. **`EnterPlanMode` 툴 호출 (mutation 전 필수)**
2. `mcp__mcp-atlassian__jira_get_issue` — 카드 조회 (읽기만)
3. 범위 분석 → `writing-plans` 스킬로 단계 설계
4. 병렬 가능성 분석 (Phase B 에서 병렬 Task 로 분할할 작업 후보 식별)
5. 예상 영향 파일 목록화 (Grep/Glob 으로 사전 조사)
6. `ExitPlanMode` 로 플랜 미리보기 제시 → **사용자 명시적 승인**
7. 승인된 플랜을 Jira 카드 **description 필드**에 기록 (`jira_update_issue` → `fields.description`)
   - **기존 description 탐지**: `jira_get_issue` 결과의 description 에 `📋 작업 플랜` 마커가 있으면 → 기존 원문 유지 + 구분선 + `## 🔁 추가 플랜 (YYYY-MM-DD, KST)` 로 **이어붙이기**. 없으면 신규 작성
   - 기존 플랜을 **덮어쓰지 말 것** — 카드 재작업 이력 보존
   - 표가 포함되면 `jira-adf-table` 스킬로 ADF 변환 후 업로드
8. `dev` 에서 `feature/DCTC-<번호>` 체크아웃 (있으면 switch)

### Phase B — 구현 (🚀 **네이티브 병렬 Task**)
Phase A 에서 식별한 병렬 가능 작업을 **여러 `Task` 에이전트로 동시 실행**한다. 순차 의존성이 있는 작업만 차례로 진행.

1. `executing-plans` 스킬로 단계 목록 확정
2. **독립 작업은 병렬 실행** — `Task(subagent_type="general-purpose", model=<tier>, prompt=...)` 를 한 메시지에 복수 호출
   - 작업 티어 매핑:
     - 단순 수정 (타입 export, 오타, 한 줄 변경) → `haiku`
     - 표준 구현 (엔드포인트, 컴포넌트, 리팩터) → `sonnet`
     - 복잡한 설계/분석/대규모 리팩터 → `opus`
   - 빌드/테스트/설치 등 30초+ 작업은 `run_in_background=true`
3. **의존성 있는 작업**은 선행 결과 확정 후 순차 실행
4. 각 병렬 배치 완료 시 체크포인트 — 빌드/타입체크 논블로킹 검증
5. 모든 배치 완료 후 Phase C 로 이행

### Phase C — 코드 리뷰 (🔍 **Claude Code 내장 `/review`** 사용)
구현이 끝나면 **Claude Code 내장 `/review` 커맨드**를 호출해 브랜치 변경사항에 대한 품질·보안·로직 리뷰를 받는다. DCT 플러그인의 `/dct-sc-review` 가 아니라 **Claude Code 네이티브 built-in `/review`** 를 쓸 것 (AI 네이티브 표준 리뷰어).

1. **`/review` 실행** — 현재 브랜치의 커밋 diff 를 자동 인식해 리뷰 수행
2. 리뷰 결과 중 **High/Critical 지적사항**은 이 Phase 안에서 처리:
   - 수정 가능한 항목 → 같은 Phase 에서 fix 커밋
   - 설계 판단 필요 항목 → 사용자에게 보고 후 의사결정
3. Low/Info 레벨 지적은 Jira 댓글/PR 설명에 TODO 로 보존 가능
4. 리뷰-수정 루프는 **최대 2회**. 이후는 사용자 개입 요청

> 💡 `/review` 는 Phase D(검증) 이전에 수행. 정적 리뷰가 끝난 코드에 대해서만 테스트/빌드 검증을 돌려 피드백 루프를 짧게 유지.

### Phase D — 검증
1. `verification-before-completion` 스킬 수행
2. 테스트/빌드/린트 실행
3. 실패 시 수정 루프 (최대 3회, 이후 사용자 개입 요청)

### Phase E — 마무리 (내부적으로 /dct-complete 로직 수행)
> 💡 순서 주의 — **PR 생성 → Jira 완료 댓글 (PR URL 포함)**. 완료 댓글 먼저 올리고 PR 링크만 추가 댓글로 남기는 방식 금지.

1. 작업 결과 요약 생성 (리뷰 결과 요약 포함)
2. 사용자에게 PR 생성 여부 확인 (y/N)
3. y 인 경우:
   - **기존 PR 상태 확인 (중복/재생성 방지)**:
     ```bash
     gh pr list --head "$(git branch --show-current)" \
       --state all --json number,state,url,mergeCommit --limit 5
     ```
     - **OPEN PR 존재** → 신규 생성 금지, 기존 PR URL 재사용
     - **MERGED PR + 추가 커밋 있음** (`git log <mergeCommit>..HEAD` 결과 존재) → 사용자에게 "이전 PR 병합됨, 추가 작업분으로 신규 PR 생성" 고지 후 **신규 PR** 생성 (제목에 `(follow-up)` 표기, 본문에 이전 병합 PR 링크)
     - **MERGED PR + 추가 커밋 없음** → PR 생성 건너뜀, Step 4 의 `## PR` 섹션에 기존 병합 PR URL 을 `🔗 <url> (이미 병합됨)` 으로 기재
     - **이력 없음** → 아래 신규 생성 흐름 진행
   - 베이스 브랜치 결정 후 `gh pr create --base <base>` 실행:
     - `dev` 존재 확인 → 있으면 `dev` 자동 선택
     - 없으면 사용자에게 어느 브랜치로 PR할지 문의 (🚨 **`main` 으로 자동 선택 금지**)
   - 생성(또는 재활용)된 **PR URL 을 변수로 보관** → Step 4 댓글 본문에 삽입
4. Jira 카드 완료 댓글 처리 — **PR 요약 포맷 + 기존 댓글 탐지 후 분기** (Slack 채널 가독성 + 일관성):
   - `jira_get_issue(fields="comment")` 로 댓글 목록 조회
   - 본문 첫 줄이 `✅ DCTC-` 또는 `✅ 작업 완료 요약` (구버전) 인 **가장 최신** 댓글 식별
   - **PR 요약 추출** (댓글 작성 전 필수): `git log <base>..HEAD --oneline` + `git diff <base>...HEAD --stat` + PR 본문 + Jira description 플랜을 종합해 변경 요약 bullet 3~5개 생성. 카드 제목 복붙 금지
   - **신규 작성 (없음)** → `jira_add_comment` 로 다음 포맷 (마커·순서 고정):
     ```
     ✅ DCTC-<번호> 완료 — <한 줄 요약>
     🔗 PR: <url>   (또는: 생성 안 함 / 이미 병합됨)

     📝 변경 요약
     • <변경 1 — 무엇/왜>
     • <변경 2>
     • <변경 3>

     📂 영향 범위   ← 파일 그룹 2개 이상일 때만 포함
     • <영역 1>: <한 줄>
     • <영역 2>: <한 줄>

     후속: <있을 때만 한 줄, 없으면 생략>
     ```
   - **이어붙이기 (있음)** → `jira_edit_comment` 로 기존 원문 + `---` + 추가 블록 (동일 마커):
     ```
     🔁 추가 작업 (YYYY-MM-DD)
     ✅ <한 줄 요약>
     🔗 PR: <url>

     📝 변경 요약
     • <변경 1>
     • <변경 2>

     후속: <있을 때만>
     ```
   - 검증 결과 / 변경 파일 전체 목록 / 표 / 코드 블록은 **댓글에 적지 말 것** — PR 본문·description 에서 확인. PR 링크만 따로 추가 댓글로 남기지 말 것
   - 마커 이름·순서 임의 변경 금지 (Slack 채널 패턴 인식 일관성)

## 중단 지점
다음 시점에서 사용자 확인을 받음:
- 플랜 승인 (Phase A.6 — `ExitPlanMode` 시점)
- `/review` High/Critical 지적 중 설계 판단 필요 항목 (Phase C)
- 구현/검증 실패 3회 초과 (Phase D)
- PR 생성 여부 (Phase E.2)

## 참조 규칙
- `~/.claude/rules/team-workflow.md` — 브랜치 네이밍, Jira/Confluence 작업 경계
- `~/.claude/rules/korean-dev.md` — 응답/커밋 한국어, 코드는 영어
- `~/.claude/rules/doc-updates.md` — 작업 완료 직전 관련 문서 동기화
- `~/.claude/rules/agents.md` — 멀티파일은 general-purpose 서브에이전트 위임

## 보안
- `~/.claude/settings.json` 전체를 `Read` 로 출력 금지 (시크릿 세션 로그 유출 방지). 검증은 `jq` 또는 `grep` 으로만

## 수동 분리 버전 (권장 파이프라인)
완전 자동화가 아니라 **단계별로 다른 도구를 섞어 쓰고 싶다면** 아래 4단계를 순서대로:

```
/dct-plan  →  병렬 Task 구현  →  /review  →  /dct-complete
   플랜         구현 (병렬)        리뷰        마무리 & PR
```

1. **`/dct-plan DCTC-1808 "설명"`** — 플랜 작성 + Jira 업로드 + `feature/DCTC-1808` 진입 (plan mode + effort:max)
2. **구현** — 상황에 맞는 방식 선택:
   - **네이티브 병렬 Task** — 독립 작업을 `Task(subagent_type="general-purpose")` 복수 호출로 한 번에 (빠름)
   - 직접 코딩, `/dct-sc-implement` 등도 가능
3. **`/review`** — Claude Code **내장** 리뷰 커맨드 (DCT 플러그인 `/dct-sc-review` 아님). 품질/보안/로직 지적 후 수정
4. **`/dct-complete DCTC-1808`** — 결과 요약 → Jira 완료 댓글 → PR 생성 (사용자 확인 후)
