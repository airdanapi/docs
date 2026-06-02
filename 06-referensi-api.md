# 06. Referensi API

Bagi para pemrogram yang memerlukan alamat teknis sistem (Endpoint), berikut adalah daftar fitur API yang dimiliki langsung oleh Gateway (di luar penerusan rute):

## Fitur Wajib

### 1. Titik Kumpul (Routing)
* **Pola Transparan**: `[METODE] /api/v1/{layanan}/{fitur}`
* **Pola Amplop**: `POST /integrator/routing_api`
Digunakan oleh aplikasi lain untuk mengirim pesan ke aplikasi tujuan.

### 2. Validasi Token (Introspection)
* **Rute**: `POST /integrator/validasi_request`
Mengembalikan data tentang siapa yang sedang masuk (login) berdasarkan token JWT-nya (Apakah dia pembeli biasa, atau administrator?).

### 3. Pencatatan (Logging)
* **Kueri Log**: `GET /integrator/logging`
* **Ringkasan Log**: `GET /integrator/logging/stats`
Mengembalikan daftar sejarah aktivitas yang melewati Gateway (Hanya bisa diakses oleh operator).

### 4. Biaya Layanan Integrasi
* **Kueri Keuangan**: `GET /integrator/biaya_layanan_integrasi`
* **Ringkasan Pendapatan**: `GET /integrator/biaya_layanan_integrasi/stats`
Mengembalikan daftar potongan 0.5% yang telah berhasil, tertunda, atau gagal.

## Fitur Pengelola Khusus

Fitur di bawah ini hanya digunakan oleh Dasbor Web (Konsol Integrator).
* `GET /integrator/routes` : Mendapatkan buku telepon sistem (Daftar rute).
* `PUT /integrator/routes/{id}` : Memperbarui alamat rute tertentu.
* `GET /integrator/circuits` : Melihat kesehatan sirkuit aplikasi (Merah/Hijau).
* `GET /health` : Memastikan peladen dan basis data masih hidup.
* `GET /ready` : Memastikan peladen sudah siap menerima lalu lintas saat baru dinyalakan.

> **Catatan Teknis:** 
> Hampir semua rute di atas akan merespons menggunakan struktur JSON baku (*Envelope*) yang berisi: `success`, `data`, `error`, dan `request_id`.
