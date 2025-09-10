# Write-up: OWASP Juice Shop – Injection > **Christmas Special (Ephemeral Accountant)**

---

## Gambaran singkat

* **Target**: order produk “**Christmas Super-Surprise-Box (2014 Edition)**” yang sudah tidak tampil di katalog.
* **Inti masalah**: endpoint pencarian hanya menampilkan produk dengan `deletedAt IS NULL`, tapi filter ini bisa kita *bypass* lewat SQLi.
* **Rujukan**: OWASP Juice Shop – Injection Challenges.

---

## Inti serangan (konsep)

Endpoint search kurang lebih membangun kueri seperti:

```sql
SELECT id,name,description,price,deluxePrice,image,createdAt,updatedAt,deletedAt
FROM Products
WHERE (name LIKE '%<q>%' OR description LIKE '%<q>%')
  AND deletedAt IS NULL
ORDER BY name;
```

Kalau kita menutup string `q`, menutup kurung, lalu *comment out* sisa kueri, bagian `AND deletedAt IS NULL` ikut terbuang.
**Payload inti (URL-encoded)**:

```
%27))--
```

`%27` = `'`, `))` menutup kurung, `--` mengubah sisa kueri jadi komentar.

Gabungkan dengan kata kunci “christmas” supaya hasilnya terarah:

```
christmas%27))--
```

---

## Langkah uji 

1. **Lihat bentuk respons normal**
   Akses `GET /rest/products/search?q=apple` → hanya produk dengan `"deletedAt": null` yang muncul.
2. **Bypass filter soft delete**
   Kirim `GET /rest/products/search?q=christmas%27))--`
   Hasilnya memuat item tersembunyi **Christmas Super-Surprise-Box (2014 Edition)**. Catat **`id`**-nya.
3. **Tangkap request tambah ke keranjang**
   Tambahkan produk apa saja dari UI agar kita mendapat format request `POST /api/BasketItems`. Di Network/Proxy terlihat body seperti:

   ```json
   {
     "ProductId": 1,
     "BasketId": <id_keranjang>,
     "quantity": 1
   }
   ```
4. **Edit & kirim ulang**
   Ubah `ProductId` menjadi **id** si “Christmas Box” dari langkah (2), lalu kirim. Respons sukses akan menambah item tersebut ke keranjang.
5. **Checkout**
   Buka halaman **Basket** → **Place your order and pay**. Muncul notifikasi hijau bahwa tantangan “**Christmas Special (Order the Christmas special offer of 2014.)**” terselesaikan.

---

## Bukti & Observasi

* Hasil search dengan payload `christmas%27))--` menampilkan produk yang sudah di-*soft delete*.
* `POST /api/BasketItems` berhasil saat `ProductId` dipaksa ke id produk “Christmas Box”, lalu order dapat diselesaikan.

---

## Akar masalah

* Query search dibangun tanpa **prepared/parameterized statement**.
* Input `q` tidak divalidasi/di-*escape* sehingga karakter `'` dan `--` dapat memanipulasi klausa `WHERE`.
* Logika bisnis “hanya tampilkan yang `deletedAt IS NULL`” dilakukan di SQL, bukan di lapisan yang aman dari injeksi.

---

## Vektor serangan 

* **Endpoint**: `GET /rest/products/search?q=`
* **Parameter rentan**: `q`
* **Payload contoh**: `christmas%27))--`
* **Follow-up**: `POST /api/BasketItems` dengan **ProductId** hasil injeksi
* **Dampak**: akses & pembelian item yang seharusnya tersembunyi (soft-deleted/promo lama).

---

## Insight yang didapat

* Menutup string + menutup kurung + komentar (`%27))--`) ampuh untuk mematikan klausa tambahan seperti `AND deletedAt IS NULL`.
* UI membatasi, tapi **API** tetap menerima `ProductId` apa pun → front-end bukan kontrol keamanan.
* Teknik ini tipikal untuk *recon* + *business logic bypass* via **SQL injection**.

---

## Rekomendasi mitigasi

* Seluruh query harus **parameterized** (prepared statements).
* Terapkan **input validation/escaping** untuk parameter pencarian (khususnya `'`, `%`, `_`).
* Pindahkan aturan bisnis (mis. filter soft delete) ke level aplikasi/ORM yang aman dari injeksi.
* Kembalikan pesan error yang generik; detail cukup di server log.
* Tambahkan **authorization** di endpoint keranjang agar hanya `ProductId` valid & aktif yang bisa ditambahkan.
* Uji berkala (SAST/DAST) untuk pola UNION/boolean/error-based SQLi.
