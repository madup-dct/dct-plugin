---
name: dct-sc-implement
description: TDD 기반 단계별 기능 구현 — Red(테스트 먼저) → Green(최소 구현) → Refactor 사이클
argument-hint: "<구현할 기능 설명>" [--tdd] [--no-tdd]
---

# /dct-sc-implement — 기능 구현

TDD(Red-Green-Refactor) 사이클 기반으로 기능을 구현한다. `--no-tdd` 시 테스트 없이 직접 구현.

## 사용법

```
/dct-sc-implement "<설명>" [--tdd] [--no-tdd]
```

**예시**
- `/dct-sc-implement "사용자 프로필 API 엔드포인트 추가"` — 기본 TDD
- `/dct-sc-implement "사이드바 애니메이션 개선" --no-tdd` — 테스트 없이 구현
- `/dct-sc-implement "./docs/feature-spec.md"` — 파일에서 명세 로드

## 실행 순서

### Phase 1 — 컨텍스트 파악
1. 설명이 파일 경로면 `Read` 로 명세 로드
2. 프로젝트 구조 탐색 (`Glob`) — 언어, 프레임워크, 테스트 프레임워크 식별
3. 관련 파일 읽기 — 수정/추가할 모듈, 기존 패턴 파악

### Phase 2 — 구현 계획
1. 변경/추가할 파일 목록 예상
2. 사용자에게 **계획 미리보기** 제시
3. TDD 모드 확인:
   - `--tdd` 또는 기본값 → Red-Green-Refactor 모드
   - `--no-tdd` → Phase 3 건너뛰고 바로 Phase 4

### Phase 3 — Red (테스트 먼저 작성) [TDD 모드만]
1. 기능 요구사항을 테스트 케이스로 번역
2. **실패하는 테스트** 작성 (기능 미구현 상태)
3. 테스트 실행 → 실패 확인 (Red)
4. 사용자에게 테스트 코드 확인 요청

### Phase 4 — Green (최소 구현)
1. 테스트를 통과하는 **최소한의 코드** 작성
2. 기존 코드 패턴/컨벤션 준수 (프로젝트 스타일 따르기)
3. 테스트 실행 → 통과 확인 (Green) [TDD 모드]

### Phase 5 — Refactor
1. 중복 제거, 네이밍 개선, 구조 정리
2. 테스트 다시 실행 → 여전히 통과 확인
3. 린트/포맷터 실행 (프로젝트에 있으면)

### Phase 6 — 완료 리포트
```
✅ 구현 완료

## 변경 파일
- src/api/profile.py (신규)
- tests/test_profile.py (신규)
- src/api/__init__.py (수정: 라우트 등록)

## 테스트 결과
- 통과: 5/5
- 커버리지: 기존 82% → 84%

## 다음 단계
- /dct-sc-review 로 코드 리뷰
- /dct-complete DCTC-XXXX 로 마무리
```

## 규칙 준수
- 기존 프로젝트 패턴을 존중 (새로운 추상화 최소화)
- 불필요한 docstring/주석 추가 금지 (변경하지 않은 코드)
- 보안: 하드코딩된 시크릿/eval()/exec() 금지
- 멀티파일 변경은 `general-purpose` 서브에이전트 위임 고려
