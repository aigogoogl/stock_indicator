# CI/CD and Testing Specification (v0.1)

## 개요

본 문서는 TradingView 지표 기반 종목 추천 서비스의 CI/CD 파이프라인 및 테스트 전략을 정의합니다.

**목표**:
- 코드 품질 보장
- 자동화된 테스트 및 배포
- 빠른 피드백 루프

---

## CI/CD 파이프라인

### 전체 워크플로우

```
┌─────────────┐
│ Git Push    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ CI Pipeline │
│ (GitHub     │
│  Actions)   │
└──────┬──────┘
       │
       ├─> Lint & Format Check
       │
       ├─> Unit Tests
       │
       ├─> Integration Tests
       │
       ├─> Contract Tests (API Schema)
       │
       ├─> Build Docker Image
       │
       └─> Security Scan
       │
       ▼
┌─────────────┐
│ Deploy to   │
│ Staging     │
└──────┬──────┘
       │
       ├─> E2E Tests (Playwright)
       │
       ├─> Performance Tests
       │
       └─> Smoke Tests
       │
       ▼
┌─────────────┐
│ Manual      │
│ Approval    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Deploy to   │
│ Production  │
└─────────────┘
```

---

## 테스트 전략

### 테스트 피라미드

```
        /\
       /E2E\       (10%)
      /------\
     /Integr-\    (20%)
    /----------\
   /   Unit     \  (70%)
  /--------------\
```

---

## 1. Unit Tests (단위 테스트)

**목표 커버리지**: 80% 이상

**도구**:
- Backend (Python): `pytest`, `pytest-cov`
- Frontend (JavaScript/TypeScript): `Jest`, `React Testing Library`

**테스트 대상**:
- 지표 계산 함수
- 데이터 변환 로직
- 유틸리티 함수
- React 컴포넌트 (UI)

**예시 (Python - RSI 계산)**:
```python
# tests/test_indicators.py
import pytest
from indicators import calculate_rsi

def test_rsi_basic():
    """Test RSI calculation with known values"""
    prices = [44, 44.34, 44.09, 43.61, 44.33, 44.83, 45.10, 45.42, 45.84, 46.08,
              45.89, 46.03, 45.61, 46.28, 46.28, 46.00, 46.03, 46.41, 46.22, 45.64]

    rsi = calculate_rsi(prices, period=14)

    assert rsi is not None
    assert 0 <= rsi <= 100
    assert pytest.approx(rsi, abs=1) == 51.78  # Known value

def test_rsi_edge_cases():
    """Test RSI with edge cases"""
    # All prices same → RSI should be 50
    prices = [100] * 20
    rsi = calculate_rsi(prices, period=14)
    assert rsi == 50

    # Insufficient data → should raise error
    with pytest.raises(ValueError):
        calculate_rsi([100, 101], period=14)
```

**예시 (JavaScript - React Component)**:
```javascript
// tests/IndicatorSelector.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import IndicatorSelector from '../components/IndicatorSelector';

describe('IndicatorSelector', () => {
  test('renders indicator list', () => {
    render(<IndicatorSelector />);
    expect(screen.getByText('RSI')).toBeInTheDocument();
    expect(screen.getByText('MACD')).toBeInTheDocument();
  });

  test('shows parameters when indicator selected', () => {
    render(<IndicatorSelector />);
    fireEvent.click(screen.getByText('RSI'));
    expect(screen.getByText('Period')).toBeInTheDocument();
  });

  test('validates parameter range', () => {
    render(<IndicatorSelector />);
    fireEvent.click(screen.getByText('RSI'));

    const input = screen.getByLabelText('Period');
    fireEvent.change(input, { target: { value: '150' } });

    expect(screen.getByText('Period must be between 2 and 100')).toBeInTheDocument();
  });
});
```

**실행 명령**:
```bash
# Python
pytest --cov=src --cov-report=html

# JavaScript
npm test -- --coverage
```

---

## 2. Integration Tests (통합 테스트)

**목표**: 모듈 간 상호작용 검증

**테스트 대상**:
- API 엔드포인트
- 데이터베이스 쿼리
- ETL 파이프라인
- 외부 API 호출 (Mock)

**예시 (API Integration Test)**:
```python
# tests/test_api_integration.py
import pytest
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

@pytest.fixture
def sample_db(db_session):
    """Setup sample data in test DB"""
    # Insert sample tickers and OHLCV data
    pass

def test_scan_endpoint(sample_db):
    """Test POST /api/v1/scan"""
    response = client.post(
        "/api/v1/scan",
        json={
            "indicator": "rsi",
            "parameters": {"period": 14},
            "condition": {"operator": "lt", "value": 30},
            "universe": "sp500",
            "date": "2025-11-20",
            "limit": 10
        }
    )

    assert response.status_code == 200
    data = response.json()
    assert data["success"] is True
    assert "results" in data["data"]
    assert len(data["data"]["results"]) <= 10

def test_scan_invalid_indicator():
    """Test scan with invalid indicator"""
    response = client.post(
        "/api/v1/scan",
        json={
            "indicator": "invalid_indicator",
            "parameters": {},
            "condition": {},
            "universe": "sp500"
        }
    )

    assert response.status_code == 400
    assert response.json()["error"]["code"] == "INVALID_INDICATOR"
```

---

## 3. Contract Tests (계약 테스트)

**목표**: API 스펙 준수 확인

**도구**: `Pact` 또는 OpenAPI Validator

**예시**:
```python
# tests/test_api_contract.py
import json
from openapi_spec_validator import validate_spec

def test_openapi_spec_valid():
    """Test OpenAPI spec is valid"""
    with open('spec/openapi.yaml') as f:
        spec = yaml.safe_load(f)

    validate_spec(spec)  # Raises error if invalid

def test_scan_response_matches_schema():
    """Test /scan response matches OpenAPI schema"""
    response = client.post("/api/v1/scan", json={...})

    # Validate against schema
    assert response.json() matches OpenAPI schema
```

---

## 4. E2E Tests (End-to-End 테스트)

**목표**: 사용자 시나리오 검증

**도구**: `Playwright` (권장) 또는 `Cypress`

**테스트 시나리오**:
1. 사용자가 지표를 선택하고 스캔 실행
2. 스캔 결과를 확인하고 종목 클릭
3. 종목 상세 페이지에서 차트 확인
4. 백테스트 실행 및 결과 확인

**예시 (Playwright)**:
```javascript
// tests/e2e/scan-flow.spec.js
const { test, expect } = require('@playwright/test');

test('user can scan stocks with RSI indicator', async ({ page }) => {
  // 1. Navigate to scan page
  await page.goto('http://localhost:3000/scan');

  // 2. Select RSI indicator
  await page.click('text=RSI');

  // 3. Set parameters
  await page.fill('input[name="period"]', '14');
  await page.selectOption('select[name="condition"]', 'lt');
  await page.fill('input[name="value"]', '30');

  // 4. Select universe
  await page.selectOption('select[name="universe"]', 'sp500');

  // 5. Click scan button
  await page.click('button:has-text("스캔 실행")');

  // 6. Wait for results
  await page.waitForSelector('table[data-testid="results-table"]');

  // 7. Verify results
  const rows = await page.locator('table tbody tr').count();
  expect(rows).toBeGreaterThan(0);

  // 8. Click first result
  await page.click('table tbody tr:first-child');

  // 9. Verify ticker detail page
  await expect(page).toHaveURL(/\/ticker\/[A-Z]+/);
  await expect(page.locator('h1')).toContainText('AAPL');
});
```

**실행 명령**:
```bash
npx playwright test
```

---

## 5. Performance Tests (성능 테스트)

**목표**: 응답 시간 및 처리량 검증

**도구**: `Locust`, `k6`

**테스트 시나리오**:
- 스캔 API: 초당 100 requests
- 동시 사용자: 500명
- 응답 시간: P95 < 3초

**예시 (Locust)**:
```python
# tests/performance/locustfile.py
from locust import HttpUser, task, between

class StockScanUser(HttpUser):
    wait_time = between(1, 3)

    @task(3)
    def scan_rsi(self):
        """Scan with RSI indicator"""
        self.client.post("/api/v1/scan", json={
            "indicator": "rsi",
            "parameters": {"period": 14},
            "condition": {"operator": "lt", "value": 30},
            "universe": "sp500",
            "date": "2025-11-20"
        })

    @task(1)
    def get_indicators(self):
        """Get indicator list"""
        self.client.get("/api/v1/indicators")
```

**실행 명령**:
```bash
locust -f tests/performance/locustfile.py --host=http://localhost:8000 --users=500 --spawn-rate=10
```

**목표 메트릭**:
- P50 응답 시간: < 1초
- P95 응답 시간: < 3초
- P99 응답 시간: < 5초
- 에러율: < 1%

---

## GitHub Actions CI/CD 설정

### `.github/workflows/ci.yml`

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install flake8 black mypy
      - name: Run linters
        run: |
          flake8 src/
          black --check src/
          mypy src/

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov
      - name: Run unit tests
        run: |
          pytest --cov=src --cov-report=xml
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: |
          docker build -t stockindicator:${{ github.sha }} .
      - name: Push to registry
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker push stockindicator:${{ github.sha }}

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: |
          # Deploy to staging environment
          kubectl set image deployment/stockindicator stockindicator=stockindicator:${{ github.sha }} -n staging
```

---

## 테스트 커버리지 목표

| 레이어 | 목표 커버리지 |
|--------|--------------|
| Unit Tests | 80% |
| Integration Tests | 60% |
| E2E Tests | 주요 사용자 플로우 커버 |

---

## Acceptance Tests Mapping

각 FR (Functional Requirement)에 대응하는 테스트 ID 매핑

| FR ID | 테스트 ID | 테스트 타입 | 설명 |
|-------|----------|------------|------|
| FR-001 | T-001 | E2E | 지표 선택 UI 테스트 |
| FR-002 | T-002 | Integration | 스캔 API 테스트 |
| FR-002 | T-003 | Performance | 스캔 API 응답 시간 테스트 |
| FR-003 | T-004 | E2E | 결과 정렬/필터링 테스트 |
| FR-004 | T-005 | Integration | 백테스트 API 테스트 |
| FR-005 | T-006 | Unit | 지표 메타데이터 테스트 |

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 0.1  | 2025-11-21 | 초안 작성 | QA Team |

---

**다음 단계**: 테스트 코드 작성 및 CI/CD 파이프라인 구축
