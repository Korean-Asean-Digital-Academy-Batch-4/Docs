# Arsitektur — EduTrack

> **Status:** Disepakati 2 Agustus 2026
> **Sumber kebenaran** untuk stack, struktur folder, batas modul, dan alur data.
> Untuk aturan peran & izin, lihat [`aktor-role.md`](./aktor-role.md) — file ini tidak mengulangnya.

---

## 1. Empat prinsip

Kalau ada yang bertanya "kenapa begini?", jawabannya hampir selalu salah satu dari empat ini.

**① Identitas dan izin dipisah tegas.**
Cognito menjawab *"siapa orang ini?"*. Postgres menjawab *"boleh sentuh baris yang mana?"*. Tidak pernah tertukar.

**② Nilai adalah data yang tidak boleh rusak diam-diam.**
Penulisan nilai atomik per kelas. Rapor dibekukan, bukan dihitung ulang. AI tidak punya izin tulis ke tabel nilai — dijaga database, bukan disiplin kode.

**③ Yang lambat dipisahkan dari yang interaktif.**
Guru tidak pernah menunggu PDF. Semua yang berat masuk antrean.

**④ Tidak ada yang mengunci ke AWS.**
SDK AWS hanya boleh muncul di `adapters/aws/`. Aplikasi yang sama harus bisa jalan di server biasa — dan itu dibuktikan lewat `entry/server.ts`, bukan diklaim.

---

## 2. Stack

| Lapisan | Pilihan |
|---|---|
| Frontend | React SPA (Vite) → S3 + CloudFront · *belum final, lihat §10* |
| Backend | TypeScript + **Hono** di AWS Lambda |
| ORM | **Drizzle** |
| Database | **RDS PostgreSQL 16**, db.t4g.micro |
| Auth | **Amazon Cognito** User Pool (3 group) |
| AI | **Amazon Bedrock** |
| Antrean | **SQS** + DLQ |
| IaC | **Terraform** 100% — tanpa Ansible |
| CI/CD | **GitHub Actions + OIDC** |
| Region | **ap-southeast-1** (Singapore) |

---

## 3. Peta besar

```
                            Pengguna
                               │
                    ┌──────────▼──────────┐
                    │     CloudFront      │ ← satu domain, satu pintu
                    │  app.edutrack.xxx   │   → tidak ada CORS
                    └──┬───────────────┬──┘   → cookie HttpOnly bisa dipakai
              /api/*   │               │  /*
              ┌────────▼─────┐   ┌─────▼──────────┐
              │ API Gateway  │   │ S3  React build│ privat, OAC
              │  HTTP API    │   └────────────────┘
              └────────┬─────┘
   ┌───────────────────▼────────────────────────────────┐
   │  VPC · 2 AZ · ap-southeast-1                       │
   │                                                    │
   │  subnet privat-app                                 │
   │     λ api (Hono)  ──SQS──▶  λ worker               │
   │         │                      │                   │
   │         │ app_rw               │ app_ai            │
   │  subnet privat-data            │                   │
   │         ▼                      ▼                   │
   │      RDS PostgreSQL 16 · db.t4g.micro · terenkripsi│
   │                                                    │
   │  subnet publik:  NAT instance t4g.nano             │
   └────────────────────────────────────────────────────┘
         │                │                 │
    Cognito          Bedrock            S3 rapor
   (user pool)    (deskripsi AI)    (presigned URL)
```

---

## 4. Alur request

### Alur A — Guru simpan nilai sekelas (sinkron)

```
PATCH /api/penugasan/{id}/nilai
{ "changes": [ 30 nilai ] }
```

| # | Di mana | Apa |
|---|---|---|
| 1 | Browser | Cookie HttpOnly ikut otomatis — **same-origin**, tidak ada preflight |
| 2 | CloudFront | Path `/api/*` → origin API Gateway. Tidak di-cache (bukan GET) |
| 3 | API Gateway | Invoke Lambda `api` |
| 4 | Middleware Hono | Baca cookie → verifikasi JWT pakai **JWKS Cognito** (cache di memori container) → dapat `sub` + group |
| 5 | Middleware Hono | `SELECT id FROM users WHERE cognito_sub = ?` → user internal |
| 6 | Route handler | `EXISTS (penugasan WHERE id=? AND guru_user_id=?)` → 403 kalau tidak |
| 7 | Validasi (Zod) | Nilai 0–100, komponen benar milik penugasan itu |
| 8 | Postgres | `BEGIN` → upsert 30 baris → tulis `audit_log` → `COMMIT` |
| 9 | Response | `200 { updated: 30 }` |

**1 request · 1 koneksi · 1 transaksi.** Warm ~80–150ms.
Kalau gagal di langkah 8, **30-duanya batal** — tidak ada keadaan setengah jadi yang mustahil ditelusuri guru.

### Alur B — Wali kelas "Buat Semua Rapor" (asinkron)

```
POST /api/kelas/{id}/rapor/generate-all
```

```
λ api                                    λ worker  (concurrency 5)
  │                                          │
  ├ authz: kelas.wali_kelas_user_id = me      │
  ├ cek kelengkapan nilai semua mapel         │
  ├ buat 30 baris rapor status='draft'        │
  ├ kirim 30 pesan ──────────► SQS ──────────►│
  └ balas 202 { jobId }                       │
     (< 1 detik)                              ├ baca nilai+presensi+sikap  [app_ai: SELECT saja]
                                              ├ hitung nilai akhir = Σ(komponen × bobot)
                                              ├ Bedrock → deskripsi naratif per mapel
                                              ├ render PDF (pdfmake)
                                              ├ upload ke S3
                                              └ UPDATE rapor: pdf_s3_key, rata_rata

frontend polling GET /api/kelas/{id}/rapor → progress
gagal 3× → DLQ + alarm CloudWatch
```

**Kenapa wajib asinkron:** API Gateway **memutus koneksi di 29 detik dan tidak bisa dinaikkan**. 30 PDF mustahil selesai di situ.

Koneksi DB tetap **5**, berapa pun jumlah siswanya.

---

## 5. Lapisan per lapisan

### 5.1 Pintu masuk — CloudFront sebagai satu-satunya alamat

Frontend dan backend di **satu domain**, dibedakan path.

| Path | Origin |
|---|---|
| `/api/*` | API Gateway → Lambda `api` |
| `/*` | S3 (React build, bucket privat, Origin Access Control) |

Yang didapat:

- **CORS hilang total** — tidak ada preflight, tidak ada debugging header
- **Cookie HttpOnly jadi mungkin** (`HttpOnly; Secure; SameSite=Strict`) → token tidak pernah tersentuh JavaScript. Untuk data nilai anak di bawah umur, jauh lebih aman daripada JWT di `localStorage` yang bisa dicuri XSS
- Frontend cukup panggil `/api/...` — tidak ada base URL berbeda antar environment

> ⚠️ Sertifikat ACM untuk CloudFront **wajib dibuat di `us-east-1`**, padahal semua sumber daya lain di `ap-southeast-1`. Ditangani dengan provider alias kedua di Terraform. Ini sering bikin `terraform apply` gagal kalau terlewat.

S3 bucket frontend **tidak** memakai S3 static website hosting — memakai **OAC**, sehingga bucket tidak pernah publik.

### 5.2 Auth

Batas yang tidak boleh dilanggar (detail di [`aktor-role.md §7`](./aktor-role.md)):

| Cognito | Postgres |
|---|---|
| Kredensial, terbitkan token | Profil, relasi, izin baris |
| 3 group: `superadmin`, `guru`, `siswa` | Wali kelas mana, mengajar apa |

Jembatannya satu kolom: `users.cognito_sub`.

- Self-signup **dimatikan**. Akun hanya dibuat Superadmin lewat `AdminCreateUser`
- Password sementara → wajib ganti saat login pertama
- **Validasi token di middleware Hono, bukan JWT authorizer API Gateway.** Alasannya konkret: authorizer bawaan hanya membaca header `Authorization` dan **tidak bisa membaca cookie**. Karena API kita satu Lambda monolit, validasi di middleware lebih sederhana, gratis, dan tanpa hop tambahan
- JWKS Cognito di-cache di memori container, refresh berkala

### 5.3 Compute — dua Lambda, bukan empat puluh

| Fungsi | Pemicu | Tugas |
|---|---|---|
| `api` | API Gateway | Seluruh REST API dalam satu handler Hono |
| `worker` | SQS | Generate PDF rapor, panggil Bedrock, parse Excel import |

Satu Lambda per endpoint terdengar "serverless yang benar", tapi untuk timeline ini merugikan: 40 paket deploy, 40 IAM role, dan **tiap fungsi punya cold start sendiri** — makin banyak fungsi, makin jarang tiap satunya hangat.

Yang lebih penting: dengan `api` monolit, memindahkannya ke ECS/EC2 **hampir tanpa perubahan kode**.

### 5.4 Data — disiplin koneksi

**Masalah yang dicegah:** Lambda scale horizontal tanpa batas (default 1000 concurrent), Postgres punya plafon keras (**~112 di db.t4g.micro, ~106 tersisa untuk aplikasi**). Satu request = satu execution environment = satu koneksi; koneksi tetap dipegang container hangat selama ~5–10 menit setelah selesai.

```ts
// benar di Lambda — terlihat "salah" bagi yang terbiasa Express
const sql = postgres(url, { max: 1, idle_timeout: 20 })
```

```hcl
reserved_concurrent_executions = 40   # rem terakhir
```

Empat aturan yang menjaga plafon tidak tersentuh:

| # | Aturan | Letaknya |
|---|---|---|
| 1 | **Tulis massal** — satu penugasan = satu request, satu transaksi | Kontrak API |
| 2 | **Autosave di-debounce & digabung** — tunggu diam 2 detik, kirim semua perubahan sekaligus, jangan tumpuk kalau masih ada yang in-flight | Frontend |
| 3 | **Endpoint gabungan** (`GET /me/dashboard`) + `Cache-Control: private, max-age=15` | Kontrak API |
| 4 | **Reserved concurrency 40** — request ke-41 dapat `429` yang bisa di-retry | Terraform |

> Menolak request itu bisa dipulihkan. Database mati saat 60 orang sedang bekerja tidak.

**Jangan pernah** memakai pool `{ max: 10 }` di Lambda: 150 container × 10 = 1500 koneksi diminta, padahal 9 dari 10 tiap container tidak akan pernah terpakai.

Kalau suatu saat benar-benar mentok, jalur naiknya **RDS Proxy** — satu blok Terraform, tanpa mengubah kode aplikasi. Tidak dipakai sekarang karena ~$22/bln untuk masalah yang sudah selesai dengan Rp 0.

### 5.5 AI tidak bisa menyentuh nilai — dijamin database

```sql
-- λ api
GRANT SELECT, INSERT, UPDATE ON nilai, presensi, penilaian_sikap TO app_rw;

-- λ worker (jalur AI & generate rapor)
GRANT SELECT ON nilai, presensi TO app_ai;
GRANT INSERT, UPDATE ON rapor, rapor_mapel, ai_insight TO app_ai;
-- TIDAK ADA UPDATE pada nilai. Sama sekali.
```

Charter mensyaratkan AI *"cannot alter grade data"*. Dengan dua role DB, kalau ada bug yang mencoba, **Postgres yang menolak** — dan QA bisa membuktikannya lewat satu query, bukan dengan membaca kode.

### 5.6 Jaringan

Lambda di dalam VPC (wajib, supaya RDS tidak pernah publik) tidak punya akses internet secara default. Bedrock, Cognito, dan Secrets Manager semuanya di luar VPC.

| Opsi jalur keluar | Biaya/bln* | Keputusan |
|---|---|---|
| **NAT instance t4g.nano** (AMI fck-nat) | **~$3** | ✅ **Dipakai.** Bukan HA, tapi tepat untuk pilot satu sekolah |
| NAT Gateway | ~$32 + data | Bayar mahal untuk ketiadaan trafik |
| VPC interface endpoints | ~$22–44 | 3 endpoint × 2 AZ malah lebih mahal dari NAT Gateway |

S3 lewat **gateway endpoint (gratis)** — upload/unduh rapor tidak pernah melewati NAT.

> **Catatan Ansible:** NAT instance adalah **satu-satunya server** di sistem ini. Kalau Ansible ingin dipertahankan di portofolio dengan pekerjaan yang jujur, hardening + konfigurasi NAT instance adalah satu-satunya tempat yang masuk akal. Kalau tidak, Ansible dilepas — dan itu keputusan yang bisa dipertanggungjawabkan: *tidak ada host untuk dikelola*.

---

## 6. Struktur kode & batas modul

```
src/
├── app.ts              Hono app. Tidak tahu Lambda itu apa
├── routes/             HTTP: parsing, status code
├── domain/             Aturan bisnis murni — TANPA I/O
│   ├── nilai.ts          hitung rata-rata berbobot, validasi Σbobot=100
│   ├── rapor.ts          transisi status draft→review→distributed
│   └── presensi.ts       hitung persentase kehadiran
├── db/                 Skema Drizzle + query
├── ports/              interface: Queue, Storage, Secrets
├── adapters/
│   ├── aws/            SQS, S3, Secrets Manager, Bedrock  ← SATU-SATUNYA tempat SDK AWS
│   └── local/          pg-boss, disk, env
└── entry/
    ├── lambda.ts       handle(app) + SQS consumer
    └── server.ts       serve(app) + worker loop
```

### Aturan impor

| Lapisan | Boleh impor | **Dilarang** impor |
|---|---|---|
| `domain/` | tidak ada | db, adapters, aws-sdk, hono |
| `routes/` | domain, db, ports | `adapters/aws` langsung |
| `adapters/aws/` | ports, aws-sdk | domain, routes |
| `entry/` | app, adapters | — |

`domain/` tanpa I/O sama sekali artinya seluruh logika penilaian bisa diuji tanpa database, tanpa AWS, dalam milidetik. Untuk bagian yang salah hitungnya berarti rapor siswa salah, ini bukan kemewahan.

---

## 7. Portabilitas

> Serverless dipilih karena profil trafik sekolah sangat tidak merata (malam/akhir pekan/libur ≈ nol, lonjakan tajam saat ujian — utilisasi nyata **~14%**), bukan karena serverless selalu lebih baik. Kode ditulis agar tidak terikat padanya.

### 7.1 Aplikasi

| Lapisan | Portabel? | Kalau pindah ke server biasa |
|---|---|---|
| Route, validasi, middleware auth | ✅ 100% | Tidak disentuh |
| Logika domain | ✅ 100% | Tidak disentuh |
| Drizzle + skema + migrasi | ✅ 100% | Tidak disentuh |
| Validasi JWT Cognito | ✅ 100% | JWKS lewat HTTPS — jalan di mana saja |
| Entry point | ❌ | `handle(app)` → `serve(app)`. **~5 baris** |
| Connection pool | ❌ | `max: 1` → `max: 10`. **1 baris** |
| Antrean (SQS) | ❌ | Ganti `pg-boss` — antrean di Postgres itu sendiri, tanpa infra baru |
| Storage (S3) | ⚠️ | Tetap S3, atau MinIO / disk lokal |
| Secrets Manager | ❌ | Ganti env var |

### 7.2 Database

**RDS PostgreSQL itu PostgreSQL biasa** — bukan versi khusus AWS.

```bash
pg_dump -h edutrack.xxx.rds.amazonaws.com -Fc edutrack > backup.dump
pg_restore -h localhost -d edutrack backup.dump
```

Skema Drizzle, file migrasi, dan semua query **tidak disentuh sama sekali**. Yang berubah hanya `DATABASE_URL`.

Yang berpindah adalah tanggung jawab operasional: backup, patch keamanan, enkripsi at-rest, dan risiko perangkat keras jadi urusan pemilik server.

> Ini juga alasan tambahan menolak DynamoDB: **DynamoDB tidak punya versi on-prem.** DynamoDB Local hanya untuk testing. Memilihnya akan membuat skenario "sekolah minta dipasang di server sendiri" mustahil tanpa menulis ulang seluruh lapisan data.

### 7.3 `entry/server.ts` bukan sekadar jaring pengaman

Itu cara tim development sehari-hari:

```bash
docker compose up      # Postgres + API di localhost:3000
```

Frontend bisa kerja **tanpa akun AWS, tanpa deploy, tanpa cold start**. QA bisa test lokal. Dan setiap kali `server.ts` jalan, portabilitasnya terbukti sendiri — bukan sekadar diklaim.

---

## 8. Infrastruktur & pengiriman

```
infra/
├── bootstrap/          S3 state + DynamoDB lock (apply sekali, state lokal)
├── modules/
│   ├── network/        VPC, subnet, NAT instance, SG, S3 gateway endpoint
│   ├── data/           RDS, subnet group, Secrets Manager
│   ├── auth/           Cognito user pool, 3 group, app client
│   ├── compute/        Lambda api + worker, API GW, SQS + DLQ, IAM
│   ├── frontend/       S3 + CloudFront + OAC + ACM (provider alias us-east-1)
│   └── observability/  log group, alarm, AWS Budgets
└── envs/
    ├── dev/
    └── prod/
```

`bootstrap/` terpisah karena masalah ayam-telur: Terraform butuh bucket state, tapi bucket itu sendiri dibuat Terraform. Di-apply sekali dengan state lokal, lalu tidak disentuh lagi.

### CI/CD

**GitHub Actions + OIDC — tidak ada `AWS_ACCESS_KEY_ID` di GitHub Secrets sama sekali.** Actions menukar token OIDC-nya langsung dengan IAM role.

```
PR         → terraform plan → hasilnya dikomentari ke PR
merge main → terraform apply
           → build & deploy Lambda
           → s3 sync frontend → CloudFront invalidation
```

### Urutan yang mengurangi risiko

Infra naik **duluan** (hari ke-3–4) dengan API *stub* — hanya health-check endpoint. VPC, RDS, Cognito, dan pipeline CI terbukti hidup **sebelum** backend asli selesai. Frontend dapat base URL stabil sejak awal, dan setelah itu yang berubah hanya isi paket deploy — bukan infra.

---

## 9. Biaya

| Komponen | /bln |
|---|---|
| RDS db.t4g.micro Single-AZ | $12–15 (**$0 kalau free tier akun masih aktif**) |
| NAT instance t4g.nano | ~$3 |
| Lambda + API Gateway | < $1 |
| S3 + CloudFront | < $1 |
| Cognito | $0 (< 50k MAU) |
| Bedrock | < $2 |
| **Total** | **~$18–22**, atau ~$6 dengan free tier |

> ⚠️ Semua angka kisaran dan **belum diverifikasi** ke pricing ap-southeast-1. Wajib dicek sebelum masuk `DEPLOYMENT.md`.
> **Tindakan:** cek status free tier akun AWS tim — kalau masih aktif, RDS bisa $0 selama 12 bulan.

Aman jauh di dalam budget Rp 2–10 juta, bahkan kalau dibiarkan hidup berbulan-bulan setelah capstone sebagai portofolio.

---

## 10. Ringkasan keputusan

| Keputusan | Alasan |
|---|---|
| **ap-southeast-1**, bukan Jakarta | Ketersediaan model Bedrock; ap-southeast-3 masih terbatas. Latensi Jakarta→Singapura ~20–40ms, tidak terasa untuk CRUD |
| **Hono**, bukan NestJS | Cold start ~200–400ms vs ~1–1,5 detik; bundle jauh lebih kecil |
| **Drizzle**, bukan Prisma | Query engine Prisma binary ~40MB → bundle bengkak & cold start naik di Lambda. Drizzle TypeScript murni, nol binary |
| **Cognito**, bukan JWT sendiri | Memperkuat narasi full-native-AWS. Biaya: dua sumber data, dipersempit ke satu kolom `cognito_sub` |
| **RDS PostgreSQL**, bukan DynamoDB | Data EduTrack relasional sampai akar (rata-rata berbobot, join lintas mapel, agregat kehadiran). Satu-satunya pilihan dengan jalur on-prem yang jelas |
| **2 Lambda**, bukan per-endpoint | Cold start & kompleksitas deploy; monolit `api` gampang dipindah ke ECS/EC2 |
| **SQS** untuk generate rapor | API Gateway putus di 29 detik, tidak bisa dinaikkan |
| **Bedrock**, bukan API LLM eksternal | Auth via IAM role — nol API key untuk disimpan & dirotasi. Tidak perlu jalur keluar tambahan |
| **NAT instance**, bukan NAT Gateway | ~$3 vs ~$32/bln untuk trafik yang nyaris nol |
| **Tanpa Ansible** | Tidak ada host untuk dikelola di arsitektur serverless. Memaksakannya terbaca sebagai tool-untuk-CV, bukan keputusan teknik |
| **100% Terraform** | Satu bahasa untuk seluruh infra |
| **GitHub Actions + OIDC** | Tidak ada kredensial AWS jangka panjang di mana pun |

---

## 11. Yang belum diputuskan

| # | Item | Menunggu | Dampak kalau berubah |
|---|---|---|---|
| 1 | Frontend SPA (Vite) atau Next.js SSR | Keputusan Frontend | Kalau SSR, hanya origin `/*` yang pindah dari S3 ke Lambda/Amplify. Sisa arsitektur tetap |
| 2 | Angka di kartu dashboard siswa | Klarifikasi UI/UX | Wireframe tidak konsisten: `92/100` di dashboard vs `Rata-rata 89.8` di Rincian Nilai |
| 3 | Ambang batas badge kehadiran | Aturan sekolah | `85%` dapat badge "Perlu Ditingkatkan" — batasnya belum ditulis di mana pun |
| 4 | Scope "Rekomendasi Belajar AI" | Keputusan PM | **Wireframe menjanjikan lebih dari yang charter setujui** (charter mencoret AI Learning Coach) |
| 5 | Team teaching? | Pihak sekolah | **Paling mendesak** — menentukan constraint `UNIQUE (mapel, kelas, periode)` di `penugasan`. Mengubahnya setelah ada data itu repot |
| 6 | Wali kelas boleh perbaiki nilai mapel lain? | Pihak sekolah | Mengubah matriks izin di `aktor-role.md` |

---

## Berkas terkait

- [`aktor-role.md`](./aktor-role.md) — peran, matriks izin, batas Cognito ↔ Postgres
- `SCHEMA.md` — definisi tabel lengkap *(belum ditulis)*
- `API.md` — kontrak endpoint & bentuk response *(belum ditulis)*
- `DEPLOYMENT.md` — Terraform, environment, rollback, verifikasi biaya *(belum ditulis)*
- `PRD.md` — scope hasil koreksi, known limitations *(belum ditulis)*
- `RULES.md` — konvensi coding & batasan agent *(belum ditulis)*
