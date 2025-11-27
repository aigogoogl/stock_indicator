# Acceptance Criteria (v0.1)

## 개요

본 문서는 각 기능 요구사항(FR)과 비기능 요구사항(NFR)에 대한 수용 기준(Acceptance Criteria)과 테스트 방법을 정의합니다.

**목적**: 기능이 올바르게 구현되었는지 검증하기 위한 명확한 기준 제시

---

## 기능 요구사항 (FR) 수용 기준

### FR-001: 지표 선택 UI 제공

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-001-1 | 지표 목록에 최소 20개 이상의 지표가 표시되어야 함 | E2E Test: 지표 목록 화면 접속 후 개수 확인 | T-001-1 | HIGH |
| AC-001-2 | 각 지표 선택 시 해당 지표의 파라미터 입력 폼이 동적으로 렌더링되어야 함 | E2E Test: RSI 선택 → Period 입력 필드 존재 확인 | T-001-2 | HIGH |
| AC-001-3 | 각 파라미터에 대한 설명이 툴팁으로 제공되어야 함 | E2E Test: (?) 아이콘 hover 시 툴팁 표시 확인 | T-001-3 | MEDIUM |
| AC-001-4 | Preset 버튼 클릭 시 파라미터가 자동으로 입력되어야 함 | E2E Test: "Oversold (30)" 클릭 → period=14, value=30 자동 입력 확인 | T-001-4 | HIGH |
| AC-001-5 | 파라미터 유효성 검사 실패 시 에러 메시지가 표시되어야 함 | Unit Test: period=150 입력 → "Period must be between 2 and 100" 메시지 확인 | T-001-5 | HIGH |

**테스트 시나리오 (T-001)**:
```javascript
// tests/e2e/indicator-selector.spec.js
test('FR-001: Indicator selector UI', async ({ page }) => {
  await page.goto('/scan');

  // AC-001-1: 지표 목록 개수 확인
  const indicators = await page.locator('[data-testid="indicator-item"]').count();
  expect(indicators).toBeGreaterThanOrEqual(20);

  // AC-001-2: RSI 선택 → 파라미터 폼 표시
  await page.click('text=RSI');
  await expect(page.locator('input[name="period"]')).toBeVisible();

  // AC-001-3: 툴팁 확인
  await page.hover('[data-testid="period-help-icon"]');
  await expect(page.locator('text=RSI 계산 기간')).toBeVisible();

  // AC-001-4: Preset 자동 입력
  await page.click('text=Oversold (30)');
  expect(await page.inputValue('input[name="period"]')).toBe('14');

  // AC-001-5: 유효성 검사
  await page.fill('input[name="period"]', '150');
  await expect(page.locator('text=Period must be between 2 and 100')).toBeVisible();
});
```

---

### FR-002: 지정일 종가 기준 스캔 API

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-002-1 | POST /api/v1/scan 요청 시 200 OK 응답을 반환해야 함 | Integration Test: 정상 요청 → 200 상태 코드 확인 | T-002-1 | HIGH |
| AC-002-2 | 캐시된 응답의 경우 Top100을 3초 이내에 반환해야 함 (universe=1000 기준) | Performance Test: 동일 요청 2회 → 2번째 요청 응답 시간 < 3초 | T-002-2 | HIGH |
| AC-002-3 | 지정 날짜의 데이터가 없을 경우 400 Bad Request 및 명확한 에러 메시지를 반환해야 함 | Integration Test: 미래 날짜 요청 → 400 + "No data available for date" 메시지 확인 | T-002-3 | HIGH |
| AC-002-4 | 결과에 ticker, name, indicator_value, score, reason 필드가 포함되어야 함 | Integration Test: 응답 JSON 스키마 검증 | T-002-4 | HIGH |
| AC-002-5 | 지원하지 않는 지표 요청 시 400 Bad Request 및 "INVALID_INDICATOR" 에러 코드를 반환해야 함 | Integration Test: indicator="invalid" 요청 → 400 + error.code="INVALID_INDICATOR" | T-002-5 | HIGH |

**테스트 시나리오 (T-002)**:
```python
# tests/integration/test_scan_api.py
def test_fr002_scan_api(client, db_session):
    """FR-002: 스캔 API 테스트"""

    # AC-002-1: 정상 요청
    response = client.post('/api/v1/scan', json={
        'indicator': 'rsi',
        'parameters': {'period': 14},
        'condition': {'operator': 'lt', 'value': 30},
        'universe': 'sp500',
        'date': '2025-11-20',
        'limit': 100
    })
    assert response.status_code == 200
    assert response.json()['success'] is True

    # AC-002-2: 캐시된 응답 속도
    import time
    start = time.time()
    response2 = client.post('/api/v1/scan', json={...})  # 동일 요청
    elapsed = time.time() - start
    assert elapsed < 3.0

    # AC-002-3: 데이터 없는 날짜
    response = client.post('/api/v1/scan', json={
        'indicator': 'rsi',
        'date': '2099-12-31',
        ...
    })
    assert response.status_code == 400
    assert 'No data available' in response.json()['error']['message']

    # AC-002-4: 응답 스키마 검증
    data = response.json()['data']
    assert 'results' in data
    if len(data['results']) > 0:
        result = data['results'][0]
        assert 'ticker' in result
        assert 'name' in result
        assert 'indicator_value' in result
        assert 'score' in result
        assert 'reason' in result

    # AC-002-5: 잘못된 지표
    response = client.post('/api/v1/scan', json={
        'indicator': 'invalid_indicator',
        ...
    })
    assert response.status_code == 400
    assert response.json()['error']['code'] == 'INVALID_INDICATOR'
```

---

### FR-003: 지표 결과 정렬·필터링 및 설명 제공

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-003-1 | 정렬 옵션 변경 시 1초 이내에 재정렬된 결과를 표시해야 함 | E2E Test: 정렬 드롭다운 변경 → 1초 내 결과 업데이트 확인 | T-003-1 | MEDIUM |
| AC-003-2 | 필터 적용 후 조건에 맞는 결과만 표시되어야 함 | E2E Test: Sector="Technology" 필터 → 모든 결과의 sector 필드가 "Technology"인지 확인 | T-003-2 | HIGH |
| AC-003-3 | 각 종목에 추천 이유 텍스트가 표시되어야 함 | E2E Test: 결과 테이블에서 "reason" 컬럼 존재 및 내용 확인 | T-003-3 | HIGH |

---

### FR-004: 백테스트 집계 및 지표 성능 랭킹

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-004-1 | GET /api/v1/backtest 요청 시 백테스트 결과 JSON을 반환해야 함 | Integration Test: 정상 요청 → 200 + JSON 스키마 검증 | T-004-1 | HIGH |
| AC-004-2 | 메트릭이 최소 5개 이상 포함되어야 함 (win_rate, avg_return, sharpe_ratio, max_drawdown, cagr) | Integration Test: 응답 JSON의 metrics 객체에 5개 필드 존재 확인 | T-004-2 | HIGH |
| AC-004-3 | 기간별 상위 10개 지표 랭킹이 표시되어야 함 | Integration Test: GET /api/v1/indicators/ranking → ranking 배열 길이 <= 10 | T-004-3 | MEDIUM |
| AC-004-4 | 백테스트 파라미터 변경 가능해야 함 | E2E Test: 파라미터 변경 → 새로운 백테스트 결과 확인 | T-004-4 | MEDIUM |

---

### FR-005: 지표 카탈로그 및 메타데이터 제공

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-005-1 | GET /api/v1/indicators 요청 시 지표 목록 JSON을 반환해야 함 | Integration Test: 200 OK + indicators 배열 존재 | T-005-1 | HIGH |
| AC-005-2 | 각 지표에 name, description, category, parameters 필드가 포함되어야 함 | Integration Test: JSON 스키마 검증 | T-005-2 | HIGH |
| AC-005-3 | Preset이 최소 3개 이상 제공되어야 함 | Integration Test: indicators[0].presets 배열 길이 >= 3 | T-005-3 | MEDIUM |

---

### FR-008: 종목 상세 정보 조회

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-008-1 | GET /api/v1/ticker/{symbol} 요청 시 OHLCV 및 지표 값을 반환해야 함 | Integration Test: GET /ticker/AAPL → ohlcv, indicators 필드 존재 확인 | T-008-1 | HIGH |
| AC-008-2 | 날짜 범위 쿼리 파라미터를 지원해야 함 (start_date, end_date) | Integration Test: start_date, end_date 파라미터로 요청 → 해당 범위 데이터만 반환 | T-008-2 | HIGH |
| AC-008-3 | 응답 시간이 1초 이내여야 함 (1년치 데이터 기준) | Performance Test: 1년 데이터 요청 → 응답 시간 < 1초 | T-008-3 | MEDIUM |

---

## 비기능 요구사항 (NFR) 수용 기준

### NFR-001: 응답 시간

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-NFR-001-1 | POST /scan API가 캐시된 경우 P95 2초 이내에 응답해야 함 | Performance Test: Locust로 100 users, 5분간 → P95 < 2s 확인 | T-NFR-001-1 | HIGH |
| AC-NFR-001-2 | POST /scan API가 캐시되지 않은 경우 P95 5초 이내에 응답해야 함 | Performance Test: 캐시 무효화 후 → P95 < 5s | T-NFR-001-2 | HIGH |
| AC-NFR-001-3 | GET /indicators API가 500ms 이내에 응답해야 함 | Performance Test: P95 < 500ms | T-NFR-001-3 | HIGH |

**테스트 방법**:
```python
# tests/performance/test_response_time.py
from locust import HttpUser, task, between

class ScanPerformanceTest(HttpUser):
    wait_time = between(1, 2)

    @task
    def scan_api_cached(self):
        """NFR-001: 캐시된 스캔 API 응답 시간"""
        with self.client.post('/api/v1/scan', json={
            'indicator': 'rsi',
            'parameters': {'period': 14},
            'condition': {'operator': 'lt', 'value': 30},
            'universe': 'sp500',
            'date': '2025-11-20'
        }, catch_response=True) as response:
            if response.elapsed.total_seconds() > 2:
                response.failure(f"Response time {response.elapsed.total_seconds()}s > 2s")
```

---

### NFR-002: 가용성

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-NFR-002-1 | 월간 uptime이 99.5% 이상이어야 함 | 모니터링: Prometheus로 uptime 메트릭 수집 | T-NFR-002-1 | HIGH |
| AC-NFR-002-2 | Health check 엔드포인트가 항상 200 OK를 반환해야 함 | Smoke Test: GET /health → 200 OK | T-NFR-002-2 | HIGH |

---

### NFR-003: 데이터 정확도

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-NFR-003-1 | 데이터 소스가 응답 메타데이터에 명시되어야 함 | Integration Test: 응답의 metadata.data_source 필드 존재 확인 | T-NFR-003-1 | MEDIUM |
| AC-NFR-003-2 | 장마감 후 30분 이내에 EOD 데이터가 업데이트되어야 함 | 모니터링: ETL 파이프라인 완료 시간 추적 | T-NFR-003-2 | HIGH |

---

### NFR-004: 동시 요청 처리량

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-NFR-004-1 | 초당 100 requests를 처리할 수 있어야 함 | Load Test: Locust로 500 users → RPS >= 100 확인 | T-NFR-004-1 | HIGH |
| AC-NFR-004-2 | 동시 500명 사용자를 지원해야 함 | Load Test: 500 concurrent users → 에러율 < 1% | T-NFR-004-2 | HIGH |

---

### NFR-007: 보안

| AC ID | 수용 기준 | 테스트 방법 | 테스트 ID | 우선순위 |
|-------|----------|------------|----------|---------|
| AC-NFR-007-1 | 모든 API가 HTTPS를 통해서만 접근 가능해야 함 | Security Test: HTTP 요청 → 301 Redirect to HTTPS | T-NFR-007-1 | HIGH |
| AC-NFR-007-2 | Rate limiting이 적용되어야 함 (분당 60 requests) | Integration Test: 61개 요청 → 61번째 요청 429 Too Many Requests | T-NFR-007-2 | HIGH |
| AC-NFR-007-3 | SQL Injection 취약점이 없어야 함 | Security Test: SQL Injection 패턴 입력 → 에러 없이 처리 또는 400 Bad Request | T-NFR-007-3 | HIGH |

---

## 수용 기준 매트릭스 (요약)

| FR/NFR ID | 총 AC 수 | 완료 AC 수 | 진행률 | 상태 |
|-----------|---------|-----------|-------|------|
| FR-001 | 5 | 0 | 0% | 🔴 Not Started |
| FR-002 | 5 | 0 | 0% | 🔴 Not Started |
| FR-003 | 3 | 0 | 0% | 🔴 Not Started |
| FR-004 | 4 | 0 | 0% | 🔴 Not Started |
| FR-005 | 3 | 0 | 0% | 🔴 Not Started |
| FR-008 | 3 | 0 | 0% | 🔴 Not Started |
| NFR-001 | 3 | 0 | 0% | 🔴 Not Started |
| NFR-002 | 2 | 0 | 0% | 🔴 Not Started |
| NFR-003 | 2 | 0 | 0% | 🔴 Not Started |
| NFR-004 | 2 | 0 | 0% | 🔴 Not Started |
| NFR-007 | 3 | 0 | 0% | 🔴 Not Started |

**전체 진행률**: 0/35 (0%)

---

## 테스트 실행 가이드

### 1. Unit Tests
```bash
pytest tests/unit/ --cov=src --cov-report=html
```

### 2. Integration Tests
```bash
pytest tests/integration/ -v
```

### 3. E2E Tests
```bash
npx playwright test tests/e2e/
```

### 4. Performance Tests
```bash
locust -f tests/performance/locustfile.py --host=https://api.stockindicator.com --users=500 --spawn-rate=10
```

### 5. Security Tests
```bash
# SQL Injection
python tests/security/test_sql_injection.py

# XSS
python tests/security/test_xss.py
```

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 0.1  | 2025-11-21 | 초안 작성 | QA Team |

---

**다음 단계**:
1. 각 AC에 대한 테스트 코드 작성
2. CI/CD 파이프라인에 테스트 통합
3. 정기적 AC 진행률 업데이트
