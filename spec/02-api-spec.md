# API Specification (v0.1)

## 개요

본 문서는 TradingView 지표 기반 종목 추천 서비스의 REST API 명세를 정의합니다.

**Base URL**:
- Development: `http://localhost:8000/api/v1`
- Staging: `https://staging-api.stockindicator.com/api/v1`
- Production: `https://api.stockindicator.com/api/v1`

**API 버전**: v1
**인증 방식**: Bearer Token (JWT) - 일부 Public API는 인증 불필요
**Content-Type**: `application/json`

---

## 공통 사항

### 인증 헤더
```
Authorization: Bearer <JWT_TOKEN>
```

### 공통 응답 형식

#### 성공 응답
```json
{
  "success": true,
  "data": { ... },
  "metadata": {
    "timestamp": "2025-11-21T10:30:00Z",
    "api_version": "v1"
  }
}
```

#### 오류 응답
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": { ... }
  },
  "metadata": {
    "timestamp": "2025-11-21T10:30:00Z",
    "api_version": "v1"
  }
}
```

### 표준 에러 코드

| 코드 | HTTP 상태 | 설명 |
|------|----------|------|
| `BAD_REQUEST` | 400 | 잘못된 요청 파라미터 |
| `UNAUTHORIZED` | 401 | 인증 토큰 없음 또는 만료 |
| `FORBIDDEN` | 403 | 권한 없음 |
| `NOT_FOUND` | 404 | 리소스를 찾을 수 없음 |
| `RATE_LIMITED` | 429 | 요청 횟수 제한 초과 |
| `DATA_NOT_FOUND` | 404 | 요청한 날짜의 데이터 없음 |
| `INVALID_INDICATOR` | 400 | 지원하지 않는 지표 |
| `INVALID_PARAMETER` | 400 | 지표 파라미터 유효성 검사 실패 |
| `INTERNAL_ERROR` | 500 | 서버 내부 오류 |

---

## API 엔드포인트

### 1. Health Check

#### `GET /health`

**설명**: 서비스 헬스 체크 (인증 불필요)

**Request**: 없음

**Response 200 OK**:
```json
{
  "status": "healthy",
  "timestamp": "2025-11-21T10:30:00Z",
  "services": {
    "database": "up",
    "cache": "up",
    "etl": "up"
  }
}
```

---

### 2. 지표 목록 조회

#### `GET /indicators`

**설명**: 지원하는 모든 지표의 목록과 메타데이터 조회 (인증 불필요)

**Query Parameters**:
- `category` (optional): 지표 카테고리 필터 (`momentum`, `trend`, `volatility`, `volume`)

**Request 예시**:
```
GET /api/v1/indicators?category=momentum
```

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "indicators": [
      {
        "id": "rsi",
        "name": "Relative Strength Index",
        "category": "momentum",
        "description": "모멘텀 오실레이터로 과매수/과매도 구간 파악",
        "parameters": [
          {
            "name": "period",
            "type": "integer",
            "default": 14,
            "min": 2,
            "max": 100,
            "description": "RSI 계산 기간"
          }
        ],
        "presets": [
          {
            "name": "Oversold (30)",
            "description": "RSI 30 이하 (과매도)",
            "parameters": { "period": 14 },
            "condition": { "operator": "lt", "value": 30 }
          },
          {
            "name": "Overbought (70)",
            "description": "RSI 70 이상 (과매수)",
            "parameters": { "period": 14 },
            "condition": { "operator": "gt", "value": 70 }
          }
        ],
        "pine_compatible": true
      },
      {
        "id": "macd",
        "name": "Moving Average Convergence Divergence",
        "category": "momentum",
        "description": "추세 추종 모멘텀 지표",
        "parameters": [
          {
            "name": "fast_period",
            "type": "integer",
            "default": 12,
            "min": 2,
            "max": 100,
            "description": "빠른 EMA 기간"
          },
          {
            "name": "slow_period",
            "type": "integer",
            "default": 26,
            "min": 2,
            "max": 100,
            "description": "느린 EMA 기간"
          },
          {
            "name": "signal_period",
            "type": "integer",
            "default": 9,
            "min": 2,
            "max": 100,
            "description": "시그널 라인 기간"
          }
        ],
        "presets": [
          {
            "name": "Golden Cross",
            "description": "MACD가 시그널을 상향 돌파",
            "parameters": { "fast_period": 12, "slow_period": 26, "signal_period": 9 },
            "condition": { "operator": "cross_above", "value": "signal" }
          }
        ],
        "pine_compatible": true
      }
    ]
  },
  "metadata": {
    "total": 2,
    "timestamp": "2025-11-21T10:30:00Z",
    "api_version": "v1"
  }
}
```

**Error 400 BAD_REQUEST**:
```json
{
  "success": false,
  "error": {
    "code": "BAD_REQUEST",
    "message": "Invalid category. Allowed values: momentum, trend, volatility, volume"
  }
}
```

---

### 3. 종목 스캔

#### `POST /scan`

**설명**: 지정한 지표와 조건으로 종목 스캔 (인증 필요)

**Request Body**:
```json
{
  "indicator": "rsi",
  "parameters": {
    "period": 14
  },
  "condition": {
    "operator": "lt",
    "value": 30
  },
  "universe": "sp500",
  "date": "2025-11-20",
  "limit": 100,
  "offset": 0,
  "sort_by": "score",
  "sort_order": "desc"
}
```

**Request 필드 설명**:
- `indicator` (required, string): 지표 ID (`rsi`, `macd`, `sma`, etc.)
- `parameters` (required, object): 지표별 파라미터
- `condition` (required, object): 조건
  - `operator`: `lt`, `gt`, `eq`, `lte`, `gte`, `cross_above`, `cross_below`
  - `value`: 비교 값 또는 다른 지표명 (예: `"signal"`)
- `universe` (required, string): 종목군 (`sp500`, `nasdaq100`, `kospi`, `kosdaq`, `all_us`, `all_kr`)
- `date` (optional, string, YYYY-MM-DD): 기준 날짜 (기본값: 전일)
- `limit` (optional, integer, default=100): 결과 개수
- `offset` (optional, integer, default=0): 페이지네이션 offset
- `sort_by` (optional, string, default=score): 정렬 기준 (`score`, `indicator_value`, `volume`, `market_cap`)
- `sort_order` (optional, string, default=desc): 정렬 순서 (`asc`, `desc`)

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "scan_id": "scan_20251121_103000_abc123",
    "date": "2025-11-20",
    "indicator": "rsi",
    "parameters": { "period": 14 },
    "condition": { "operator": "lt", "value": 30 },
    "results": [
      {
        "rank": 1,
        "ticker": "AAPL",
        "name": "Apple Inc.",
        "exchange": "NASDAQ",
        "sector": "Technology",
        "market_cap": 2800000000000,
        "close_price": 178.50,
        "volume": 52000000,
        "indicator_value": 28.3,
        "score": 95.2,
        "reason": "RSI 28.3 (과매도 구간, 반등 가능성 높음)"
      },
      {
        "rank": 2,
        "ticker": "TSLA",
        "name": "Tesla Inc.",
        "exchange": "NASDAQ",
        "sector": "Consumer Cyclical",
        "market_cap": 780000000000,
        "close_price": 242.80,
        "volume": 95000000,
        "indicator_value": 29.1,
        "score": 92.8,
        "reason": "RSI 29.1 (과매도 구간)"
      }
    ],
    "total_matches": 45,
    "total_scanned": 503
  },
  "metadata": {
    "cached": true,
    "cache_ttl": 3600,
    "execution_time_ms": 120,
    "timestamp": "2025-11-21T10:30:00Z",
    "api_version": "v1"
  }
}
```

**Error 400 BAD_REQUEST**:
```json
{
  "success": false,
  "error": {
    "code": "INVALID_INDICATOR",
    "message": "Indicator 'xyz' is not supported",
    "details": {
      "indicator": "xyz",
      "supported_indicators": ["rsi", "macd", "sma", "ema", "bollinger"]
    }
  }
}
```

**Error 404 DATA_NOT_FOUND**:
```json
{
  "success": false,
  "error": {
    "code": "DATA_NOT_FOUND",
    "message": "No data available for date 2025-11-20",
    "details": {
      "requested_date": "2025-11-20",
      "latest_available_date": "2025-11-19"
    }
  }
}
```

---

### 4. 백테스트 조회

#### `GET /backtest`

**설명**: 지표 및 조건의 백테스트 성과 조회 (인증 불필요)

**Query Parameters**:
- `indicator` (required): 지표 ID
- `parameters` (required, JSON string): 지표 파라미터 (URL-encoded JSON)
- `condition` (required, JSON string): 조건 (URL-encoded JSON)
- `period` (optional, default=1y): 백테스트 기간 (`1d`, `1w`, `1m`, `3m`, `6m`, `1y`, `3y`)
- `universe` (optional, default=sp500): 종목군

**Request 예시**:
```
GET /api/v1/backtest?indicator=rsi&parameters=%7B%22period%22%3A14%7D&condition=%7B%22operator%22%3A%22lt%22%2C%22value%22%3A30%7D&period=1y&universe=sp500
```

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "indicator": "rsi",
    "parameters": { "period": 14 },
    "condition": { "operator": "lt", "value": 30 },
    "period": "1y",
    "universe": "sp500",
    "start_date": "2024-11-21",
    "end_date": "2025-11-20",
    "metrics": {
      "total_signals": 342,
      "win_rate": 0.623,
      "avg_return": 0.0347,
      "avg_return_per_trade": 0.0127,
      "sharpe_ratio": 1.42,
      "max_drawdown": -0.156,
      "cagr": 0.234,
      "total_return": 0.213,
      "volatility": 0.187
    },
    "performance_by_sector": [
      {
        "sector": "Technology",
        "win_rate": 0.68,
        "avg_return": 0.042
      },
      {
        "sector": "Healthcare",
        "win_rate": 0.61,
        "avg_return": 0.031
      }
    ],
    "equity_curve": [
      { "date": "2024-11-21", "value": 10000 },
      { "date": "2024-11-22", "value": 10050 },
      { "date": "2024-11-25", "value": 10120 }
    ]
  },
  "metadata": {
    "timestamp": "2025-11-21T10:30:00Z",
    "api_version": "v1"
  }
}
```

---

### 5. 종목 상세 정보

#### `GET /ticker/{symbol}`

**설명**: 특정 종목의 OHLCV 데이터 및 지표 값 조회 (인증 불필요)

**Path Parameters**:
- `symbol` (required): 티커 심볼 (예: `AAPL`)

**Query Parameters**:
- `start_date` (optional, YYYY-MM-DD): 시작일 (기본값: 1년 전)
- `end_date` (optional, YYYY-MM-DD): 종료일 (기본값: 오늘)
- `indicators` (optional, comma-separated): 포함할 지표 목록 (예: `rsi,macd,sma_20`)

**Request 예시**:
```
GET /api/v1/ticker/AAPL?start_date=2025-10-01&end_date=2025-11-20&indicators=rsi,macd
```

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "symbol": "AAPL",
    "name": "Apple Inc.",
    "exchange": "NASDAQ",
    "sector": "Technology",
    "market_cap": 2800000000000,
    "ohlcv": [
      {
        "date": "2025-10-01",
        "open": 175.20,
        "high": 178.50,
        "low": 174.80,
        "close": 177.90,
        "volume": 48000000,
        "adjusted_close": 177.90
      },
      {
        "date": "2025-10-02",
        "open": 178.00,
        "high": 180.10,
        "low": 177.50,
        "close": 179.30,
        "volume": 52000000,
        "adjusted_close": 179.30
      }
    ],
    "indicators": {
      "rsi": [
        { "date": "2025-10-01", "value": 62.3 },
        { "date": "2025-10-02", "value": 64.1 }
      ],
      "macd": [
        {
          "date": "2025-10-01",
          "macd": 1.23,
          "signal": 1.15,
          "histogram": 0.08
        },
        {
          "date": "2025-10-02",
          "macd": 1.35,
          "signal": 1.20,
          "histogram": 0.15
        }
      ]
    }
  },
  "metadata": {
    "total_days": 35,
    "timestamp": "2025-11-21T10:30:00Z",
    "api_version": "v1"
  }
}
```

**Error 404 NOT_FOUND**:
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Ticker 'ABCD' not found"
  }
}
```

---

### 6. 인기 지표 랭킹

#### `GET /indicators/ranking`

**설명**: 백테스트 성과 기준 상위 지표 랭킹 조회 (인증 불필요)

**Query Parameters**:
- `period` (optional, default=1y): 백테스트 기간
- `metric` (optional, default=sharpe_ratio): 랭킹 기준 메트릭 (`win_rate`, `avg_return`, `sharpe_ratio`, `cagr`)
- `limit` (optional, default=10): 결과 개수

**Request 예시**:
```
GET /api/v1/indicators/ranking?period=1y&metric=sharpe_ratio&limit=10
```

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "period": "1y",
    "metric": "sharpe_ratio",
    "ranking": [
      {
        "rank": 1,
        "indicator": "rsi",
        "preset_name": "Oversold (30)",
        "parameters": { "period": 14 },
        "condition": { "operator": "lt", "value": 30 },
        "sharpe_ratio": 1.68,
        "win_rate": 0.65,
        "cagr": 0.28
      },
      {
        "rank": 2,
        "indicator": "macd",
        "preset_name": "Golden Cross",
        "parameters": { "fast_period": 12, "slow_period": 26, "signal_period": 9 },
        "condition": { "operator": "cross_above", "value": "signal" },
        "sharpe_ratio": 1.54,
        "win_rate": 0.61,
        "cagr": 0.24
      }
    ]
  },
  "metadata": {
    "timestamp": "2025-11-21T10:30:00Z",
    "api_version": "v1"
  }
}
```

---

### 7. 사용자 필터 저장 (향후 기능)

#### `POST /filters`

**설명**: 사용자의 스캔 필터 저장 (인증 필요)

**Request Body**:
```json
{
  "name": "My RSI Strategy",
  "description": "RSI 과매도 종목 스캔",
  "indicator": "rsi",
  "parameters": { "period": 14 },
  "condition": { "operator": "lt", "value": 30 },
  "universe": "sp500"
}
```

**Response 201 CREATED**:
```json
{
  "success": true,
  "data": {
    "filter_id": "filter_abc123",
    "name": "My RSI Strategy",
    "created_at": "2025-11-21T10:30:00Z",
    "share_url": "https://stockindicator.com/filters/filter_abc123"
  }
}
```

---

## Rate Limiting

- **비인증 사용자**: 시간당 100 requests
- **인증 사용자 (Free)**: 시간당 500 requests
- **인증 사용자 (Premium)**: 시간당 5000 requests

**Rate Limit 헤더**:
```
X-RateLimit-Limit: 500
X-RateLimit-Remaining: 487
X-RateLimit-Reset: 1700568000
```

**429 Too Many Requests**:
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Try again in 3600 seconds.",
    "details": {
      "limit": 500,
      "remaining": 0,
      "reset_at": "2025-11-21T12:00:00Z"
    }
  }
}
```

---

## OpenAPI 3.0 스키마

완전한 OpenAPI 스펙은 `spec/openapi.yaml` 참조.

스펙 다운로드: `GET /api/v1/openapi.yaml`

Swagger UI: `https://api.stockindicator.com/docs`

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 0.1  | 2025-11-21 | 초안 작성 | API Team |

---

**다음 단계**: `spec/openapi.yaml` 파일 생성 및 Swagger UI 통합
