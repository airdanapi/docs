# 09. Pemecahan Masalah (Troubleshooting)

Dokumen ini berisi solusi cepat jika Anda sebagai Operator menemui kendala saat memantau sistem.

### Masalah 1: Mengapa aplikasi *PasarKita* tiba-tiba tidak bisa dibuka sama sekali?
**Cek Pertama:** Buka Dasbor, pergi ke **Service Health**.
**Analisis:** Lihat kartu *PasarKita*. Jika warnanya merah (*OPEN*), artinya *PasarKita* sedang bermasalah dan Gateway berhenti mengirim pesan ke sana (Sirkuit Terbuka).
**Solusi:**
1. Hubungi tim *PasarKita* untuk memperbaiki peladen mereka.
2. Setelah mereka mengabarkan sudah diperbaiki, sirkuit biasanya akan tertutup sendiri dalam 60 detik.
3. Jika Anda tidak sabar, Anda bisa menekan tombol "Force Probe" untuk memaksa Gateway mencoba mengirim pesan ke *PasarKita* sekarang juga.

### Masalah 2: Ada pelanggan yang uangnya terpotong, tapi pesanannya gagal.
**Cek Pertama:** Buka halaman **Request Logs**.
**Analisis:**
1. Cari berdasarkan `user_id` pelanggan tersebut.
2. Anda akan menemukan log transaksinya. Klik log tersebut.
3. Lihat bagian `response_body` (atau `status_code`). Anda akan tahu apakah kegagalan terjadi di Gateway (kode 4xx/5xx) atau di aplikasi toko (kode 200 tapi isinya pesan gagal dari toko).
**Solusi:** Anda bisa memberikan *Request ID* ini kepada tim pengembang (*Developer*) sebagai bukti pelacakan agar mereka bisa memeriksa di peladen toko.

### Masalah 3: Dasbor Keuangan (Gateway Fees) menunjukkan banyak status FAILED.
**Cek Pertama:** Buka **Gateway Fees**.
**Analisis:** "FAILED" berarti Gateway gagal memotong biaya 0.5% dari pengguna setelah mencoba 5 kali berturut-turut. Ini bisa terjadi karena saldo pengguna di *SmartBank* ternyata habis, atau *SmartBank* sedang terputus lama.
**Solusi:**
Tindakan ini memerlukan intervensi manusia. Klik status FAILED tersebut, lalu koordinasikan dengan tim Keuangan manual (Rekonsiliasi), apakah tagihan akan diabaikan atau ditagih di transaksi berikutnya.

### Masalah 4: Dasbor menampilkan pesan Error `Failed to fetch`.
**Cek Pertama:** Terminal Anda.
**Analisis:** Ini artinya Dasbor Web Anda kehilangan kontak dengan Peladen Backend (Mesin Utama).
**Solusi:**
Buka terminal dan pastikan perintah `go run cmd/gateway/main.go` masih berjalan. Jika terhenti, jalankan ulang perintah tersebut.
