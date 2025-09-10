# Write-up: OWASP Juice Shop – Injection > **Login Admin**

---

## Gambaran singkat

* **Target**: tantangan *Injection* – login menggunakan akun administrator.
* **Tujuan**: membuktikan bahwa validasi input pada form login lemah sehingga bisa dibypass.
* **Sumber rujukan**: OWASP Juice Shop – Injection Challenges.

---

## Inti serangan 

Form login kemungkinan besar menjalankan kueri mirip:

```sql
SELECT * FROM Users 
WHERE email = '<input_email>' AND password = '<input_password>';
```

Jika input email kita isi dengan payload yang menutup string lalu memasukkan kondisi yang selalu benar, pengecekan password jadi “terpotong”.

**Payload yang dipakai**

```text
' OR 1=1--
```

* `'` menutup string email.
* `OR 1=1` selalu bernilai benar.
* `--` mengubah sisa kueri (termasuk cek password) menjadi komentar.

Hasilnya, kondisi `WHERE` selalu *true* dan autentikasi terlewati.


---

## Langkah uji 

1. Buka halaman **Login** di Juice Shop.
2. Pada kolom **Email**, isi dengan:

   ```
   ' OR 1=1--
   ```
3. Kolom **Password** isi sembarang teks (tidak berpengaruh).
4. Tekan **Log in**.
5. Setelah masuk, buka menu **Account** di pojok kanan atas untuk melihat bahwa sesi masuk menggunakan akun admin (alamat seperti `admin@juice-sh.op` akan terlihat).

![](https://media.discordapp.net/attachments/1249245055185715221/1415301313645903923/image.png?ex=68c2b590&is=68c16410&hm=8c505fb7c1e734dd20e106682c0fc82af04aeb82fffc17b16ff6367a5ad7e1df&=&format=webp&quality=lossless&width=1842&height=856)
![](https://media.discordapp.net/attachments/1249245055185715221/1415301999423000710/image.png?ex=68c2b633&is=68c164b3&hm=16477e91db024890d5078ef2646a645e464bd5efb6c4063d0dae66432a71c088&=&format=webp&quality=lossless&width=550&height=263)
---

## Bukti & Observasi

* Login berhasil tanpa mengetahui password admin.
* Dropdown **Account** menampilkan email admin → membuktikan eskalasi sesi.

---

## Akar masalah

* Tidak ada sanitasi/validasi input pada parameter login.
* Kueri dibangun secara *string concatenation* tanpa **prepared statement/parameterized query**.
* Tanda komentar SQL (`--`) memungkinkan sisa kueri (termasuk cek password) diabaikan.

---

## Vektor serangan (ringkas)

* **Endpoint**: form login.
* **Parameter**: `email` (rentan SQLi).
* **Payload**: `' OR 1=1--`
* **Dampak**: bypass autentikasi, akses akun admin.

---

## Insight yang didapat

* Kondisi `1=1` menjadikan klausa `WHERE` selalu benar → proses login dianggap valid.
* Kombinasi penutupan string + komentar efektif mematikan verifikasi password.
* Contoh klasik bagaimana **SQL Injection** bisa langsung menabrak mekanisme login.

---

## Rekomendasi mitigasi

* Gunakan **parameterized queries / prepared statements** di semua akses DB.
* Terapkan **input validation** dan **output encoding**.
* Nonaktifkan **verbose error** yang bisa memberi petunjuk struktur kueri.
* Tambahkan **rate limiting** & **monitoring** pada endpoint autentikasi.
* Lakukan **security testing** rutin (SAST/DAST) untuk mendeteksi pola injeksi.

---
