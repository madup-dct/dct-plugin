---
name: dct-plan
description: Jira DCTC 카드 기반 작업 진입 — 플랜 작성, Jira 업로드, feature 브랜치 체크아웃까지 수행 (구현은 유저 자유)
argument-hint: <DCTC-번호> ["<작업 설명>" | <파일경로>]
effort: max
---

# /dct-plan — 플랜 작성 & 진입

Jira 카드 기반 작업을 **시작**하는 커맨드. 플랜 작성과 브랜치 진입까지만 수행하고, 이후 구현은 사용자가 자유롭게 진행한다(다른 플러그인/도구 조합 가능).

> **🧠 Reasoning Effort: MAX + ultrathink** — frontmatter `effort: max` (Opus 4.6 전용) + 본문 `ultrathink` 키워드로 최대 추론 예산을 사용한다. 카드 범위 판단, 단계 분해, 영향 파일 예측, 검증 전략 설계 전 과정에서 깊이 있게 사고한 뒤 산출물을 작성한다.

## 🚨 필수 선행 액션 — Plan Mode 진입

이 커맨드의 **맨 첫 번째 액션은 반드시 `EnterPlanMode` 툴 호출**이다. Claude Code 공식 frontmatter 에는 plan mode 자동진입 필드가 없으므로, 본문 지시에 따라 Claude 가 명시적으로 호출해야 한다.

- **plan mode 진입 이유**: 승인되지 않은 상태로 Jira description 변경, 브랜치 체크아웃 등 mutation 을 일으키면 안 됨
- **plan mode 이탈**: 플랜이 사용자 승인을 받은 직후 `ExitPlanMode` 로 벗어나 Jira 업로드 + 브랜치 작업 수행
- **plan mode 중 허용**: 카드 조회, 파일 Read, Grep/Glob 탐색 등 **읽기 전용 작업만**

## 사용법

```
/dct-plan <DCTC-번호> [설명|파일경로]
```

**예시**
- `/dct-plan DCTC-1808 "프론트 사이드바 애니메이션 개선"`
- `/dct-plan DCTC-1234 ./docs/job.md`
- `/dct-plan DCTC-1500` (설명 생략 시 Jira 카드 내용만 사용)

## 실행 순서

`dct-jira-workflow` 스킬의 Phase 1~3 만 수행:

### 1. 카드 조회
- `mcp__mcp-atlassian__jira_get_issue` 로 카드 요약/설명/현재 상태 조회
- 작업명령이 파일 경로면 `Read` 로 내용 추가 로드

### 2. 범위 분석
- 카드 내용 + 작업명령을 읽고 범위 판단
- 범위가 넓으면 `writing-plans` 스킬로 세부 단계 작성
- 병렬 가능한 독립 작업이 3개 이상이면 사용자에게 제안

### 3. 플랜 작성
- 목표, 단계(step-by-step), 예상 영향 파일, 검증 방법 포함
- 사용자에게 **플랜 미리보기** 제시 → 승인 받기

### 4. Jira 업로드 (기존 플랜 있으면 이어붙이기)
> 💡 **이미 `📋 작업 플랜` 섹션이 description 에 있으면 신규 작성/덮어쓰기 금지 → 기존 description 을 유지한 채 이어붙인다.** 카드가 여러 번 재작업될 때 플랜 이력이 사라지지 않도록 한다.

#### 4-1. 기존 description 탐지
- `jira_get_issue(issue_key=DCTC-<번호>, fields="description")` 로 현재 description 원문 조회 (Step 1 결과 재사용 가능)
- 본문에 `📋 작업 플랜` 마커가 포함되어 있는지 확인

#### 4-2. 분기 처리
**기존 플랜이 있는 경우 → 이어붙이기(append)**
- 새 description = 기존 원문 + 구분선 + 이번 이어붙임 섹션:
  ```
  <기존 원문 그대로>

  ---

  ## 🔁 추가 플랜 (YYYY-MM-DD)

  ## 목표
  ...

  ## 단계
  1. ...

  ## 예상 영향 파일
  - ...

  ## 검증
  - ...
  ```
- 날짜는 **Asia/Seoul (KST)** 기준 `YYYY-MM-DD`

**기존 플랜이 없는 경우 → 신규 작성**
- description 포맷 (Markdown):
  ```
  📋 작업 플랜 (by Claude Code)

  ## 목표
  ...

  ## 단계
  1. ...
  2. ...

  ## 예상 영향 파일
  - ...

  ## 검증
  - ...
  ```

#### 4-3. 업로드
- **표가 포함되면** `jira-adf-table` 스킬로 ADF 변환 (컬럼 너비 내용 비례 자동 계산). 표 없으면 markdown 그대로
- 도구: `mcp__mcp-atlassian__jira_update_issue` (`fields.description`) — ADF dict 거부 시 curl Jira REST v3 fallback

### 5. 브랜치 진입
- 부모 브랜치 결정 (`dev` 우선, 없으면 기본 브랜치)
- `git fetch origin` 으로 최신화
- `feature/DCTC-<번호>` 브랜치:
  - **존재하면** → `git switch feature/DCTC-<번호>` (그대로 체크아웃)
  - **없으면** → `git switch -c feature/DCTC-<번호> dev`
- 메모리 엔트리 이름에 스토리 번호 포함 (예: `DCTC-1808-plan`)

### 6. 종료 메시지
- 플랜이 반영된 Jira 카드 링크 (description 갱신됨)
- 현재 브랜치 확인
- 다음 액션 안내 — **권장 파이프라인**:
  ```
  ✅ 플랜 업로드 완료 & feature/DCTC-1808 진입

  권장 파이프라인 (이 커맨드 이후):
  ┌──────────────┐   ┌─────────────────────┐   ┌────────┐   ┌────────────────┐
  │ /dct-plan ✓  │ → │ 병렬 Task 구현      │ → │/review │ → │ /dct-complete  │
  │   (완료)     │   │                     │   │        │   │                │
  └──────────────┘   └─────────────────────┘   └────────┘   └────────────────┘
                         구현 (병렬)            코드 리뷰     요약+Jira+PR

  2단계 — 구현:
    • 네이티브 병렬 Task : 독립 작업을 general-purpose 에이전트 복수 호출 (빠름)
    • 또는 직접 코딩 / /dct-sc-implement 등

  3단계 — 리뷰:
    • /review  : Claude Code 내장 리뷰 커맨드 (DCT /dct-sc-review 아님)
                 품질·보안·로직 지적 후 수정

  4단계 — 마무리:
    • /dct-complete DCTC-1808  : Jira 댓글 + PR 생성

  전 과정을 자동화하려면 /dct-job 을 대신 사용하세요.
  ```

## 규칙 준수
- **브랜치 네이밍**: `feature/DCTC-<번호>` 고정 (team-workflow 규칙)
- **Jira 문서 가시성**: 플랜을 description 에 작성하기 전에 **제3자가 읽었을 때 바로 이해할 수 있는 구조인지** 먼저 고민한다. 표, 번호 리스트, 헤더 계층, 코드 블록 등을 활용해 **스캔 가능한 문서**를 작성할 것. 장문의 산문체 금지.
- **구현은 수행하지 않음**: 이 커맨드는 플랜과 진입까지만
- **보안**: `~/.claude/settings.json` 을 `Read` 로 전체 출력하지 말 것 (시크릿 세션 로그 유출 방지). 검증은 `jq` 키 확인만
