---
name: dct
description: 데이터 컨설팅 팀(DCT) Claude Code 온보딩 — MCP 셋업, SSH 키 생성, 팀 CLAUDE.md/rules 배포 가이드
---

# /dct — DCT 온보딩

데컨팀 신규 팀원이 Claude Code를 처음 셋업할 때 실행. MCP 연결과 팀 기본 설정을 단계별로 안내한다.

## 실행 단계

`dct-onboarding` 스킬을 참조하여 아래 순서로 진행:

### 0. 현재 환경 진단 (자동)
온보딩 시작 전 **이미 설정된 항목을 자동 검사**해서, 필요한 단계만 안내한다. 사용자 입력 없이 Claude 가 직접 실행.

```bash
# A. Atlassian MCP
jq '.mcpServers["mcp-atlassian"] // empty' ~/.claude.json 2>/dev/null

# B. GitHub SSH
ls ~/.ssh/*.pub 2>/dev/null

# C. gh CLI
gh auth status 2>&1

# D. 팀 CLAUDE.md / rules
ls ~/.claude/CLAUDE.md ~/.claude/rules/ 2>/dev/null

# E. AWS CLI
aws sts get-caller-identity 2>&1

# F. GCP CLI (Cloud Storage 중심)
gcloud auth list --filter=status:ACTIVE --format="value(account)" 2>/dev/null
gcloud config get-value project 2>/dev/null

# G. Slack MCP
jq '.mcpServers.slack // empty' ~/.claude.json 2>/dev/null

# H. RTK (토큰 절감)
which rtk 2>/dev/null && rtk --version

# I. RTK hook
jq '.hooks.PreToolUse // empty' ~/.claude/settings.json 2>/dev/null | grep -q 'rtk' && echo "RTK hook registered" || echo "RTK hook not registered"

# J. Playwright MCP (로컬 QA, 선택)
jq '.mcpServers.playwright // empty' ~/.claude.json 2>/dev/null
```

진단 결과를 **체크리스트 형식**으로 사용자에게 출력:
```
🔍 DCT 온보딩 환경 진단

✅ Atlassian MCP — 연결됨 (mcp-atlassian in ~/.claude.json)
✅ SSH 키 — ~/.ssh/id_ed25519.pub 존재
✅ gh CLI — JHN-MAD 로그인 / SSH 프로토콜
✅ CLAUDE.md — 존재
✅ rules/ — 9개 파일 존재
❌ AWS CLI — 인증 안 됨 → 6단계에서 설정
❌ GCP CLI — 인증 안 됨 → 7단계에서 설정 (Cloud Storage 중심)
❌ Slack MCP — 미등록 → 8단계에서 설정 (AWS 선행 필수)
❌ RTK — 미설치 → 9단계에서 설정 (토큰 60~90% 절감)
❌ Playwright MCP — 미등록 → 10단계에서 설정 (로컬 QA 자동화)
```

- ✅ 항목은 **건너뛰기**
- ❌ 항목만 해당 단계로 안내 (단계 번호와 함께 링크)
- 전부 ✅ 이면: **"온보딩 완료 상태입니다. 추가 설정이 필요하면 개별 단계를 직접 요청하세요."** 메시지 출력 후 종료

> **주의**: 진단 시 `~/.claude.json` 이나 `~/.claude/settings.json` 의 env 값(토큰)은 절대 출력하지 말 것. 키 존재 여부(`// empty`)만 확인.

---

### 1. 사용자가 `settings-example.json` 직접 편집
사용자는 리포 루트의 `settings-example.json` 파일을 에디터로 열어 Atlassian 값을 실제 값으로 교체한다. **Claude 는 이 단계에서 대기**.

- 발급처: https://id.atlassian.com/manage-profile/security/api-tokens
- 교체 대상 키: `JIRA_USERNAME`, `JIRA_API_TOKEN`, `CONFLUENCE_USERNAME`, `CONFLUENCE_API_TOKEN`

> **⚠️ 커밋 금지 경고**: `settings-example.json` 은 git 추적 대상이다. 실제 토큰을 채운 뒤 **절대 `git add` / 커밋하지 말 것**. 작업이 끝나면 `git checkout settings-example.json` 으로 플레이스홀더 버전으로 되돌려 둔다. Claude 도 이 파일을 스테이지에 올리지 않는다.

사용자가 "작성 완료" 라고 알리면 다음 단계로.

### 2. `settings-example.json` → `~/.claude.json` 병합 반영
사용자가 채운 `settings-example.json` 의 `mcpServers.mcp-atlassian` 객체를 **`~/.claude.json`** 에 병합한다.

> **⚠️ 중요 — 대상 파일**: Claude Code 의 MCP 레지스트리는 `~/.claude.json` 이다 (`claude mcp list` 가 읽는 파일). `~/.claude/settings.json` 은 훅/권한/테마용이며 `mcpServers` 를 넣어도 무시될 수 있다. **반드시 `~/.claude.json` 을 타겟으로 할 것.**

1. 기존 `~/.claude.json` 존재 여부 확인
   - **존재**: `cp ~/.claude.json ~/.claude.json.bak-$(date +%Y%m%d-%H%M%S)` 로 백업. **절대 `Write` 로 통째 덮어쓰지 말 것** (하단 "절대 규칙" 참조). 이 파일에는 Claude Code 의 프로젝트별 히스토리, OAuth 토큰, 플러그인 메타 등 수십 개 최상위 키가 있어 손상 시 복구 불가능
   - **없음**: **중단**. 사용자에게 "Claude Code 를 먼저 한 번 실행해 `~/.claude.json` 초기 설정을 생성한 뒤 `/dct` 를 다시 실행하세요" 안내. 빈 객체로 수동 생성하지 말 것 (기존 사용자가 파일을 실수로 삭제한 경우 빈 객체 생성이 복구 불가능한 손실을 유발)
2. `settings-example.json` 에서 Atlassian 플레이스홀더(`YOUR_..._HERE`, `your-email@example.com` 등)가 남아있는지 검증 — 남아있으면 중단하고 사용자에게 재편집 요청
3. **병합 방식으로 반영** (`jq` 사용):
   ```bash
   jq --slurpfile src <(jq '.mcpServers["mcp-atlassian"]' settings-example.json) \
      'if .mcpServers then . else . + {mcpServers: {}} end
       | .mcpServers["mcp-atlassian"] = $src[0]' \
      ~/.claude.json > ~/.claude.json.tmp \
      && mv ~/.claude.json.tmp ~/.claude.json
   ```
   - `mcp-atlassian` 키만 추가/갱신. `projects`, `oauthAccount`, `installedPlugins` 등 다른 키는 반드시 보존
   - `mcpServers` 키 자체가 없던 파일에도 대응 (위 jq 표현식의 if 분기)
4. 검증: `jq '.mcpServers | keys' ~/.claude.json` 으로 `mcp-atlassian` 키 존재 확인 (값은 출력하지 말 것)
5. **작업 종료 후 반드시 `git checkout settings-example.json`** 으로 리포 파일을 플레이스홀더 버전으로 되돌림 — 실수 커밋 방지

### 3. GitHub SSH 키 생성 및 등록 (조건부)
**기존 SSH 키가 이미 있으면 재사용한다.** 먼저 `ls ~/.ssh/*.pub 2>/dev/null` 로 기존 키 확인.

- **기존 키 있음** (`id_ed25519.pub`, `id_rsa.pub` 등) → 건너뛰고 4단계로. 키 이름에 `dct` 가 붙어있지 않아도 무방
- **기존 키 없음** → 아래 명령으로 새 키 생성:
  ```bash
  ssh-keygen -t ed25519 -C "$(whoami)@madup-dct" -f ~/.ssh/id_ed25519
  ```
  - 파일 이름은 기본 `id_ed25519` 권장 (별도 이름은 필수 아님)
  - 생성 후 `~/.ssh/id_ed25519.pub` 내용을 GitHub 계정 → Settings → SSH and GPG keys 에 등록
- `~/.ssh/config` 는 기본 키를 쓰면 별도 설정 불필요. 여러 계정을 분리해야 하는 경우에만 Host 블록 추가

### 4. `gh` CLI 인증 (`gh auth login`)
GitHub PR 생성 및 이슈 조회는 `gh` CLI 로 수행한다. PAT 수동 발급 없이 OAuth flow로 인증한다.

```bash
gh auth login
```
- `GitHub.com` 선택
- `SSH` 프로토콜 선택 (3단계에서 만든 SSH 키와 연동)
- 브라우저 인증 (one-time code)
- 완료 후 확인:
  ```bash
  gh auth status
  ```
  `Logged in to github.com` 및 `Git operations protocol: ssh` 가 보여야 정상.
- 이미 로그인돼 있으면 이 단계는 건너뛴다.

### 5. 팀 기본 CLAUDE.md / rules 배포 (조건부)
**기존 파일이 이미 있으면 건너뛴다.** 사용자가 OMC 등 다른 환경을 이미 설정해둔 경우 통째 교체는 치명적 손실을 유발하므로 절대 덮어쓰지 않는다 ("절대 규칙" 참조).

분기 로직:
- **`~/.claude/CLAUDE.md` 없음** → 플러그인 `core/CLAUDE.md` 를 그대로 복사
- **`~/.claude/CLAUDE.md` 있음** → **건너뛰기**. 사용자에게 "기존 CLAUDE.md 가 존재하여 배포를 건너뜁니다. 필요하면 플러그인의 `core/CLAUDE.md` 내용을 수동으로 참고해 병합하세요" 안내만 출력
- **`~/.claude/rules/` 디렉터리 없음** → 플러그인 `rules/*.md` 전체 복사
- **`~/.claude/rules/` 존재** → 파일별로 검사, **없는 파일만** 추가 복사. 기존 파일은 **덮어쓰지 않음**

> **안내**: 배포되는 규칙은 **DCT 팀 프로젝트 전용**이다 (예: `team-workflow.md` 의 "main PR 금지, dev 기본" 등). 개인 프로젝트에는 프로젝트별 `.claude/CLAUDE.md` 에서 override 가능하므로 전역 규칙이 개인 작업을 방해하지 않음을 안내한다.

### 6. AWS CLI (선택, Slack MCP 전제 조건)
AWS 리소스 접근이 필요하거나 **8단계 Slack MCP 를 쓸 계획이면 필수**. Slack 봇 토큰을 AWS Secrets Manager 에서 가져오기 때문.

MCP 는 설치하지 않고 CLI 만 설정 — Claude 가 `Bash` 로 `aws` 직접 호출하는 것만으로 충분하다.

1. AWS CLI 설치 확인:
   ```bash
   aws --version
   ```
   없으면: `brew install awscli`
2. IAM Console 에서 **Access Key + Secret** 발급 (본인 IAM 사용자)
3. `aws configure` 실행 후 입력:
   - AWS Access Key ID
   - AWS Secret Access Key
   - Default region name: `ap-northeast-2` (서울)
   - Default output format: `json`
4. 인증 검증:
   ```bash
   aws sts get-caller-identity
   ```
   본인 IAM ARN 이 보이면 정상.
5. 한 번 설정하면 `~/.aws/credentials` + `~/.aws/config` 에 저장되어 **모든 AWS SDK/CLI/Claude Bash 호출에서 공유**된다. 도구별 재설정 불필요.

> **보안**: Access Key 는 `~/.aws/credentials` 에만 두고 `Read` 로 출력하지 말 것. 검증은 `aws configure list` 로 (실제 키 값은 마스킹 출력됨).

### 7. GCP CLI (선택, Cloud Storage 중심)
데컨팀 GCP 리소스(특히 **Cloud Storage**)에 접근이 필요한 경우만. AWS 와 동일한 패턴으로 **MCP 는 설치하지 않고 `gcloud` CLI 만 설정**한다. Claude 가 `Bash` 로 `gcloud storage` 를 직접 호출.

**전제 조건**: 없음 (AWS, Slack 과 독립 — 어느 단계든 먼저 진행 가능)

1. `gcloud` CLI 설치 확인:
   ```bash
   gcloud --version
   ```
   없으면: `brew install --cask google-cloud-sdk` (Homebrew) 또는 https://cloud.google.com/sdk/docs/install

2. **개인 계정 인증** (사용자 본인 identity):
   ```bash
   gcloud auth login
   ```
   - 브라우저가 열리면 **`@madup.com` 구글 계정**으로 로그인
   - SSO 가 동작하지 않거나 권한이 없는 경우 권한 담당자에게 접근 방법 문의
   - `~/.config/gcloud/` 에 credentials 저장 (이후 모든 `gcloud` 호출이 공유)

3. **Application Default Credentials** 설정 (Python SDK `google-cloud-storage` 등이 사용):
   ```bash
   gcloud auth application-default login
   ```
   `gcloud auth login` 과 별개의 credential. SDK/라이브러리 경유 접근이면 필수.

4. **기본 프로젝트 선택** — 데컨팀은 프로젝트가 여러 개:
   ```bash
   gcloud projects list                              # 접근 가능한 프로젝트 목록
   gcloud config set project <PROJECT_ID>            # 기본 프로젝트 지정
   ```
   작업별로 프로젝트 자주 전환해야 하면 named configuration 사용:
   ```bash
   gcloud config configurations create dct-analytics
   gcloud config set project <project-A>
   gcloud config configurations activate dct-analytics
   # 다른 프로젝트로 전환 시
   gcloud config configurations activate default
   ```

5. 인증/설정 검증:
   ```bash
   gcloud auth list                                   # active account (마스킹 이메일)
   gcloud config get-value project                    # 기본 project
   ```

6. **Cloud Storage 스모크 테스트**:
   ```bash
   gcloud storage buckets list --limit=5              # 접근 가능 버킷 5개
   gcloud storage ls gs://<bucket-name>/              # 버킷 내용
   gcloud storage cp local-file.txt gs://<bucket>/    # 업로드
   gcloud storage cp gs://<bucket>/file.txt ./        # 다운로드
   ```

   > **중요 — `gcloud storage` vs `gsutil`**: 구형 `gsutil` 대신 **`gcloud storage`** 사용 권장. 성능 2~5배 빠르고 Google 이 적극 관리. Claude 가 `Bash` 호출할 때 `gcloud storage` 를 우선 선택하도록 CLAUDE.md 규칙에 반영돼 있음.

> **보안**:
> - `~/.config/gcloud/` 전체를 `Read` 로 출력하지 말 것 (credentials 유출 방지). 검증은 `gcloud auth list` 로 (마스킹 이메일만 출력됨)
> - **서비스 계정 JSON 키 사용 지양** — 개인 계정(`gcloud auth login`) 으로 대체. JSON 키가 필요한 상황이면 담당자에게 문의. 키 파일은 절대 리포에 커밋 금지
> - `gcloud auth application-default print-access-token` 같은 명령은 실제 토큰을 출력하므로 **절대 실행 금지**

> **비용 주의**:
> - 대용량 버킷에 `gcloud storage ls -r gs://...` (재귀) 는 과금 + 장시간 소요. 테스트는 항상 `--limit` 사용
> - Cross-region 데이터 전송은 egress 비용 발생 — 버킷 리전(`gcloud storage buckets describe gs://...`) 확인 후 작업
> - BigQuery 쿼리 실행 시 스캔량 기반 과금 → `--dry-run` 으로 예상 스캔량 확인 후 실행

### 8. Slack MCP (선택)
팀 Slack(`madupteam.slack.com`) 봇 `@매도비` 를 통해 DM, 채널 메시지 전송이 필요한 경우만. 공식 `@modelcontextprotocol/server-slack` 서버 + **AWS Secrets Manager 의 `prod/gen-ai/slack` 에 저장된 봇 토큰** 사용.

**전제 조건**:
- 6단계 AWS CLI 설정 완료
- IAM 권한: `secretsmanager:GetSecretValue` on `prod/gen-ai/slack` (없으면 권한 담당자에게 요청)

**동작 방식**:
1. 온보딩 시 1회 `aws secretsmanager get-secret-value` 로 토큰 fetch
2. `~/.claude.json` 의 `mcpServers.slack.env.SLACK_BOT_TOKEN` 에 저장
3. 이후 세션은 파일에 저장된 토큰 사용 (매 세션 AWS 호출 X)
4. 토큰 로테이션 시 `/dct-refresh-slack` 로 갱신

**실행**:
```bash
bash ~/.claude/plugins/cache/dct-marketplace/dct-claude-plugin/0.1.0/scripts/refresh-slack-token.sh
```
스크립트가 수행하는 작업:
- 권한 검증 (`aws sts get-caller-identity`)
- Secrets Manager 에서 `SLACK_TOKEN` fetch
- `~/.claude.json` 백업 후 `jq` 로 `mcpServers.slack` 키만 병합
- `SLACK_TEAM_ID=T5D95TP5Z` 함께 설정
- 검증 출력

**캐시 워밍 (중요 — 첫 연결 타임아웃 방지)**:
korotovsky slack-mcp-server 는 첫 실행 시 워크스페이스 전체 유저/채널을 캐시하느라 **10초 이상** 소요될 수 있다. Claude Code 는 MCP 서버 시작 시 타임아웃이 있어 첫 연결이 실패할 수 있으므로, **온보딩 중 서버를 한 번 수동 실행해 캐시를 미리 생성**한다:
```bash
# 캐시 워밍 (1회만, 서버가 "Loaded users from cache" 출력하면 Ctrl+C)
npx -y slack-mcp-server
```
캐시 파일이 `~/Library/Caches/slack-mcp-server/` 에 저장되면, 이후 Claude Code 재시작 시 캐시에서 즉시 로드되어 타임아웃 없이 연결된다.

**중요 — `~/.claude.json` vs `~/.claude/settings.json`**:
- `~/.claude.json` ⭐ — Claude Code 의 **실제 MCP 레지스트리** (`claude mcp list` 가 읽는 곳). Slack MCP 는 이 파일에 등록해야 한다
- `~/.claude/settings.json` — 훅, 권한, 테마 등. `mcpServers` 키가 있어도 Claude Code 는 무시할 수 있음 (환경에 따라)
- 둘 다 **jq 로 특정 키만 병합** 해야 하며, 통째 덮어쓰기 금지 (절대 규칙)

**봇 권한 확인**:
- 봇 이름: `@매도비` (app_id: `A069U5ZRHR7`, bot user: `U06AHN2ALJW`)
- Token Scopes: `chat:write`, `im:write` 필수
- Private 채널에 메시지 보내려면 해당 채널에서 `/invite @매도비` 먼저 실행

### 9. RTK 설치 — 토큰 절감 (권장)
RTK(Rust Token Killer)는 Bash 명령 출력을 자동 압축해 **토큰 소비를 60~90% 줄여주는** CLI 프록시. Claude Code `PreToolUse` hook 으로 `git status` → `rtk git status` 처럼 투명하게 rewrite 한다.

1. RTK 설치:
   ```bash
   brew install rtk
   ```
   Homebrew 없으면: `curl -fsSL https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh | sh`
2. 설치 확인: `rtk --version`
3. Claude Code hook 등록:
   ```bash
   rtk init -g
   ```
   기존 hook 이 있으면 RTK hook 을 **추가** (덮어쓰지 않음)
4. 등록 확인: `rtk init --show`
5. (선택) Telemetry 비활성화: `~/.zshrc` 에 `export RTK_TELEMETRY_DISABLED=1` 추가
6. Claude Code **재시작** — 이후 Bash 도구의 `git status`, `ls`, `docker ps` 등이 자동으로 RTK 를 거쳐 압축된 출력 전달

> **참고**: RTK 는 Claude Code 내장 도구(`Read`, `Grep`, `Glob`)에는 영향 없음 — `Bash` 도구 셸 명령에만 적용. 제거: `rtk init -g --uninstall && brew uninstall rtk`. 리포: https://github.com/rtk-ai/rtk

### 10. Playwright MCP (선택, 로컬 QA 자동화)
프론트 QA/E2E 시나리오를 Claude 가 브라우저를 직접 조작하며 수행할 때 사용. `webapp-testing` 스킬과 tmux 를 조합하면 장기 세션(백엔드/프론트 로그 동시 관찰 + Playwright 실행)을 유지할 수 있다.

**전제 조건**: 없음 (독립)

1. MCP 등록:
   ```bash
   claude mcp add playwright -- npx -y @playwright/mcp@latest
   ```
   `claude mcp add` 명령이 내부적으로 `~/.claude.json` 의 `mcpServers.playwright` 키에 병합한다 (통째 덮어쓰기 아님).

2. Chromium 사전 다운로드 (첫 실행 타임아웃 방지):
   ```bash
   # 1회 실행 — 브라우저 번들 다운로드
   npx -y @playwright/mcp@latest --help
   ```
   용량이 커서(~200MB) 첫 실행 시 수십 초 걸림. Claude Code 가 MCP 시작 시점에 타임아웃 내리는 것을 막기 위해 미리 받아둔다.

3. 등록 확인:
   ```bash
   claude mcp list | grep playwright
   ```
   `playwright: ✓ Connected` 이 보여야 정상.

4. **사용 패턴** — 로컬 QA 흐름
   - 별도 터미널에서 대상 서비스 기동 (예: `make dev`) — **tmux 권장**
   - Claude Code 에서 `webapp-testing` 스킬로 QA 시나리오 실행
   - 스킬이 Playwright MCP 로 브라우저 조작 + 스크린샷/콘솔 로그 수집
   - 발견된 버그는 Jira DCTC 카드 생성 (또는 기존 카드 댓글) 로 연결

> **주의 사항**
> - Playwright MCP 는 `localhost` 접근에 제약 없음 — 내부 개발 서버 테스트 적합
> - 배포된 스테이징/프로덕션을 자동 조작할 때는 팀 합의 필수 (의도치 않은 mutation 방지)
> - MCP 비활성화: `claude mcp remove playwright` — 필요할 때만 활성화해도 됨 (컨텍스트 절약)

### 11. 최종 점검
- `claude mcp list` 로 MCP 연결 상태 확인 (값은 출력되지 않음)
  - `mcp-atlassian: ✓ Connected` 필수
  - (Slack 설정 시) `slack: ✓ Connected`
- `jq '.mcpServers | keys' ~/.claude.json` 로 등록된 MCP 키 확인 (값 출력 금지)
- `mcp-atlassian` 도구 호출 테스트 (예: `mcp__mcp-atlassian__jira_get_user_profile`)
- `gh auth status` 로 GitHub 인증 확인 (`Git operations protocol: ssh`)
- (AWS 설정 시) `aws sts get-caller-identity` 로 인증 확인
- (GCP 설정 시) `gcloud auth list` + `gcloud config get-value project` + `gcloud storage buckets list --limit=5` 로 확인
- (Slack 설정 시) `/dct-slack U<본인ID> 온보딩 테스트` 로 DM 스모크 테스트
- (RTK 설정 시) `rtk --version` + `rtk init --show` 로 hook 등록 확인
- (Playwright MCP 설정 시) `claude mcp list | grep playwright` 로 `✓ Connected` 확인
- `~/.claude/CLAUDE.md` 와 `~/.claude/rules/` 존재 확인
- 백업 파일 위치 안내 (복원 필요 시):
  - `cp ~/.claude.json.bak-<timestamp> ~/.claude.json`
  - `cp ~/.claude/settings.json.bak-<timestamp> ~/.claude/settings.json`
- 다음 커맨드 안내: `/dct-plan DCTC-1234 "작업 설명"` (또는 `/dct-job` 완전 자동화)

## 주의사항

### 🚨 절대 규칙 — 기존 파일 통째 교체 금지
**`~/.claude/settings.json`, `~/.claude/CLAUDE.md`, `~/.claude/rules/*` 등 사용자의 기존 설정 파일은 어떤 경우에도 `Write` 도구로 통째 덮어쓰지 않는다.** 기존 파일에는 OMC 훅, 권한, 플러그인 설정, 커스텀 메모리 등 플러그인이 모르는 내용이 들어있을 수 있고, 통째 교체 시 **복구 불가능한 손실**이 발생한다.

**필수 절차**:
1. **항상 먼저 백업**: `cp <file> <file>.bak-$(date +%Y%m%d-%H%M%S)` — 실패해도 복원 가능
2. **기존 구조 파악**: `jq 'keys'` 로 최상위 키만 확인 (값은 출력하지 말 것 — 시크릿 노출 방지)
3. **병합 방식 사용**:
   - `settings.json` → `jq` 로 `mcpServers.mcp-atlassian` 같은 **특정 키만 추가/갱신**
   - `CLAUDE.md` → 통째 교체 시 반드시 사용자에게 **diff 를 보여주고 명시적 승인** 받기
   - `rules/*.md` → 파일 단위로, 기존 동일 파일이 있으면 **덮어쓰기 전 사용자 확인**
4. **통째 덮어쓰기는 기존 파일이 존재하지 않을 때만 허용** (신규 생성)
5. **백업 경로를 사용자에게 명확히 알림** — 문제 발생 시 즉시 복원 가능하도록

이 규칙은 `/dct`, `/dct-job`, `/dct-plan`, `/dct-complete` 모든 커맨드에 공통 적용된다.

### 그 외
- API 토큰은 절대 커밋/로그/채팅에 노출 금지
- Claude 는 `~/.claude/settings.json` 전체를 `Read` 로 출력하지 않는다 (시크릿이 세션 로그에 남음). 검증은 `jq` 또는 `grep` 으로 키/패턴만 확인
- 덮어쓰기는 반드시 사용자 확인 후 진행
