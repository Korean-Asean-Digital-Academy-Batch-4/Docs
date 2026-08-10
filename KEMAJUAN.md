# Kemajuan Implementasi — EduTrack

| Keterangan | Isi |
|---|---|
| **Tanggal** | 10 Agustus 2026 |
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
| **A6** | Nilai dan presensi | ⬜ Belum | — |
| **A7** | Rapor | ⬜ Belum | — |
| **A8** | Jalur AI | ⬜ Belum | — |

**Angka gerbang pada saat A5 ditutup.** Diperbarui hanya ketika satu tahap selesai, bukan setiap commit.

| Perintah | Hasil |
|---|---|
| `npm run periksa` | keluar 0 · unit 165 lulus · DB coverage suite 269 lulus |
| `npm run test:db` | 269 lulus |
| `npm run lint:migrations` | 0 temuan pada 10 berkas |
| `npm run coverage:global` | statements 91,38% · branches 87,69% · functions 84,29% · lines 91,38% |
| `npm audit --omit=dev` | 0 kerentanan produksi |
| `git diff --check` | keluar 0 |
| Cakupan `src/domain` | 100% pada keempat metrik tetap terjaga oleh konfigurasi Vitest |

---

## 2. Jalur B — infrastruktur

| # | Tahap | Keadaan | Bukti |
|:--:|---|:--:|---|
| **B0** | IAM: user, grup, role | 🟨 Sebagian | [DEPLOYMENT.md §9.9](./DEPLOYMENT.md) — **bertanggal 6 Agustus dan belum diperbarui**. Keberhasilan B0.5 membuktikan role OIDC sudah ada, tetapi §9.9 masih mendaftarnya sebagai belum ada |
| **B0.5** | OIDC provider, role, jabat tangan | ✅ Selesai | [Gitaction.md](./Gitaction.md) · workflow `oidc-smoke.yml` |
| **B1** | `terraform apply` pada `bootstrap/` | ⬜ Belum | — |
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
| Format rapor resmi sekolah | **V5** · [Techstack.md §9](./Techstack.md) butir 2 | A7 |
| Retensi dan pencadangan data | **V6** · [Techstack.md §9](./Techstack.md) butir 3 | Tidak menghambat |

## 4. Temuan yang masih terbuka

| # | Tercatat pada |
|---|---|
| S-01, S-07 | [SCHEMA.md §12](./SCHEMA.md) |
| A-04 | [API.md §13.1](./API.md) |
| T-01, T-04, T-05, T-06 | [RFC-001 §10](./RFC-001-model-data-konseptual.md) |

**Yang ditutup sepanjang 7–8 Agustus 2026:** S-02, S-03, S-04, S-05, S-06 · T-02, T-03 · A-05, A-06. Rinciannya pada dokumen masing-masing.

**Yang ditutup 10 Agustus 2026:** A-03.

## 5. Utang teknis yang sudah disepakati

| Isi | Jatuh tempo | Tercatat pada |
|---|---|---|
| Kerentanan `npm audit` pada `devDependencies` | Kapan saja — nol pada jalur produksi | — |
| Lapis 4 dan 5 penjagaan migrasi | Sebelum data sekolah dimuat | [DEPLOYMENT.md §6.5](./DEPLOYMENT.md) |
| Pengukuran lama render tiga puluh PDF | A7 | [API.md §13.3](./API.md) |

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 10 Agustus 2026 | A5 Administrasi ditandai selesai pada backend `4a553f7` di branch `fitur/a5-administrasi`; tabel gerbang diperbarui dari angka A4 ke angka A5 dan utang cakupan global dihapus |
| 10 Agustus 2026 | A-03 dikeluarkan dari daftar temuan terbuka setelah AC-26 diselaraskan dengan kontrak unggah tolak-seluruhnya pada API |
| 8 Agustus 2026 | Bukti A3 diperbaiki dari PR #2 menjadi **PR #4**. PR #2 menggabungkan A3 ke `fitur/a2-skema-migrasi` **sesudah** cabang itu sendiri sudah masuk `main`, sehingga A3 tidak pernah sampai ke `main`; PR #4 yang membawanya |
| 8 Agustus 2026 | Dokumen dibuat sesudah A4 selesai. Sebelumnya tidak ada satu tempat pun yang mencatat tahap mana sudah selesai — jawabannya hanya dapat diperoleh dengan membaca riwayat git atau daftar pull request, dan keduanya bukan dokumen |
