# Backtest Specification (v0.1)

## 개요

본 문서는 지표 및 전략의 과거 성과를 검증하기 위한 백테스트 방법론과 메트릭을 정의합니다.

**목적**: 사용자에게 각 지표의 과거 성과를 제공하여 의사결정 지원

---

## 백테스트 방법론

### 1. Walk-Forward Testing

**설명**: 과최적화(overfitting)를 방지하기 위해 데이터를 학습 기간과 검증 기간으로 분리하여 순차적으로 테스트

**구조**:
```
Timeline: [========== Data ==========]
          [Train][Test][Train][Test]...
```

**예시**:
- In-Sample (학습): 2024-01-01 ~ 2024-12-31 (1년)
- Out-of-Sample (검증): 2025-01-01 ~ 2025-03-31 (3개월)

---

### 2. Rolling Window Backtest

**설명**: 고정된 기간의 윈도우를 시간 순으로 이동하며 백테스트 수행

**윈도우 크기**: 1년 (252 거래일)
**이동 간격**: 1개월 (21 거래일)

**예시**:
```
Window 1: 2023-01-01 ~ 2024-01-01
Window 2: 2023-02-01 ~ 2024-02-01
...
```

---

## 거래 규칙

### 1. 진입 규칙 (Entry)

**신호 발생 시점**: 장마감 시점에 지표 조건 충족 확인
**주문 실행**: 다음 거래일 시가(Open)에 진입
**포지션 타입**: Long only (매수만)

**예시**:
```
2025-11-20 종가: RSI = 28.5 (조건 충족: RSI < 30)
→ 2025-11-21 시가에 매수 진입
```

---

### 2. 청산 규칙 (Exit)

**기본 청산**: N일 후 자동 청산 (기본값: 5일)
**Stop Loss**: 진입 가격 대비 -10% 손실 시 즉시 청산
**Take Profit**: 진입 가격 대비 +20% 수익 시 즉시 청산

**청산 우선순위**:
1. Stop Loss (최우선)
2. Take Profit
3. N일 경과 시 자동 청산

---

### 3. 포지션 사이징

#### 옵션 1: 고정 금액 (Fixed Amount)
```
각 거래당 고정 금액: $10,000
매수 주수 = $10,000 / 진입 가격
```

#### 옵션 2: 비례 할당 (Percentage of Portfolio)
```
각 거래당 포트폴리오의 10% 할당
매수 금액 = 총 자산 * 0.1
```

#### 옵션 3: ATR 기반 (Volatility-Adjusted)
```
Risk per Trade = $500 (포트폴리오의 0.5%)
Stop Loss Distance = ATR * 2
Position Size = Risk per Trade / Stop Loss Distance
```

**기본 설정**: 고정 금액 $10,000

---

## 비용 및 슬리피지

### 1. 거래 비용 (Commission)

| 시장 | 수수료 |
|------|--------|
| 미국 | $0.005/주 (최소 $1) |
| 한국 | 0.015% (매수/매도 각각) |

**예시 (미국)**:
```
매수: 100주 * $150 = $15,000
수수료: 100 * $0.005 = $0.50
총 비용: $15,000.50
```

---

### 2. 슬리피지 (Slippage)

**정의**: 주문 가격과 실제 체결 가격의 차이

**모델링**:
- **Percentage Slippage**: 진입/청산 시 각 0.1%
- **Fixed Slippage**: 진입/청산 시 각 $0.05

**기본 설정**: Percentage 0.1%

**예시**:
```
이론적 진입 가격: $100
슬리피지 0.1% 적용: $100 * 1.001 = $100.10 (실제 진입 가격)
```

---

## 백테스트 메트릭

### 1. 기본 메트릭

| 메트릭 | 설명 | 계산 공식 | 목표값 |
|--------|------|----------|--------|
| **Total Signals** | 총 신호 발생 횟수 | Count | - |
| **Total Trades** | 실제 거래 횟수 | Count | - |
| **Win Rate** | 승률 | 수익 거래 / 총 거래 | > 0.55 |
| **Avg Return** | 평균 수익률 | Σ(Return) / Total Trades | > 0.02 |
| **Total Return** | 총 수익률 | (Final Value - Initial Value) / Initial Value | > 0.15 |
| **CAGR** | 연평균 복리 수익률 | (Final Value / Initial Value)^(1/Years) - 1 | > 0.20 |

---

### 2. 리스크 메트릭

| 메트릭 | 설명 | 계산 공식 | 목표값 |
|--------|------|----------|--------|
| **Max Drawdown (MDD)** | 최대 낙폭 | Max(Peak - Trough) / Peak | < -0.20 |
| **Sharpe Ratio** | 샤프 비율 | (Return - Risk-Free Rate) / Std Dev | > 1.0 |
| **Sortino Ratio** | 소르티노 비율 | (Return - Risk-Free Rate) / Downside Dev | > 1.5 |
| **Calmar Ratio** | 칼마 비율 | CAGR / Max Drawdown | > 2.0 |
| **Volatility** | 변동성 (연환산) | Std Dev of Returns * √252 | < 0.30 |

---

### 3. 거래 분석 메트릭

| 메트릭 | 설명 |
|--------|------|
| **Avg Win** | 평균 수익 거래 수익률 |
| **Avg Loss** | 평균 손실 거래 손실률 |
| **Win/Loss Ratio** | Avg Win / Avg Loss |
| **Profit Factor** | Gross Profit / Gross Loss |
| **Avg Hold Days** | 평균 보유 기간 |
| **Max Consecutive Wins** | 최대 연속 수익 |
| **Max Consecutive Losses** | 최대 연속 손실 |

---

## 메트릭 계산 예시

### Sharpe Ratio
```python
import numpy as np

def calculate_sharpe_ratio(returns, risk_free_rate=0.02):
    """
    returns: 일별 수익률 리스트
    risk_free_rate: 무위험 수익률 (연환산)
    """
    excess_returns = returns - (risk_free_rate / 252)
    sharpe = np.mean(excess_returns) / np.std(excess_returns) * np.sqrt(252)
    return sharpe

# 예시
daily_returns = [0.01, -0.005, 0.02, 0.015, -0.01, ...]
sharpe = calculate_sharpe_ratio(daily_returns)
print(f"Sharpe Ratio: {sharpe:.2f}")
```

---

### Max Drawdown
```python
def calculate_max_drawdown(equity_curve):
    """
    equity_curve: 일별 자산 가치 리스트
    """
    peak = equity_curve[0]
    max_dd = 0

    for value in equity_curve:
        if value > peak:
            peak = value
        dd = (value - peak) / peak
        if dd < max_dd:
            max_dd = dd

    return max_dd

# 예시
equity = [10000, 10500, 10200, 10800, 9500, 11000, ...]
mdd = calculate_max_drawdown(equity)
print(f"Max Drawdown: {mdd:.2%}")
```

---

## 백테스트 리포트 구조

### 1. 요약 섹션
```json
{
  "summary": {
    "indicator": "rsi",
    "parameters": { "period": 14 },
    "condition": { "operator": "lt", "value": 30 },
    "period": "1y",
    "universe": "sp500",
    "start_date": "2024-11-21",
    "end_date": "2025-11-20",
    "initial_capital": 100000,
    "final_value": 123456
  }
}
```

---

### 2. 메트릭 섹션
```json
{
  "metrics": {
    "total_signals": 342,
    "total_trades": 312,
    "win_rate": 0.623,
    "avg_return": 0.0347,
    "total_return": 0.234,
    "cagr": 0.234,
    "max_drawdown": -0.156,
    "sharpe_ratio": 1.42,
    "sortino_ratio": 1.89,
    "volatility": 0.187
  }
}
```

---

### 3. 거래 이력
```json
{
  "trades": [
    {
      "trade_id": 1,
      "ticker": "AAPL",
      "entry_date": "2024-12-01",
      "entry_price": 178.50,
      "exit_date": "2024-12-06",
      "exit_price": 185.20,
      "shares": 56,
      "return": 0.0375,
      "pnl": 375.20,
      "hold_days": 5,
      "exit_reason": "auto_close"
    }
  ]
}
```

---

### 4. Equity Curve (자산 곡선)
```json
{
  "equity_curve": [
    { "date": "2024-11-21", "value": 100000, "drawdown": 0 },
    { "date": "2024-11-22", "value": 100500, "drawdown": 0 },
    { "date": "2024-11-25", "value": 99800, "drawdown": -0.007 }
  ]
}
```

---

### 5. 섹터별 성과
```json
{
  "performance_by_sector": [
    {
      "sector": "Technology",
      "trades": 85,
      "win_rate": 0.68,
      "avg_return": 0.042
    },
    {
      "sector": "Healthcare",
      "trades": 62,
      "win_rate": 0.61,
      "avg_return": 0.031
    }
  ]
}
```

---

## 백테스트 파라미터 설정

### 기본 설정 (Default)
```json
{
  "initial_capital": 100000,
  "position_sizing": "fixed_amount",
  "position_size": 10000,
  "hold_days": 5,
  "stop_loss": -0.10,
  "take_profit": 0.20,
  "commission": 0.0001,
  "slippage": 0.001,
  "risk_free_rate": 0.02
}
```

### 사용자 커스터마이징
- 사용자가 웹 UI에서 파라미터 변경 가능
- Preset: Conservative, Moderate, Aggressive

---

## 백테스트 실행 프로세스

```
1. 파라미터 입력 (지표, 조건, 기간, 종목군)
   ↓
2. 데이터 로드 (OHLCV + 지표 값)
   ↓
3. 신호 생성 (조건 충족 확인)
   ↓
4. 거래 시뮬레이션 (진입/청산 규칙 적용)
   ↓
5. 비용 및 슬리피지 적용
   ↓
6. 메트릭 계산
   ↓
7. 리포트 생성 (JSON + 차트)
   ↓
8. 결과 저장 (backtest_results 테이블)
```

---

## 백테스트 제약사항 및 주의사항

### 1. Look-Ahead Bias (미래 정보 유출)
- ❌ 잘못된 예: 당일 종가로 당일 진입
- ✅ 올바른 예: 당일 종가로 신호 확인 → 다음 거래일 시가에 진입

### 2. Survivorship Bias (생존 편향)
- 상장 폐지된 종목 데이터도 포함해야 함
- 현재 PoC에서는 생존 종목만 포함 (MVP에서 개선)

### 3. Data Quality
- 스플릿 및 배당 조정 필수
- 데이터 누락 시 해당 거래 제외

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 0.1  | 2025-11-21 | 초안 작성 | Quant Team |

---

**다음 단계**: 백테스트 엔진 구현 (`backtesting/` 디렉토리)
