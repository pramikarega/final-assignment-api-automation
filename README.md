# Script Labs API Test Automation

API test automation untuk [labs.hendri.me](https://labs.hendri.me) (base API: `https://api-script-labs.hendri.me`) memakai **Postman + Newman**, dengan **data-driven testing (CSV)** dan **CI/CD GitHub Actions**.

![API Test Automation](https://github.com/pramikarega/final-assignment-api-automation/actions/workflows/api-test.yml/badge.svg)

## Cakupan Test

Collection: [`collections/script-labs-api.postman_collection.json`](collections/script-labs-api.postman_collection.json) berisi **15 request** dalam 4 folder.

| # | Folder | Request | Method | Endpoint | Expected |
|---|---|---|---|---|---|
| 1 | `01 - Auth` | Login - Valid Credentials (simpan token) | `POST` | `/api/auth/login` | 200 |
| 2 | `02 - Auth Negative` | Login - Wrong Password | `POST` | `/api/auth/login` | 401 |
| 3 | | Login - Invalid Email Format | `POST` | `/api/auth/login` | 400 |
| 4 | | Get All Labs - Without Token | `GET` | `/api/labs` | 401 |
| 5 | | Get All Labs - Invalid Token | `GET` | `/api/labs` | 401 |
| 6 | `03 - Labs CRUD` | Create Lab | `POST` | `/api/labs` | 201 |
| 7 | | Get All Labs | `GET` | `/api/labs` | 200 |
| 8 | | Get Lab by ID | `GET` | `/api/labs/{{lab_id}}` | 200 |
| 9 | | Update Lab | `PUT` | `/api/labs/{{lab_id}}` | 200 |
| 10 | | Get Lab by ID - Verify Update | `GET` | `/api/labs/{{lab_id}}` | 200 |
| 11 | | Update Lab - Non-existent ID | `PUT` | `/api/labs/999999999` | 404 |
| 12 | | Delete Lab | `DELETE` | `/api/labs/{{lab_id}}` | 200 |
| 13 | | Get Lab by ID - After Delete | `GET` | `/api/labs/{{lab_id}}` | 404 |
| 14 | | Delete Lab - Already Deleted | `DELETE` | `/api/labs/{{lab_id}}` | 404 |
| 15 | `04 - Data-Driven (CSV)` | Create Lab - Data Driven (CSV) | `POST` | `/api/labs` | dari CSV (14 baris) |

Semua URL memakai variable `{{baseUrl}}`. Hasil eksekusi setiap request (method, URL, status, waktu) bisa dilihat di log step *Run Auth + CRUD tests* pada tab [Actions](https://github.com/pramikarega/final-assignment-api-automation/actions/workflows/api-test.yml).

Setiap request punya assertion untuk **status code**, **response body**, dan **response time** (batas diatur lewat variable `max_response_time`, default 5000 ms).

### Alur autentikasi
1. `Login - Valid Credentials` mengirim `{{email}}` dan `{{password}}` dari environment.
2. Test script menyimpan `data.token` ke environment variable `token`.
3. Auth collection diset **Bearer `{{token}}`** sehingga semua request `/api/labs` otomatis memakai token hasil login. Tidak ada token yang di-hardcode.

### Data-driven testing (`data/labs-data.csv`)

| Tipe | Skenario |
|---|---|
| valid | input normal, karakter spesial & tanda kutip |
| edge | title 1 karakter, title 255 karakter (maks), description 1000 karakter (maks), title dengan spasi di awal/akhir (di-trim API) |
| invalid | title 256 karakter, description 1001 karakter, title kosong, title hanya spasi, description kosong, field title tidak dikirim, field description tidak dikirim |

Kolom CSV: `scenario, type, title, description, expected_status, expected_message`.
Nilai khusus: `@repeat(X,N)` = huruf X diulang N kali, `@omit` = field tidak dikirim.
Data yang berhasil dibuat langsung dihapus lagi (cleanup) di test script.

## Struktur Folder

```
.
├── .github/workflows/api-test.yml   # Pipeline CI: jalan otomatis saat push / PR ke main
├── collections/
│   └── script-labs-api.postman_collection.json   # Postman collection (4 folder, 15 request)
├── environments/
│   └── script-labs.postman_environment.json      # Environment tanpa kredensial (dipakai CI)
├── data/
│   └── labs-data.csv                # Data untuk data-driven testing
├── docs/screenshots/                # Bukti hasil run di tab Actions
├── package.json                     # Dependency Newman + npm scripts
└── README.md
```

`reports/` (hasil HTML report) dan `environments/local.postman_environment.json` (berisi kredensial asli) ada di `.gitignore`.

## Cara Run Lokal

Prasyarat: Node.js 18+ dan akun di [labs.hendri.me](https://labs.hendri.me) (bisa register lewat website).

```bash
npm install
```

Buat environment lokal berisi kredensial (file ini tidak ikut di-commit):

```bash
cp environments/script-labs.postman_environment.json environments/local.postman_environment.json
# lalu isi value "email" dan "password" di file local tersebut
```

Jalankan test:

```bash
npm run test:crud   # folder Auth, Auth Negative, Labs CRUD
npm run test:ddt    # folder Auth + Data-Driven dengan data/labs-data.csv
npm test            # keduanya
```

Laporan HTML tersimpan di `reports/crud-report.html` dan `reports/ddt-report.html`.

**Lewat Postman app:** import collection dan environment, isi `email`/`password` di environment, lalu:
- Run collection folder `01`–`03` seperti biasa.
- Untuk data-driven: buka Collection Runner, pilih folder `01 - Auth` dan `04 - Data-Driven (CSV)`, lalu pilih file `data/labs-data.csv` di bagian *Data*.

## CI/CD (GitHub Actions)

Workflow `.github/workflows/api-test.yml` berjalan otomatis setiap **push** dan **pull request** ke `main` (bisa juga dijalankan manual lewat *Run workflow*). Langkahnya:

1. Install Newman (`npm ci`)
2. Cek secrets sudah di-set
3. Run suite Auth + CRUD
4. Run suite data-driven (CSV)
5. Upload HTML/JUnit report sebagai artifact `newman-reports`

Kredensial disimpan sebagai **GitHub Secrets** dan dikirim ke Newman lewat `--env-var`:

| Secret | Isi |
|---|---|
| `API_EMAIL` | email akun labs.hendri.me |
| `API_PASSWORD` | password akun tersebut |

Set di **Settings → Secrets and variables → Actions → New repository secret**.

## Skenario Gatekeeper

Bukti bahwa pipeline berfungsi sebagai gatekeeper ada di [Pull Request #1](https://github.com/pramikarega/final-assignment-api-automation/pull/1). `main` dilindungi branch protection yang mewajibkan check `Newman API Tests` lulus sebelum merge.

| Tahap | Commit | Hasil CI | Bukti |
|---|---|---|---|
| 1. Test sengaja dibuat salah: baris CSV mengharapkan title 300 karakter **diterima** (201), padahal API membatasi 255 karakter | `d58661a` | ❌ 2 assertion gagal di iterasi 14, merge diblokir | [actions-fail-gatekeeper.png](docs/screenshots/actions-fail-gatekeeper.png), [pr-blocked.png](docs/screenshots/pr-blocked.png) |
| 2. Test diperbaiki: title 300 karakter harus **ditolak** (400, "Title cannot exceed 255 characters") | `c372c77` | ✅ semua check lulus, PR bisa di-merge | [pr-fixed.png](docs/screenshots/pr-fixed.png) |

Pengaturan branch protection: **Settings → Branches → Add rule** untuk `main`, centang *Require status checks to pass before merging*, pilih `Newman API Tests`.

## Bukti Run

Screenshot ada di [`docs/screenshots/`](docs/screenshots/):

- [`actions-pass.png`](docs/screenshots/actions-pass.png): run di `main` berhasil
- [`actions-fail-gatekeeper.png`](docs/screenshots/actions-fail-gatekeeper.png): run di PR gatekeeper gagal
- [`pr-blocked.png`](docs/screenshots/pr-blocked.png): PR tertahan karena check gagal
- [`pr-fixed.png`](docs/screenshots/pr-fixed.png): setelah diperbaiki, check lulus dan PR siap di-merge
