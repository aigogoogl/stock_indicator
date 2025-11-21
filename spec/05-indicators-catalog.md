# Indicators Catalog (v0.1)

## 개요

본 문서는 TradingView 지표 기반 종목 추천 서비스에서 지원하는 기술적 지표(Technical Indicators)의 목록과 상세 사양을 정의합니다.

**지원 지표 수**:
- **PoC**: 5개
- **MVP**: 15개
- **Production**: 25개 이상

---

## 지표 카테고리

| 카테고리 | 설명 | 예시 지표 |
|---------|------|----------|
| **Momentum** | 가격 변화의 속도와 강도 측정 | RSI, MACD, Stochastic |
| **Trend** | 추세의 방향과 강도 파악 | SMA, EMA, ADX |
| **Volatility** | 가격 변동성 측정 | Bollinger Bands, ATR |
| **Volume** | 거래량 기반 지표 | OBV, Volume Profile |

---

## 지원 지표 목록

### 1. RSI (Relative Strength Index) - Momentum

**설명**: 일정 기간 동안의 가격 상승폭과 하락폭의 비율로 과매수/과매도 구간을 파악하는 모멘텀 오실레이터

**개발자**: J. Welles Wilder (1978)

**파라미터**:
| 파라미터 | 타입 | 기본값 | 범위 | 설명 |
|---------|------|--------|------|------|
| `period` | integer | 14 | 2-100 | RSI 계산 기간 |

**계산 공식**:
```
RS = (N일간 상승폭 평균) / (N일간 하락폭 평균)
RSI = 100 - (100 / (1 + RS))
```

**해석**:
- RSI > 70: 과매수 구간 (Overbought)
- RSI < 30: 과매도 구간 (Oversold)
- RSI = 50: 중립

**Presets**:
```json
[
  {
    "name": "Oversold (30)",
    "description": "RSI 30 이하 - 과매도 구간, 반등 가능성",
    "parameters": { "period": 14 },
    "condition": { "operator": "lt", "value": 30 }
  },
  {
    "name": "Overbought (70)",
    "description": "RSI 70 이상 - 과매수 구간, 조정 가능성",
    "parameters": { "period": 14 },
    "condition": { "operator": "gt", "value": 70 }
  },
  {
    "name": "Extreme Oversold (20)",
    "description": "RSI 20 이하 - 극심한 과매도",
    "parameters": { "period": 14 },
    "condition": { "operator": "lt", "value": 20 }
  }
]
```

**Pine Script 호환**: ✅ Yes
**TA-Lib 함수**: `talib.RSI(close, timeperiod=14)`

**제약사항**:
- 최소 `period + 1`일 데이터 필요
- 추세장에서는 과매수/과매도가 오래 지속될 수 있음 (Divergence 분석 필요)

---

### 2. MACD (Moving Average Convergence Divergence) - Momentum

**설명**: 단기 및 장기 지수이동평균(EMA)의 차이를 이용한 추세 추종 모멘텀 지표

**개발자**: Gerald Appel (1970년대)

**파라미터**:
| 파라미터 | 타입 | 기본값 | 범위 | 설명 |
|---------|------|--------|------|------|
| `fast_period` | integer | 12 | 2-100 | 빠른 EMA 기간 |
| `slow_period` | integer | 26 | 2-200 | 느린 EMA 기간 |
| `signal_period` | integer | 9 | 2-50 | 시그널 라인 기간 |

**계산 공식**:
```
MACD Line = EMA(12) - EMA(26)
Signal Line = EMA(MACD Line, 9)
Histogram = MACD Line - Signal Line
```

**해석**:
- MACD > Signal: 상승 신호 (Golden Cross)
- MACD < Signal: 하락 신호 (Death Cross)
- Histogram > 0: 상승 모멘텀
- Histogram < 0: 하락 모멘텀

**Presets**:
```json
[
  {
    "name": "Golden Cross",
    "description": "MACD가 시그널을 상향 돌파 - 매수 신호",
    "parameters": { "fast_period": 12, "slow_period": 26, "signal_period": 9 },
    "condition": { "operator": "cross_above", "target": "signal" }
  },
  {
    "name": "Death Cross",
    "description": "MACD가 시그널을 하향 돌파 - 매도 신호",
    "parameters": { "fast_period": 12, "slow_period": 26, "signal_period": 9 },
    "condition": { "operator": "cross_below", "target": "signal" }
  },
  {
    "name": "Positive Histogram",
    "description": "히스토그램 > 0 - 상승 모멘텀",
    "parameters": { "fast_period": 12, "slow_period": 26, "signal_period": 9 },
    "condition": { "operator": "gt", "value": 0, "field": "histogram" }
  }
]
```

**Pine Script 호환**: ✅ Yes
**TA-Lib 함수**: `talib.MACD(close, fastperiod=12, slowperiod=26, signalperiod=9)`

**출력 필드**:
- `macd`: MACD Line
- `signal`: Signal Line
- `histogram`: MACD - Signal

---

### 3. SMA (Simple Moving Average) - Trend

**설명**: 일정 기간의 종가 산술 평균

**파라미터**:
| 파라미터 | 타입 | 기본값 | 범위 | 설명 |
|---------|------|--------|------|------|
| `period` | integer | 20 | 2-200 | 이동평균 기간 |

**계산 공식**:
```
SMA = (C1 + C2 + ... + Cn) / n
```

**해석**:
- Price > SMA: 상승 추세
- Price < SMA: 하락 추세

**일반적인 기간**:
- 20일: 단기 추세
- 50일: 중기 추세
- 200일: 장기 추세

**Presets**:
```json
[
  {
    "name": "Price Above SMA 20",
    "description": "종가가 20일 이동평균 위 - 단기 상승 추세",
    "parameters": { "period": 20 },
    "condition": { "operator": "gt", "target": "close" }
  },
  {
    "name": "Golden Cross (SMA 50/200)",
    "description": "SMA 50이 SMA 200을 상향 돌파 - 강력한 매수 신호",
    "parameters": { "short_period": 50, "long_period": 200 },
    "condition": { "operator": "cross_above", "target": "sma_long" }
  }
]
```

**Pine Script 호환**: ✅ Yes
**TA-Lib 함수**: `talib.SMA(close, timeperiod=20)`

---

### 4. EMA (Exponential Moving Average) - Trend

**설명**: 최근 데이터에 더 큰 가중치를 부여한 이동평균

**파라미터**:
| 파라미터 | 타입 | 기본값 | 범위 | 설명 |
|---------|------|--------|------|------|
| `period` | integer | 20 | 2-200 | 이동평균 기간 |

**계산 공식**:
```
EMA_today = (Price_today * K) + (EMA_yesterday * (1 - K))
where K = 2 / (period + 1)
```

**특징**:
- SMA보다 가격 변화에 민감하게 반응
- 단기 트레이딩에 적합

**Pine Script 호환**: ✅ Yes
**TA-Lib 함수**: `talib.EMA(close, timeperiod=20)`

---

### 5. Bollinger Bands - Volatility

**설명**: 이동평균을 중심으로 표준편차를 이용한 상하한 밴드

**개발자**: John Bollinger (1980년대)

**파라미터**:
| 파라미터 | 타입 | 기본값 | 범위 | 설명 |
|---------|------|--------|------|------|
| `period` | integer | 20 | 2-100 | 이동평균 기간 |
| `std_dev` | float | 2.0 | 0.5-5.0 | 표준편차 배수 |

**계산 공식**:
```
Middle Band = SMA(20)
Upper Band = Middle Band + (2 * Standard Deviation)
Lower Band = Middle Band - (2 * Standard Deviation)
```

**해석**:
- Price > Upper Band: 과매수 또는 강한 상승 추세
- Price < Lower Band: 과매도 또는 강한 하락 추세
- 밴드 폭 축소: 변동성 감소 (Squeeze) → 큰 움직임 임박
- 밴드 폭 확대: 변동성 증가

**Presets**:
```json
[
  {
    "name": "Price Below Lower Band",
    "description": "종가가 하단 밴드 아래 - 과매도 반등 가능",
    "parameters": { "period": 20, "std_dev": 2.0 },
    "condition": { "operator": "lt", "target": "lower_band" }
  },
  {
    "name": "Bollinger Squeeze",
    "description": "밴드 폭이 20일 평균 대비 70% 이하 - 큰 움직임 예상",
    "parameters": { "period": 20, "std_dev": 2.0 },
    "condition": { "operator": "squeeze", "threshold": 0.7 }
  }
]
```

**Pine Script 호환**: ✅ Yes
**TA-Lib 함수**: `talib.BBANDS(close, timeperiod=20, nbdevup=2, nbdevdn=2)`

**출력 필드**:
- `upper`: 상단 밴드
- `middle`: 중간 밴드 (SMA)
- `lower`: 하단 밴드

---

### 6. ATR (Average True Range) - Volatility

**설명**: 실제 가격 변동폭의 평균으로 변동성 측정

**개발자**: J. Welles Wilder

**파라미터**:
| 파라미터 | 타입 | 기본값 | 범위 | 설명 |
|---------|------|--------|------|------|
| `period` | integer | 14 | 2-100 | ATR 계산 기간 |

**계산 공식**:
```
True Range = max(High - Low, |High - Previous Close|, |Low - Previous Close|)
ATR = SMA(True Range, period)
```

**해석**:
- ATR 높음: 변동성 높음 (리스크 높음, 기회 높음)
- ATR 낮음: 변동성 낮음 (안정적)

**활용**:
- Stop Loss 설정: Entry Price ± (ATR * 2)
- Position Sizing: 변동성에 반비례하여 포지션 크기 조절

**Pine Script 호환**: ✅ Yes
**TA-Lib 함수**: `talib.ATR(high, low, close, timeperiod=14)`

---

### 7. ADX (Average Directional Index) - Trend

**설명**: 추세의 강도를 측정하는 지표 (방향성 무관)

**개발자**: J. Welles Wilder

**파라미터**:
| 파라미터 | 타입 | 기본값 | 범위 | 설명 |
|---------|------|--------|------|------|
| `period` | integer | 14 | 2-100 | ADX 계산 기간 |

**해석**:
- ADX > 25: 강한 추세
- ADX < 20: 추세 없음 (횡보)
- ADX 상승: 추세 강화
- ADX 하락: 추세 약화

**Presets**:
```json
[
  {
    "name": "Strong Trend",
    "description": "ADX > 25 - 강한 추세 진행 중",
    "parameters": { "period": 14 },
    "condition": { "operator": "gt", "value": 25 }
  }
]
```

**Pine Script 호환**: ✅ Yes
**TA-Lib 함수**: `talib.ADX(high, low, close, timeperiod=14)`

---

### 8. OBV (On-Balance Volume) - Volume

**설명**: 거래량을 이용해 자금 흐름을 파악하는 지표

**개발자**: Joseph Granville (1963)

**파라미터**: 없음

**계산 공식**:
```
If Close > Previous Close: OBV = Previous OBV + Volume
If Close < Previous Close: OBV = Previous OBV - Volume
If Close = Previous Close: OBV = Previous OBV
```

**해석**:
- OBV 상승 + 가격 상승: 상승 추세 확인
- OBV 하락 + 가격 하락: 하락 추세 확인
- OBV와 가격 Divergence: 추세 반전 신호

**Pine Script 호환**: ✅ Yes
**TA-Lib 함수**: `talib.OBV(close, volume)`

---

### 9. Stochastic Oscillator - Momentum

**설명**: 현재 종가가 일정 기간의 가격 범위에서 어디에 위치하는지 측정

**개발자**: George Lane (1950년대)

**파라미터**:
| 파라미터 | 타입 | 기본값 | 범위 | 설명 |
|---------|------|--------|------|------|
| `k_period` | integer | 14 | 2-100 | %K 기간 |
| `d_period` | integer | 3 | 2-50 | %D 기간 (%K의 SMA) |
| `smooth_k` | integer | 3 | 1-50 | %K 스무딩 |

**계산 공식**:
```
%K = (Current Close - Lowest Low) / (Highest High - Lowest Low) * 100
%D = SMA(%K, d_period)
```

**해석**:
- %K > 80: 과매수
- %K < 20: 과매도
- %K > %D: 매수 신호
- %K < %D: 매도 신호

**Pine Script 호환**: ✅ Yes
**TA-Lib 함수**: `talib.STOCH(high, low, close, fastk_period=14, slowk_period=3, slowd_period=3)`

---

### 10. Volume (거래량) - Volume

**설명**: 일별 거래량

**해석**:
- 거래량 급증 + 가격 상승: 강한 상승 신호
- 거래량 급증 + 가격 하락: 강한 하락 신호
- 거래량 감소: 추세 약화

**Presets**:
```json
[
  {
    "name": "High Volume (2x Avg)",
    "description": "거래량이 20일 평균의 2배 이상 - 이벤트 발생",
    "condition": { "operator": "gt", "value": "avg_volume_20 * 2" }
  }
]
```

---

## 향후 추가 예정 지표

### Phase 2 (MVP)
- CCI (Commodity Channel Index)
- Williams %R
- Parabolic SAR
- Ichimoku Cloud
- Fibonacci Retracement

### Phase 3 (Production)
- Custom Pine Script Support (제한적)
- Volume Profile
- Market Profile
- Order Flow Indicators

---

## 지표 조합 예시

### 1. 트렌드 확인 전략
```
조건:
- SMA(50) > SMA(200): 장기 상승 추세
- ADX > 25: 강한 추세
- RSI < 30: 과매도 (단기 반등 포착)
```

### 2. 변동성 돌파 전략
```
조건:
- Bollinger Squeeze 감지
- ATR > 20일 평균의 1.5배
- 종가 > Upper Band: 돌파 확인
```

---

## Pine Script 호환성 노트

**지원 방식**:
- Phase 1 (PoC): TA-Lib 기반 내장 지표만 지원
- Phase 2 (MVP): 사용자 커스텀 Pine Script 파싱 및 실행 (제한적)
- Phase 3 (Production): Pine Script v5 완전 지원 (sandbox 환경)

**제약사항**:
- 보안상 네트워크 호출 불가
- 최대 실행 시간: 5초
- 메모리 제한: 256MB per script

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 0.1  | 2025-11-21 | 초안 작성 (10개 지표) | Product Team |

---

**다음 단계**: 각 지표별 단위 테스트 및 백테스트 검증
