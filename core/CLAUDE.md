# 규칙 인덱스 (On-Demand Loading)

모든 사용자 규칙은 `~/.claude/rules/` 에서 관리된다. **전체 규칙을 자동으로 로드하지 말 것.**
아래 인덱스에서 현재 작업과 관련된 규칙 파일만 `Read` 도구로 읽어 적용한다. (컨텍스트 절약)

## 상시 적용 (세션 시작 시 1회 Read 권장)
- `~/.claude/rules/korean-dev.md` — 응답 언어(한국어), 커밋/네이밍 컨벤션
- `~/.claude/rules/personal.md` — 사용 정책 관련 금지사항
- `~/.claude/rules/performance.md` — 모델 선택, 컨텍스트 관리
- `~/.claude/rules/agents.md` — 서브에이전트 위임 기준
- `~/.claude/rules/read-before-edit.md` — 코드 수정 전 반드시 파일 읽기 (추측 수정 금지)

## 상황별 (트리거 시 Read)
- `~/.claude/rules/python.md` — Python(.py, pytest, poetry 등) 작업 시
- `~/.claude/rules/team-workflow.md` — GitHub 브랜치/커밋, Jira, Confluence 작업 시
- `~/.claude/rules/jira-writing.md` — Jira 완료 댓글/플랜 작성 시 (가독성·근거 중심 라이팅)
- `~/.claude/rules/aws.md` — AWS/aws-cli/IAM/S3/EC2 관련 작업 시
- `~/.claude/rules/gcp.md` — GCP/gcloud/Cloud Storage/BigQuery 관련 작업 시
- `~/.claude/rules/doc-updates.md` — 작업 완료 직전 (문서 동기화 체크)

## 운영 원칙
- 트리거 키워드가 없으면 해당 규칙을 읽지 않는다.
- 동일 세션 내 이미 읽은 규칙은 재-Read 하지 않는다.
- 새 규칙은 이 인덱스에 한 줄씩 추가한다 (본문을 CLAUDE.md에 인라인하지 말 것).

## DCT 플러그인 커맨드
- `/dct` — 신규 팀원 온보딩 (MCP/SSH/팀 기본 설정)
- `/dct-plan <DCTC-번호> [설명]` — 플랜 작성 + Jira 업로드 + 브랜치 진입 (구현은 자유)
- `/dct-complete <DCTC-번호>` — 결과 요약 + Jira 완료 댓글 + PR 생성 (확인 후)
- `/dct-job <DCTC-번호> <타입> <설명>` — 플랜→구현→검증→PR 완전 자동화
- `/dct-slack <ID|이름> <메시지>` — 본인 계정으로 Slack 전송 (korotovsky)
- `/dct-slack-bot <ID|이름> <메시지>` — 봇 `@매도비` 로 Slack 전송
- `/dct-refresh-slack` — AWS Secrets Manager 에서 Slack 봇 토큰 갱신
- `/dct-sc-analyze`, `/dct-sc-implement`, `/dct-sc-test`, `/dct-sc-debug`, `/dct-sc-optimize`, `/dct-sc-review` — 개발 워크플로우 커맨드

## 핵심 원칙
- **Evidence > assumptions** — 추측 대신 근거
- **Read before Write** — 쓰기 전 반드시 읽기
- **Batch parallel** — 독립 호출은 병렬 실행
- **Verify after completion** — 완료 주장 전 검증

## 금지
- 명시적 요청 없는 자동 커밋
- 검증 생략, 안전 프로토콜 우회
- 프로젝트 전역 탐색 없이 코드베이스 변경
