---
name: dct-complete
description: Jira DCTC 작업 마무리 — 작업 결과 요약, Jira 완료 댓글 등록, PR 생성(사용자 확인)
argument-hint: <DCTC-번호>
---

# /dct-complete — 작업 마무리

Jira 카드 작업을 **마무리**하는 커맨드. 작업 결과를 요약해 Jira에 댓글로 남기고, 사용자 확인 후 PR을 생성한다.

## 사용법

```
/dct-complete <DCTC-번호>
```

**예시**
- `/dct-complete DCTC-1808`
- `/dct-complete DCTC-1234`

## 실행 순서

### 1. 현재 상태 수집
- `git status` — 미커밋 변경사항 확인
- `git log <base-branch>..HEAD --oneline` — 이 브랜치에서 추가된 커밋
- `git diff <base-branch>...HEAD --stat` — 변경된 파일 목록과 규모
- 부모 브랜치 자동 감지:
  ```bash
  git ls-remote --heads origin dev | grep -q dev && BASE=dev || \
    BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
  ```
  - `dev` 존재 → `dev` 사용
  - 없으면 원격 기본 브랜치 감지 → **main 이면 사용자에게 어디로 PR할지 문의** (절대 자동 main 금지)

### 2. 미커밋 변경사항 처리
- 미커밋 변경이 있으면 사용자에게 알림:
  - 커밋 후 진행할지
  - 그대로 진행할지 (PR은 커밋된 것만 포함됨)
- Jira 카드 조회 (`jira_get_issue`) — 카드 요약 및 description 에 기록된 플랜 참조

### 3. 작업 결과 요약
- 커밋 로그, 변경 파일, 규모를 바탕으로 요약 초안 작성
- 사용자에게 요약 미리보기 제시 → 승인/수정

### 4. PR 생성 여부 확인 (Jira 댓글보다 **먼저** 수행)
> 💡 PR URL 을 Jira 완료 댓글 본문에 포함해야 하므로, PR 생성을 **반드시 Step 5 앞에** 수행한다. 댓글 먼저 달고 PR 링크만 추가 댓글로 남기는 방식은 금지.

사용자에게 물어봄:
```
PR을 생성할까요? (y/N)
```

**y 인 경우**:
- 브랜치가 아직 원격에 없으면 `git push -u origin feature/DCTC-<번호>` (원격 없으면 `fork` 또는 `origin` 사용)
- **기존 PR 상태 확인 (중복/재생성 방지)** — `gh pr create` 전 반드시 수행:
  ```bash
  gh pr list --head "$(git branch --show-current)" \
    --state all --json number,state,url,mergeCommit \
    --limit 5
  ```
  판정:
  - **OPEN PR 이 이미 존재** → 신규 생성 금지. 해당 PR URL 을 재사용하고 "기존 PR 재활용" 안내. (필요하면 `gh pr edit` 으로 설명 업데이트만)
  - **MERGED PR 만 존재** + **병합 이후 신규 커밋 있음** (`git log <mergeCommit>..HEAD --oneline` 에 결과 있음) → **신규 PR 생성**. 사용자에게 "이전 PR 은 이미 병합됨. 추가 작업분으로 신규 PR 을 올립니다" 고지.
  - **MERGED PR 만 존재** + **추가 커밋 없음** → PR 생성할 내용이 없음. 사용자에게 "병합 이후 신규 커밋 없음" 보고하고 PR 생성 건너뜀 → Step 5 의 `## PR` 섹션에 기존 병합 PR URL 을 `🔗 <merged pr url> (이미 병합됨)` 으로 기재.
  - **PR 이력 없음** → 아래 신규 PR 생성 흐름 진행.
- **베이스 브랜치 결정 (team-workflow 규칙):**
  - `git ls-remote --heads origin dev` 로 `dev` 존재 여부 확인
  - **존재** → `--base dev` 사용
  - **없음** → 사용자에게 "dev 브랜치가 없습니다. 어느 브랜치로 PR을 올릴까요?" 문의
  - **절대 `main` 으로 자동 선택하지 말 것** (🚨 main 직접 PR 금지)
- `gh pr create --base <결정된 브랜치>` 실행:
  - **제목**: `[DCTC-<번호>] <작업 요약 한 줄>` (이전 병합 건의 후속이면 `[DCTC-<번호>] ... (follow-up)` 권장)
  - **본문**: Jira 카드 링크 + 변경 요약 + 테스트 플랜 체크리스트 + (follow-up 인 경우 이전 병합 PR 링크)
  - **리뷰어**: 사용자에게 확인 후 지정 (기본값 없음 — 프롬프트로 요청)
- 생성된 **PR URL 을 변수에 보관** → Step 5 댓글 본문에 삽입

**N 인 경우**:
- PR 생성 건너뜀, Step 5 댓글 본문의 `## PR` 섹션은 `생성 안 함` 으로 기재

### 5. Jira 완료 댓글 등록/수정 (PR 요약 포맷)
> 💡 **Jira-Slack 연동 채널 가독성 우선**. 채널 카드만 보고도 "무엇이 어떻게 변경됐는지" 즉시 파악 가능해야 함. PR 클릭/Jira 진입 없이 컨텍스트 전달이 목표.
> 💡 **작성 기준은 `~/.claude/rules/jira-writing.md` 참조** — 원칙(결론 먼저·왜>무엇·검색성), 마커 세트, 분량 상한이 거기에 정의됨. 아래는 이 커맨드의 운영 흐름만 다룬다.
> 💡 **일관된 마커 유지** — `📝 변경 요약`, `후속:` 순서·이름 고정. 매번 다른 헤더 쓰지 말 것.
> 💡 **이미 완료 댓글이 있으면 신규 작성 금지 → 기존 댓글을 수정해 이어붙인다.**

#### 5-1. 기존 완료 댓글 탐지
- `mcp__mcp-atlassian__jira_get_issue(issue_key=DCTC-<번호>, fields="comment")` 로 댓글 조회
- 본문 첫 줄이 **`✅ DCTC-` 로 시작** 하거나 **`✅ 작업 완료 요약` (구버전 마커)** 인 **가장 최신** 댓글을 "기존 완료 댓글" 로 식별 (작성자 필터 없음)
- 발견된 댓글의 `id` 와 원문 보관

#### 5-2. PR 요약 추출 (댓글 작성 전 필수)
댓글 본문에 넣을 **변경 요약 bullet** 을 만들기 위해 아래 소스를 종합:
- `git log <base>..HEAD --oneline` — 커밋 메시지 (사용자 의도)
- `git diff <base>...HEAD --stat` — 파일별 변경 규모 (영향 범위)
- PR 본문 (있으면) — Step 4 에서 작성한 변경 요약
- Jira description 의 `📋 작업 플랜` — 사용자 승인된 목표

추출 규칙:
- **변경 요약**: 3~5개 bullet. 각 줄 `<무엇> — <왜/효과>` 형식. 영향 범위·파일 목록은 PR 본문에서 확인(댓글 중복 금지)
- 카드 제목 그대로 복붙 금지 — 실제 변경 내용 기반으로 재작성

#### 5-3. 분기 처리

**A. 기존 완료 댓글이 없는 경우 → 신규 작성**
- `mcp__mcp-atlassian__jira_add_comment` 로 단일 댓글 등록
- 포맷 (마커·순서 고정):
  ```
  ✅ DCTC-<번호> 완료 — <결과 한 줄, 고유명사 포함>
  🔗 PR: <pr url>   (또는: 생성 안 함 / 이미 병합됨)

  📝 변경 요약
  • <무엇> — <왜/효과>
  (3~5개 bullet)

  💡 결정: <대안 대비 왜, 한 줄>   ← 판단 갈렸을 때만, 평소 생략
  후속: <있을 때만 한 줄, 없으면 줄 자체 생략>
  ```
- "한 줄 요약" 은 카드 제목이 아니라 **이번 작업 핵심 30자 내외** (예: `사이드바 애니메이션 개선 + 접근성 보강`)
- 검증 결과·변경 파일 전체 목록·코드 스니펫·표는 **댓글에 넣지 말 것** — PR 본문에서 확인

**B. 기존 완료 댓글이 있는 경우 → 수정(append)**
- `mcp__mcp-atlassian__jira_edit_comment(issue_key, comment_id, comment=<새 본문>)` 로 교체
- 새 본문 = 기존 원문 + 구분선 + 추가 블록 (동일 마커 재사용):
  ```
  <기존 원문 그대로>

  ---
  🔁 추가 작업 (YYYY-MM-DD)
  ✅ <한 줄 요약>
  🔗 PR: <pr url>   (또는: 생성 안 함 / 이미 병합됨)

  📝 변경 요약
  • <변경 1>
  • <변경 2>
  (3~5개 bullet)

  후속: <있을 때만 한 줄>
  ```
- 날짜는 **Asia/Seoul (KST)** 기준 `YYYY-MM-DD`
- 헤더(`##`)·코드 블록·표 사용 금지 — 마커는 `✅`/`🔗`/`📝`/`💡`(선택)/`후속`(선택) 만

#### 5-4. 공통 규칙
- 표·코드 블록은 댓글에 포함하지 말 것. 꼭 필요하면 PR 본문/Jira description 에 둠
- ADF 변환 불필요 (표 없으므로 평문 markdown 으로 충분). 표가 들어가야 하는 예외는 `jira-adf-table` + curl fallback
- **PR URL 만 따로 추가 댓글로 남기지 말 것** — 위 포맷의 `🔗 PR:` 라인에 통합
- **새 완료 댓글을 여러 개 만들지 말 것** — 재작업 시 기존 댓글 수정
- 마커 이름·순서를 **임의로 바꾸지 말 것** (Slack 채널 패턴 인식 일관성)

### 6. 종료 메시지
```
✅ DCTC-<번호> 마무리 완료
- PR: <pr url> (또는 "생성 안 함")
- Jira 댓글: <comment link>   ← PR URL 포함된 단일 완료 댓글
```

## 규칙 준수
- **한국어** 응답/Jira 댓글
- **Jira 문서 가시성**: 완료 요약 댓글을 작성하기 전에 **제3자가 읽었을 때 즉시 파악할 수 있는 구조인지** 먼저 고민한다. 표, 번호 리스트, 헤더 계층, 코드 블록 등으로 **스캔 가능하게** 구성. 장문의 산문체 금지.
- **커밋 메시지**: 영어 prefix(`feat:`, `fix:`, 등) + 한국어 본문 (korean-dev 규칙)
- **PR 제목**: `[DCTC-N] ...` 포맷으로 Jira 카드와 연결
- **PR 베이스**: `dev` 기본, 없으면 사용자 문의. **`main` 직접 PR 금지** (team-workflow 규칙)
- **doc-updates 규칙**: 코드 변경과 관련된 문서(`.claude/agents/*`, 프로젝트 CLAUDE.md 등) 업데이트 여부 체크
- **보안**: `~/.claude/settings.json` 전체를 `Read` 로 출력 금지 (시크릿 세션 로그 유출 방지). 검증은 `jq` 또는 `grep` 으로만
