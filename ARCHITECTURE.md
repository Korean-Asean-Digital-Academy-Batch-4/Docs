# Arsitektur — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v1.0 |
| **Tanggal** | 6 Agustus 2026 |
| **Disusun oleh** | Re:Code |
| **Sumber kebenaran** | [PRD.md](PRD.md) v3.0, [ATURAN-DAN-KRITERIA.md](ATURAN-DAN-KRITERIA.md) v1.0, [aktor-role.md](aktor-role.md) v3.0, [RFC-001](RFC-001-model-data-konseptual.md), dan [Techstack.md](Techstack.md) v2.0 |
| **Kedudukan** | Menetapkan **bagaimana bagian-bagian sistem terhubung**. Menggantikan `ARCHITECTURE.md` versi 2 Agustus 2026, yang sudah dihapus dan hanya tersedia pada riwayat Git |
| **Dokumen lanjutan** | `SCHEMA.md` — skema fisik · `API.md` — kontrak endpoint |

> Dokumen ini menjawab **bagaimana bagian-bagian terhubung**. Pilihan teknologi beserta alasannya berada pada [Techstack.md](Techstack.md); prosedur penerapan dan operasional berada pada [DEPLOYMENT.md](DEPLOYMENT.md). Dokumen ini tidak mengulang keduanya.
>
> Entitas dan invarian basis data ditetapkan [RFC-001](RFC-001-model-data-konseptual.md); aturan peran dan izin ditetapkan [aktor-role.md](aktor-role.md). Dokumen ini menjelaskan **cara keduanya ditegakkan**, bukan mengulang isinya.
>
> Mengikuti [konvensi dokumen](README.md), badan dokumen memuat **deskripsi keadaan yang berlaku** dan disunting langsung ketika berubah, sedangkan **Lampiran Catatan Keputusan** bernomor dan bertanggal serta hanya ditambah, tidak disunting.
>
> Bentuk endpoint yang disebut pada dokumen ini bersifat penjelas alur, bukan kontrak. Kontrak endpoint ditetapkan `API.md`.

---

## 1. Prinsip

[Techstack.md §1](Techstack.md) menetapkan tiga prinsip yang mendasari pilihan teknologi. Prinsip keempat berikut bersifat arsitektural dan menjadi alasan hampir setiap pilihan pada dokumen ini.

**④ Yang dapat dijamin basis data tidak diserahkan kepada disiplin kode.**

Invarian pada [RFC-001 §6](RFC-001-model-data-konseptual.md) adalah pernyataan yang harus benar sepanjang umur sistem. Sebagian di antaranya dapat dijadikan **keadaan yang mustahil tersimpan**, dan sebagian lagi tidak. Yang pertama ditegakkan basis data; hanya yang kedua yang menjadi tanggung jawab lapisan aplikasi.

### 1.1 Yang ditegakkan basis data

| Invarian | Cara penegakan |
|---|---|
| I-03 satu semester aktif per tahun ajaran | Partial unique index — `UNIQUE (tahun_ajaran_ref) WHERE aktif` |
| I-09 satu wali kelas per kelas per semester | Partial unique index |
| I-06 jenjang kelas sama dengan jenjang mata pelajaran | Composite foreign key `(mapel_ref, tingkat) → mapel(id, tingkat)` |
| I-13 satu nilai per siswa per komponen per penugasan | Unique constraint |
| I-14 satu sesi per penugasan per tanggal | Unique constraint |
| I-16 penghapusan sesi menghapus seluruh presensinya | `ON DELETE CASCADE` |
| I-23 AI tidak pernah menulis ke data akademik | Role `app_ro` tanpa hak tulis (Pasal 8) |
| I-12 nilai kosong terbedakan dari nilai nol | Ketiadaan baris sebagai representasi tunggal — tidak ada nilai kosong yang ambigu |

Perbedaannya menentukan. Ketidakcocokan jenjang pada I-06 bukan **dicegah pemeriksaan aplikasi** melainkan **tidak dapat tersimpan**, sehingga kekeliruan kode di jalur mana pun tetap ditolak PostgreSQL.

### 1.2 Yang tidak dapat ditegakkan basis data

Lima invarian berikut adalah aturan alur kerja lintas entitas atau aturan agregat. Keduanya berada di luar jangkauan constraint, dan menjadi tanggung jawab lapisan aplikasi secara terang-terangan.

| Invarian | Letak penegakan |
|---|---|
| I-10 jumlah bobot seluruh komponen tepat 100 | Validasi agregat di `domain/nilai.ts`, diperiksa pada setiap perubahan templat komponen |
| I-15 setiap siswa memiliki tepat satu status pada setiap sesi | **Terbelah.** Batas atas ditegakkan unique constraint `(sesi_ref, siswa_ref)`; kelengkapannya dijamin transaksi pembukaan sesi yang menyisipkan seluruh siswa kelas berstatus Alpa sekaligus |
| I-20 rapor hanya dapat difinalisasi bila seluruh mata pelajaran lengkap | Query pemeriksaan kelengkapan di dalam transaksi finalisasi (Pasal 11) |
| I-22 rapor final tidak dapat diubah Guru maupun Wali Kelas | Pemeriksaan status pada lapisan rute, sebelum transaksi dimulai |
| I-25 siswa hanya dapat membaca datanya sendiri | Lapis baris pada pemeriksaan kewenangan (Pasal 9) |

Penyebutan ini bukan formalitas. [RFC-001 §6.1](RFC-001-model-data-konseptual.md) menuntut agar invarian yang tidak dapat ditegakkan basis data **dinyatakan eksplisit, bukan diasumsikan aman**. Kelima baris di atas adalah tempat kekeliruan kode dapat menghasilkan data yang salah tanpa ditolak siapa pun, dan karenanya menjadi sasaran utama pengujian integrasi.

---

## 2. Peta besar

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
   │  VPC · 2 AZ · ap-southeast-3        │                  │
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

**Tidak ada sumber daya yang dapat dihubungi langsung dari internet.** Function URL disetel dengan auth type `AWS_IAM` sehingga hanya menerima request bertanda tangan SigV4 dari distribusi CloudFront yang ditunjuk — URL yang bocor tidak dapat dipakai siapa pun. Bucket S3 frontend tidak pernah publik dan hanya dapat dibaca CloudFront lewat Origin Access Control. Fungsi Lambda berada di dalam VPC agar RDS tidak pernah dapat dihubungi dari luar.

| Bagian | Tanggung jawab | Dihubungi oleh |
|---|---|---|
| CloudFront | TLS, domain, pembagian path, header keamanan | Publik |
| S3 frontend | Menyimpan hasil `vite build` | CloudFront saja |
| Lambda Function URL | Titik masuk backend | CloudFront saja, bertanda tangan SigV4 |
| Fungsi `api` | Seluruh logika aplikasi | Function URL saja |
| RDS PostgreSQL | Data dan penegakan invarian | Fungsi `api` saja |
| S3 rapor | Menyimpan berkas rapor terender | Fungsi `api`; pengguna lewat presigned URL |
| NAT instance | Jalur keluar menuju Elice AI Cloud | Fungsi `api` |

S3 dihubungi lewat **gateway endpoint** yang tidak berbiaya, sehingga unggah dan unduh berkas rapor tidak melewati NAT.

---

## 3. Pintu masuk dan pembagian path

Satu domain, dua origin. CloudFront membagi trafik berdasarkan path.

| Path | Origin | Perlindungan |
|---|---|---|
| `/*` | S3 — hasil `vite build` | Bucket privat, Origin Access Control |
| `/api/*` | Lambda Function URL | Origin Access Control, auth type `AWS_IAM` |

Karena frontend dan backend berada pada **satu domain**, tiga hal didapat sekaligus:

1. **CORS hilang seluruhnya.** Tidak ada preflight, tidak ada daftar origin yang perlu dipelihara, dan tidak ada perbedaan perilaku antara pengembangan dan produksi.
2. **Cookie sesi dapat memakai `HttpOnly`.** Token sesi tidak pernah tersentuh JavaScript, sehingga tidak dapat dicuri lewat XSS. Untuk sistem berisi data akademik anak di bawah umur, ini jauh lebih aman daripada menyimpan token di `localStorage`.
3. **Frontend cukup memanggil `/api/...`** tanpa base URL yang berbeda antar lingkungan.

Ketiganya adalah akibat langsung dari susunan ini, bukan fitur yang dibangun terpisah.

---

## 4. Frontend

React SPA yang dibangun Vite menjadi berkas statis, lalu disalin ke bucket S3 privat dan disajikan CloudFront. Menempatkan frontend di dalam container tidak dilakukan: hasil build adalah berkas statis, sehingga membayar compute untuk menyajikannya tidak memberi manfaat apa pun.

**Empat kebutuhan produk membentuk bentuk aplikasinya:**

| Kebutuhan | Dasar | Bentuk penerapan |
|---|---|---|
| Pemberitahuan berhasil atau gagal pada setiap perubahan data | P21, AC-27 | Satu komponen notifikasi global yang dipakai seluruh mutasi. Kegagalan menampilkan alasannya, bukan sekadar "gagal" |
| Penyimpanan hanya setelah tombol ditekan | P22, AC-15 | Keadaan formulir bersifat lokal sampai tombol simpan ditekan. **Tidak ada penyimpanan otomatis.** Meninggalkan halaman tanpa menyimpan tidak mengubah data |
| Matriks siswa terhadap komponen pada layar nilai | RFC-001 §5.1 | Data tersimpan memanjang satu baris per nilai; pemutaran menjadi matriks dilakukan di frontend. Tiga puluh siswa dikali delapan komponen berarti 240 sel |
| Terbaca pada perangkat bergerak | NG5 | Tata letak responsif. **Bukan** aplikasi Android maupun iOS |

**Fakta sumber dan penanda Data Sementara ditampilkan halaman**, bukan dituntut menjadi bagian teks keluaran AI (AC-19). Ini menempatkan kewajiban tersebut pada bagian yang dapat dikendalikan, bukan pada keluaran model yang susunannya bebas.

---

## 5. Struktur kode dan batas modul

```
src/
├── app.ts                    Express app. Tidak mengetahui Lambda maupun AWS
├── routes/                   HTTP: parsing, kode status, bentuk respons
├── domain/                   Aturan bisnis murni — TANPA I/O
│   ├── nilai.ts                rata-rata berbobot, kelengkapan komponen
│   ├── rapor.ts                transisi draft → finalized → distributed
│   └── presensi.ts             persentase kehadiran
├── db/                       Skema Drizzle, migrasi, dan query
├── ports/                    interface: Storage, AiAdvisor, Secrets
├── adapters/
│   ├── aws/                  S3, Secrets Manager, SSM  ← SATU-SATUNYA tempat SDK AWS
│   ├── openai-compatible/    implementasi AiAdvisor    ← dinamai menurut protokol (CK-A-02)
│   └── local/                disk, env                 ← dipakai on-prem dan pengembangan
└── entry/
    └── server.ts             satu-satunya entry point
```

### 5.1 Aturan impor

| Lapisan | Boleh mengimpor | Dilarang mengimpor |
|---|---|---|
| `domain/` | tidak ada | `db`, `adapters`, SDK AWS, `express` |
| `routes/` | `domain`, `db`, `ports` | `adapters/aws` secara langsung |
| `adapters/*` | `ports`, SDK yang bersangkutan | `domain`, `routes` |
| `entry/` | `app`, `adapters` | — |

`domain/` tanpa I/O berarti seluruh logika penilaian dapat diuji tanpa basis data dan tanpa AWS, dalam hitungan milidetik. Untuk bagian yang salah hitungnya berarti rapor siswa salah, ini bukan kemewahan.

Aturan `adapters/aws/` sebagai satu-satunya tempat SDK AWS adalah penerapan langsung prinsip ③ pada [Techstack.md §1](Techstack.md). Lapisan domain, rute, dan basis data tidak mengetahui keberadaan AWS.

**`entry/` hanya memuat satu berkas.** Tidak ada entry point terpisah untuk AWS, karena AWS menjalankan container yang sama dengan on-prem. Ini yang membedakannya dari rancangan 2 Agustus 2026, yang mengikat kode ke Lambda lewat `serverless-http`.

---

## 6. Penerapan Lambda Web Adapter

Backend dikemas sebagai container image berisi Express biasa. **Lambda Web Adapter** ditambahkan sebagai satu binary di dalam image, dan bertugas menerjemahkan event Lambda menjadi request HTTP ke aplikasi.

```dockerfile
FROM node:24-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY tsconfig*.json ./
COPY src ./src
RUN npm run build

FROM node:24-slim
# Versi adapter WAJIB dipatok (CK-13).
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:1.0.1 /lambda-adapter /opt/extensions/lambda-adapter

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist

ENV PORT=8080
ENV AWS_LWA_READINESS_CHECK_PATH=/healthz
EXPOSE 8080
CMD ["node", "dist/entry/server.js"]
```

**Dua tahap, bukan satu.** Tahap pertama membangun TypeScript **di dalam image**, sehingga `docker build` berdiri sendiri dan tidak menuntut `dist/` sudah ada di mesin yang membangunnya. Tahap kedua hanya membawa hasilnya beserta dependensi produksi. Bentuk ini yang terpasang dan terbukti jalan pada repositori backend.

Versi image adapter **wajib dipatok**. Variabel lingkungan tanpa prefiks `AWS_LWA_` sudah usang dan akan dihapus pada versi 2.0 (CK-13).

**Cara kerjanya.** Lambda menyetel `AWS_LAMBDA_EXEC_WRAPPER`, sehingga adapter dijalankan lebih dahulu, menyalakan `CMD`, lalu menunggu readiness check pada `/healthz` sebelum trafik dialirkan. Setiap invocation diterjemahkan menjadi request HTTP/1.1 ke `127.0.0.1:8080`, dan responsnya diterjemahkan kembali menjadi payload Lambda.

**Aplikasi tidak mengetahui keberadaan Lambda.** Tidak ada handler, tidak ada `event`, tidak ada `context`, dan tidak ada `serverless-http`. Di luar Lambda, `AWS_LAMBDA_EXEC_WRAPPER` tidak ada, sehingga container langsung menjalankan `CMD` dan binary adapter menganggur di dalam image. Inilah sebabnya `entry/` cukup memuat satu berkas, dan inilah yang membuat portabilitas pada Pasal 13 tidak memerlukan lapisan abstraksi apa pun.

| Parameter | Nilai | Alasan |
|---|---|---|
| Arsitektur | arm64 | ~20% lebih murah dari x86 pada harga Lambda |
| Memori | 1024 MB | Cukup untuk Express, Drizzle, dan render pdfmake. Memori juga menentukan porsi CPU |
| Batas waktu fungsi | 30 detik | Request terpanjang adalah finalisasi sekelas, yang merender berkas rapor dengan anggaran lunak 20 detik (Pasal 11). Batas keras Function URL sendiri 15 menit |
| Connection pool | `max: 1` | Satu instance Lambda melayani satu request pada satu waktu. Pool lebih besar hanya meminta koneksi yang tidak akan terpakai |
| Reserved concurrency | 40 | Rem terakhir. Plafon `db.t4g.micro` sekitar 106 koneksi, sehingga 40 instance serentak tetap aman. Request ke-41 memperoleh `429` yang dapat diulang |
| Readiness check | `GET /healthz` — memeriksa proses dan koneksi basis data | Trafik tidak masuk sebelum pool siap |
| Mode invocation | `buffered` | Respons besar tidak pernah terjadi: berkas rapor dikembalikan sebagai presigned URL, bukan sebagai isi respons (Pasal 11) |
| Penggantian versi | Perbarui image, lalu pindahkan alias | Tidak ada waktu mati |

**Cold start.** Request pertama setelah masa senggang memerlukan sekitar 0,8–1,5 detik karena image container dan penempatan di dalam VPC. Ini paling terasa pada pengguna pertama di pagi hari. Konsekuensi ini diterima untuk MVP; apabila terbukti mengganggu saat UAT, jalur naiknya ada dua — provisioned concurrency, atau berpindah ke ECS Fargate memakai **image yang sama persis** tanpa perubahan kode aplikasi (CK-13).

> Menolak request itu dapat dipulihkan. Basis data yang mati saat enam puluh orang sedang bekerja tidak.

---

## 7. Pembatasan laju

Tidak ada API Gateway maupun ALB, sehingga pembatasan laju berada **di dalam Express** dan disandarkan pada **identitas pengguna**, bukan alamat IP — satu sekolah kerap berbagi satu alamat IP publik.

Karena setiap instance Lambda memiliki memorinya sendiri, penghitung pembatas laju disimpan di **PostgreSQL**, bukan di memori proses. Tanpa itu, batas hanya berlaku per instance dan mudah dilampaui hanya dengan menunggu instance baru menyala (CK-A-03).

| Jalur | Batas | Alasan |
|---|---|---|
| `POST /api/auth/masuk` | **Dua lapis.** Per akun: 5 kegagalan per 15 menit. Per alamat IP: **30 kegagalan per 15 menit** | Menahan percobaan kata sandi beruntun. Relevan karena PRD §6.1.3 meniadakan syarat kerumitan kata sandi, sehingga pembatasan percobaan adalah satu-satunya pertahanan yang tersisa. Dua ambang yang berbeda, beserta alasannya, pada **CK-A-08** |
| Tombol Suggestion | **5 kali per jam per siswa** | Setiap penekanan memanggil layanan AI dan menggerus kredit KADA. Cukup untuk pemakaian wajar dalam satu sesi belajar |
| Unggah berkas | **10 unggahan per jam per pengguna**, dengan **batas ukuran 2 MB per berkas** | Administrator yang mengunggah ulang karena salah format tetap leluasa, sedangkan pemakaian memori tetap terkendali |

**Dasar angka 2 MB.** Berkas terbesar yang mungkin ada adalah daftar 360 siswa dengan tiga kolom — Kelas, NIS, dan Nama — yang berukuran sekitar 20–30 KB. Batas 2 MB memberi margin sekitar tujuh puluh kali lipat, longgar bagi berkas Excel yang membengkak karena pemformatan, dan tetap jauh dari membahayakan memori fungsi 1024 MB.

Percobaan yang melampaui batas dijawab `429` beserta keterangan kapan dapat dicoba kembali, dan ditampilkan frontend sebagai pemberitahuan kegagalan yang menyebutkan alasannya (P21).

Seluruh pembatasan ini berada **di dalam aplikasi**, sehingga ikut berpindah ke on-prem — berbeda dari pembatasan laju di tepi jaringan yang akan tertinggal di AWS.

---

## 8. Dua role basis data

Aplikasi menyambung ke PostgreSQL memakai dua role yang berbeda, dipilih berdasarkan jalur yang sedang dilayani.

```sql
-- role aplikasi: seluruh jalur tulis
GRANT SELECT, INSERT, UPDATE, DELETE ON nilai, presensi, sesi, rapor, rapor_mapel TO app_rw;

-- role jalur AI: hanya membaca
GRANT SELECT ON nilai, presensi, sesi, mapel, penugasan, penugasan_komponen TO app_ro;
-- TIDAK ADA INSERT, UPDATE, maupun DELETE. Sama sekali.
```

I-23 dan AC-20 mensyaratkan AI tidak pernah menulis ke data akademik. Dengan dua role terpisah, seandainya ada kekeliruan kode yang mencoba, **PostgreSQL yang menolak.**

**Cara penguji membuktikannya.** Sambung ke basis data sebagai `app_ro`, lalu jalankan satu pernyataan tulis:

```sql
-- dijalankan sebagai app_ro
UPDATE nilai SET nilai = 100 WHERE id = '<pengenal mana pun>';
-- ERROR: permission denied for table nilai
```

Pembuktian AC-20 dengan demikian berupa **satu query yang gagal**, bukan pembacaan kode. Ini perbedaan yang menentukan: pembacaan kode membuktikan bahwa jalur yang ada saat ini tidak menulis, sedangkan penolakan basis data membuktikan bahwa jalur mana pun tidak akan bisa.

Kedua kredensial disimpan di Secrets Manager dan dapat dirotasi tanpa mengubah kode aplikasi ([Techstack.md §7](Techstack.md)).

---

## 9. Autentikasi dan sesi

Kredensial dikelola sendiri di dalam PostgreSQL. Tidak ada layanan identitas terkelola (CK-05).

### 9.1 Masuk dan sesi

| Aspek | Penerapan | Dasar |
|---|---|---|
| Pengenal masuk | `pengguna.nama_pengguna` — NIP bagi Guru, NIS bagi Siswa, nama pengguna tersendiri bagi Administrator | P20, RFC-001 §7.1 |
| Penyimpanan kata sandi | Argon2id | — |
| Kata sandi awal | Dihasilkan sistem, ditampilkan sekali kepada Administrator untuk diserahkan kepada pengguna | PRD §6.1.3, P17 |
| Syarat kerumitan | **Tidak ada** | PRD §6.1.3 |
| Kewajiban ganti saat masuk pertama | **Tidak ada** | PRD §6.1.3 |
| Pemulihan mandiri | **Tidak ada.** Tombol Lupa kata sandi hanya menampilkan pesan menghubungi Wali Kelas atau Administrator | PRD §6.1.3, AC-33 |
| Bentuk sesi | Token acak buram di dalam cookie `HttpOnly; Secure; SameSite=Strict`, dengan barisnya sendiri di basis data | CK-A-04 |
| Umur sesi | **12 jam, tanpa perpanjangan otomatis** | Cukup untuk satu hari kerja sekolah |
| Pencabutan | Baris sesi dihapus. Berlaku seketika pada request berikutnya | CK-A-04 |

**Mekanisme sesi berlaku sama bagi seluruh aktor** — Administrator, Guru, dan Siswa. Tidak ada perbedaan bentuk token, umur, maupun cara pencabutan antar peran; yang berbeda hanya kewenangan sesudah masuk.

Umur 12 jam tanpa perpanjangan otomatis dipilih karena pola pemakaiannya harian: pengguna masuk pada pagi hari dan selesai pada sore hari, lalu masuk kembali keesokan paginya. Perpanjangan bergulir menambah mekanisme yang perlu ditulis dan diuji tanpa menjawab kebutuhan yang ada.

Seluruh ketentuan PRD §6.1.3 di atas merupakan **perilaku bawaan** dari pendekatan ini, bukan hasil mematikan fitur pada layanan pihak lain (CK-05).

### 9.2 Pemeriksaan kewenangan per request

Kewenangan diperiksa **dua lapis** pada setiap request. Keduanya harus lulus; kegagalan salah satu dijawab `403`.

| Lapis | Sumber | Yang diperiksa |
|---|---|---|
| **Peran** | `pengguna.peran` dibaca dari basis data pada setiap request | Apakah peran ini memiliki kemampuan tersebut sama sekali |
| **Baris** | `penugasan.guru_ref`, `kelas.wali_kelas_ref`, atau `siswa_ref` | Apakah pengguna ini berhak atas **baris data yang diminta** |

Penerjemahan [matriks kewenangan aktor-role §6](aktor-role.md) menjadi kedua lapis tersebut:

| Kemampuan | Lapis peran | Lapis baris |
|---|---|---|
| Mengisi dan mengubah nilai | `administrator`, `guru` | Guru: `penugasan.guru_ref = pengguna.id`. Administrator: tanpa batas baris |
| Membuka, menyunting, menghapus sesi presensi | `administrator`, `guru` | Sama seperti di atas |
| Melihat nilai dan presensi lintas mata pelajaran | `administrator`, `guru`, `siswa` | Guru: `kelas.wali_kelas_ref = pengguna.id`. Siswa: `siswa_ref = pengguna.id` |
| Menulis catatan, finalisasi, distribusi rapor | `administrator`, `guru` | Guru: `kelas.wali_kelas_ref = pengguna.id` |
| Mengunduh rapor | `administrator`, `guru`, `siswa` | Guru: `kelas.wali_kelas_ref = pengguna.id`. Siswa: rapor miliknya **dan** berstatus `distributed` |
| Tombol Suggestion | `siswa` saja | `siswa_ref = pengguna.id` |

**Tidak ada nilai peran bernama `wali_kelas`.** Kewenangan Wali Kelas seluruhnya diturunkan dari lapis baris, yaitu keberadaan `kelas.wali_kelas_ref` yang menunjuk pengguna tersebut. Ini penerapan langsung [aktor-role.md §1](aktor-role.md), yang menyatakan Wali Kelas adalah kewenangan tambahan pada akun Guru dan bukan jenis akun keempat (CK-A-01).

Akibatnya dapat ditelusuri langsung pada contoh [aktor-role.md §4](aktor-role.md): Pak Cahyo lulus lapis peran untuk finalisasi karena ia `guru`, tetapi lulus lapis baris hanya pada X IPA 1 — kelas yang `wali_kelas_ref`-nya menunjuk dirinya. Pada X IPA 2 dan X IPA 3 yang juga diajarnya, lapis baris menolak.

**Guru Mata Pelajaran tidak memiliki jalur unduh rapor** (P23, AC-32). Pada tabel di atas, peran `guru` lulus lapis peran untuk pengunduhan, tetapi lapis baris menolak setiap Guru yang bukan Wali Kelas kelas tersebut. Tidak diperlukan aturan terpisah.

Kegagalan kewenangan dijawab **`403`, bukan `404`**. Sistem ini tertutup bagi publik dan seluruh penggunanya sudah terautentikasi, sehingga menyamarkan keberadaan sumber daya tidak memberi perlindungan tambahan, sementara pesan yang jelas mempercepat penelusuran ketika Administrator salah menetapkan penugasan.

### 9.3 Akun Administrator

**Akun Administrator dibuat langsung ke basis data melalui perintah CLI**, bukan melalui antarmuka aplikasi. Aplikasi tidak memiliki layar maupun endpoint pembuatan akun Administrator dalam bentuk apa pun.

**Dua perintah, dua keadaan yang berbeda** (CK-A-09).

```bash
# Bootstrap — kata sandi ditentukan operator, dibaca dari stdin.
printf '%s' '<kata sandi>' | npm run seed:admin -- <pengenal> "<nama lengkap>"

# Reset — kata sandi dibangkitkan sistem, dicetak sekali.
npm run admin:create -- --nama-pengguna <pengenal> --nama "<nama lengkap>"
```

`seed:admin` dipakai membuat Administrator **pertama**, ketika belum ada seorang pun yang dapat masuk. `admin:create` dipakai sesudahnya, dan mengikuti P17 seperti akun Guru dan Siswa: kata sandi dibangkitkan sistem lalu diserahkan.

`admin:create` mencetak kata sandi yang dihasilkan sistem ke keluaran terminal, sekali dan tidak dapat ditampilkan ulang **pada on-prem dan pengembangan**. `seed:admin` tidak mencetak apa pun, karena operator sudah mengetahui kata sandinya.

**Di AWS `admin:create` menolak berjalan.** Fungsi `migrate` mengalirkan seluruh `stdout` ke CloudWatch Logs, dan itu perilaku runtime Lambda yang tidak dapat dimatikan dari dalam aplikasi. Mencetak kata sandi di sana berarti menyimpannya sebagai teks polos yang bertahan selama retensi log — dapat dibaca siapa pun yang memegang hak baca CloudWatch, tanpa perlu menyentuh basis data. Jaminan "tidak dapat ditampilkan ulang" karenanya **tidak berlaku** di jalur itu, dan perintahnya berhenti dengan pesan alih-alih diam-diam membocorkannya.

Cara membuat Administrator pertama di AWS **belum diputuskan**, dan tercatat sebagai titik henti manusia. Jalur yang paling mungkin: menulis kata sandi ke Secrets Manager berumur pendek lalu mencetak ARN-nya saja ke log, sehingga yang masuk CloudWatch adalah rujukan, bukan rahasianya. Di lingkungan AWS, perintah dijalankan dengan memanggil fungsi Lambda `migrate` yang memakai image yang sama dengan argumen berbeda; di lingkungan on-prem maupun pengembangan, dijalankan langsung di dalam container. Penggantian kata sandi Administrator memakai perintah yang sama dengan sub-perintah berbeda.

Hal ini menutup temuan **T-03** pada [RFC-001 §10](RFC-001-model-data-konseptual.md), yang mencatat bahwa PRD tidak mengatur cara akun Administrator dibuat.

---

## 10. AI Insight

Tombol Suggestion (PRD §8.5) dilayani melalui satu endpoint sinkron yang hanya membaca.

```
POST /api/saya/suggestion
  → baca data siswa penekan tombol lewat koneksi app_ro
  → hitung kelengkapan dan persentase kehadiran di domain/
  → susun prompt tanpa identitas
  → panggil Elice AI Cloud (POST /v1/chat/completions)
  → kembalikan teks ke frontend
  → tidak menulis apa pun
```

| Aspek | Penerapan | Dasar |
|---|---|---|
| Pemicu | Hanya penekanan tombol oleh siswa. Tidak berjalan otomatis | PRD §8.5, AC-16 |
| Data yang dibaca | Hanya milik siswa yang menekan tombol, dibatasi ganda: klausa `WHERE` pada siswa yang bersangkutan, **dan** koneksi `app_ro` yang tidak dapat menulis | I-23, AC-17, AC-20 |
| Perhitungan | Rata-rata berbobot, kelengkapan, dan persentase kehadiran dihitung `domain/`, **bukan oleh AI** | PRD §8.6 butir 1 |
| Penyimpanan keluaran | **Tidak ada tabel, tidak ada cache, tidak ada log isi keluaran** | I-24, NG14, AC-16 |
| Kegagalan layanan | Kegagalan lunak: halaman menampilkan pesan bahwa rekomendasi tidak dapat dibuat, sedangkan nilai, presensi, finalisasi, dan distribusi tetap berjalan | AC-21, PRD §8.6 butir 7 |
| Batas waktu | 20 detik, lalu dibatalkan | Menahan permintaan menggantung |
| Fakta sumber dan penanda Data Sementara | Ditampilkan **oleh halaman** di sekitar keluaran AI | PRD §8.5, AC-19 |

### 10.1 Susunan prompt

Prompt disusun **tanpa identitas siswa**. Nama, NIS, dan pengenal apa pun tidak dikirim; yang dikirim hanya nama mata pelajaran, nilai per komponen, KKM, kelengkapan, topik, dan persentase kehadiran.

```
Sistem : peran konsultan pendidikan, bahasa profesional,
         wajib memuat tiga unsur — rekomendasi belajar,
         alasan rekomendasi, dan dua pilihan tindakan yang realistis
Konteks: per mata pelajaran — nama, KKM, topik per komponen,
         nilai per komponen, kelengkapan, persentase kehadiran
         TANPA nama siswa, TANPA NIS, TANPA pengenal apa pun
```

Larangan pada PRD §8.6 dinyatakan di dalam prompt sistem, tetapi **tidak disandarkan kepadanya**. Larangan yang benar-benar mengikat ditegakkan arsitektur: AI tidak dapat mengubah data karena koneksinya tidak memiliki hak tulis, dan AI tidak dapat membaca data siswa lain karena kueri sumbernya sudah dibatasi sebelum prompt disusun. Prompt sistem hanya mengatur gaya dan susunan keluaran.

Minimalisasi identitas bukan sekadar kehati-hatian. Data akademik dikirim ke layanan pihak ketiga, dan **V6** pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md) mensyaratkan kebijakan privasi serta penggunaan data nyata untuk AI **divalidasi dengan sekolah**. Dengan identitas tidak pernah dikirim, yang perlu divalidasi menjadi jauh lebih sempit.

### 10.2 Adapter

Pemanggilan melewati interface `AiAdvisor` di `ports/`, sehingga lapisan domain dan rute tidak mengetahui penyedia mana yang dipakai.

```
POST https://mlapi.run/{endpoint-id}/v1/chat/completions
Authorization: Bearer {API_KEY}
Content-Type: application/json
```

Implementasinya berada di `adapters/openai-compatible/` — dinamai menurut **protokol**, bukan menurut penyedia (CK-A-02). Berpindah penyedia, atau berpindah ke model yang dipasang sendiri di server sekolah lewat vLLM, Ollama, maupun LiteLLM, berarti **mengganti base URL dan kunci API** tanpa menyentuh nama direktori maupun isi adapter.

---

## 11. Berkas rapor

Berkas rapor **dirender pada saat finalisasi**, dengan render-saat-unduh sebagai jalur cadangan yang tidak dapat dihapus (CK-A-07, mengamandemen CK-09).

| Tahap | Yang terjadi |
|---|---|
| **Finalisasi** oleh Wali Kelas | Satu transaksi: memeriksa kelengkapan seluruh mata pelajaran (I-20, AC-07), menulis baris `rapor_mapel` beserta `snapshot_komponen` sebagai salinan beku, lalu mengubah status menjadi `finalized`. **Sesudah `COMMIT`**, seluruh berkas rapor kelas dirender dan diunggah ke S3 di dalam request yang sama, dengan **anggaran lunak 20 detik** |
| **Unduh per siswa** oleh Administrator, Wali Kelas, atau Siswa | Berkas yang sudah ada dipakai apa adanya. Yang belum ada dirender saat itu juga dari salinan beku, lalu disimpan. Dikembalikan sebagai presigned URL |
| **Unduh sekelas** oleh Administrator atau Wali Kelas | Berkas yang sudah ada disusun menjadi satu arsip ZIP — **murni pekerjaan I/O, tanpa render**. Arsip tidak pernah dipakai ulang dan dihapus aturan daur hidup S3 |

Karena PDF selalu dirender dari salinan beku, keluarannya selalu sama dengan data yang difinalisasi, sehingga AC-13 terpenuhi meskipun templat bobot berubah kemudian.

**Render tidak pernah berada di dalam transaksi**, dan kegagalannya tidak pernah membatalkan finalisasi. Finalisasi yang sudah `COMMIT` bersifat sah dengan sendirinya; berkas hanyalah turunannya. Berkas yang tidak sempat dirender dalam anggaran 20 detik dilaporkan apa adanya lewat `berkas_terender` dan diselesaikan jalur unduh ([API.md §8.3](API.md)).

**Tidak ada pekerjaan latar yang ditambahkan.** Seluruh render berjalan di dalam request yang memicunya, sehingga CK-07 tetap berlaku penuh: tidak ada antrean, tidak ada worker, dan tidak ada progres yang perlu dipantau. Yang membuat ini mungkin adalah anggaran lunak — bukan keyakinan bahwa tiga puluh render pasti selesai tepat waktu.

### 11.1 Penyajian lewat presigned URL

Bucket rapor bersifat **privat** dan tidak dapat dihubungi siapa pun secara langsung. Ketika pengguna menekan unduh, backend memeriksa kewenangannya lebih dahulu (Pasal 9.2), lalu meminta S3 menerbitkan tautan bertanda tangan berumur pendek. Browser mengunduh langsung dari S3, sehingga isi berkas tidak pernah melewati proses aplikasi.

**Umur presigned URL: 5 menit.** Cukup bagi browser untuk memulai unduhan, terlalu pendek untuk disalin dan disebarkan kepada orang yang tidak berhak. Tautan yang sudah kedaluwarsa tidak dapat dipakai ulang, dan pengguna cukup menekan unduh sekali lagi.

### 11.2 Kesegaran berkas terhadap koreksi Administrator

P14 memberi Administrator kewenangan mengubah data yang sudah final. Tanpa penanganan khusus, berkas hasil render yang tersimpan di S3 akan **tetap dipakai ulang** meskipun data sumbernya sudah berubah — siswa mengunduh PDF lama, sementara aplikasi menampilkan angka baru. Keadaan ini melanggar AC-13 tanpa memunculkan kesalahan apa pun.

**Karena itu, setiap perubahan Administrator terhadap data rapor yang sudah final menghapus berkas terkait dari S3 di dalam transaksi yang sama.** Unduhan berikutnya tidak menemukan berkas, sehingga merender ulang dari salinan beku yang sudah diperbarui (CK-A-05).

Yang tetap berada di luar jangkauan sistem: berkas yang **sudah terlanjur diunduh** pengguna sebelum koreksi. Salinan tersebut berada di perangkat masing-masing dan tidak dapat ditarik kembali. Hal ini berkaitan dengan temuan **T-04** pada [RFC-001 §10](RFC-001-model-data-konseptual.md) serta butir 2 pada [aktor-role.md §12](aktor-role.md), yang keduanya bersifat produk: apakah pihak sekolah perlu diberi tahu ketika koreksi terjadi setelah distribusi.

### 11.3 Isi berkas rapor

Ditetapkan **V5** pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md), dijawab 11 Agustus 2026 dengan merujuk rapor resmi yang dipakai sekolah. Susunannya tiga bagian, berurutan dari atas:

| Bagian | Isi | Sumber |
|---|---|---|
| **Kepala** | Periode akademik · Nama · NIS · Kelas · Wali Kelas | `rapor.periode_ref`, `pengguna.nama`, `pengguna.nama_pengguna`, `kelas.nama`, `kelas.wali_kelas_ref` |
| **Tabel** | No · Mata Pelajaran · KKM · Nilai Akhir · Kehadiran | `rapor_mapel` seluruhnya |
| **Kaki** | Catatan Wali Kelas | `rapor.catatan_wali` |

**Seluruh bidangnya sudah ada.** Tidak ada satu pun kolom, tabel, maupun endpoint yang perlu ditambahkan untuk memenuhi bentuk ini — V5 dijawab tanpa menyentuh [SCHEMA.md](SCHEMA.md) maupun [API.md](API.md). `No` adalah nomor urut baris, bukan data yang disimpan.

**Rincian komponen tidak dicetak.** Tabelnya berhenti pada nilai akhir per mata pelajaran; kode, nama, bobot, dan nilai tiap komponen tidak muncul di berkas. Meskipun begitu `rapor_mapel.snapshot_komponen` **tetap dibekukan pada saat finalisasi** — ia yang membuat nilai akhir dapat dipertanggungjawabkan kembali ketika orang tua bertanya dari mana angkanya berasal, dan tanpanya AC-13 kehilangan dasarnya. Yang berubah hanya apa yang tercetak, bukan apa yang disimpan.

**Empat hal yang lazim ada pada rapor SMA sengaja tidak dimuat**, karena tidak satu pun memiliki tempat pada model data dan menambahkannya berarti mengamandemen [RFC-001](RFC-001-model-data-konseptual.md) beserta PRD:

| Tidak dimuat | Sebab |
|---|---|
| Deskripsi capaian naratif per mata pelajaran | Sistem hanya menyimpan angka. Menambahkannya berarti entitas baru beserta layar penulisannya |
| Predikat huruf | Tidak ada tabel konversi angka ke huruf yang ditetapkan dokumen mana pun |
| Ekstrakurikuler | Tidak ada entitasnya. Berada di luar cakupan MVP bersama NG8 |
| Rekap ketidakhadiran dalam satuan hari | Presensi tercatat per sesi per mata pelajaran (I-15). Satu hari memuat beberapa mata pelajaran, sehingga rekap harian tidak dapat diturunkan tanpa menetapkan lebih dahulu apa artinya "tidak hadir sehari" |

Identitas sekolah — nama, NPSN, alamat, logo — juga belum dimuat, dengan sebab yang berbeda: ia bukan data siswa melainkan tetapan pemasangan, dan tempatnya belum ditetapkan dokumen mana pun. Dicatat sebagai temuan **A-09** pada [API.md §13.1](API.md).

---

## 12. Jaringan dan keamanan

| Lapisan | Ketentuan |
|---|---|
| **Pintu masuk** | CloudFront adalah satu-satunya alamat yang dapat dihubungi publik |
| **Function URL** | Auth type `AWS_IAM`. Hanya menerima request bertanda tangan SigV4 dari distribusi CloudFront yang ditunjuk lewat Origin Access Control |
| **Fungsi Lambda** | Di dalam VPC, subnet privat, tanpa alamat IP publik |
| **RDS** | Subnet privat-data, security group hanya mengizinkan security group fungsi Lambda |
| **S3 frontend** | Bucket privat, hanya dapat dibaca CloudFront lewat Origin Access Control |
| **S3 rapor** | Bucket privat, akses hanya lewat presigned URL berumur 5 menit (Pasal 11.1) |
| **Jalur keluar** | NAT instance `t4g.nano` untuk menghubungi Elice AI Cloud. S3 lewat gateway endpoint yang tidak berbiaya |
| **Header keamanan** | HSTS, `X-Content-Type-Options`, `Referrer-Policy`, dan Content Security Policy diatur di CloudFront Response Headers Policy |
| **Validasi masukan** | Zod di batas HTTP maupun batas berkas unggahan. Tidak ada data luar yang masuk ke domain tanpa melewatinya |

### 12.1 Cara rahasia dibaca

Keempat rahasia infrastruktur beserta tempat penyimpanannya ditetapkan [Techstack.md §7](Techstack.md) dan tidak diulang di sini. Yang menjadi urusan dokumen ini adalah **jalur pembacaannya**.

Pembacaan melewati interface `Secrets` di `ports/`, dengan dua implementasi: `adapters/aws/` membaca dari Secrets Manager dan SSM Parameter Store, sedangkan `adapters/local/` membaca dari variabel lingkungan. Perbedaan antara AWS dan on-prem dengan demikian tidak pernah menyentuh kode aplikasi.

**Rahasia dibaca sekali pada saat container menyala**, lalu disimpan di memori selama container hidup — bukan pada setiap request. Pada Lambda, pembacaan terjadi pada saat cold start, sehingga request berikutnya di instance yang sama tidak memanggil Secrets Manager sama sekali. Nilai rahasia tidak pernah dicetak ke log, baik log aplikasi maupun log CI.

### 12.2 Dua peringatan

> ⚠️ Sertifikat ACM untuk CloudFront **wajib diterbitkan di `us-east-1`**, sedangkan seluruh sumber daya lain berada di `ap-southeast-3` (CK-16). Ditangani dengan provider alias kedua pada Terraform. Kelalaian pada butir ini menggagalkan `terraform apply`.
>
> ⚠️ **Wajib dibuktikan pada hari pertama infrastruktur naik:** perilaku penandatanganan Origin Access Control terhadap request **ber-body** — `POST` dan `PATCH` seperti Simpan Nilai. Kombinasi OAC dengan Function URL memiliki ketentuan tersendiri mengenai penyertaan body dalam tanda tangan SigV4. Diuji lewat request sungguhan sejak API masih berupa stub, bukan ditemukan ketika frontend mulai menyimpan nilai. Prosedurnya ditetapkan [DEPLOYMENT.md](DEPLOYMENT.md) pasal 5.

---

## 13. Portabilitas ke on-prem

| Lapisan | Portabel? | Yang berubah ketika dipasang di server sekolah |
|---|---|---|
| Image aplikasi | ✅ | **Tidak ada.** Image yang sama dijalankan `docker run`. Binary Lambda Web Adapter di dalamnya menganggur karena `AWS_LAMBDA_EXEC_WRAPPER` tidak ada (Pasal 6) |
| Rute, validasi, middleware, autentikasi | ✅ | Tidak disentuh |
| Logika domain | ✅ | Tidak disentuh |
| Drizzle, skema, dan migrasi | ✅ | Tidak disentuh |
| Pembatasan laju | ✅ | Tidak disentuh — berada di dalam aplikasi (Pasal 7) |
| Dua role basis data | ✅ | Tidak disentuh — `app_rw` dan `app_ro` dibuat skrip pemasangan |
| Basis data | ✅ | RDS PostgreSQL adalah PostgreSQL biasa. `pg_dump` lalu `pg_restore`; yang berubah hanya `DATABASE_URL` |
| Frontend | ✅ | Berkas statis yang sama disajikan Caddy, atau langsung oleh Express |
| Penyimpanan berkas | ⚠️ | Tetap S3, atau ditukar ke MinIO maupun disk lokal lewat `adapters/local` |
| Rahasia | ⚠️ | Secrets Manager dan SSM ditukar berkas `.env` berizin `600` (Pasal 12.1) |
| Penyedia AI | ⚠️ | Elice AI Cloud tetap dipakai, atau diarahkan ke penyedia lain maupun model yang dipasang sendiri. Karena antarmukanya setara OpenAI, yang berubah hanya base URL dan kunci API (Pasal 10.2) |
| Connection pool | ⚠️ | `max: 1` menjadi `max: 10`. **Satu baris konfigurasi**, dibaca dari variabel lingkungan |
| CloudFront, Function URL, dan NAT | ❌ | Digantikan **Caddy** sebagai reverse proxy tunggal. Pembagian path `/*` dan `/api/*` pada Pasal 3 berpindah ke sana |
| Jalan masuk dari internet | ❌ | Digantikan **Cloudflare Tunnel**. `cloudflared` berjalan sebagai service kedua pada `docker-compose.yml` on-prem (CK-17) |

Yang berpindah bersama sistem adalah tanggung jawab operasional: pencadangan, pembaruan keamanan, enkripsi at-rest, dan risiko perangkat keras menjadi urusan pemilik server.

### 13.1 Jalan masuk on-prem

Server sekolah tidak memiliki alamat masuk yang dapat dihubungi dari internet, dan tidak boleh menuntut satu pun. **Cloudflare Tunnel** menyelesaikannya dengan membalik arah: `cloudflared` membuka koneksi **keluar** ke Cloudflare, lalu trafik masuk mengalir balik lewat koneksi itu.

```
Pengguna ──► Cloudflare
                  ╎
                  ╎ koneksi dibuka dari dalam ke luar
                  ╎ oleh server sekolah, bukan sebaliknya
   ┌──────────────╎───────────────────────────────────┐
   │ Server sekolah                                   │
   │              ▼                                   │
   │        cloudflared ──► Caddy ──┬──► /*    statis │
   │                                └──► /api/* :8080 │
   └──────────────────────────────────────────────────┘
```

Tiga akibatnya langsung. **Tidak ada port yang dibuka di router sekolah**, sehingga pemasangan tidak bergantung pada kerja sama admin jaringan dan tidak menambah permukaan serangan. **IP publik statis tidak diperlukan**, sehingga sambungan internet sekolah yang biasa sudah cukup. **TLS tidak perlu diurus**, yang penting karena cookie sesi `HttpOnly` pada Pasal 12 menuntut HTTPS sementara sekolah tidak memiliki staf untuk memperbarui sertifikat.

**Janji satu domain pada Pasal 3 tetap utuh.** Caddy menempati peran yang di AWS dipegang CloudFront — satu hostname, dua tujuan, dibagi berdasarkan path. Frontend tetap memanggil `/api/...` secara relatif, sehingga CORS tetap tidak ada dan tidak ada base URL yang berbeda antar lingkungan.

**Yang tidak ikut berpindah** adalah OAC dan `auth_type = AWS_IAM`. Keduanya mekanisme AWS, dan tidak memiliki padanan di sini. Perlindungan on-prem karenanya bertumpu pada tunnel sebagai satu-satunya jalan masuk dan pada pemeriksaan kewenangan di dalam aplikasi (Pasal 9), bukan pada lapisan infrastruktur.

**Tidak ada lapisan abstraksi yang dibangun khusus demi portabilitas.** Portabilitas berasal dari bentuk artefaknya — sebuah container — dan dari kenyataan bahwa tim menjalankan container yang sama setiap hari.

---

## 14. Alur request

Dua alur berikut ditelusuri langkah demi langkah karena keduanya menyentuh bagian yang paling mudah salah diterapkan: keatomikan transaksi pada yang pertama, dan pembatasan akses AI pada yang kedua.

### 14.1 Simpan Nilai sekelas

Satu penekanan tombol **Simpan Nilai** oleh Guru pada satu kelas, memuat seluruh perubahan nilai pada layar matriks — sampai 240 sel untuk 30 siswa dikali 8 komponen.

```
POST /api/penugasan/:id/nilai
body: [ { siswa_ref, komponen_ref, nilai }, ... ]     nilai boleh null
```

| # | Langkah | Kegagalan menghasilkan |
|:--:|---|---|
| 1 | CloudFront menandatangani request dengan SigV4 lewat Origin Access Control; Function URL memverifikasinya | `403` dari AWS, tidak mencapai aplikasi |
| 2 | Lambda Web Adapter menerjemahkan invocation menjadi HTTP ke `127.0.0.1:8080`; Express menerimanya sebagai request biasa | — |
| 3 | Middleware sesi membaca cookie, mencari baris sesi, memeriksa masa berlakunya, lalu memuat `pengguna` beserta perannya | `401` — sesi tidak ada, kedaluwarsa, atau sudah dicabut |
| 4 | Pembatas laju **dilewati**. Jalur ini tidak termasuk tiga jalur terbatas pada Pasal 7: ia tidak memanggil layanan berbayar, tidak menerima berkas, dan membatasinya justru menghalangi Guru yang sedang bekerja | — |
| 5 | Zod memvalidasi bentuk body: setiap baris memiliki pengenal yang sah, dan `nilai` bernilai 0–100 atau `null` | `400` menyebutkan baris dan bidang yang tidak sah |
| 6 | Lapis peran: peran wajib `guru` atau `administrator` | `403` |
| 7 | Lapis baris: bagi Guru, `penugasan.guru_ref` wajib sama dengan `pengguna.id`. Bagi Administrator, dilewati | `403` — bukan `404` (Pasal 9.2) |
| 8 | Pemeriksaan status rapor: apabila rapor kelas tersebut sudah `finalized` atau `distributed`, Guru ditolak; Administrator dilanjutkan | `409` bagi Guru — penegakan I-22 dan AC-14 |
| 9 | **Transaksi dibuka** pada koneksi `app_rw` | — |
| 10 | Untuk setiap baris: `nilai` terisi → sisip atau perbarui; `nilai` bernilai `null` → **hapus barisnya**. Ketiadaan baris adalah satu-satunya representasi nilai kosong (I-12) | Kesalahan mana pun membatalkan seluruh transaksi |
| 11 | `COMMIT` | `ROLLBACK` — **tidak satu baris pun tersimpan** |
| 12 | `200` beserta ringkasan jumlah baris tersimpan dan terhapus | Kode kegagalan beserta alasannya |
| 13 | Frontend menampilkan pemberitahuan berhasil atau gagal; kegagalan menyebutkan alasannya (P21, AC-27) | — |

**Langkah 9 sampai 11 adalah pemenuhan C-02.** Seluruh perubahan pada satu penekanan tombol berada dalam satu transaksi, sehingga kegagalan di baris ke-200 membatalkan 199 baris sebelumnya. Tidak ada keadaan "tersimpan separuh" (P22, AC-15).

**Yang tidak terjadi pada alur ini:**

- **Tidak ada penyimpanan otomatis.** Perubahan di layar bersifat lokal sampai langkah 1 dijalankan. Meninggalkan halaman tanpa menekan tombol tidak mengubah data (AC-15).
- **Tidak ada langkah publikasi.** Setelah `COMMIT`, nilai langsung terlihat siswa yang bersangkutan (UC-09).
- **Tidak ada penulisan `audit_log`** (CK-A-06).

### 14.2 Tombol Suggestion

Satu penekanan tombol **Suggestion** oleh Siswa pada halamannya sendiri.

```
POST /api/saya/suggestion
body: kosong
```

| # | Langkah | Kegagalan menghasilkan |
|:--:|---|---|
| 1 | Sama seperti langkah 1–3 pada §14.1: tanda tangan OAC, adapter, dan middleware sesi | `403` atau `401` |
| 2 | Lapis peran: peran wajib **`siswa`**. Administrator, Guru, dan Wali Kelas ditolak — tombol ini hanya tersedia bagi Siswa ([aktor-role.md §6](aktor-role.md)) | `403` |
| 3 | Pembatas laju: paling banyak 5 penekanan per jam per siswa (Pasal 7) | `429` beserta waktu percobaan berikutnya |
| 4 | Baca data lewat koneksi **`app_ro`**: nilai per komponen, topik, KKM, dan presensi seluruh mata pelajaran siswa tersebut pada semester berjalan, dengan klausa `WHERE` pada `siswa_ref = pengguna.id` | `500` — kegagalan basis data, bukan kegagalan lunak |
| 5 | `domain/` menghitung kelengkapan per mata pelajaran dan persentase kehadiran. **Bukan AI yang menghitung** (PRD §8.6 butir 1). Nilai akhir tidak disertakan bila datanya belum lengkap (PRD §8.6 butir 5) | — |
| 6 | Prompt disusun **tanpa nama, tanpa NIS, tanpa pengenal apa pun** (Pasal 10.1) | — |
| 7 | `AiAdvisor.suggest()` memanggil Elice AI Cloud dengan batas waktu 20 detik | Lanjut ke langkah 9 |
| 8 | Teks keluaran dikembalikan `200` apa adanya. **Tidak ditulis ke tabel mana pun, tidak di-cache, dan isinya tidak dicatat ke log** (I-24, NG14, AC-16) | — |
| 9 | **Kegagalan lunak.** Batas waktu terlampaui, layanan menjawab `5xx`, atau kredit habis → `503` beserta pesan bahwa rekomendasi tidak dapat dibuat saat ini | — |
| 10 | Halaman menampilkan pesan kegagalan pada area rekomendasi saja. Nilai, presensi, finalisasi, dan distribusi **tetap berjalan** (AC-21) | — |

**Pembatasan akses berlapis ganda pada langkah 4.** Klausa `WHERE` membatasi data yang dibaca kepada siswa penekan tombol, dan koneksi `app_ro` memastikan jalur ini tidak dapat menulis apa pun sekalipun kodenya keliru. Yang pertama dapat rusak karena kekeliruan kode; yang kedua tidak. Inilah yang membuat AC-17 dan AC-20 dapat dibuktikan, bukan sekadar dinyatakan.

**Langkah 9 adalah pembeda terpenting alur ini dari alur §14.1.** Kegagalan pada Simpan Nilai adalah kegagalan sungguhan yang membatalkan pekerjaan pengguna. Kegagalan pada Suggestion tidak boleh menghambat apa pun — habisnya kredit KADA tidak menghentikan sekolah bekerja.

---

## Lampiran — Catatan Keputusan

Bernomor dan bertanggal. Entri tidak disunting; perubahan keputusan ditulis sebagai entri baru yang menyebut nomor yang digantikannya.

Penomoran memakai awalan `CK-A-` sehingga tidak bertabrakan dengan `CK-xx` pada [Techstack.md](Techstack.md) maupun `CK-D-xx` pada [DEPLOYMENT.md](DEPLOYMENT.md).

### CK-A-01 · 6 Agustus 2026 · Kewenangan Wali Kelas diturunkan dari data, bukan dari peran

**Diputuskan.** Tidak ada nilai `pengguna.peran` bernama `wali_kelas`. Kewenangan Wali Kelas diperoleh sepenuhnya dari lapis baris, yaitu `kelas.wali_kelas_ref` yang menunjuk pengguna tersebut.

**Alasan.** [aktor-role.md §1](aktor-role.md) menyatakan Wali Kelas adalah kewenangan tambahan pada akun Guru, bukan jenis akun keempat. Menurunkannya dari data membuat pernyataan itu benar secara struktural: seorang Guru memperoleh dan kehilangan kewenangan Wali Kelas semata-mata melalui penetapan Administrator pada kelas, tanpa langkah kedua yang dapat terlupa.

**Alternatif yang ditolak.** *Menambah nilai peran `wali_kelas`.* Menghasilkan dua sumber kebenaran yang dapat menyimpang — seorang Guru dapat berperan `wali_kelas` tanpa satu pun kelas menunjuk dirinya, atau sebaliknya. Selain itu memaksa pertanyaan yang tidak memiliki jawaban: apa peran seseorang yang mengajar tiga kelas dan menjadi wali pada satu di antaranya.

**Konsekuensi yang diterima.** Setiap pemeriksaan kewenangan Wali Kelas memerlukan pembacaan `kelas`, bukan cukup membaca kolom peran. Pada volume RFC-001 §8.1 — 12 kelas — biaya kuerinya tidak berarti.

### CK-A-02 · 6 Agustus 2026 · Adapter AI dinamai menurut protokol, bukan penyedia

**Diputuskan.** Implementasi `AiAdvisor` berada di `adapters/openai-compatible/`, menggantikan `adapters/openrouter/` pada rancangan Techstack versi 1.

**Alasan.** Penyedia sudah berganti satu kali melalui CK-14, dari OpenRouter ke Elice AI Cloud, tanpa mengubah satu baris pun isi adapter — karena yang menentukan bukan penyedianya, melainkan protokolnya. Nama direktori yang mengikuti penyedia menjanjikan hal yang tidak benar, dan memaksa penggantian nama pada pergantian berikutnya.

**Alternatif yang ditolak.** *`adapters/elice/`.* Konsisten dengan CK-14 pada hari ini, tetapi mengulang persoalan yang sama pada pergantian penyedia berikutnya, termasuk perpindahan ke model yang dipasang sendiri di server sekolah.

### CK-A-03 · 6 Agustus 2026 · Penghitung pembatas laju disimpan di PostgreSQL

**Diputuskan.** Penghitung `express-rate-limit` disimpan di PostgreSQL, bukan di memori proses.

**Alasan.** Setiap instance Lambda memiliki memorinya sendiri dan dapat menyala kapan saja. Penghitung di memori berarti batas berlaku per instance, sehingga percobaan kata sandi beruntun cukup menunggu instance baru untuk memperoleh jatah baru — pembatasan yang terlihat ada tetapi tidak menahan apa pun.

**Alternatif yang ditolak.** *Redis atau ElastiCache.* Bentuk yang lazim dan cepat, tetapi menambah sumber daya yang harus dinaikkan, dibayar, dan dipelihara, sekaligus menambah bagian yang tidak ikut berpindah ke server sekolah. PostgreSQL sudah ada, sudah di jalur yang sama, dan volume tulisnya kecil.

**Konsekuensi yang diterima.** Setiap request pada jalur terbatas menambah satu operasi tulis ringan ke basis data. Pada volume RFC-001 §8.1, ini tidak berarti.

### CK-A-04 · 6 Agustus 2026 · Sesi berupa baris di basis data yang dapat dicabut

**Diputuskan.** Token sesi berupa nilai acak buram yang memiliki barisnya sendiri di basis data, berumur 12 jam tanpa perpanjangan otomatis. Bukan JWT.

**Alasan.** PRD §6.1.3 menempatkan penggantian kata sandi sepenuhnya pada Administrator, dan itulah satu-satunya jalur pemulihan yang tersedia ketika sebuah akun diduga disalahgunakan. Agar jalur itu bermakna, penggantian kata sandi harus **memutus sesi yang sedang berjalan seketika**. Dengan sesi berupa baris, pencabutan berarti menghapus baris.

**Alternatif yang ditolak.** *JWT.* Menghapus kebutuhan pembacaan basis data per request, tetapi tidak dapat dicabut sebelum kedaluwarsa. Menambahkan daftar cabut untuk mengatasinya mengembalikan pembacaan basis data yang tadinya dihindari, sehingga keuntungannya habis sementara kerumitannya bertambah.

**Konsekuensi yang diterima.** Satu pembacaan basis data pada setiap request. Pembacaan ini digabungkan dengan pemuatan `pengguna` beserta perannya, sehingga menjadi satu kueri, bukan dua.

### CK-A-05 · 6 Agustus 2026 · Berkas rapor dihapus ketika Administrator mengoreksi data final

**Diputuskan.** Setiap perubahan Administrator terhadap data rapor yang sudah `finalized` atau `distributed` menghapus berkas PDF terkait dari S3, di dalam transaksi yang sama. Unduhan berikutnya merender ulang.

**Alasan.** CK-09 menetapkan berkas hasil render dipakai ulang, sedangkan P14 memberi Administrator kewenangan mengubah data final. Keduanya digabungkan tanpa penanganan menghasilkan berkas usang yang tetap disajikan selamanya: aplikasi menampilkan angka yang benar, sementara PDF yang diunduh siswa memuat angka lama. Ini melanggar AC-13 tanpa memunculkan kesalahan apa pun, sehingga tidak akan tertangkap pengujian yang hanya memeriksa keberhasilan operasi.

**Alternatif yang ditolak.** *Merender ulang seketika saat koreksi.* Memindahkan pekerjaan render ke dalam transaksi tulis, sehingga koreksi satu angka menunggu render satu PDF. Penghapusan memberi hasil yang sama dengan satu perintah, dan biaya render tetap dibayar pada saat unduhan seperti sedianya.

**Konsekuensi yang diterima.** Berkas yang **sudah terlanjur diunduh** pengguna sebelum koreksi tidak dapat ditarik kembali. Apakah pihak sekolah perlu diberi tahu ketika koreksi terjadi setelah distribusi adalah persoalan produk, tercatat sebagai T-04 pada [RFC-001 §10](RFC-001-model-data-konseptual.md) dan butir 2 pada [aktor-role.md §12](aktor-role.md).

### CK-A-06 · 6 Agustus 2026 · `audit_log` tidak ditulis pada MVP

**Diputuskan.** Tidak ada alur request yang menulis ke `audit_log`. Tabelnya tetap ada pada model data, tetapi kosong sepanjang MVP.

**Alasan.** Tidak ada aktor yang memiliki kemampuan melihat riwayat aktivitas. [aktor-role.md §6](aktor-role.md) tidak memuat kemampuan tersebut bagi peran mana pun, dan [ATURAN-DAN-KRITERIA §3](ATURAN-DAN-KRITERIA.md) tidak memuat layar yang menampilkannya. Kekurangan ini sudah diterima secara sadar oleh PRD. Menulis jejak yang tidak dapat dibaca siapa pun berarti membayar operasi tulis pada setiap mutasi untuk data yang tidak memiliki pembaca.

**Alternatif yang ditolak.** *Menulis `audit_log` untuk tindakan Administrator saja.* Bentuk paling murah yang tetap memberi jejak, dan sempat dipertimbangkan karena P14 memberi Administrator kewenangan mengubah data final. Ditolak karena tetap tidak memiliki pembaca, sehingga nilainya baru muncul ketika layar penelusuran dibangun — dan pada saat itu ketentuannya perlu ditetapkan ulang bersama layarnya.

**Konsekuensi yang diterima.** Tidak tersedia penelusuran administratif atas perubahan nilai maupun presensi selama MVP. Ini sejalan dengan P7, yang sudah meniadakan riwayat koreksi presensi, dan dengan butir 3 pada [aktor-role.md §12](aktor-role.md) yang mencatat bahwa MVP tidak mencatat riwayat perubahan.

`audit_log` sendiri masih dipertahankan pada [RFC-001 D-07](RFC-001-model-data-konseptual.md). Keputusan ini **tidak menggugurkan entitas tersebut dari model data** — pencabutannya, apabila kelak dikehendaki, dilakukan melalui amandemen RFC-001, bukan melalui dokumen ini.

### CK-A-07 · 6 Agustus 2026 · Berkas rapor dirender pada saat finalisasi — mengamandemen CK-09

**Diputuskan.** Finalisasi merender seluruh berkas rapor kelas sesudah `COMMIT`, di dalam request yang sama, dengan anggaran lunak 20 detik. Jalur render-saat-unduh tetap ada. Ditambahkan unduh sekelas berbentuk arsip ZIP yang tidak merender apa pun.

**Yang berubah dari CK-09.** Entri tersebut menolak pembuatan berkas sekelas pada saat finalisasi karena "menuntut pemrosesan latar dan tampilan progres". Keberatan itu mengandaikan tiga puluh render tidak muat di dalam satu request. Dengan batas waktu fungsi 30 detik (Pasal 6) dan anggaran lunak 20 detik, render yang tidak selesai **dihentikan alih-alih dipindahkan ke latar**, sehingga pemrosesan latar tidak pernah diperlukan. CK-07 tetap berlaku penuh.

Keberatan kedua CK-09 — tidak ada manfaat produk karena rapor tidak selalu diunduh seluruhnya — juga gugur, tetapi bukan karena kecepatan unduhan. Manfaatnya adalah **terbukanya unduh sekelas**: dengan berkas sudah tersedia, penyusunan arsip menjadi pekerjaan I/O yang muat di dalam anggaran request. Tanpa pra-render, endpoint yang sama berarti tiga puluh render dalam satu permintaan, yaitu persis keadaan yang ditolak CK-09.

**Yang tidak berubah.** CK-A-05 tetap berlaku dan justru menjadi lebih penting: koreksi Administrator atas data final menghapus berkas terkait, dan unduhan berikutnya merender ulang. Karena itu jalur render-saat-unduh **tidak pernah dapat dihapus**, dan pra-render selalu bersifat tambahan.

**Alternatif yang ditolak.** *Mempertahankan CK-09 apa adanya.* Menghindari perubahan pada keputusan yang sudah terkunci. Ditolak karena konsekuensi yang diterimanya — Wali Kelas menekan unduh tiga puluh kali — dapat dinilai sekarang tanpa perlu menunggu UAT. *Merender di dalam transaksi finalisasi.* Menjamin berkas selalu lengkap, tetapi menahan transaksi selama belasan detik pada basis data yang sedang melayani seluruh sekolah.

**Konsekuensi yang diterima.** Request finalisasi menjadi belasan detik, sekali per kelas per semester. Perkiraan lama render **belum diukur**, dan kewajiban pengukurannya tercatat pada [API.md §13.3](API.md). Kegagalan perkiraan itu tidak berakibat apa pun selain berkas yang dirender belakangan.

---

### CK-A-08 · 8 Agustus 2026 · Pembatasan masuk dua lapis dengan ambang berbeda

**Diputuskan.** `POST /api/auth/masuk` dibatasi dua lapis sekaligus: **5 kegagalan per 15 menit per akun**, dan **30 kegagalan per 15 menit per alamat IP**. Keduanya diperiksa; salah satu terlampaui berarti ditolak `429`.

**Alasan.** Satu lapis saja bocor ke salah satu arah, dan keduanya nyata.

*Per akun saja* menahan serangan terhadap satu akun, tetapi tidak menahan **password spraying**: penyerang mencoba satu kata sandi umum terhadap ratusan akun, dan setiap akun menyumbang jatah lima kegagalannya sendiri tanpa plafon bersama. Pengenal masuk di sini adalah NIP dan NIS yang berpola dan mudah ditebak, sehingga daftar sasarannya tidak perlu dicuri lebih dahulu.

*Per IP dengan ambang yang sama* menahan spraying tetapi mengunci sekolah. Pasal 7 sudah mencatat satu sekolah kerap berbagi satu alamat IP publik; pada ambang lima, lima kesalahan ketik dari lima orang berbeda memutus akses seluruh sekolah selama lima belas menit. Itu penolakan layanan terhadap penggunanya sendiri.

**Angka 30 dipilih dari kedua sisi.** Dari sisi pengguna: pada 379 akun, tiga puluh kegagalan dalam seperempat jam dari satu gedung jauh di atas laju kesalahan ketik yang wajar, termasuk pada pagi pertama pemakaian. Dari sisi penyerang: tiga puluh percobaan per seperempat jam menjadikan penyisiran kata sandi umum atas ratusan akun memakan waktu berhari-hari, bukan menit — dan sepanjang itu terlihat pada penghitung.

**Alamat IP diambil dari entri TERAKHIR `X-Forwarded-For`, bukan yang pertama.** CloudFront dan Caddy sama-sama **menambahkan** alamat yang mereka lihat di ujung daftar, sehingga entri terakhir berasal dari proksi tepercaya dan tidak dapat dipalsukan klien. Entri pertama justru sepenuhnya dikuasai klien; memakainya berarti pembatas laju yang dapat dilewati hanya dengan mengarang satu header, yaitu keadaan yang lebih buruk daripada tidak ada pembatas sama sekali karena ia tampak melindungi.

**Alternatif yang ditolak.** *Kunci gabungan `(akun, IP)`.* Tidak mengunci sekolah dan sederhana, tetapi melemahkan batas per akun: penyerang cukup berpindah alamat untuk memperoleh lima percobaan baru atas akun yang sama. *`app.set("trust proxy", n)`.* Bergantung pada jumlah lompatan yang berbeda antara AWS dan on-prem; salah menghitungnya menghasilkan pembacaan alamat yang keliru tanpa gejala apa pun.

**Konsekuensi yang diterima.** Satu sekolah yang benar-benar mengalami tiga puluh kegagalan dalam seperempat jam akan tertahan bersama-sama. Bila itu terjadi pada pemakaian sungguhan, angkanya dinaikkan lewat amandemen catatan ini — bukan lewat penyuntingan diam-diam pada kode.

---

### CK-A-09 · 8 Agustus 2026 · Dua perintah akun Administrator; kata sandi bootstrap lewat stdin

**Diputuskan.** Terdapat dua perintah. `seed:admin` membuat Administrator pertama dengan kata sandi yang **ditentukan operator** dan dibaca dari **stdin**; `admin:create` membangkitkan kata sandi acak lalu mencetaknya sekali, sebagaimana ditetapkan §9.3 semula.

**Alasan adanya dua.** Keduanya melayani keadaan yang berbeda. Pada bootstrap, belum ada seorang pun yang dapat masuk, dan operator yang memasang sistem adalah orang yang sama yang akan memegang akun itu — sehingga membangkitkan kata sandi lalu mencetaknya hanya menambah satu nilai yang perlu disalin. Pada reset sesudahnya, Administrator menerima kata sandi dari pihak yang tidak boleh mengetahuinya, dan di sana pembangkitan sistem beserta pencetakan sekali memang bentuk yang benar (P17).

**Alasan lewat stdin, bukan argumen.** Argumen proses terbaca pengguna lain pada mesin yang sama lewat `/proc/<pid>/cmdline`, dan tersimpan pada riwayat shell. Pada server sekolah yang dipakai bersama, itu kebocoran yang bentuknya sama persis dengan kebocoran CloudWatch yang justru sedang ditutup §9.3 — hanya pembacanya yang berbeda. Stdin tidak muncul pada keduanya.

Bila stdin berupa terminal, perintah meminta kata sandi tanpa menampilkan ketikannya. Bila stdin berupa pipa, kata sandi dibaca apa adanya dengan satu akhir baris di ujung dibuang — sehingga `printf` maupun `echo` sama-sama bekerja.

**Alternatif yang ditolak.** *Kata sandi sebagai argumen.* Paling ringkas diketik, dan itu satu-satunya kelebihannya. *Variabel lingkungan.* Lebih baik daripada argumen karena `/proc/<pid>/environ` hanya terbaca pemilik proses dan root, tetapi tetap tertinggal pada riwayat shell bila ditulis sebaris dengan perintahnya, dan tetap terwarisi seluruh proses anak. *Satu perintah dengan bendera pilihan.* Menggabungkan dua keadaan yang aturan keamanannya berbeda ke dalam satu jalur, sehingga bendera yang salah ketik menghasilkan perilaku yang salah tanpa terlihat.

**Konsekuensi yang diterima.** Kata sandi bootstrap tidak memiliki syarat kerumitan, sama seperti seluruh kata sandi pada sistem ini (PRD §6.1.3), sehingga operator dapat memilih kata sandi lemah. Pilihan itu ada di tangan orang yang sama yang memegang kredensial basis data pada saat itu, sehingga tidak menambah kewenangan siapa pun.

### CK-A-10 · 11 Agustus 2026 · Berkas rapor memuat nilai akhir per mata pelajaran, tanpa rincian komponen

**Diputuskan.** Berkas rapor tersusun tiga bagian sesuai §11.3: kepala berisi periode akademik beserta identitas siswa dan wali kelas, satu tabel berkolom **No, Mata Pelajaran, KKM, Nilai Akhir, Kehadiran**, dan kaki berisi catatan wali kelas. Rincian komponen penilaian tidak dicetak.

**Alasan.** Bentuk ini diambil dari rapor resmi yang sudah dipakai sekolah, sehingga menjawab V5 tanpa menegosiasikan ulang apa pun. Yang menentukan bagi keputusan ini: seluruh bidangnya **sudah tersedia** pada model data, sehingga V5 tidak lagi menghalangi A7 dan tidak menimbulkan satu pun amandemen pada SCHEMA maupun API.

Rincian komponen ditinggalkan karena rapor adalah dokumen ringkas yang dibaca orang tua, sedangkan rincian per komponen sudah tersedia sepanjang semester lewat layar nilai yang dapat dibuka siswa kapan saja (AC-06). Mencetaknya dua kali menambah halaman tanpa menambah informasi yang belum dapat dilihat.

**Yang tidak berubah.** `rapor_mapel.snapshot_komponen` tetap ditulis pada saat finalisasi. Ia bukan bahan cetak melainkan dasar pertanggungjawaban angka, dan CK-A-07 beserta AC-13 bersandar padanya. Menghapusnya karena tidak tercetak akan menukar jaminan dengan penghematan satu kolom.

**Alternatif yang ditolak.** *Mencetak rincian komponen seperti rancangan sementara sebelum V5 turun.* Ditolak sesudah dibandingkan dengan rapor sekolah yang sesungguhnya. *Menambahkan deskripsi naratif, predikat huruf, ekstrakurikuler, dan rekap ketidakhadiran harian agar setara rapor resmi selengkapnya.* Ditolak pada tahap ini karena keempatnya menuntut entitas baru; sebabnya masing-masing pada §11.3.

**Konsekuensi yang diterima.** Templat pdfmake pada `adapters/local/rapor-berkas/templat.ts` ditulis ulang mengikuti §11.3. Perubahannya terbatas pada satu berkas, sebagaimana memang dirancang.

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 11 Agustus 2026 | §10 — alamat tombol Suggestion dikoreksi dari `/api/me/suggestion` menjadi **`/api/saya/suggestion`**, sesuai [API.md §9.1](API.md) dan konvensi alamat berbahasa Indonesia pada AGENTS §5.1. `/api/saya/nilai` sudah memakai bentuk itu sejak A6 |
| 11 Agustus 2026 | **§11.3 dan CK-A-10** — isi berkas rapor ditetapkan menjawab V5: kepala, satu tabel No/Mata Pelajaran/KKM/Nilai Akhir/Kehadiran, dan catatan wali kelas. Seluruh bidangnya sudah ada pada model data, sehingga tidak ada amandemen SCHEMA maupun API. Empat hal yang lazim ada pada rapor SMA dicatat sebagai sengaja tidak dimuat beserta sebabnya |
| 6 Agustus 2026 | Kerangka dibuat sebagai bagian dari pemecahan `Techstack.md` menjadi tiga dokumen. Isi belum ditulis. Menggantikan `ARCHITECTURE.md` versi 2 Agustus 2026, yang diturunkan menjadi arsip dengan nama `ARCHITECTURE-2026-08-02.md` |
| 6 Agustus 2026 | **Versi 1.0 — isi ditulis.** Pasal 1 sampai 13 memindahkan isi yang sudah tervalidasi pada `Techstack.md` versi 1, dengan empat penyesuaian terhadap keadaan terbaru: adapter AI mengikuti CK-14, penyimpanan rahasia mengikuti `Techstack.md` §7, alamat endpoint Elice mengikuti `Techstack.md` §6, dan susunan jaringan mengikuti CK-13. Pasal 9 diperkaya dengan penerjemahan matriks kewenangan `aktor-role.md` menjadi dua lapis pemeriksaan. **Pasal 14 Alur request ditulis baru.** Ditetapkan pula lima angka yang sebelumnya belum pernah ditentukan: umur sesi 12 jam, umur presigned URL 5 menit, tiga batas laju, dan batas ukuran unggahan 2 MB. Lampiran Catatan Keputusan dibuka dengan **CK-A-01** sampai **CK-A-06** |
| 6 Agustus 2026 | Pasal 11 ditulis ulang: berkas rapor dirender pada saat finalisasi dengan anggaran lunak 20 detik, render-saat-unduh menjadi jalur cadangan yang tidak dapat dihapus, dan ditambahkan unduh sekelas berbentuk arsip ZIP yang tidak merender apa pun (**CK-A-07**, mengamandemen CK-09). Batas waktu fungsi pada Pasal 6 disesuaikan: request terpanjang kini finalisasi sekelas, bukan render satu PDF |
| 7 Agustus 2026 | Region pada Pasal 2 dan peringatan ACM pada §12.2 disesuaikan menjadi `ap-southeast-3` mengikuti **CK-16** pada [Techstack.md](Techstack.md). Kewajiban ACM di `us-east-1` tidak berubah |
| 7 Agustus 2026 | Cuplikan Dockerfile pada Pasal 6 disesuaikan menjadi dua tahap, mengikuti bentuk yang terpasang dan terbukti jalan di repositori backend. Bentuk satu tahap sebelumnya mengandaikan `dist/` sudah dibangun di luar image. Salinan Lambda Web Adapter, `PORT`, `AWS_LWA_READINESS_CHECK_PATH`, dan `CMD` tidak berubah |
| 8 Agustus 2026 | **Pasal 13 memperoleh §13.1 Jalan masuk on-prem**, mengikuti **CK-17** pada [Techstack.md](Techstack.md). Reverse proxy on-prem ditetapkan **Caddy** menggantikan penyebutan "Nginx atau Caddy" yang belum memilih, dan jalan masuk dari internet ditetapkan **Cloudflare Tunnel** sebagai baris tersendiri pada tabel portabilitas. Dicatat pula bahwa OAC dan `auth_type = AWS_IAM` tidak memiliki padanan on-prem, sehingga perlindungan bertumpu pada tunnel dan pemeriksaan kewenangan di dalam aplikasi. Susunan AWS pada Pasal 2, 3, dan 7 tidak berubah |
| 8 Agustus 2026 | **Pasal 7 dan §9.3 disesuaikan setelah tinjauan keamanan tahap A4.** Pembatasan masuk ditetapkan **dua lapis** dengan ambang berbeda (**CK-A-08**), karena satu lapis bocor ke salah satu arah: per akun saja tidak menahan password spraying, sedangkan per IP berambang sama mengunci seluruh sekolah yang berbagi satu alamat. §9.3 dikoreksi: jaminan "tidak dapat ditampilkan ulang" **tidak berlaku** di AWS karena `stdout` fungsi `migrate` mengalir ke CloudWatch Logs, sehingga perintahnya kini menolak berjalan di sana |
| 8 Agustus 2026 | §9.3 memperoleh perintah kedua. **`seed:admin`** membuat Administrator pertama dengan kata sandi yang ditentukan operator dan dibaca dari **stdin**, bukan dari argumen proses yang terbaca pengguna lain lewat `/proc/<pid>/cmdline` dan tersimpan pada riwayat shell. **`admin:create`** tetap ada bagi reset sesudahnya dan tetap membangkitkan kata sandi sesuai P17. Menutup titik henti "cara membuat Administrator pertama" (**CK-A-09**) |
