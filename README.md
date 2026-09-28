# Script Labs API Test Automation

Repo ini berisi test otomatis untuk API di website [labs.hendri.me](https://labs.hendri.me). Test dibuat pakai Postman, dijalankan lewat Newman (Postman versi command line), dan otomatis jalan di GitHub Actions setiap ada push atau pull request ke `main`.

## Apa saja yang dites

Ada 15 request yang dibagi ke 4 folder:

- 01 - Auth: login pakai email dan password. Token hasil login otomatis disimpan dan dipakai oleh semua request lain, jadi tidak perlu di-copy manual.
- 02 - Auth Negative: memastikan API menolak login dengan password salah atau email tidak valid, serta menolak akses ke `/api/labs` tanpa token atau dengan token palsu.
- 03 - Labs CRUD: alur tambah, lihat, ubah, lalu hapus data lab. Setelah dihapus, dicek lagi bahwa datanya sudah hilang.
- 04 - Data-Driven (CSV): satu request yang dijalankan berulang pakai data dari file CSV. Satu baris CSV berarti satu kali percobaan.

<details>
<summary>Daftar lengkap 15 request</summary>

| No | Request | Method | Endpoint | Hasil yang diharapkan |
|---|---|---|---|---|
| 1 | Login - Valid Credentials | POST | /api/auth/login | 200 |
| 2 | Login - Wrong Password | POST | /api/auth/login | 401 |
| 3 | Login - Invalid Email Format | POST | /api/auth/login | 400 |
| 4 | Get All Labs - Without Token | GET | /api/labs | 401 |
| 5 | Get All Labs - Invalid Token | GET | /api/labs | 401 |
| 6 | Create Lab | POST | /api/labs | 201 |
| 7 | Get All Labs | GET | /api/labs | 200 |
| 8 | Get Lab by ID | GET | /api/labs/{{lab_id}} | 200 |
| 9 | Update Lab | PUT | /api/labs/{{lab_id}} | 200 |
| 10 | Get Lab by ID - Verify Update | GET | /api/labs/{{lab_id}} | 200 |
| 11 | Update Lab - Non-existent ID | PUT | /api/labs/999999999 | 404 |
| 12 | Delete Lab | DELETE | /api/labs/{{lab_id}} | 200 |
| 13 | Get Lab by ID - After Delete | GET | /api/labs/{{lab_id}} | 404 |
| 14 | Delete Lab - Already Deleted | DELETE | /api/labs/{{lab_id}} | 404 |
| 15 | Create Lab - Data Driven (CSV) | POST | /api/labs | sesuai isi CSV |

</details>

Setiap request mengecek tiga hal: status code benar, isi response sesuai, dan waktu respons di bawah 5 detik.

File [data/labs-data.csv](data/labs-data.csv) berisi 14 skenario untuk field `title` dan `description`:

- Valid: input biasa dan input dengan karakter spesial. API harus menerimanya.
- Edge case: input tepat di batas aturan, misalnya title 255 karakter atau description 1000 karakter. API masih harus menerimanya.
- Invalid: input yang melanggar aturan, misalnya title kosong, title 256 karakter, atau field tidak dikirim. API harus menolaknya dengan pesan error yang tepat.

Data yang berhasil dibuat selama test langsung dihapus lagi, jadi akun test tetap bersih.

## Isi folder

| Folder / file | Isi |
|---|---|
| .github/workflows/api-test.yml | Pengaturan GitHub Actions: kapan dan bagaimana test dijalankan |
| collections/ | Postman collection berisi semua request dan script test |
| environments/ | Variable seperti alamat API (tanpa password) |
| data/labs-data.csv | Data untuk test berulang (data-driven) |
| docs/screenshots/ | Bukti hasil run |
| package.json | Daftar tool yang dipakai (Newman) dan perintah untuk menjalankan test |

Dua hal sengaja tidak ikut di-upload ke GitHub:

- `environments/local.postman_environment.json`, karena berisi email dan password asli.
- `reports/`, karena berisi laporan test yang dibuat ulang setiap kali test dijalankan.

## Cara menjalankan di komputer sendiri

Yang perlu disiapkan: [Node.js](https://nodejs.org) versi 18 ke atas dan akun di [labs.hendri.me](https://labs.hendri.me) (bisa daftar gratis di website-nya).

1. Install Newman:

   ```bash
   npm install
   ```

2. Salin file environment bawaan menjadi file lokal:

   ```bash
   # Mac / Linux / Git Bash
   cp environments/script-labs.postman_environment.json environments/local.postman_environment.json

   # Windows
   copy environments\script-labs.postman_environment.json environments\local.postman_environment.json
   ```

   Buka `local.postman_environment.json`, lalu isi `value` untuk `email` dan `password` dengan akun kamu. File ini tidak ikut ter-upload ke GitHub.

3. Jalankan test:

   ```bash
   npm run test:crud   # login, test negatif, dan CRUD
   npm run test:ddt    # test berulang pakai data CSV
   npm test            # keduanya sekaligus
   ```

Hasilnya muncul di terminal. Laporan versi HTML tersimpan di folder `reports/` dan bisa dibuka pakai browser.

Kalau mau pakai aplikasi Postman: import file di folder `collections/` dan `environments/`, isi email dan password di environment, lalu klik Send di request Login. Setelah itu request lain bisa langsung dijalankan.

## Cara kerja di GitHub Actions

Setiap ada push atau pull request ke `main`, GitHub otomatis:

1. Mengambil kode terbaru
2. Menyiapkan Node.js dan meng-install Newman
3. Menjalankan test login dan CRUD
4. Menjalankan test data-driven dari CSV
5. Menyimpan laporan hasil test (bisa diunduh dari halaman run di tab Actions)

Email dan password tidak ditulis di kode. Keduanya disimpan sebagai GitHub Secrets bernama `API_EMAIL` dan `API_PASSWORD` (menu Settings, Secrets and variables, Actions).

## Bukti pipeline sebagai gatekeeper

Branch `main` diatur supaya pull request hanya bisa di-merge kalau semua test lulus. Untuk membuktikannya, dibuat [Pull Request #1](https://github.com/pramikarega/final-assignment-api-automation/pull/1):

1. Test sengaja dibuat salah (commit `d58661a`). Satu baris CSV diubah seolah-olah API harus menerima title 300 karakter, padahal batasnya 255. Hasilnya test gagal dan tombol merge terkunci. Bukti: [actions-fail-gatekeeper.png](docs/screenshots/actions-fail-gatekeeper.png), [pr-blocked.png](docs/screenshots/pr-blocked.png).
2. Test diperbaiki (commit `c372c77`). Ekspektasinya dikembalikan sesuai aturan: title 300 karakter harus ditolak. Hasilnya test lulus dan PR bisa di-merge. Bukti: [pr-fixed.png](docs/screenshots/pr-fixed.png).

## Screenshot lainnya

- [actions-pass.png](docs/screenshots/actions-pass.png): semua langkah di GitHub Actions berhasil
- [postman-login.png](docs/screenshots/postman-login.png): collection dibuka di Postman, login berhasil dan 5/5 test lulus
