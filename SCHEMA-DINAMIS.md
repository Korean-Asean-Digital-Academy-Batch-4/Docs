# SCHEMA — Opsi DINAMIS

> **Status:** Direkomendasikan. Belum dikunci final.
> Bandingkan dengan [`SCHEMA-STATIS.md`](./SCHEMA-STATIS.md).
> Peran & izin: [`aktor-role.md`](./aktor-role.md) · Arsitektur: [`ARCHITECTURE.md`](./ARCHITECTURE.md)

**Ciri:** komponen nilai disimpan sebagai **baris**, bukan kolom. Rumus berlaku per `mapel × tingkat × periode × jenis`.

**Total 20 tabel.** Beda dari statis hanya 4: `bobot` dihapus; `rumus`, `komponen_rumus`, `penugasan_rumus` ditambah; `nilai` berubah bentuk. 16 tabel lainnya identik.

---

## Keputusan tata kelola (2 Agustus 2026)

**Guru mengusulkan, Superadmin menyetujui.**

```
[draft]      guru mapel bebas tambah/kurangi komponen & atur bobot
   │
   │  guru menekan "Ajukan"
   ▼
[diajukan]   terkunci dari guru. Masuk antrean Superadmin
   │
   │  Superadmin menekan "Sinkronkan"
   ▼
[aktif]      terkunci. Guru mulai input nilai.
```

| Status | Siapa boleh ubah komponen & bobot |
|---|---|
| `draft` | Guru mana pun yang punya `penugasan` untuk mapel+tingkat+periode itu |
| `diajukan` | Tidak ada — hanya Superadmin yang bisa kembalikan ke `draft` |
| `aktif` | Tidak ada — perubahan wajib persetujuan ulang Superadmin + tercatat di `audit_log` |

**Standardisasi terjadi dengan sendirinya:** guru A (X-IPA-1) dan guru B (X-IPS-1) sama-sama mengajar Matematika tingkat X → menemukan baris `rumus` yang sama. Struktur data yang membuat perbedaan mustahil, bukan aturan yang harus ditegakkan manual.

> ⚠️ **Konsekuensi UI:** butuh **1 layar baru di sidebar Guru** (editor rumus, aktif hanya saat `draft`). Belum ada di wireframe — perlu disampaikan ke UI/UX.

**Aturan perubahan setelah `aktif`:**

```
✅ menyentuh nilai berjalan & rapor berstatus draft
❌ rapor review      → tolak
❌ rapor distributed → tolak; wajib terbit ulang dengan versi baru
```

---

## Tabel yang khas dinamis

```
Table Name        Field Name            Type & Nullable
rumus             id                    uuid | NOT NULL
rumus             mapel_ref             uuid | NOT NULL
rumus             tingkat               text | NOT NULL
rumus             periode_ref           uuid | NOT NULL
rumus             jenis                 text | NOT NULL
rumus             nama                  text | NOT NULL
rumus             status                text | NOT NULL
rumus             diusulkan_oleh        uuid | NULLABLE
rumus             diajukan_pada         timestamptz | NULLABLE
rumus             disinkron_oleh        uuid | NULLABLE
rumus             disinkron_pada        timestamptz | NULLABLE
rumus             dibuat_pada           timestamptz | NOT NULL
rumus             diperbarui_pada       timestamptz | NOT NULL

komponen_rumus    id                    uuid | NOT NULL
komponen_rumus    rumus_ref             uuid | NOT NULL
komponen_rumus    kode                  text | NOT NULL
komponen_rumus    nama                  text | NOT NULL
komponen_rumus    bobot                 numeric(5,2) | NOT NULL
komponen_rumus    urutan                integer | NOT NULL
komponen_rumus    materi_ref            uuid | NULLABLE
komponen_rumus    dibuat_pada           timestamptz | NOT NULL

penugasan_rumus   penugasan_ref         uuid | NOT NULL
penugasan_rumus   rumus_ref             uuid | NOT NULL

nilai             id                    uuid | NOT NULL
nilai             penugasan_ref         uuid | NOT NULL
nilai             rumus_ref             uuid | NOT NULL
nilai             komponen_ref          uuid | NOT NULL
nilai             siswa_ref             uuid | NOT NULL
nilai             nilai                 numeric(5,2) | NULLABLE
nilai             diperbarui_oleh       uuid | NULLABLE
nilai             dibuat_pada           timestamptz | NOT NULL
nilai             diperbarui_pada       timestamptz | NOT NULL
```

## Tabel yang sama dengan statis

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
rapor_mapel       jenis                 text | NOT NULL
rapor_mapel       nilai_akhir           numeric(5,2) | NULLABLE
rapor_mapel       kehadiran_persen      numeric(5,2) | NULLABLE
rapor_mapel       sikap_spiritual       text | NULLABLE
rapor_mapel       ket_spiritual         text | NULLABLE
rapor_mapel       sikap_sosial          text | NULLABLE
rapor_mapel       ket_sosial            text | NULLABLE
rapor_mapel       deskripsi_ai          text | NULLABLE
rapor_mapel       snapshot_komponen     jsonb | NOT NULL

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

`rapor_mapel.snapshot_komponen` menyimpan salinan beku tiap komponen (`kode, nama, bobot, nilai`) — karena jumlah komponen berbeda antar mapel, `jsonb` lebih tepat daripada tabel anak dengan kolom tetap.

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
penilaian_sikap  UNIQUE (penugasan_ref, siswa_ref)
presensi         UNIQUE (penugasan_ref, siswa_ref, tanggal)
rapor            UNIQUE (siswa_ref, periode_ref, versi)

rumus            UNIQUE (mapel_ref, tingkat, periode_ref, jenis)
komponen_rumus   UNIQUE (rumus_ref, kode)
komponen_rumus   UNIQUE (rumus_ref, id)          -- penopang FK komposit
penugasan_rumus  PRIMARY KEY (penugasan_ref, rumus_ref)
nilai            UNIQUE (penugasan_ref, siswa_ref, komponen_ref)

periode          CHECK (semester IN ('ganjil','genap'))
kelas            CHECK (tingkat  IN ('X','XI','XII'))
users            CHECK (role     IN ('superadmin','guru','siswa'))
presensi         CHECK (status   IN ('hadir','izin','sakit','alpa'))
rapor            CHECK (status   IN ('draft','review','distributed'))
rumus            CHECK (jenis    IN ('pengetahuan','keterampilan'))
rumus            CHECK (status   IN ('draft','diajukan','aktif'))
komponen_rumus   CHECK (bobot > 0 AND bobot <= 100)
nilai            CHECK (nilai BETWEEN 0 AND 100)
```

### Foreign key komposit — penjaga integritas nilai

```sql
nilai  FOREIGN KEY (penugasan_ref, rumus_ref)
       REFERENCES penugasan_rumus (penugasan_ref, rumus_ref);

nilai  FOREIGN KEY (rumus_ref, komponen_ref)
       REFERENCES komponen_rumus (rumus_ref, id);
```

Membuat **mustahil** menyimpan nilai ke komponen milik mapel lain — Postgres yang menolak, bukan kode aplikasi. Untuk data yang salahnya berarti rapor siswa salah, jaminan ini sepadan dengan satu kolom tambahan.

### Trigger Σ bobot = 100

Dicek saat `rumus.status` berubah ke `aktif` — **bukan** saat tiap komponen diedit, supaya guru bisa menyusun bertahap (sempat 90% atau 110% di tengah proses) tanpa diblokir.

```sql
CREATE FUNCTION cek_total_bobot() RETURNS trigger AS $$
BEGIN
  IF NEW.status = 'aktif' AND
     (SELECT COALESCE(SUM(bobot),0) FROM komponen_rumus WHERE rumus_ref = NEW.id) <> 100
  THEN
    RAISE EXCEPTION 'Total bobot harus 100%%, bukan %',
      (SELECT COALESCE(SUM(bobot),0) FROM komponen_rumus WHERE rumus_ref = NEW.id);
  END IF;
  RETURN NEW;
END $$ LANGUAGE plpgsql;
```

## Index

```
nilai            (penugasan_ref, siswa_ref)
nilai            (komponen_ref)
komponen_rumus   (rumus_ref, urutan)
rumus            (mapel_ref, tingkat, periode_ref)
presensi         (penugasan_ref, tanggal)
presensi         (siswa_ref, tanggal)
kelas_siswa      (kelas_ref)
kelas_siswa      (siswa_ref)
penugasan        (guru_ref, periode_ref)
audit_log        (dibuat_pada DESC)
```

---

## Resolusi rumus — `penugasan` tidak memiliki, dia mencari

```
penugasan: Bu Rina × Matematika × X-IPA-1 × Ganjil-2026
                          │            │
                          │            └─ kelas.tingkat = X
                          ▼
   rumus WHERE mapel_ref = Matematika
           AND tingkat   = 'X'
           AND periode_ref = Ganjil-2026
           AND jenis IN ('pengetahuan','keterampilan')
```

Hasil resolusi dibekukan ke `penugasan_rumus` saat rumus disinkronkan. Satu penugasan bisa dapat **lebih dari satu** rumus — makanya lewat tabel join, bukan kolom `rumus_ref` di `penugasan`.

---

## Volume — skenario 12 kelas / 360 siswa / 3 mapel

| Tabel | Baris |
|---|---|
| tahun_ajaran · periode · mapel | 1 · 2 · 3 |
| kelas | 12 |
| users | ~375 |
| kelas_siswa | 360 |
| penugasan | 24 |
| **rumus** | **12** |
| **komponen_rumus** | **~55** |
| **penugasan_rumus** | **30** |
| **nilai** | **~4.440** |
| penilaian_sikap | 720 |
| presensi | ~11.500 |
| rapor · rapor_mapel | 360 · 720 |

Contoh isi `rumus`:

| mapel | tingkat | jenis | komponen |
|---|---|---|---|
| Matematika | X | pengetahuan | T1 T2 UH1 UH2 UTS UAS |
| Matematika | XI | pengetahuan | T1 T2 UH1 UH2 UTS UAS |
| Matematika | XII | pengetahuan | T1 T2 UH1 UH2 **UH3** UTS UAS |
| Fisika | X · XI · XII | pengetahuan | T1 UH1 UH2 UTS UAS |
| Fisika | X · XI · XII | **keterampilan** | Lab1 Lab2 Proyek |
| Ekonomi | X · XI · XII | pengetahuan | T1 UH1 UTS UAS |

Matematika XII punya 7 komponen, Ekonomi 4, Fisika punya dua rumus — **tanpa satu pun `ALTER TABLE`**.

---

## Perhitungan nilai akhir

Rumus **ada di data**, bukan di kode. Satu query, berlaku untuk berapa pun jumlah komponen, tidak pernah diubah:

```sql
SELECT n.siswa_ref,
       ROUND(SUM(n.nilai * k.bobot) / 100.0, 2) AS nilai_akhir
FROM nilai n
JOIN komponen_rumus k ON k.id = n.komponen_ref
WHERE n.penugasan_ref = $1 AND n.rumus_ref = $2
GROUP BY n.siswa_ref;
```

### Versi yang menangani komponen belum terisi

Query di atas **salah diam-diam** kalau ada komponen kosong. Berangkat dari komponen, bukan dari nilai:

```sql
SELECT s.siswa_ref,
       ROUND(SUM(COALESCE(n.nilai,0) * k.bobot) / 100.0, 2) AS nilai_sementara,
       COUNT(n.nilai) = COUNT(k.id)                          AS lengkap
FROM kelas_siswa s
CROSS JOIN komponen_rumus k
LEFT JOIN nilai n ON n.siswa_ref    = s.siswa_ref
                 AND n.komponen_ref = k.id
                 AND n.penugasan_ref = $1
WHERE s.kelas_ref = $2 AND k.rumus_ref = $3
GROUP BY s.siswa_ref;
```

Kolom `lengkap` = state **"Menunggu sinkronisasi nilai"** di layar siswa, sekaligus syarat FR-07 (*generate rapor setelah seluruh nilai lengkap*).

---

## Pivot untuk tabel guru

Data tersimpan memanjang, layar guru menampilkan matriks:

```
Tersimpan (tall)              Ditampilkan (wide)
siswa  komponen  nilai        siswa    T1  T2  UH1  UH2  UTS  UAS
─────  ────────  ─────        ───────  ──  ──  ───  ───  ───  ───
Aditya   T1        85    →    Aditya   85  88   82   84   80   85
Aditya   T2        88         Bunga    90  92   88   90   85   92
```

**Pivot di aplikasi**, bukan di SQL. 30 siswa × 6 komponen = 180 baris — beberapa milidetik. `crosstab()` dari `tablefunc` butuh jumlah kolom diketahui di muka, yang justru mengembalikan masalah statis.

---

## Kenapa dinamis dipilih

| | Statis | Dinamis |
|---|---|---|
| Tambah komponen | `ALTER TABLE` + deploy | `INSERT` |
| Komponen beda antar mapel | ❌ | ✅ |
| Rumus jamak (Pengetahuan + Keterampilan) | ❌ | ✅ |
| Bedakan "tidak ada" vs "belum diisi" | ❌ | ✅ |
| Rumus perhitungan | Di kode | Satu query, tidak pernah berubah |
| Layar Rumus Nilai superadmin bisa dibangun | ❌ | ✅ |
| Kolom `NULL` sia-sia | Banyak | Tidak ada |
| Butuh pivot di aplikasi | Tidak | Ya (~15 baris kode) |

Dua baris terakhir tabel statis (`layar tidak bisa dibangun`, `rumus jamak`) sudah **digambar di wireframe** — jadi dinamis bukan preferensi, tapi syarat.
