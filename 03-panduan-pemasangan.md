# 03. Panduan Pemasangan

Dokumen ini menjelaskan cara menyalakan (menjalankan) API Gateway di komputer lokal Anda.

## Persyaratan Awal

Pastikan komputer Anda sudah terpasang perangkat lunak berikut:
1. **Go (Golang)** versi 1.22 atau yang lebih baru.
2. **Node.js** versi 20 atau yang lebih baru (beserta NPM).
3. **MySQL** versi 8.
4. **Redis** versi 7.

## Langkah 1: Menyiapkan Basis Data

Sebelum menyalakan Gateway, kita harus membuat "rak buku" untuk menyimpan datanya (Basis Data).

1. Buka aplikasi Terminal (atau *Command Prompt*).
2. Masuk ke folder `backend/migrations`.
3. Anda dapat menggunakan skrip otomatis yang telah disediakan. 
   - Di Windows: Jalankan `setup_db.bat`.
   - Di Linux/Mac: Jalankan `bash setup_db.sh`.
4. Skrip tersebut akan otomatis membuat basis data bernama `integrator_gateway`, membuat semua tabel (seperti `request_logs` dan `routes_registry`), dan memasukkan data rute-rute awal.

## Langkah 2: Menyalakan Peladen Backend (Mesin Utama)

1. Buka jendela Terminal baru.
2. Masuk ke folder utama proyek, lalu masuk ke dalam folder `backend`:
   ```bash
   cd backend
   ```
3. Jalankan perintah berikut untuk menyalakan peladen:
   ```bash
   go run cmd/gateway/main.go
   ```
4. Jika berhasil, peladen akan mulai berjalan di *port* `8080`. Biarkan jendela terminal ini tetap terbuka.

## Langkah 3: Menyalakan Dasbor Web (Frontend)

1. Buka jendela Terminal baru lainnya.
2. Masuk ke folder utama proyek, lalu masuk ke dalam folder `frontend`:
   ```bash
   cd frontend
   ```
3. Jika ini adalah pertama kalinya Anda membukanya, unduh dahulu paket-paket pendukungnya dengan perintah:
   ```bash
   npm install
   ```
4. Setelah selesai, jalankan dasbor dengan perintah:
   ```bash
   npm run dev
   ```
5. Dasbor web sekarang berjalan di *port* `5173`.

## Langkah 4: Mengakses Sistem

- Buka peramban (browser) web Anda (seperti Google Chrome atau Firefox).
- Kunjungi **http://localhost:5173**.
- Anda akan melihat halaman masuk (Login). Gunakan akun contoh berikut:
  - **Email**: `operator@gateway.com`
  - **Kata Sandi**: `password123`
- Selamat! Anda kini sudah berada di dalam Dasbor Pemantauan (Integrator Console).
