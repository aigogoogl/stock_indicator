# TradingView 지표 기반 종목 추천 서비스 - SDD (v0.1)

## 문서 개요

본 디렉토리는 **TradingView 지표 기반 종목 추천 서비스**의 Specification-Driven Development(SDD) 문서를 포함합니다.

### 프로젝트 목적
사용자가 웹 UI에서 기술적 지표(Technical Indicators)와 파라미터를 선택하면, 지정된 날짜(기본: 전일 종가)를 기준으로 전체 또는 선택된 종목군에서 지표 조건을 충족하는 종목을 점수화·정렬하여 제공하는 서비스입니다.

### 핵심 기능
- **지표 기반 스캔**: RSI, MACD, SMA/EMA, Bollinger Bands 등 다양한 지표로 종목 스캔
- **과거 날짜 기준 분석**: 전일 또는 사용자 지정 날짜의 종가 기준 분석
- **백테스트**: 지표별 과거 성능 검증 및 통계 제공
- **사용자 커스터마이징**: 지표 파라미터 조정, 필터 저장, 알림 설정(향후)

## 문서 구조

| 파일명 | 설명 | 우선순위 |
|--------|------|---------|
| `00-readme.md` | SDD 요약 및 문서 가이드 (현재 문서) | HIGH |
| `01-requirements.md` | 기능/비기능 요구사항 (FR/NFR) | HIGH |
| `02-api-spec.md` | REST API 엔드포인트 명세 (OpenAPI) | HIGH |
| `03-data-model.md` | DB 스키마, 테이블, 인덱스 설계 | HIGH |
| `04-etl-spec.md` | 데이터 수집/변환/적재 파이프라인 | HIGH |
| `05-indicators-catalog.md` | 지원 지표 목록 및 파라미터 설명 | HIGH |
| `06-backtest-spec.md` | 백테스트 방법론 및 메트릭 정의 | MEDIUM |
| `07-ui-ux-spec.md` | 화면 플로우 및 컴포넌트 명세 | MEDIUM |
| `08-ci-cd-tests.md` | CI/CD 파이프라인 및 테스트 전략 | MEDIUM |
| `09-security-ops.md` | 보안, 인증, 권한, 라이선스 관리 | MEDIUM |
| `10-deployment.md` | 인프라 구성 및 배포 절차 | LOW |
| `11-acceptance-criteria.md` | 각 FR별 수용 기준 및 테스트 매핑 | HIGH |
| `assets/` | ERD, 다이어그램, 목업 이미지 | - |

## 버전 관리

- **현재 버전**: v0.1 (초안)
- **변경 이력**:
  - 2025-11-21: v0.1 - 초기 SDD 문서 생성

## 문서 변경 프로세스

1. **변경 제안**: GitHub Issue 또는 Discussion에 변경 사유와 영향도 기술
2. **PR 생성**: `spec/` 하위 파일 수정 후 PR 생성
3. **리뷰**: 최소 2명 승인 필요 (제품 소유자 + 기술 리드 또는 QA)
4. **버전 업데이트**: Major 변경 시 버전 번호 증가
5. **Merge**: Squash merge 권장, CHANGELOG 업데이트

### 승인자 (CODEOWNERS)
- 제품 소유자: 기능 요구사항 변경 승인
- 데이터 엔지니어: ETL, 데이터 모델 변경 승인
- 백엔드 리드: API 스펙 변경 승인
- QA 리드: Acceptance Criteria 변경 승인

## 개발 워크플로우

```
1. 요구사항 정의 (spec/01-requirements.md 업데이트)
   ↓
2. Acceptance Criteria 작성 (spec/11-acceptance-criteria.md)
   ↓
3. 테스트 케이스 작성 (TDD)
   ↓
4. 구현 (feature/<FR-ID>-description 브랜치)
   ↓
5. CI 파이프라인 실행 (lint → unit → contract → E2E)
   ↓
6. Code Review + Merge
   ↓
7. Staging 배포 + Acceptance Test
   ↓
8. Production 배포
```

## 기여 가이드

### 새로운 기능 제안 시
1. `spec/01-requirements.md`에 FR-XXX 번호 부여
2. `spec/11-acceptance-criteria.md`에 수용 기준 추가
3. 관련 API 변경이 있다면 `spec/02-api-spec.md` 업데이트
4. 새로운 지표 추가 시 `spec/05-indicators-catalog.md` 업데이트

### 문서 작성 원칙
- **ID 부여**: 모든 요구사항, API, 테스트 케이스에 고유 ID 사용
- **명확성**: 모호한 표현 지양, 구체적인 예시 포함
- **추적 가능성**: FR-ID ↔ Test-ID ↔ API-ID 매핑 유지
- **버전 관리**: 주요 변경 시 문서 상단 버전 및 날짜 갱신

## 로드맵

### Phase 1: PoC (v0.1) - 2주
- [ ] 기본 지표 5개 지원 (RSI, SMA, MACD, Bollinger, Volume)
- [ ] 단순 스캔 API 1개 (`POST /api/v1/scan`)
- [ ] yfinance 기반 EOD 데이터 수집
- [ ] 정적 HTML/JS 기반 간단한 UI

### Phase 2: MVP (v0.2) - 4주
- [ ] 지표 10개 이상 확장
- [ ] 백테스트 기능 추가
- [ ] 사용자 인증 (OAuth2)
- [ ] React 기반 UI 개선
- [ ] PostgreSQL + TimescaleDB 적용

### Phase 3: Production (v1.0) - 8주
- [ ] 실시간 알림 기능
- [ ] 커스텀 Pine 스크립트 지원 (제한적)
- [ ] 프리미엄 데이터 소스 연동 (Polygon, Tiingo)
- [ ] K8s 기반 Auto-scaling
- [ ] 모니터링 및 알럿 시스템 구축

## 참고 자료

- [TradingView Pine Script Documentation](https://www.tradingview.com/pine-script-docs/)
- [TA-Lib: Technical Analysis Library](https://ta-lib.org/)
- [OpenAPI 3.0 Specification](https://swagger.io/specification/)
- [Acceptance Test-Driven Development](https://en.wikipedia.org/wiki/Acceptance_test-driven_development)

## 연락처 및 지원

- **프로젝트 관리자**: [Product Owner Email]
- **기술 리드**: [Tech Lead Email]
- **이슈 트래킹**: GitHub Issues
- **문서 관련 질문**: GitHub Discussions

---

**마지막 업데이트**: 2025-11-21
**문서 버전**: v0.1
**관리자**: Product Team
