# Docs — EduTrack

Kumpulan dokumen sumber kebenaran untuk produk **EduTrack** (Project ID: EDU-2026-001).

## Daftar Dokumen

### Berlaku

| Dokumen | Isi |
|---|---|
| [PRD.md](./PRD.md) | Product Requirements Document v3.0: lingkup, pengguna, dan kebutuhan produk |
| [ATURAN-DAN-KRITERIA.md](./ATURAN-DAN-KRITERIA.md) | Lampiran operasional PRD: aturan produk, use case, layar, dan kriteria kesiapan |
| [RFC-001-model-data-konseptual.md](./RFC-001-model-data-konseptual.md) | Entitas, relasi, dan invarian basis data; netral teknologi |

### Menunggu penyelarasan dengan PRD v3.0

| Dokumen | Isi |
|---|---|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Stack teknologi, struktur folder, batas modul, dan alur data |
| [aktor-role.md](./aktor-role.md) | Aktor, peran, dan aturan izin akses |
| [superadmin.md](./superadmin.md) | Kebutuhan dan alur khusus Super Admin |

Ketiganya disusun 2 Agustus 2026 dan memuat ketentuan yang bertentangan dengan PRD v3.0. Rinciannya tercatat pada [RFC-001 §2.3](./RFC-001-model-data-konseptual.md).

### Arsip

| Dokumen | Isi |
|---|---|
| [SCHEMA-STATIS.md](./SCHEMA-STATIS.md) | Rancangan skema komponen nilai sebagai kolom tetap. Tidak berlaku |
| [SCHEMA-DINAMIS.md](./SCHEMA-DINAMIS.md) | Rancangan skema komponen nilai sebagai baris beserta tata kelola rumus. Tidak berlaku |

Diagram alur tersimpan di direktori [`img/`](./img).

## Urutan Penguncian Keputusan

```
PRD.md  →  RFC-001  →  ARCHITECTURE.md  →  SCHEMA.md  →  API.md
           model data    stack & arsitektur   skema fisik   kontrak endpoint
```

Biaya perubahan naik pada setiap langkah, sehingga yang paling mahal diubah dikunci paling akhir.

## Konvensi Dokumen

Dua jenis isi dipisahkan tegas, karena keduanya berumur berbeda:

| Jenis | Menjawab | Perlakuan |
|---|---|---|
| **Deskripsi** | Apa yang berlaku hari ini | Disunting langsung; bagian yang usang diganti, bukan ditumpuk |
| **Catatan Keputusan** | Kenapa dipilih, apa yang ditolak, kapan, oleh siapa | Bernomor dan bertanggal; tidak disunting, hanya ditambah atau diamandemen entri baru |

`ARCHITECTURE.md`, `SCHEMA.md`, dan `API.md` masing-masing memuat **bagian deskripsi** di badan dokumen dan **lampiran Catatan Keputusan** di akhir. `RFC-001` seluruhnya berupa catatan keputusan, karena model data konseptual tidak memiliki padanan deskriptif.

Alasan pemisahan: ketika PRD berubah, bagian deskripsi cukup dimutakhirkan, sedangkan Catatan Keputusan menunjukkan **keputusan mana yang perlu dibuka ulang**. Tanpa pemisahan ini, keduanya tidak dapat dibedakan — sebagaimana terjadi pada `ARCHITECTURE.md` versi 2 Agustus 2026.

## Konvensi

- Seluruh dokumen ditulis dalam Bahasa Indonesia formal.
- Setiap dokumen memiliki satu tanggung jawab dan tidak mengulang isi dokumen lain; gunakan tautan antar dokumen bila diperlukan.
- Perubahan pada dokumen dilakukan dengan mengganti bagian yang usang, bukan menumpuk versi baru di atas versi lama.
