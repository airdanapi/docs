# 08. Keamanan

Gateway adalah benteng utama sistem. Jika keamanan Gateway jebol, seluruh ekosistem akan runtuh. Oleh karena itu, kita menggunakan beberapa lapis pengamanan.

## 1. Token JWT (Pemeriksaan KTP)

Gateway tidak sembarangan menerima permintaan.
Setiap orang atau aplikasi yang mengirim pesan harus menyerahkan "KTP digital" yang disebut **JSON Web Token (JWT)**.
1. KTP ini dikeluarkan oleh *SmartBank*.
2. Gateway memeriksa tanda tangan digital (berupa Kunci Publik/Public Key) di KTP tersebut untuk memastikan bahwa KTP itu tidak dipalsukan.
3. KTP ini juga memiliki tanggal kadaluarsa. Jika sudah basi, Gateway akan langsung menolaknya.

## 2. Rotasi Kunci (Key Rotation)

Bayangkan kunci gembok rumah Anda. Kalau Anda menggunakan kunci yang sama selama bertahun-tahun, ada risiko seseorang telah menduplikatnya.
Sistem ini mewajibkan Administrator untuk menukar (merotasi) kunci utama setiap **90 hari**. Kunci yang lama akan diberikan waktu transisi (*grace period*) selama 30 hari sebelum benar-benar tidak bisa dipakai lagi, agar pengguna punya waktu untuk login ulang.

## 3. Daftar Hitam (Blacklist)

Jika ada pengguna yang kehilangan kata sandinya atau akunnya diretas, Administrator dapat memasukkan ID KTP pengguna tersebut ke dalam Daftar Hitam (melalui halaman Security & JWT di Dasbor).
Gateway akan langsung memblokir semua KTP yang masuk dalam daftar ini meskipun KTP-nya belum kadaluarsa.

## 4. Keamanan Halaman Operator

Dasbor Web (Konsol Integrator) tempat operator bekerja juga diamankan:
- Tidak sembarang orang bisa masuk. Hanya akun khusus seperti `Operator`, `FinanceAuditor`, dan `AdminFull` yang bisa masuk.
- Tindakan berbahaya (seperti mengubah Rute atau memblokir pengguna) mewajibkan konfirmasi (*Confirm Dialog*) untuk mencegah operator salah tekan.
- Setiap kali Operator melakukan tindakan di Konsol, nama operator dan apa yang ia lakukan akan dicatat di buku tamu (*Audit Trail*).
