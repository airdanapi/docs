# 01. Gambaran Umum

## Apa itu API Gateway?

Bayangkan sebuah pusat perbelanjaan besar. Di pintu masuk utama, terdapat satpam yang memeriksa tiket pengunjung, memastikan mereka tidak membawa barang berbahaya, dan mengarahkan mereka ke toko yang tepat. 

Dalam Ekosistem Ekonomi UMKM, **API Gateway** (atau Integrator) bertindak persis seperti pintu masuk utama tersebut. API Gateway adalah satu-satunya jalur yang menghubungkan tujuh aplikasi berbeda (seperti *SmartBank*, *PasarKita*, dan *WarungPOS*). 

Tidak ada aplikasi yang boleh saling "berbicara" langsung satu sama lain lewat pintu belakang. Semuanya harus melewati pintu masuk utama ini.

## Mengapa Kita Membutuhkannya?

Jika ketujuh aplikasi saling terhubung secara langsung, sistem akan menjadi sangat rumit seperti benang kusut. Jika satu aplikasi berpindah alamat, enam aplikasi lainnya harus diubah satu per satu. Dengan adanya API Gateway, kita mendapatkan beberapa keuntungan besar:

1. **Keamanan Terpusat**: Gateway memastikan setiap permintaan yang masuk memiliki "tiket" (Token JWT) yang sah. Jika tiketnya palsu atau kadaluarsa, Gateway akan langsung menolaknya.
2. **Pencatatan yang Rapi (Buku Tamu)**: Setiap permintaan yang lewat dicatat dengan rapi. Kapan permintaan itu datang, dari siapa, mau ke mana, dan apakah berhasil atau gagal. Ini sangat penting untuk pelacakan masalah.
3. **Papan Penunjuk Jalan (Routing)**: Gateway tahu persis di mana setiap layanan berada. Aplikasi pengirim hanya perlu tahu alamat Gateway, dan Gateway akan meneruskannya ke tujuan yang benar.
4. **Pemungutan Biaya Otomatis**: Setiap kali ada transaksi jual beli, Gateway akan memotong sedikit biaya layanan (sebesar 0.5%) secara otomatis sebagai pendapatan operasional sistem.
5. **Polisi Lalu Lintas (Pembatas Laju)**: Jika ada pengguna yang mengirim permintaan terlalu banyak dalam waktu singkat (spam), Gateway akan memperlambat atau menolaknya sementara agar sistem tidak mati (down).

## Apa yang TIDAK Dilakukan Gateway?

Agar Gateway tetap cepat dan efisien, ada batasan ketat tentang apa yang *tidak* boleh dilakukannya:

- **Bukan Pengelola Uang**: Gateway tidak pernah menambah atau mengurangi saldo secara langsung. Ia selalu meminta *SmartBank* untuk melakukannya.
- **Bukan Otak Bisnis**: Gateway tidak mengurus stok barang, harga produk, atau logika keranjang belanja. Itu adalah tugas aplikasi masing-masing.
- **Bukan Aplikasi Pengguna Akhir**: Pembeli atau pemilik warung tidak pernah melihat bentuk Gateway. Mereka menggunakan aplikasi *PasarKita* atau *WarungPOS*, lalu aplikasi-aplikasi itulah yang diam-diam "berbicara" dengan Gateway di latar belakang.

## Antarmuka Pemantauan (Konsol Integrator)

Meskipun Gateway bekerja di latar belakang, tim pengelola memiliki sebuah antarmuka web khusus (Dashboard/Konsol). Melalui konsol ini, tim pengelola dapat memantau kesehatan seluruh sistem, melihat grafik lalu lintas data, memeriksa pendapatan biaya layanan, dan mengatur rute jika ada perubahan alamat aplikasi.
