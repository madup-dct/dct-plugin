# Changelog

## 0.9.0 (2026-05-18)

### Added
- **`/dct-gcs` 커맨드 + `dct-gcs` 스킬** — DCT 멀티프로젝트 GCP Cloud Storage 운영 전용. 반복 함정(프로젝트 혼선, cross-region egress 과금, 재귀 `ls` 폭주, ADC 누락)을 가드레일로 차단. `dct-slack` 와 동일한 2-파일(커맨드+스킬) 구조. MCP 없이 `gcloud` CLI 만 사용
  - 서브커맨드: `check`(검증) / `use`(named config·프로젝트 전환) / `ls`(자동 `--limit`) / `cp`(cross-region 경고) / `sync`(dry-run 선행)
  - 인증 분기: CLI=`gcloud auth login`, SDK=ADC. `DefaultCredentialsError` → ADC 누락 진단
  - 보안: credentials/access token 비노출 (`print-access-token` → `/dev/null`, exit code 만), 서비스 계정 JSON 키 지양
  - 설치·인증 셋업은 스킬에 중복하지 않고 온보딩 E단계로 위임

### Changed
- **`dct-onboarding` 스킬에 GCP CLI 추가** — `commands/dct.md`(이미 7단계 GCP 포함)와 동기화. 진단 테이블에 GCP 행, 체크리스트에 `E. GCP CLI` 신설(설치·`auth login`·ADC·프로젝트 지정·`gcloud storage` 검증). 중복되던 `### E.` 레터링 버그 수정 → AWS·Slack·RTK 재정렬해 A~H 일관화. 최종 검증·실패 대응에 GCP 항목 추가

### Files
- `commands/dct-gcs.md` — 신규
- `skills/dct-gcs/SKILL.md` — 신규
- `skills/dct-onboarding/SKILL.md` — GCP 단계 추가 + 레터링 수정
- `README.md` — 커맨드 표·파일 트리·skills 라인 동기화
- `.claude-plugin/plugin.json` — 버전 0.8.0 → 0.9.0

## 0.8.0 (2026-05-15)

### Changed
- **완료 댓글에 PR 요약 추가** — 0.7.0 컴팩트 포맷이 Jira-Slack 연동 채널에서 "무엇이 변경됐는지" 즉시 파악 안 된다는 피드백 반영. 한 줄 요약 + PR URL 유지하면서 `📝 변경 요약` bullet 3~5개와 (선택) `📂 영향 범위` 섹션을 추가해 **PR 본문 클릭 없이 채널에서 컨텍스트 전달** 가능하게 함
- **일관 포맷 강제** — 마커·순서 고정 (`✅` → `🔗` → `📝` → `📂` → `후속`). 매번 다른 헤더 쓰지 못하게 명시. 헤더(`##`)·코드 블록·표는 댓글에서 금지 유지
- **PR 요약 추출 절차 추가** — 댓글 작성 전 `git log <base>..HEAD --oneline` + `git diff --stat` + PR 본문 + Jira description 플랜을 종합해 bullet 생성. 카드 제목 그대로 복붙 금지
- **이어붙이기 블록도 동일 마커** — `🔁 추가 작업` 블록에도 `📝 변경 요약` 포함 (영향 범위는 선택)

### Files
- `commands/dct-complete.md` Step 5 — PR 요약 포맷으로 재작성 (5-2 추출 절차 신설)
- `commands/dct-job.md` Phase E.4 — 동일 포맷 반영
- `skills/dct-jira-workflow/SKILL.md` Phase 6 — 동일 포맷 반영

## 0.7.0 (2026-04-29)

### Changed
- **완료 댓글 컴팩트 포맷** — Jira-Slack 연동 채널 노이즈를 줄이기 위해 완료 댓글을 3~4줄로 축소. 신규 포맷 `✅ DCTC-<번호> 완료 — <한 줄 요약>` + `🔗 PR: <url>` + (선택) `후속:`. 변경 파일·검증 결과·표는 댓글에서 제거 (PR 본문/Jira description 으로 일원화)
- **이어붙이기 블록도 짧게** — `🔁 추가 작업 (YYYY-MM-DD)` + `✅ 한 줄 요약` + `🔗 PR` + 후속(선택), 헤더 남발 금지
- **마커 호환성** — 기존 댓글 탐지 시 신규 마커(`✅ DCTC-`)와 구버전 마커(`✅ 작업 완료 요약`) 모두 인식
- description(플랜) 포맷은 자세함 유지 — 본 변경은 댓글에만 한정

### Files
- `commands/dct-complete.md` Step 5 — 컴팩트 포맷으로 재작성
- `commands/dct-job.md` Phase E.4 — 동일 포맷 반영
- `skills/dct-jira-workflow/SKILL.md` Phase 6 — 동일 포맷 반영

## 0.6.0 (2026-04-21)

### Changed
- **`/dct-complete` · `/dct-job` Phase E — PR 생성 순서 재편**: 완료 댓글을 먼저 올리고 PR 링크만 따로 추가 댓글로 남기던 흐름을 폐기. **PR 생성 → PR URL 을 포함한 단일 완료 댓글** 순서로 통합. 댓글 포맷에 `## PR` 섹션 필수
- **PR 생성 전 병합 상태 확인 추가**: `gh pr list --head <branch> --state all` 로 기존 PR 상태를 먼저 조회. OPEN PR 이 이미 있으면 재사용, MERGED 상태에서 추가 커밋이 있으면 `(follow-up)` 표기로 **신규 PR**, 추가 커밋이 없으면 생성 건너뜀
- **재작업 시 덮어쓰기 금지 정책**:
  - Jira description 에 `📋 작업 플랜` 마커가 있으면 → 기존 원문 보존 + `## 🔁 추가 플랜 (YYYY-MM-DD, KST)` 섹션으로 **이어붙이기**
  - Jira 완료 댓글에 `✅ 작업 완료 요약` 마커가 있으면 → `jira_edit_comment` 로 **이어붙이기** (기존 원문 + `## 🔁 추가 작업 (YYYY-MM-DD, KST)` 섹션). **신규 댓글 생성 금지**
- `skills/dct-jira-workflow/SKILL.md` Phase 3/6 에 동일 정책 반영
- README / TEAM-GUIDE 한 줄 설명을 새 순서로 갱신

## 0.5.0 (2026-04-16)

### Added
- **`/dct` 온보딩 Step 10 — Playwright MCP (선택)** — 로컬 QA/E2E 자동화용 브라우저 조작 MCP
  - `claude mcp add playwright -- npx -y @playwright/mcp@latest` 한 줄 설치
  - Chromium 사전 다운로드 가이드 (첫 실행 타임아웃 방지)
  - `qa-tester` 에이전트 + tmux 조합 사용 패턴 안내
  - 스테이징/프로덕션 자동 조작 시 주의사항
- 환경 진단(Step 0) 에 Playwright MCP 체크 항목 추가

### Changed
- `/dct` 최종 점검(Step 10) 을 Step 11 로 재번호
- 최종 점검 체크리스트에 Playwright MCP 연결 확인 추가

## 0.4.0 (2026-04-16)

### Added
- **`/dct-job` 에 `/review` Phase 추가 (Phase C)** — 구현(Phase B) 직후 Claude Code 내장 `/review` 커맨드를 호출해 품질/보안/로직 정적 리뷰 수행. High/Critical 지적사항은 Phase 안에서 수정 루프 (최대 2회)
- Phase 구조 재편: A(플랜) → B(구현) → **C(/review)** → D(검증) → E(마무리)
- 중단 지점에 `/review` High/Critical 설계 판단 항목 추가

### Changed
- **권장 파이프라인 문서화**: `/dct-plan` → `/ultrawork` | `/autopilot` → `/review` → `/dct-complete`
- `/dct-plan` 종료 메시지를 4단계 파이프라인 다이어그램으로 갱신
- `/dct-job` "수동 분리 버전" 섹션을 4단계 파이프라인으로 갱신
- README 빠른 시작에 완전 자동화(`/dct-job`) vs 수동 4단계 두 가지 경로 명시

### Notes
- `/review` 는 **Claude Code 네이티브 빌트인 커맨드** (DCT 플러그인의 `/dct-sc-review` 아님). 둘은 별도로 존재하며 빌트인 사용을 권장
- 기존 `/dct-sc-review` 는 DCT 커스텀 리뷰 흐름용으로 유지

## 0.3.0 (2026-04-14)

### Added
- **`/dct` 온보딩에 GCP CLI 섹션 (Step 7)** — Cloud Storage 중심 설정 가이드
  - `gcloud auth login` + `gcloud auth application-default login` 2단계 인증
  - 다중 프로젝트 대응 (named configuration 가이드 포함)
  - `gcloud storage` 우선 사용 권장 (구 `gsutil` 대비 2~5배 빠름)
  - 비용 주의사항 (egress, 재귀 조회, BigQuery 스캔량)
- **`rules/gcp.md` 신규 규칙** — GCP/gcloud/Cloud Storage 작업 시 권장 패턴 + 금지사항
  - IAM 권한 변경 금지, 서비스 계정 JSON 키 사용 지양, 토큰 출력 명령 금지
  - Cloud Storage `gcloud storage` 우선, Cross-region egress 비용 경고
- `core/CLAUDE.md` 규칙 인덱스에 `gcp.md` 등록

### Changed
- `/dct` 온보딩 단계 재번호 매김: Slack MCP 7→8, RTK 8→9, 최종점검 9→10
- 환경 진단(Step 0) 에 GCP auth/project 체크 추가
- README 규칙 목록 9개로 확장

## 0.2.1 (2026-04-14)

### Fixed
- **`/dct-plan` 실제 plan mode 진입**: Claude Code frontmatter 에 `mode: plan` 필드는 존재하지 않음 (공식 지원 필드 아님). 본문에 **"첫 액션으로 `EnterPlanMode` 툴 호출 필수"** 지시 추가로 실제 plan mode 진입 보장
- `/dct-plan` frontmatter 에 `effort: max` 추가 (Opus 4.6 전용 최대 추론 예산)
- `/dct-job` Phase A 에도 동일한 `EnterPlanMode` 첫 액션 지시 추가

## 0.2.0 (2026-04-14)

### Added
- **`jira-adf-table` 스킬** — markdown 표를 Jira ADF 로 변환하면서 컬럼 너비를 가중 글자수 비례로 자동 계산. 한글/CJK/이모지 2배 가중. mcp-atlassian + curl fallback 경로 모두 지원
- **`/dct-plan` 에 ultrathink 지시문** — 카드 범위 판단/단계 분해/영향 파일 예측을 최대 추론 예산으로 수행
- README 헤더 배너 이미지 추가

### Changed
- **`/dct-job` Phase A → planmode**: 내장 plan mode (`EnterPlanMode`/`ExitPlanMode`) 로 증거 기반 분석 후 명시적 승인. 승인 전 mutation 금지
- **`/dct-job` Phase B → ultrawork**: 독립 작업을 ultrawork 스킬로 병렬 executor 위임, 티어 매핑(haiku/sonnet/opus), 장시간 작업 background 실행
- `/dct-plan`, `/dct-complete` 에 `jira-adf-table` 스킬 트리거 추가 (표가 포함될 때만)

## 0.1.0 (초기 릴리스)

- `/dct` 온보딩 커맨드 (Atlassian MCP, SSH, gh CLI, CLAUDE.md/rules 배포)
- `/dct-plan`, `/dct-complete`, `/dct-job` Jira 워크플로우
- `/dct-slack`, `/dct-slack-bot`, `/dct-refresh-slack` Slack 전송 + AWS Secrets 연동
- `/dct-rtk` RTK 토큰 절감 설정
- `/dct-sc-*` 개발 워크플로우 커맨드 (analyze/implement/test/debug/optimize/review)
- 팀 기본 규칙 8종 (`rules/*.md`)
