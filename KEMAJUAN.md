# Kemajuan Implementasi — EduTrack

| Keterangan | Isi |
|---|---|
| **Tanggal** | 11 Agustus 2026 |
| **Kedudukan** | Mencatat **tahap mana sudah selesai dan apa buktinya**. Berada di luar rantai penguncian dan tidak menetapkan apa pun |
| **Tahapnya sendiri** | [AGENTS.md §8](./AGENTS.md) — dokumen ini tidak mengulang isi maupun gerbangnya |

> **Berkas ini tidak pernah memuat keputusan.** Alasan, ketentuan, dan kontrak berada pada rantai penguncian; yang dicatat di sini hanya **keadaan** dan **penunjuk ke buktinya**. Setiap baris yang mulai menjelaskan *kenapa* sudah salah tempat, dan wajib dipindahkan ke dokumen yang berwenang.
>
> Alasannya: dua tempat yang menjelaskan hal yang sama akan menyimpang, dan yang menyimpang selalu yang jarang dibaca. Dokumen ini sengaja dibuat tidak layak dijadikan rujukan untuk apa pun selain "sudah sampai mana".

---

## 1. Jalur A — aplikasi

> **Urutan sejak 11 Agustus 2026: lokal lebih dahulu.** Jalur A diselesaikan sampai seluruh fiturnya berjalan setempat; Jalur B ditahan setelah B1 — [AGENTS.md §8](./AGENTS.md).


| # | Tahap | Keadaan | Bukti |
|:--:|---|:--:|---|
| **A0** | Kerangka repositori | ✅ Selesai | `1b52439` |
| **A1** | Docker dan compose | ✅ Selesai | `1b52439` |
| **A2** | Skema dan migrasi | ✅ Selesai | `a562b60` · PR #1 |
| **A3** | `domain/` murni | ✅ Selesai | `15365e5`, `a1e357f` · PR #4 |
| **A4** | Auth dan sesi | ✅ Selesai | `474d789`, `57b7e91`, `0f4135c` · PR #3 |
| **A5** | Administrasi | ✅ Selesai | `4a553f7` · branch `fitur/a5-administrasi` · PR #5 |
| **A6** | Nilai dan presensi | ✅ Selesai | `8a2d93f` · branch `fitur/a6-nilai-presensi` · PR #6 |
| **A7** | Rapor | 🟨 Sebagian | `743b86d`, `f3cdce0` · PR #7 **sudah masuk `main`** (`d022a43`). Seluruh enam endpoint [API §8](./API.md) beserta gerbangnya lulus. Templat sudah selaras dengan [ARCHITECTURE §11.3](./ARCHITECTURE.md). **Satu hal belum tuntas dan memang tidak dapat lokal:** pengukuran render di Lambda — **dikerjakan sebagai bagian Jalur B**, bukan sebagai pekerjaan A7 tersendiri |
| **A8** | Jalur AI | ✅ Selesai | branch `fitur/a8-jalur-ai` · PR #8 **sudah masuk `main`** (`863f40b`). `POST /api/saya/suggestion` beserta port, adapter, konteks `app_ro`, dan pembatas laju. **AC-18 dan AC-31 dibuktikan terhadap Gemini 3.6 Flash sungguhan** lewat `npm run uji:saran` |

**Angka gerbang pada saat A8 ditutup.** Diperbarui hanya ketika satu tahap selesai, bukan setiap commit.

| Perintah | Hasil |
|---|---|
| `npm run periksa` | keluar 0 · unit 22 berkas / 234 tes lulus · DB coverage suite 23 berkas / 407 tes lulus |
| `npm run test:db` | keluar 0 · 23 berkas / 407 tes lulus |
| `npm run lint:migrations` | 0 temuan pada 10 berkas |
| `npm run coverage:global` | keluar 0 · statements 91,65% · branches 88,28% · functions 87,98% · lines 91,65% |
| `npm audit --omit=dev` | 0 kerentanan produksi |
| `git diff --check` | keluar 0 |
| Cakupan `src/domain` | statements 100% · branches 100% · functions 100% · lines 100% |
| `npm run uji:saran` | AC-18 dan AC-31 lulus pada tiga konteks; tanpa pelanggaran terdeteksi |
| `npm run ukur:render` | 30 berkas dalam **431 ms** pada mesin pengembang — **bukan** angka Lambda yang dituntut [API §13.3](./API.md) |

---

## 2. Jalur B — infrastruktur

> **Infrastruktur berdiri 12 Agustus 2026.** B2 sampai B5 selesai; yang tersisa hanya pengisian rahasia, rilis pertama, dan pengukuran render.
>
> Akun berjalan pada **AWS Free Plan** yang membatasi apa yang boleh dibuat — bukan memberi potongan (**CK-19** [Techstack](./Techstack.md)). Dua tetapan disesuaikan karenanya: NAT `t4g.micro` dan retensi cadangan **1 hari**. Reserved concurrency tidak dapat disetel sama sekali karena plafon akun **10** (**CK-A-11**).

| # | Tahap | Keadaan | Bukti |
|:--:|---|:--:|---|
| **B0** | IAM: user, grup, role | 🟨 Sebagian | [DEPLOYMENT.md §9.9](./DEPLOYMENT.md), diperbarui 11 Agustus. Yang tersisa **hanya pekerjaan manusia**: role `edutrack-readonly`, pendaftaran MFA, dan penghapusan user `admin` bawaan |
| **B0.5** | OIDC provider, role, jabat tangan | ✅ Selesai | [Gitaction.md](./Gitaction.md) · workflow `oidc-smoke.yml` |
| **B1** | `terraform apply` pada `bootstrap/` | ✅ Selesai | Repositori **`infra`** `b925880`. `apply` bersih, **7 sumber daya dibuat**: bucket state beserta versioning, enkripsi, blok akses publik, dan kebijakan TLS; ECR `edutrack` beserta aturan daur hidupnya |
| **B2** | Push image bootstrap ke ECR | ✅ Selesai | Image `edutrack:bootstrap` terpasang pada kedua fungsi dan menjawab `200` lewat CloudFront |
| **B3** | `terraform apply` pada `infra/` | ✅ Selesai | **Infrastruktur berdiri 12 Agustus 2026** — `infra` `da04e1b`. VPC 2 AZ, NAT `t4g.micro`, RDS PostgreSQL 17, CloudFront `d2mw289fm4g0eo.cloudfront.net`, dua fungsi Lambda beserta alias `live`. `plan -detailed-exitcode` keluar **0** — tanpa drift |
| **B4** | Pembuktian penandatanganan OAC | ✅ Selesai | **5 lulus, 0 gagal** terhadap infrastruktur sungguhan. Hasilnya **CK-A-12** [ARCHITECTURE §12.2](./ARCHITECTURE.md): request ber-body wajib membawa `x-amz-content-sha256`. Dua jebakan penaikannya pada [DEPLOYMENT §5.2](./DEPLOYMENT.md) |
| **B5** | Izin ECR dan Lambda pada role OIDC | ✅ Selesai | Role `edutrack-gha-backend` **di-`import`**, bukan dibuat ulang; kebijakannya dikelola Terraform. Role frontend menunggu nilai `sub`-nya |
| **B6** | `pr.yml` dan `deploy.yml` | ✅ Selesai | `pr.yml` menyala; `deploy.yml` ditulis pada backend `5e061e0`, dua belas langkah [DEPLOYMENT §3.3](./DEPLOYMENT.md) masing-masing menyebut nomornya. **Penyalaannya menunggu B5** |
| **B7** | Pengukuran render 30 PDF di Lambda | 🟨 Sebagian | Prosedur beserta skripnya ditulis — `infra/ukur-render/`. Menutup gerbang **A7** yang tersisa ([API.md §13.3](./API.md)). **Pengukurannya menunggu B6 menyala** |

**Butir 0 — sambungan dua lingkungan.** Prasyarat seluruh Jalur B, dan **selesai** pada backend branch `fitur/b0-sambungan-dua-lingkungan`, dibuka sebagai **PR #9** (`df4b069`…`b3918f3`). Port `Rahasia`, adapter AWS untuk Secrets Manager, SSM, dan S3, serta pemilihan adapter lewat satu variabel `LINGKUNGAN`. Seluruh adapter AWS diuji dengan klien tiruan; tidak ada satu pun panggilan sungguhan ke AWS.

---

## 3. Yang sedang menunggu manusia

Dicatat di sini hanya **judul dan tempatnya**. Isinya tidak disalin.

| Yang ditunggu | Tercatat pada | Menghambat |
|---|---|---|
| Sisa **B0**: role `edutrack-readonly`, pendaftaran MFA, penghapusan user `admin` bawaan | [DEPLOYMENT.md §9.8 dan §9.9](./DEPLOYMENT.md) | Tidak menghambat apa pun, tetapi menahan §9.8 |
| ~~Pengisian kedua rahasia dan parameter SSM~~ — **selesai 12 Agustus 2026** | [DEPLOYMENT.md §5.1](./DEPLOYMENT.md) | — |
| ~~Setelan `AWS_ROLE_ARN` dan `ALAMAT_PUBLIK`~~ — **selesai 12 Agustus 2026** | `backend/.github/workflows/deploy.yml` | — |
| **Pembungkus `fetch` frontend** yang menghitung `x-amz-content-sha256` | **CK-A-12** [ARCHITECTURE §12.2](./ARCHITECTURE.md) | Seluruh jalur tulis dari frontend |
| Nama domain dan pembeliannya | [Techstack.md §9](./Techstack.md) butir 4 · [AGENTS.md §10](./AGENTS.md) | Penerapan CK-17. **Sengaja dikerjakan paling akhir** |
| Kredensial AWS, `terraform apply`, pembuatan rahasia | [AGENTS.md §10](./AGENTS.md) | Seluruh Jalur B |
| Validasi komponen dan bobot templat | **V1** pada [ATURAN-DAN-KRITERIA.md §5](./ATURAN-DAN-KRITERIA.md) | Tidak menghambat — hanya data |
| Kredensial AWS untuk mengukur render 30 PDF di Lambda 1024 MB arm64 | [API.md §13.3](./API.md) | Penutupan gerbang A7. Angka pembanding lokal sudah ada |
| Retensi dan pencadangan data | **V6** · [Techstack.md §9](./Techstack.md) butir 3 | Tidak menghambat |
| Status kelayakan free tier RDS pada akun AWS tim | [Techstack.md §9](./Techstack.md) butir 7 | Tidak menghambat — menentukan $32,41 atau $11,40 per bulan |

## 4. Temuan yang masih terbuka

| # | Tercatat pada |
|---|---|
| S-01, S-07 | [SCHEMA.md §12](./SCHEMA.md) |
| A-04, A-07, A-08, A-09 | [API.md §13.1](./API.md) |
| T-01, T-04, T-05, T-06 | [RFC-001 §10](./RFC-001-model-data-konseptual.md) |

**Yang ditutup sepanjang 7–8 Agustus 2026:** S-02, S-03, S-04, S-05, S-06 · T-02, T-03 · A-05, A-06. Rinciannya pada dokumen masing-masing.

**Yang ditutup 10 Agustus 2026:** A-03.

## 5. Utang teknis yang sudah disepakati

| Isi | Jatuh tempo | Tercatat pada |
|---|---|---|
| Kerentanan `npm audit` pada `devDependencies` | Kapan saja — nol pada jalur produksi | — |
| Lapis 4 dan 5 penjagaan migrasi | Sebelum data sekolah dimuat | [DEPLOYMENT.md §6.5](./DEPLOYMENT.md) |
| Pengukuran lama render tiga puluh PDF **di Lambda** | Sebelum rilis pertama | [API.md §13.3](./API.md) |
| ~~Aplikasi menyambung sebagai `edutrack_owner`~~ — **ditutup A8** | — | [ARCHITECTURE §8](./ARCHITECTURE.md) menetapkan aplikasi memakai `app_rw` dan jalur AI memakai `app_ro`. Kenyataannya `docker-compose.yml` dan `.env.example` sejak A1 memakai `edutrack_owner`, yaitu role yang boleh DDL. Tidak terlihat selama ini karena suite penegakan basis data menyambung sebagai `app_ro` sendiri, sehingga I-23 tetap terbukti sementara aplikasinya berjalan dengan hak berlebih. Ditutup bersama A8, yang memang menuntut koneksi kedua |
| ~~**Sambungan dua lingkungan**~~ — **ditutup 11 Agustus 2026** | — | Port `Rahasia` beserta adapter lokal dan AWS, adapter S3, dan pemilihan adapter lewat `LINGKUNGAN` pada `entry/`. Branch `fitur/b0-sambungan-dua-lingkungan` |

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 12 Agustus 2026 | **Rahasia terisi dan terverifikasi.** Kedua role `app_rw` dan `app_ro` disetel pada RDS, kata sandinya disimpan di Secrets Manager, dan kunci Elice masuk ke SSM. Dibuktikan bukan dengan memeriksa keberadaan versi, melainkan dengan **masuk sungguhan ke PostgreSQL memakai kata sandi yang dibaca balik dari Secrets Manager**. Aturan security group sementara dicabut kembali; RDS hanya menerima dari security group Lambda. `AWS_ROLE_ARN` dan `ALAMAT_PUBLIK` disetel pada repositori backend |
| 12 Agustus 2026 | **Infrastruktur berdiri — B2, B3, B4, dan B5 selesai.** `terraform apply` dijalankan agen memakai sesi MFA yang sudah dipinjam manusia; `plan -detailed-exitcode` keluar 0 tanpa drift. **B4 lulus 5 dari 5** dan melahirkan **CK-A-12**: request ber-body wajib membawa `x-amz-content-sha256`, sementara seluruh jalur `GET` sehat tanpanya. Dua jebakan penaikan dicatat [DEPLOYMENT §5.2](./DEPLOYMENT.md) |
| 12 Agustus 2026 | **CK-19** dan **CK-A-11** — akun ternyata berada pada AWS Free Plan yang membatasi. NAT `t4g.nano` → `t4g.micro`, retensi cadangan 7 → 1 hari, dan reserved concurrency dicabut karena plafon akun 10 menolak reservasi berapa pun |
| 11 Agustus 2026 | **Jalur B ditulis sampai tuntas, B2 sampai B7.** Image `:bootstrap` tersendiri beserta alat ukur body, `infra/` lengkap, prosedur pembuktian OAC, izin OIDC, `deploy.yml`, dan prosedur pengukuran render Lambda. Seluruhnya digerbang — `terraform fmt -check` dan `terraform validate` bersih, `npm run periksa` keluar 0. Yang tersisa pada setiap tahap adalah tindakan yang menuntut kredensial AWS |
| 11 Agustus 2026 | **Utang sambungan dua lingkungan ditutup.** Port `Rahasia`, adapter AWS untuk Secrets Manager, SSM, dan S3, serta pemilihan adapter lewat satu variabel `LINGKUNGAN`. Janji "satu image, dua lingkungan" [ARCHITECTURE Pasal 13](./ARCHITECTURE.md) kini memiliki penerapannya |
| 11 Agustus 2026 | Empat keputusan baru pada [DEPLOYMENT.md](./DEPLOYMENT.md): **CK-D-06** kata sandi master RDS dikelola RDS sendiri, **CK-D-07** alias `live` lahir menunjuk `$LATEST`, **CK-D-08** fungsi `migrate` dipilih variabel lingkungan `PERAN`. Ketiganya lahir dari pertentangan antara dokumen dan apa yang benar-benar dapat berjalan, dan ketiganya ditulis mendahului kodenya. §3.3 langkah 11 dikoreksi menjadi `/api/healthz` |
| 11 Agustus 2026 | A8 dibuka sebagai **PR #8**. Pengukuran render Lambda dipindahkan menjadi butir **B7** pada Jalur B — ia memang pekerjaan infrastruktur, bukan sisa pekerjaan A7 |
| 11 Agustus 2026 | **A8 selesai.** AC-18 dan AC-31 dibuktikan terhadap Gemini 3.6 Flash sungguhan lewat `npm run uji:saran`; sapaan pada prompt dipatok supaya keluarannya tidak berganti-ganti. **Seluruh tahap Jalur A yang dapat dikerjakan lokal kini tuntas** |
| 11 Agustus 2026 | **A8 dikerjakan** pada `9c2fae6`: tombol Suggestion beserta port `AiAdvisor`, adapter OpenAI-compatible, konteks `app_ro`, dan pembatas laju 5 per jam. Ditandai sebagian karena AC-18 dan AC-31 menuntut model sungguhan. Utang pemisahan `app_rw`/`app_ro` **ditutup** |
| 11 Agustus 2026 | PR #7 digabungkan ke `main` (`d022a43`). Seluruh bagian A7 yang dapat dikerjakan lokal **sudah selesai**; yang menahan statusnya tetap sebagian hanyalah pengukuran render di Lambda |
| 11 Agustus 2026 | Tercatat bahwa aplikasi menyambung sebagai `edutrack_owner` alih-alih `app_rw` sejak A1 — celah yang tertutup dari pandangan karena suite penegakan basis data memakai `app_ro` sendiri. Ditutup bersama A8 |
| 11 Agustus 2026 | **A7-a selesai** — templat rapor diselaraskan ke [ARCHITECTURE §11.3](./ARCHITECTURE.md) beserta tesnya. `KomponenCetak` dan tanggal finalisasi dikeluarkan dari port, karena port seharusnya menggambarkan persis apa yang tercetak. Render 30 berkas turun 598 → 431 ms |
| 11 Agustus 2026 | **Urutan berubah: lokal lebih dahulu.** Jalur B ditahan setelah B1; Jalur A diselesaikan sampai seluruh fiturnya berjalan setempat. Utang "sambungan dua lingkungan" dicatat menggantikan butir adapter S3, karena persoalannya lebih luas daripada satu adapter |
| 11 Agustus 2026 | **B1 selesai** — `terraform apply` pada `bootstrap/` menambahkan 7 sumber daya. Jalur B menyala untuk pertama kalinya. **CK-18** menolak RDS Proxy |
| 11 Agustus 2026 | **V5 terjawab.** Isi berkas rapor ditetapkan [ARCHITECTURE §11.3](./ARCHITECTURE.md) beserta CK-A-10, mengikuti rapor resmi yang dipakai sekolah. Seluruh bidangnya sudah ada pada model data, sehingga tidak ada amandemen SCHEMA maupun API. Temuan **A-09** dibuka untuk identitas sekolah |
| 11 Agustus 2026 | Repositori ketiga **`infra`** dibuat berisi Terraform Jalur B. `bootstrap/` selesai ditulis: bucket state, penguncian bawaan S3 (CK-D-04), dan ECR `edutrack` |
| 11 Agustus 2026 | Gerbang biaya [Techstack §8.2](./Techstack.md) ditutup: tarif `ap-southeast-3` diverifikasi terhadap AWS Price List API, total $32,41 per bulan atau $11,40 dengan free tier RDS. Jalur B dapat dimulai |
| 11 Agustus 2026 | A7 Rapor dikerjakan pada backend `743b86d` di branch `fitur/a7-rapor` dan dibuka sebagai PR #7: enam endpoint [API §8](./API.md), salinan beku `rapor_mapel`, perenderan PDF pada saat finalisasi beserta anggaran lunak 20 detik, dan unduh per siswa maupun sekelas. Ditandai **sebagian** karena tata letak PDF menunggu V5 dan pengukuran render Lambda menunggu Jalur B. Temuan **A-07** dan **A-08** dibuka |
| 11 Agustus 2026 | A6 Nilai dan presensi ditandai selesai pada backend `8a2d93f` di branch `fitur/a6-nilai-presensi` dan dibuka sebagai PR #6; 40 berkas / 514 tes lulus pada gabungan suite unit dan DB |
| 10 Agustus 2026 | A5 Administrasi ditandai selesai pada backend `4a553f7` di branch `fitur/a5-administrasi`; tabel gerbang diperbarui dari angka A4 ke angka A5 dan utang cakupan global dihapus |
| 10 Agustus 2026 | A-03 dikeluarkan dari daftar temuan terbuka setelah AC-26 diselaraskan dengan kontrak unggah tolak-seluruhnya pada API |
| 8 Agustus 2026 | Bukti A3 diperbaiki dari PR #2 menjadi **PR #4**. PR #2 menggabungkan A3 ke `fitur/a2-skema-migrasi` **sesudah** cabang itu sendiri sudah masuk `main`, sehingga A3 tidak pernah sampai ke `main`; PR #4 yang membawanya |
| 8 Agustus 2026 | Dokumen dibuat sesudah A4 selesai. Sebelumnya tidak ada satu tempat pun yang mencatat tahap mana sudah selesai — jawabannya hanya dapat diperoleh dengan membaca riwayat git atau daftar pull request, dan keduanya bukan dokumen |
