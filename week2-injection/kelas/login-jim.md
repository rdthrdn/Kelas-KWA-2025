# Write-up: OWASP Juice Shop – Injection > **Login Jim**

---

## Gambaran singkat

* **Target**: tantangan *Injection* – login sebagai pengguna **Jim**.
* **Tujuan**: membuktikan validasi input pada form login lemah sehingga bisa dibypass tanpa tahu password.
* **Sumber rujukan**: OWASP Juice Shop – Injection Challenges.

---

## Inti serangan (konsep)

Aplikasi kemungkinan mengeksekusi kueri seperti:

```sql
SELECT * FROM Users 
WHERE email = '<input_email>' AND password = '<input_password>';
```

Dengan menutup string email lalu menjadikan sisa kueri sebagai komentar, pengecekan password tidak dievaluasi.

**Payload**

```
jim@juice-sh.op'--
```

* `'` menutup string email yang dibangun dalam kueri.
* `--` membuat bagian setelahnya (termasuk `AND password=...`) diabaikan oleh SQL.

Akibatnya kueri hanya memeriksa `email='jim@juice-sh.op'` dan autentikasi lolos.

---

## Langkah uji 

![](https://media.discordapp.net/attachments/1249245055185715221/1415303510375010326/image.png?ex=68c2b79b&is=68c1661b&hm=d07bf987390701299112d7e576a6ec312d653158b457b3363c1cbe442eaffd5a&=&format=webp&quality=lossless)

1. Buka halaman **Login** di Juice Shop.
2. Dapatkan alamat email Jim (tersedia di ulasan produk/reviews).
3. Isi kolom **Email** dengan:

   ```
   jim@juice-sh.op'--
   ```

   Kolom **Password** isi teks apa saja.

![](https://media.discordapp.net/attachments/1249245055185715221/1415303988173078609/image.png?ex=68c2b80d&is=68c1668d&hm=5e8c0fe3d6edf8a8b23b8e272bfd9043f70ed5637173828a02edab60dc9777ee&=&format=webp&quality=lossless)

4. Tekan **Log in**.
5. Buka menu **Account** di kanan atas untuk melihat bahwa sesi aktif atas nama Jim.
![](https://media.discordapp.net/attachments/1249245055185715221/1415304162153074808/image.png?ex=68c2b837&is=68c166b7&hm=e6155d53cbaaf3f0ed493e05b045aa7688d14b3860bc21041c7145e2d1024d80&=&format=webp&quality=lossless)

---

## Bukti & Observasi

* Login berhasil tanpa mengetahui password.
* Dropdown **Account** menampilkan email Jim → konfirmasi sesi user Jim.

---

## Akar masalah

* Input dimasukkan ke kueri melalui **string concatenation**.
* Tidak menggunakan **prepared/parameterized query**.
* Karakter khusus (`'`, `--`) tidak disaring, memungkinkan injeksi.

---

## Vektor serangan (ringkas)

* **Endpoint**: form login.
* **Parameter rentan**: `email`.
* **Payload**: `jim@juice-sh.op'--`
* **Dampak**: **bypass autentikasi** untuk akun target.

---

## Insight yang didapat

* Teknik `'--` efektif untuk memotong klausa `AND password=...` tanpa perlu kondisi `OR 1=1`.
* Lebih “terarah” dibanding payload *boolean-based* karena langsung menargetkan satu akun (Jim).
* Menunjukkan betapa berbahayanya query yang tidak diparameterisasi.

---

## Rekomendasi mitigasi

* Terapkan **parameterized queries / prepared statements** di semua akses DB.
* Lakukan **input validation** dan **escaping** sesuai konteks.
* Aktifkan **rate limiting** dan **account lockout** di endpoint login.
* Tambahkan **logging & monitoring** untuk pola login anomalis.
* Jalankan **SAST/DAST** secara rutin untuk mendeteksi injeksi.


