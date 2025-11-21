# Deployment Specification (v0.1)

## 개요

본 문서는 TradingView 지표 기반 종목 추천 서비스의 인프라 구성 및 배포 절차를 정의합니다.

**클라우드 Provider**: AWS (권장) 또는 GCP
**컨테이너 오케스트레이션**: Kubernetes (EKS/GKE)
**IaC (Infrastructure as Code)**: Terraform

---

## 인프라 아키텍처

### 전체 구조

```
                    ┌────────────────┐
                    │  CloudFlare    │
                    │  (CDN + WAF)   │
                    └────────┬───────┘
                             │
                    ┌────────▼───────┐
                    │  Load Balancer │
                    │  (ALB/NLB)     │
                    └────────┬───────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼────┐        ┌─────▼─────┐      ┌──────▼──────┐
   │  Web    │        │  API      │      │  Worker     │
   │  (React)│        │  (FastAPI)│      │  (Celery)   │
   │  Pod x3 │        │  Pod x5   │      │  Pod x3     │
   └─────────┘        └─────┬─────┘      └──────┬──────┘
                            │                   │
                    ┌───────▼───────────────────▼──┐
                    │  PostgreSQL (Primary)        │
                    │  + TimescaleDB               │
                    ├──────────────────────────────┤
                    │  Read Replica x2             │
                    └──────────────────────────────┘

                    ┌──────────────────────────────┐
                    │  Redis (ElastiCache)         │
                    │  - Cache                     │
                    │  - Rate Limiting             │
                    │  - Celery Broker             │
                    └──────────────────────────────┘

                    ┌──────────────────────────────┐
                    │  S3 / Object Storage         │
                    │  - Backups                   │
                    │  - Static Assets             │
                    │  - Logs Archive              │
                    └──────────────────────────────┘
```

---

## 환경별 구성

### 1. Development (로컬)

**구성**:
- Docker Compose
- PostgreSQL 컨테이너
- Redis 컨테이너
- 로컬 파일 시스템

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  postgres:
    image: timescale/timescaledb:latest-pg14
    environment:
      POSTGRES_DB: stockindicator_dev
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  api:
    build: ./backend
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis
    environment:
      DATABASE_URL: postgresql://dev:dev@postgres:5432/stockindicator_dev
      REDIS_URL: redis://redis:6379/0
    volumes:
      - ./backend:/app

  web:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - api
    volumes:
      - ./frontend:/app

volumes:
  postgres_data:
```

---

### 2. Staging

**구성**:
- Kubernetes Cluster (2 nodes, t3.medium)
- RDS PostgreSQL (db.t3.small)
- ElastiCache Redis (cache.t3.micro)
- S3 Bucket

**리소스**:
| 컴포넌트 | CPU | Memory | Replicas |
|---------|-----|--------|----------|
| API | 500m | 512Mi | 2 |
| Web | 200m | 256Mi | 2 |
| Worker | 500m | 512Mi | 1 |

---

### 3. Production

**구성**:
- Kubernetes Cluster (5+ nodes, t3.large)
- RDS PostgreSQL (db.r5.xlarge, Multi-AZ)
- Read Replicas x2
- ElastiCache Redis (cache.r5.large)
- S3 Bucket (versioning enabled)
- CloudFront CDN

**리소스**:
| 컴포넌트 | CPU | Memory | Replicas | Auto-scaling |
|---------|-----|--------|----------|--------------|
| API | 1000m | 1Gi | 5 | 5-20 |
| Web | 500m | 512Mi | 3 | 3-10 |
| Worker | 1000m | 1Gi | 3 | 3-10 |

**Auto-scaling 조건**:
- CPU > 70% → Scale up
- CPU < 30% (10분 지속) → Scale down
- Request rate > 500/min → Scale up

---

## Kubernetes 구성

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: stockindicator-prod
```

---

### API Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: stockindicator-prod
spec:
  replicas: 5
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: stockindicator/api:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        resources:
          requests:
            cpu: 1000m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 2Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: stockindicator-prod
spec:
  selector:
    app: api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

---

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: stockindicator-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 5
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

### Ingress (HTTPS)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: stockindicator-prod
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.stockindicator.com
    secretName: api-tls
  rules:
  - host: api.stockindicator.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

## 데이터베이스 구성

### RDS PostgreSQL (Production)

**인스턴스 타입**: `db.r5.xlarge`
**스토리지**: 500GB GP3 (IOPS 12000)
**Multi-AZ**: Enabled
**Automated Backups**: 7일 보관
**Read Replicas**: 2개 (다른 AZ)

**파라미터 그룹**:
```
max_connections = 500
shared_buffers = 8GB
effective_cache_size = 24GB
work_mem = 64MB
maintenance_work_mem = 2GB
```

---

### TimescaleDB 설정

```sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Create hypertables
SELECT create_hypertable('ohlcv', 'date', chunk_time_interval => INTERVAL '1 month');
SELECT create_hypertable('indicator_values', 'date', chunk_time_interval => INTERVAL '1 month');

-- Set retention policies
SELECT add_retention_policy('ohlcv', INTERVAL '10 years');
SELECT add_retention_policy('indicator_values', INTERVAL '3 years');

-- Create continuous aggregates (optional)
CREATE MATERIALIZED VIEW ohlcv_daily
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 day', date) AS day,
       ticker_id,
       first(open, date) AS open,
       max(high) AS high,
       min(low) AS low,
       last(close, date) AS close,
       sum(volume) AS volume
FROM ohlcv
GROUP BY day, ticker_id;
```

---

## 배포 절차

### 1. CI/CD 파이프라인 (GitHub Actions)

**Workflow**: `.github/workflows/deploy-prod.yml`

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        run: |
          aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}

      - name: Build and push Docker image
        run: |
          docker build -t ${{ secrets.ECR_REGISTRY }}/api:${{ github.sha }} .
          docker push ${{ secrets.ECR_REGISTRY }}/api:${{ github.sha }}

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --name stockindicator-prod --region us-east-1

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/api api=${{ secrets.ECR_REGISTRY }}/api:${{ github.sha }} -n stockindicator-prod
          kubectl rollout status deployment/api -n stockindicator-prod

      - name: Run smoke tests
        run: |
          ./scripts/smoke-tests.sh https://api.stockindicator.com
```

---

### 2. Blue-Green Deployment

**전략**:
1. Green 환경에 새 버전 배포
2. Green 환경에서 smoke test 실행
3. 트래픽을 Blue → Green으로 전환 (Canary: 10% → 50% → 100%)
4. Blue 환경 모니터링 (5분)
5. 이상 없으면 Blue 환경 제거

---

### 3. Rollback 절차

**자동 롤백 조건**:
- Health check 실패
- Error rate > 10%
- P95 latency > 10초

**수동 롤백**:
```bash
# 이전 버전으로 롤백
kubectl rollout undo deployment/api -n stockindicator-prod

# 특정 버전으로 롤백
kubectl rollout undo deployment/api --to-revision=3 -n stockindicator-prod
```

---

## 모니터링 및 알럿

### Prometheus + Grafana

**메트릭 수집**:
- API 응답 시간 (Histogram)
- 요청 처리량 (Counter)
- 에러율 (Counter)
- Database 연결 수 (Gauge)
- Redis 히트율 (Gauge)

**Grafana 대시보드**:
- API Performance
- Database Metrics
- System Resources
- Business Metrics (스캔 수, 사용자 수)

---

### 알럿 규칙

```yaml
groups:
- name: api_alerts
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High error rate detected"

  - alert: SlowResponseTime
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 5
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "API response time is slow"

  - alert: DatabaseConnectionPoolExhausted
    expr: pg_connection_pool_available < 10
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Database connection pool nearly exhausted"
```

---

## 운영 체크리스트

### 배포 전 (Pre-Deployment)

- [ ] 모든 테스트 통과 (Unit, Integration, E2E)
- [ ] Staging 환경에서 검증 완료
- [ ] Database 마이그레이션 스크립트 준비
- [ ] Rollback 계획 수립
- [ ] 이해관계자에게 배포 공지

---

### 배포 중 (During Deployment)

- [ ] Database 마이그레이션 실행 (필요 시)
- [ ] 새 버전 배포
- [ ] Health check 확인
- [ ] Smoke tests 실행
- [ ] 트래픽 점진적 전환

---

### 배포 후 (Post-Deployment)

- [ ] 에러율 모니터링 (30분)
- [ ] 응답 시간 확인
- [ ] 사용자 피드백 수집
- [ ] 로그 확인
- [ ] Post-Mortem 작성 (문제 발생 시)

---

## 재해 복구 (Disaster Recovery)

### RTO/RPO

| 시나리오 | RTO | RPO |
|---------|-----|-----|
| 단일 서버 장애 | 5분 | 0 (무손실) |
| Database 장애 | 15분 | 5분 |
| AZ 전체 장애 | 30분 | 5분 |
| Region 전체 장애 | 4시간 | 1시간 |

---

### Backup 전략

**자동 백업**:
- RDS: 일일 스냅샷 (7일 보관)
- S3: Versioning enabled
- 설정 파일: Git 버전 관리

**수동 백업 (주요 배포 전)**:
```bash
# Database backup
pg_dump -h <rds-endpoint> -U admin stockindicator_prod > backup_$(date +%Y%m%d).sql

# Upload to S3
aws s3 cp backup_$(date +%Y%m%d).sql s3://stockindicator-backups/
```

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 0.1  | 2025-11-21 | 초안 작성 | DevOps Team |

---

**다음 단계**: Terraform 코드 작성 및 인프라 프로비저닝
