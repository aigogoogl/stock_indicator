# UI/UX Specification (v0.1)

## 개요

본 문서는 TradingView 지표 기반 종목 추천 서비스의 사용자 인터페이스 및 사용자 경험을 정의합니다.

**디자인 원칙**:
- **단순성**: 복잡한 지표 개념을 직관적으로 표현
- **반응성**: 모바일/태블릿 지원
- **성능**: 빠른 로딩 및 인터랙션
- **접근성**: WCAG 2.1 AA 레벨 준수

---

## 화면 목록

| 화면명 | 경로 | 우선순위 | 설명 |
|--------|------|---------|------|
| Dashboard | `/` | HIGH | 메인 대시보드, 인기 지표 및 최근 스캔 결과 |
| Indicator Selector | `/scan` | HIGH | 지표 선택 및 파라미터 설정 |
| Scan Results | `/scan/results/:scanId` | HIGH | 스캔 결과 리스트 |
| Ticker Detail | `/ticker/:symbol` | MEDIUM | 종목 상세 정보 및 차트 |
| Backtest View | `/backtest` | MEDIUM | 백테스트 결과 및 성과 분석 |
| Indicator Ranking | `/indicators/ranking` | MEDIUM | 지표 성과 랭킹 |
| Saved Filters | `/filters` | LOW | 사용자 저장 필터 관리 |
| Settings | `/settings` | LOW | 사용자 설정 |

---

## 화면별 상세 설계

### 1. Dashboard (메인 대시보드)

**경로**: `/`
**인증**: 불필요 (Public)

**레이아웃**:
```
┌──────────────────────────────────────────────┐
│  Header (Logo, Menu, Login)                  │
├──────────────────────────────────────────────┤
│  Hero Section                                │
│  "지표 기반 종목 발굴 플랫폼"                   │
│  [스캔 시작하기 버튼]                          │
├──────────────────────────────────────────────┤
│  Popular Indicators (카드 3개)                │
│  [RSI Oversold] [MACD Golden] [Bollinger]    │
├──────────────────────────────────────────────┤
│  Recent Top Performers (테이블)               │
│  Ticker | Name | Score | Indicator | ...     │
├──────────────────────────────────────────────┤
│  Footer (Links, Social, Copyright)           │
└──────────────────────────────────────────────┘
```

**컴포넌트**:
- `Header`: 로고, 네비게이션 메뉴, 로그인/회원가입 버튼
- `HeroSection`: 서비스 소개 및 CTA 버튼
- `PopularIndicatorCard`: 인기 지표 카드 (3개)
  - 지표명, 간단한 설명, "사용해보기" 버튼
- `TopPerformersTable`: 최근 24시간 상위 종목 테이블
  - 컬럼: 티커, 이름, 점수, 지표, 종가, 등락률

**상호작용**:
- "스캔 시작하기" 클릭 → `/scan` 이동
- 인기 지표 카드 클릭 → `/scan?preset=rsi_oversold` (파라미터 자동 입력)
- 종목 클릭 → `/ticker/:symbol` 이동

---

### 2. Indicator Selector (지표 선택)

**경로**: `/scan`
**인증**: 불필요 (Free tier 제한 있음)

**레이아웃**:
```
┌────────────────────────────────────────────────────┐
│  Header                                            │
├────────────────┬───────────────────────────────────┤
│  Indicator     │  Parameters Panel                 │
│  List (왼쪽)    │  (오른쪽)                          │
│                │                                   │
│  [x] RSI       │  ┌─ RSI Parameters ───────────┐  │
│  [ ] MACD      │  │ Period: [14]       [?]     │  │
│  [ ] SMA       │  │ Condition: [< 30]  [?]     │  │
│  [ ] Bollinger │  │                            │  │
│  ...           │  │ Presets:                   │  │
│                │  │ [Oversold (30)]            │  │
│                │  │ [Overbought (70)]          │  │
│                │  └────────────────────────────┘  │
│                │                                   │
│                │  Universe: [S&P 500 ▼]            │
│                │  Date: [2025-11-20]               │
│                │                                   │
│                │  [스캔 실행]                       │
└────────────────┴───────────────────────────────────┘
```

**컴포넌트**:
- `IndicatorList`: 지표 목록 (카테고리별 그룹)
  - 검색 필터
  - 카테고리 필터 (Momentum, Trend, Volatility, Volume)
- `ParameterPanel`: 선택한 지표의 파라미터 입력 폼
  - 파라미터 입력 필드 (숫자, 선택)
  - 툴팁 (?) 아이콘으로 설명 제공
  - Preset 버튼 (인기 설정 1-클릭 적용)
- `UniverseSelector`: 종목군 선택 드롭다운
- `DatePicker`: 기준 날짜 선택
- `ScanButton`: 스캔 실행 버튼 (로딩 상태 표시)

**상호작용**:
- 지표 선택 → 오른쪽 패널에 파라미터 폼 표시
- Preset 클릭 → 파라미터 자동 입력
- 파라미터 입력 시 실시간 유효성 검사
- "스캔 실행" 클릭 → API 호출 → `/scan/results/:scanId` 이동

**유효성 검사**:
- 필수 파라미터 누락 시 에러 메시지
- 범위 초과 시 경고 메시지

---

### 3. Scan Results (스캔 결과)

**경로**: `/scan/results/:scanId`
**인증**: 불필요

**레이아웃**:
```
┌──────────────────────────────────────────────┐
│  Header                                      │
├──────────────────────────────────────────────┤
│  Scan Info Bar                               │
│  RSI (period=14) < 30 | S&P 500 | 2025-11-20│
│  45 matches / 503 scanned                    │
├──────────────────────────────────────────────┤
│  Filters & Sort                              │
│  Sector: [All ▼]  Market Cap: [Any ▼]       │
│  Sort by: [Score ▼] [Price ▼] [Volume ▼]    │
├──────────────────────────────────────────────┤
│  Results Table                               │
│  Rank | Ticker | Name | Score | Indicator    │
│   1   | AAPL   | Apple| 95.2  | RSI: 28.3    │
│   2   | TSLA   | Tesla| 92.8  | RSI: 29.1    │
│   ...                                        │
├──────────────────────────────────────────────┤
│  Pagination [< 1 2 3 ... >]                  │
└──────────────────────────────────────────────┘
```

**컴포넌트**:
- `ScanInfoBar`: 스캔 조건 요약
  - 지표명, 파라미터, 조건, 종목군, 날짜
  - 매칭 종목 수 / 전체 스캔 종목 수
- `FilterBar`: 결과 필터링
  - 섹터, 시가총액, 가격 범위, 거래량
- `SortSelector`: 정렬 기준 선택
- `ResultsTable`: 결과 테이블
  - 컬럼: Rank, Ticker, Name, Score, Indicator Value, Close, Volume, Reason
  - 행 클릭 → `/ticker/:symbol` 이동
- `Pagination`: 페이지네이션 (100개씩)

**상호작용**:
- 필터 변경 → API 재호출 (쿼리 파라미터 업데이트)
- 정렬 변경 → 클라이언트 사이드 정렬 또는 API 재호출
- 종목 행 클릭 → 종목 상세 페이지 이동
- "새 스캔" 버튼 → `/scan` 이동

**성능 최적화**:
- 가상 스크롤 (react-window)
- 캐싱 (1시간)

---

### 4. Ticker Detail (종목 상세)

**경로**: `/ticker/:symbol`
**인증**: 불필요

**레이아웃**:
```
┌──────────────────────────────────────────────┐
│  Header                                      │
├──────────────────────────────────────────────┤
│  Ticker Header                               │
│  AAPL - Apple Inc. | $178.50 (+2.3%)        │
├──────────────────────────────────────────────┤
│  Chart (OHLCV + Indicators)                  │
│  [Lightweight Charts]                        │
│  - OHLCV Candlestick                         │
│  - RSI Overlay                               │
│  - Volume Bars                               │
├──────────────────────────────────────────────┤
│  Indicator Values (현재)                     │
│  RSI(14): 62.3 | MACD: 1.23 | SMA(20): 175   │
├──────────────────────────────────────────────┤
│  Company Info                                │
│  Sector: Technology | Market Cap: $2.8T     │
└──────────────────────────────────────────────┘
```

**컴포넌트**:
- `TickerHeader`: 티커, 이름, 현재가, 등락률
- `Chart`: 가격 차트 (Lightweight Charts 또는 TradingView Widget)
  - 시간대 선택: 1D, 1W, 1M, 3M, 1Y, ALL
  - 지표 오버레이 토글
- `IndicatorValuesCard`: 현재 지표 값 표시
- `CompanyInfoCard`: 기업 정보

**상호작용**:
- 시간대 선택 → API 호출하여 데이터 로드
- 지표 오버레이 토글 → 차트에 지표 추가/제거

---

### 5. Backtest View (백테스트)

**경로**: `/backtest`
**인증**: 선택적 (Free tier 제한)

**레이아웃**:
```
┌──────────────────────────────────────────────┐
│  Header                                      │
├──────────────────────────────────────────────┤
│  Backtest Config (왼쪽 사이드바)              │
│  Indicator: [RSI ▼]                          │
│  Parameters: [period: 14]                    │
│  Condition: [< 30]                           │
│  Period: [1y ▼]                              │
│  [실행]                                       │
├──────────────────────────────────────────────┤
│  Results Panel (오른쪽)                       │
│  ┌─ Metrics ──────────────────┐             │
│  │ Win Rate: 62.3%            │             │
│  │ CAGR: 23.4%                │             │
│  │ Sharpe: 1.42               │             │
│  │ Max DD: -15.6%             │             │
│  └────────────────────────────┘             │
│                                              │
│  Equity Curve Chart                          │
│  [Line Chart]                                │
│                                              │
│  Trades Table (최근 10개)                    │
└──────────────────────────────────────────────┘
```

**컴포넌트**:
- `BacktestConfigPanel`: 백테스트 설정
- `MetricsCard`: 주요 메트릭 요약
- `EquityCurveChart`: 자산 곡선 차트
- `TradesTable`: 거래 이력 테이블

---

### 6. Indicator Ranking (지표 랭킹)

**경로**: `/indicators/ranking`
**인증**: 불필요

**레이아웃**:
```
┌──────────────────────────────────────────────┐
│  Header                                      │
├──────────────────────────────────────────────┤
│  Ranking Controls                            │
│  Period: [1y ▼]  Metric: [Sharpe Ratio ▼]   │
├──────────────────────────────────────────────┤
│  Ranking Table                               │
│  Rank | Indicator | Preset | Sharpe | CAGR  │
│   1   | RSI       | <30    | 1.68   | 28%   │
│   2   | MACD      | Golden | 1.54   | 24%   │
│   ...                                        │
└──────────────────────────────────────────────┘
```

---

## 디자인 시스템

### 색상 팔레트

| 용도 | 색상 | Hex Code |
|------|------|----------|
| Primary | Blue | `#2563EB` |
| Secondary | Gray | `#64748B` |
| Success | Green | `#10B981` |
| Warning | Yellow | `#F59E0B` |
| Danger | Red | `#EF4444` |
| Background | White | `#FFFFFF` |
| Surface | Light Gray | `#F8FAFC` |
| Text | Dark Gray | `#1E293B` |

---

### 타이포그래피

| 용도 | 폰트 | 크기 | 굵기 |
|------|------|------|------|
| H1 | Inter | 32px | 700 |
| H2 | Inter | 24px | 600 |
| H3 | Inter | 20px | 600 |
| Body | Inter | 16px | 400 |
| Small | Inter | 14px | 400 |
| Code | Fira Code | 14px | 400 |

---

### 컴포넌트 라이브러리

**추천**: Tailwind CSS + Shadcn UI
**대안**: Material-UI, Ant Design

---

## 반응형 디자인

### 브레이크포인트

| 디바이스 | 최소 너비 |
|---------|----------|
| Mobile | 320px |
| Tablet | 768px |
| Desktop | 1024px |
| Large Desktop | 1440px |

### 레이아웃 조정
- Mobile: 단일 컬럼, 햄버거 메뉴
- Tablet: 2컬럼, 사이드 메뉴
- Desktop: 3컬럼, 고정 사이드바

---

## 접근성 (Accessibility)

- **키보드 네비게이션**: Tab, Enter, Arrow keys 지원
- **스크린 리더**: ARIA 레이블 및 역할
- **색상 대비**: WCAG AA 레벨 (4.5:1)
- **포커스 표시**: 명확한 아웃라인

---

## 성능 최적화

- **Code Splitting**: 라우트별 번들 분리
- **Lazy Loading**: 이미지 및 컴포넌트
- **Memoization**: React.memo, useMemo
- **Virtual Scrolling**: 긴 리스트 렌더링 최적화

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 0.1  | 2025-11-21 | 초안 작성 | Design Team |

---

**다음 단계**: Figma 목업 작성 및 사용자 테스트
