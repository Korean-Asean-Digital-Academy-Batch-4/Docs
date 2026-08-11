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

| # | Tahap | Keadaan | Bukti |
|:--:|---|:--:|---|
| **A0** | Kerangka repositori | ✅ Selesai | `1b52439` |
| **A1** | Docker dan compose | ✅ Selesai | `1b52439` |
| **A2** | Skema dan migrasi | ✅ Selesai | `a562b60` · PR #1 |
| **A3** | `domain/` murni | ✅ Selesai | `15365e5`, `a1e357f` · PR #4 |
| **A4** | Auth dan sesi | ✅ Selesai | `474d789`, `57b7e91`, `0f4135c` · PR #3 |
| **A5** | Administrasi | ✅ Selesai | `4a553f7` · branch `fitur/a5-administrasi` · PR #5 |
| **A6** | Nilai dan presensi | ✅ Selesai | `8a2d93f` · branch `fitur/a6-nilai-presensi` · PR #6 |
| **A7** | Rapor | 🟨 Sebagian | `743b86d` · branch `fitur/a7-rapor` · PR #7. Seluruh enam endpoint [API §8](./API.md) beserta gerbangnya lulus. **Satu hal belum tuntas:** pengukuran render belum dijalankan di Lambda. V5 sudah terjawab ([ARCHITECTURE §11.3](./ARCHITECTURE.md), CK-A-10); templatnya tinggal ditulis ulang |
| **A8** | Jalur AI | ⬜ Belum | — |

**Angka gerbang pada saat A7 dikerjakan.** Diperbarui hanya ketika satu tahap selesai, bukan setiap commit.

| Perintah | Hasil |
|---|---|
| `npm run periksa` | keluar 0 · unit 21 berkas / 206 tes lulus · DB coverage suite 22 berkas / 386 tes lulus |
| `npm run test:db` | keluar 0 · 22 berkas / 386 tes lulus |
| `npm run lint:migrations` | 0 temuan pada 10 berkas |
| `npm run coverage:global` | keluar 0 · statements 92,38% · branches 88,15% · functions 88,14% · lines 92,38% |
| `npm audit --omit=dev` | 0 kerentanan produksi |
| `git diff --check` | keluar 0 |
| Cakupan `src/domain` | statements 100% · branches 100% · functions 100% · lines 100% |
| `npm run ukur:render` | 30 berkas dalam 598 ms pada mesin pengembang — **bukan** angka Lambda yang dituntut [API §13.3](./API.md) |

---

## 2. Jalur B — infrastruktur

| # | Tahap | Keadaan | Bukti |
|:--:|---|:--:|---|
| **B0** | IAM: user, grup, role | 🟨 Sebagian | [DEPLOYMENT.md §9.9](./DEPLOYMENT.md) — **bertanggal 6 Agustus dan belum diperbarui**. Keberhasilan B0.5 membuktikan role OIDC sudah ada, tetapi §9.9 masih mendaftarnya sebagai belum ada |
| **B0.5** | OIDC provider, role, jabat tangan | ✅ Selesai | [Gitaction.md](./Gitaction.md) · workflow `oidc-smoke.yml` |
| **B1** | `terraform apply` pada `bootstrap/` | 🟨 Sebagian | Kodenya siap pada repositori **`infra`** `b925880` — `bootstrap/` beserta READMEnya. `terraform fmt` dan `validate` bersih. **`apply` menunggu manusia** (kode MFA) |
| **B2** | Push image bootstrap ke ECR | ⬜ Belum | — |
| **B3** | `terraform apply` pada `infra/` | ⬜ Belum | — |
| **B4** | Pembuktian penandatanganan OAC | ⬜ Belum | — |
| **B5** | Izin ECR dan Lambda pada role OIDC | ⬜ Belum | — |
| **B6** | `pr.yml` dan `deploy.yml` | 🟨 Sebagian | `pr.yml` menyala; `deploy.yml` menunggu B5 |

---

## 3. Yang sedang menunggu manusia

Dicatat di sini hanya **judul dan tempatnya**. Isinya tidak disalin.

| Yang ditunggu | Tercatat pada | Menghambat |
|---|---|---|
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
| Adapter penyimpanan S3 (`adapters/aws/`) | Sebelum rilis pertama | A7 memakai adapter disk lokal; tautan `file://` bukan tautan bertanda tangan |
| Templat pdfmake diselaraskan ke [ARCHITECTURE §11.3](./ARCHITECTURE.md) | Sebelum rapor pertama dibagikan | Templat saat ini masih bentuk sementara pra-V5, mencetak rincian komponen yang kini tidak diminta |

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 11 Agustus 2026 | **V5 terjawab.** Isi berkas rapor ditetapkan [ARCHITECTURE §11.3](./ARCHITECTURE.md) beserta CK-A-10, mengikuti rapor resmi yang dipakai sekolah. Seluruh bidangnya sudah ada pada model data, sehingga tidak ada amandemen SCHEMA maupun API. Temuan **A-09** dibuka untuk identitas sekolah |
| 11 Agustus 2026 | Repositori ketiga **`infra`** dibuat berisi Terraform Jalur B. `bootstrap/` selesai ditulis: bucket state, penguncian bawaan S3 (CK-D-04), dan ECR `edutrack` |
| 11 Agustus 2026 | Gerbang biaya [Techstack §8.2](./Techstack.md) ditutup: tarif `ap-southeast-3` diverifikasi terhadap AWS Price List API, total $32,41 per bulan atau $11,40 dengan free tier RDS. Jalur B dapat dimulai |
| 11 Agustus 2026 | A7 Rapor dikerjakan pada backend `743b86d` di branch `fitur/a7-rapor` dan dibuka sebagai PR #7: enam endpoint [API §8](./API.md), salinan beku `rapor_mapel`, perenderan PDF pada saat finalisasi beserta anggaran lunak 20 detik, dan unduh per siswa maupun sekelas. Ditandai **sebagian** karena tata letak PDF menunggu V5 dan pengukuran render Lambda menunggu Jalur B. Temuan **A-07** dan **A-08** dibuka |
| 11 Agustus 2026 | A6 Nilai dan presensi ditandai selesai pada backend `8a2d93f` di branch `fitur/a6-nilai-presensi` dan dibuka sebagai PR #6; 40 berkas / 514 tes lulus pada gabungan suite unit dan DB |
| 10 Agustus 2026 | A5 Administrasi ditandai selesai pada backend `4a553f7` di branch `fitur/a5-administrasi`; tabel gerbang diperbarui dari angka A4 ke angka A5 dan utang cakupan global dihapus |
| 10 Agustus 2026 | A-03 dikeluarkan dari daftar temuan terbuka setelah AC-26 diselaraskan dengan kontrak unggah tolak-seluruhnya pada API |
| 8 Agustus 2026 | Bukti A3 diperbaiki dari PR #2 menjadi **PR #4**. PR #2 menggabungkan A3 ke `fitur/a2-skema-migrasi` **sesudah** cabang itu sendiri sudah masuk `main`, sehingga A3 tidak pernah sampai ke `main`; PR #4 yang membawanya |
| 8 Agustus 2026 | Dokumen dibuat sesudah A4 selesai. Sebelumnya tidak ada satu tempat pun yang mencatat tahap mana sudah selesai — jawabannya hanya dapat diperoleh dengan membaca riwayat git atau daftar pull request, dan keduanya bukan dokumen |
