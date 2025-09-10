# Write-up: OWASP Juice Shop – Injection > **Login Bender**

> Catatan: uji ini dikerjakan di **OWASP Juice Shop** (lab yang memang dibuat rentan) untuk pembelajaran.

---

## Gambaran singkat

* **Target**: tantangan *Injection* – login sebagai pengguna **Bender**.
* **Tujuan**: menunjukkan form login bisa dibypass tanpa tahu password.
* **Sumber rujukan**: OWASP Juice Shop – Injection Challenges.

---

## Inti serangan (konsep)

Aplikasi kemungkinan menjalankan kueri seperti:

```sql
SELECT * FROM Users 
WHERE email = '<input_email>' AND password = '<input_password>';
```

Kalau kita **menutup string email** lalu **mengomentari sisa kueri**, bagian cek password tidak ikut dieksekusi.

**Payload**

```
bender@juice-sh.op'--
```

* `'` menutup string email.
* `--` membuat sisa kueri (termasuk `AND password=...`) diabaikan.
  Akhirnya kueri hanya memeriksa `email='bender@juice-sh.op'`.

---

## Langkah uji 

1. Buka halaman **Login**.
2. Dapatkan alamat email Bender (bisa dari **product reviews**).
3. Isi **Email** dengan:

   ```
   bender@juice-sh.op'--
   ```

   **Password** isi apa saja.
4. Klik **Log in**.
5. Buka menu **Account** di kanan atas untuk memastikan sesi masuk sebagai **Bender**.
![](https://media.discordapp.net/attachments/1249245055185715221/1415305417998729226/image.png?ex=68c2b962&is=68c167e2&hm=0a2abf462f0078d504e222c944225f9cb480ee9efce41c950b33633be32d0a9a&=&format=webp&quality=lossless)
---

## Bukti & Observasi

* Login berhasil tanpa password valid.
* Dropdown **Account** menampilkan email milik Bender.

---

## Akar masalah

* Query dibangun via **string concatenation**.
* Tidak memakai **prepared/parameterized statement**.
* Karakter khusus (`'`, `--`) tidak disaring.

---

## Vektor serangan 

* **Endpoint**: form login.
* **Parameter rentan**: `email`.
* **Payload**: `bender@juice-sh.op'--`
* **Dampak**: **bypass autentikasi** dan masuk ke akun Bender.

---

## Insight yang didapat

* Teknik `'--` efektif memotong klausa `AND password=...` sehingga password jadi tidak relevan.
* Ini **injeksi terarah**: kita memilih akun target (Bender).
* Payload generik seperti `' OR 1=1--` juga bisa bypass, tapi tidak menjamin akun tertentu.

---

## Rekomendasi mitigasi

* Wajib gunakan **parameterized queries / prepared statements**.
* Terapkan **validasi & escaping input**.
* Tambahkan **rate limiting / account lockout** di endpoint login.
* Logging & monitoring untuk pola login mencurigakan.
* Jalankan **SAST/DAST** rutin.


