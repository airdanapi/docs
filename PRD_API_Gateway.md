# Product Requirements Document
## API Gateway / Integrator — Ekosistem Ekonomi UMKM

---

## Document Control

| Field | Value |
|---|---|
| Document Title | PRD — API Gateway / Integrator |
| Project | Tugas Besar RPL 2 — Ekosistem Ekonomi UMKM |
| Application Code | Kelompok 7.0 |
| Document Version | 1.0 |
| Document Status | Final draft, ready for review |
| Document Date | 29 April 2026 |
| Author | Tim Kelompok 7.0 |
| Reviewer | M. Yusril Helmi Setyawan, S.Kom., M.Kom. |
| Related Documents | `api_gateway_blueprint.md`, `stitch_prompts.md`, `stitch_prompts_addendum.md`, `stitch_sidebar_refactor.md`, lima diagram Graphviz workflow |

---

## 1. Executive Summary

API Gateway / Integrator adalah komponen *infrastructure middleware* yang menjadi satu-satunya pintu masuk dan bidang orkestrasi untuk seluruh komunikasi antar tujuh aplikasi dalam Ekosistem Ekonomi UMKM (SmartBank, PasarKita, WarungPOS, SupplierHub, LogistiKita, UMKM Insight, dan Integrator itu sendiri). Gateway memberlakukan otentikasi terpusat, merutekan permintaan, mencatat seluruh jejak audit, menerapkan kontrol laju, dan memungut biaya layanan integrasi sebesar 0.5% per transaksi melalui SmartBank.

Gateway bukan aplikasi konsumer dan tidak memiliki *business logic* domain. Gateway menyediakan satu *API publik tunggal* yang setiap endpoint-nya memetakan ke API internal aplikasi domain. End-user UMKM (pembeli, kasir, pemilik warung) berinteraksi dengan ekosistem melalui frontend aplikasi domain masing-masing; Gateway hanya menjadi jalur transit bagi request HTTP yang dikirim oleh frontend tersebut menuju backend yang sesuai.

Komponen ini memiliki dua antarmuka berbeda: API publik yang dipanggil oleh frontend aplikasi domain, dan **Integrator Console** sebagai konsol operasional internal yang digunakan tim platform untuk memantau kesehatan Gateway, menelusuri log, mengelola registry route, dan merekonsiliasi pendapatan fee.

---

## 2. Strategic Context

### 2.1 Latar Belakang Ekosistem

Ekosistem Ekonomi UMKM adalah simulasi sistem ekonomi tertutup yang dibangun untuk Tugas Besar RPL 2 dengan tujuh aplikasi independen yang saling terintegrasi. Setiap aplikasi memiliki domain yang berbeda: SmartBank sebagai otoritas moneter pusat, PasarKita sebagai marketplace, WarungPOS sebagai sistem kasir, SupplierHub sebagai platform pengadaan, LogistiKita sebagai layanan logistik, UMKM Insight sebagai analitik *read-only*, dan Integrator sebagai gateway lalu lintas antar layanan.

Spesifikasi proyek menetapkan sepuluh aturan arsitektural fundamental (Sheet 4) yang membentuk batasan keras pada desain seluruh aplikasi. Tiga aturan paling berdampak pada Gateway adalah Aturan 4 (SmartBank sebagai otoritas moneter pusat), Aturan 5 (seluruh komunikasi antar aplikasi wajib melalui Gateway), dan Aturan 8 (uang tidak boleh diciptakan secara bebas).

### 2.2 Mengapa Pola Gateway

Tanpa Gateway terpusat, tujuh aplikasi akan membentuk *mesh integration* dengan kompleksitas O(n²) — setiap aplikasi harus mengetahui URL, mekanisme otentikasi, dan kontrak setiap aplikasi lain. Gateway mengubah topologi menjadi *hub-and-spoke* dengan kompleksitas O(n), sekaligus menyediakan tempat sentral untuk menegakkan otentikasi, mengumpulkan audit log, dan memungut biaya integrasi.

Posisi Gateway sebagai satu-satunya pemegang service registry juga mengisolasi perubahan: ketika satu aplikasi domain berpindah host, hanya satu baris di registry yang berubah, tanpa modifikasi pada enam aplikasi lain.

### 2.3 Sumber Pendapatan Gateway

Gateway memonetisasi dirinya sendiri melalui biaya layanan integrasi 0.5% per transaksi yang dipotong via SmartBank (Sheet 6 item 10). Mekanisme ini sejajar dengan biaya layanan aplikasi domain lain (marketplace 2%, POS 1%, supplier 3%, logistik 5%) dan memenuhi Aturan 9 yang menetapkan setiap layanan harus memiliki struktur fee.

---

## 3. Problem Statement

Tujuh aplikasi domain dalam ekosistem UMKM memerlukan mekanisme komunikasi antar layanan yang aman, terobservasi, dan dapat dikenakan biaya secara konsisten. Tanpa lapisan integrasi terpusat, setiap aplikasi akan mengimplementasikan ulang otentikasi, *rate limiting*, *audit logging*, dan integrasi pembayaran SmartBank — menghasilkan duplikasi kode, inkonsistensi keamanan, dan ketidakmampuan menelusuri transaksi secara end-to-end.

Lebih jauh, aturan ekosistem mensyaratkan bahwa setiap pergerakan moneter melewati SmartBank sebagai sumber kebenaran tunggal. Tanpa Gateway, setiap aplikasi domain harus secara mandiri memanggil SmartBank untuk pembayaran utama dan kemudian memanggilnya lagi untuk biaya layanan integrasi — pola yang rentan terhadap inkonsistensi *idempotency*, kehilangan trace, dan biaya yang tidak terpungut.

Gateway memecahkan masalah ini dengan menjadi pintu masuk tunggal yang menegakkan kontrak operasional yang seragam pada setiap permintaan, mencatat seluruh aktivitas dalam satu *audit trail* yang dapat dikorelasikan, dan secara otomatis memungut biaya 0.5% pada permintaan transaksional setelah pemrosesan downstream berhasil.

---

## 4. Goals dan Non-Goals

### 4.1 Primary Goals

1. **Routing terpadu**: Menyediakan satu endpoint publik yang merutekan permintaan ke aplikasi downstream yang tepat berdasarkan registry yang dapat dikonfigurasi.
2. **Otentikasi terpusat**: Memvalidasi JWT pada setiap permintaan dengan mekanisme yang seragam sebelum diteruskan ke downstream.
3. **Audit trail komprehensif**: Mencatat seluruh permintaan dan respons dengan korelasi `request_id` lintas hop, mendukung penelusuran end-to-end.
4. **Pemungutan fee otomatis**: Memotong 0.5% dari nilai transaksi melalui SmartBank pada setiap permintaan transaksional yang berhasil.
5. **Perlindungan operasional**: Menerapkan *rate limiting*, *idempotency*, dan *circuit breaker* untuk melindungi ekosistem dari beban berlebih dan kegagalan kaskade.

### 4.2 Secondary Goals

1. Menyediakan konsol operasional internal (Integrator Console) bagi tim platform untuk pemantauan, konfigurasi, dan rekonsiliasi.
2. Mengekspos endpoint *introspection* untuk validasi token yang dapat dipanggil oleh layanan terpercaya.
3. Mendukung *zero-downtime key rotation* untuk JWT signing keys.
4. Memberikan API kueri log yang dapat diakses oleh peran admin untuk investigasi insiden.

### 4.3 Explicit Non-Goals

Bagian ini menetapkan secara eksplisit apa yang **bukan** menjadi tanggung jawab Gateway. Klarifikasi ini krusial untuk mencegah *scope creep* dan menutup ruang interpretasi yang sebelumnya menghasilkan drift desain.

1. **Bukan aplikasi konsumer**. Gateway tidak memiliki onboarding flow untuk end-user UMKM, halaman pendaftaran publik, atau alur pembelian. End-user berinteraksi dengan ekosistem melalui frontend aplikasi domain masing-masing.
2. **Bukan pemilik business logic domain**. Gateway tidak menghitung total transaksi, tidak memverifikasi stok, tidak menetapkan harga, tidak memutasi saldo, dan tidak mengeksekusi keputusan bisnis apa pun. Seluruh logika domain berada di aplikasi downstream.
3. **Bukan otoritas moneter**. Gateway tidak pernah memutasi saldo pengguna secara langsung. Setiap pemotongan fee dilakukan dengan memanggil endpoint SmartBank `POST /smartbank/pembayaran_transaksi`.
4. **Bukan platform multi-tenant**. Gateway melayani tujuh aplikasi yang dikonfigurasi secara statis di registry. Konsep "merchant management", "tenant onboarding", "API consumer marketplace", dan "webhook configuration" berada di luar scope.
5. **Bukan penyedia analitik bisnis**. Analitik bisnis ditangani UMKM Insight. Gateway hanya menyediakan metrik operasional internal (latency, error rate, fee revenue) untuk konsumsi tim platform sendiri.
6. **Bukan layanan notifikasi end-user**. Gateway tidak mengirim email, SMS, atau push notification kepada end-user. Komunikasi dengan end-user ditangani oleh aplikasi domain masing-masing.

---

## 5. Target Users dan Personas

Gateway memiliki dua kategori pengguna: *direct users* yang berinteraksi langsung dengan permukaan Gateway, dan *indirect users* yang interaksinya difasilitasi Gateway namun mereka sendiri tidak menyentuh permukaan Gateway secara langsung.

### 5.1 Direct Users (Pengguna Integrator Console)

**Persona 1 — Platform Operator**

Profil: Anggota tim Kelompok 7.0 yang bertanggung jawab atas kesehatan operasional Gateway selama pengembangan dan demo. Background teknis: pemahaman dasar HTTP, JWT, dan database relasional.

Tugas utama: memantau dashboard kesehatan, menelusuri log saat investigasi bug, mengonfigurasi route baru, melakukan rotasi JWT key, dan menjelaskan perilaku Gateway selama presentasi Tugas Besar.

Kebutuhan utama: kepadatan informasi tinggi, kemampuan filter log yang cepat, akses langsung ke konfigurasi tanpa perlu redeploy, transparansi penuh terhadap jejak audit.

**Persona 2 — Finance Auditor**

Profil: Anggota tim yang memverifikasi bahwa pemungutan fee 0.5% berjalan benar dan rekonsiliasi pendapatan akurat.

Tugas utama: meninjau laporan pendapatan harian, menyelidiki *deferred fees*, mengeksekusi *manual reconciliation* untuk fee yang gagal setelah lima percobaan retry.

Kebutuhan utama: agregasi pendapatan per periode, daftar fee yang membutuhkan intervensi manual, jejak audit perubahan status fee.

### 5.2 Indirect Users

**Persona 3 — Developer Aplikasi Domain (Kelompok lain)**

Profil: Anggota dari enam kelompok lain yang membangun aplikasi domain (SmartBank, PasarKita, WarungPOS, SupplierHub, LogistiKita, UMKM Insight). Mereka tidak menggunakan Integrator Console, namun aplikasi mereka harus berkomunikasi melalui Gateway.

Kebutuhan utama: dokumentasi API yang jelas, kontrak yang stabil dan ter-versioning, pesan kesalahan yang dapat ditindaklanjuti, *latency* yang rendah dan dapat diprediksi.

**Persona 4 — End-User Ekosistem (UMKM, pembeli, kasir, driver)**

Profil: Pengguna akhir aplikasi domain yang permintaannya transit melalui Gateway tanpa mereka sadari. Mereka tidak melihat permukaan Gateway sama sekali.

Kebutuhan implisit: keandalan dan kecepatan ekosistem secara keseluruhan, yang Gateway pengaruhi secara tidak langsung melalui reliabilitas operasionalnya.

---

## 6. Scope

### 6.1 In-Scope

**Empat fitur wajib (mandat Sheet 2)**:

1. Routing API — `/integrator/routing_api`
2. Validasi Request — `/integrator/validasi_request`
3. Logging — `/integrator/logging` (POST untuk ingestion, GET untuk kueri)
4. Biaya Layanan Integrasi — `/integrator/biaya_layanan_integrasi`

**Cross-cutting concerns**:

5. Rate limiting per pengguna dan per route class.
6. Idempotency untuk seluruh route transaksional.
7. Circuit breaker per layanan downstream.
8. Request tracing dengan propagasi `request_id` dan `parent_request_id`.
9. JWT signing key rotation dengan dual-key window.

**Antarmuka pengguna**:

10. Integrator Console dengan sembilan halaman: Login, Dashboard, Request Logs, Gateway Fees, Route Registry, Service Health, Security & JWT, Konfigurasi, dan Settings.

### 6.2 Out-of-Scope

Selain enam non-goals di bagian 4.3, item berikut dinyatakan eksplisit di luar scope rilis 1.0:

1. Mobile app untuk operator (Console hanya desktop, viewport minimum 1280×800).
2. Integrasi dengan sistem identitas eksternal (SSO, LDAP, SAML).
3. Multi-region deployment dengan replikasi data lintas region.
4. Pemrosesan stream real-time untuk log (sistem menggunakan polling-based query).
5. Machine learning untuk deteksi anomali otomatis.
6. Marketplace API untuk pihak ketiga di luar tujuh aplikasi ekosistem.

---

## 7. Functional Requirements

Setiap requirement memiliki ID unik dengan format `FR-{kategori}-{nomor}`, mengikuti konvensi: `RT` untuk routing, `AU` untuk authentication, `LG` untuk logging, `FE` untuk fee, `RL` untuk rate limit, `ID` untuk idempotency, `CB` untuk circuit breaker, dan `UI` untuk antarmuka pengguna.

### 7.1 Routing (FR-RT)

**FR-RT-01** — Gateway harus menerima permintaan HTTP pada dua pola routing yang setara: pola transparan `{METHOD} /api/v1/{service}/{feature}` dan pola envelope `POST /integrator/routing_api` dengan body yang menyebutkan `target_service`, `target_feature`, `method`, dan `payload`.

**FR-RT-02** — Gateway harus melakukan resolusi route dengan mencari kombinasi `(service_name, feature_name, method)` di tabel `routes_registry` dan menolak permintaan dengan kode `404 ROUTE_NOT_FOUND` apabila tidak ditemukan.

**FR-RT-03** — Gateway harus meneruskan request body, header yang relevan (Authorization, Content-Type, X-Request-Id, X-Idempotency-Key, X-User-Id), dan query parameter ke layanan downstream tanpa mengubah konten.

**FR-RT-04** — Gateway harus menerapkan timeout yang dapat dikonfigurasi per route (default 5000 ms) dan mengembalikan kode `502 UPSTREAM_FAILED` apabila downstream melampaui timeout.

**FR-RT-05** — Gateway harus mendukung jumlah retry yang dapat dikonfigurasi per route (default 0) dengan exponential backoff jittered untuk *idempotent methods* saja.

**FR-RT-06** — Gateway harus mempropagasi `X-Request-Id` ke layanan downstream dan menerima kembali `X-Parent-Request-Id` apabila downstream memicu permintaan turunan kembali ke Gateway.

**FR-RT-07** — Gateway harus menolak permintaan dengan `X-Hop-Count >= 3` dengan kode kesalahan untuk mencegah circular routing.

### 7.2 Authentication dan Validation (FR-AU)

**FR-AU-01** — Gateway harus memverifikasi tanda tangan JWT dengan algoritma RS256 menggunakan public key yang diambil dari endpoint JWKS SmartBank.

**FR-AU-02** — Gateway harus memvalidasi *standard claims* JWT: `iss`, `sub`, `aud="ecosystem"`, `exp` (belum kedaluwarsa), `iat`, `nbf` (apabila ada). *Clock skew tolerance* sebesar 30 detik harus diterapkan.

**FR-AU-03** — Gateway harus mendukung rotasi key dengan menerima JWT yang ditandatangani oleh `kid` aktif maupun `kid` yang masih dalam *deprecation window* selama 30 hari.

**FR-AU-04** — Gateway harus memeriksa `jti` token terhadap blacklist Redis dan menolak token yang ter-revoke dengan kode `401 AUTH_INVALID_TOKEN`.

**FR-AU-05** — Gateway harus memberlakukan `required_scope` per route. Token tanpa scope yang dibutuhkan harus ditolak dengan kode `403 AUTH_SCOPE_DENIED`.

**FR-AU-06** — Gateway harus mengekspos endpoint *introspection* `POST /integrator/validasi_request` yang mengembalikan `{ valid, user_id, roles, scopes, exp }` untuk pemanggil internal terpercaya.

### 7.3 Logging dan Audit (FR-LG)

**FR-LG-01** — Gateway harus mencatat setiap permintaan dengan tiga state lifecycle: `STARTED` saat permintaan masuk, `COMPLETED` saat respons berhasil dikembalikan, dan `FAILED` saat permintaan menemui kesalahan terminal.

**FR-LG-02** — Setiap baris log harus memuat field minimum: `request_id`, `parent_request_id` (nullable), `user_id`, `source_app`, `target_app`, `endpoint`, `method`, `status_code`, `latency_ms`, `ip_address`, `request_hash` (SHA-256), `response_hash` (SHA-256), `lifecycle`, `created_at`.

**FR-LG-03** — Gateway harus menyimpan body permintaan dan respons sebagai SHA-256 hash secara default. Penyimpanan body lengkap hanya boleh dilakukan untuk route yang ditandai `audit_full=true` dan disimpan dalam kolom terenkripsi dengan retensi maksimum 30 hari.

**FR-LG-04** — Gateway harus mengredaksi field PII (nomor telepon, alamat email, nomor identitas) sebelum persistensi log.

**FR-LG-05** — Gateway harus mengekspos endpoint kueri log `GET /integrator/logging` dengan filter `user_id`, `request_id`, `from`, `to`, `status_code`, `target_app`, dengan paginasi dan otorisasi scope `admin:read`.

**FR-LG-06** — Tabel log harus menggunakan partisi bulanan dan partisi yang lebih lama dari 90 hari diarsipkan ke penyimpanan dingin.

### 7.4 Pemungutan Fee (FR-FE)

**FR-FE-01** — Gateway harus menghitung fee dengan rumus `fee = round(transaction_amount × 0.005)` menggunakan *rounding half up* sebagai default yang dapat dikonfigurasi.

**FR-FE-02** — Gateway harus memanggil endpoint SmartBank `POST /smartbank/pembayaran_transaksi` dengan payload yang menyebutkan `from_user=user_id`, `to_user="GATEWAY_REVENUE"`, `amount=fee`, dan metadata `kind="gateway_fee"` setelah downstream berhasil dan sebelum respons dikembalikan ke caller.

**FR-FE-03** — Gateway harus menggunakan idempotency key `fee-{request_id}` pada panggilan SmartBank untuk mencegah pemungutan ganda saat retry.

**FR-FE-04** — Apabila pemungutan fee gagal sementara transaksi downstream sukses, Gateway harus tetap mengembalikan respons sukses kepada caller dengan `fee.status="deferred"`, mencatat baris `gateway_fees` dengan status `PENDING`, dan menjadwalkan retry dengan exponential backoff 30 detik, 2 menit, 8 menit, 32 menit, dan 2 jam.

**FR-FE-05** — Setelah lima percobaan retry gagal, baris `gateway_fees` harus ditandai `FAILED` dan tim finance harus diberi tahu melalui mekanisme alert yang dikonfigurasi.

**FR-FE-06** — Gateway dilarang melakukan rollback atas transaksi downstream apabila pemungutan fee gagal, sesuai Aturan 4 yang melarang Gateway memiliki otoritas atas saldo.

**FR-FE-07** — Gateway harus mengklasifikasikan route sebagai *transactional* atau *non-transactional* berdasarkan flag `transactional` di registry. Hanya route transactional yang dipungut fee.

### 7.5 Rate Limiting (FR-RL)

**FR-RL-01** — Gateway harus menerapkan token bucket per kombinasi `(user_id, route_class)` dengan default 60 permintaan per menit untuk class `read` dan 10 permintaan per menit untuk class `transactional`.

**FR-RL-02** — Gateway harus menerapkan token bucket sekunder per `ip_address` dengan limit yang lebih tinggi untuk mencegah abuse dari satu sumber tanpa membatasi pengguna sah.

**FR-RL-03** — State token bucket harus disimpan di Redis dengan operasi atomik `INCR` dan TTL.

**FR-RL-04** — Gateway harus menerapkan *cooldown* antar transaksi untuk pengguna yang sama pada route transactional, dengan durasi default 10 detik dan dapat dikonfigurasi dalam rentang 10–30 detik (Sheet 6 item 15).

**FR-RL-05** — Gateway harus memberlakukan batas maksimum 10 transaksi per pengguna per hari pada route class transactional (Sheet 6 item 16).

**FR-RL-06** — Permintaan yang melampaui rate limit harus dijawab dengan kode `429 RATE_LIMITED` dan header `Retry-After` yang menyebutkan detik hingga bucket terisi kembali.

### 7.6 Idempotency (FR-ID)

**FR-ID-01** — Gateway harus mensyaratkan header `X-Idempotency-Key` pada setiap permintaan ke route transactional. Permintaan tanpa header ini ditolak dengan `400 VALIDATION_FAILED`.

**FR-ID-02** — Gateway harus menghitung `key_hash = SHA-256(method + path + user_id + idempotency_key + body)` dan menyimpan pasangan `(key_hash, cached_response)` di Redis dengan TTL 24 jam.

**FR-ID-03** — Permintaan ulang dengan key dan body yang sama harus mengembalikan respons cache tanpa memanggil downstream dan tanpa memungut fee tambahan.

**FR-ID-04** — Permintaan ulang dengan key yang sama tetapi body yang berbeda harus dijawab dengan `409 IDEMPOTENCY_REPLAY`.

### 7.7 Circuit Breaker (FR-CB)

**FR-CB-01** — Gateway harus memelihara state circuit breaker independen per layanan downstream dengan tiga state: CLOSED, OPEN, HALF-OPEN.

**FR-CB-02** — Circuit harus berpindah dari CLOSED ke OPEN setelah 5 respons 5xx berturut-turut atau 3 timeout dalam jendela 30 detik.

**FR-CB-03** — Saat OPEN, circuit harus mengembalikan kode `503 CIRCUIT_OPEN` dalam waktu kurang dari 10 ms tanpa menghubungi downstream.

**FR-CB-04** — Setelah 60 detik di state OPEN, circuit berpindah ke HALF-OPEN dan mengizinkan satu *probe request* lewat. Sukses pada probe mengembalikan ke CLOSED, gagal mengembalikan ke OPEN.

**FR-CB-05** — Operator harus dapat memaksa probe lebih awal melalui Integrator Console dengan otorisasi scope `admin:manage`.

### 7.8 User Interface (FR-UI)

**FR-UI-01** — Integrator Console harus menyediakan sembilan halaman: Login, Dashboard, Request Logs, Gateway Fees, Route Registry, Service Health, Security & JWT, Konfigurasi, Settings.

**FR-UI-02** — Sidebar navigasi harus konsisten lintas seluruh halaman dengan delapan item dalam urutan tetap: Dashboard, Request Logs, Gateway Fees, Route Registry, Service Health, Security & JWT, Konfigurasi, Settings.

**FR-UI-03** — Console hanya dapat diakses oleh akun dengan peran Operator, FinanceAuditor, AdminFull, atau ReadOnlyViewer. Akun end-user ekosistem dilarang mengakses Console.

**FR-UI-04** — Console mendukung viewport minimum 1280×800. Tampilan mobile berada di luar scope rilis 1.0.

**FR-UI-05** — Seluruh aksi destruktif (rotate key, mark as failed, delete blacklist entry, force logout all sessions) harus dilindungi modal konfirmasi yang menyebutkan dampak dan mensyaratkan input verifikasi (password atau frasa konfirmasi).

**FR-UI-06** — Console harus menampilkan environment badge (DEV / STAGING / PROD) di top bar untuk mencegah operator mengeksekusi aksi pada environment yang salah.

---

## 8. Non-Functional Requirements

### 8.1 Performance

| Metrik | Target | Catatan |
|---|---|---|
| p50 latency overhead Gateway | < 30 ms | Diukur dari ingress hingga egress, tidak termasuk waktu downstream |
| p95 latency overhead Gateway | < 100 ms | |
| p99 latency overhead Gateway | < 250 ms | |
| Throughput per node | > 500 requests per second | Pada hardware setara t3.medium |
| Cold start time | < 3 detik | Dari proses dimulai hingga siap menerima trafik |

### 8.2 Reliability

| Metrik | Target |
|---|---|
| Uptime di environment DEV | 99.5% |
| Uptime di environment PROD (apabila ada) | 99.9% |
| Recovery time objective (RTO) setelah crash | < 30 detik |
| Data loss tolerance untuk log | 0 baris (write-ahead semantics) |
| Data loss tolerance untuk fee records | 0 baris (kritikal finansial) |

### 8.3 Security

| Aspek | Persyaratan |
|---|---|
| JWT algoritma | RS256 (asymmetric, public key di Gateway) |
| Rotation period | 90 hari, dual-key window 30 hari |
| TLS | TLS 1.3 minimum, terminasi di reverse proxy |
| Rate limiting against brute force | 5 percobaan login Console per 15 menit per IP |
| Session timeout Console | 30 menit idle, 8 jam absolute |
| Password hashing operator account | argon2id |
| 2FA Console | TOTP wajib untuk peran AdminFull |
| Audit log Console | Setiap aksi tercatat dengan operator_id dan diff |

### 8.4 Observability

Gateway harus mengekspos metrik Prometheus pada endpoint `/metrics` dengan minimum: `gateway_requests_total{service,method,status}`, `gateway_request_duration_seconds{service,method}`, `gateway_fees_total{status}`, `gateway_circuit_state{service}`, `gateway_rate_limit_exceeded_total{user,class}`.

Health check endpoint `/health` harus mengembalikan 200 OK ketika Gateway dapat menjangkau database dan Redis. Health check terpisah `/ready` mengembalikan 200 hanya ketika seluruh dependency terinisialisasi.

Structured logging menggunakan format JSON dengan level INFO untuk lifecycle log normal, WARN untuk degradasi (deferred fee, retry), ERROR untuk kegagalan terminal, dan FATAL untuk crash.

### 8.5 Scalability

Gateway harus *stateless* sehingga dapat di-scale horizontal di belakang load balancer. State volatile (rate buckets, idempotency cache) berada di Redis yang dibagi antar instance. State persisten (logs, fees, registry, blacklist) berada di PostgreSQL.

Database harus mendukung partisi bulanan untuk `request_logs` dan vacuum otomatis untuk mencegah degradasi performa pada volume tinggi.

### 8.6 Maintainability

Codebase harus mengikuti struktur Clean Architecture / MVC hibrida sebagaimana didokumentasikan di blueprint section 10. Cakupan unit test minimum 70%, integration test mencakup seluruh 15 skenario test di blueprint section 12.

Migrasi database harus berbasis file dengan tooling standar (Flyway, Liquibase, atau setara). Konfigurasi runtime dimuat dari environment variable dengan loader bertipe.

---

## 9. User Stories

### 9.1 Stories untuk Platform Operator

**US-OP-01** — Sebagai operator, saya ingin melihat dashboard kesehatan Gateway sekali pandang sehingga saya dapat mengetahui status sistem dalam waktu kurang dari 5 detik setelah login.

**US-OP-02** — Sebagai operator, saya ingin memfilter Request Logs berdasarkan rentang waktu, status code, layanan target, dan user ID sehingga saya dapat menelusuri penyebab kegagalan tertentu.

**US-OP-03** — Sebagai operator, saya ingin mengeklik baris log dan melihat detail lengkap (headers, body hash, lifecycle timeline) di drawer samping tanpa kehilangan konteks daftar.

**US-OP-04** — Sebagai operator, saya ingin menambahkan, mengubah, dan menonaktifkan route di registry tanpa redeploy aplikasi.

**US-OP-05** — Sebagai operator, saya ingin memaksa probe pada circuit breaker yang sedang HALF-OPEN sehingga saya dapat mempercepat pemulihan setelah saya yakin layanan downstream sudah pulih.

**US-OP-06** — Sebagai operator, saya ingin merotasi JWT signing key dengan satu klik dan window deprecation 30 hari otomatis terkonfigurasi.

### 9.2 Stories untuk Finance Auditor

**US-FA-01** — Sebagai auditor, saya ingin melihat total pendapatan fee per hari, minggu, dan bulan sehingga saya dapat memvalidasi konsistensi pemungutan.

**US-FA-02** — Sebagai auditor, saya ingin melihat daftar fee yang berstatus DEFERRED dan FAILED beserta alasan kegagalan sehingga saya dapat melakukan rekonsiliasi manual.

**US-FA-03** — Sebagai auditor, saya ingin memicu retry manual pada fee tertentu apabila saya yakin penyebab kegagalan sudah tertangani.

**US-FA-04** — Sebagai auditor, saya ingin mengekspor laporan fee bulanan dalam format CSV sehingga saya dapat melaporkannya dalam dokumentasi Tugas Besar.

### 9.3 Stories untuk Developer Aplikasi Domain

**US-DD-01** — Sebagai developer downstream, saya ingin menerima pesan kesalahan dengan kode kanonis dan field `request_id` sehingga saya dapat menelusuri permintaan yang gagal di sisi Gateway.

**US-DD-02** — Sebagai developer downstream, saya ingin permintaan yang saya kirim memiliki overhead Gateway kurang dari 100 ms p95 sehingga performa aplikasi saya tidak terganggu.

**US-DD-03** — Sebagai developer downstream, saya ingin idempotency dijamin pada retry yang saya lakukan sehingga saya tidak khawatir double-charging pengguna.

### 9.4 Stories untuk End-User Ekosistem (Tidak Langsung)

**US-EU-01** — Sebagai pembeli marketplace, saya ingin transaksi checkout saya selesai dalam waktu kurang dari 2 detik sehingga pengalaman belanja saya nyaman. (Diturunkan menjadi target latency Gateway p95 < 100 ms.)

**US-EU-02** — Sebagai pengguna ekosistem, saya ingin permintaan saya tidak terkena double-charge ketika koneksi terputus dan saya melakukan retry. (Dipenuhi melalui FR-ID-03.)

---

## 10. Antarmuka Pengguna

### 10.1 Integrator Console — Sembilan Halaman

Detail wireframe dan komponen tiap halaman didokumentasikan dalam `stitch_prompts.md` (halaman 1–7) dan `stitch_prompts_addendum.md` (halaman 8–9). Berikut ringkasan tujuan dan komponen utama tiap halaman.

| # | Halaman | Tujuan | Komponen Utama |
|---|---|---|---|
| 1 | Login | Otentikasi operator | Email field, password field, 2FA, environment badge |
| 2 | Dashboard | Snapshot operasional 24 jam | 4 KPI tiles, throughput chart, top services, circuit breakers, recent errors, deferred fees |
| 3 | Request Logs | Investigasi log audit | Filter bar, dense table 12+ kolom, drawer detail dengan tabs Overview/Headers/Body/Trace/Related |
| 4 | Gateway Fees | Visibilitas pendapatan dan rekonsiliasi | 3 summary cards, revenue trend chart, donut by source app, leaderboard top users, tabel pending/failed fees |
| 5 | Route Registry | Manajemen kontrak API | Tab strip per service, tabel route 11 kolom, drawer edit dengan validasi |
| 6 | Service Health | Monitoring real-time | Grid 6 service cards, timeline circuit breaker 24 jam, tabel rate limit buckets |
| 7 | Security & JWT | Manajemen kunci dan token | Tabs Signing Keys / Token Blacklist / Scopes & Roles / Audit Log |
| 8 | Settings | Pengaturan akun operator | Sub-nav 7 panel: Profil, Keamanan, Sesi Aktif, Notifikasi, Tampilan, Audit Aktivitas, API Keys |
| 9 | Konfigurasi | Tunables sistem-wide | Tab strip Umum / Fee & Pajak / Rate Limit / Timeout & Retry / Logging / Circuit Breaker |

### 10.2 Design System

Palette utama menggunakan amber-orange (`#EA580C`) sebagai aksen pada aksi primer dan item navigasi aktif, dipadukan dengan slate-grey neutral untuk chrome dan teks. Deep purple (`#7E22CE`) digunakan khusus untuk angka moneter agar selaras dengan identitas SmartBank. Semantic colours mengikuti konvensi industri: hijau untuk sukses, kuning untuk peringatan, merah untuk bahaya, biru untuk informasi.

Typography menggunakan Inter untuk teks UI dan JetBrains Mono untuk ID, hash, dan data teknis. Spacing grid berbasis 8 piksel, border radius 8 piksel pada input dan tombol, 12 piksel pada kartu.

Detail spesifikasi penuh tersedia di `stitch_prompts.md` Prompt 0 (Design System Foundation).

### 10.3 Sidebar Konsistensi

Seluruh halaman selain Login menggunakan sidebar identik dengan delapan item dalam urutan tetap. Spesifikasi presisi (dimensi, warna hex, ikon, state aktif/hover, footer indicator) didokumentasikan di `stitch_sidebar_refactor.md`. Konsistensi sidebar dipertahankan melalui ekspor sebagai komponen Figma dengan variant per state aktif.

---

## 11. Technical Architecture

### 11.1 Tech Stack

| Layer | Pilihan Utama | Alternatif |
|---|---|---|
| Runtime | Golang 1.22 + net/http | Node.js 20 LTS + Express |
| HTTP client | undici | axios, httpx |
| JWT | jose | PyJWT |
| Validation | zod | pydantic |
| Database | MySQL 8 |  |
| Cache | Redis 7 | KeyDB |
| Logging | pino (JSON) | winston, loguru |
| Process manager | PM2 atau Docker | systemd |
| Reverse proxy | Nginx | Traefik |
| Testing | Jest + Supertest | pytest + httpx |

### 11.2 Database Schema

Skema lengkap didokumentasikan di blueprint section 8. Ringkasan tabel utama:

| Tabel | Lokasi | Fungsi |
|---|---|---|
| `routes_registry` | MySQL | Mapping service → endpoint downstream |
| `request_logs` | MySQL (dipartisi bulanan) | Audit trail seluruh permintaan |
| `gateway_fees` | MySQL | Pencatatan fee 0.5% beserta status |
| `jwt_blacklist` | MySQL | Token yang ter-revoke |
| `rate_buckets` | Redis | State token bucket per pengguna |
| `idempotency_keys` | Redis | Cache respons untuk retry idempoten |

### 11.3 Integrasi SmartBank

Gateway memanggil dua endpoint SmartBank: `POST /smartbank/pembayaran_transaksi` untuk pemungutan fee, dan endpoint JWKS untuk public key JWT. Service token Gateway (untuk memanggil SmartBank dengan otoritas sistem) di-issue oleh SmartBank dengan `aud="smartbank"` dan scope `payment:write`.

Setiap pemanggilan SmartBank menggunakan `X-Idempotency-Key` dengan format `fee-{request_id}` untuk fee, dan `gw-{operation}-{uuid}` untuk operasi internal lain.

### 11.4 Integrasi Aplikasi Downstream

Setiap aplikasi downstream berkomunikasi dengan Gateway melalui HTTP POST dengan service token sendiri. Downstream tidak diizinkan memanggil aplikasi lain secara langsung — seluruh komunikasi inter-service harus transit melalui Gateway sesuai Aturan 5.

Downstream menerima header `X-Request-Id` dari Gateway dan harus mempropagasinya kembali pada panggilan turunan ke Gateway, mengisi `X-Parent-Request-Id` untuk membentuk trace tree.

### 11.5 Diagram Arsitektur

Lima diagram Graphviz mendokumentasikan arsitektur secara visual:

1. **Topologi Ekosistem** — posisi Gateway di antara tujuh aplikasi.
2. **Request Lifecycle** — middleware chain 14 langkah dari ingress hingga response.
3. **End-to-End Marketplace Checkout** — flow transaksi 12 hop dengan pemungutan fee.
4. **Failure Handling** — circuit breaker state machine dan deferred fee reconciliation.
5. **Empat Fitur IPO** — Input-Process-Output kontrak setiap fitur wajib.

File source `.dot` dan render PNG/SVG tersedia di output sebagai `01_ecosystem_topology.*` hingga `05_features_ipo.*`.

---

## 12. Dependencies dan Integrations

### 12.1 Dependencies Internal Ekosistem

| Dependency | Tipe | Kritikalitas | Failure Impact |
|---|---|---|---|
| SmartBank | Hard dependency | Critical | Tanpa SmartBank, Gateway tidak dapat memungut fee dan validasi JWT gagal |
| 5 aplikasi domain lain | Soft dependency | Medium | Kegagalan satu aplikasi domain hanya berdampak pada route ke aplikasi tersebut |
| Auth Issuer (sub-komponen SmartBank) | Hard dependency | Critical | Tanpa JWKS, seluruh validasi JWT gagal |

### 12.2 Dependencies Infrastruktur

| Dependency | Tipe | Failure Impact |
|---|---|---|
| PostgreSQL | Hard | Logging dan fee persistence terhenti; Gateway dapat tetap merutekan dengan log buffer |
| Redis | Hard | Rate limiting dan idempotency cache tidak berfungsi; Gateway dapat fail-open atau fail-closed berdasarkan konfigurasi |
| DNS | Hard | Resolusi hostname downstream gagal |
| Network egress | Hard | Tidak dapat menjangkau aplikasi downstream |

### 12.3 Strategi Degradasi

Gateway menerapkan tiga mode degradasi tergantung dependency yang gagal:

**Mode 1 — Logging unavailable**: Gateway tetap merutekan permintaan, namun menulis log ke file lokal sebagai buffer. Log direplay ke database setelah pemulihan.

**Mode 2 — Redis unavailable**: Rate limiting fail-closed (tolak semua transactional dengan 503) untuk environment finansial; idempotency fail-open dengan warning log untuk environment dev.

**Mode 3 — SmartBank unavailable**: Validasi JWT gagal pada cache miss; fee charging tertunda dengan status DEFERRED. Operator menerima alert kritikal.

---

## 13. Success Metrics

### 13.1 KPI Operasional (Pengukuran Mingguan)

| Metrik | Target | Sumber |
|---|---|---|
| Request volume | > 100,000 per minggu | Dashboard total requests |
| Error rate | < 1% | Dashboard error rate |
| p95 latency | < 100 ms | Service health |
| Uptime | > 99.5% | Health check eksternal |
| Fee collection rate | > 99% | Gateway fees dashboard |
| Deferred fee resolution rate | > 95% dalam 24 jam | Gateway fees dashboard |

### 13.2 KPI Finansial

| Metrik | Target |
|---|---|
| Total revenue per minggu | Bergantung volume ekosistem |
| Failed fee count | 0 (zero tolerance untuk kegagalan permanen) |
| Manual reconciliation count | < 5 per minggu |
| Revenue accuracy | 100% (audit log harus rekonsiliabel dengan SmartBank ledger) |

### 13.3 KPI Kualitas Kode

| Metrik | Target |
|---|---|
| Unit test coverage | > 70% |
| Integration test pass rate | 100% pada 15 skenario blueprint |
| Build time CI | < 5 menit |
| Deployment frequency | Sesuai phase roadmap |
| Mean time to recovery | < 15 menit |

### 13.4 Acceptance Criteria untuk Tugas Besar

| Sheet 5 Item | Deliverable | Status |
|---|---|---|
| 1. Deskripsi Aplikasi | Bagian 1–3 PRD ini | Final |
| 2. Tujuan Aplikasi | Bagian 4 (Goals) | Final |
| 3. Aturan Bisnis | Pemetaan 10 Aturan di blueprint section 2 | Final |
| 4. Diagram Arsitektur | 5 diagram Graphviz | Final |
| 5. API Endpoint | Blueprint section 5 + Route Registry mockup | Final |
| 6. Database Design | Blueprint section 8 | Final |
| 7. Diagram Alur | 5 diagram Graphviz (#2, #3, #4) | Final |
| 8. Mekanisme Transaksi | Blueprint section 6 + Gateway Fees mockup | Final |
| 9. UI Sederhana | 9 mockup Stitch | Pending eksekusi |
| 10. Skenario Pengujian | 15 skenario di blueprint section 12 | Final |
| 11. Kendala dan Solusi | Bagian 14 PRD ini (Risks) | Final |
| 12. Pembagian Tugas Tim | Tabel internal tim | Pending |

---

## 14. Risks dan Mitigations

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-01 | Gateway menjadi single point of failure ekosistem | Medium | Critical | Stateless design dengan horizontal scaling; health check; circuit breaker per downstream |
| R-02 | Fee charge sukses namun respons hilang dalam transit | Low | High | Idempotency pada pemanggilan SmartBank dengan key `fee-{request_id}` |
| R-03 | Volume log mendegradasi performa kueri | High over time | Medium | Partisi bulanan; arsip ke cold storage setelah 90 hari |
| R-04 | JWT key kompromi | Low | Critical | RS256 dengan `kid` mendukung zero-downtime rotation; blacklist menyerap eksposur jangka pendek |
| R-05 | Circular routing antar service | Medium | Medium | Reject permintaan dengan `X-Source-Service` cocok target; cap rekursi via `X-Hop-Count` |
| R-06 | Cooldown aturan keuangan dilewati | Medium | Medium | Enforcement server-side di Gateway dengan `last_tx_at` per pengguna |
| R-07 | Time skew antar service menyebabkan JWT failure salah | Low | Medium | Toleransi clock skew 30 detik di verifier |
| R-08 | Operator salah eksekusi destructive action di PROD | Medium | High | Modal konfirmasi dengan input verifikasi; environment badge prominent di top bar |
| R-09 | Drift desain UI saat iterasi Stitch | Medium | Low | Sidebar refactor prompt; eksport ke Figma sebagai master component |
| R-10 | Misinterpretasi scope (mis. Gateway dianggap multi-tenant) | Low | High | Section 4.3 dan 5 PRD menyatakan eksplisit non-goals dan persona |

---

## 15. Implementation Plan

### 15.1 Phased Roadmap (Enam Minggu)

**Phase 1 — Foundation (Minggu 1)**: Bootstrap repositori, konfigurasi tooling, Docker Compose dengan PostgreSQL dan Redis, CI pipeline, environment loader, response envelope helper.

**Phase 2 — Core Middleware (Minggu 2)**: JWT validation middleware terhadap mock issuer, rate limiter Redis, request-id propagation, structured logging middleware, error handler dengan kode kanonis.

**Phase 3 — Routing Engine (Minggu 3)**: Route registry config-first dan DB-promotable, route resolver, proxy dengan undici connection pools, timeout dan retry dengan jittered exponential backoff, circuit breaker per service.

**Phase 4 — SmartBank Fee Integration (Minggu 4)**: Fee service dan SmartBank client typed, post-route fee charge hook, persistence `gateway_fees`, deferred fee retry worker, endpoint `/integrator/biaya_layanan_integrasi` untuk kueri dan admin trigger.

**Phase 5 — Idempotency, Observability, Hardening (Minggu 5)**: Idempotency middleware, endpoint kueri log dengan paginasi, metrik Prometheus, test suite seluruh skenario GW-T, load test pada batas Aturan Keuangan.

**Phase 6 — Documentation dan Demo (Minggu 6)**: Kompilasi dokumentasi Sheet 5, Integrator Console UI dari mockup Stitch, demo end-to-end transaksi (Marketplace → SmartBank → LogistiKita) yang ditelusuri melalui Gateway logs.

### 15.2 Pembagian Tanggung Jawab Tim

Pembagian role minimal untuk eksekusi enam fase:

| Role | Tanggung Jawab |
|---|---|
| Backend Lead | Phase 1, 3, 4 (core engine) |
| Middleware Engineer | Phase 2, 5 (auth, rate limit, idempotency) |
| Database Engineer | Schema, migration, query optimization |
| Frontend Engineer | Integrator Console implementation dari mockup Stitch |
| QA / DevOps | Test suite, load test, CI/CD pipeline |
| Documentation Lead | Sheet 5 deliverables, demo recording, presentation deck |

---

## 16. Test Strategy

Lima belas skenario test didokumentasikan di blueprint section 12 dengan ID GW-T01 hingga GW-T15. Skenario mencakup:

**Skenario sukses**: GW-T01 (read non-transactional), GW-T02 (transactional dengan fee), GW-T15 (concurrent calls).

**Skenario otentikasi**: GW-T03 (missing JWT), GW-T04 (expired JWT), GW-T05 (revoked token), GW-T13 (scope denied).

**Skenario routing**: GW-T06 (unknown route).

**Skenario rate limit dan idempotency**: GW-T07 (rate exceeded), GW-T08 (idempotency replay), GW-T09 (idempotency conflict).

**Skenario kegagalan downstream**: GW-T10 (timeout), GW-T11 (circuit open), GW-T12 (fee charge failed).

**Skenario administratif**: GW-T14 (log query API).

Setiap skenario harus memiliki test code di repository, dieksekusi di CI pada setiap pull request, dan tercatat hasilnya dalam test report yang disertakan dalam dokumentasi Sheet 5 item 10.

---

## 17. Open Questions dan Future Work

### 17.1 Open Questions untuk Klarifikasi

1. Apakah Gateway perlu mendukung GraphQL atau gRPC pada rilis berikutnya, atau REST cukup untuk seluruh Tugas Besar?
2. Apakah konfigurasi fee rate (saat ini 0.5% hard-coded di registry per route) perlu dapat diubah per pengguna untuk skenario VIP?
3. Apakah deferred fee perlu mekanisme escalation otomatis ke pihak ketiga (mis. tim finance eksternal) atau cukup notifikasi internal?

### 17.2 Future Work (Pasca Tugas Besar)

1. Mobile companion app untuk operator on-call.
2. Anomaly detection berbasis ML untuk pola permintaan mencurigakan.
3. Multi-region deployment dengan replikasi aktif-aktif.
4. Real-time streaming log via Server-Sent Events untuk Console.
5. Self-service developer portal untuk dokumentasi API otomatis.

---

## 18. Appendices

### Appendix A — Glossary

| Istilah | Definisi |
|---|---|
| Gateway | API Gateway / Integrator, komponen yang dispesifikasikan dalam dokumen ini |
| Downstream service | Salah satu dari enam aplikasi domain lain yang permintaannya transit melalui Gateway |
| Route | Mapping dari endpoint publik Gateway ke endpoint internal aplikasi domain |
| Transactional route | Route yang ditandai `transactional=true` di registry; dipungut fee 0.5% |
| Request ID | UUID unik per permintaan untuk korelasi audit log lintas service |
| Service token | JWT dengan `aud="ecosystem"` yang di-issue untuk akun layanan, bukan pengguna |
| Deferred fee | Fee yang gagal dipungut pada percobaan pertama dan dijadwalkan retry |
| Circuit breaker | Mekanisme fail-fast yang menutup jalur ke layanan yang gagal berulang |
| Idempotency key | Header `X-Idempotency-Key` yang menjamin permintaan ulang tidak menghasilkan efek ganda |
| JWKS | JSON Web Key Set, endpoint publik yang menyediakan public key JWT |
| KBBI | Kamus Besar Bahasa Indonesia, standar bahasa formal Indonesia |

### Appendix B — Reference Documents

| Dokumen | Lokasi | Deskripsi |
|---|---|---|
| Blueprint teknis | `api_gateway_blueprint.md` | Detail arsitektur, schema, dan implementasi |
| Stitch prompts batch 1 | `stitch_prompts.md` | Prompt 0 design system dan Prompt 1–7 untuk halaman utama |
| Stitch prompts batch 2 | `stitch_prompts_addendum.md` | Prompt 8 Settings dan Prompt 9 Konfigurasi |
| Sidebar refactor | `stitch_sidebar_refactor.md` | Spesifikasi canonical sidebar 8 item |
| Diagram Topologi | `01_ecosystem_topology.{dot,svg,png}` | Posisi Gateway di ekosistem |
| Diagram Lifecycle | `02_request_lifecycle.{dot,svg,png}` | Middleware chain 14 langkah |
| Diagram Checkout | `03_end_to_end_checkout.{dot,svg,png}` | Flow transaksi marketplace |
| Diagram Failure | `04_failure_handling.{dot,svg,png}` | Circuit breaker dan deferred fee |
| Diagram IPO | `05_features_ipo.{dot,svg,png}` | Input-Process-Output empat fitur |

### Appendix C — Pemetaan Sheet 5 ke PRD

| Sheet 5 Item | Bagian PRD yang Memenuhi |
|---|---|
| 1. Deskripsi Aplikasi | Bagian 1, 2 |
| 2. Tujuan Aplikasi | Bagian 4.1, 4.2 |
| 3. Aturan Bisnis | Bagian 2.1; blueprint section 2 |
| 4. Diagram Arsitektur | Bagian 11.5; lima diagram Graphviz |
| 5. API Endpoint | Bagian 7.1–7.4; blueprint section 5 |
| 6. Database Design | Bagian 11.2; blueprint section 8 |
| 7. Diagram Alur | Diagram 2, 3, 4 |
| 8. Mekanisme Transaksi | Bagian 7.4; blueprint section 6; diagram 3 |
| 9. UI Sederhana | Bagian 10; sembilan mockup Stitch |
| 10. Skenario Pengujian | Bagian 16; blueprint section 12 |
| 11. Kendala dan Solusi | Bagian 14 |
| 12. Pembagian Tugas Tim | Bagian 15.2 (template) |

### Appendix D — Pemetaan Aturan Sheet 4 ke Implementasi

| Aturan | Implementasi di Gateway |
|---|---|
| Aturan 1 — Setiap fitur satu node sistem | Empat fitur dideploy sebagai modul terpisah |
| Aturan 2 — Input-Process-Output | Setiap endpoint memiliki kontrak IPO eksplisit (Diagram 5) |
| Aturan 3 — Output transaksi adalah payment request | Fee 0.5% direalisasikan sebagai POST ke SmartBank |
| Aturan 4 — SmartBank otoritas moneter | Gateway tidak pernah memutasi saldo; selalu memanggil SmartBank |
| Aturan 5 — Komunikasi melalui Gateway | Gateway pemegang tunggal service registry |
| Aturan 6 — Validasi dan logging wajib | Setiap permintaan menghasilkan validasi JWT dan log entry atomik |
| Aturan 7 — Analitik read-only | UMKM Insight call ke SmartBank dilewatkan dengan scope read-only |
| Aturan 8 — Uang tidak boleh diciptakan bebas | Fee 0.5% berasal dari saldo eksisting pengguna |
| Aturan 9 — Setiap layanan punya fee | Gateway memungut 0.5% per transaksi |
| Aturan 10 — Endpoint adalah kontrak sistem | Spesifikasi endpoint ter-versioning dan immutable per rilis |

---

**End of Document**
