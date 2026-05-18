---
name: dct-gcs
description: DCT GCP Cloud Storage 운영 — 멀티프로젝트 전환·비용 가드레일·안전 검증. gcloud CLI 기반 (MCP 없음).
argument-hint: <ls | cp | sync | use | check> [인자]
---

# /dct-gcs — DCT Cloud Storage 운영

데컨팀 여러 GCP 프로젝트의 Cloud Storage 를 **안전하게** 다룬다. `dct-gcs` 스킬을 백엔드로 사용하며, MCP 없이 `gcloud` CLI 만 호출한다.

> 최초 인증 셋업이 안 됐으면 `/dct` E단계 먼저.

## 사용법

```
/dct-gcs <서브커맨드> [인자]
```

| 서브커맨드 | 용도 | 예시 |
|---|---|---|
| `check` | 인증·ADC·프로젝트·버킷 검증 (변경 없음) | `/dct-gcs check` |
| `use` | 프로젝트/named config 전환 | `/dct-gcs use dct-analytics` |
| `ls` | 객체 목록 (자동 `--limit`, 재귀 차단) | `/dct-gcs ls gs://my-bucket/raw/` |
| `cp` | 업로드/다운로드 (cross-region 경고) | `/dct-gcs cp ./out.csv gs://my-bucket/exports/` |
| `sync` | rsync (dry-run 선행) | `/dct-gcs sync ./data gs://my-bucket/data` |

## 동작 원칙

- 모든 데이터 작업 전 **현재 프로젝트를 출력**해 혼선 차단 (`gcloud config get-value project`)
- 재귀 `ls -r` / cross-region `cp`·`sync` / `rsync --delete` 는 **실행 전 1회 확인**
- `gsutil` 대신 **`gcloud storage`** 사용 (2~5배 빠름)
- credentials·access token 절대 출력 안 함

## 인증 분기

| 접근 | credential | 셋업 |
|---|---|---|
| `gcloud storage` CLI | user 계정 | `gcloud auth login` |
| Python SDK (`google-cloud-storage` 등) | ADC | `gcloud auth application-default login` |

`DefaultCredentialsError` 면 ADC 누락 — `gcloud auth application-default login` 안내.

## 프로젝트 전환

```
/dct-gcs use <named-config>      # gcloud config configurations activate
/dct-gcs use <PROJECT_ID>        # gcloud config set project
```
단발성이면 각 명령에 `--project=<ID>` 플래그 (활성 config 유지).

## 실패 대응

| 에러 | 1순위 원인 | 해결 |
|---|---|---|
| `404 bucket does not exist` | 활성 프로젝트 혼선 | `gcloud config get-value project` → `/dct-gcs use ...` |
| `403 permission` | IAM 권한 부족 | 담당자에 버킷+role 명시 요청 |
| `DefaultCredentialsError` | ADC 누락 | `gcloud auth application-default login` |
| `Reauthentication required` | 토큰 만료 | `gcloud auth login` |

## 규칙 준수

- `~/.config/gcloud/` `Read`/`cat` 금지 — 검증은 `gcloud auth list` 로
- 서비스 계정 JSON 키 지양 (개인 계정 사용), 키 파일 커밋 금지
- 파괴적/비용 작업은 사용자 확인 후 — wildcard 삭제 금지

## 관련

- `dct-gcs` 스킬 — 프로젝트 전환·가드레일·에러 대응 로직
- `/dct` E단계 — GCP CLI 설치·인증 셋업
- `/dct-slack` — 동일 패턴의 팀 운영 커맨드 (Slack)
