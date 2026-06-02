# 🔐 Operator Accounts Documentation

## Setup Akun Operator

File seeder telah membuat 4 akun operator untuk testing di Integrator Console.

### 🚀 Langkah Setup

#### 1. Import Schema + Seeder ke MySQL
```bash
# Login ke MySQL
mysql -u root -p

# Jalankan migration
mysql -u root -p integrator_gateway < backend/migrations/001_init_schema.sql
mysql -u root -p integrator_gateway < backend/migrations/002_create_operators_table.sql
mysql -u root -p integrator_gateway < backend/migrations/003_seed_operators_and_data.sql

# Verifikasi
mysql -u root -p integrator_gateway -e "SELECT email, role FROM operators;"
```

#### 2. Alternate: Satu Command
```bash
# Jalankan semua migration sekaligus
cat backend/migrations/001_init_schema.sql \
    backend/migrations/002_create_operators_table.sql \
    backend/migrations/003_seed_operators_and_data.sql | mysql -u root -p integrator_gateway
```

#### 3. Verifikasi Data Tersimpan
```bash
mysql -u root -p integrator_gateway << EOF
SELECT email, role, is_active FROM operators;
SELECT COUNT(*) as total_logs FROM request_logs;
SELECT COUNT(*) as total_fees FROM gateway_fees;
SELECT SUM(fee_amount) as total_revenue FROM gateway_fees WHERE status='PAID';
EOF
```

---

## 👥 Available Operator Accounts

Semua akun menggunakan password: **`password123`**

| Email | Password | Role | Permissions | Status |
|-------|----------|------|-------------|--------|
| operator@gateway.com | password123 | **Operator** | View logs, fees, health | ✅ Active |
| auditor@gateway.com | password123 | **FinanceAuditor** | View fees, export CSV, reconciliation | ✅ Active |
| admin@gateway.com | password123 | **AdminFull** | Full access + 2FA required | ✅ Active (2FA) |
| viewer@gateway.com | password123 | **ReadOnlyViewer** | Read-only access to all | ✅ Active |

---

## 🔑 Role Permissions Matrix

| Action | Operator | FinanceAuditor | AdminFull | ReadOnlyViewer |
|--------|----------|-----------------|-----------|----------------|
| View Dashboard | ✅ | ✅ | ✅ | ✅ |
| View Logs | ✅ | ✅ | ✅ | ✅ |
| View Fees | ✅ | ✅ | ✅ | ✅ |
| Export CSV | ❌ | ✅ | ✅ | ❌ |
| Retry Fees | ✅ | ✅ | ✅ | ❌ |
| Edit Routes | ✅ | ❌ | ✅ | ❌ |
| Rotate JWT Key | ❌ | ❌ | ✅ | ❌ |
| Force Circuit Probe | ✅ | ❌ | ✅ | ❌ |
| Manage Operators | ❌ | ❌ | ✅ | ❌ |
| View Audit Log | ✅ | ✅ | ✅ | ❌ |

---

## 📊 Test Data Included

### Request Logs
- **Total Requests**: 29
- **Success (200)**: 25
- **Failed (429, 502, 503, 401)**: 4
- **Time Range**: Last 7 days (2026-05-14 to 2026-05-20)

### Gateway Fees
- **Total Fee Records**: 29
- **PAID**: 25 (Total: Rp 26,615)
- **PENDING**: 1 (Rp 1,000)
- **DEFERRED**: 1 (Rp 600)
- **FAILED**: 1 (Rp 475)
- **Collection Rate**: ~87.5%

---

## 🧪 Testing Login

### Using Frontend Console

1. **Start Backend**:
```bash
cd backend
go run cmd/gateway/main.go
```

2. **Start Frontend**:
```bash
cd frontend
npm run dev
```

3. **Open Browser**:
```
http://localhost:5173
```

4. **Login dengan salah satu akun**:
```
Email: operator@gateway.com
Password: password123
```

5. **Explore Pages**:
   - ✅ Dashboard → Lihat KPI dan charts
   - ✅ Request Logs → Filter dan detail logs
   - ✅ Gateway Fees → Lihat revenue dan pending fees
   - ✅ Route Registry → Lihat daftar route
   - ✅ Service Health → Circuit breaker status
   - ✅ Security & JWT → Key management
   - ✅ Settings → Account settings

---

## 🔒 Password Management (Future Improvement)

### Untuk Development Sekarang
Menggunakan bcrypt hash (untuk semua akun):
```
Hash: $2a$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5YmMxSUnnMuFm
Plain: password123
```

### Untuk Production
Perlu implementasi:
1. **Proper Password Hashing**: Bcrypt dengan cost=12+ atau Argon2
2. **Secure Password Reset**: OTP via email
3. **Login Attempt Limiting**: Max 5 failures per 15 menit
4. **Session Management**: Secure cookies + HTTPS
5. **2FA Enforcement**: TOTP untuk AdminFull role

---

## 📝 Database Schema

### Operators Table
```sql
- id: BIGINT (PK)
- email: VARCHAR (UNIQUE)
- password_hash: VARCHAR
- full_name: VARCHAR
- role: ENUM ('Operator', 'FinanceAuditor', 'AdminFull', 'ReadOnlyViewer')
- is_active: BOOLEAN
- requires_2fa: BOOLEAN
- totp_secret: VARCHAR (nullable)
- last_login_at: TIMESTAMP
- failed_login_attempts: INT
- locked_until: TIMESTAMP
```

### Operator Sessions Table
```sql
- id: BIGINT (PK)
- operator_id: BIGINT (FK)
- session_token: VARCHAR (UNIQUE)
- ip_address: VARCHAR
- user_agent: TEXT
- expires_at: TIMESTAMP
- created_at: TIMESTAMP
```

### Operator Audit Log Table
```sql
- id: BIGINT (PK)
- operator_id: BIGINT (FK)
- action: VARCHAR
- resource_type: VARCHAR
- resource_id: VARCHAR
- changes: JSON
- ip_address: VARCHAR
- status: ENUM ('SUCCESS', 'FAILURE')
```

---

## ⚡ Quick Reference

### Check All Operators
```bash
mysql -u root -p integrator_gateway -e "SELECT id, email, role, is_active FROM operators;"
```

### Reset a Password (manual)
```bash
# Generate new bcrypt hash via backend
go run -c 'import "golang.org/x/crypto/bcrypt"; hash, _ := bcrypt.GenerateFromPassword([]byte("newpass"), 12); println(string(hash))'

# Update in DB
mysql -u root -p integrator_gateway -e "UPDATE operators SET password_hash='NEW_HASH' WHERE email='operator@gateway.com';"
```

### Count Test Data
```bash
mysql -u root -p integrator_gateway << EOF
SELECT 'Request Logs' as type, COUNT(*) as count FROM request_logs
UNION ALL
SELECT 'Gateway Fees', COUNT(*) FROM gateway_fees
UNION ALL
SELECT 'Operators', COUNT(*) FROM operators
UNION ALL
SELECT 'Routes', COUNT(*) FROM routes_registry;
EOF
```

---

## 🆘 Troubleshooting

### "Operator not found" during login
1. Verifikasi migration 002 dan 003 sudah dijalankan
2. Check: `SELECT COUNT(*) FROM operators;`
3. Re-run: `mysql -u root -p integrator_gateway < backend/migrations/003_seed_operators_and_data.sql`

### "Wrong password" error
1. Pastikan password: `password123` (case-sensitive)
2. Verifikasi hash di DB sama

### 2FA Required Error
- Login dengan akun `admin@gateway.com` perlu 2FA TOTP
- Untuk testing, gunakan akun lain (Operator, FinanceAuditor, ReadOnlyViewer)

---

**Last Updated**: 2026-05-20  
**Status**: ✅ Ready for Testing
