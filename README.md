# Geothermal EPC Cost Estimator — Panduan Hosting Online

Tool ini sekarang bisa jalan dengan dua cara **sekaligus**, memakai Google Sheet yang sama sebagai penyimpanan skenario:

- **Dibuka lewat GitHub Pages** (atau hosting statis lain) — untuk dibagikan sebagai tools online.
- **Dibuka lewat URL Apps Script (`/exec`)** — cara lama, tetap berfungsi persis seperti sebelumnya.

Perubahan utama ada di `Code.gs`: sekarang backend juga melayani permintaan REST (`fetch`) dari domain luar, dengan opsi token supaya tidak sembarang orang bisa menulis ke Sheet Anda.

---

## Bagian 1 — Deploy backend (Google Apps Script)

1. Buat Google Sheet baru → **Extensions > Apps Script**.
2. Timpa isi `Code.gs` bawaan dengan isi `Code.gs` di paket ini.
3. **File > New > HTML**, beri nama persis **`Index`** (jangan `Index.html`, Apps Script otomatis menambah ekstensinya). Tempel seluruh isi `Index.html` ke sana.
4. *(Sangat disarankan)* Klik ikon gerigi **Project Settings** → **Script Properties** → **Add script property**:
   - Key: `API_TOKEN`
   - Value: string acak yang panjang, contoh: `sgc7-K9pQ2vXz-mR4tYw8` (buat sendiri, jangan pakai contoh ini)

   Ini mencegah orang lain menulis skenario sampah ke Sheet Anda kalau URL Web App bocor.
5. Jalankan fungsi `setupSheet` sekali (pilih dari dropdown fungsi di toolbar editor → klik **Run** ▶). Ini akan meminta izin akses — setujui. Sheet **"Scenarios"** akan otomatis dibuat.
6. **Deploy > New deployment**:
   - Type: **Web app**
   - Execute as: **Me**
   - Who has access: **Anyone** (wajib "Anyone", bukan "Anyone within [organisasi]", supaya bisa diakses dari GitHub Pages)
7. Salin **Web app URL** yang diberikan (diakhiri `/exec`). Simpan — ini yang dipakai di Bagian 2.

> Setiap kali mengubah kode `Code.gs` atau `Index.html` di editor Apps Script, gunakan **Manage deployments** → ikon pensil → **New version** → **Deploy**, agar perubahan terlihat di URL yang sudah dibagikan (URL `/exec` tidak berubah).

---

## Bagian 2 — Hosting frontend ke GitHub Pages

1. Buat repo baru di GitHub, upload file `Index.html` dari paket ini (bisa direname jadi `index.html` huruf kecil semua, standar GitHub Pages).
2. **Settings > Pages** pada repo tersebut → Source: **Deploy from a branch** → pilih branch `main` dan folder `/ (root)` → **Save**.
3. Tunggu 1-2 menit, GitHub akan memberi URL seperti `https://<username>.github.io/<nama-repo>/`. Buka URL itu — kalkulator akan langsung jalan (semua hitungan & chart tetap 100% jalan tanpa backend).
4. Untuk mengaktifkan simpan/muat skenario ke Google Sheet: buka sidebar tool → gulir ke bawah ke panel **"Sinkronisasi ke Google Sheet"** → isi:
   - **URL Web App**: URL `/exec` dari Bagian 1 langkah 7
   - **Token API**: isi persis sama dengan `API_TOKEN` yang Anda set di langkah 4 (kosongkan kalau tidak pakai token)
   - Klik **Simpan Koneksi**, lalu **Tes Koneksi** untuk memastikan status "✅ Terhubung".

Setelah itu tombol **Simpan** dan **Muat dari Google Sheet** di sidebar akan membaca/menulis ke Sheet yang sama seperti versi Apps Script.

Koneksi ini tersimpan di `localStorage` browser masing-masing pengguna — setiap orang yang membuka tool dari GitHub Pages perlu mengisi URL + token sendiri sekali (atau Anda bagikan URL + token itu kepada tim yang berhak).

---

## Kenapa tidak bisa langsung pakai `google.script.run` di GitHub Pages?

`google.script.run` hanya tersedia di dalam sandbox Apps Script (saat halaman dibuka lewat URL `script.google.com/.../exec`). Begitu `Index.html` di-hosting di domain lain (GitHub Pages, Netlify, dst), objek itu tidak ada sama sekali. Solusinya: `Code.gs` sekarang juga menerima request HTTP biasa lewat `doGet(e)` / `doPost(e)` (dengan `?action=list`, `?action=save`, dst), dan `Index.html` otomatis mendeteksi mode mana yang aktif:

- **`IS_GAS = true`** → pakai `google.script.run` (perilaku lama, tidak berubah).
- **`IS_GAS = false`** + URL Web App sudah diisi → pakai `fetch()` ke endpoint REST di atas.
- **`IS_GAS = false`** + URL belum diisi → mode standalone, kalkulator tetap jalan penuh, hanya simpan/muat skenario yang nonaktif.

Request `POST` dikirim dengan `Content-Type: text/plain` (bukan `application/json`) supaya browser tidak mengirim *preflight* `OPTIONS` — Apps Script Web App tidak mengimplementasikan handler untuk itu, jadi kalau pakai `application/json` permintaan akan gagal karena CORS.

## Varian "sudah tertanam" (hardcoded) untuk dibagikan ke tim internal

Kalau Anda memilih menanam URL Web App + `API_TOKEN` langsung di dalam `index.html` (lewat konstanta `DEFAULT_GAS_URL` / `DEFAULT_GAS_TOKEN`) supaya rekan tim tinggal buka link tanpa isi apa pun — itu valid untuk dibagikan terbatas ke tim, TAPI perlu diingat:

- Siapa pun yang buka **View Page Source** di browser bisa melihat URL dan token tersebut.
- Kalau repo GitHub-nya **publik**, token otomatis ikut publik. Pakai repo **private**, atau terima risikonya kalau memang cuma untuk mencegah spam biasa (bukan melindungi data rahasia).
- Panel "Sinkronisasi ke Google Sheet" di sidebar tetap bisa dipakai untuk override manual (mis. kalau nanti ganti ke Sheet lain) — nilai yang diisi manual di situ akan menimpa nilai default yang tertanam.

## Keamanan & batasan yang perlu diketahui

- Kalau **`API_TOKEN` dikosongkan**, siapa pun yang tahu URL Web App bisa menulis (dan membaca) baris skenario ke Sheet Anda. Untuk tool yang dibagikan publik, sebaiknya selalu isi token.
- Token dikirim sebagai parameter biasa (bukan header khusus) supaya kompatibel dengan `doGet`/`doPost` Apps Script — ini setara dengan API key sederhana, bukan otentikasi penuh. Jangan gunakan untuk data sensitif/rahasia perusahaan.
- Kalau butuh kontrol lebih ketat (login per-user, hak akses berbeda per orang), itu di luar cakupan perubahan ini dan perlu arsitektur tambahan (mis. OAuth, Google Identity).
