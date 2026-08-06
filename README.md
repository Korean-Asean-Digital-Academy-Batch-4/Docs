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

### Kerangka — isi belum ditulis

| Dokumen | Isi |
|---|---|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Bagaimana bagian-bagian sistem terhubung: peta besar, batas modul, jaringan, alur request |
| [DEPLOYMENT.md](./DEPLOYMENT.md) | Bagaimana sistem dikirim dan dioperasikan: Terraform, CI/CD, rollback, pemasangan on-prem, pencadangan |

Keduanya lahir dari pemecahan `Techstack.md` pada 6 Agustus 2026. Susunan pasalnya sudah ditetapkan; isinya menyusul.

### Menunggu penyelarasan dengan PRD v3.0

| Dokumen | Isi |
|---|---|
| [superadmin.md](./superadmin.md) | Kebutuhan dan alur khusus Super Admin |

Disusun 2 Agustus 2026 dan memuat ketentuan yang bertentangan dengan PRD v3.0. Rinciannya tercatat pada [RFC-001 §2.3](./RFC-001-model-data-konseptual.md).

### Arsip

| Dokumen | Isi |
|---|---|
| [SCHEMA-STATIS.md](./SCHEMA-STATIS.md) | Rancangan skema komponen nilai sebagai kolom tetap. Tidak berlaku |
| [SCHEMA-DINAMIS.md](./SCHEMA-DINAMIS.md) | Rancangan skema komponen nilai sebagai baris beserta tata kelola rumus. Tidak berlaku |
| [ARCHITECTURE-2026-08-02.md](./ARCHITECTURE-2026-08-02.md) | Arsitektur versi 2 Agustus 2026, berbasis Hono di Lambda, Cognito, Bedrock, dan SQS. Digantikan `Techstack.md` dan `ARCHITECTURE.md`. Tidak berlaku |

## Urutan Penguncian Keputusan

```
PRD.md  →  RFC-001  →  Techstack.md  →  ARCHITECTURE.md  →  SCHEMA.md  →  API.md
           model data   pilihan teknologi   hubungan antar bagian   skema fisik   kontrak endpoint
                                    ↓
                              DEPLOYMENT.md
                        penerapan & operasional
```

Biaya perubahan naik pada setiap langkah, sehingga yang paling mahal diubah dikunci paling akhir.

`DEPLOYMENT.md` berada di luar rantai penguncian karena isinya mengikuti keadaan infrastruktur yang berjalan, bukan menjadi dasar bagi dokumen berikutnya. Ia mulai diisi ketika Terraform mulai ditulis.

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

Rujukan pasal di dalam `CK-01` sampai `CK-15` mengacu pada penomoran `Techstack.md` **sebelum** pemecahan 6 Agustus 2026. Entri tidak disunting, sesuai konvensi di atas; isi yang dirujuk kini berada pada `ARCHITECTURE.md` atau `DEPLOYMENT.md`.

Alasan pemisahan: ketika PRD berubah, bagian deskripsi cukup dimutakhirkan, sedangkan Catatan Keputusan menunjukkan **keputusan mana yang perlu dibuka ulang**. Tanpa pemisahan ini, keduanya tidak dapat dibedakan — sebagaimana terjadi pada `ARCHITECTURE.md` versi 2 Agustus 2026.

## Konvensi

- Seluruh dokumen ditulis dalam Bahasa Indonesia formal.
- Setiap dokumen memiliki satu tanggung jawab dan tidak mengulang isi dokumen lain; gunakan tautan antar dokumen bila diperlukan.
- Perubahan pada dokumen dilakukan dengan mengganti bagian yang usang, bukan menumpuk versi baru di atas versi lama.
