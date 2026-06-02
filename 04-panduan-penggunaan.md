# 04. Panduan Penggunaan

Panduan ini ditujukan bagi Operator atau Auditor Keuangan yang menggunakan **Konsol Integrator** (Dasbor Web) untuk memantau sistem.

Setelah Anda berhasil masuk ke dalam konsol (melalui http://localhost:5173), Anda akan melihat menu navigasi di sebelah kiri dengan beberapa halaman berikut:

## 1. Dashboard Utama
Ini adalah halaman pusat komando Anda.
- Anda dapat melihat ringkasan cepat: Berapa banyak lalu lintas data (*requests*) hari ini? Seberapa cepat sistem merespons (*latency*)? Apakah ada persentase kegagalan (*error rate*)?
- Terdapat grafik berbentuk gelombang yang menunjukkan kapan jam-jam sibuk terjadi.
- Anda juga bisa melihat daftar layanan mana yang sedang mengalami masalah jaringan (pada bagian *Circuit Breakers*).

## 2. Request Logs (Catatan Lalu Lintas)
Ini adalah "buku tamu" sistem. Setiap klik atau transaksi yang melewati Gateway tercatat di sini.
- Jika ada pelanggan yang mengeluh transaksinya gagal, Anda bisa mencari transaksinya di sini menggunakan *Filter* (penyaring).
- Anda bisa menekan salah satu baris log untuk memunculkan detail lengkap (seperti isi pesan yang dikirim dan kapan tepatnya terjadi).
- Informasi pribadi (seperti nomor telepon atau surel) disembunyikan (*redacted*) di sini demi keamanan.

## 3. Gateway Fees (Pendapatan Integrasi)
Halaman ini khusus untuk tim Keuangan/Auditor.
- Menampilkan total pendapatan bersih dari potongan 0.5% pada setiap transaksi berhasil.
- Terdapat grafik berbentuk donat (kue) yang menunjukkan aplikasi mana yang memberikan sumbangan keuntungan terbesar (Misalnya, apakah dari *PasarKita* atau *LogistiKita*?).
- **Pending/Failed Fees**: Jika sistem gagal memotong uang 0.5% dari pengguna (misalnya saldo tidak cukup saat pemotongan fee), transaksinya akan masuk ke tabel kegagalan di bagian bawah, agar tim keuangan bisa menagihnya nanti (rekonsiliasi).

## 4. Route Registry (Daftar Rute)
Ini adalah buku telepon Gateway.
- Di sini, Anda mendaftarkan alamat aplikasi kelompok lain.
- Misalnya, jika *WarungPOS* pindah alamat dari `port 8003` menjadi `port 9000`, Anda cukup menekan tombol pensil (Edit) di sini, mengganti angkanya, dan menekan *Save Changes*. Anda tidak perlu mematikan peladen sama sekali.
- Anda juga bisa mematikan sementara (*Disable*) sebuah rute jika aplikasinya sedang dalam perbaikan.

## 5. Service Health (Kesehatan Layanan)
Layar radar pemantauan jaringan.
- Memperlihatkan kartu-kartu status untuk setiap layanan aplikasi. Warna hijau berarti Sehat (Sirkuit Tertutup), warna merah berarti Rusak (Sirkuit Terbuka).
- Jika ada layanan yang warnanya merah, Gateway sedang berhenti mengirimkan lalu lintas ke sana untuk sementara waktu demi mencegah kemacetan total.

## 6. Security & JWT (Keamanan)
Mengatur kunci-kunci rahasia untuk sistem tiket masuk (Token).
- Biasanya halaman ini hanya disentuh oleh Administrator Utama (AdminFull) setiap 90 hari sekali untuk mengganti kunci gembok sistem (*Key Rotation*) agar tetap aman dari peretasan.
- Menampilkan daftar tiket (Token) yang diblokir, misalnya jika ada pengguna yang mencuri akun pengguna lain.

## 7. Pengaturan & Konfigurasi
Halaman untuk mengubah aturan-aturan utama secara langsung, seperti mengubah persentase *Fee*, mengubah jumlah percobaan ulang (*Max Retries*), atau membatasi seberapa banyak pengguna boleh mengirim permintaan dalam semenit (*Rate Limiter*).
