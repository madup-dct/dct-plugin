# Jira 라이팅 규칙 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Jira 완료 댓글/플랜을 "왜·근거·검색성" 중심으로 쓰도록 공용 규칙 파일을 만들고 기존 커맨드/스킬이 이를 참조하게 한다.

**Architecture:** `rules/jira-writing.md`를 단일 소스로 신규 작성하고, `core/CLAUDE.md` 인덱스에 등록한다. `commands/dct-complete.md`와 `skills/dct-jira-workflow/SKILL.md`의 중복 포맷 블록은 규칙 참조로 슬림화하고, 두 파일에서 `📂 영향 범위` 마커를 제거한다. 코드가 아닌 markdown 문서 작업이므로 각 task는 "작성 → grep 검증" 사이클로 닫는다.

**Tech Stack:** Markdown, git, grep (검증).

## Global Constraints

- 응답·문서·커밋은 한국어. 컨벤셔널 prefix만 영어 (`docs:`, `refactor:` 등).
- 코드/파일 수정 전 반드시 `Read` 후 `Edit`.
- 기존 규칙 파일 톤 유지: 짧게(~2KB), 불릿 위주, 헤더 계층 얕게.
- 기존 운영 디테일(이어붙이기, main 직접 PR 금지, 중복 PR 방지)은 **손실 없이 유지**.
- 마커 최종 세트: `✅ / 🔗 / 📝 / 💡(선택) / 후속(선택)`. `📂 영향 범위`는 제거.
- 스펙: `docs/superpowers/specs/2026-06-19-jira-writing-rules-design.md`.

---

### Task 1: 규칙 파일 신규 작성 + 인덱스 등록

**Files:**
- Create: `rules/jira-writing.md`
- Modify: `core/CLAUDE.md` (상황별 인덱스, line 13 아래)

**Interfaces:**
- Produces: `rules/jira-writing.md` — Task 2·3이 "작성 기준은 `rules/jira-writing.md` 참조"로 가리키는 단일 소스. 마커 세트 `✅ / 🔗 / 📝 / 💡(선택) / 후속(선택)`, 원칙 5개, 안티패턴, 완료 댓글/플랜 포맷, before/after 예시를 정의한다.

- [ ] **Step 1: `rules/jira-writing.md` 작성**

아래 전문을 그대로 파일로 만든다.

```markdown
# Jira Writing (지라 라이팅)

Jira 완료 댓글·description 플랜을 "협업자가 다시 읽을 가치가 있게" 쓰는 규칙.
AI 라이팅의 함정(과정 나열·세션 로그 복붙)을 막고, 독자가 **왜·결론·근거**를 즉시
파악하고 **나중에 검색**할 수 있게 한다. `/dct-complete`, `dct-jira-workflow` 스킬이 참조한다.

## 원칙 5

1. **결론 먼저** — 첫 줄은 결과("무엇이 어떻게 됐다"). 과정·배경 narration 금지.
2. **왜 > 무엇** — 변경 bullet은 `<무엇> — <왜/효과>` 한 문장. 코드 보면 아는 "무엇"만 적지 않는다.
3. **검색되게** — 첫 줄·bullet에 고유명사(모듈/기능/에러명)를 넣는다. 카드 제목 복붙 금지.
4. **스캔 가능** — 마커·순서 고정, 산문체 금지. 5초 안에 구조 파악되게.
5. **세션 로그 ≠ 문서** — AI가 한 모든 일을 옮기지 않는다. PR에서 확인 가능한 것(변경 파일 목록, diff stat)은 댓글에 중복하지 않는다. 독자가 행동·판단에 쓸 것만 남긴다.

## 안티패턴 (하지 말 것)

| 하지 말 것 | 대신 |
|------------|------|
| 세션 히스토리·도구 호출·내부 사고 나열 | 결과와 판단 근거만 |
| 매번 다른 헤더 사용 | 마커·순서 고정 |
| 카드 제목 그대로 복붙 | 실제 변경 기반 재작성 |
| "왜" 없이 변경만 나열 | `<무엇> — <왜>` 한 문장 |
| PR에서 보이는 정보(파일 목록·stat) 댓글에 복붙 | PR 링크로 위임 |

## 완료 댓글 포맷

```
✅ DCTC-<번호> 완료 — <결과 한 줄, 고유명사 포함>
🔗 PR: <url>   (또는: 생성 안 함 / 이미 병합됨)

📝 변경 요약
• <무엇> — <왜/효과>   (3~5개)

💡 결정: <대안 대비 왜 이 방향, 한 줄>   ← 판단이 갈렸을 때만, 평소 생략
후속: <있을 때만 한 줄, 없으면 줄 생략>
```

- 마커: `✅ / 🔗 / 📝 / 💡(선택) / 후속(선택)`. 다른 헤더(`##`)·코드 블록·표 금지.
- 본문 **10줄 이내** 권장. 이어붙이기(재작업) 블록은 별도 카운트.
- 영향 범위·변경 파일 전체 목록·검증 결과는 **PR 본문**에서 확인 (댓글에 중복 금지).
- 재작업 시: 기존 완료 댓글을 수정해 `---` + `🔁 추가 작업 (YYYY-MM-DD, KST)` 블록으로 이어붙인다. 새 댓글 생성 금지.

## description 플랜 포맷

- 목표: 사용자 언어로 1~2줄. AI 재진술·장황한 배경 금지.
- 단계: "할 일" 동사구 + 검증 기준. 내부 사고과정·도구 나열 금지.
- 영향 파일: 경로만. 추측성 "예상" 남발 금지.
- 분량: 목표+단계+영향파일이 **화면 한 눈**. 넘으면 PR/별도 문서로.

## 예시 (before → after)

**나쁨** (세션 로그·과정 나열·왜 없음):
```
✅ 작업 완료 요약
로그인 기능을 수정했습니다. 먼저 코드를 읽고, AuthService를 분석한 뒤
여러 파일을 수정했습니다. LoginForm.tsx, authService.ts, api.ts 를 변경했고
테스트를 돌려서 통과했습니다. 자세한 건 PR을 봐주세요.
```

**좋음** (결론 먼저·왜·검색 키워드):
```
✅ DCTC-1808 완료 — 만료 토큰 무한 리다이렉트 해결
🔗 PR: https://github.com/.../pull/42

📝 변경 요약
• AuthService 토큰 갱신을 응답 인터셉터로 이동 — 만료 시 401 루프 차단
• LoginForm 리다이렉트를 useEffect 단일 진입으로 정리 — 중복 네비게이션 제거

💡 결정: 토큰 갱신을 컴포넌트가 아닌 인터셉터에 둠 — 페이지 전역 재사용 위해
```
```

- [ ] **Step 2: 작성 검증 (마커·핵심 항목 존재)**

Run: `grep -c "원칙 5\|안티패턴\|완료 댓글 포맷\|description 플랜 포맷\|before → after" rules/jira-writing.md`
Expected: `5` (다섯 항목 모두 존재)

Run: `grep -n "📂" rules/jira-writing.md`
Expected: 출력 없음 (영향 범위 마커 미사용)

- [ ] **Step 3: `core/CLAUDE.md` 인덱스에 등록**

`## 상황별 (트리거 시 Read)` 섹션의 `team-workflow.md` 줄 바로 아래에 한 줄 추가한다.

기존:
```
- `~/.claude/rules/team-workflow.md` — GitHub 브랜치/커밋, Jira, Confluence 작업 시
```
교체:
```
- `~/.claude/rules/team-workflow.md` — GitHub 브랜치/커밋, Jira, Confluence 작업 시
- `~/.claude/rules/jira-writing.md` — Jira 완료 댓글/플랜 작성 시 (가독성·근거 중심 라이팅)
```

- [ ] **Step 4: 인덱스 등록 검증**

Run: `grep -n "jira-writing.md" core/CLAUDE.md`
Expected: `## 상황별` 섹션에 한 줄 출력

- [ ] **Step 5: 커밋**

```bash
git add rules/jira-writing.md core/CLAUDE.md
git commit -m "feat(dct-jira-workflow): Jira 라이팅 규칙 파일 추가 및 인덱스 등록

왜·근거·검색성 중심의 완료 댓글/플랜 작성 규칙을 rules/jira-writing.md
단일 소스로 정의. core/CLAUDE.md 상황별 인덱스에 등록해 on-demand 로딩.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: `dct-complete` 커맨드 슬림화 + 📂 제거

**Files:**
- Modify: `commands/dct-complete.md` (line 82, 98-99, 104-146 영역)

**Interfaces:**
- Consumes: `rules/jira-writing.md` (Task 1) — 포맷 상세를 이 파일로 위임.

- [ ] **Step 1: Step 5 헤더에 규칙 참조 추가**

line 80~83 영역의 `### 5. Jira 완료 댓글 등록/수정 (PR 요약 포맷)` 블록 안내문 중,
`> 💡 **일관된 마커 유지** — \`📝 변경 요약\`, \`📂 영향 범위\`, \`후속:\` 순서·이름 고정. 매번 다른 헤더 쓰지 말 것.` 줄을 아래로 교체한다.

교체 후:
```
> 💡 **작성 기준은 `~/.claude/rules/jira-writing.md` 참조** — 원칙(결론 먼저·왜>무엇·검색성), 마커 세트, 분량 상한이 거기에 정의됨. 아래는 이 커맨드의 운영 흐름만 다룬다.
> 💡 **일관된 마커 유지** — `📝 변경 요약`, `후속:` 순서·이름 고정. 매번 다른 헤더 쓰지 말 것.
```

- [ ] **Step 2: 5-2 추출 규칙에서 영향 범위 분리 문구 정리**

line 98~99 두 줄을 아래로 교체한다.

기존:
```
- **변경 요약**: 3~5개 bullet. 각 줄 **60~90자**. "무엇을 / 왜" 위주, "어디서" 는 영향 범위로 분리
- **영향 범위**: 변경 파일이 2개 영역 이상일 때만 추가. 1개 영역이면 생략
```
교체:
```
- **변경 요약**: 3~5개 bullet. 각 줄 `<무엇> — <왜/효과>` 형식. 영향 범위·파일 목록은 PR 본문에서 확인(댓글 중복 금지)
```

- [ ] **Step 3: 5-3 A안 신규 포맷에서 📂 블록 제거**

line 104~124 영역의 신규 작성 포맷(```✅ DCTC-... ``` 코드 블록)에서 `📂 영향 범위` 블록과 관련 안내를 제거하고 `💡 결정:` 선택 라인을 추가한다.

기존 코드 블록:
```
  ✅ DCTC-<번호> 완료 — <한 줄 요약>
  🔗 PR: <pr url>   (또는: 생성 안 함 / 이미 병합됨)

  📝 변경 요약
  • <변경 1 — 무엇/왜>
  • <변경 2>
  • <변경 3>
  (3~5개 bullet)

  📂 영향 범위   ← 파일 그룹 2개 이상일 때만 포함
  • <영역/모듈 1>: <한 줄 설명>
  • <영역/모듈 2>: <한 줄 설명>

  후속: <있을 때만 한 줄, 없으면 줄 자체 생략>
```
교체:
```
  ✅ DCTC-<번호> 완료 — <결과 한 줄, 고유명사 포함>
  🔗 PR: <pr url>   (또는: 생성 안 함 / 이미 병합됨)

  📝 변경 요약
  • <무엇> — <왜/효과>
  (3~5개 bullet)

  💡 결정: <대안 대비 왜, 한 줄>   ← 판단 갈렸을 때만, 평소 생략
  후속: <있을 때만 한 줄, 없으면 줄 자체 생략>
```

이어서 line 123 `- "한 줄 요약" 은 카드 제목이 아니라 ...` 와 line 124 `- 검증 결과·변경 파일 ...` 안내는 그대로 둔다(영향 범위 언급 없음).

- [ ] **Step 4: 5-3 B안(이어붙이기)에서 📂 안내 제거**

line 144 `- 추가 블록의 \`📂 영향 범위\` 는 선택 (변경이 여러 영역에 걸쳐있을 때만)` 줄을 **삭제**한다.

- [ ] **Step 5: 5-3 마커 목록에서 📂 제거**

line 146 줄을 아래로 교체한다.

기존:
```
- 헤더(`##`)·코드 블록·표 사용 금지 — 마커는 위 4종 (`✅`/`🔗`/`📝`/`📂`/`후속`) 만
```
교체:
```
- 헤더(`##`)·코드 블록·표 사용 금지 — 마커는 `✅`/`🔗`/`📝`/`💡`(선택)/`후속`(선택) 만
```

- [ ] **Step 6: 📂 잔존 없음 검증**

Run: `grep -n "📂\|영향 범위" commands/dct-complete.md`
Expected: line 93의 `git diff ... --stat — 파일별 변경 규모 (영향 범위)` 만 남음(이건 PR 컨텍스트 설명이라 유지). `📂` 이모지는 0건.

Run: `grep -c "📂" commands/dct-complete.md`
Expected: `0`

- [ ] **Step 7: 규칙 참조 존재 검증**

Run: `grep -n "rules/jira-writing.md" commands/dct-complete.md`
Expected: Step 1에서 추가한 참조 줄 출력

- [ ] **Step 8: 커밋**

```bash
git add commands/dct-complete.md
git commit -m "refactor(dct-complete): 포맷 규칙을 jira-writing 참조로 슬림화

완료 댓글 포맷 상세를 rules/jira-writing.md 로 위임하고 운영 흐름만 유지.
📂 영향 범위 마커 제거(PR diff stat 중복), 변경 bullet을 무엇—왜 형식으로,
💡 결정 선택 라인 추가.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: `dct-jira-workflow` 스킬 슬림화 + 📂 제거

**Files:**
- Modify: `skills/dct-jira-workflow/SKILL.md` (Phase 6, line 83-113 영역)

**Interfaces:**
- Consumes: `rules/jira-writing.md` (Task 1).

- [ ] **Step 1: Phase 6 신규 포맷에서 📂 블록 제거 + 규칙 참조**

line 83~99 영역의 완료 댓글 포맷 안내와 신규 코드 블록을 수정한다.
먼저 line 83 `3. 완료 댓글 포맷 — **마커·순서 고정** (Slack 채널 패턴 인식 일관성):` 아래에 참조 한 줄을 추가한다.

교체 후:
```
3. 완료 댓글 포맷 — **마커·순서 고정** (작성 기준 상세는 `~/.claude/rules/jira-writing.md` 참조):
```

이어서 line 84~99의 신규 코드 블록에서 `📂 영향 범위` 블록을 제거하고 `💡 결정:` 라인을 추가한다.

기존:
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
교체:
```
    ✅ DCTC-<번호> 완료 — <결과 한 줄, 고유명사 포함>
    🔗 PR: <url>   (또는: 생성 안 함 / 이미 병합됨)

    📝 변경 요약
    • <무엇> — <왜/효과>
    (3~5개)

    💡 결정: <대안 대비 왜, 한 줄>   ← 판단 갈렸을 때만, 평소 생략
    후속: <있을 때만 한 줄, 없으면 생략>
```

- [ ] **Step 2: 마커 목록에서 📂 제거**

line 113 줄을 아래로 교체한다.

기존:
```
  - 마커 4종(`✅`/`🔗`/`📝`/`📂`/`후속`) 외 헤더(`##`)·코드 블록·표 사용 금지
```
교체:
```
  - 마커(`✅`/`🔗`/`📝`/`💡`(선택)/`후속`(선택)) 외 헤더(`##`)·코드 블록·표 사용 금지
```

- [ ] **Step 3: 📂 잔존 없음 검증**

Run: `grep -c "📂" skills/dct-jira-workflow/SKILL.md`
Expected: `0`

- [ ] **Step 4: 규칙 참조 존재 검증**

Run: `grep -n "rules/jira-writing.md" skills/dct-jira-workflow/SKILL.md`
Expected: Step 1에서 추가한 참조 줄 출력

- [ ] **Step 5: 커밋**

```bash
git add skills/dct-jira-workflow/SKILL.md
git commit -m "refactor(dct-jira-workflow): 완료 댓글 포맷을 jira-writing 참조로 정리

Phase 6 포맷에서 📂 영향 범위 마커 제거, 변경 bullet 무엇—왜 형식,
💡 결정 선택 라인 추가. 작성 기준 상세는 rules/jira-writing.md 위임.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: 전체 일관성 최종 검증

**Files:** 없음 (검증만)

- [ ] **Step 1: 네 파일 전체에서 📂 마커 완전 제거 확인**

Run: `grep -rn "📂" rules/jira-writing.md core/CLAUDE.md commands/dct-complete.md skills/dct-jira-workflow/SKILL.md`
Expected: 출력 없음

- [ ] **Step 2: 규칙 단일 소스 참조 일관성 확인**

Run: `grep -rln "jira-writing.md" core/CLAUDE.md commands/dct-complete.md skills/dct-jira-workflow/SKILL.md`
Expected: 세 파일 모두 출력 (인덱스 + 두 참조)

- [ ] **Step 3: 운영 디테일 손실 없음 확인**

Run: `grep -c "main 직접\|이미 병합\|기존 PR\|이어붙\|append\|기존 완료 댓글" commands/dct-complete.md skills/dct-jira-workflow/SKILL.md`
Expected: 두 파일 모두 1 이상 (PR 중복방지·이어붙이기·main 금지 흐름 유지됨)

- [ ] **Step 4: 변경 요약 마커 일관성 확인**

Run: `grep -rn "💡 결정" rules/jira-writing.md commands/dct-complete.md skills/dct-jira-workflow/SKILL.md`
Expected: 세 파일 모두 `💡 결정` 출현 (마커 세트 일치)

검증 통과 시 작업 완료. 추가 커밋 불필요(검증만 수행).
