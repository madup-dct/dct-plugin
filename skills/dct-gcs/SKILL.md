---
name: dct-gcs
description: DCT 팀 GCP Cloud Storage 운영 스킬. 멀티프로젝트 config 전환, 비용/리전 가드레일, ADC 분기, 안전 검증 원라이너를 담당한다. MCP 없이 gcloud CLI 만 사용.
---

# DCT GCS 스킬

`/dct-gcs` 커맨드의 백엔드. 데컨팀이 **여러 GCP 프로젝트**에 걸쳐 Cloud Storage 를 다룰 때 반복되는 함정(프로젝트 혼선, cross-region egress 과금, 재귀 ls 폭주, ADC 누락)을 가드레일로 막는다.

## 설계 원칙

- **MCP 없음 — `gcloud` CLI 만** — Claude 가 `Bash` 로 `gcloud storage` 직접 호출 (AWS 패턴과 동일)
- **`gsutil` 금지, `gcloud storage` 사용** — 2~5배 빠르고 Google 이 적극 관리
- **프로젝트 전환은 gcloud named configuration 으로** — 별도 JSON 캐시 만들지 않음 (gcloud 가 이미 관리)
- **비용 작업은 사전 확인 후 실행** — 재귀/대용량/cross-region 은 디폴트 차단
- **credentials 절대 출력 금지** — `~/.config/gcloud/` Read/cat 금지, access token 출력 명령 금지

## 인증 모델 (분기 기준)

| 접근 경로 | 필요한 credential | 명령 |
|---|---|---|
| `gcloud storage` CLI 직접 | user 계정 | `gcloud auth login` |
| Python `google-cloud-storage` 등 SDK 경유 | ADC | `gcloud auth application-default login` |

> **반복 실패 패턴**: SDK 코드가 `DefaultCredentialsError` → `gcloud auth login` 만 했고 **ADC 누락**. `gcloud auth application-default login` 안내.

## 멀티프로젝트 전환

데컨팀은 GCP 프로젝트가 여러 개다. **현재 프로젝트를 매 작업 전 확인**하고, 전환은 named configuration 으로:

```bash
gcloud config configurations list                       # 등록된 config 목록
gcloud config configurations activate <config-name>      # 전환
gcloud config get-value project                          # 현재 프로젝트 확인 (작업 전 필수)
```

config 가 없으면 1회 생성 안내:
```bash
gcloud config configurations create <name>
gcloud config set project <PROJECT_ID>
```

> 단발성 전환이면 `gcloud storage ... --project=<PROJECT_ID>` 플래그로 처리 (활성 config 안 건드림).

## 핵심 명령 매핑

| 용도 | 명령 | 가드 |
|------|------|------|
| 버킷 목록 | `gcloud storage buckets list --limit=N` | `--limit` 필수 |
| 객체 목록 | `gcloud storage ls gs://<bucket>/<prefix>` | 재귀(`-r`) 디폴트 금지 |
| 버킷 메타(리전) | `gcloud storage buckets describe gs://<bucket> --format="value(location)"` | copy/sync 전 선행 |
| 업로드 | `gcloud storage cp <local> gs://<bucket>/<path>` | 대상 버킷 리전 확인 후 |
| 다운로드 | `gcloud storage cp gs://<bucket>/<path> <local>` | — |
| 동기화 | `gcloud storage rsync <src> <dst>` | dry-run 우선 |

## 실행 플로우

### Phase 1 — 인자 파싱
- 첫 토큰 = 서브커맨드 (`ls` / `cp` / `sync` / `use` / `check`)
- 나머지 = 서브커맨드 인자

### Phase 2 — 사전 점검 (모든 데이터 작업 공통)
1. `gcloud config get-value project` 로 현재 프로젝트 출력 → 사용자에게 명시
2. `gcloud auth list --filter=status:ACTIVE --format="value(account)"` 비어있으면 → "인증 안 됨, `gcloud auth login` 실행" 안내 후 중단

### Phase 3 — 서브커맨드별 처리

**`use <config|PROJECT_ID>`** — 프로젝트/config 전환
- config 이름 매칭되면 `configurations activate`
- `^[a-z][-a-z0-9]{4,28}[a-z0-9]$` (프로젝트 ID 패턴) 이면 `gcloud config set project`
- 전환 후 `gcloud config get-value project` 로 확정 출력

**`ls <gs://...>`** — 객체 목록
- 입력에 `-r`/`--recursive` 포함 → **확인 프롬프트** (아래 비용 확인)
- 기본은 단일 prefix, `--limit` 미지정 시 자동으로 `--limit=100` 부여

**`cp <src> <dst>`** — 복사
1. 대상이 `gs://` 면 버킷 리전 조회 (`buckets describe`)
2. src 도 `gs://` 면 두 버킷 리전 비교 → 다르면 **cross-region egress 경고 + 확인**
3. 확인되면 `gcloud storage cp` 실행

**`sync <src> <dst>`** — rsync
1. **먼저 dry-run**: `gcloud storage rsync --dry-run <src> <dst>` 결과 요약 제시
2. 사용자 확인 후 실제 `rsync` 실행 (`--delete-unmatched-destination-objects` 는 명시 요청 시에만)

**`check`** — 환경 검증 (변경 없음)
```bash
gcloud auth list --filter=status:ACTIVE --format="value(account)"
gcloud config get-value project
gcloud auth application-default print-access-token >/dev/null 2>&1 && echo "ADC OK" || echo "ADC 미설정"
gcloud storage buckets list --limit=5 --format="value(name)"
```
> `print-access-token` 의 stdout 은 `/dev/null` 로 버려 토큰 노출 차단. exit code 만 사용.

### Phase 4 — 결과 보고
- 성공: `✅ <서브커맨드> 완료 — project=<현재프로젝트>`
- 실패: 아래 에러 대응표

## 비용 확인 프롬프트

아래 작업은 **실행 전 1회 확인**:

| 트리거 | 프롬프트 |
|---|---|
| 재귀 `ls -r` | `⚠️ 재귀 목록은 대용량 버킷에서 과금+장시간. prefix 좁히거나 --limit 권장. 계속? (y/N)` |
| cross-region cp/sync | `⚠️ <src리전> → <dst리전> 전송은 egress 비용 발생. 계속? (y/N)` |
| `rsync` 실삭제 옵션 | `⚠️ --delete 는 대상의 미매칭 객체를 삭제합니다. 계속? (y/N)` |

## 에러 대응

### `DefaultCredentialsError` / ADC 관련
SDK 경유 접근인데 ADC 누락. 안내:
```
❌ Application Default Credentials 없음
해결: gcloud auth application-default login
(gcloud auth login 과 별개 credential — SDK/라이브러리는 ADC 사용)
```

### `403` / `does not have storage.objects.* permission`
IAM 권한 부족. 안내:
```
❌ 권한 부족 — 현재 계정에 해당 버킷 권한 없음
확인: gcloud config get-value account / gcloud config get-value project
해결: 권한 담당자에게 버킷+role 명시해 요청 (예: roles/storage.objectViewer)
```

### `404` / `bucket does not exist`
프로젝트 혼선 가능성 1순위. 안내:
```
❌ 버킷/객체 없음 — 활성 프로젝트가 맞는지 먼저 확인
gcloud config get-value project
다른 프로젝트면: /dct-gcs use <config-or-project>
```

### `Reauthentication required` / 토큰 만료
```
❌ 인증 만료
해결: gcloud auth login (CLI) / gcloud auth application-default login (SDK)
```

## 규칙 준수 / 보안

- **`~/.config/gcloud/` 를 `Read`·`cat` 금지** — credentials 유출 방지. 검증은 `gcloud auth list` 로 (마스킹 이메일만)
- **access token 출력 명령 금지** — `gcloud auth ... print-access-token` 의 stdout 은 절대 노출하지 않음 (`>/dev/null`, exit code 만)
- **서비스 계정 JSON 키 지양** — 개인 계정(`gcloud auth login`) 으로 대체. 키 필요 시 담당자 문의, 키 파일 **리포 커밋 금지**
- **파괴적 작업(`rm`, `rsync --delete`) 은 사용자 확인 후** — wildcard 삭제 금지
- **비용 작업은 사전 확인** — 재귀/cross-region/대용량은 디폴트 차단

## 관련

- `/dct-gcs` — 본 스킬의 커맨드 진입점
- `/dct` E단계 — GCP CLI 설치·인증 셋업 (`gcloud auth login`, ADC, 프로젝트 지정)
- `dct-onboarding` 스킬 — 최초 셋업 가이드
