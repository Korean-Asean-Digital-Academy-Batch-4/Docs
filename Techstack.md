# Techstack — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v1.0 |
| **Tanggal** | 6 Agustus 2026 |
| **Disusun oleh** | Re:Code |
| **Sumber kebenaran** | [PRD.md](PRD.md) v3.0, [ATURAN-DAN-KRITERIA.md](ATURAN-DAN-KRITERIA.md) v1.0, dan [RFC-001](RFC-001-model-data-konseptual.md) — terutama §8 Kendala bagi `ARCHITECTURE.md` |
| **Kedudukan** | Menetapkan pilihan teknologi dan bentuk penerapannya. Menggantikan bagian stack pada [ARCHITECTURE.md](ARCHITECTURE.md) versi 2 Agustus 2026, yang disusun sebelum PRD v3.0 dan RFC-001 |
| **Dokumen lanjutan** | `SCHEMA.md` (skema fisik), `API.md` (kontrak endpoint), `DEPLOYMENT.md` (Terraform, environment, rollback) |

> Dokumen ini menjawab **dengan apa** sistem dibangun dan **di mana** ia dijalankan.
> Entitas dan invarian basis data ditetapkan [RFC-001](RFC-001-model-data-konseptual.md); dokumen ini tidak mengulangnya.
> Aturan peran dan izin ditetapkan [aktor-role.md](aktor-role.md); dokumen ini hanya menetapkan cara penegakannya.
>
> Mengikuti [konvensi dokumen](README.md), badan dokumen memuat **deskripsi keadaan yang berlaku** dan disunting langsung ketika berubah, sedangkan **Lampiran Catatan Keputusan** bernomor dan bertanggal serta hanya ditambah, tidak disunting.

---

## 1. Prinsip

Empat prinsip berikut menjadi alasan hampir setiap pilihan pada dokumen ini.

**① Portabilitas dibuktikan, bukan diklaim.**
Artefak yang dijalankan di AWS adalah **image Docker yang sama persis** dengan yang dijalankan di laptop pengembang dan di server sekolah. Karena tim menjalankannya setiap hari, portabilitas terbukti sendiri tanpa pengujian khusus.

**② Yang dapat dijamin basis data tidak diserahkan kepada disiplin kode.**
Invarian RFC-001 §6 sedapat mungkin ditegakkan lewat constraint, partial index, dan pemisahan role. Aturan yang tidak dapat ditegakkan basis data dinyatakan terang-terangan pada `SCHEMA.md`, bukan diasumsikan aman.

**③ Kompleksitas hanya dibayar apabila ada kebutuhan produk yang membayarnya.**
RFC-001 C-07 mencabut kebutuhan antrean, dan volume pada C-06 kecil. Antrean pesan, worker terpisah, dan pekerjaan latar karenanya tidak dibangun.

**④ Ketergantungan pada AWS dikurung di satu tempat.**
SDK AWS hanya boleh muncul di `adapters/aws/`. Lapisan domain, rute, dan basis data tidak mengetahui keberadaan AWS.

---

## 2. Stack

| Lapisan | Pilihan | Versi |
|---|---|---|
| **Frontend** | React SPA, dibangun dengan Vite | React 19, Vite 7 |
| **Bahasa** | TypeScript, `strict` di frontend maupun backend | TypeScript 5.x |
| **Backend** | Express | Express 5, Node.js 24 LTS |
| **ORM** | Drizzle | — |
| **Basis data** | PostgreSQL di Amazon RDS | PostgreSQL 17, `db.t4g.micro` |
| **Autentikasi** | Dikelola sendiri: Argon2id + sesi lewat cookie `HttpOnly` | — |
| **AI** | OpenRouter, dipanggil lewat port adapter | — |
| **Penyajian frontend** | S3 (privat, OAC) + CloudFront | — |
| **Penyajian backend** | ECS Fargate di belakang ALB privat (CloudFront VPC Origin) | — |
| **Penyimpanan berkas** | S3, diakses lewat presigned URL | — |
| **Validasi** | Zod, di batas HTTP maupun batas berkas unggahan | — |
| **Berkas rapor** | pdfmake, dirender saat diunduh | — |
| **IaC** | Terraform | — |
| **CI/CD** | GitHub Actions + OIDC | — |
| **Region** | ap-southeast-1 (Singapura) | — |

**Yang sengaja tidak dipakai:** API Gateway, SQS, Lambda, Cognito, Bedrock, dan Ansible. Alasan masing-masing tercatat pada Lampiran Catatan Keputusan.

---

## 3. Peta besar

```
                              Pengguna
                                 │
                      ┌──────────▼───────────┐
                      │      CloudFront      │ ← satu domain, satu pintu masuk
                      │   app.edutrack.xxx   │   → tanpa CORS
                      └───┬──────────────┬───┘   → cookie HttpOnly dapat dipakai
                    /*    │              │   /api/*
              ┌───────────▼──┐           │
              │ S3  build    │           │  VPC Origin
              │ React        │           │  (private link AWS)
              │ privat, OAC  │           │
              └──────────────┘           │
   ┌─────────────────────────────────────▼──────────────────┐
   │  VPC · 2 AZ · ap-southeast-1                           │
   │                                                        │
   │  subnet privat-app                                     │
   │      ALB internal                                      │
   │          │                                             │
   │          ▼                                             │
   │      ECS Fargate · service "api" · Express             │
   │          │                    │                        │
   │          │ app_rw             │ app_ro                 │
   │  subnet privat-data           │                        │
   │          ▼                    ▼                        │
   │      RDS PostgreSQL 17 · db.t4g.micro · terenkripsi    │
   │                                                        │
   │  subnet publik:  NAT instance t4g.nano                 │
   └────────────────────────────────────────────────────────┘
                       │                    │
                  OpenRouter            S3 rapor
                (tombol Suggestion)   (presigned URL)
```

Tidak ada sumber daya yang dapat dihubungi langsung dari internet. ALB berada di subnet privat dan hanya menerima trafik dari CloudFront melalui private link AWS; bucket S3 frontend tidak pernah publik dan hanya dapat dibaca CloudFront lewat Origin Access Control.

---

## 4. Pemenuhan kendala RFC-001 §8

Tabel ini adalah pertanggungjawaban langsung dokumen ini terhadap RFC-001. Setiap kendala harus memiliki jawaban yang dapat ditunjuk.

| # | Kendala | Cara dipenuhi |
|---|---|---|
| **C-01** | Data relasional sampai akar | PostgreSQL. Rata-rata berbobot, penggabungan lintas mata pelajaran, agregat kehadiran, dan pemeriksaan kelengkapan seluruhnya dikerjakan sebagai query SQL |
| **C-02** | Penulisan atomik atas banyak baris | Satu penekanan **Simpan Nilai** menjadi satu request, satu transaksi. Kegagalan membatalkan seluruh baris. Memenuhi P22 dan AC-15 |
| **C-03** | Penghapusan berantai | `ON DELETE CASCADE` dari `sesi` ke `presensi`, memenuhi I-16 dan AC-25 |
| **C-04** | Penyimpanan berkas rapor terpisah | S3 dengan presigned URL berumur pendek. Berkas tidak pernah melewati proses aplikasi saat diunduh |
| **C-05** | Pembedaan hak baca dan tulis pada tingkat data | Dua role basis data, `app_rw` dan `app_ro`. Jalur AI memakai koneksi `app_ro` yang tidak memiliki hak tulis sama sekali (§8) |
| **C-06** | Volume kecil, bukan pendorong pemilihan | `db.t4g.micro` Single-AZ memadai dengan margin besar. Ukuran instance tidak menjadi pertimbangan |
| **C-07** | Jalur AI sinkron, sekali jalan, hanya membaca | Panggilan HTTP biasa di dalam request handler. **Tidak ada antrean, tidak ada worker, tidak ada pekerjaan latar** |

C-07 adalah pencabutan terbesar terhadap rancangan 2 Agustus 2026. Antrean pada rancangan tersebut dibenarkan oleh AI yang menulis deskripsi naratif ke rapor secara latar; PRD v3.0 mencabut seluruh dasar itu.

---

## 5. Frontend

React SPA yang dibangun Vite menjadi berkas statis, lalu disalin ke bucket S3 privat dan disajikan CloudFront.

| Path | Origin |
|---|---|
| `/*` | S3 — hasil `vite build`, bucket privat, Origin Access Control |
| `/api/*` | ALB privat — ECS Fargate |

Karena frontend dan backend berada pada **satu domain**, tiga hal didapat sekaligus: CORS hilang seluruhnya, cookie sesi dapat memakai `HttpOnly` sehingga token tidak pernah tersentuh JavaScript, dan frontend cukup memanggil `/api/...` tanpa base URL berbeda antar lingkungan.

**Kebutuhan yang membentuk pilihan pustaka:**

| Kebutuhan | Dasar | Pilihan |
|---|---|---|
| Pemberitahuan berhasil atau gagal pada setiap perubahan data | P21, AC-27 | Satu komponen notifikasi global yang dipakai seluruh mutasi. Kegagalan menampilkan alasannya |
| Penyimpanan hanya setelah tombol ditekan | P22, AC-15 | Keadaan formulir bersifat lokal sampai tombol simpan ditekan. **Tidak ada penyimpanan otomatis** |
| Matriks siswa terhadap komponen pada layar nilai | RFC-001 §5.1 | Pemutaran bentuk dari baris menjadi matriks dilakukan di frontend. Tiga puluh siswa dikali delapan komponen berarti 240 sel |
| Terbaca pada perangkat bergerak | NG5 | Tata letak responsif. **Bukan** aplikasi Android maupun iOS |

Menempatkan frontend di container tidak dilakukan: hasil build adalah berkas statis, sehingga membayar compute untuk menyajikannya tidak memberi manfaat apa pun.

---

## 6. Backend

### 6.1 Struktur kode dan batas modul

```
src/
├── app.ts              Express app. Tidak mengetahui ECS maupun AWS
├── routes/             HTTP: parsing, kode status, bentuk respons
├── domain/             Aturan bisnis murni — TANPA I/O
│   ├── nilai.ts          rata-rata berbobot, kelengkapan komponen
│   ├── rapor.ts          transisi draft → finalized → distributed
│   └── presensi.ts       persentase kehadiran
├── db/                 Skema Drizzle, migrasi, dan query
├── ports/              interface: Storage, AiAdvisor, Secrets
├── adapters/
│   ├── aws/            S3, Secrets Manager   ← SATU-SATUNYA tempat SDK AWS
│   ├── openrouter/     implementasi AiAdvisor
│   └── local/          disk, env             ← dipakai on-prem dan pengembangan
└── entry/
    └── server.ts       satu-satunya entry point
```

### 6.2 Aturan impor

| Lapisan | Boleh mengimpor | Dilarang mengimpor |
|---|---|---|
| `domain/` | tidak ada | `db`, `adapters`, SDK AWS, `express` |
| `routes/` | `domain`, `db`, `ports` | `adapters/aws` secara langsung |
| `adapters/*` | `ports`, SDK yang bersangkutan | `domain`, `routes` |
| `entry/` | `app`, `adapters` | — |

`domain/` tanpa I/O berarti seluruh logika penilaian dapat diuji tanpa basis data dan tanpa AWS, dalam hitungan milidetik. Untuk bagian yang salah hitungnya berarti rapor siswa salah, ini bukan kemewahan.

Berbeda dari rancangan 2 Agustus 2026, **`entry/` hanya memuat satu berkas**. Tidak ada entry point terpisah untuk AWS, karena AWS menjalankan container yang sama dengan on-prem.

### 6.3 Penerapan di ECS Fargate

| Parameter | Nilai | Alasan |
|---|---|---|
| Ukuran task | 0,25 vCPU · 0,5 GB | Volume C-06 kecil; dapat dinaikkan tanpa mengubah kode |
| Jumlah task | 2, tersebar di 2 AZ | Ketersediaan saat penggantian versi dan saat satu AZ terganggu |
| Connection pool | `max: 10` per task | Server long-running dengan pool biasa. Dua task berarti paling banyak 20 koneksi dari plafon ~106 pada `db.t4g.micro` |
| Health check | `GET /healthz` — memeriksa proses dan koneksi basis data | ALB menarik task yang tidak sehat |
| Penggantian versi | Rolling update, `minimumHealthyPercent = 100` | Tidak ada waktu mati saat deploy |
| Batas waktu request | ALB idle timeout 60 detik | Tidak ada batas keras 29 detik sebagaimana API Gateway |

### 6.4 Pembatasan laju

Tidak ada API Gateway, sehingga pembatasan laju berada di dalam Express (`express-rate-limit`) dan disandarkan pada **identitas pengguna**, bukan alamat IP — satu sekolah kerap berbagi satu alamat IP publik.

| Jalur | Batas | Alasan |
|---|---|---|
| `POST /api/auth/login` | Per akun dan per alamat IP | Menahan percobaan kata sandi beruntun. Relevan karena §6.1.3 meniadakan syarat kerumitan kata sandi |
| Tombol Suggestion | Per siswa | Setiap penekanan memanggil OpenRouter dan berbiaya token (§9) |
| Unggah berkas | Per pengguna, disertai batas ukuran | Menahan pemakaian memori yang tidak wajar |

Pembatasan ini berada di dalam aplikasi, sehingga **ikut berpindah ke on-prem** — berbeda dari pembatasan laju di tepi jaringan yang akan tertinggal di AWS.

---

## 7. Basis data

### 7.1 Mesin

PostgreSQL 17 di Amazon RDS, `db.t4g.micro`, Single-AZ, terenkripsi at-rest, dengan pencadangan otomatis 7 hari.

Empat kemampuan PostgreSQL menjadi penentu pemilihan, dan seluruhnya berasal dari invarian RFC-001 §6:

| Kemampuan | Invarian yang ditegakkan |
|---|---|
| **Partial unique index** — `UNIQUE (...) WHERE aktif` | I-03 satu semester aktif per tahun ajaran; I-09 satu wali kelas per kelas |
| **Composite foreign key** — `(mapel_ref, tingkat) → mapel(id, tingkat)` | I-06 jenjang kelas wajib sama dengan jenjang mata pelajaran, sehingga ketidakcocokan **mustahil tersimpan** dan bukan sekadar dicegah pemeriksaan aplikasi (AC-24) |
| **JSONB** | `rapor_mapel.snapshot_komponen` dan `audit_log.sebelum`/`sesudah` |
| **DDL transaksional** | Migrasi yang gagal batal seluruhnya, tidak meninggalkan skema separuh jadi pada basis data berisi data sekolah sungguhan |

### 7.2 Dua role, bukan satu

```sql
-- role aplikasi: seluruh jalur tulis
GRANT SELECT, INSERT, UPDATE, DELETE ON nilai, presensi, sesi, rapor, rapor_mapel TO app_rw;

-- role jalur AI: hanya membaca
GRANT SELECT ON nilai, presensi, sesi, mapel, penugasan, penugasan_komponen TO app_ro;
-- TIDAK ADA INSERT, UPDATE, maupun DELETE. Sama sekali.
```

I-23 dan AC-20 mensyaratkan AI tidak pernah menulis ke data akademik. Dengan dua role terpisah, seandainya ada kekeliruan kode yang mencoba, **PostgreSQL yang menolak** — dan penguji dapat membuktikannya lewat satu query, bukan dengan membaca kode.

Kedua kredensial disimpan di Secrets Manager dan dirotasi tanpa mengubah kode aplikasi.

### 7.3 ORM

Drizzle dipilih karena skema, migrasi, dan query ditulis dekat dengan SQL. Partial index, composite foreign key, dan `ON DELETE CASCADE` pada §7.1 dan §7.2 dapat dinyatakan langsung tanpa jalan memutar, dan migrasi berupa berkas SQL yang dapat dibaca serta ditinjau.

---

## 8. Autentikasi dan sesi

Kredensial dikelola sendiri di dalam PostgreSQL. Tidak ada layanan identitas terkelola.

| Aspek | Penerapan | Dasar |
|---|---|---|
| Pengenal masuk | `pengguna.nama_pengguna` — NIP bagi Guru, NIS bagi Siswa, nama pengguna tersendiri bagi Administrator | P20, RFC-001 §7.1 |
| Penyimpanan kata sandi | Argon2id | — |
| Kata sandi awal | Dihasilkan sistem, ditampilkan sekali kepada Administrator untuk diserahkan kepada pengguna | §6.1.3, P17 |
| Syarat kerumitan | **Tidak ada** | §6.1.3 |
| Kewajiban ganti saat masuk pertama | **Tidak ada** | §6.1.3 |
| Pemulihan mandiri | **Tidak ada.** Tombol Lupa kata sandi hanya menampilkan pesan menghubungi Wali Kelas atau Administrator | §6.1.3, AC-33 |
| Sesi | Token sesi acak di dalam cookie `HttpOnly; Secure; SameSite=Strict` | — |
| Perolehan peran | `pengguna.peran` dibaca dari basis data pada setiap request; kewenangan baris diperiksa lewat `penugasan` dan `kelas.wali_kelas_ref` | aktor-role.md §6 |

Seluruh ketentuan §6.1.3 di atas merupakan **perilaku bawaan** dari pendekatan ini, bukan hasil mematikan fitur pada layanan pihak lain. Inilah alasan utama pemilihannya (CK-05).

**Akun Administrator pertama** dibuat lewat perintah seed yang dijalankan sekali setelah migrasi, dengan kata sandi awal diambil dari Secrets Manager. Hal ini menutup temuan **T-03** pada RFC-001 §10, yang mencatat bahwa PRD tidak mengatur cara akun Administrator dibuat.

---

## 9. AI Insight

Tombol Suggestion (PRD §8.5) dilayani melalui satu endpoint sinkron yang hanya membaca.

```
POST /api/me/suggestion
  → baca data siswa penekan tombol lewat koneksi app_ro
  → susun prompt
  → panggil OpenRouter
  → kembalikan teks ke frontend
  → tidak menulis apa pun
```

| Aspek | Penerapan | Dasar |
|---|---|---|
| Pemicu | Hanya penekanan tombol oleh siswa. Tidak berjalan otomatis | §8.5, AC-16 |
| Data yang dibaca | Hanya milik siswa yang menekan tombol, dibatasi ganda: klausa `WHERE` pada siswa yang bersangkutan, dan koneksi `app_ro` yang tidak dapat menulis | I-23, AC-17, AC-20 |
| Penyimpanan keluaran | **Tidak ada tabel, tidak ada cache, tidak ada log isi keluaran** | I-24, NG14, AC-16 |
| Kegagalan layanan | Ditangani sebagai kegagalan lunak: halaman menampilkan pesan bahwa rekomendasi tidak dapat dibuat, sedangkan nilai, presensi, finalisasi, dan distribusi tetap berjalan | AC-21, §8.6 butir 7 |
| Batas waktu | 20 detik, lalu dibatalkan | Menahan permintaan menggantung |
| Fakta sumber dan penanda Data Sementara | Ditampilkan **oleh halaman** di sekitar keluaran AI, tidak dituntut menjadi bagian teks AI | §8.5, AC-19 |

Pemanggilan melewati interface `AiAdvisor` di `ports/`, dan OpenRouter hanya salah satu implementasinya. Penggantian penyedia — termasuk ke model yang dipasang sendiri di lingkungan on-prem — berarti menukar satu adapter tanpa menyentuh lapisan domain maupun rute.

Kunci API OpenRouter disimpan di Secrets Manager dan tidak pernah masuk ke repositori maupun ke image.

---

## 10. Berkas rapor

Berkas rapor **dirender saat diunduh**, bukan dibuat massal pada saat finalisasi.

| Tahap | Yang terjadi |
|---|---|
| **Finalisasi** oleh Wali Kelas | Satu transaksi: memeriksa kelengkapan seluruh mata pelajaran (I-20, AC-07), menulis baris `rapor_mapel` beserta `snapshot_komponen` sebagai salinan beku, lalu mengubah status menjadi `finalized`. **Tidak ada berkas yang dibuat pada tahap ini** |
| **Unduh** oleh Wali Kelas atau Siswa | PDF dirender dari salinan beku tersebut dengan pdfmake, diunggah ke S3, lalu dikembalikan sebagai presigned URL berumur pendek. Berkas yang sudah ada dipakai ulang |

Karena PDF dirender dari salinan beku, keluarannya selalu sama dengan data yang difinalisasi, sehingga AC-13 terpenuhi meskipun templat bobot berubah kemudian.

Pendekatan ini menghapus seluruh kebutuhan pekerjaan latar. Finalisasi satu kelas menjadi satu transaksi basis data yang selesai dalam hitungan ratusan milidetik, bukan tiga puluh pekerjaan render yang perlu dipantau, diulang, dan dilaporkan progresnya.

Konsekuensi yang diterima: unduhan pertama satu rapor memerlukan waktu render, ditaksir di bawah dua detik untuk dokumen satu sampai dua halaman.

---

## 11. Jaringan dan keamanan

| Lapisan | Ketentuan |
|---|---|
| **Pintu masuk** | CloudFront adalah satu-satunya alamat yang dapat dihubungi publik |
| **ALB** | Subnet privat, hanya menerima trafik CloudFront lewat VPC Origin |
| **ECS task** | Subnet privat, tanpa alamat IP publik |
| **RDS** | Subnet privat-data, security group hanya mengizinkan ECS task |
| **S3 frontend** | Bucket privat, hanya dapat dibaca CloudFront lewat Origin Access Control |
| **S3 rapor** | Bucket privat, akses hanya lewat presigned URL berumur pendek |
| **Jalur keluar** | NAT instance `t4g.nano` untuk OpenRouter. S3 lewat gateway endpoint yang tidak berbiaya, sehingga unggah dan unduh rapor tidak melewati NAT |
| **Header keamanan** | HSTS, `X-Content-Type-Options`, `Referrer-Policy`, dan Content Security Policy diatur di CloudFront Response Headers Policy |
| **Rahasia** | Secrets Manager. Tidak ada kredensial di dalam image maupun repositori |

> ⚠️ Sertifikat ACM untuk CloudFront **wajib diterbitkan di `us-east-1`**, sedangkan seluruh sumber daya lain berada di `ap-southeast-1`. Ditangani dengan provider alias kedua pada Terraform. Kelalaian pada butir ini menggagalkan `terraform apply`.

---

## 12. Infrastruktur dan pengiriman

```
infra/
├── bootstrap/          S3 state + tabel kunci (dijalankan sekali, state lokal)
├── modules/
│   ├── network/        VPC, subnet, NAT instance, security group, S3 gateway endpoint
│   ├── data/           RDS, subnet group, Secrets Manager
│   ├── compute/        ECR, ECS cluster, task definition, service, ALB, IAM
│   ├── frontend/       S3 + CloudFront + OAC + VPC Origin + ACM (provider alias us-east-1)
│   └── observability/  log group, alarm, AWS Budgets
└── envs/
    ├── dev/
    └── prod/
```

`bootstrap/` terpisah karena masalah ayam dan telur: Terraform memerlukan bucket state, sedangkan bucket tersebut dibuat Terraform. Dijalankan sekali dengan state lokal, lalu tidak disentuh lagi.

**CI/CD — GitHub Actions dengan OIDC.** Tidak ada `AWS_ACCESS_KEY_ID` di GitHub Secrets; Actions menukar token OIDC-nya langsung dengan IAM role.

```
pull request  → terraform plan → hasilnya dikomentari ke pull request
                lint, typecheck, uji unit dan integrasi

merge ke main → terraform apply
              → build image → push ke ECR → ECS rolling update
              → migrasi basis data dijalankan sebagai ECS task tersendiri sebelum service diperbarui
              → s3 sync frontend → invalidasi CloudFront
```

Migrasi dijalankan sebagai task terpisah, bukan pada saat proses aplikasi menyala, agar dua task yang naik bersamaan tidak menjalankan migrasi yang sama secara serentak.

**Urutan yang mengurangi risiko:** infrastruktur dinaikkan lebih dahulu dengan API yang hanya memuat `GET /healthz`. VPC, RDS, ALB, CloudFront, dan pipeline terbukti hidup **sebelum** backend sesungguhnya selesai. Setelah itu yang berubah hanya isi image, bukan infrastruktur.

---

## 13. Lingkungan pengembangan

```bash
docker compose up      # PostgreSQL + Express di localhost:3000
```

Satu perintah menyalakan basis data dan API. Frontend berjalan dengan `vite dev` dan meneruskan `/api` ke `localhost:3000`.

Tidak diperlukan akun AWS untuk mengembangkan maupun menguji. Adapter yang dipakai adalah `adapters/local` untuk penyimpanan berkas dan variabel lingkungan untuk rahasia, sedangkan `adapters/openrouter` tetap dipakai apabila kunci API tersedia.

**Pengujian:**

| Tingkat | Alat | Yang diuji |
|---|---|---|
| Unit | Vitest | `domain/` — rata-rata berbobot, kelengkapan komponen, persentase kehadiran, transisi status rapor. Tanpa basis data |
| Integrasi | Vitest + Supertest + PostgreSQL kontainer | Rute, kewenangan per peran, keatomikan transaksi, dan penegakan invarian oleh basis data |
| Penegakan basis data | Query langsung | Bahwa `app_ro` benar-benar ditolak ketika mencoba menulis (AC-20) |

---

## 14. Portabilitas ke on-prem

| Lapisan | Portabel? | Yang berubah ketika dipasang di server sekolah |
|---|---|---|
| Image aplikasi | ✅ | **Tidak ada.** Image yang sama dijalankan `docker run` |
| Rute, validasi, middleware, autentikasi | ✅ | Tidak disentuh |
| Logika domain | ✅ | Tidak disentuh |
| Drizzle, skema, dan migrasi | ✅ | Tidak disentuh |
| Pembatasan laju | ✅ | Tidak disentuh — berada di dalam aplikasi |
| Basis data | ✅ | RDS PostgreSQL adalah PostgreSQL biasa. `pg_dump` lalu `pg_restore`; yang berubah hanya `DATABASE_URL` |
| Frontend | ✅ | Berkas statis yang sama disajikan Nginx, atau langsung oleh Express |
| Penyimpanan berkas | ⚠️ | Tetap S3, atau ditukar ke MinIO maupun disk lokal lewat `adapters/local` |
| Rahasia | ⚠️ | Secrets Manager ditukar variabel lingkungan |
| Penyedia AI | ⚠️ | OpenRouter tetap dipakai, atau ditukar adapter lain |
| CloudFront, ALB, dan NAT | ❌ | Digantikan reverse proxy tunggal, misalnya Nginx atau Caddy |

Yang berpindah bersama sistem adalah tanggung jawab operasional: pencadangan, pembaruan keamanan, enkripsi at-rest, dan risiko perangkat keras menjadi urusan pemilik server.

Tidak ada lapisan abstraksi yang dibangun khusus demi portabilitas. Portabilitas berasal dari bentuk artefaknya — sebuah container — dan dari kenyataan bahwa tim menjalankan container yang sama setiap hari.

---

## 15. Perkiraan biaya

| Komponen | Per bulan |
|---|---|
| ECS Fargate — 2 task, 0,25 vCPU dan 0,5 GB | ~$14 |
| Application Load Balancer | ~$16 |
| RDS `db.t4g.micro` Single-AZ | $12–15, atau **$0** apabila free tier akun masih berlaku |
| NAT instance `t4g.nano` | ~$3 |
| S3 dan CloudFront | < $1 |
| ECR | < $1 |
| OpenRouter | < $5, bergantung pemakaian tombol Suggestion |
| **Total** | **~$50–55**, atau ~$38 dengan free tier |

> ⚠️ Seluruh angka merupakan kisaran dan **belum diverifikasi** terhadap daftar harga `ap-southeast-1`. Wajib diperiksa sebelum masuk `DEPLOYMENT.md`.

Selisih dengan rancangan berbasis Lambda (~$18–22) sebagian besar berasal dari ALB dan Fargate. Ini adalah harga yang dibayar untuk menghapus batas 29 detik, cold start, disiplin `max: 1`, seluruh jalur antrean, dan lapisan abstraksi portabilitas. Apabila biaya ALB kemudian terasa memberatkan, **ECS Express Mode** menyatukan beberapa layanan di balik satu ALB dan dapat ditinjau tanpa mengubah kode aplikasi.

---

## 16. Yang belum diputuskan

| # | Item | Menunggu | Dampak apabila berubah |
|---|---|---|---|
| 1 | Model OpenRouter yang dipakai beserta prompt sistemnya | Uji keluaran terhadap AC-18 dan AC-31 | Hanya isi adapter. Tidak menyentuh arsitektur |
| 2 | Format rapor resmi sekolah | V5 pada ATURAN-DAN-KRITERIA §5 | Menentukan templat pdfmake. Apabila tata letaknya rumit, perlu ditinjau ulang terhadap CK-09 |
| 3 | Kebijakan penyimpanan dan pencadangan data | V6 | Menentukan lama retensi cadangan RDS dan aturan daur hidup bucket rapor |
| 4 | Nama domain dan penerbitan sertifikat | Pihak sekolah | Menentukan modul `frontend` pada Terraform |
| 5 | Apakah `dev` memerlukan RDS tersendiri atau cukup PostgreSQL lokal | Keputusan tim | Menentukan biaya lingkungan `dev` |

Temuan RFC-001 §10 yang masih terbuka — T-01, T-02, T-04, T-05, dan T-06 — bersifat produk dan tidak dipengaruhi pilihan teknologi mana pun pada dokumen ini. T-03 ditutup oleh §8.

---

## Lampiran — Catatan Keputusan

Bernomor dan bertanggal. Entri tidak disunting; perubahan keputusan ditulis sebagai entri baru yang menyebut nomor yang digantikannya.

### CK-01 · 6 Agustus 2026 · ECS Fargate untuk backend

**Diputuskan.** Express dijalankan sebagai container di ECS Fargate di belakang Application Load Balancer.

**Alasan.** Express adalah server long-running, sedangkan Lambda adalah fungsi. Menjalankan Express di Lambda memerlukan adapter, disiplin `max: 1` pada connection pool, dan pengelolaan cold start. Yang lebih menentukan: batas **29 detik** API Gateway tidak dapat dinaikkan, sehingga pekerjaan panjang memaksa antrean — padahal RFC-001 C-07 baru saja mencabut satu-satunya pembenaran antrean yang tersisa. Fargate juga menjadikan **image Docker** sebagai artefak, yaitu bentuk yang sama persis dengan yang akan dijalankan di server sekolah.

**Alternatif yang ditolak.**

*Lambda dengan `serverless-http`.* Biaya di bawah $1 per bulan, jauh lebih murah. Ditolak karena menuntut SQS, worker, dan DLQ hanya untuk mengakali batas platform, serta memerlukan entry point terpisah dan penukaran adapter antrean yang harus ditulis dan diuji terus-menerus demi portabilitas.

*AWS App Runner.* **Tidak tersedia.** App Runner dipindahkan ke mode pemeliharaan pada 31 Maret 2026 dan tidak menerima pelanggan baru sejak 30 April 2026. Akun baru tidak dapat membuat layanan App Runner. Jalur pengganti resmi yang ditunjuk AWS adalah ECS Express Mode.

*ECS Express Mode.* Menghemat biaya ALB dengan menyatukan sampai 25 layanan di balik satu ALB. Ditolak untuk saat ini karena mengabstraksi ALB, subnet, dan penskalaan di balik satu sumber daya berkendali terbatas, sedangkan EduTrack memerlukan koneksi ke RDS di subnet privat. Layak ditinjau ulang apabila biaya ALB memberatkan (§15).

*Elastic Beanstalk.* Menyediakan ALB, penskalaan, dan rolling deploy dalam satu paket dengan biaya setara. Ditolak karena yang di-deploy adalah bundel kode, bukan image, sehingga konfigurasi `.ebextensions` dan `Procfile` tidak ikut berpindah ke on-prem. Beanstalk juga menyisakan EC2 yang harus dipatch dan platform branch yang memiliki tanggal pensiun berkala. Ringkasnya: Beanstalk mempermudah deploy ke AWS, sedangkan Fargate mempermudah pindah dari AWS — dan syarat kedualah yang ditetapkan.

**Konsekuensi yang diterima.** Biaya naik dari ~$18–22 menjadi ~$50–55 per bulan.

### CK-02 · 6 Agustus 2026 · Tanpa API Gateway

**Diputuskan.** ALB menjadi satu-satunya penerus trafik ke aplikasi.

**Alasan.** API Gateway pada rancangan sebelumnya bukan pilihan gaya melainkan keharusan, karena Lambda memerlukan pemicu yang mengubah request HTTP menjadi event. Express tidak memerlukannya. Menambahkannya di depan ALB berarti satu hop tanpa manfaat, sekaligus mengembalikan batas 29 detik yang justru menjadi alasan meninggalkan Lambda.

**Yang hilang beserta penggantinya.** Pembatasan laju dan validasi request berpindah ke dalam Express (§6.4), sehingga keduanya ikut berpindah ke on-prem alih-alih tertinggal di AWS.

### CK-03 · 6 Agustus 2026 · Frontend statis di S3 dan CloudFront

**Diputuskan.** React dibangun menjadi berkas statis di bucket S3 privat, disajikan CloudFront pada domain yang sama dengan API.

**Alasan.** Hasil build tidak memerlukan compute. Satu domain menghilangkan CORS dan memungkinkan cookie `HttpOnly`, yang untuk data nilai anak di bawah umur jauh lebih aman daripada menyimpan token di `localStorage` yang dapat dicuri lewat XSS.

**Alternatif yang ditolak.** *React di dalam container ECS.* Membayar compute untuk pekerjaan yang tidak memerlukan compute. *Next.js dengan SSR.* Menambahkan runtime server untuk lapisan yang seluruh datanya bersifat privat per pengguna dan tidak dapat di-cache; SEO tidak relevan bagi aplikasi internal sekolah.

### CK-04 · 6 Agustus 2026 · PostgreSQL

**Diputuskan.** PostgreSQL 17 di Amazon RDS.

**Alasan.** Empat kemampuan pada §7.1 menegakkan invarian RFC-001 langsung di basis data. Yang paling menentukan adalah **partial unique index**, yang tanpanya I-03 dan I-09 harus dijaga disiplin kode, serta **DDL transaksional**, yang membuat migrasi gagal batal seluruhnya pada basis data berisi data sekolah sungguhan.

**Alternatif yang ditolak.** *MySQL 8.* Tidak memiliki partial index dan DDL-nya tidak transaksional. *DynamoDB.* Data EduTrack relasional sampai akar (C-01), dan DynamoDB tidak memiliki versi on-prem, sehingga skenario pemasangan di server sekolah menjadi mustahil tanpa menulis ulang seluruh lapisan data.

### CK-05 · 6 Agustus 2026 · Kredensial dikelola sendiri, bukan Cognito

**Diputuskan.** Kata sandi disimpan sebagai hash Argon2id pada tabel `pengguna`; sesi memakai token acak di dalam cookie `HttpOnly`. Menutup K-01 pada RFC-001 §9.

**Alasan.** PRD §6.1.3 menetapkan tanpa syarat kerumitan, tanpa kewajiban penggantian, dan tanpa pemulihan mandiri. Pada pendekatan ini ketiganya adalah **perilaku bawaan**; pada layanan identitas terkelola ketiganya adalah fitur yang harus dimatikan satu per satu. Identitas juga tetap berada pada satu sumber data, sehingga tidak ada kolom penghubung yang dapat menyimpang.

**Alternatif yang ditolak.** *Amazon Cognito.* Memperkuat narasi native AWS, tetapi menghasilkan dua sumber data yang harus dijembatani, memaksa seluruh ketentuan §6.1.3 dijalankan sebagai penonaktifan fitur, dan memerlukan penggantian penyedia identitas ketika dipasang di server sekolah.

**Konsekuensi yang diterima.** Penyimpanan kata sandi menjadi tanggung jawab tim. Dikurangi dengan Argon2id, pembatasan laju pada endpoint masuk (§6.4), dan tidak adanya jalur pemulihan mandiri yang dapat disalahgunakan.

### CK-06 · 6 Agustus 2026 · OpenRouter, bukan Bedrock

**Diputuskan.** Tombol Suggestion dilayani OpenRouter melalui interface `AiAdvisor`.

**Alasan.** Penyedia yang sama dipakai di AWS maupun on-prem, sehingga perilaku keluaran tidak berbeda antar lingkungan. Penggantian model dapat dilakukan tanpa berpindah layanan.

**Alternatif yang ditolak.** *Amazon Bedrock.* Autentikasi lewat IAM role sehingga tidak ada kunci API untuk dirotasi. Ditolak karena mengikat jalur AI ke AWS, sehingga pemasangan on-prem memerlukan penyedia lain dan keluarannya belum tentu setara.

**Konsekuensi yang diterima.** Kunci API perlu disimpan di Secrets Manager dan dirotasi. Jalur keluar internet menjadi kebutuhan tetap, sehingga NAT instance tidak dapat dihapus.

### CK-07 · 6 Agustus 2026 · Tanpa antrean dan tanpa worker

**Diputuskan.** Tidak ada SQS, tidak ada proses worker, dan tidak ada pekerjaan latar.

**Alasan.** RFC-001 C-07 menetapkan jalur AI bersifat sinkron, sekali jalan, dan hanya membaca. Pembuatan berkas rapor dipindahkan ke saat unduh (CK-09), sehingga tidak ada lagi pekerjaan panjang yang tersisa. Antrean pada rancangan 2 Agustus 2026 dibenarkan oleh AI yang menulis deskripsi naratif ke rapor secara latar; PRD v3.0 mencabut seluruh dasar tersebut.

**Konsekuensi yang diterima.** Apabila kelak muncul kebutuhan pekerjaan latar yang sungguhan, jalur naiknya adalah `pg-boss` — antrean di dalam PostgreSQL itu sendiri, tanpa infrastruktur baru dan tanpa kehilangan portabilitas.

### CK-08 · 6 Agustus 2026 · Dua role basis data

**Diputuskan.** `app_rw` untuk jalur tulis aplikasi, `app_ro` untuk jalur AI.

**Alasan.** I-23 dan AC-20 mensyaratkan AI tidak pernah menulis ke data akademik. RFC-001 C-05 menuntut pembedaan tersebut ditegakkan pada tingkat data. Dengan dua role, penguji dapat membuktikannya lewat satu query alih-alih membaca kode.

### CK-09 · 6 Agustus 2026 · Rapor dirender saat diunduh

**Diputuskan.** Finalisasi hanya membekukan data ke `rapor_mapel`. Berkas PDF dirender pada saat unduhan pertama, lalu disimpan di S3 dan dipakai ulang.

**Alasan.** Karena PDF dirender dari salinan beku, hasilnya selalu sama dengan data yang difinalisasi (AC-13). Pendekatan ini menghapus kebutuhan pekerjaan latar beserta pemantauan progres, pengulangan, dan pelaporan kegagalannya.

**Alternatif yang ditolak.** *Membuat seluruh PDF sekelas pada saat finalisasi.* Menghasilkan tiga puluh pekerjaan render dalam satu tindakan, yang menuntut pemrosesan latar dan tampilan progres. Kompleksitas ini dibayar tanpa manfaat produk, karena rapor tidak selalu diunduh seluruhnya.

**Konsekuensi yang diterima.** Unduhan pertama memerlukan waktu render. pdfmake dipilih karena murni JavaScript dan menjaga ukuran image tetap kecil; apabila format rapor sekolah (V5) menuntut tata letak yang tidak dapat dicapai pdfmake, keputusan ini ditinjau ulang lewat entri baru.

### CK-10 · 6 Agustus 2026 · CloudFront VPC Origin dengan ALB privat

**Diputuskan.** ALB ditempatkan di subnet privat dan hanya dapat dihubungi CloudFront melalui VPC Origin.

**Alasan.** CloudFront menjadi satu-satunya pintu masuk sistem, dan ALB tidak pernah terekspos ke internet. Untuk sistem berisi data akademik anak di bawah umur, penguatan ini berbiaya rendah. Fitur ini tersedia umum sejak 20 November 2024.

**Alternatif yang ditolak.** *ALB internet-facing dengan header rahasia dari CloudFront.* Bergantung pada kerahasiaan sebuah header, sedangkan ALB tetap dapat dihubungi siapa pun yang mengetahui alamatnya.

### CK-11 · 6 Agustus 2026 · Drizzle

**Diputuskan.** Drizzle sebagai ORM.

**Alasan.** Skema, migrasi, dan query ditulis dekat dengan SQL, sehingga partial index, composite foreign key, dan `ON DELETE CASCADE` yang menegakkan invarian RFC-001 dapat dinyatakan langsung. Migrasi berupa berkas SQL yang dapat dibaca dan ditinjau.

**Catatan.** Alasan pemilihan Drizzle pada rancangan 2 Agustus 2026 adalah ukuran bundel dan cold start di Lambda. Alasan tersebut **gugur** bersama CK-01; keputusannya tetap, tetapi dasarnya berganti.

### CK-12 · 6 Agustus 2026 · Tanpa Ansible

**Diputuskan.** Seluruh infrastruktur dikelola Terraform.

**Alasan.** NAT instance adalah satu-satunya host di dalam sistem, dan konfigurasinya cukup ditangani user data. Tidak ada armada server untuk dikelola. Apabila kelak sistem dipasang di server sekolah, pengelolaan konfigurasi host di sana adalah pekerjaan yang berbeda dan ditetapkan tersendiri.

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 6 Agustus 2026 | Dokumen dibuat. Menetapkan stack di atas PRD v3.0 dan RFC-001. Menggantikan bagian stack pada `ARCHITECTURE.md` versi 2 Agustus 2026. Menutup K-01 dan K-02 pada RFC-001 §9 melalui CK-05, CK-01, dan CK-04, serta menutup temuan T-03 melalui §8 |
