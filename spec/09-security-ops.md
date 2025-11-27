# Security and Operations Specification (v0.1)

## 개요

본 문서는 TradingView 지표 기반 종목 추천 서비스의 보안 정책, 인증/권한, 데이터 라이선스 관리, 운영 절차를 정의합니다.

---

## 인증 및 권한 (Authentication & Authorization)

### 1. 인증 방식

#### OAuth2 + OpenID Connect

**지원 Provider**:
- Google OAuth2
- GitHub OAuth2
- 로컬 인증 (이메일/비밀번호) - 옵션

**Flow**: Authorization Code Flow with PKCE

**구현**:
```python
# Example using authlib
from authlib.integrations.flask_client import OAuth

oauth = OAuth(app)

google = oauth.register(
    name='google',
    client_id=os.getenv('GOOGLE_CLIENT_ID'),
    client_secret=os.getenv('GOOGLE_CLIENT_SECRET'),
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={
        'scope': 'openid email profile'
    }
)

@app.route('/login/google')
def login_google():
    redirect_uri = url_for('auth_callback', _external=True)
    return google.authorize_redirect(redirect_uri)

@app.route('/auth/callback')
def auth_callback():
    token = google.authorize_access_token()
    user_info = google.parse_id_token(token)
    # Create session and JWT
    return create_jwt(user_info)
```

---

#### JWT (JSON Web Token)

**사용 목적**: API 인증

**토큰 구조**:
```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user_id_12345",
    "email": "user@example.com",
    "plan": "free",
    "iat": 1700000000,
    "exp": 1700086400
  }
}
```

**토큰 유효 기간**:
- Access Token: 15분
- Refresh Token: 7일

**보안 요구사항**:
- RS256 알고리즘 (비대칭키)
- HTTPS only
- HttpOnly, Secure, SameSite=Strict 쿠키

---

### 2. 권한 관리 (RBAC)

**역할 (Roles)**:

| 역할 | 권한 | 설명 |
|------|------|------|
| `anonymous` | 읽기 제한 | 지표 목록 조회, 제한된 스캔 |
| `free` | 기본 읽기/쓰기 | 스캔 (시간당 10회), 백테스트 조회 |
| `premium` | 고급 기능 | 스캔 무제한, 백테스트 실행, 알림 설정 |
| `admin` | 전체 권한 | 시스템 관리, 사용자 관리 |

**권한 매트릭스**:

| 엔드포인트 | anonymous | free | premium | admin |
|-----------|-----------|------|---------|-------|
| `GET /indicators` | ✅ | ✅ | ✅ | ✅ |
| `POST /scan` | ❌ | ✅ (10/h) | ✅ (무제한) | ✅ |
| `GET /backtest` | ✅ | ✅ | ✅ | ✅ |
| `POST /filters` | ❌ | ✅ | ✅ | ✅ |
| `POST /alerts` | ❌ | ❌ | ✅ | ✅ |
| `GET /admin/*` | ❌ | ❌ | ❌ | ✅ |

---

## API 보안

### 1. Rate Limiting

**목적**: DDoS 방지 및 공정한 리소스 사용

**제한**:
| 사용자 등급 | 시간당 요청 수 | 분당 요청 수 |
|------------|---------------|-------------|
| anonymous | 100 | 10 |
| free | 500 | 30 |
| premium | 5000 | 100 |

**구현** (Redis + Token Bucket):
```python
from redis import Redis
import time

redis_client = Redis(host='localhost', port=6379)

def check_rate_limit(user_id, limit=500, window=3600):
    """Token bucket rate limiter"""
    key = f"rate_limit:{user_id}"
    current_time = int(time.time())

    # Get current tokens
    tokens = redis_client.get(key)
    if tokens is None:
        tokens = limit
    else:
        tokens = int(tokens)

    if tokens > 0:
        redis_client.decr(key)
        redis_client.expire(key, window)
        return True
    else:
        return False
```

**응답 헤더**:
```
X-RateLimit-Limit: 500
X-RateLimit-Remaining: 487
X-RateLimit-Reset: 1700568000
```

---

### 2. API Key (대안)

**사용 목적**: 서버 간 통신, 외부 통합

**발급 방식**:
```
API Key Format: sk_live_1234567890abcdef
Prefix: sk_live (production), sk_test (testing)
```

**보안 요구사항**:
- HTTPS only
- 환경 변수 저장 (코드에 하드코딩 금지)
- 정기적 갱신 (90일)
- IP 화이트리스트 (옵션)

---

## 데이터 보안

### 1. 데이터 암호화

#### 전송 중 암호화 (In-Transit)
- **HTTPS/TLS 1.3**: 모든 API 통신
- **Certificate**: Let's Encrypt 또는 AWS Certificate Manager

#### 저장 암호화 (At-Rest)
- **Database**: PostgreSQL Transparent Data Encryption (TDE)
- **S3/Object Storage**: Server-Side Encryption (SSE)
- **Secrets**: AWS Secrets Manager 또는 HashiCorp Vault

---

### 2. 민감 정보 관리

**민감 정보 목록**:
- 사용자 비밀번호
- API Keys
- OAuth Client Secrets
- Database Credentials
- Third-party API Tokens

**저장 방식**:
```python
# Password hashing (bcrypt)
from passlib.hash import bcrypt

def hash_password(password: str) -> str:
    return bcrypt.hash(password)

def verify_password(password: str, hash: str) -> bool:
    return bcrypt.verify(password, hash)
```

**환경 변수 관리**:
```bash
# .env (로컬 개발용 - Git에 커밋 금지)
DATABASE_URL=postgresql://user:pass@localhost:5432/db
JWT_SECRET=your-secret-key
GOOGLE_CLIENT_SECRET=xxx
```

**Production 환경**:
- AWS Secrets Manager
- Kubernetes Secrets
- Vault by HashiCorp

---

## 취약점 방어

### 1. SQL Injection 방어

**방법**: Prepared Statements, ORM 사용

```python
# ❌ 취약한 코드
query = f"SELECT * FROM tickers WHERE symbol = '{symbol}'"
db.execute(query)

# ✅ 안전한 코드
query = "SELECT * FROM tickers WHERE symbol = %s"
db.execute(query, (symbol,))
```

---

### 2. XSS (Cross-Site Scripting) 방어

**방법**:
- 사용자 입력 출력 시 이스케이프
- Content-Security-Policy 헤더 설정

```python
# CSP Header
response.headers['Content-Security-Policy'] = "default-src 'self'; script-src 'self' 'unsafe-inline'"
```

---

### 3. CSRF (Cross-Site Request Forgery) 방어

**방법**:
- CSRF 토큰 사용
- SameSite=Strict 쿠키 속성

```python
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect(app)
```

---

### 4. 파일 업로드 보안 (향후 기능)

**제한**:
- 파일 타입 화이트리스트 (`.csv`, `.json`)
- 파일 크기 제한 (10MB)
- 바이러스 스캔 (ClamAV)

---

## 데이터 라이선스 관리

### 1. 데이터 소스별 라이선스

| 데이터 소스 | 라이선스 | 상업적 사용 | 재배포 |
|-----------|---------|-----------|-------|
| yfinance | Apache-2.0 | 제한적 (Yahoo ToS) | ❌ |
| Polygon.io | Commercial | ✅ (구독 필요) | ❌ |
| Tiingo | Commercial | ✅ (구독 필요) | ❌ |

### 2. Attribution (출처 표시)

**UI 표시**:
```
"데이터 제공: Yahoo Finance, Polygon.io"
"이 정보는 투자 조언이 아닙니다."
```

**API 응답**:
```json
{
  "data": { ... },
  "metadata": {
    "data_source": "polygon",
    "license": "commercial",
    "disclaimer": "Not financial advice"
  }
}
```

---

## 로깅 및 모니터링

### 1. 로그 정책

**로그 레벨**:
- `DEBUG`: 개발 환경
- `INFO`: 운영 환경 (기본)
- `WARNING`: 잠재적 문제
- `ERROR`: 오류 발생
- `CRITICAL`: 치명적 오류

**민감 정보 필터링**:
```python
import logging

class SensitiveDataFilter(logging.Filter):
    def filter(self, record):
        # Mask passwords, API keys, etc.
        record.msg = record.msg.replace(password, '***')
        return True

logger.addFilter(SensitiveDataFilter())
```

**로그 보관**:
- 로그 보관 기간: 6개월
- 압축 및 아카이브: 30일 이상

---

### 2. 보안 이벤트 로깅

**기록 대상**:
- 로그인 실패 (5회 이상 연속 실패 시 알림)
- 권한 없는 접근 시도
- Rate limit 초과
- 비정상적인 API 패턴

**예시**:
```json
{
  "timestamp": "2025-11-21T10:30:00Z",
  "event": "unauthorized_access",
  "user_id": "user_12345",
  "ip": "192.168.1.100",
  "endpoint": "/admin/users",
  "user_agent": "Mozilla/5.0..."
}
```

---

### 3. 모니터링 및 알럿

**메트릭**:
- API 응답 시간 (P95 > 5초)
- 에러율 (> 5%)
- Database Connection Pool (> 80% 사용)
- 디스크 사용률 (> 90%)

**알럿 채널**:
- PagerDuty (Critical)
- Slack (Warning)
- Email (Info)

---

## 컴플라이언스

### 1. GDPR (EU)

**사용자 권리**:
- 데이터 접근 권한
- 데이터 삭제 권한 (Right to be forgotten)
- 데이터 이동 권한

**구현**:
```python
@app.route('/api/v1/user/data-export', methods=['POST'])
@require_auth
def export_user_data():
    """Export all user data (GDPR compliance)"""
    user_id = get_current_user_id()
    data = get_all_user_data(user_id)
    return jsonify(data)

@app.route('/api/v1/user/delete-account', methods=['DELETE'])
@require_auth
def delete_user_account():
    """Delete user account and all associated data"""
    user_id = get_current_user_id()
    anonymize_user_data(user_id)
    return jsonify({"message": "Account deleted"})
```

---

### 2. 금융 데이터 면책 조항

**필수 문구**:
```
"본 서비스는 정보 제공 목적으로만 사용됩니다.
투자 결정은 사용자 본인의 책임입니다.
과거 성과가 미래 수익을 보장하지 않습니다."
```

---

## 사고 대응 절차 (Incident Response)

### 1. 보안 사고 분류

| 레벨 | 설명 | 대응 시간 |
|------|------|----------|
| P0 (Critical) | 데이터 유출, 서비스 전면 중단 | 즉시 (15분 이내) |
| P1 (High) | 부분 서비스 중단, 인증 문제 | 1시간 이내 |
| P2 (Medium) | 성능 저하, 일부 기능 장애 | 4시간 이내 |
| P3 (Low) | 경미한 버그 | 24시간 이내 |

### 2. 대응 절차

```
1. 탐지 (Detection)
   ↓
2. 격리 (Containment)
   ↓
3. 분석 (Analysis)
   ↓
4. 복구 (Recovery)
   ↓
5. 사후 검토 (Post-Mortem)
```

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 0.1  | 2025-11-21 | 초안 작성 | Security Team |

---

**다음 단계**: 보안 감사 및 침투 테스트 수행
