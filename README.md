<p align="center">
  <img src="assets/madobi-community.jpeg" alt="DCT Plugin" width="480" />
</p>

# DCT Claude Code Plugin

데이터 컨설팅 팀(DCT)의 AI Native 개발 환경 온보딩 및 **Jira 워크플로우 자동화** 플러그인. GitHub 연동은 `gh` CLI + SSH 조합으로 단순화했다.

## 이게 뭐예요?

데컨팀 신규 팀원이 Claude Code를 빠르게 셋업하고, 팀 표준 워크플로우(Jira 카드 기반 개발 → 브랜치 → 구현 → 댓글 정리)를 한 커맨드로 실행할 수 있게 해주는 **Claude Code 네이티브 플러그인**입니다.

## 주요 기능

### 커맨드
| 커맨드 | 설명 |
|--------|------|
| `/dct` | 신규 팀원 온보딩 — Atlassian MCP 설정, SSH 키, `gh auth login`, 팀 CLAUDE.md/rules 배포 (Slack/AWS 선택) |
| `/dct-plan <DCTC-번호> [설명]` | 플랜 작성 → Jira 업로드 → `feature/DCTC-N` 브랜치 진입 (구현은 자유) |
| `/dct-complete <DCTC-번호>` | 결과 요약 → PR 생성(사용자 확인 후) → PR URL 포함한 Jira 완료 댓글 |
| `/dct-job <DCTC-번호> <타입> <설명>` | 플랜→구현→검증→PR 완전 자동화 파이프라인 |
| `/dct-slack <ID\|이름> <메시지>` | **본인 계정**으로 Slack 메시지 전송 (korotovsky, xoxc/xoxd) |
| `/dct-slack-bot <ID\|이름> <메시지>` | 봇 `@매도비` 로 Slack 메시지 전송 (server-slack, xoxb) |
| `/dct-refresh-slack` | AWS Secrets Manager 에서 Slack 봇 토큰 갱신 |
| `/dct-gcs <ls\|cp\|sync\|use\|check> [인자]` | DCT Cloud Storage 운영 — 멀티프로젝트 전환·비용 가드레일 (`gcloud` CLI) |
| `/dct-rtk` | RTK(Rust Token Killer) 설치 + Claude Code hook 연동 — 토큰 60~90% 절감 |
| `/dct-sc-analyze <경로>` | 코드 품질/보안/성능/아키텍처 종합 분석 (P0~P3 리포트) |
| `/dct-sc-implement "<설명>"` | TDD 기반 기능 구현 (Red-Green-Refactor) |
| `/dct-sc-test [경로]` | 테스트 실행·커버리지·누락 테스트 자동 생성 |
| `/dct-sc-debug "<에러>"` | 근본 원인 추적·경쟁 가설 검증·재현 테스트 |
| `/dct-sc-optimize <경로>` | 성능 병목 분석·3가지 최적화 제안 |
| `/dct-sc-review [--branch]` | 코드 리뷰 + PR 준비 상태 판정 |

### 스킬
- `dct-onboarding` — `/dct` 백엔드
- `dct-jira-workflow` — `/dct-job` 백엔드
- `test-driven-development` — TDD 워크플로우
- `systematic-debugging` — 체계적 디버깅
- `writing-plans` / `executing-plans` — 계획 작성·실행
- `verification-before-completion` — 완료 전 검증
- `requesting-code-review` / `receiving-code-review` — 코드 리뷰
- `webapp-testing` — 웹앱 테스트 (Playwright 기반)

### 팀 기본 규칙 (`rules/`)
플러그인 설치 후 `/dct` 실행 시 `~/.claude/rules/` 로 배포되는 9개 규칙:
- `korean-dev.md` — 한국어 응답/커밋, 코드는 영어 유지
- `personal.md` — 사용 정책
- `performance.md` — 모델 선택, 컨텍스트 관리
- `agents.md` — 서브에이전트 위임 기준
- `python.md` — Python 가이드
- `team-workflow.md` — GitHub/Jira/Confluence 워크플로우
- `aws.md` — AWS 작업 가이드
- `gcp.md` — GCP/Cloud Storage 작업 가이드 (`gcloud storage` 우선)
- `doc-updates.md` — 문서 동기화

## 설치

### 방법 1: 플러그인 디렉토리 참조 (로컬 개발)
```bash
git clone https://github.com/JHN-MAD/dct-plugin.git ~/dct-plugin
claude --plugin ~/dct-plugin
```

### 방법 2: marketplace 등록 후 설치 (권장)
Claude Code 내에서:
```
/plugin marketplace add https://github.com/JHN-MAD/dct-plugin
/plugin install dct-claude-plugin
```

설치 후 Claude Code를 **재시작**하고 `/dct` 를 실행하세요.

## 빠른 시작

```bash
# 1. 플러그인 설치 (위 참고)

# 2. Claude Code 재시작

# 3. 온보딩 실행
/dct
```

### Jira 카드 작업 — 완전 자동화
한 커맨드로 논스톱 실행:
```
/dct-job DCTC-1808 feat "프론트 사이드바 애니메이션 개선"
```
내부 파이프라인: `planmode` → `ultrawork` → `/review` → 검증 → PR

### Jira 카드 작업 — 수동 4단계 (권장, 도구 교체 자유)
```
/dct-plan DCTC-1808 "설명"      # 1. 플랜 + Jira 업로드 + 브랜치 진입
/ultrawork  또는  /autopilot    # 2. 구현 (병렬 / 자율 — 상황에 맞게)
/review                         # 3. Claude Code 내장 리뷰
/dct-complete DCTC-1808         # 4. PR 생성 → PR URL 포함한 Jira 완료 댓글
```

## MCP 설정

`settings-example.json` 이 리포 루트에 있습니다. 사용자가 이 파일에 값을 채우면 `/dct` 커맨드가 `jq` 로 `~/.claude.json` (Claude Code 의 실제 MCP 레지스트리) 에 **병합**합니다 (기존 설정 보존).

- **mcp-atlassian** (uvx) — Jira + Confluence (필수)
- **slack** (korotovsky/slack-mcp-server, xoxc/xoxd) — 선택
- GitHub 은 MCP 를 쓰지 않고 **`gh` CLI + SSH** 로 처리
- AWS 는 MCP 없이 **`aws` CLI** 로 처리 (Claude 가 Bash 로 호출)

## 구조

```
claude-team-config/
├── .claude-plugin/
│   ├── plugin.json          # 플러그인 매니페스트
│   └── marketplace.json     # /plugin marketplace add 지원
├── commands/
│   ├── dct.md               # /dct — 온보딩
│   ├── dct-plan.md          # /dct-plan — 플랜 + 브랜치 진입
│   ├── dct-complete.md      # /dct-complete — 마무리 + PR
│   ├── dct-job.md           # /dct-job — 완전 자동화 래퍼
│   ├── dct-slack.md         # /dct-slack — Slack 메시지 전송
│   ├── dct-refresh-slack.md # /dct-refresh-slack — 토큰 갱신
│   └── dct-gcs.md           # /dct-gcs — Cloud Storage 운영
├── skills/                  # dct-onboarding, dct-jira-workflow, dct-slack, dct-gcs + 품질 스킬
├── scripts/                 # refresh-slack-token.sh
├── rules/                   # 팀 기본 규칙 8개
├── core/
│   └── CLAUDE.md            # 사용자 전역 CLAUDE.md 템플릿
├── templates/
│   └── project-claude.md.template
├── settings-example.json    # Atlassian/Slack MCP 설정 템플릿
└── README.md
```

## 문의/기여
- 리포: https://github.com/JHN-MAD/dct-plugin
- 이슈/PR 환영
