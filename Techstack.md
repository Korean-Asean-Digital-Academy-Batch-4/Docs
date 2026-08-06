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
| **Backend** | Express, dikemas sebagai container image | Express 5, Node.js 24 LTS |
| **ORM** | Drizzle | — |
| **Basis data** | PostgreSQL di Amazon RDS | PostgreSQL 17, `db.t4g.micro` |
| **Autentikasi** | Dikelola sendiri: Argon2id + sesi lewat cookie `HttpOnly` | — |
| **AI** | Elice AI Cloud (KADA), antarmuka setara OpenAI, dipanggil lewat port adapter | `/v1/chat/completions` |
| **Penyajian frontend** | S3 (privat, OAC) + CloudFront | — |
| **Penyajian backend** | AWS Lambda dari container image, memakai **Lambda Web Adapter**, diakses lewat **Function URL** dengan CloudFront OAC | LWA 1.0.x, arm64 |
| **Penyimpanan berkas** | S3, diakses lewat presigned URL | — |
| **Validasi** | Zod, di batas HTTP maupun batas berkas unggahan | — |
| **Berkas rapor** | pdfmake, dirender saat diunduh | — |
| **IaC** | Terraform | — |
| **CI/CD** | GitHub Actions + OIDC | — |
| **Region** | ap-southeast-1 (Singapura) | — |

**Yang sengaja tidak dipakai:** API Gateway, Application Load Balancer, ECS, SQS, Cognito, Bedrock, dan Ansible. Alasan masing-masing tercatat pada Lampiran Catatan Keputusan.

**Catatan bentuk artefak.** Backend berjalan di Lambda, tetapi yang di-deploy tetap **image Docker berisi Express yang mendengarkan di sebuah port** — bukan fungsi bergaya Lambda. Aplikasi tidak mengetahui keberadaan Lambda, dan image yang sama dijalankan di laptop, di ECS Fargate, maupun di server sekolah tanpa perubahan (§6.3 dan §14).

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
              │ S3  build    │           │  OAC · SigV4
              │ React        │           │
              │ privat, OAC  │           ▼
              └──────────────┘   Lambda Function URL
                                 (auth type AWS_IAM)
   ┌─────────────────────────────────────┼──────────────────┐
   │  VPC · 2 AZ · ap-southeast-1        │                  │
   │                                     ▼                  │
   │  subnet privat-app                                     │
   │      λ api · container image                           │
   │          [ Lambda Web Adapter ] → Express :8080        │
   │          │                    │                        │
   │          │ app_rw             │ app_ro                 │
   │  subnet privat-data           │                        │
   │          ▼                    ▼                        │
   │      RDS PostgreSQL 17 · db.t4g.micro · terenkripsi    │
   │                                                        │
   │  subnet publik:  NAT instance t4g.nano                 │
   └────────────────────────────────────────────────────────┘
                       │                    │
              Elice AI Cloud          S3 rapor
                (tombol Suggestion)   (presigned URL)
```

Tidak ada sumber daya yang dapat dihubungi langsung dari internet. Function URL disetel dengan auth type `AWS_IAM` sehingga hanya menerima request bertanda tangan SigV4 dari distribusi CloudFront yang ditunjuk — URL yang bocor tidak dapat dipakai siapa pun. Bucket S3 frontend tidak pernah publik dan hanya dapat dibaca CloudFront lewat Origin Access Control. Fungsi Lambda berada di dalam VPC agar RDS tidak pernah dapat dihubungi dari luar.

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
| `/api/*` | Lambda Function URL, dilindungi Origin Access Control |

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
├── app.ts              Express app. Tidak mengetahui Lambda maupun AWS
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

### 6.3 Penerapan: Lambda Web Adapter

Backend dikemas sebagai container image berisi Express biasa. **Lambda Web Adapter** ditambahkan sebagai satu binary di dalam image, dan bertugas menerjemahkan event Lambda menjadi request HTTP ke aplikasi.

```dockerfile
FROM node:24-slim
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:1.0.1 /lambda-adapter /opt/extensions/lambda-adapter

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY dist ./dist

ENV PORT=8080
ENV AWS_LWA_READINESS_CHECK_PATH=/healthz
CMD ["node", "dist/server.js"]
```

**Cara kerjanya.** Lambda menyetel `AWS_LAMBDA_EXEC_WRAPPER`, sehingga adapter dijalankan lebih dahulu, menyalakan `CMD`, lalu menunggu readiness check pada `/healthz` sebelum trafik dialirkan. Setiap invocation diterjemahkan menjadi request HTTP/1.1 ke `127.0.0.1:8080`, dan responsnya diterjemahkan kembali menjadi payload Lambda.

**Aplikasi tidak mengetahui keberadaan Lambda.** Tidak ada handler, tidak ada `event`, tidak ada `context`, dan tidak ada `serverless-http`. Di luar Lambda, `AWS_LAMBDA_EXEC_WRAPPER` tidak ada, sehingga container langsung menjalankan `CMD` dan binary adapter menganggur di dalam image. Inilah sebabnya `entry/` cukup memuat satu berkas.

| Parameter | Nilai | Alasan |
|---|---|---|
| Arsitektur | arm64 | ~20% lebih murah dari x86 pada harga Lambda |
| Memori | 1024 MB | Cukup untuk Express, Drizzle, dan render pdfmake. Memori juga menentukan porsi CPU |
| Batas waktu fungsi | 30 detik | Request terpanjang adalah render satu PDF, di bawah 2 detik. Batas keras Function URL sendiri 15 menit |
| Connection pool | `max: 1` | Satu instance Lambda melayani satu request pada satu waktu. Pool lebih besar hanya meminta koneksi yang tidak akan terpakai |
| Reserved concurrency | 40 | Rem terakhir. Plafon `db.t4g.micro` sekitar 106 koneksi, sehingga 40 instance serentak tetap aman. Request ke-41 memperoleh `429` yang dapat diulang |
| Readiness check | `GET /healthz` — memeriksa proses dan koneksi basis data | Trafik tidak masuk sebelum pool siap |
| Mode invocation | `buffered` | Respons besar tidak pernah terjadi: berkas rapor dikembalikan sebagai presigned URL, bukan sebagai isi respons (§10) |
| Penggantian versi | Perbarui image, lalu pindahkan alias | Tidak ada waktu mati |

**Cold start.** Request pertama setelah masa senggang memerlukan sekitar 0,8–1,5 detik karena image container dan penempatan di dalam VPC. Ini paling terasa pada pengguna pertama di pagi hari. Konsekuensi ini diterima untuk MVP; apabila terbukti mengganggu saat UAT, jalur naiknya ada dua — provisioned concurrency, atau berpindah ke ECS Fargate memakai **image yang sama persis** tanpa perubahan kode aplikasi (CK-13).

> Menolak request itu dapat dipulihkan. Basis data yang mati saat enam puluh orang sedang bekerja tidak.

### 6.4 Pembatasan laju

Tidak ada API Gateway maupun ALB, sehingga pembatasan laju berada di dalam Express (`express-rate-limit`) dan disandarkan pada **identitas pengguna**, bukan alamat IP — satu sekolah kerap berbagi satu alamat IP publik.

Karena setiap instance Lambda memiliki memorinya sendiri, penghitung pembatas laju disimpan di PostgreSQL, bukan di memori proses. Tanpa itu, batas hanya berlaku per instance dan mudah dilampaui.

| Jalur | Batas | Alasan |
|---|---|---|
| `POST /api/auth/login` | Per akun dan per alamat IP | Menahan percobaan kata sandi beruntun. Relevan karena §6.1.3 meniadakan syarat kerumitan kata sandi |
| Tombol Suggestion | Per siswa | Setiap penekanan memanggil layanan AI dan menggerus kredit (§9, §15.3) |
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

**Akun Administrator dibuat langsung ke basis data melalui perintah CLI**, bukan melalui antarmuka aplikasi. Aplikasi tidak memiliki layar maupun endpoint pembuatan akun Administrator dalam bentuk apa pun.

```bash
npm run admin:create -- --nama-pengguna <pengenal> --nama "<nama lengkap>"
```

Perintah mencetak kata sandi awal yang dihasilkan sistem ke keluaran terminal, sekali dan tidak dapat ditampilkan ulang. Di lingkungan AWS, perintah dijalankan dengan memanggil fungsi Lambda `migrate` yang memakai image yang sama dengan argumen berbeda (§12); di lingkungan on-prem maupun pengembangan, dijalankan langsung di dalam container. Penggantian kata sandi Administrator memakai perintah yang sama dengan sub-perintah berbeda.

Hal ini menutup temuan **T-03** pada RFC-001 §10, yang mencatat bahwa PRD tidak mengatur cara akun Administrator dibuat — asumsi selama ini adalah Administrator sudah ada sejak awal.

---

## 9. AI Insight

Tombol Suggestion (PRD §8.5) dilayani melalui satu endpoint sinkron yang hanya membaca.

```
POST /api/me/suggestion
  → baca data siswa penekan tombol lewat koneksi app_ro
  → susun prompt
  → panggil Elice AI Cloud (POST /v1/chat/completions)
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

### 9.1 Penyedia

Layanan AI disediakan **Elice AI Cloud** melalui program KADA, yang antarmukanya **setara OpenAI**: `POST /v1/chat/completions` dengan otorisasi `Bearer`, ditambah `GET /v1/models` dan `POST /v1/responses`.

```
POST https://mlapi.run/{endpoint-id}
Authorization: Bearer {API_KEY}
Content-Type: application/json
```

Kesetaraan dengan antarmuka OpenAI inilah yang membuat pilihan ini tidak mengikat. Adapter yang ditulis adalah klien OpenAI-compatible biasa, sehingga berpindah penyedia — ke layanan lain, atau ke model yang dipasang sendiri di server sekolah lewat vLLM, Ollama, maupun LiteLLM — berarti **mengganti base URL dan kunci API**, bukan menulis ulang adapter.

Pemanggilan tetap melewati interface `AiAdvisor` di `ports/`, sehingga lapisan domain dan rute tidak mengetahui penyedia mana yang dipakai.

Kunci API disimpan di Secrets Manager dan tidak pernah masuk ke repositori maupun ke image.

### 9.2 Minimalisasi data yang dikirim

Prompt disusun **tanpa identitas siswa**. Nama, NIS, dan pengenal apa pun tidak dikirim; yang dikirim hanya nama mata pelajaran, nilai per komponen, KKM, kelengkapan, topik, dan persentase kehadiran.

Ini bukan sekadar kehati-hatian: data akademik dikirim ke layanan pihak ketiga, dan **V6** pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md) mensyaratkan kebijakan privasi serta penggunaan data nyata untuk AI **divalidasi dengan sekolah** sebelum sistem memuat data sungguhan. Dengan identitas tidak pernah dikirim, yang perlu divalidasi menjadi jauh lebih sempit.

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
| **Function URL** | Auth type `AWS_IAM`. Hanya menerima request bertanda tangan SigV4 dari distribusi CloudFront yang ditunjuk lewat Origin Access Control |
| **Fungsi Lambda** | Di dalam VPC, subnet privat, tanpa alamat IP publik |
| **RDS** | Subnet privat-data, security group hanya mengizinkan security group fungsi Lambda |
| **S3 frontend** | Bucket privat, hanya dapat dibaca CloudFront lewat Origin Access Control |
| **S3 rapor** | Bucket privat, akses hanya lewat presigned URL berumur pendek |
| **Jalur keluar** | NAT instance `t4g.nano` untuk menghubungi Elice AI Cloud. S3 lewat gateway endpoint yang tidak berbiaya, sehingga unggah dan unduh rapor tidak melewati NAT |
| **Header keamanan** | HSTS, `X-Content-Type-Options`, `Referrer-Policy`, dan Content Security Policy diatur di CloudFront Response Headers Policy |
| **Rahasia** | Secrets Manager. Tidak ada kredensial di dalam image maupun repositori |

> ⚠️ Sertifikat ACM untuk CloudFront **wajib diterbitkan di `us-east-1`**, sedangkan seluruh sumber daya lain berada di `ap-southeast-1`. Ditangani dengan provider alias kedua pada Terraform. Kelalaian pada butir ini menggagalkan `terraform apply`.
>
> ⚠️ **Wajib dibuktikan pada hari pertama infrastruktur naik:** perilaku penandatanganan Origin Access Control terhadap request **ber-body** — `POST` dan `PATCH` seperti Simpan Nilai. Kombinasi OAC dengan Function URL memiliki ketentuan tersendiri mengenai penyertaan body dalam tanda tangan SigV4. Diuji lewat request sungguhan sejak API masih berupa stub, bukan ditemukan ketika frontend mulai menyimpan nilai.

---

## 12. Infrastruktur dan pengiriman

```
infra/
├── bootstrap/          S3 state + tabel kunci (dijalankan sekali, state lokal)
├── modules/
│   ├── network/        VPC, subnet, NAT instance, security group, S3 gateway endpoint
│   ├── data/           RDS, subnet group, Secrets Manager
│   ├── compute/        ECR, fungsi Lambda api + migrate, Function URL, IAM
│   ├── frontend/       S3 + CloudFront + OAC (S3 dan Function URL) + ACM (provider alias us-east-1)
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
              → build image → push ke ECR
              → invoke fungsi migrate, tunggu selesai
              → perbarui image fungsi api, pindahkan alias
              → s3 sync frontend → invalidasi CloudFront
```

Migrasi dijalankan oleh **fungsi Lambda tersendiri yang memakai image yang sama** dengan perintah berbeda, dipanggil sekali oleh pipeline sebelum fungsi `api` diperbarui. Menjalankannya pada saat proses aplikasi menyala tidak dapat diterima di Lambda, karena banyak instance dapat menyala bersamaan dan menjalankan migrasi yang sama secara serentak.

Perintah pembuatan akun Administrator (§8) memakai fungsi yang sama dengan argumen berbeda, sehingga tidak diperlukan jalur akses tambahan ke basis data.

**Urutan yang mengurangi risiko:** infrastruktur dinaikkan lebih dahulu dengan API yang hanya memuat `GET /healthz`. VPC, RDS, Function URL, CloudFront, dan pipeline terbukti hidup **sebelum** backend sesungguhnya selesai — termasuk pembuktian penandatanganan OAC atas request ber-body (§11). Setelah itu yang berubah hanya isi image, bukan infrastruktur.

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
| Image aplikasi | ✅ | **Tidak ada.** Image yang sama dijalankan `docker run`. Binary Lambda Web Adapter di dalamnya menganggur karena `AWS_LAMBDA_EXEC_WRAPPER` tidak ada (§6.3) |
| Rute, validasi, middleware, autentikasi | ✅ | Tidak disentuh |
| Logika domain | ✅ | Tidak disentuh |
| Drizzle, skema, dan migrasi | ✅ | Tidak disentuh |
| Pembatasan laju | ✅ | Tidak disentuh — berada di dalam aplikasi |
| Basis data | ✅ | RDS PostgreSQL adalah PostgreSQL biasa. `pg_dump` lalu `pg_restore`; yang berubah hanya `DATABASE_URL` |
| Frontend | ✅ | Berkas statis yang sama disajikan Nginx, atau langsung oleh Express |
| Penyimpanan berkas | ⚠️ | Tetap S3, atau ditukar ke MinIO maupun disk lokal lewat `adapters/local` |
| Rahasia | ⚠️ | Secrets Manager ditukar variabel lingkungan |
| Penyedia AI | ⚠️ | Elice AI Cloud tetap dipakai, atau diarahkan ke penyedia lain maupun model yang dipasang sendiri. Karena antarmukanya setara OpenAI, yang berubah hanya base URL dan kunci API (§9.1) |
| Connection pool | ⚠️ | `max: 1` menjadi `max: 10`. **Satu baris konfigurasi**, dibaca dari variabel lingkungan |
| CloudFront, Function URL, dan NAT | ❌ | Digantikan reverse proxy tunggal, misalnya Nginx atau Caddy |

Yang berpindah bersama sistem adalah tanggung jawab operasional: pencadangan, pembaruan keamanan, enkripsi at-rest, dan risiko perangkat keras menjadi urusan pemilik server.

Tidak ada lapisan abstraksi yang dibangun khusus demi portabilitas. Portabilitas berasal dari bentuk artefaknya — sebuah container — dan dari kenyataan bahwa tim menjalankan container yang sama setiap hari.

---

## 15. Perkiraan biaya

### 15.1 Asumsi beban

Diturunkan dari volume RFC-001 §8.1 — 360 siswa, 18 guru, 1 administrator.

| Sumber | Perkiraan request per bulan |
|---|--:|
| Guru — 18 orang × 20 hari × ~150 request | 54.000 |
| Siswa — 360 orang × ~12 hari × ~30 request | 130.000 |
| Administrator dan lain-lain | ~15.000 |
| **Dipakai untuk perhitungan** (dibulatkan naik sebagai margin) | **300.000** |

### 15.2 Rincian

| Komponen | Per bulan |
|---|---|
| Lambda — arm64, 1024 MB, rata-rata ~120 ms, 300 ribu request | **~$1** |
| Function URL | $0 |
| RDS `db.t4g.micro` Single-AZ + 20 GB gp3 | $15–18, atau **$0** apabila free tier akun masih berlaku |
| NAT instance `t4g.nano` + alamat IPv4 publik + EBS | ~$8 |
| CloudFront dan S3 | $0–2 |
| ECR | < $1 |
| Elice AI Cloud — sekitar 1.400 panggilan tombol Suggestion | **$0** — memakai kredit program KADA, di luar tagihan AWS |
| **Total** | **$27–35**, atau **$12–20** dengan free tier |

> ⚠️ Seluruh angka berasal dari daftar harga terbitan AWS untuk `ap-southeast-1` dan **belum diverifikasi lewat AWS Pricing Calculator**. Wajib diperiksa sebelum masuk `DEPLOYMENT.md`.

### 15.3 Yang perlu diperhatikan

**Compute bukan lagi pos yang perlu dioptimalkan.** Lambda menyumbang sekitar 3% dari tagihan; sepuluh kali lipat trafik pun tetap di bawah $6. Tagihan didominasi RDS (~55%) dan NAT (~28%).

Dua tuas yang tersisa, keduanya perlu diperiksa lebih dahulu, bukan diasumsikan berhasil:

| Tuas | Hemat | Syarat |
|---|--:|---|
| Free tier RDS | −$15 | Status kelayakan akun AWS tim perlu diperiksa. Berlaku 12 bulan |
| Egress-only Internet Gateway lewat IPv6, menggantikan NAT instance | −$8 | Hanya berlaku apabila `mlapi.run` dapat dihubungi lewat IPv6. **Wajib diuji** |

Apabila keduanya berhasil, tagihan turun ke sekitar **$5–10 per bulan**.

**Alamat IPv4 publik kini ditagih** sekitar $3,65 per bulan per alamat, sejak Februari 2024. Inilah sebabnya NAT instance berbiaya ~$8, bukan ~$3 sebagaimana perkiraan pada rancangan 2 Agustus 2026.

**Sebagai pembanding**, rancangan ECS Fargate dengan dua task di belakang Application Load Balancer berbiaya **$60–71 per bulan** — selisihnya berasal dari Fargate (~$20) dan ALB (~$16). Perbandingan ini dicatat karena Fargate tetap menjadi jalur naik apabila cold start atau plafon koneksi terbukti mengganggu (CK-13).

**Biaya AI berada di luar tagihan AWS.** Layanan AI memakai kredit program KADA, bukan kartu tagihan tim. Konsekuensinya kredit bersifat **terbatas dan menipis**, bukan biaya berulang: pembatasan laju per siswa (§6.4) adalah pengendali pemakaiannya, dan habisnya kredit tidak menghentikan aplikasi karena kegagalan layanan AI ditangani sebagai kegagalan lunak (AC-21, §9). Besaran kredit dan tanggal berakhirnya dicatat sebagai butir 9 pada §16.

---

## 16. Yang belum diputuskan

| # | Item | Menunggu | Dampak apabila berubah |
|---|---|---|---|
| 1 | Model yang dipilih dari Model Library Elice beserta prompt sistemnya | Uji keluaran terhadap AC-18 dan AC-31 | Hanya isi adapter. Tidak menyentuh arsitektur |
| 2 | Format rapor resmi sekolah | V5 pada ATURAN-DAN-KRITERIA §5 | Menentukan templat pdfmake. Apabila tata letaknya rumit, perlu ditinjau ulang terhadap CK-09 |
| 3 | Kebijakan penyimpanan dan pencadangan data | V6 | Menentukan lama retensi cadangan RDS dan aturan daur hidup bucket rapor |
| 4 | Nama domain dan penerbitan sertifikat | Pihak sekolah | Menentukan modul `frontend` pada Terraform |
| 5 | Apakah `dev` memerlukan RDS tersendiri atau cukup PostgreSQL lokal | Keputusan tim | Menentukan biaya lingkungan `dev` |
| 6 | Apakah `mlapi.run` dapat dihubungi lewat IPv6, sehingga NAT instance dapat digantikan Egress-only Internet Gateway | Uji jaringan saat infrastruktur naik | Menghemat ~$8 per bulan, yaitu 28% tagihan (§15.3) |
| 7 | Status kelayakan free tier akun AWS tim | Pemeriksaan akun | Menentukan apakah tagihan ~$27 atau ~$12 per bulan |
| 8 | Apakah cold start ~0,8–1,5 detik dapat diterima pengguna | UAT | Apabila tidak, jalur naiknya provisioned concurrency atau ECS Fargate memakai image yang sama (CK-13) |
| 9 | Besaran sisa kredit Elice dan **tanggal berakhirnya program KADA** | Ketentuan program | Menentukan kapan penyedia AI harus diganti. Karena antarmukanya setara OpenAI, penggantian berarti mengubah base URL dan kunci API (§9.1) |
| 10 | Memakai **Dedicated Endpoint** (`mlapi.run/{id}`) atau endpoint bersama `/v1/chat/completions` | Uji ketersediaan dan latensi | Menentukan nilai base URL pada adapter. Tidak menyentuh arsitektur |
| 11 | Persetujuan sekolah atas pengiriman data akademik ke layanan AI pihak ketiga | V5 dan **V6** pada ATURAN-DAN-KRITERIA §5 | Apabila ditolak, tombol Suggestion memerlukan model yang dipasang sendiri. Dimitigasi sejak awal dengan tidak pernah mengirim identitas siswa (§9.2) |

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

**Konsekuensi yang diterima.** Kunci API perlu disimpan di Secrets Manager dan dirotasi. Jalur keluar internet menjadi kebutuhan tetap, sehingga NAT instance tidak dapat dihapus — dan sejak alamat IPv4 publik ditagih, pos ini menjadi 28% tagihan (§15.3). Kemungkinan penggantiannya dengan Egress-only Internet Gateway lewat IPv6 dicatat sebagai butir 6 pada §16.

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

**Catatan.** Pada rancangan 2 Agustus 2026, Drizzle dipilih karena ukuran bundel dan cold start di Lambda. Alasan itu **berlaku kembali** setelah CK-13, tetapi tidak lagi menjadi alasan utamanya: yang menentukan adalah kedekatannya dengan SQL sebagaimana diuraikan di atas.

### CK-12 · 6 Agustus 2026 · Tanpa Ansible

**Diputuskan.** Seluruh infrastruktur dikelola Terraform.

**Alasan.** NAT instance adalah satu-satunya host di dalam sistem, dan konfigurasinya cukup ditangani user data. Tidak ada armada server untuk dikelola. Apabila kelak sistem dipasang di server sekolah, pengelolaan konfigurasi host di sana adalah pekerjaan yang berbeda dan ditetapkan tersendiri.

### CK-13 · 6 Agustus 2026 · Lambda Web Adapter dan Function URL — mengamandemen CK-01 dan CK-02

**Diputuskan.** Backend berjalan di AWS Lambda dari **container image** memakai **Lambda Web Adapter**, diakses lewat **Function URL** dengan CloudFront Origin Access Control. ECS Fargate dan Application Load Balancer **tidak dipakai**, dan tetap tercatat sebagai jalur naik.

**Yang berubah dari CK-01 dan CK-02.** Keduanya menolak Lambda dengan tiga alasan. Ketiganya ditinjau ulang, dan hanya satu yang bertahan:

| Alasan penolakan pada CK-01 dan CK-02 | Status setelah ditinjau |
|---|---|
| Express memerlukan `serverless-http`, sehingga tidak berjalan apa adanya | **Gugur.** Lambda Web Adapter menerjemahkan event menjadi request HTTP ke aplikasi yang mendengarkan di port 8080. Tidak ada adapter di dalam kode aplikasi, dan `entry/` tetap satu berkas |
| Batas 29 detik API Gateway tidak dapat dinaikkan | **Gugur dua kali.** Pertama, sejak Juni 2024 batas tersebut dapat dinaikkan untuk REST API regional dan privat, dengan konsekuensi penurunan account-level throttle quota. Kedua, Function URL tidak memakai API Gateway sama sekali dan berbatas 15 menit. Ditambah lagi, CK-09 sudah menghapus satu-satunya pekerjaan panjang yang ada |
| Portabilitas menuntut entry point terpisah dan penukaran adapter antrean | **Gugur.** Artefaknya adalah image Docker yang sama, yang oleh AWS dinyatakan dapat dijalankan di Lambda, EC2, Fargate, dan komputer lokal. Di luar Lambda, `AWS_LAMBDA_EXEC_WRAPPER` tidak ada sehingga adapter tidak pernah dipanggil |
| Satu instance melayani satu request, sehingga pool wajib `max: 1` | **Bertahan.** Ini konsekuensi yang diterima, ditangani dengan reserved concurrency 40 terhadap plafon ~106 koneksi (§6.3) |

**Alasan.** Setelah tiga dari empat keberatan gugur, yang tersisa adalah selisih biaya yang besar untuk manfaat yang tidak lagi ada: Fargate dengan dua task di belakang ALB berbiaya **$60–71 per bulan**, sedangkan susunan ini **$27–35** — dan compute di dalamnya hanya ~$1. Membayar ~$36 per bulan untuk penyeimbang beban terhadap dua container yang melayani 379 pengguna tidak sepadan.

**Alternatif yang ditolak.**

*ECS Fargate satu task tanpa redundansi AZ.* Menekan biaya menjadi $45–55. Ditolak karena masih membayar ALB penuh sambil mengorbankan ketersediaan — kombinasi terburuk dari kedua pilihan.

*EC2 dengan Docker, tanpa ALB.* Sekitar $33–40. Ditolak karena mengembalikan sistem operasi yang harus dipatch, tanpa rolling deploy dan tanpa penskalaan, demi penghematan yang lebih kecil daripada susunan ini.

*API Gateway di depan Lambda.* Ditolak karena TLS, domain kustom, dan titik pemasangan WAF sudah disediakan CloudFront, sedangkan pembatasan laju dan validasi request sengaja ditempatkan di dalam Express agar ikut berpindah ke on-prem (§6.4). HTTP API juga justru lebih ketat, terkunci di 30 detik. Akan ditinjau ulang apabila kelak ada konsumen API di luar frontend sendiri — hal yang saat ini dicoret NG2.

**Konsekuensi yang diterima.**

1. **Cold start ~0,8–1,5 detik** pada request pertama setelah masa senggang. Dicatat sebagai butir 8 pada §16 untuk dibuktikan saat UAT.
2. **Pool `max: 1`** beserta disiplin reserved concurrency.
3. **Ketergantungan pada proyek `awslabs/aws-lambda-web-adapter`**, yang merupakan open source milik AWS dan bukan layanan berdukungan formal. Versi image adapter **wajib dipatok** — variabel lingkungan tanpa prefiks `AWS_LWA_` sudah usang dan akan dihapus pada versi 2.0.
4. **Perilaku penandatanganan OAC atas request ber-body wajib dibuktikan** sejak hari pertama infrastruktur naik (§11).

**Yang membuat keputusan ini dapat dibalik.** Ketiga konsekuensi pertama diselesaikan dengan berpindah ke ECS Fargate memakai **image yang sama persis**: yang berubah hanya modul Terraform, ditambah `max: 1` menjadi `max: 10` yang dibaca dari variabel lingkungan. Nol perubahan kode aplikasi. Inilah yang membedakannya dari rancangan Lambda 2 Agustus 2026, yang mengikat kode ke Lambda lewat `serverless-http` dan entry point terpisah.

### CK-14 · 6 Agustus 2026 · Elice AI Cloud, bukan OpenRouter — mengamandemen CK-06

**Diputuskan.** Tombol Suggestion dilayani **Elice AI Cloud** melalui program KADA, dengan antarmuka setara OpenAI (`POST /v1/chat/completions`, otorisasi `Bearer`).

**Alasan.**

1. **Kredit sudah tersedia** melalui program KADA, sehingga biaya AI keluar dari tagihan berulang dan menjadi kredit terbatas yang perlu dijaga pemakaiannya.
2. **Antarmukanya setara OpenAI.** Ini yang menentukan, bukan penyedianya. Adapter yang ditulis adalah klien OpenAI-compatible biasa, sehingga tuntutan portabilitas pada CK-06 justru terpenuhi lebih baik daripada dengan OpenRouter: berpindah ke penyedia lain maupun ke model yang dipasang sendiri di server sekolah cukup dengan mengubah base URL dan kunci API.

**Yang tidak berubah dari CK-06.** Penolakan terhadap **Amazon Bedrock** tetap berlaku, dengan alasan yang sama: Bedrock mengikat jalur AI ke AWS sehingga pemasangan on-prem memerlukan penyedia lain yang keluarannya belum tentu setara. Interface `AiAdvisor` di `ports/` juga tetap, sehingga lapisan domain dan rute tidak mengetahui penyedia mana yang dipakai.

**Konsekuensi yang diterima.**

1. **Kredit terbatas dan terikat program.** Sisa kredit dan tanggal berakhirnya dicatat sebagai butir 9 pada §16. Habisnya kredit tidak menghentikan aplikasi, karena kegagalan layanan AI ditangani sebagai kegagalan lunak (AC-21).
2. **Data akademik dikirim ke layanan pihak ketiga.** Dimitigasi dengan tidak pernah mengirim identitas siswa (§9.2), dan tetap memerlukan persetujuan sekolah sesuai V6 — dicatat sebagai butir 11 pada §16.
3. **Jalur keluar internet tetap dibutuhkan**, sehingga NAT instance tidak dapat dihapus kecuali IPv6 terbukti bekerja (§16 butir 6).

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 6 Agustus 2026 | Dokumen dibuat. Menetapkan stack di atas PRD v3.0 dan RFC-001. Menggantikan bagian stack pada `ARCHITECTURE.md` versi 2 Agustus 2026. Menutup K-01 dan K-02 pada RFC-001 §9 melalui CK-05, CK-01, dan CK-04, serta menutup temuan T-03 melalui §8 |
| 6 Agustus 2026 | Penyedia AI berpindah dari OpenRouter ke Elice AI Cloud melalui program KADA (**CK-14**, mengamandemen CK-06). §2, §3, §6.4, §9, §11, §14, §15, dan §16 disesuaikan. Ditambahkan §9.1 penyedia beserta antarmuka setara OpenAI, dan §9.2 minimalisasi data yang menetapkan identitas siswa tidak pernah dikirim |
| 6 Agustus 2026 | Compute berpindah dari ECS Fargate dengan ALB ke Lambda Web Adapter dengan Function URL (**CK-13**, mengamandemen CK-01 dan CK-02). §2, §3, §6.3, §6.4, §11, §12, §14, §15, dan §16 disesuaikan. Perkiraan biaya diperbaiki: NAT menjadi ~$8 karena alamat IPv4 publik kini ditagih, dan total turun menjadi $27–35 per bulan. Pembuatan akun Administrator ditetapkan melalui perintah CLI (§8) |
