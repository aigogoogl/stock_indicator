# ETL Specification (v0.1)

## 개요

본 문서는 TradingView 지표 기반 종목 추천 서비스의 ETL (Extract, Transform, Load) 파이프라인을 정의합니다.

**목적**: 외부 데이터 소스로부터 OHLCV 데이터를 수집하고, 지표를 계산하여 데이터베이스에 적재

**주요 데이터 소스**:
- **PoC**: yfinance (무료)
- **MVP/Production**: Polygon.io, Tiingo (유료)

---

## ETL 아키텍처

```
┌─────────────────┐      ┌──────────────┐      ┌─────────────┐
│  Data Sources   │      │   ETL Jobs   │      │  Database   │
│  (yfinance,     │ ───> │  (Airflow/   │ ───> │ (PostgreSQL │
│   Polygon, etc) │      │   Python)    │      │  TimescaleDB)│
└─────────────────┘      └──────────────┘      └─────────────┘
                              │
                              ├─> Data Quality Checks
                              ├─> Indicator Calculation
                              └─> Cache Warm-up (Redis)
```

---

## 데이터 소스

### 1. yfinance (PoC)

**장점**:
- 무료
- Python 라이브러리 제공
- 간단한 API

**단점**:
- Rate limit 있음
- 데이터 품질 보장 없음
- 상업적 사용 제한

**사용 예시**:
```python
import yfinance as yf

ticker = yf.Ticker("AAPL")
hist = ticker.history(start="2024-01-01", end="2025-11-20")
# DataFrame: Date, Open, High, Low, Close, Volume
```

**지원 시장**: 미국, 한국 등 주요 시장

---

### 2. Polygon.io (Production)

**장점**:
- 신뢰성 높은 데이터
- REST API + WebSocket
- 실시간 데이터 지원

**단점**:
- 유료 (Starter $199/월)

**API 예시**:
```
GET https://api.polygon.io/v2/aggs/ticker/AAPL/range/1/day/2024-01-01/2025-11-20?apiKey=YOUR_KEY
```

**Rate Limit**: Starter 플랜 기준 분당 5 requests

---

### 3. Tiingo (Production Alternative)

**장점**:
- 합리적인 가격 ($10/월~)
- 백테스트용 과거 데이터 풍부

**단점**:
- API 속도 다소 느림

**API 예시**:
```
GET https://api.tiingo.com/tiingo/daily/AAPL/prices?startDate=2024-01-01&endDate=2025-11-20&token=YOUR_TOKEN
```

---

## ETL 파이프라인

### 파이프라인 1: EOD Data Collection

**목적**: 장마감 후 일별 OHLCV 데이터 수집

**스케줄**:
- 미국 시장: 매일 EST 16:30 (장마감 후 30분)
- 한국 시장: 매일 KST 15:30 (장마감)

**단계**:
1. **Extract**: 데이터 소스에서 전일 OHLCV 데이터 가져오기
2. **Transform**:
   - 조정 종가(adjusted close) 계산 (스플릿, 배당 반영)
   - 데이터 정규화 (컬럼명 통일, 타입 변환)
   - 이상치 탐지 및 필터링
3. **Load**: `ohlcv` 테이블에 INSERT (중복 체크: UPSERT)
4. **Quality Check**: 누락 데이터, 이상값 검사
5. **Alert**: 실패 시 Slack/Email 알림

**구현 예시 (Python + Airflow)**:
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import yfinance as yf
import pandas as pd

def extract_ohlcv(ticker_list, date):
    """Extract OHLCV data from yfinance"""
    data = []
    for ticker in ticker_list:
        try:
            hist = yf.Ticker(ticker).history(start=date, end=date)
            if not hist.empty:
                data.append({
                    'ticker': ticker,
                    'date': date,
                    'open': hist['Open'].iloc[0],
                    'high': hist['High'].iloc[0],
                    'low': hist['Low'].iloc[0],
                    'close': hist['Close'].iloc[0],
                    'volume': hist['Volume'].iloc[0],
                    'adjusted_close': hist['Close'].iloc[0]
                })
        except Exception as e:
            print(f"Error fetching {ticker}: {e}")
    return pd.DataFrame(data)

def transform_ohlcv(df):
    """Transform and validate data"""
    # Remove outliers (price change > 50% in one day)
    df['price_change'] = df['close'] / df['open'] - 1
    df = df[df['price_change'].abs() < 0.5]

    # Validate volume > 0
    df = df[df['volume'] > 0]

    return df

def load_ohlcv(df, db_conn):
    """Load data to database"""
    # UPSERT logic
    df.to_sql('ohlcv', db_conn, if_exists='append', index=False,
              method='multi', chunksize=1000)

# Airflow DAG
default_args = {
    'owner': 'data-team',
    'retries': 3,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'eod_data_collection',
    default_args=default_args,
    description='Daily EOD OHLCV data collection',
    schedule_interval='30 16 * * 1-5',  # Mon-Fri 16:30 EST
    start_date=datetime(2025, 1, 1),
    catchup=False
)

extract_task = PythonOperator(
    task_id='extract_ohlcv',
    python_callable=extract_ohlcv,
    dag=dag
)

transform_task = PythonOperator(
    task_id='transform_ohlcv',
    python_callable=transform_ohlcv,
    dag=dag
)

load_task = PythonOperator(
    task_id='load_ohlcv',
    python_callable=load_ohlcv,
    dag=dag
)

extract_task >> transform_task >> load_task
```

**재시도 로직**:
- 최대 3회 재시도
- 재시도 간격: 5분, 10분, 20분 (exponential backoff)
- 3회 실패 시 알림 발송 및 수동 개입 대기

---

### 파이프라인 2: Indicator Calculation

**목적**: OHLCV 데이터를 기반으로 지표 계산

**스케줄**: EOD 데이터 수집 완료 후 즉시 (체이닝)

**단계**:
1. **Load**: 당일 수집된 OHLCV 데이터 조회
2. **Calculate**: TA-Lib 또는 pandas-ta 사용하여 지표 계산
3. **Store**: `indicator_values` 테이블에 INSERT

**지표 계산 예시 (RSI)**:
```python
import talib
import pandas as pd

def calculate_rsi(ticker_id, period=14):
    """Calculate RSI for a ticker"""
    # Fetch last N days of close prices
    query = f"""
    SELECT date, close
    FROM ohlcv
    WHERE ticker_id = {ticker_id}
    ORDER BY date DESC
    LIMIT {period + 100}
    """
    df = pd.read_sql(query, db_conn)

    # Calculate RSI
    df['rsi'] = talib.RSI(df['close'], timeperiod=period)

    # Get latest RSI
    latest_rsi = df.iloc[0]['rsi']
    latest_date = df.iloc[0]['date']

    return {
        'ticker_id': ticker_id,
        'date': latest_date,
        'indicator_id': 'rsi',
        'parameters': {'period': period},
        'values': {'rsi': latest_rsi}
    }
```

**지원 지표 목록**:
- RSI (Relative Strength Index)
- MACD (Moving Average Convergence Divergence)
- SMA/EMA (Simple/Exponential Moving Average)
- Bollinger Bands
- ATR (Average True Range)
- ADX (Average Directional Index)
- OBV (On-Balance Volume)
- Stochastic Oscillator

**병렬 처리**:
- Celery 또는 Ray를 사용하여 종목별 병렬 계산
- Worker 수: CPU 코어 수 * 2

---

### 파이프라인 3: Backtest Calculation (Daily Batch)

**목적**: 지표별 백테스트 성과 계산

**스케줄**: 매일 자정

**단계**:
1. **Load**: 지표 정의 및 파라미터 조합 조회
2. **Backtest**: 과거 1년치 데이터로 백테스트 실행
3. **Aggregate**: 승률, 평균 수익률, 샤프 비율 등 메트릭 계산
4. **Store**: `backtest_results` 테이블에 UPSERT

**백테스트 로직**:
```python
def backtest_indicator(indicator_id, parameters, condition, universe, period='1y'):
    """Run backtest for an indicator"""
    # 1. Get historical signals
    signals = get_historical_signals(indicator_id, parameters, condition, universe, period)

    # 2. Simulate trades
    trades = []
    for signal in signals:
        entry_price = signal['close']
        exit_date = signal['date'] + timedelta(days=5)  # 5일 후 청산
        exit_price = get_close_price(signal['ticker_id'], exit_date)

        return_pct = (exit_price - entry_price) / entry_price
        trades.append({
            'ticker': signal['ticker'],
            'entry_date': signal['date'],
            'exit_date': exit_date,
            'return': return_pct
        })

    # 3. Calculate metrics
    df = pd.DataFrame(trades)
    metrics = {
        'total_signals': len(trades),
        'win_rate': (df['return'] > 0).mean(),
        'avg_return': df['return'].mean(),
        'sharpe_ratio': df['return'].mean() / df['return'].std() * np.sqrt(252),
        'max_drawdown': calculate_max_drawdown(df)
    }

    return metrics
```

---

## 데이터 품질 체크

### 1. 누락 데이터 탐지

**규칙**:
- 거래일에 데이터가 없으면 알림
- 전체 종목 중 80% 이상 데이터 수집 실패 시 파이프라인 중단

**구현**:
```python
def check_missing_data(date, universe):
    total_tickers = get_ticker_count(universe)
    collected = get_collected_count(date, universe)

    if collected / total_tickers < 0.8:
        raise DataQualityError(f"Only {collected}/{total_tickers} collected")
```

---

### 2. 이상치 탐지

**규칙**:
- 일일 가격 변동 > 50%: 스플릿 또는 데이터 오류 가능성
- 거래량 = 0: 데이터 오류
- High < Low: 데이터 오류

**처리**:
- 이상치 발견 시 해당 레코드 플래그 (`is_anomaly = true`)
- 수동 검토 후 수정 또는 삭제

---

### 3. 데이터 정합성 검사

**규칙**:
- Open, High, Low, Close 관계: `Low <= Open <= High`, `Low <= Close <= High`
- Adjusted Close 일관성: 스플릿/배당 이벤트 확인

---

## 데이터 변환 규칙

### 1. 조정 종가 (Adjusted Close)

**목적**: 스플릿 및 배당을 반영한 실제 수익률 계산

**공식**:
```
Adjusted Close = Close * (1 + Dividend / Close) * Split Coefficient
```

**예시**:
- 2025-11-15: Close = $100
- 2025-11-16: 2-for-1 split → Close = $50, Split Coefficient = 2.0
- Adjusted Close (2025-11-15) = $100 * 2.0 = $200 (역산)

---

### 2. 환율 고려 (다국가 지원 시)

**목적**: USD 기준 통일

**공식**:
```
Price (USD) = Price (Local Currency) / Exchange Rate
```

---

## 오류 처리 및 복구

### 오류 유형

| 오류 유형 | 처리 방법 | 알림 |
|----------|----------|------|
| API Rate Limit | 5분 대기 후 재시도 | Warning |
| Network Timeout | 즉시 재시도 (최대 3회) | Warning |
| 데이터 누락 (< 80%) | 파이프라인 중단 | Critical |
| 이상치 탐지 | 플래그 후 계속 진행 | Info |
| DB Connection Error | 재시도 (exponential backoff) | Critical |

---

### 재시도 전략

```python
import time
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    delay = base_delay * (2 ** attempt)
                    print(f"Retry {attempt+1}/{max_retries} after {delay}s: {e}")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry_with_backoff(max_retries=3, base_delay=5)
def fetch_data_from_api(ticker):
    # API call logic
    pass
```

---

## 모니터링 및 알림

### 메트릭 수집

| 메트릭 | 설명 | 알럿 조건 |
|--------|------|----------|
| `etl_duration_seconds` | ETL 실행 시간 | > 3600s (1시간) |
| `etl_success_rate` | 성공률 | < 0.95 |
| `data_freshness_hours` | 최신 데이터 경과 시간 | > 48 hours |
| `missing_data_ratio` | 누락 데이터 비율 | > 0.2 |

### 알림 채널

- **Critical**: PagerDuty (On-call 엔지니어)
- **Warning**: Slack #data-alerts 채널
- **Info**: CloudWatch Logs

---

## 데이터 버전 관리

### 데이터 버저닝

- 각 레코드에 `data_version` 필드 추가
- 데이터 소스 변경 시 버전 증가
- 예: yfinance → Polygon으로 전환 시 `data_version = 2`

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 0.1  | 2025-11-21 | 초안 작성 | Data Team |

---

**다음 단계**: Airflow DAG 파일 작성 및 ETL 코드 구현 (`etl/` 디렉토리)
