# 07. Basis Data (Database)

API Gateway menyimpan datanya di dalam sistem **MySQL**. Terdapat empat "tabel" (lemari arsip) utama:

### 1. `routes_registry` (Buku Telepon)
Tabel ini memetakan nama layanan menjadi URL asli.
* **Kolom penting:** `service_name` (mis. marketplace), `feature_name` (mis. checkout), `gateway_path` (alamat perantara), `downstream_url` (alamat asli), `transactional` (apakah rute ini kena pajak 0.5%?).

### 2. `request_logs` (Buku Tamu / Jejak Audit)
Menyimpan sejarah setiap lalu lintas yang lewat.
* **Kolom penting:** `request_id` (kode unik pelacakan), `user_id` (siapa yang mengirim), `source_app` (dari aplikasi mana), `target_app` (menuju aplikasi mana), `latency_ms` (berapa milidetik selesainya), `status_code` (apakah sukses atau gagal).

### 3. `gateway_fees` (Buku Rekening Pemasukan)
Menyimpan daftar potongan 0.5%.
* **Kolom penting:** `request_id` (terikat ke buku tamu), `transaction_amount` (total uang), `fee_amount` (potongan yang didapat Gateway), `status` (PAID/Berhasil, PENDING/Menunggu, DEFERRED/Ditunda, FAILED/Gagal Permanen).

### 4. `jwt_blacklist` (Buku Daftar Hitam)
Tabel untuk mencabut tiket/token JWT sebelum waktu kadaluarsanya habis.
* **Kolom penting:** `jti` (nomor seri token), `user_id` (pemilik token), `revoked_at` (waktu pencabutan), `reason` (alasan diblokir).

### 5. Redis (Memori Singkat)
Selain MySQL, Gateway menggunakan **Redis** yang bekerja super cepat. Redis digunakan untuk dua hal:
1. **Buku Catatan Pembatas Laju (*Rate Bucket*)**: Menghitung berapa kali pengguna menekan tombol dalam semenit terakhir. Kalau sudah lebih dari 60 kali, catatannya akan membuat pengguna diblokir sejenak. Catatan ini dibersihkan otomatis setiap menit.
2. **Kunci Tanda Terima (*Idempotency Cache*)**: Menyimpan jawaban selama 24 jam. Jika pengguna mengirim ID Tanda Terima yang sama, Redis akan langsung memberikan jawaban lama tanpa perlu mengulang pemrosesan.
