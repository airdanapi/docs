# 10. Implementasi Algoritma API Gateway

Dokumen ini menjelaskan *walkthrough* dan rincian implementasi 6 kelompok algoritma utama yang diterapkan pada sistem Integrator Gateway, merujuk pada rencana implementasi (Plan Implementasi Algoritma).

## Ringkasan Algoritma

| Algoritma | Modul | Deskripsi |
|-----------|-------|-----------|
| **String Matching** | PII Redaction | Menggunakan regular expression (regex) untuk mendeteksi dan menutupi data sensitif (email, telepon, NIK) dalam *value* atau nilai dari payload request sebelum dicatat dalam log. |
| **BFS / DFS** | Request Tracing | Menelusuri log request secara Depth-First Search (DFS) atau Breadth-First Search (BFS) berdasarkan relasi `parent_request_id` untuk visualisasi *trace tree* hierarkis dan deteksi *circular routing* (siklus request). |
| **Divide & Conquer** | Dashboard Metrics & Fee Agregation | Memecah data query log yang besar berdasarkan dimensi seperti rentang waktu, status, dan service. Menghitung metrik agregat independen seperti persentil latensi (p50/p95/p99) dan tren pendapatan. |
| **Decrease & Conquer** | Fee Retry (Exponential Backoff) | Menangani tagihan fee yang gagal. Waktu tunggu antar-retries (*backoff delay*) terus ditingkatkan (30s, 2m, 8m, 32m, 2h) sementara sisa kuota retry diturunkan hingga mencapai batas (5 retries). |
| **Backtracking** | Route Config Validation | Memvalidasi konfigurasi route dengan melihat *constraint* (konflik, timeout, limit retry). Mensimulasikan pembaruan ke dalam memori; jika tidak valid, pembaruan di-*rollback* dan dibatalkan. Jika sesuai, di-*commit*. |
| **Greedy** | Rate Limiting & Circuit Breaker | *(Sudah diimplementasi sebelumnya)* Algoritma token-bucket memotong request berlebih di saat itu juga (lokal maksimum), dan status breaker berubah seketika ketika error rate melewati batas toleransi. |

---

## Detail Implementasi per Modul

### 1. PII Redaction Enhancement (String Matching)
- **File Modifikasi:** `backend/internal/middleware/logging.go`
- **Konsep:**
  Sistem sebelumnya hanya menutupi (redact) field yang memiliki kemiripan nama dengan pola PII (misal: "email", "address"). Sekarang, lapisan kedua (**Layer 2**) diaktifkan: setiap nilai bertipe string diperiksa dengan regex untuk mendeteksi format sensitif meskipun nama field-nya aman.
  - Format Email → `[REDACTED_EMAIL]`
  - Format Telepon ID (+62/08...) → `[REDACTED_PHONE]`
  - Format NIK/Identity (16 digit) → `[REDACTED_ID]`

### 2. Request Tracing & Circular Detection (BFS/DFS)
- **File Kunci:** 
  - `backend/internal/service/tracing.go` (Baru)
  - `backend/internal/handler/logs.go`
- **Konsep:**
  - Middleware secara otomatis meneruskan HTTP Header `X-Parent-Request-Id` menuju *downstream* service agar rantai panggilan request saling terkait di database log.
  - Menyediakan endpoint investigasi `GET /integrator/logging/trace?request_id=xxx`.
  - **DFS Trace:** Menggunakan fungsi rekursif *depth-first* yang berjalan sampai kedalaman daun (leaf). Menyimpan jejak `visited` dan `path` untuk mendeteksi *cycle* atau putaran tidak berujung (Circular Routing).
  - **BFS Trace:** Menggunakan struktur antrian (Queue) untuk menelusuri log berdasarkan tingkat kedalaman yang sama (per level) sebelum masuk ke level di bawahnya.

### 3. Dashboard Metrics & Query Partitions (Divide & Conquer)
- **File Kunci:** `backend/internal/repository/log_repo.go`
- **Konsep:**
  - **Latensi Persentil:** Pendekatan membagi dataset. Dengan menjumlah row berdasarkan filter, tabel diurutkan (`ORDER BY`), lalu offset diambil (pada indeks 50%, 95%, dan 99%) untuk mendapatkan skor latensi sejati.
  - **Statistik Service:** Query memecah (`GROUP BY`) hasil berdasarkan `target_app` guna memastikan performa per service transparan.
  - **Filter Waktu:** Menggunakan `from` & `to` untuk mempersempit pindaian tabel sehingga performa database tetap ringan meskipun jumlah baris puluhan juta.

### 4. Agregasi Real-Time Fee Revenue (Divide & Conquer)
- **File Kunci:** `backend/internal/repository/fee_repo.go`, `backend/internal/handler/fees.go`
- **Konsep:**
  Tampilan dasbor pendapatan fee sekarang menggunakan agregasi murni, bukan *mock data*:
  - Tren pendapatan (Trend): Dipecah ke dalam interval (harian, mingguan, bulanan).
  - Pendapatan per *source_app*: Menghubungkan (JOIN) tabel tagihan fee dengan log pemanggilan API asli untuk menelisik *source app* mana yang menghasilkan uang paling banyak.
  - Top user: Mengelompokkan transaksi per pengguna, dijumlah total sumbangan pendapatannya.

### 5. Fee Retry & Exponential Backoff (Decrease & Conquer)
- **File Kunci:** `backend/internal/service/fee.go`, `backend/migrations/004_add_next_retry_at.sql`
- **Konsep:**
  - Ditambahkan properti `next_retry_at` pada *gateway_fees* agar cron worker mengabaikan record hingga waktu tiba.
  - Iterasi tagihan diatur berkurang limitnya (sisa mencoba dikurangkan 1).
  - Diikuti jeda tunggu eksponensial (Backoff Delay): `30s -> 2m -> 8m -> 32m -> 2h`.
  - Apabila retries habis, iterasi disetop penuh, dan status dipindah ke `FAILED`.

### 6. Route Configuration Validation (Backtracking)
- **File Kunci:** `backend/internal/handler/routes.go`, `backend/internal/repository/route_repo.go`
- **Konsep:**
  - Menerapkan cek limitasi (misalnya tidak boleh ada route tumpang-tindih untuk endpoint `service + feature + method` yang persis sama).
  - Mengecek ambang batas batas seperti `timeout_ms` (harus 100-30000) dan `max_retries` (0-10).
  - Saat modifikasi (update route), konfigurasi awal dibaca terlebih dahulu. Jika melanggar satupun *constraint* di atas, eksekusi dibatalkan seketika (*rolled back*), membuang hasil manipulasi in-memory dan menampilkan error API spesifik. Jika lolos seluruh syarat, dilanjutkan dengan persistensi (*commit*) ke database.

---

## Kesimpulan

Penerapan beragam tipe algoritma di dalam kode API Gateway memastikan seluruh pemrosesan bekerja optimal, *error tolerant*, memiliki penanganan performa (via partisi dan query cermat), sekaligus mendukung dokumentasi jejak rekam terpusat yang aman dari eksposur data pengguna berkat modifikasi String Matching. Semua fungsionalitas ini telah stabil dan melalui proses `go vet` serta `go build` yang sempurna tanpa peringatan kompilasi.
