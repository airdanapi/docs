# 05. Panduan Integrasi (Menyambungkan Aplikasi Anda)

Dokumen ini ditujukan untuk **developer dari kelompok lain** (seperti tim *PasarKita*, *SmartBank*, *WarungPOS*, dsb.) yang ingin agar aplikasinya bisa diakses melalui API Gateway.

## 1. Daftarkan Aplikasi Anda

Sebelum aplikasi Anda bisa menerima pesan, Anda harus meminta kelompok 7.0 (pengelola Gateway) untuk memasukkan alamat aplikasi Anda ke dalam sistem mereka (Route Registry). 
- Contoh: Anda meminta agar permintaan `/api/v1/PasarKita/checkout` diteruskan ke peladen Anda di `http://localhost:8002/checkout`.

## 2. Kunci Pintu (Token JWT)

Untuk melewati penjagaan Gateway, **setiap permintaan harus menyertakan "Tiket Masuk"** (Token JWT).
Tiket ini didapatkan ketika pengguna Anda masuk (login) ke dalam sistem *SmartBank*.

Cara menyertakan tiket masuk dalam permintaan Anda adalah dengan menambahkannya di bagian tajuk (*Header*):
```http
Authorization: Bearer <TOKEN_JWT_DARI_SMARTBANK>
```

## 3. Cara Mengirim Permintaan (2 Cara)

Jika aplikasi A ingin memanggil aplikasi B, tidak boleh dilakukan secara langsung! Anda harus memanggil Gateway di `http://localhost:8080`.

**Cara A (Pola Transparan / Paling Gampang):**
Tembak saja URL Gateway, dan tambahkan nama layanan serta fiturnya di belakang `/api/v1/`.
*Contoh:* Anda ingin memanggil fitur `checkout` di `PasarKita`.
```bash
POST http://localhost:8080/api/v1/marketplace/checkout
```
*Catatan: Pastikan nama layanan (`marketplace`) sama persis dengan yang didaftarkan di Route Registry.*

**Cara B (Pola Amplop / Envelope):**
Kirimkan permintaan Anda menggunakan metode POST ke rute khusus Gateway, dan bungkus tujuan akhir Anda di dalam pesannya (body JSON).
```bash
POST http://localhost:8080/integrator/routing_api
Body JSON:
{
  "target_service": "marketplace",
  "target_feature": "checkout",
  "method": "POST",
  "payload": {
    "barang_id": "123",
    "jumlah": 2
  }
}
```

## 4. Transaksi Uang & Tanda Terima (Idempotency)

Jika permintaan yang Anda buat akan melibatkan perpindahan uang (transaksional), Gateway akan memotong **biaya layanan sebesar 0.5%**.

Untuk mencegah uang pengguna terpotong ganda jika terjadi koneksi putus lalu Anda menekan tombol bayar dua kali, **Anda diwajibkan** mengirimkan satu tajuk (*Header*) tambahan berupa Tanda Terima unik (*Idempotency Key*).

```http
X-Idempotency-Key: efe91b2c-3a33-4f11-9a70-8b1b8b2b3a9a
```
Gunakan kode acak (*UUID*) yang berbeda setiap kali ada transaksi baru. Jika Anda menggunakan kode yang sama dua kali, Gateway akan mengembalikan jawaban transaksi lama tanpa memotong uang lagi.

## 5. Batasan Laju (Rate Limit)

Jangan membombardir Gateway.
- Untuk kegiatan biasa (seperti melihat daftar barang): Maksimal **60 kali per menit**.
- Untuk kegiatan transaksional (seperti bayar keranjang): Maksimal **10 kali per menit**, dan Anda harus memberi jeda setidaknya 10 detik di antara pembayaran.
Jika Anda melebihi batas, Gateway akan menolak Anda dengan jawaban "Terlalu Banyak Permintaan" (Status HTTP 429).
