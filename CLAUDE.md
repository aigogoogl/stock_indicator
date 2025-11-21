# SDD: TradingView 지표 기반 종목 추천 서비스

> 목적: TradingView 스타일(커스텀 Pine 포함 가능) 지표를 사용자가 선택하면, **전일(또는 지정일) 종가 기준**으로 지표 조건에 맞는 종목을 랭킹하여 제공하는 서비스의 Spec-Driven Development 문서.

---

## 1. 개요 & 목표

* **제품 요약**: 사용자가 웹 UI에서 지표와 파라미터를 선택하면, 지정된 날짜(기본: 전일 종가)를 기준으로 전체 또는 선택된 종목군에서 지표 조건을 충족하는 종목을 점수화·정렬해 제공한다. 추가로 지표별 백테스트 성능과 인기 지표 가이드를 제공한다.
* **주요 목표**:

  * PoC → MVP → Production으로 이어지는 단계별 구현 가능하도록 세부 요구사항과 수용 기준(acceptance criteria)을 정의.
  * SDD를 기준으로 자동화 테스트 및 CI/CD 파이프라인과 연계.

## 2. 권고: 문서 형식(Why?)

* **권장 방식**: GitHub 저장소의 루트에 `spec/` 디렉토리를 만들고, 핵심 SDD 문서는 **Markdown (`.md`) 파일**로 관리하며, 보조 산출물(ERD, 다이어그램)은 `spec/assets`에 SVG/PNG로 추가.
* **이유**:

  * Markdown은 GitHub에서 바로 렌더링되어 PR 리뷰/코멘트가 쉬움.
  * `spec-kit` 같은 별도 툴은 구조화·템플릿 제공이 장점이나, 초기 단계에서는 단순 MD 기반으로 빠르게 협업·수정하기 유리. 필요 시 `spec-kit`으로 옮길 수 있도록 `spec/manifest.md`를 준비.
  * Markdown + 깃 워크플로우(Branch, PR, CODEOWNERS)로 SDD 기반 개발(테스트 → 코드)을 자연스럽게 연결 가능.

## 3. 문서 구조 (파일/폴더)

```
/spec
  /00-readme.md                 # SDD 요약 (이 파일)
  /01-requirements.md           # 기능/비기능 요구사항
  /02-api-spec.md               # REST API / Request-Response 스펙
  /03-data-model.md             # DB 스키마, 테이블별 필드, 인덱스, retention
  /04-etl-spec.md               # 데이터 수집/스케줄/정합성 규칙
  /05-indicators-catalog.md     # 지원 지표 목록 + 파라미터 설명 + preset
  /06-backtest-spec.md          # 백테스트 정책, 메트릭, 예시 구성
  /07-ui-ux-spec.md             # 화면 플로우, 컴포넌트, 예시 이미지
  /08-ci-cd-tests.md            # Acceptance tests, E2E, unit tests matrix
  /09-security-ops.md          # 인증·권한·데이터 라이선스·비밀관리
  /10-deployment.md             # infra 구성, 배포 단계, 오퍼레이션 체크리스트
  /11-acceptance-criteria.md    # 각 기능별 수용 기준 (테이블)
  /assets/*.png/svg             # ERD, sequence diagrams, mockups
```

## 4. 핵심 섹션 요약 (각 파일에 절대 들어가야 할 항목)

### 01-requirements.md (기능/비기능)

* 기능 요구사항(FR) 번호화: FR-001 ~

  * FR-001: 지표 선택 UI 제공 (지표 목록+파라미터)
  * FR-002: 지정일(기본 전일)의 종가 기준 스캔 API 제공
  * FR-003: 지표 결과에 대한 정렬·필터링 및 설명(왜 추천됐는지) 제공
  * FR-004: 백테스트 집계(기간: 1d,1w,1m,1y) 및 지표 성능 랭킹
  * FR-005: 사용자 저장 필터·알림 기능(향후)
* 비기능 요구사항(NFR): 응답시간(예: Top100 요청 2초 이내 캐시 포함), 신뢰성, 데이터 정확도(데이터 소스 명시), 동시요청 처리량, 로그 및 모니터링 요구

### 02-api-spec.md (REST API 계약)

* 각 엔드포인트 표준화 (OpenAPI/Swagger 스타일 예시 포함)

  * `POST /api/v1/scan`  — body: indicator, params, universe, date, limit; response: { date, indicator, results[] }
  * `GET /api/v1/indicators` — supported indicators, presets, descriptions
  * `GET /api/v1/backtest` — indicator, params, period → metrics
  * `GET /api/v1/ticker/{symbol}` — OHLC, indicator values
* 요청/응답 사례(JSON) 2개 이상 포함(성공/오류)
* 에러 코드 표준(BAD_REQUEST, RATE_LIMITED, DATA_NOT_FOUND, INTERNAL)

### 03-data-model.md

* 테이블 목록 + 샘플 DDL

  * tickers, ohlcv, indicator_values, signals, backtest_results, users_filters
* 인덱스 권장, retention 정책(예: OHLCV raw 10년, indicator_values 3년)
* 데이터 소스 메타 (source id, update_time, license)

### 04-etl-spec.md

* 수집 소스(예: yfinance for PoC, Polygon/Tiingo for production)
* 스케줄: EOD 파이프라인(장마감+30분 후 실행), 재시도 로직, 데이터 품질 체크(누락, 스플릿/배당 보정)
* 데이터 변환 규칙(조정종가, 환율 고려 등)

### 05-indicators-catalog.md

* 각 지표: short desc, input params, default values, preset examples, limitations, pine-compatibility note
* 예: RSI(14): 입력(period=14), signal: below X, above Y

### 06-backtest-spec.md

* 백테스트 기법: walk-forward(rolling), 비용(슬리피지/수수료), 거래 규칙(진입/청산), 포지션사이징(고정/비례)
* 메트릭 정의 및 계산식(CAGR, Sharpe, MDD 등)
* 리포트 샘플(테이블+그래프 명세)

### 07-ui-ux-spec.md

* 화면 목록: 대시보드, Indicator selector, Scan result, Ticker detail, Backtest view, Saved filters
* 컴포넌트 명세: IndicatorCard, ParamForm, ResultsTable, ChartModal
* UX 흐름 다이어그램(간단한 sequence + wireframe 이미지)

### 08-ci-cd-tests.md

* Acceptance tests 매핑: 각 FR에 대응하는 Test ID
* Unit test coverage 목표, E2E 시나리오(Playwright or Cypress)
* Performance benchmark: scan on universe 2000 tickers (목표 시간)

### 09-security-ops.md

* 인증(OAuth2 + API keys), 권한(읽기/쓰기), 비밀관리(vault), TLS 요구사항
* 데이터 라이선스 관리: 무료 vs 유료 소스 표시, 로깅 규칙

### 10-deployment.md

* Infra: Docker image + k8s (namespace), DB(Primary+Replica), object storage
* Runbook: daily job, incident checklist, alert thresholds

### 11-acceptance-criteria.md

* 표 형태로 각 FR의 수용 기준과 테스트 방법 포함

  * 예: FR-002 — "`POST /api/v1/scan`이 200 OK로 Top100을 3초 이내 반환" + Test case id

## 5. Acceptance Criteria 예시 (간단 테이블)

| ID     | 기능       | 기준                             | 테스트 방법                    |
| ------ | -------- | ------------------------------ | ------------------------- |
| FR-001 | 지표 선택 UI | 지표 목록 20개 이상, 각 지표의 파라미터 툴팁 존재 | UI E2E 테스트(스크린샷 비교)       |
| FR-002 | 스캔 API   | 캐시된 응답 Top100 2초 이내            | 부하 테스트(가정: universe=1000) |

## 6. SDD/Specification 스타일 가이드

* 모든 요구사항은 **ID**가 있어야 함(FR/NFR/DF for Data). PR에서 변경 시 반드시 CHANGELOG 업데이트.
* API 스펙은 OpenAPI 3.0 YAML/JSON로 동시에 제공 (자동으로 Swagger UI 생성되도록). `spec/02-api-spec.md` 내에 YAML 스니펫 포함.
* Acceptance tests는 Test-ID를 명시하고, 각 Test-ID는 CI 파이프라인에서 자동 실행되도록 매핑.

## 7. 개발-검증 워크플로우 (SDD 중심)

1. 요구사항 작성(PR → 리뷰 (기능 소유자, 데이터 엔지니어, QA))
2. Acceptance Criteria 합의(테스트 케이스 작성)
3. 구현 브랜치: `feature/<FR-ID>-short-desc` → PR 생성
4. CI: lint → unit tests → contract tests(예: API schema) → E2E(샘플 환경)
5. Merge → Deploy to staging → Acceptance tests → Production

## 8. 문서 유지·관리 정책

* 변경 프로세스: SDD 변경은 PR로만 가능, 승인자 최소 2명(제품·데이터 소유자)
* 버전관리: `spec/00-readme.md` 상단에 spec 버전 표기(예: v0.1)
* 릴리즈와 연동: 서비스 major release 시 SDD 버전 bump

## 9. 초기 작업(당장 할 일 목록)

1. GitHub 리포지토리 루트에 `spec/` 디렉토리 생성하고 위 파일들 초안 추가. (우선 00~05 파일 우선 작성)
2. `spec/00-readme.md`에 목적·버전·컨트리뷰션 가이드 추가
3. OpenAPI 스켈레톤 파일 생성(간단한 `POST /api/v1/scan` 포함)
4. Indicators catalog: PoC 지원 지표(예: RSI, SMA/EMA, MACD, Bollinger, ATR, OBV, ADX) 문서화
5. Acceptance Criteria 표(핵심 FR 5개 우선 정의)

## 10. 템플릿(복사해서 쓸 수 있는 마크다운 템플릿 예시)

`spec/01-requirements.md` 시작 템플릿:

```
# Requirements (v0.1)

## 개요
- 목적:

## 기능 요구사항 (FR)
- FR-001: [Short description]
  - 상세:
  - 우선순위: High/Medium/Low
  - Acceptance Criteria:

## 비기능 요구사항 (NFR)
- NFR-001: 응답시간

```

## 11. 결론 (요약)

* **문서 형식**: Markdown 기반 GitHub `spec/` 권장. 초기 민첩성 확보 후 필요하면 `spec-kit` 또는 다른 도구로 마이그레이션.
* **지금 작업 우선순위**: `spec/00-readme.md`, `spec/01-requirements.md`, `spec/02-api-spec.md`, `spec/05-indicators-catalog.md`를 먼저 채우기.

---


