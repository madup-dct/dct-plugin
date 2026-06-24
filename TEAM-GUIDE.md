# DCT Claude Code 플러그인 — 팀 온보딩 가이드

## 이게 뭔가요?

데이터 컨설팅 팀(DCT)이 Claude Code를 **같은 품질·같은 워크플로우**로 쓸 수 있게 만든 플러그인입니다.

설치하면:
- **`/dct` 온보딩 커맨드** — Atlassian MCP 설정, SSH 키, `gh auth login`, 팀 기본 설정을 단계별로 안내 (Slack/AWS 선택)
- **`/dct-job` Jira 워크플로우** — 카드 번호만 주면 브랜치 생성 → 플랜 → 구현 → Jira 댓글까지 자동 실행
- **팀 기본 규칙 8개** — 한국어 응답, 커밋 컨벤션, 브랜치 네이밍, Python/AWS/문서 동기화 등
- **`/dct-slack`, `/dct-refresh-slack`** — 봇 `@매도비` 통해 Slack 메시지 전송 및 토큰 갱신 (AWS Secrets Manager 연동)
- **품질 스킬** — TDD, 디버깅, 계획, 검증, 코드리뷰, 웹앱 테스트

## 왜 만들었나요?

데컨팀 초반에 AI Native 개발 환경 셋업에 시간이 많이 들고, 팀원마다 Claude 사용 방식이 달라서 품질 편차가 생겼습니다. 이 플러그인은:

- 신규 입사자가 **하루 안에** Claude Code + Jira/GitHub MCP 환경을 갖추게 하고
- Jira 카드 단위 작업 플로우를 **한 커맨드로 표준화** 하며
- 팀 공통 코딩/문서/워크플로우 규칙을 **한 곳에서** 관리합니다.

## 설치 (3분)

```bash
# 로컬 개발 모드
git clone https://github.com/JHN-MAD/dct-plugin.git ~/dct-plugin
claude --plugin ~/dct-plugin
```

또는 Claude Code 내에서:
```
/plugin marketplace add https://github.com/JHN-MAD/dct-plugin
/plugin install dct-claude-plugin
```

설치 후 Claude Code를 **재시작**하고 아래를 실행하세요:
```
/dct
```

## 온보딩 체크리스트 (`/dct` 가 안내)

1. **Atlassian MCP** — Jira/Confluence API 토큰 발급, `settings.json` 병합 (기존 설정 보존)
2. **GitHub SSH 키** — 기존 키 있으면 재사용, 없으면 기본 `id_ed25519` 생성
3. **`gh auth login`** — SSH 프로토콜로 OAuth 인증 (이미 로그인돼 있으면 스킵)
4. **팀 CLAUDE.md 배포** — `~/.claude/CLAUDE.md`, `~/.claude/rules/*.md` 8개 (기존 파일 있으면 건너뛰기)
5. **최종 점검** — MCP 연결, `gh auth status`, 파일 존재 확인
6. **선택** — Slack MCP (xoxc/xoxd), AWS CLI (`aws configure`)

## 일상 워크플로우

### Jira 카드 작업 — 수동 분리 (권장)
다른 도구/플러그인과 조합하고 싶을 때:
```
/dct-plan DCTC-1808 "프론트 사이드바 애니메이션 개선"
# → 플랜 작성, Jira 업로드, feature/DCTC-1808 브랜치 진입

# (자유) 직접 코딩, 네이티브 병렬 Task 등 무엇이든

/dct-complete DCTC-1808
# → 결과 요약 → PR 생성(확인 후) → PR URL 포함한 Jira 완료 댓글
```

### Jira 카드 작업 — 완전 자동화
한 번에 끝까지 실행하고 싶을 때:
```
/dct-job DCTC-1808 feat "프론트 사이드바 애니메이션 개선"
```
플랜 → 구현 → 검증 → PR 생성까지 논스톱. 중간 중단 지점(플랜 승인, PR 생성 여부)에서만 사용자 확인.

### Slack 메시지 전송
```
/dct-slack 정하늘 배포 완료했습니다          # 주소록에 저장된 이름으로 DM
/dct-slack mkt_데이터컨설팅_cp 스프린트 완료  # 채널에 공지
/dct-slack U0AL4JJMC48 테스트                 # Member ID 직접 (최초 전송 시)
```

## 프로젝트별 설정

각 프로젝트에 `.claude/CLAUDE.md` 를 두면 프로젝트 맞춤 설정이 가능합니다:

```bash
mkdir -p .claude
cp ~/dct-plugin/templates/project-claude.md.template .claude/CLAUDE.md
# 프로젝트에 맞게 수정 후 커밋
```

## 문제가 있다면
- 기존 `~/.claude/CLAUDE.md` 가 있으면 `/dct` 는 **건너뜁니다** (덮어쓰지 않음). 수동 병합이 필요하면 플러그인의 `core/CLAUDE.md` 참고
- MCP 연결 실패 시 `~/.claude/settings.json` 의 JSON 문법/토큰을 먼저 확인
- 이슈 리포트: https://github.com/JHN-MAD/dct-plugin/issues
