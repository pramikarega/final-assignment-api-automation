# Script Labs API Test Automation

API test automation untuk [labs.hendri.me](https://labs.hendri.me) (base API: `https://api-script-labs.hendri.me`) memakai **Postman + Newman**, dengan **data-driven testing (CSV)** dan **CI/CD GitHub Actions**.

![API Test Automation](https://github.com/pramikarega/final-assignment-api-automation/actions/workflows/api-test.yml/badge.svg)

## Cakupan Test

| Folder | Request | Isi |
|---|---|---|
| `01 - Auth` | 1 | `POST /api/auth/login`, token otomatis disimpan ke variable `token` |
| `02 - Auth Negative` | 4 | Password salah (401), format email salah (400), tanpa token (401), token invalid (401) |
| `03 - Labs CRUD` | 9 | Create, Get All, Get by ID, Update, cek hasil update, Update ID tidak ada (404), Delete, Get setelah delete (404), Delete ulang (404) |
| `04 - Data-Driven (CSV)` | 1 × 13 baris | `POST /api/labs` dengan data dari `data/labs-data.csv` |

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

Branch `demo/gatekeeper-fail` sengaja menambahkan satu baris di CSV yang mengharapkan API **menerima** title 300 karakter (`expected_status` 201), padahal aturan API maksimal 255 karakter sehingga API membalas 400. Ini mensimulasikan perubahan yang tidak sesuai spesifikasi.

Saat branch tersebut dibuka sebagai Pull Request ke `main`, workflow gagal (❌) dan PR tidak bisa di-merge karena branch protection mewajibkan check `Newman API Tests` lulus. Sementara itu, run di `main` tetap hijau (✅).

Aktifkan branch protection di **Settings → Branches → Add rule** untuk `main`: centang *Require status checks to pass before merging* lalu pilih `Newman API Tests`.

## Bukti Run

Screenshot hasil run ada di [`docs/screenshots/`](docs/screenshots/):

- `actions-pass.png`: run di `main` berhasil
- `actions-fail-gatekeeper.png`: run di PR gatekeeper gagal
- `pr-blocked.png`: PR tertahan karena check gagal
