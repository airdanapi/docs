# 📘 Dokumentasi API Gateway — Ekosistem Ekonomi UMKM

**Kelompok 7.0 — Tugas Besar RPL 2**

---

## Selamat Datang

Dokumen ini merupakan panduan lengkap untuk memahami, memasang, dan menggunakan **API Gateway (Integrator)** dalam Ekosistem Ekonomi UMKM. Seluruh penjelasan ditulis menggunakan bahasa Indonesia yang mudah dipahami agar dapat diakses oleh anggota tim dari berbagai latar belakang.

## Daftar Isi Dokumentasi

| Dokumen | Isi | Untuk Siapa |
|---------|-----|-------------|
| [Gambaran Umum](./01-gambaran-umum.md) | Apa itu API Gateway dan mengapa diperlukan | Semua pembaca |
| [Arsitektur Sistem](./02-arsitektur-sistem.md) | Bagaimana komponen-komponen saling terhubung | Pengembang dan penguji |
| [Panduan Pemasangan](./03-panduan-pemasangan.md) | Langkah demi langkah memasang dan menjalankan | Pengembang |
| [Panduan Penggunaan](./04-panduan-penggunaan.md) | Cara mengoperasikan konsol dan fitur-fitur | Operator dan auditor |
| [Panduan Integrasi](./05-panduan-integrasi.md) | Cara kelompok lain menyambungkan aplikasinya | Kelompok lain |
| [Referensi API](./06-referensi-api.md) | Daftar lengkap alamat dan format permintaan | Pengembang |
| [Basis Data](./07-basis-data.md) | Struktur tabel dan penjelasan kolom | Pengembang basis data |
| [Keamanan](./08-keamanan.md) | Mekanisme autentikasi, otorisasi, dan perlindungan | Semua pembaca |
| [Pemecahan Masalah](./09-pemecahan-masalah.md) | Solusi untuk kendala yang sering ditemui | Operator |

## Teknologi yang Digunakan

| Komponen | Teknologi | Keterangan |
|----------|-----------|------------|
| Peladen API | Go (Golang) 1.25 | Bahasa pemrograman untuk sisi peladen |
| Basis data | MySQL 8 | Penyimpanan data utama (log, fee, rute) |
| Tembolok | Redis 7 | Penyimpanan sementara (pembatas laju, idempotensi) |
| Antarmuka web | React + Vite | Konsol pemantauan untuk operator |
| Autentikasi | JWT RS256 | Token keamanan dengan tanda tangan digital |

## Cara Membaca Dokumentasi Ini

- Jika Anda **baru mengenal proyek ini**, mulailah dari [Gambaran Umum](./01-gambaran-umum.md).
- Jika Anda ingin **memasang dan menjalankan**, langsung ke [Panduan Pemasangan](./03-panduan-pemasangan.md).
- Jika Anda dari **kelompok lain** dan ingin menyambungkan aplikasi, baca [Panduan Integrasi](./05-panduan-integrasi.md).

---

*Dokumentasi ini disusun oleh Tim Kelompok 7.0 — Tugas Besar RPL 2, 2026.*
