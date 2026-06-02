# 🧪 PANDUAN TESTING - API GATEWAY

**Project**: Tugas Besar RPL 2 — Ekosistem Ekonomi UMKM  
**Aplikasi**: API Gateway / Integrator  
**Kelompok**: 7.0

---

## 📋 DAFTAR ISI

1. [Setup Environment](#setup-environment)
2. [Testing Backend](#testing-backend)
3. [Testing Frontend](#testing-frontend)
4. [Integration Testing](#integration-testing)
5. [15 Test Scenarios dari PRD](#15-test-scenarios-dari-prd)
6. [Performance Testing](#performance-testing)

---

## 🔧 SETUP ENVIRONMENT

### Prerequisites
```bash
# Install dependencies
- MySQL 8.0+
- Redis 7.0+
- Go 1.25.4+
- Node.js 18+
- npm atau yarn
```

### 1. Setup Database
```bash
# Login ke MySQL
mysql -u root -p

# Create database
CREATE DATABASE integrator_gateway;

# Import schema
mysql -u root -p integrator_gateway < backend/migrations/001_init_schema.sql

# Verify tables
USE integrator_gateway;
SHOW TABLES;
# Expected: routes_registry, request_logs, gateway_fees, jwt_blacklist
```

### 2. Setup Redis
```bash
# Start Redis server
redis-server

# Verify Redis is running
redis-cli ping
# Expected: PONG
```

### 3. Setup Backend
```bash
cd backend

# Set environment variables
export DB_HOST=localhost
export DB_PORT=3306
export DB_USER=root
export DB_PASSWORD=your_password
export DB_NAME=integrator_gateway
export REDIS_ADDR=localhost:6379
export JWKS_URL=http://localhost:8001/.well-known/jwks.json
export PORT=8080

# Build
go build -o gateway cmd/gateway/main.go

# Run
./gateway
# Expected: Server listening on :8080
```

### 4. Setup Frontend
```bash
cd frontend

# Install dependencies
npm install

# Create .env file
echo "VITE_API_URL=http://localhost:8080" > .env
echo "VITE_ENV=DEV" >> .env

# Run development server
npm run dev
# Expected: Server running on http://localhost:5173
```

---

## 🔍 TESTING BACKEND

### 1. Health Check Endpoints

```bash
# Health check
curl http://localhost:8080/health
# Expected: {"status":"healthy"}

# Readiness check
curl http://localhost:8080/ready
# Expected: {"status":"ready","database":"connected","redis":"connected"}

# Info endpoint
curl http://localhost:8080/info
# Expected: {"service":"integrator-gateway","version":"1.0.0",...}
```

### 2. Route Registry Endpoints

```bash
# Get all routes
curl http://localhost:8080/integrator/routes

# Expected response:
{
  "success": true,
  "data": [
    {
      "id": 1,
      "service_name": "SmartBank",
      "feature_name": "pembayaran_transaksi",
      "method": "POST",
      "downstream_url": "http://smartbank:8001/api/payment",
      "timeout_ms": 5000,
      "max_retries": 0,
      "is_transactional": true,
      "is_active": true,
      "required_scope": "payment:write"
    }
  ]
}
```

### 3. Logging Endpoints

```bash
# Get logs
curl "http://localhost:8080/integrator/logging?limit=10"

# Get log statistics
curl http://localhost:8080/integrator/logging/stats

# Expected response:
{
  "success": true,
  "data": {
    "total_requests": 12847,
    "error_rate": 0.38,
    "p95_latency_ms": 42,
    "hourly_requests": [...],
    "top_services": [...]
  }
}
```

### 4. Fee Endpoints

```bash
# Get fees
curl "http://localhost:8080/integrator/biaya_layanan_integrasi?status=PENDING"

# Get fee statistics
curl http://localhost:8080/integrator/biaya_layanan_integrasi/stats

# Expected response:
{
  "success": true,
  "data": {
    "total_revenue": 284500,
    "collection_rate": 99.2,
    "pending_count": 4,
    "revenue_trend": [...],
    "revenue_by_source": [...]
  }
}
```

### 5. Circuit Breaker Endpoints

```bash
# Get circuit states
curl http://localhost:8080/integrator/circuits

# Expected response:
{
  "success": true,
  "data": [
    {
      "service_name": "SmartBank",
      "state": "CLOSED",
      "avg_latency_ms": 12,
      "p95_latency_ms": 28,
      "uptime_pct": 99.98,
      "requests_24h": 2100,
      "errors_24h": 2
    }
  ]
}
```

### 6. Token Introspection

```bash
# Validate JWT token
curl -X POST http://localhost:8080/integrator/validasi_request \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"

# Expected response:
{
  "active": true,
  "user_id": "user_123",
  "roles": ["Operator"],
  "scopes": ["payment:read", "payment:write"],
  "exp": 1735689600
}
```

---

## 🎨 TESTING FRONTEND

### 1. Manual UI Testing

#### Login Page
1. Buka http://localhost:5173
2. Masukkan email dan password
3. Klik "Sign In"
4. Verify: Redirect ke Dashboard
5. Verify: Token tersimpan di localStorage

#### Dashboard
1. Verify: KPI tiles menampilkan data real
2. Verify: Throughput chart menampilkan data 24 jam
3. Verify: Top Services chart menampilkan data
4. Verify: Circuit Breakers table menampilkan status
5. Verify: Recent Errors table menampilkan error logs
6. Verify: Auto-refresh setiap 30 detik

#### Request Logs
1. Verify: Table menampilkan logs dari API
2. Test: Filter by status (200, 401, 429, 502, 503)
3. Test: Filter by service (PasarKita, WarungPOS, dll)
4. Test: Filter by lifecycle (STARTED, COMPLETED, FAILED)
5. Test: Search by request_id atau user_id
6. Test: Click row untuk detail drawer
7. Verify: Refresh button reload data

#### Gateway Fees
1. Verify: Summary cards menampilkan revenue data
2. Test: Period selector (Hari, Minggu, Bulan)
3. Verify: Revenue trend chart update sesuai period
4. Verify: Revenue by source pie chart
5. Verify: Top users list
6. Verify: Pending/Failed fees table
7. Test: Retry button untuk failed fees
8. Test: Export CSV button

#### Route Registry
1. Verify: Service tabs dari API data
2. Verify: Routes table per service
3. Test: Click Edit button
4. Test: Modify route settings
5. Test: Save changes
6. Test: Toggle active/inactive dengan confirmation
7. Verify: Changes reflected di table

#### Service Health
1. Verify: Service cards menampilkan health status
2. Verify: Circuit breaker states (CLOSED, OPEN, HALF-OPEN)
3. Verify: Latency metrics (p50, p95, p99)
4. Verify: Uptime percentage
5. Verify: Circuit breaker timeline
6. Verify: Rate limit buckets visualization
7. Test: Force probe button
8. Verify: Auto-refresh setiap 10 detik

### 2. Browser Console Testing

```javascript
// Check API service
console.log(localStorage.getItem('auth_token'));

// Test API calls
fetch('http://localhost:8080/health')
  .then(r => r.json())
  .then(console.log);

// Check for errors
// Should see no console errors
```

---

## 🔗 INTEGRATION TESTING

### Scenario 1: End-to-End Request Flow

```bash
# 1. Login (get JWT token)
TOKEN=$(curl -X POST http://localhost:8001/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"operator@gateway.com","password":"password"}' \
  | jq -r '.token')

# 2. Make transactional request through gateway
curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d '{
    "user_id": "user_123",
    "amount": 50000,
    "recipient": "merchant_456"
  }'

# 3. Verify log created
curl "http://localhost:8080/integrator/logging?limit=1"

# 4. Verify fee charged
curl "http://localhost:8080/integrator/biaya_layanan_integrasi?limit=1"
# Expected: Fee = 50000 * 0.005 = 250
```

### Scenario 2: Idempotency Testing

```bash
# Send same request twice with same idempotency key
IDEM_KEY=$(uuidgen)

# First request
curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Idempotency-Key: $IDEM_KEY" \
  -d '{"user_id":"user_123","amount":50000}'

# Second request (should return cached response)
curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Idempotency-Key: $IDEM_KEY" \
  -d '{"user_id":"user_123","amount":50000}'

# Expected: Same response, no duplicate fee charged
```

### Scenario 3: Rate Limiting Testing

```bash
# Send 11 transactional requests rapidly (limit: 10/min)
for i in {1..11}; do
  curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
    -H "Authorization: Bearer $TOKEN" \
    -H "X-Idempotency-Key: $(uuidgen)" \
    -d '{"user_id":"user_123","amount":1000}'
  echo "Request $i"
done

# Expected: 11th request returns 429 RATE_LIMITED
```

### Scenario 4: Circuit Breaker Testing

```bash
# Simulate downstream failure (5x 502 errors)
# This requires mocking SmartBank to return 502

# After 5 failures, circuit should OPEN
# Verify circuit state
curl http://localhost:8080/integrator/circuits | jq '.data[] | select(.service_name=="SmartBank")'

# Expected: state = "OPEN"
```

---

## ✅ 15 TEST SCENARIOS DARI PRD

### 1. Transparent Routing
```bash
curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"amount":50000}'
# Expected: 200 OK, routed to SmartBank
```

### 2. Envelope Routing
```bash
curl -X POST http://localhost:8080/integrator/routing_api \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "service_name": "SmartBank",
    "feature_name": "pembayaran_transaksi",
    "payload": {"amount":50000}
  }'
# Expected: 200 OK
```

### 3. JWT Validation (RS256)
```bash
curl http://localhost:8080/api/v1/SmartBank/manajemen_saldo \
  -H "Authorization: Bearer INVALID_TOKEN"
# Expected: 401 AUTH_INVALID_TOKEN
```

### 4. JWT Blacklist Check
```bash
# Revoke token in Redis
redis-cli SET "jwt:blacklist:jti_123" "1" EX 3600

# Try to use revoked token
curl http://localhost:8080/api/v1/SmartBank/manajemen_saldo \
  -H "Authorization: Bearer REVOKED_TOKEN"
# Expected: 401 AUTH_INVALID_TOKEN
```

### 5. Scope Validation
```bash
# Token without payment:write scope
curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
  -H "Authorization: Bearer TOKEN_WITHOUT_SCOPE" \
  -d '{"amount":50000}'
# Expected: 403 AUTH_SCOPE_DENIED
```

### 6. PII Redaction in Logs
```bash
# Send request with PII
curl -X POST http://localhost:8080/api/v1/PasarKita/checkout \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "email": "user@example.com",
    "phone": "081234567890",
    "ktp": "1234567890123456"
  }'

# Check logs
curl "http://localhost:8080/integrator/logging?limit=1"
# Expected: PII fields show [REDACTED]
```

### 7. Idempotency Enforcement
```bash
# Missing idempotency key
curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"amount":50000}'
# Expected: 400 IDEMPOTENCY_KEY_REQUIRED
```

### 8. Idempotency Conflict Detection
```bash
IDEM_KEY=$(uuidgen)

# First request
curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Idempotency-Key: $IDEM_KEY" \
  -d '{"amount":50000}'

# Second request with different body
curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Idempotency-Key: $IDEM_KEY" \
  -d '{"amount":60000}'
# Expected: 409 IDEMPOTENCY_REPLAY
```

### 9. Fee Charging (0.5%)
```bash
curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d '{"amount":100000}'

# Check fee
curl "http://localhost:8080/integrator/biaya_layanan_integrasi?limit=1"
# Expected: fee = 500 (100000 * 0.005)
```

### 10. Rate Limiting (Transactional)
```bash
# Send 11 requests (limit: 10/min)
for i in {1..11}; do
  curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
    -H "Authorization: Bearer $TOKEN" \
    -H "X-Idempotency-Key: $(uuidgen)" \
    -d '{"amount":1000}'
done
# Expected: 11th request returns 429
```

### 11. Rate Limiting (Read)
```bash
# Send 61 GET requests (limit: 60/min)
for i in {1..61}; do
  curl http://localhost:8080/api/v1/SmartBank/manajemen_saldo \
    -H "Authorization: Bearer $TOKEN"
done
# Expected: 61st request returns 429
```

### 12. Circuit Breaker (OPEN)
```bash
# Simulate 5 consecutive 502 errors
# Circuit should OPEN after 5th failure
# Subsequent requests should return 503 CIRCUIT_OPEN
```

### 13. Circuit Breaker (HALF-OPEN)
```bash
# Wait 60 seconds after circuit OPEN
# Next request should trigger HALF-OPEN state
# If successful, circuit returns to CLOSED
```

### 14. Request Lifecycle Logging
```bash
# Make request
curl -X POST http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d '{"amount":50000}'

# Check logs
curl "http://localhost:8080/integrator/logging?limit=1"
# Expected: lifecycle = COMPLETED, latency_ms > 0
```

### 15. Token Introspection
```bash
curl -X POST http://localhost:8080/integrator/validasi_request \
  -H "Authorization: Bearer $TOKEN"
# Expected: {active:true, user_id, roles, scopes, exp}
```

---

## ⚡ PERFORMANCE TESTING

### Load Testing dengan Apache Bench

```bash
# Install Apache Bench
sudo apt-get install apache2-utils

# Test health endpoint (baseline)
ab -n 1000 -c 10 http://localhost:8080/health

# Test with authentication
ab -n 1000 -c 10 -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/SmartBank/manajemen_saldo

# Target: > 500 req/s
# Expected p95 latency: < 100ms
```

### Load Testing dengan wrk

```bash
# Install wrk
sudo apt-get install wrk

# Test throughput
wrk -t4 -c100 -d30s http://localhost:8080/health

# Test with POST requests
wrk -t4 -c100 -d30s -s post.lua http://localhost:8080/api/v1/SmartBank/pembayaran_transaksi

# post.lua:
# wrk.method = "POST"
# wrk.headers["Authorization"] = "Bearer TOKEN"
# wrk.headers["Content-Type"] = "application/json"
# wrk.body = '{"amount":50000}'
```

### Stress Testing

```bash
# Gradually increase load
for c in 10 50 100 200 500; do
  echo "Testing with $c concurrent connections"
  ab -n 10000 -c $c http://localhost:8080/health
done

# Monitor:
# - Response time degradation
# - Error rate increase
# - Memory usage
# - CPU usage
```

---

## 📊 EXPECTED RESULTS

### Performance Targets
- **p50 latency overhead**: < 30ms ✅
- **p95 latency overhead**: < 100ms ✅
- **p99 latency overhead**: < 250ms ✅
- **Throughput**: > 500 req/s ⚠️ (needs verification)
- **Error rate**: < 0.5% ✅
- **Uptime**: > 99.9% ✅

### Functional Requirements
- ✅ All 15 test scenarios pass
- ✅ JWT RS256 validation works
- ✅ Idempotency enforcement works
- ✅ Rate limiting works
- ✅ Circuit breaker works
- ✅ Fee charging works (0.5%)
- ✅ PII redaction works
- ✅ Logging works

### UI/UX Requirements
- ✅ All 9 pages render correctly
- ✅ Real API data flows to frontend
- ✅ Auto-refresh works
- ✅ Error handling works
- ✅ Loading states work
- ✅ Modal confirmations work

---

## 🐛 TROUBLESHOOTING

### Backend tidak start
```bash
# Check MySQL connection
mysql -u root -p -e "SELECT 1"

# Check Redis connection
redis-cli ping

# Check port availability
lsof -i :8080
```

### Frontend tidak connect ke backend
```bash
# Check CORS settings
# Backend should allow http://localhost:5173

# Check .env file
cat frontend/.env
# Should have: VITE_API_URL=http://localhost:8080
```

### API returns 401
```bash
# Check JWT token
echo $TOKEN | jwt decode -

# Check token expiration
# Regenerate token if expired
```

### Database errors
```bash
# Check schema
mysql -u root -p integrator_gateway -e "SHOW TABLES"

# Re-run migrations if needed
mysql -u root -p integrator_gateway < backend/migrations/001_init_schema.sql
```

---

**Happy Testing!** 🚀
