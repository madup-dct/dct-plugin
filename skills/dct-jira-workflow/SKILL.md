---
name: dct-jira-workflow
description: Jira DCTC 카드 기반 작업 실행 스킬. 카드 조회→브랜치 생성→플랜을 description에 기록→구현→완료 댓글 정리까지의 전체 워크플로우를 관리한다.
---

# DCT Jira 워크플로우 스킬

`/dct-job` 커맨드의 백엔드. Jira 카드 단위로 작업을 시작·완료하고 근거를 Jira에 남긴다.

## 핵심 도구 매핑

| 단계 | MCP 도구 |
|------|---------|
| 카드 조회 | `mcp__mcp-atlassian__jira_get_issue` |
| 전이 조회 | `mcp__mcp-atlassian__jira_get_transitions` |
| 상태 변경 | `mcp__mcp-atlassian__jira_transition_issue` |
| 댓글 추가 | `mcp__mcp-atlassian__jira_add_comment` |
| 카드 업데이트 | `mcp__mcp-atlassian__jira_update_issue` |

## 표준 실행 흐름

### Phase 1 — 진입
1. 인자 파싱: `<스토리번호> <타입> <명령|경로>`
2. 카드 조회 → 요약/설명/현재 상태 파악
3. 작업명령이 파일 경로면 `Read` 로 내용 로드

### Phase 2 — 브랜치
1. 부모 브랜치 결정 (`dev` 우선, 없으면 기본 브랜치)
2. `git fetch origin` 후 최신 상태 확인
3. `feature/<스토리번호>` 브랜치 체크아웃 또는 생성
4. 이미 변경사항이 있는지 `git status` 로 확인

### Phase 3 — 플랜
- 작업 범위가 넓으면 `writing-plans` 스킬 호출
- 병렬 가능한 독립 작업이 있으면 사용자에게 제안
- **플랜에 표가 포함되면 `jira-adf-table` 스킬로 ADF 변환 후 업로드** (컬럼 너비가 내용 비례로 자동 배분됨). 표가 없으면 markdown 문자열 그대로 전달해도 됨
- 플랜 확정 시 Jira 카드의 **description 필드**에 기록 (`jira_update_issue` 의 `fields.description`)
- **재작업 시 기존 플랜 보존**:
  - `jira_get_issue(fields="description")` 로 현재 description 조회
  - `📋 작업 플랜` 마커가 **있으면** → 기존 원문 + `---` + `## 🔁 추가 플랜 (YYYY-MM-DD, KST)` 섹션으로 **이어붙이기**. 덮어쓰기 금지
  - **없으면** → 아래 포맷으로 신규 작성
  ```
  📋 작업 플랜 (by Claude Code)

  ## 목표
  ...

  ## 단계
  1. ...
  2. ...

  ## 예상 영향 파일
  - ...
  ```

### Phase 4 — 구현
- `executing-plans` 스킬로 단계별 실행
- 멀티파일 변경은 `general-purpose` 서브에이전트 위임
- 메모리 엔트리 이름에 스토리 번호 포함

### Phase 5 — 검증
- `verification-before-completion` 스킬 필수
- 테스트/빌드/린트 실행
- 실패 시 수정 반복

### Phase 6 — 마무리
> 💡 순서 — **PR 생성 먼저, Jira 완료 댓글은 그다음**. 완료 댓글 본문에 PR URL 을 포함해 **단일 댓글**로 남긴다. PR 링크만 따로 추가 댓글로 남기는 방식 금지.

1. 사용자에게 PR 생성 여부 확인. y 인 경우:
   - **기존 PR 상태 확인 (중복/재생성 방지)**: `gh pr list --head <branch> --state all --json number,state,url,mergeCommit`
     - OPEN PR 존재 → 신규 생성 금지, 기존 URL 재사용
     - MERGED PR + 병합 이후 신규 커밋 있음 → **신규 PR** 생성 (제목에 `(follow-up)`, 본문에 이전 병합 PR 링크)
     - MERGED PR + 추가 커밋 없음 → 생성 건너뜀, `## PR` 섹션에 `🔗 <url> (이미 병합됨)` 기재
     - 이력 없음 → 일반 신규 생성
   - `gh pr create --base <dev|사용자 지정>` 실행, PR URL 확보
2. **PR 요약 포맷 + 재작업 시 이어붙이기** (Slack 채널 가독성 + 일관성):
   - `jira_get_issue(fields="comment")` 로 댓글 목록 조회
   - 본문 첫 줄이 `✅ DCTC-` 또는 `✅ 작업 완료 요약` (구버전) 인 **가장 최신** 댓글 식별
   - **PR 요약 추출 (댓글 작성 전 필수)**: `git log <base>..HEAD --oneline` + `git diff <base>...HEAD --stat` + PR 본문 + description 플랜을 종합해 **변경 요약 bullet 3~5개** 생성. 카드 제목 복붙 금지
   - **있음** → `jira_edit_comment` 로 수정 (기존 원문 + `---` + 추가 블록). **신규 댓글 생성 금지**
   - **없음** → `jira_add_comment` 로 단일 댓글 등록
   - 검증 결과 / 변경 파일 전체 목록 / 표 / 코드 블록은 댓글에 넣지 말 것 (PR 본문/description 에서 확인)
3. 완료 댓글 포맷 — **마커·순서 고정** (작성 기준 상세는 `~/.claude/rules/jira-writing.md` 참조):
  - 신규:
    ```
    ✅ DCTC-<번호> 완료 — <결과 한 줄, 고유명사 포함>
    🔗 PR: <url>   (또는: 생성 안 함 / 이미 병합됨)

    📝 변경 요약
    • <무엇> — <왜/효과>
    (3~5개)

    💡 결정: <대안 대비 왜, 한 줄>   ← 판단 갈렸을 때만, 평소 생략
    후속: <있을 때만 한 줄, 없으면 생략>
    ```
  - 이어붙이기 (기존 원문 아래 추가):
    ```
    ---
    🔁 추가 작업 (YYYY-MM-DD, KST)
    ✅ <결과 한 줄, 고유명사 포함>
    🔗 PR: <url>

    📝 변경 요약
    • <무엇> — <왜/효과>
    (3~5개)

    💡 결정: <대안 대비 왜, 한 줄>   ← 판단 갈렸을 때만, 평소 생략
    후속: <있을 때만>
    ```
  - 마커(`✅`/`🔗`/`📝`/`💡`(선택)/`후속`(선택)) 외 헤더(`##`)·코드 블록·표 사용 금지

## 규칙 준수
- **브랜치 네이밍**: `feature/DCTC-<번호>` 고정
- **커밋 메시지**: 영어 prefix(`feat:`, `fix:`, 등) + 한국어 본문 (예: `fix: 로그인 버튼 클릭 이슈 해결`)
- **PR 베이스**: `dev` 기본. 없으면 사용자에게 문의. **`main` 직접 PR 금지**
- **Confluence**: Data Consulting Team 스페이스의 DCT CP / Matrix 문서만 수정 허용
- **삭제 금지**: Jira 이슈, Confluence 페이지 삭제 절대 금지
