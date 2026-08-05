# SCHEMA — Opsi STATIS

> ## ⚠️ ARSIP — TIDAK BERLAKU
>
> **Status:** Dokumentasi lama. Disusun 2 Agustus 2026 berdasarkan asumsi sebelum PRD v3.0.
> Digantikan oleh [`RFC-001-model-data-konseptual.md`](./RFC-001-model-data-konseptual.md).
> Disimpan sebagai rekam jejak pertimbangan, **bukan sebagai acuan pembangunan**.
>
> Alasan tidak berlaku: NG10 pada PRD v3.0 mencabut pembobotan per mata pelajaran,
> NG11 mencabut ranah Keterampilan dan Sikap, dan NG14 mencabut penyimpanan keluaran AI.
> Selain itu dokumen ini tidak memuat KKM, entitas sesi presensi, maupun model mata
> pelajaran per jenjang yang diwajibkan PRD v3.0 §8.2, §8.3, dan §8.4.
>
> Bandingkan dengan [`SCHEMA-DINAMIS.md`](./SCHEMA-DINAMIS.md), yang juga berstatus arsip.

**Ciri:** komponen nilai disimpan sebagai **kolom tetap** (`t1 … uas`). Bobot per `mapel × tingkat × periode`.

**Total 18 tabel.** Yang khas statis hanya 2: `bobot` dan `nilai`.

---

## Field list

```
Table Name        Field Name            Type & Nullable
tahun_ajaran      id                    uuid | NOT NULL
tahun_ajaran      nama                  text | NOT NULL
tahun_ajaran      tgl_mulai             date | NOT NULL
tahun_ajaran      tgl_selesai           date | NOT NULL
tahun_ajaran      aktif                 boolean | NOT NULL
tahun_ajaran      dibuat_pada           timestamptz | NOT NULL
tahun_ajaran      diperbarui_pada       timestamptz | NOT NULL

periode           id                    uuid | NOT NULL
periode           tahun_ajaran_ref      uuid | NOT NULL
periode           semester              text | NOT NULL
periode           tgl_mulai             date | NOT NULL
periode           tgl_selesai           date | NOT NULL
periode           status                text | NOT NULL
periode           dibuat_pada           timestamptz | NOT NULL
periode           diperbarui_pada       timestamptz | NOT NULL

mapel             id                    uuid | NOT NULL
mapel             kode                  text | NOT NULL
mapel             nama                  text | NOT NULL
mapel             dibuat_pada           timestamptz | NOT NULL
mapel             diperbarui_pada       timestamptz | NOT NULL

kelas             id                    uuid | NOT NULL
kelas             nama                  text | NOT NULL
kelas             tingkat               text | NOT NULL
kelas             jurusan               text | NULLABLE
kelas             periode_ref           uuid | NOT NULL
kelas             wali_kelas_ref        uuid | NULLABLE
kelas             dibuat_pada           timestamptz | NOT NULL
kelas             diperbarui_pada       timestamptz | NOT NULL

users             id                    uuid | NOT NULL
users             cognito_sub           text | NOT NULL
users             nama                  text | NOT NULL
users             role                  text | NOT NULL
users             aktif                 boolean | NOT NULL
users             dibuat_pada           timestamptz | NOT NULL
users             diperbarui_pada       timestamptz | NOT NULL

guru              user_ref              uuid | NOT NULL
guru              nip                   text | NULLABLE
guru              email                 text | NOT NULL
guru              gelar                 text | NULLABLE

siswa             user_ref              uuid | NOT NULL
siswa             nis                   text | NOT NULL
siswa             nisn                  text | NULLABLE
siswa             jenis_kelamin         text | NULLABLE
siswa             tgl_lahir             date | NULLABLE

kelas_siswa       id                    uuid | NOT NULL
kelas_siswa       kelas_ref             uuid | NOT NULL
kelas_siswa       siswa_ref             uuid | NOT NULL
kelas_siswa       status                text | NOT NULL
kelas_siswa       dibuat_pada           timestamptz | NOT NULL

penugasan         id                    uuid | NOT NULL
penugasan         guru_ref              uuid | NOT NULL
penugasan         mapel_ref             uuid | NOT NULL
penugasan         kelas_ref             uuid | NOT NULL
penugasan         periode_ref           uuid | NOT NULL
penugasan         dibuat_pada           timestamptz | NOT NULL
penugasan         diperbarui_pada       timestamptz | NOT NULL

bobot             id                    uuid | NOT NULL
bobot             mapel_ref             uuid | NOT NULL
bobot             tingkat               text | NOT NULL
bobot             periode_ref           uuid | NOT NULL
bobot             t1                    numeric(5,2) | NULLABLE
bobot             t2                    numeric(5,2) | NULLABLE
bobot             uh1                   numeric(5,2) | NULLABLE
bobot             uh2                   numeric(5,2) | NULLABLE
bobot             uts                   numeric(5,2) | NULLABLE
bobot             uas                   numeric(5,2) | NULLABLE
bobot             status                text | NOT NULL
bobot             disinkron_oleh        uuid | NULLABLE
bobot             disinkron_pada        timestamptz | NULLABLE
bobot             dibuat_pada           timestamptz | NOT NULL
bobot             diperbarui_pada       timestamptz | NOT NULL

nilai             id                    uuid | NOT NULL
nilai             penugasan_ref         uuid | NOT NULL
nilai             siswa_ref             uuid | NOT NULL
nilai             t1                    numeric(5,2) | NULLABLE
nilai             t2                    numeric(5,2) | NULLABLE
nilai             uh1                   numeric(5,2) | NULLABLE
nilai             uh2                   numeric(5,2) | NULLABLE
nilai             uts                   numeric(5,2) | NULLABLE
nilai             uas                   numeric(5,2) | NULLABLE
nilai             diperbarui_oleh       uuid | NULLABLE
nilai             dibuat_pada           timestamptz | NOT NULL
nilai             diperbarui_pada       timestamptz | NOT NULL

penilaian_sikap   id                    uuid | NOT NULL
penilaian_sikap   penugasan_ref         uuid | NOT NULL
penilaian_sikap   siswa_ref             uuid | NOT NULL
penilaian_sikap   spiritual             text | NULLABLE
penilaian_sikap   ket_spiritual         text | NULLABLE
penilaian_sikap   sosial                text | NULLABLE
penilaian_sikap   ket_sosial            text | NULLABLE
penilaian_sikap   diperbarui_oleh       uuid | NULLABLE
penilaian_sikap   diperbarui_pada       timestamptz | NOT NULL

presensi          id                    uuid | NOT NULL
presensi          penugasan_ref         uuid | NOT NULL
presensi          siswa_ref             uuid | NOT NULL
presensi          tanggal               date | NOT NULL
presensi          status                text | NOT NULL
presensi          catatan               text | NULLABLE
presensi          dicatat_oleh          uuid | NULLABLE
presensi          dibuat_pada           timestamptz | NOT NULL

materi            id                    uuid | NOT NULL
materi            mapel_ref             uuid | NOT NULL
materi            tingkat               text | NOT NULL
materi            nama                  text | NOT NULL
materi            urutan                integer | NOT NULL

rapor             id                    uuid | NOT NULL
rapor             siswa_ref             uuid | NOT NULL
rapor             kelas_ref             uuid | NOT NULL
rapor             periode_ref           uuid | NOT NULL
rapor             status                text | NOT NULL
rapor             versi                 integer | NOT NULL
rapor             rata_rata             numeric(5,2) | NULLABLE
rapor             pdf_s3_key            text | NULLABLE
rapor             dibuat_oleh           uuid | NULLABLE
rapor             dibuat_pada           timestamptz | NOT NULL
rapor             diajukan_pada         timestamptz | NULLABLE
rapor             disetujui_oleh        uuid | NULLABLE
rapor             diterbitkan_pada      timestamptz | NULLABLE

rapor_mapel       id                    uuid | NOT NULL
rapor_mapel       rapor_ref             uuid | NOT NULL
rapor_mapel       mapel_nama            text | NOT NULL
rapor_mapel       nilai_akhir           numeric(5,2) | NULLABLE
rapor_mapel       kehadiran_persen      numeric(5,2) | NULLABLE
rapor_mapel       sikap_spiritual       text | NULLABLE
rapor_mapel       ket_spiritual         text | NULLABLE
rapor_mapel       sikap_sosial          text | NULLABLE
rapor_mapel       ket_sosial            text | NULLABLE
rapor_mapel       deskripsi_ai          text | NULLABLE
rapor_mapel       snapshot_nilai        jsonb | NULLABLE
rapor_mapel       snapshot_bobot        jsonb | NULLABLE

audit_log         id                    uuid | NOT NULL
audit_log         user_ref              uuid | NULLABLE
audit_log         judul                 text | NOT NULL
audit_log         deskripsi             text | NULLABLE
audit_log         aksi                  text | NOT NULL
audit_log         entitas               text | NOT NULL
audit_log         entitas_ref           uuid | NULLABLE
audit_log         sebelum               jsonb | NULLABLE
audit_log         sesudah               jsonb | NULLABLE
audit_log         severity              text | NOT NULL
audit_log         ip_address            inet | NULLABLE
audit_log         dibuat_pada           timestamptz | NOT NULL

ai_insight        id                    uuid | NOT NULL
ai_insight        siswa_ref             uuid | NOT NULL
ai_insight        periode_ref           uuid | NOT NULL
ai_insight        mapel_ref             uuid | NULLABLE
ai_insight        tipe                  text | NOT NULL
ai_insight        ringkasan             text | NOT NULL
ai_insight        detail                jsonb | NULLABLE
ai_insight        model                 text | NULLABLE
ai_insight        dibuat_pada           timestamptz | NOT NULL
```

---

## Constraints

```
periode          UNIQUE (tahun_ajaran_ref, semester)
kelas            UNIQUE (nama, periode_ref)
users            UNIQUE (cognito_sub)
guru             UNIQUE (email)
siswa            UNIQUE (nis)
kelas_siswa      UNIQUE (kelas_ref, siswa_ref)
penugasan        UNIQUE (mapel_ref, kelas_ref, periode_ref)
bobot            UNIQUE (mapel_ref, tingkat, periode_ref)
nilai            UNIQUE (penugasan_ref, siswa_ref)
penilaian_sikap  UNIQUE (penugasan_ref, siswa_ref)
presensi         UNIQUE (penugasan_ref, siswa_ref, tanggal)
rapor            UNIQUE (siswa_ref, periode_ref, versi)

nilai            CHECK (t1  BETWEEN 0 AND 100)   -- dst untuk t2, uh1, uh2, uts, uas
periode          CHECK (semester IN ('ganjil','genap'))
kelas            CHECK (tingkat IN ('X','XI','XII'))
users            CHECK (role IN ('superadmin','guru','siswa'))
presensi         CHECK (status IN ('hadir','izin','sakit','alpa'))
rapor            CHECK (status IN ('draft','review','distributed'))
bobot            CHECK (status IN ('draft','aktif'))
```

**Σ bobot = 100** tidak bisa jadi constraint biasa — dicek trigger saat `bobot.status` → `aktif`.

## Index

```
nilai            (penugasan_ref, siswa_ref)
presensi         (penugasan_ref, tanggal)
presensi         (siswa_ref, tanggal)
kelas_siswa      (kelas_ref)
kelas_siswa      (siswa_ref)
penugasan        (guru_ref, periode_ref)
audit_log        (dibuat_pada DESC)
```

---

## Volume — skenario 12 kelas / 360 siswa / 3 mapel

| Tabel | Baris |
|---|---|
| tahun_ajaran | 1 |
| periode | 2 |
| mapel | 3 |
| kelas | 12 |
| users | ~375 |
| kelas_siswa | 360 |
| penugasan | 24 |
| **bobot** | **9** |
| **nilai** | **720** |
| penilaian_sikap | 720 |
| presensi | ~11.500 |
| rapor | 360 |
| rapor_mapel | 720 |

---

## Perhitungan nilai akhir

Rumus **ada di kode aplikasi**, bukan di data:

```ts
const nilaiAkhir =
  n.t1*b.t1 + n.t2*b.t2 + n.uh1*b.uh1 + n.uh2*b.uh2 + n.uts*b.uts + n.uas*b.uas
```

---

## Batasan yang harus diterima

| # | Batasan |
|---|---|
| 1 | Tambah komponen (mis. `UH3`, `Lab1`) = `ALTER TABLE nilai` **dan** `ALTER TABLE bobot` + deploy, pada tabel berisi data |
| 2 | Jumlah komponen sama untuk semua mapel. Mapel dengan komponen lebih sedikit menyimpan `NULL` permanen |
| 3 | `NULL` ambigu: tidak bisa membedakan *"mapel ini tidak punya UH2"* dari *"punya, belum diisi"* — padahal layar siswa butuh membedakannya (`Menunggu sinkronisasi nilai`) |
| 4 | Rumus perhitungan di kode → ubah bobot berarti ubah kode |
| 5 | **Layar Rumus Nilai superadmin tidak bisa dibangun** — drag urutan, "+ Tambahkan Komponen", hapus komponen semuanya berarti `ALTER TABLE` dari klik tombol |
| 6 | Tidak mendukung rumus jamak (Pengetahuan + Keterampilan) tanpa menggandakan seluruh kolom |

Batasan #5 dan #6 yang paling menentukan: keduanya **sudah digambar di wireframe**.
