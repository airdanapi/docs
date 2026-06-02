# 02. Arsitektur Sistem

## Posisi Gateway dalam Ekosistem

API Gateway berada persis di tengah-tengah seluruh ekosistem (berbentuk pola *hub-and-spoke* atau seperti roda sepeda dengan poros di tengahnya). 

- **Aplikasi Domain (Spokes):** *SmartBank*, *PasarKita*, *WarungPOS*, *SupplierHub*, *LogistiKita*, dan *UMKM Insight*.
- **Poros (Hub):** API Gateway.

Ketika pengguna menekan tombol "Beli" di *PasarKita*, alur yang terjadi adalah:
1. *PasarKita* mengirim permintaan pembelian ke Gateway.
2. Gateway memeriksa keamanan dan mencatat kedatangannya.
3. Gateway meneruskan permintaan itu ke *SmartBank* untuk memotong uang pembeli.
4. Jika *SmartBank* mengatakan "Berhasil", Gateway akan mengambil sedikit potongan fee (0.5%) untuk dirinya sendiri.
5. Gateway mengembalikan jawaban "Berhasil" ke *PasarKita*.

## Teknologi di Balik Layar

Gateway dibangun agar sangat cepat dan tidak mudah mati (tangguh). Oleh karena itu, arsitekturnya disusun dengan beberapa lapisan:

1. **Mesin Utama (Go/Golang):** Merupakan otak peladen (server) yang menerima dan meneruskan lalu lintas. Bahasa Go dipilih karena kemampuannya menangani ribuan permintaan secara bersamaan dengan sangat cepat.
2. **Penyimpanan Permanen (MySQL):** Tempat menyimpan data yang tidak boleh hilang, seperti catatan sejarah (Log), daftar rute jalan (Route Registry), dan catatan keuangan (Fee).
3. **Penyimpanan Cepat/Sementara (Redis):** Tempat menyimpan data yang sering berubah dengan cepat, seperti memori tentang siapa saja yang sedang terlalu banyak mengirim permintaan (*Rate Limiting*) atau data agar permintaan tidak terulang (*Idempotency*).
4. **Antarmuka Pengguna (React):** Tampilan dasbor (Dashboard) yang bisa dibuka melalui peramban (browser) oleh tim operasional untuk memantau sistem.

## Perlindungan Berlapis

Gateway dirancang dengan tiga lapisan perlindungan (seperti sekering listrik) untuk menjaga agar sistem tidak kelebihan beban:

- **Idempotensi**: Jika koneksi internet terputus di tengah jalan dan pengguna mengirim ulang permintaan pembayaran, Gateway ingat bahwa permintaan tersebut sudah pernah diproses. Pengguna tidak akan ditagih dua kali.
- **Pembatas Laju (Rate Limit)**: Mencegah satu aplikasi atau pengguna membanjiri sistem dengan ratusan permintaan dalam satu detik.
- **Pengaman Sirkuit (Circuit Breaker)**: Jika *LogistiKita* sedang rusak, Gateway akan berhenti mengirimkan lalu lintas ke sana sejenak (sirkuit terbuka) agar *LogistiKita* punya waktu untuk memulihkan diri. Gateway akan langsung merespons "Layanan sedang sibuk" tanpa perlu menunggu lama.
