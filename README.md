# Docs — EduTrack

Kumpulan dokumen sumber kebenaran untuk produk **EduTrack** (Project ID: EDU-2026-001).

## Daftar Dokumen

### Berlaku

| Dokumen | Isi |
|---|---|
| [PRD.md](./PRD.md) | Product Requirements Document v3.0: lingkup, pengguna, dan kebutuhan produk |
| [ATURAN-DAN-KRITERIA.md](./ATURAN-DAN-KRITERIA.md) | Lampiran operasional PRD: aturan produk, use case, layar, dan kriteria kesiapan |
| [aktor-role.md](./aktor-role.md) | Aktor, peran, dan aturan izin akses |
| [RFC-001-model-data-konseptual.md](./RFC-001-model-data-konseptual.md) | Entitas, relasi, dan invarian basis data; netral teknologi |
| [Techstack.md](./Techstack.md) | Teknologi apa yang dipilih dan mengapa, beserta perkiraan biaya |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Bagaimana bagian-bagian sistem terhubung: peta besar, batas modul, jaringan, alur request |
| [SCHEMA.md](./SCHEMA.md) | Skema fisik PostgreSQL: tabel, constraint, indeks, pemicu, role, dan migrasi |
| [API.md](./API.md) | Kontrak endpoint: alamat, bentuk permintaan dan respons, kode status, katalog kesalahan |

### Berlaku sebagian

| Dokumen | Isi |
|---|---|
| [DEPLOYMENT.md](./DEPLOYMENT.md) | Bagaimana sistem dikirim dan dioperasikan: Terraform, CI/CD, rollback, identitas, pemasangan on-prem, pencadangan |

**Pasal 2, 3, 6, dan 9 sudah berlaku** — pembagian kepemilikan Terraform dan CI, urutan rilis, aturan migrasi, serta identitas dan akses. Pasal 1, 4, 5, 7, dan 8 masih rancangan, dan mulai diisi ketika Terraform mulai ditulis.

### Panduan kerja

| Dokumen | Isi |
|---|---|
| [AGENTS.md](./AGENTS.md) | Bagaimana agen membangun EduTrack: alur kerja, batas yang tidak boleh dilanggar, urutan tahap, dan gerbang selesai |
| [GLOSARIUM.md](./GLOSARIUM.md) | Setiap singkatan dan istilah teknis yang dipakai di seluruh dokumen |
| [KEMAJUAN.md](./KEMAJUAN.md) | Tahap mana sudah selesai dan apa buktinya. **Tidak memuat keputusan** — hanya keadaan dan penunjuk |

Berbeda dari dokumen di atasnya, `AGENTS.md` **tidak menetapkan apa pun tentang produk**. Ia menetapkan cara bekerja di atas dokumen yang sudah ada, sehingga berada di luar rantai penguncian.

`KEMAJUAN.md` berada lebih jauh lagi di luar: ia bahkan tidak menetapkan cara bekerja. Ia hanya menjawab "sudah sampai mana", dan setiap baris di dalamnya yang mulai menjelaskan *kenapa* adalah baris yang salah tempat.

### Runbook

Prosedur yang dijalankan tangan. Runbook **tidak menetapkan apa pun** — ia menjalankan keputusan yang sudah diambil dokumen di atasnya, dan menyebut dokumen mana yang mendasarinya.

| Dokumen | Isi |
|---|---|
| [RUNBOOK-OIDC.md](./RUNBOOK-OIDC.md) | Menyiapkan OIDC GitHub Actions ke AWS, langkah demi langkah, beserta diagnosa kegagalannya |
| [Gitaction.md](./Gitaction.md) | ✅ **Selesai.** Catatan penelusuran jabat tangan OIDC: penyebabnya *custom subject claim* tingkat organisasi yang mengubah format `sub` |

Catatan penelusuran seperti `Gitaction.md` bersifat **sementara**. Penyebabnya sudah dipindahkan ke tabel diagnosa `RUNBOOK-OIDC.md` Bagian 6 — tempat orang berikutnya akan mencarinya — sehingga berkas ini boleh dihapus kapan saja.

### Dokumen yang sudah dihapus

Empat dokumen dihapus pada 6 Agustus 2026 karena tidak lagi berlaku. Isinya tetap tersedia pada **riwayat Git** dan tidak boleh dijadikan rujukan.

| Dokumen | Alasan penghapusan |
|---|---|
| `SCHEMA-STATIS.md` | Rancangan skema komponen nilai sebagai kolom tetap. Digugurkan RFC-001 |
| `SCHEMA-DINAMIS.md` | Rancangan skema komponen nilai sebagai baris beserta tata kelola rumus. Digugurkan RFC-001 |
| `ARCHITECTURE-2026-08-02.md` | Arsitektur berbasis Hono di Lambda, Cognito, Bedrock, dan SQS. Digantikan `Techstack.md` dan `ARCHITECTURE.md` |
| `superadmin.md` | Memuat ketentuan yang bertentangan dengan PRD v3.0 |

Direktori `prototype/` juga dihapus pada tanggal yang sama. Isinya dibuat 4 Agustus 2026, satu hari sebelum PRD v3.0, dan masih menampilkan rumus penilaian dinamis serta ranah Sikap dan Keterampilan yang sudah dicabut NG10 dan NG11, sekaligus tidak memuat KKM, finalisasi, maupun tombol Suggestion. Direktori itu berada di luar repositori ini sehingga **tidak tersimpan pada riwayat Git**.

## Urutan Penguncian Keputusan

```
PRD.md  →  RFC-001  →  Techstack.md  →  ARCHITECTURE.md  →  SCHEMA.md  →  API.md
           model data   pilihan teknologi   hubungan antar bagian   skema fisik   kontrak endpoint
                                    ↓
                              DEPLOYMENT.md
                        penerapan & operasional
```

Biaya perubahan naik pada setiap langkah, sehingga yang paling mahal diubah dikunci paling akhir. Seluruh rantai sudah terkunci pada 6 Agustus 2026.

`DEPLOYMENT.md` berada di luar rantai penguncian karena isinya mengikuti keadaan infrastruktur yang berjalan, bukan menjadi dasar bagi dokumen berikutnya.

## Konvensi Dokumen

Dua jenis isi dipisahkan tegas, karena keduanya berumur berbeda:

| Jenis | Menjawab | Perlakuan |
|---|---|---|
| **Deskripsi** | Apa yang berlaku hari ini | Disunting langsung; bagian yang usang diganti, bukan ditumpuk |
| **Catatan Keputusan** | Kenapa dipilih, apa yang ditolak, kapan, oleh siapa | Bernomor dan bertanggal; tidak disunting, hanya ditambah atau diamandemen entri baru |

`Techstack.md`, `ARCHITECTURE.md`, `DEPLOYMENT.md`, `SCHEMA.md`, dan `API.md` masing-masing memuat **bagian deskripsi** di badan dokumen dan **lampiran Catatan Keputusan** di akhir. `RFC-001` seluruhnya berupa catatan keputusan, karena model data konseptual tidak memiliki padanan deskriptif.

**Penomoran Catatan Keputusan** memakai awalan per dokumen agar rujukan silang tidak ambigu:

| Dokumen | Awalan |
|---|---|
| `Techstack.md` | `CK-01` sampai `CK-15`, tanpa awalan — penomoran asli sebelum pemecahan, tidak dinomori ulang |
| `ARCHITECTURE.md` | `CK-A-01` dan seterusnya |
| `DEPLOYMENT.md` | `CK-D-01` dan seterusnya |
| `SCHEMA.md` | `CK-S-01` dan seterusnya |
| `API.md` | `CK-API-01` dan seterusnya |

Rujukan pasal di dalam `CK-01` sampai `CK-15` mengacu pada penomoran `Techstack.md` **sebelum** pemecahan 6 Agustus 2026. Entri tidak disunting, sesuai konvensi di atas; isi yang dirujuk kini berada pada `ARCHITECTURE.md` atau `DEPLOYMENT.md`.

**Amandemen lintas dokumen** ditulis pada dokumen yang memuat isinya hari ini, bukan pada dokumen yang memuat entri aslinya. `CK-A-07` mengamandemen `CK-09` dari `ARCHITECTURE.md` karena isi yang dirujuk `CK-09` sudah berpindah ke sana.

Alasan pemisahan: ketika PRD berubah, bagian deskripsi cukup dimutakhirkan, sedangkan Catatan Keputusan menunjukkan **keputusan mana yang perlu dibuka ulang**.

## Konvensi

- Seluruh dokumen ditulis dalam Bahasa Indonesia formal.
- **Singkatan dijelaskan pada pemakaian pertama di setiap dokumen, dan seluruhnya terdaftar pada [GLOSARIUM.md](./GLOSARIUM.md).** Singkatan yang muncul tanpa penjelasan adalah cacat dokumen, bukan pengetahuan yang boleh diandaikan.
- Setiap dokumen memiliki satu tanggung jawab dan tidak mengulang isi dokumen lain; gunakan tautan antar dokumen bila diperlukan.
- Perubahan pada dokumen dilakukan dengan mengganti bagian yang usang, bukan menumpuk versi baru di atas versi lama.
- Dokumen yang tidak lagi berlaku **dihapus**, bukan disimpan sebagai arsip. Riwayat Git yang menyimpannya.
