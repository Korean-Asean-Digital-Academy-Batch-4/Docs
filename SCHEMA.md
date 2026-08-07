# Skema Fisik — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v1.0 |
| **Tanggal** | 6 Agustus 2026 |
| **Disusun oleh** | Re:Code |
| **Sumber kebenaran** | [PRD.md](PRD.md) v3.0, [ATURAN-DAN-KRITERIA.md](ATURAN-DAN-KRITERIA.md) v1.0, [aktor-role.md](aktor-role.md) v3.0, [RFC-001](RFC-001-model-data-konseptual.md) — terutama §6 Invarian, [Techstack.md](Techstack.md) v2.0, dan [ARCHITECTURE.md](ARCHITECTURE.md) v1.0 |
| **Kedudukan** | Menetapkan **bentuk fisik basis data**: tabel, tipe kolom, constraint, indeks, pemicu, role, dan migrasi |
| **Dokumen lanjutan** | `API.md` — kontrak endpoint |

> Dokumen ini menjawab **bagaimana model data disimpan dan bagaimana setiap invarian ditegakkan**. Entitas dan relasinya ditetapkan [RFC-001](RFC-001-model-data-konseptual.md); mesin basis data dan ORM ditetapkan [Techstack.md](Techstack.md); pembagian tanggung jawab antara basis data dan lapisan aplikasi ditetapkan [ARCHITECTURE.md §1](ARCHITECTURE.md). Dokumen ini tidak mengulang ketiganya.
>
> [RFC-001 §6](RFC-001-model-data-konseptual.md) mewajibkan dokumen ini **menunjukkan cara penegakan setiap invarian**, dan **menyatakan terang-terangan mana yang tidak dapat ditegakkan basis data**. Kewajiban tersebut dipenuhi pada Pasal 5.
>
> Mengikuti [konvensi dokumen](README.md), badan dokumen memuat **deskripsi keadaan yang berlaku** dan disunting langsung ketika berubah, sedangkan **Lampiran Catatan Keputusan** bernomor dan bertanggal serta hanya ditambah, tidak disunting.

---

## 1. Cakupan

### 1.1 Termasuk

Definisi tabel beserta tipe kolom, seluruh constraint dan indeks, dua pemicu, tiga role basis data beserta hak aksesnya, urutan migrasi, dan data awal.

### 1.2 Tidak termasuk

| Hal | Ditetapkan pada |
|---|---|
| Entitas, relasi, dan invarian | [RFC-001](RFC-001-model-data-konseptual.md) |
| Mesin basis data, ORM, dan penyimpanan rahasia | [Techstack.md](Techstack.md) |
| Alur request, pemeriksaan kewenangan, dan pembagian tanggung jawab penegakan | [ARCHITECTURE.md](ARCHITECTURE.md) |
| Kontrak endpoint dan bentuk respons | `API.md` |
| Prosedur penerapan migrasi dan pencadangan | [DEPLOYMENT.md](DEPLOYMENT.md) |

### 1.3 Prinsip

Prinsip ④ pada [ARCHITECTURE.md §1](ARCHITECTURE.md) — **yang dapat dijamin basis data tidak diserahkan kepada disiplin kode** — adalah alasan hampir setiap pilihan pada dokumen ini. Penerapannya terlihat paling jelas pada tiga tempat: composite foreign key pada `penugasan` (§4.3), kolom pembeda peran pada `guru` dan `siswa` (§4.1), dan role `app_ro` tanpa hak tulis sekaligus tanpa hak baca atas identitas (§7).

---

## 2. Konvensi

| Aspek | Ketentuan | Alasan |
|---|---|---|
| **Penamaan** | `snake_case` Bahasa Indonesia, mengikuti nama entitas dan atribut pada [RFC-001 §4](RFC-001-model-data-konseptual.md) | Nama pada model konseptual dan skema fisik dapat ditelusuri satu lawan satu |
| **Pengenal** | `uuid`, dibangkitkan `gen_random_uuid()` — tersedia bawaan pada PostgreSQL 17 tanpa ekstensi | CK-S-01 |
| **Teks** | `text`, dibatasi `CHECK (length(...) <= n)` bila perlu | PostgreSQL tidak membedakan kinerja `text` dan `varchar(n)`; batas panjang menjadi aturan yang dapat diubah tanpa mengubah tipe kolom |
| **Himpunan tertutup** | `text` beserta `CHECK (... IN (...))`, bukan tipe `ENUM` bawaan | CK-S-02 |
| **Waktu** | `timestamptz` untuk momen, `date` untuk tanggal kalender | Sesi presensi melekat pada tanggal kalender sekolah, bukan pada momen dengan zona waktu |
| **Angka nilai** | `numeric(5,2)` untuk nilai dan persentase, `smallint` untuk bobot dan KKM | CK-S-08 |
| **Nama constraint** | `pk_`, `uq_`, `fk_`, `ck_`, `idx_`, `trg_` | Pesan kesalahan PostgreSQL menyebut nama constraint; nama yang berbicara mempercepat penerjemahannya menjadi pesan bagi pengguna (P21) |
| **Kolom waktu** | `dibuat_pada` diisi `DEFAULT now()`; `diperbarui_pada` diisi aplikasi lewat `$onUpdate` Drizzle | Tidak memerlukan pemicu tersendiri untuk pekerjaan yang sudah dilakukan ORM |

Seluruh pengenal berupa `uuid`, sehingga **tidak ada satu pun sequence** di dalam skema. Hal ini menyederhanakan pemberian hak akses pada Pasal 7: tidak diperlukan `GRANT USAGE ON SEQUENCE`.

---

## 3. Peta tabel

**Sembilan belas tabel.** Tujuh belas di antaranya adalah entitas [RFC-001 §4](RFC-001-model-data-konseptual.md); dua sisanya adalah tabel penopang yang lahir dari keputusan arsitektur.

| # | Kelompok | Tabel |
|---|---|---|
| 1 | Identitas | `pengguna`, `guru`, `siswa` |
| 2 | Periode akademik | `tahun_ajaran`, `periode`, `kelas`, `kelas_siswa` |
| 3 | Kurikulum dan penugasan | `mapel`, `penugasan`, `komponen_penilaian`, `penugasan_komponen` |
| 4 | Pencatatan harian | `nilai`, `sesi`, `presensi` |
| 5 | Rapor | `rapor`, `rapor_mapel` |
| 6 | Jejak | `audit_log` |
| 7 | **Penopang** | `sesi_masuk`, `pembatas_laju` |

**Kelompok 7 tidak menambah entitas pada model data.** `sesi_masuk` adalah wujud fisik dari CK-A-04 — sesi berupa baris yang dapat dicabut — dan `pembatas_laju` adalah wujud fisik dari CK-A-03 — penghitung pembatas laju disimpan di PostgreSQL. Keduanya berada di dalam wewenang dokumen ini karena [RFC-001 §3.2](RFC-001-model-data-konseptual.md) menempatkan **mekanisme kredensial** di luar cakupan RFC. Keduanya tidak memuat data akademik dan tidak dapat dibaca jalur AI (Pasal 7).

`pengguna.kata_sandi_hash` juga tidak terdapat pada [RFC-001 §4](RFC-001-model-data-konseptual.md). Kolom ini adalah jawaban langsung atas **K-01** pada [RFC-001 §9](RFC-001-model-data-konseptual.md), yang menyatakan atribut kredensial ditentukan setelah mekanismenya dipilih. Mekanismenya ditutup [Techstack.md §5](Techstack.md): kredensial dikelola sendiri, Argon2id.

---

## 4. Definisi tabel

### 4.1 Identitas

```sql
CREATE TABLE pengguna (
    id              uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    nama_pengguna   text        NOT NULL,
    nama            text        NOT NULL,
    peran           text        NOT NULL,
    kata_sandi_hash text        NOT NULL,
    aktif           boolean     NOT NULL DEFAULT true,
    dibuat_pada     timestamptz NOT NULL DEFAULT now(),
    diperbarui_pada timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT uq_pengguna_id_peran  UNIQUE (id, peran),
    CONSTRAINT ck_pengguna_peran     CHECK (peran IN ('administrator', 'guru', 'siswa')),
    CONSTRAINT ck_pengguna_nama_pengguna CHECK (length(nama_pengguna) BETWEEN 1 AND 32),
    CONSTRAINT ck_pengguna_nama      CHECK (length(nama) BETWEEN 1 AND 128)
);

-- I-02: pengenal masuk unik lintas seluruh pengguna, tidak peka huruf besar-kecil
CREATE UNIQUE INDEX uq_pengguna_nama_pengguna ON pengguna (lower(nama_pengguna));

CREATE TABLE guru (
    pengguna_ref uuid PRIMARY KEY,
    peran        text NOT NULL DEFAULT 'guru',

    CONSTRAINT ck_guru_peran   CHECK (peran = 'guru'),
    CONSTRAINT fk_guru_pengguna FOREIGN KEY (pengguna_ref, peran)
        REFERENCES pengguna (id, peran) ON DELETE RESTRICT
);

CREATE TABLE siswa (
    pengguna_ref uuid PRIMARY KEY,
    peran        text NOT NULL DEFAULT 'siswa',

    CONSTRAINT ck_siswa_peran   CHECK (peran = 'siswa'),
    CONSTRAINT fk_siswa_pengguna FOREIGN KEY (pengguna_ref, peran)
        REFERENCES pengguna (id, peran) ON DELETE RESTRICT
);
```

**Keunikan tidak peka huruf besar-kecil.** NIP dan NIS berupa angka sehingga tidak terpengaruh. Yang dilindungi adalah nama pengguna Administrator, agar `admin` dan `Admin` tidak menjadi dua akun berbeda yang sulit dibedakan saat penyerahan kata sandi.

**Kolom `peran` pada `guru` dan `siswa` tidak menyimpan informasi baru.** Nilainya tetap dan dijaga `CHECK`; kegunaannya semata-mata menjadi bagian kedua dari composite foreign key ke `pengguna (id, peran)`. Akibatnya, baris `guru` **tidak dapat dibuat** untuk pengguna berperan `siswa`, dan sebaliknya. Karena `mapel.guru_ref` dan `kelas.wali_kelas_ref` menunjuk `guru`, penetapan seorang siswa sebagai guru pengampu maupun wali kelas menjadi mustahil di tingkat data — persis maksud yang dinyatakan [RFC-001 §7.1](RFC-001-model-data-konseptual.md), kini ditegakkan dan bukan sekadar diharapkan (CK-S-04).

### 4.2 Periode akademik

```sql
CREATE TABLE tahun_ajaran (
    id          uuid    PRIMARY KEY DEFAULT gen_random_uuid(),
    nama        text    NOT NULL,
    tgl_mulai   date    NOT NULL,
    tgl_selesai date    NOT NULL,
    aktif       boolean NOT NULL DEFAULT false,

    CONSTRAINT uq_tahun_ajaran_nama    UNIQUE (nama),
    CONSTRAINT ck_tahun_ajaran_rentang CHECK (tgl_selesai > tgl_mulai)
);

CREATE TABLE periode (
    id               uuid    PRIMARY KEY DEFAULT gen_random_uuid(),
    tahun_ajaran_ref uuid    NOT NULL REFERENCES tahun_ajaran (id) ON DELETE RESTRICT,
    semester         text    NOT NULL,
    tgl_mulai        date    NOT NULL,
    tgl_selesai      date    NOT NULL,
    aktif            boolean NOT NULL DEFAULT false,

    CONSTRAINT uq_periode_tahun_semester UNIQUE (tahun_ajaran_ref, semester),
    CONSTRAINT ck_periode_semester       CHECK (semester IN ('ganjil', 'genap')),
    CONSTRAINT ck_periode_rentang        CHECK (tgl_selesai > tgl_mulai)
);

-- I-03: satu tahun ajaran memiliki paling banyak satu semester aktif
CREATE UNIQUE INDEX uq_periode_aktif_per_tahun ON periode (tahun_ajaran_ref) WHERE aktif;

CREATE TABLE kelas (
    id             uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    periode_ref    uuid        NOT NULL REFERENCES periode (id) ON DELETE RESTRICT,
    nama           text        NOT NULL,
    tingkat        text        NOT NULL,
    jurusan        text,
    wali_kelas_ref uuid        REFERENCES guru (pengguna_ref) ON DELETE RESTRICT,
    dibuat_pada    timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT uq_kelas_periode_nama UNIQUE (periode_ref, nama),
    CONSTRAINT uq_kelas_id_tingkat   UNIQUE (id, tingkat),
    CONSTRAINT uq_kelas_id_periode   UNIQUE (id, periode_ref),
    CONSTRAINT ck_kelas_tingkat      CHECK (tingkat IN ('X', 'XI', 'XII')),
    CONSTRAINT ck_kelas_nama         CHECK (length(nama) BETWEEN 1 AND 32)
);

-- aktor-role §12 butir 1: satu Guru menjadi wali paling banyak satu kelas per periode
CREATE UNIQUE INDEX uq_kelas_wali_per_periode
    ON kelas (periode_ref, wali_kelas_ref) WHERE wali_kelas_ref IS NOT NULL;

CREATE TABLE kelas_siswa (
    id          uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    kelas_ref   uuid        NOT NULL,
    siswa_ref   uuid        NOT NULL REFERENCES siswa (pengguna_ref) ON DELETE RESTRICT,
    periode_ref uuid        NOT NULL,
    dibuat_pada timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT fk_kelas_siswa_kelas FOREIGN KEY (kelas_ref, periode_ref)
        REFERENCES kelas (id, periode_ref) ON DELETE RESTRICT,
    CONSTRAINT uq_kelas_siswa_periode UNIQUE (siswa_ref, periode_ref),
    CONSTRAINT uq_kelas_siswa_kelas   UNIQUE (kelas_ref, siswa_ref)
);

CREATE INDEX idx_kelas_siswa_kelas ON kelas_siswa (kelas_ref);
```

**`kelas_siswa.periode_ref` adalah penopang I-08, sebagaimana dicatat [RFC-001 §4](RFC-001-model-data-konseptual.md).** Cara kerjanya: `fk_kelas_siswa_kelas` mengunci `periode_ref` agar selalu sama dengan periode kelasnya, sehingga kolom itu tidak dapat menyimpang; `uq_kelas_siswa_periode` kemudian menjamin satu siswa hanya memiliki satu baris per periode. Keduanya bersama-sama menjadikan "satu siswa berada pada dua kelas dalam satu semester" sebagai keadaan yang **tidak dapat tersimpan**.

**`tahun_ajaran.aktif` sengaja tidak dibatasi partial unique index.** P19 hanya membatasi jumlah semester aktif per tahun ajaran, bukan jumlah tahun ajaran aktif. Membatasinya akan menghalangi penyiapan tahun berikutnya selagi tahun berjalan masih aktif — pembatasan yang tidak diminta dokumen mana pun.

**`uq_kelas_wali_per_periode` menegakkan ketentuan sekolah.** Sekolah menyatakan pada 7 Agustus 2026 bahwa satu Guru menjadi Wali Kelas paling banyak satu kelas, menutup butir 1 pada [aktor-role.md §12](aktor-role.md) dan temuan **S-02** pada Pasal 12. Indeks ini karenanya bukan lagi penegak asumsi melainkan penegak ketentuan, dan tidak dijatuhkan.

### 4.3 Kurikulum dan penugasan

```sql
CREATE TABLE mapel (
    id              uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    kode            text        NOT NULL,
    nama            text        NOT NULL,
    tingkat         text        NOT NULL,
    kkm             smallint    NOT NULL DEFAULT 75,
    guru_ref        uuid        NOT NULL REFERENCES guru (pengguna_ref) ON DELETE RESTRICT,
    dibuat_pada     timestamptz NOT NULL DEFAULT now(),
    diperbarui_pada timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT uq_mapel_kode       UNIQUE (kode),
    CONSTRAINT uq_mapel_guru       UNIQUE (guru_ref),
    CONSTRAINT uq_mapel_id_tingkat UNIQUE (id, tingkat),
    CONSTRAINT uq_mapel_id_guru    UNIQUE (id, guru_ref),
    CONSTRAINT ck_mapel_tingkat    CHECK (tingkat IN ('X', 'XI', 'XII')),
    CONSTRAINT ck_mapel_kkm        CHECK (kkm BETWEEN 0 AND 100),
    CONSTRAINT ck_mapel_nama       CHECK (length(nama) BETWEEN 1 AND 64)
);

CREATE TABLE penugasan (
    id          uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    guru_ref    uuid        NOT NULL,
    mapel_ref   uuid        NOT NULL,
    kelas_ref   uuid        NOT NULL,
    tingkat     text        NOT NULL,
    dibuat_pada timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT fk_penugasan_mapel_tingkat FOREIGN KEY (mapel_ref, tingkat)
        REFERENCES mapel (id, tingkat) ON DELETE RESTRICT,
    CONSTRAINT fk_penugasan_kelas_tingkat FOREIGN KEY (kelas_ref, tingkat)
        REFERENCES kelas (id, tingkat) ON DELETE RESTRICT,
    CONSTRAINT fk_penugasan_mapel_guru FOREIGN KEY (mapel_ref, guru_ref)
        REFERENCES mapel (id, guru_ref) ON DELETE RESTRICT,
    CONSTRAINT uq_penugasan_kelas_mapel UNIQUE (kelas_ref, mapel_ref)
);

CREATE INDEX idx_penugasan_guru  ON penugasan (guru_ref);
CREATE INDEX idx_penugasan_kelas ON penugasan (kelas_ref);

CREATE TABLE komponen_penilaian (
    id     uuid     PRIMARY KEY DEFAULT gen_random_uuid(),
    kode   text     NOT NULL,
    nama   text     NOT NULL,
    bobot  smallint NOT NULL,
    urutan smallint NOT NULL,

    CONSTRAINT uq_komponen_kode   UNIQUE (kode),
    CONSTRAINT uq_komponen_urutan UNIQUE (urutan),
    CONSTRAINT ck_komponen_bobot  CHECK (bobot > 0 AND bobot <= 100)
);

CREATE TABLE penugasan_komponen (
    penugasan_ref uuid NOT NULL REFERENCES penugasan (id) ON DELETE RESTRICT,
    komponen_ref  uuid NOT NULL REFERENCES komponen_penilaian (id) ON DELETE RESTRICT,
    topik         text,

    CONSTRAINT pk_penugasan_komponen PRIMARY KEY (penugasan_ref, komponen_ref),
    CONSTRAINT ck_penugasan_komponen_topik CHECK (topik IS NULL OR length(topik) <= 200)
);
```

**Tiga composite foreign key pada `penugasan` adalah inti dokumen ini.** [RFC-001 §5.3](RFC-001-model-data-konseptual.md) menyatakan `tingkat` bersifat redundan secara logika dan menuntut konsistensinya **dijamin basis data, bukan disiplin kode**, dengan cara penjaminannya ditetapkan dokumen ini. Inilah caranya:

| Constraint | Yang menjadi mustahil |
|---|---|
| `fk_penugasan_mapel_tingkat` + `fk_penugasan_kelas_tingkat` | Kelas berjenjang X dihubungkan dengan mata pelajaran berjenjang XI. Karena satu kolom `tingkat` menjadi bagian dari kedua foreign key, kedua sisi wajib menunjuk jenjang yang sama (I-06, AC-24) |
| `fk_penugasan_mapel_guru` | `penugasan.guru_ref` menyimpang dari `mapel.guru_ref`. Penugasan hanya sah bila gurunya memang pengampu mata pelajaran tersebut (I-07) |
| `uq_penugasan_kelas_mapel` | Satu mata pelajaran diberikan dua kali pada satu kelas |

Ketiganya bersama-sama membuat penolakan AC-24 tidak bergantung pada satu pun baris kode: kekeliruan di jalur mana pun, termasuk perbaikan data manual, tetap ditolak PostgreSQL.

**`penugasan` tidak menyimpan rujukan periode**, sesuai [RFC-001 §5.3](RFC-001-model-data-konseptual.md). Periode diperoleh melalui `kelas.periode_ref`.

### 4.4 Pencatatan harian

```sql
CREATE TABLE nilai (
    id              uuid         PRIMARY KEY DEFAULT gen_random_uuid(),
    penugasan_ref   uuid         NOT NULL REFERENCES penugasan (id) ON DELETE RESTRICT,
    komponen_ref    uuid         NOT NULL REFERENCES komponen_penilaian (id) ON DELETE RESTRICT,
    siswa_ref       uuid         NOT NULL REFERENCES siswa (pengguna_ref) ON DELETE RESTRICT,
    nilai           numeric(5,2) NOT NULL,
    diperbarui_oleh uuid         NOT NULL REFERENCES pengguna (id) ON DELETE RESTRICT,
    dibuat_pada     timestamptz  NOT NULL DEFAULT now(),
    diperbarui_pada timestamptz  NOT NULL DEFAULT now(),

    CONSTRAINT uq_nilai         UNIQUE (penugasan_ref, komponen_ref, siswa_ref),
    CONSTRAINT ck_nilai_rentang CHECK (nilai >= 0 AND nilai <= 100)
);

CREATE INDEX idx_nilai_penugasan_siswa ON nilai (penugasan_ref, siswa_ref);
CREATE INDEX idx_nilai_siswa           ON nilai (siswa_ref);

CREATE TABLE sesi (
    id            uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    penugasan_ref uuid        NOT NULL REFERENCES penugasan (id) ON DELETE RESTRICT,
    tanggal       date        NOT NULL,
    dibuka_oleh   uuid        NOT NULL REFERENCES pengguna (id) ON DELETE RESTRICT,
    dibuat_pada   timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT uq_sesi_penugasan_tanggal UNIQUE (penugasan_ref, tanggal)
);

CREATE TABLE presensi (
    id              uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    sesi_ref        uuid        NOT NULL REFERENCES sesi (id) ON DELETE CASCADE,
    siswa_ref       uuid        NOT NULL REFERENCES siswa (pengguna_ref) ON DELETE RESTRICT,
    status          text        NOT NULL DEFAULT 'alpa',
    catatan         text,
    diperbarui_oleh uuid        NOT NULL REFERENCES pengguna (id) ON DELETE RESTRICT,
    diperbarui_pada timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT uq_presensi         UNIQUE (sesi_ref, siswa_ref),
    CONSTRAINT ck_presensi_status  CHECK (status IN ('hadir', 'izin', 'sakit', 'alpa')),
    CONSTRAINT ck_presensi_catatan CHECK (catatan IS NULL OR length(catatan) <= 200)
);

CREATE INDEX idx_presensi_siswa ON presensi (siswa_ref);
```

**`nilai.nilai` bersifat `NOT NULL`, dan inilah penegakan I-12.** Baris `nilai` hanya ada apabila nilainya terisi. Menghapus isian berarti menghapus baris, sehingga "belum lengkap" memiliki satu representasi tunggal — ketiadaan baris — dan tidak ada nilai kosong yang ambigu. Kolom nullable akan menghasilkan dua representasi yang berarti sama, yang justru dilarang [RFC-001 §6.1](RFC-001-model-data-konseptual.md). Jalur penghapusannya berada pada langkah 10 alur [ARCHITECTURE.md §14.1](ARCHITECTURE.md).

**`presensi.status` berdefault `alpa`**, sesuai P6 dan AC-11: sesi selalu terbuka dengan seluruh siswa berstatus Alpa. Default ini menjadikan transaksi pembukaan sesi cukup menyisipkan pasangan `(sesi_ref, siswa_ref)` tanpa menyebut status.

**`ON DELETE CASCADE` hanya satu di seluruh skema**, yaitu dari `sesi` ke `presensi` (I-16, C-03, AC-25). Selebihnya `RESTRICT`; rinciannya pada Pasal 6.

### 4.5 Rapor

```sql
CREATE TABLE rapor (
    id                   uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    siswa_ref            uuid        NOT NULL REFERENCES siswa (pengguna_ref) ON DELETE RESTRICT,
    kelas_ref            uuid        NOT NULL,
    periode_ref          uuid        NOT NULL,
    status               text        NOT NULL DEFAULT 'draft',
    catatan_wali         text,
    difinalisasi_oleh    uuid        REFERENCES pengguna (id) ON DELETE RESTRICT,
    difinalisasi_pada    timestamptz,
    didistribusikan_pada timestamptz,
    kunci_berkas         text,
    dibuat_pada          timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT fk_rapor_kelas FOREIGN KEY (kelas_ref, periode_ref)
        REFERENCES kelas (id, periode_ref) ON DELETE RESTRICT,
    CONSTRAINT uq_rapor_siswa_periode UNIQUE (siswa_ref, periode_ref),
    CONSTRAINT ck_rapor_status  CHECK (status IN ('draft', 'finalized', 'distributed')),
    CONSTRAINT ck_rapor_catatan CHECK (catatan_wali IS NULL OR length(catatan_wali) <= 1000),
    CONSTRAINT ck_rapor_finalisasi CHECK (
        (status = 'draft'
            AND difinalisasi_pada IS NULL AND difinalisasi_oleh IS NULL)
        OR (status <> 'draft'
            AND difinalisasi_pada IS NOT NULL AND difinalisasi_oleh IS NOT NULL)
    ),
    CONSTRAINT ck_rapor_distribusi CHECK (
        (status = 'distributed') = (didistribusikan_pada IS NOT NULL)
    )
);

CREATE INDEX idx_rapor_kelas ON rapor (kelas_ref);

CREATE TABLE rapor_mapel (
    id                uuid         PRIMARY KEY DEFAULT gen_random_uuid(),
    rapor_ref         uuid         NOT NULL REFERENCES rapor (id) ON DELETE CASCADE,
    mapel_nama        text         NOT NULL,
    kkm               smallint     NOT NULL,
    nilai_akhir       numeric(5,2) NOT NULL,
    kehadiran_persen  numeric(5,2) NOT NULL,
    snapshot_komponen jsonb        NOT NULL,

    CONSTRAINT uq_rapor_mapel       UNIQUE (rapor_ref, mapel_nama),
    CONSTRAINT ck_rapor_mapel_kkm   CHECK (kkm BETWEEN 0 AND 100),
    CONSTRAINT ck_rapor_mapel_nilai CHECK (nilai_akhir BETWEEN 0 AND 100),
    CONSTRAINT ck_rapor_mapel_hadir CHECK (kehadiran_persen BETWEEN 0 AND 100),
    CONSTRAINT ck_rapor_mapel_snapshot CHECK (jsonb_typeof(snapshot_komponen) = 'array')
);
```

**Dua `CHECK` konsistensi status menutup celah yang tidak terlihat.** Tanpa `ck_rapor_finalisasi`, sebuah rapor dapat berstatus `finalized` tanpa diketahui siapa yang memfinalisasi dan kapan — keadaan yang tidak melanggar constraint mana pun tetapi membuat rapor kehilangan pertanggungjawabannya. Tanpa `ck_rapor_distribusi`, `didistribusikan_pada` dapat terisi pada rapor yang belum didistribusikan.

**`fk_rapor_kelas` bersifat komposit** agar `periode_ref` pada rapor selalu sama dengan periode kelasnya. Tanpa itu, rapor dapat menunjuk kelas semester ganjil sekaligus periode semester genap, dan `uq_rapor_siswa_periode` yang menegakkan I-19 kehilangan artinya.

**`snapshot_komponen` berbentuk larik JSONB** berisi salinan beku kode, nama, bobot, dan nilai setiap komponen pada saat finalisasi ([RFC-001 §5.5](RFC-001-model-data-konseptual.md)):

```json
[
  { "kode": "T1",  "nama": "Tugas 1",              "bobot": 6,  "nilai": 85.00, "topik": "Sel dan jaringan" },
  { "kode": "UAS", "nama": "Ujian Akhir Semester", "bobot": 26, "nilai": 78.50, "topik": null }
]
```

Bentuknya sengaja tidak dinormalisasi menjadi tabel tersendiri: isinya **tidak pernah dikueri sebagai data**, melainkan dibaca utuh pada saat render PDF ([ARCHITECTURE.md §11](ARCHITECTURE.md)). Karena itu tidak diperlukan indeks GIN maupun skema JSON di tingkat basis data; bentuknya divalidasi Zod di batas aplikasi, sejalan dengan [ARCHITECTURE.md §12](ARCHITECTURE.md).

### 4.6 Jejak

```sql
CREATE TABLE audit_log (
    id           uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    pengguna_ref uuid        NOT NULL REFERENCES pengguna (id) ON DELETE RESTRICT,
    judul        text        NOT NULL,
    deskripsi    text,
    aksi         text        NOT NULL,
    entitas      text        NOT NULL,
    entitas_ref  uuid,
    sebelum      jsonb,
    sesudah      jsonb,
    severity     text        NOT NULL,
    alamat_ip    inet,
    dibuat_pada  timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT ck_audit_severity CHECK (severity IN ('success', 'warning', 'failed'))
);

CREATE INDEX idx_audit_log_waktu ON audit_log (dibuat_pada DESC);
```

**Tabel ini dibangun tetapi tidak ditulis sepanjang MVP** (CK-A-06). Ia tetap dibangun karena [RFC-001 D-07](RFC-001-model-data-konseptual.md) masih mempertahankannya pada model data, dan pencabutannya hanya dapat dilakukan melalui amandemen RFC. Membangunnya sekarang berbiaya nol pada basis data kosong; membangunnya kelak berarti migrasi pada basis data berisi data sekolah.

`alamat_ip` memakai tipe `inet`, bukan `text`, sehingga alamat yang tidak sah ditolak dan IPv4 maupun IPv6 tertampung tanpa perlakuan berbeda.

### 4.7 Tabel penopang

```sql
CREATE TABLE sesi_masuk (
    token_hash       text        PRIMARY KEY,
    pengguna_ref     uuid        NOT NULL REFERENCES pengguna (id) ON DELETE CASCADE,
    dibuat_pada      timestamptz NOT NULL DEFAULT now(),
    kedaluwarsa_pada timestamptz NOT NULL,

    CONSTRAINT ck_sesi_masuk_umur CHECK (kedaluwarsa_pada > dibuat_pada)
);

CREATE INDEX idx_sesi_masuk_pengguna     ON sesi_masuk (pengguna_ref);
CREATE INDEX idx_sesi_masuk_kedaluwarsa  ON sesi_masuk (kedaluwarsa_pada);

CREATE TABLE pembatas_laju (
    kunci         text        NOT NULL,
    jendela_mulai timestamptz NOT NULL,
    jumlah        integer     NOT NULL DEFAULT 0,

    CONSTRAINT pk_pembatas_laju PRIMARY KEY (kunci, jendela_mulai),
    CONSTRAINT ck_pembatas_laju_jumlah CHECK (jumlah >= 0)
);
```

**Yang disimpan adalah hash token, bukan tokennya.** Token sesi berupa nilai acak 256 bit; yang tersimpan adalah SHA-256 atasnya. Akibatnya, salinan basis data — cadangan, dump pengembangan, maupun basis data yang bocor — **tidak memuat satu pun sesi yang dapat dipakai masuk**. Argon2id tidak diperlukan di sini karena masukannya sudah acak penuh sehingga tidak dapat ditebak, berbeda dari kata sandi buatan manusia (CK-S-06).

`ON DELETE CASCADE` pada `sesi_masuk` adalah satu-satunya cascade selain `sesi → presensi`, dan bersifat kepemilikan: sesi tidak memiliki arti tanpa penggunanya. Pencabutan sesi pada penggantian kata sandi (CK-A-04) dilakukan dengan menghapus baris berdasarkan `idx_sesi_masuk_pengguna`.

**`pembatas_laju` dibersihkan sendiri tanpa pekerjaan latar.** Karena tidak ada worker maupun penjadwal (CK-07), setiap penulisan penghitung menghapus jendela yang sudah lewat untuk kunci yang sama di dalam pernyataan yang sama. Tabel dengan demikian tetap kecil tanpa proses tambahan:

```sql
DELETE FROM pembatas_laju WHERE kunci = $1 AND jendela_mulai < $2;
```

`kunci` menggabungkan jalur dan subjeknya — misalnya `login:pengguna:<id>`, `login:ip:<alamat>`, `suggestion:<id>`, `unggah:<id>` — sesuai tiga jalur terbatas pada [ARCHITECTURE.md §7](ARCHITECTURE.md).

---

## 5. Penegakan invarian

Pasal ini adalah pertanggungjawaban langsung dokumen ini terhadap [RFC-001 §6](RFC-001-model-data-konseptual.md), yang menuntut setiap invarian ditunjukkan cara penegakannya, dan yang tidak dapat ditegakkan basis data dinyatakan terang-terangan.

### 5.1 Peta lengkap

| # | Invarian | Ditegakkan | Mekanisme |
|---|---|:--:|---|
| I-01 | Satu pengguna satu peran | 🗄️ | `pengguna.peran NOT NULL` + `ck_pengguna_peran`; `fk_guru_pengguna` dan `fk_siswa_pengguna` mencegah profil lintas peran |
| I-02 | Pengenal masuk unik | 🗄️ | `uq_pengguna_nama_pengguna` atas `lower(nama_pengguna)` |
| I-03 | Satu semester aktif per tahun ajaran | 🗄️ | Partial unique index `uq_periode_aktif_per_tahun` |
| I-04 | Mapel tepat satu jenjang dan satu guru | 🗄️ | `mapel.tingkat NOT NULL`, `mapel.guru_ref NOT NULL` |
| I-05 | Satu guru paling banyak satu mapel | 🗄️ | `uq_mapel_guru` |
| I-06 | Jenjang kelas sama dengan jenjang mapel | 🗄️ | `fk_penugasan_mapel_tingkat` + `fk_penugasan_kelas_tingkat` berbagi kolom `tingkat` |
| I-07 | Satu mapel pada satu kelas diajar satu guru | 🗄️ | `fk_penugasan_mapel_guru` + `uq_penugasan_kelas_mapel` |
| I-08 | Satu siswa satu kelas per semester | 🗄️ | `fk_kelas_siswa_kelas` komposit + `uq_kelas_siswa_periode` |
| I-09 | Satu kelas paling banyak satu wali | 🗄️ | Bentuk kolom `kelas.wali_kelas_ref` — tidak diperlukan constraint tambahan |
| I-10 | Jumlah bobot tepat 100 | 🗄️ + ⚙️ | `trg_komponen_bobot` (§5.2) sebagai penjamin; `domain/nilai.ts` sebagai penghasil pesan |
| I-11 | KKM awal 75, dapat diubah Administrator | 🗄️ | `DEFAULT 75` + `ck_mapel_kkm` |
| I-12 | Nilai kosong terbedakan dari nol | 🗄️ + ⚙️ | `nilai.nilai NOT NULL`; jalur hapus baris pada [ARCHITECTURE §14.1](ARCHITECTURE.md) langkah 10 |
| I-13 | Satu nilai per komponen per penugasan | 🗄️ | `uq_nilai` |
| I-14 | Satu sesi per penugasan per tanggal | 🗄️ | `uq_sesi_penugasan_tanggal` |
| I-15 | Setiap siswa tepat satu status per sesi | 🗄️ + ⚙️ | Batas atas: `uq_presensi`. Kelengkapan: transaksi pembukaan sesi menyisipkan seluruh anggota kelas |
| I-16 | Hapus sesi menghapus seluruh statusnya | 🗄️ | `ON DELETE CASCADE` pada `presensi.sesi_ref` |
| I-17 | Izin dan Sakit terhitung hadir | ⚙️ | Rumus pada `domain/presensi.ts`. Bukan aturan penyimpanan |
| I-18 | Penyebut kehadiran adalah jumlah sesi | ⚙️ | Kueri agregat §8.3 |
| I-19 | Satu rapor per siswa per semester | 🗄️ | `uq_rapor_siswa_periode` + `fk_rapor_kelas` komposit |
| I-20 | Finalisasi hanya bila seluruh mapel lengkap | ⚙️ | Kueri kelengkapan §8.2 di dalam transaksi finalisasi |
| I-21 | Status rapor hanya bergerak maju | 🗄️ | `trg_rapor_status_maju` (§5.2) |
| I-22 | Rapor final terkunci bagi Guru dan Wali Kelas | ⚙️ | Pemeriksaan status pada lapisan rute; basis data tidak mengetahui aktor |
| I-23 | AI tidak pernah menulis data akademik | 🗄️ | Role `app_ro` tanpa hak tulis (Pasal 7) |
| I-24 | Keluaran AI tidak disimpan | 🗄️ | Ketiadaan tabel. Tidak ada tempat untuk menyimpannya |
| I-25 | Siswa hanya membaca datanya sendiri | ⚙️ | Lapis baris pada [ARCHITECTURE §9.2](ARCHITECTURE.md) |

🗄️ basis data · ⚙️ lapisan aplikasi

**Tujuh belas invarian ditegakkan basis data, delapan menyentuh lapisan aplikasi, dan tiga di antaranya terbelah.** Lima invarian yang **sepenuhnya** menjadi tanggung jawab aplikasi — I-17, I-18, I-20, I-22, dan I-25 — adalah tempat kekeliruan kode dapat menghasilkan data atau tampilan yang salah tanpa ditolak siapa pun. Kelimanya menjadi sasaran utama pengujian integrasi, sebagaimana dituntut [ARCHITECTURE.md §1.2](ARCHITECTURE.md).

### 5.2 Dua penguatan terhadap ARCHITECTURE §1.2

[ARCHITECTURE.md §1.2](ARCHITECTURE.md) menempatkan **I-10** di lapisan aplikasi, dan tidak menyebut **I-21** pada tabel mana pun. Dokumen ini memindahkan keduanya ke basis data melalui dua pemicu. Ini **menambah**, bukan mengganti: validasi aplikasi tetap ada dan tetap menjadi penghasil pesan bagi pengguna. Dasarnya adalah prinsip ④ — apa yang dapat dijamin basis data tidak diserahkan kepada disiplin kode (CK-S-05).

```sql
-- I-10: jumlah bobot seluruh komponen penilaian tepat 100
CREATE FUNCTION jaga_jumlah_bobot() RETURNS trigger
LANGUAGE plpgsql AS $$
DECLARE
    total integer;
BEGIN
    SELECT coalesce(sum(bobot), 0) INTO total FROM komponen_penilaian;
    IF total <> 100 THEN
        RAISE EXCEPTION 'Jumlah bobot komponen penilaian harus tepat 100, saat ini %', total
            USING ERRCODE = 'check_violation';
    END IF;
    RETURN NULL;
END;
$$;

CREATE CONSTRAINT TRIGGER trg_komponen_bobot
    AFTER INSERT OR UPDATE OR DELETE ON komponen_penilaian
    DEFERRABLE INITIALLY DEFERRED
    FOR EACH ROW EXECUTE FUNCTION jaga_jumlah_bobot();
```

**`DEFERRABLE INITIALLY DEFERRED` adalah bagian yang menentukan.** Penyesuaian bobot selalu menyentuh beberapa baris sekaligus — menaikkan UTS berarti menurunkan komponen lain. Pemeriksaan yang berjalan per baris akan menolak langkah pertama dari perubahan yang sah. Dengan penangguhan sampai `COMMIT`, yang diperiksa adalah **keadaan akhir transaksi**, sehingga penyesuaian sah lolos dan penyesuaian yang meleset ditolak. Pesan kesalahannya menyebutkan total saat ini, persis seperti yang dituntut AC-04.

```sql
-- I-21: status rapor hanya bergerak maju
CREATE FUNCTION urutan_status_rapor(s text) RETURNS integer
LANGUAGE sql IMMUTABLE AS $$
    SELECT CASE s WHEN 'draft' THEN 1 WHEN 'finalized' THEN 2 WHEN 'distributed' THEN 3 END;
$$;

CREATE FUNCTION cegah_status_mundur() RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
    IF urutan_status_rapor(NEW.status) < urutan_status_rapor(OLD.status) THEN
        RAISE EXCEPTION 'Status rapor hanya bergerak maju, tidak dapat kembali dari % ke %',
            OLD.status, NEW.status
            USING ERRCODE = 'check_violation';
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_rapor_status_maju
    BEFORE UPDATE OF status ON rapor
    FOR EACH ROW WHEN (NEW.status IS DISTINCT FROM OLD.status)
    EXECUTE FUNCTION cegah_status_mundur();
```

Pemicu ini menjadikan "tidak tersedia mekanisme buka kembali" ([PRD §9](PRD.md), [RFC-001 §5.5](RFC-001-model-data-konseptual.md)) sebagai keadaan yang **tidak dapat tersimpan**, dan bukan fitur yang kebetulan belum dibangun. Perbedaannya muncul ketika kelak seseorang menambahkan endpoint pembatalan finalisasi tanpa membaca PRD.

Kedua pemicu ditulis sebagai berkas SQL migrasi tersendiri. Drizzle tidak membangkitkan pemicu, tetapi migrasinya memang berupa SQL yang dapat dibaca dan ditinjau ([Techstack.md §4.2](Techstack.md)), sehingga pemicu ditempatkan di sana tanpa jalan memutar.

---

## 6. Perilaku penghapusan

| Dari | Ke | Perilaku | Alasan |
|---|---|---|---|
| `sesi` | `presensi` | **CASCADE** | I-16 dan AC-25 — penghapusan sesi menghapus seluruh status di dalamnya |
| `pengguna` | `sesi_masuk` | **CASCADE** | Kepemilikan. Sesi masuk tidak memiliki arti tanpa penggunanya |
| `rapor` | `rapor_mapel` | **CASCADE** | Kepemilikan. Baris rapor mata pelajaran adalah bagian dari satu rapor |
| Seluruh sisanya | — | **RESTRICT** | Bawaan |

**`RESTRICT` adalah bawaan yang disengaja.** Selain sesi presensi, MVP tidak memiliki satu pun alur penghapusan: [ATURAN-DAN-KRITERIA §3](ATURAN-DAN-KRITERIA.md) tidak memuat layar penghapusan akun, kelas, mata pelajaran, maupun penugasan, dan akun dinonaktifkan lewat `pengguna.aktif`, bukan dihapus. Dengan `RESTRICT`, upaya penghapusan yang tidak dirancang akan **gagal dengan berisik** alih-alih menghapus nilai satu semester secara diam-diam.

Konsekuensi yang diterima: penghapusan data uji memerlukan urutan yang benar, atau `TRUNCATE ... CASCADE` pada lingkungan pengembangan. Ini harga yang murah dibanding kehilangan data yang tidak disengaja pada basis data sekolah.

---

## 7. Role dan hak akses

Tiga role, bukan dua. Migrasi memerlukan role tersendiri yang memiliki seluruh objek, dan kredensialnya tercatat pada [Techstack.md §7](Techstack.md) dengan perlakuan sama seperti `app_rw`. Pembacanya dibatasi role IAM `edutrack-lambda-migrate` saja ([DEPLOYMENT.md §9.5](DEPLOYMENT.md)), sehingga fungsi `api` tidak dapat mengambilnya.

| Role | Dipakai oleh | Hak |
|---|---|---|
| `edutrack_owner` | Fungsi `migrate` | Pemilik seluruh objek. Satu-satunya yang boleh DDL |
| `app_rw` | Seluruh jalur aplikasi | Baca-tulis pada data operasional |
| `app_ro` | **Hanya jalur AI** | Baca saja, dan tidak atas seluruh tabel |

```sql
-- Kedua role aplikasi dibuat lebih dahulu; GRANT di bawahnya mustahil tanpa keduanya ada.
-- Dibuat TANPA kata sandi: nilainya ditetapkan di luar migrasi (Techstack §7 butir 1),
-- sehingga tidak ada satu pun kata sandi di dalam repositori. Role LOGIN tanpa kata sandi
-- tidak dapat diautentikasi scram, sehingga keadaan antara tidak membuka apa pun.
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'app_rw') THEN
        CREATE ROLE app_rw LOGIN;
    END IF;
    IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'app_ro') THEN
        CREATE ROLE app_ro LOGIN;
    END IF;
END
$$;

-- Tidak ada hak apa pun yang diberikan secara diam-diam
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT USAGE ON SCHEMA public TO app_rw, app_ro;

-- Jalur tulis aplikasi
GRANT SELECT, INSERT, UPDATE, DELETE ON
    pengguna, guru, siswa,
    tahun_ajaran, periode, kelas, kelas_siswa,
    mapel, penugasan, komponen_penilaian, penugasan_komponen,
    nilai, sesi, presensi,
    rapor, rapor_mapel,
    sesi_masuk, pembatas_laju
TO app_rw;
GRANT SELECT, INSERT ON audit_log TO app_rw;

-- Jalur AI: hanya membaca, dan hanya yang diperlukan menyusun prompt
GRANT SELECT ON
    nilai, presensi, sesi,
    mapel, penugasan, penugasan_komponen, komponen_penilaian,
    kelas, kelas_siswa, periode, tahun_ajaran
TO app_ro;
-- TIDAK ADA INSERT, UPDATE, maupun DELETE. Sama sekali.

-- Tabel yang tidak disebutkan tetap tertutup bagi app_ro:
--   pengguna, guru, siswa      → identitas
--   rapor, rapor_mapel         → di luar cakupan tombol Suggestion
--   audit_log, sesi_masuk, pembatas_laju
```

**`edutrack_owner` tidak dibuat migrasi.** Ia adalah role yang *menjalankan* migrasi, sehingga sudah ada sebelum pernyataan pertama dieksekusi. Yang dibuat migrasi hanyalah kedua role aplikasi (CK-S-09).

**Penjaga `IF NOT EXISTS` bukan hiasan.** Role di PostgreSQL bersifat lintas basis data dalam satu cluster, bukan milik satu basis data. Tanpa penjaga tersebut, penerapan migrasi pada basis data kedua di cluster yang sama — lingkungan uji di samping lingkungan pengembangan — gagal pada `CREATE ROLE` yang sudah ada.

### 7.1 Dua jaminan dari satu role

Ketiadaan hak tulis menegakkan **I-23**, dan cara membuktikannya berupa satu kueri yang gagal, sebagaimana diuraikan [ARCHITECTURE.md §8](ARCHITECTURE.md).

Yang belum dinyatakan dokumen sebelumnya adalah jaminan kedua. **`app_ro` tidak memiliki hak baca atas `pengguna`, `guru`, maupun `siswa`.** Karena nama dan pengenal masuk hanya berada di `pengguna`, sementara `siswa` hanya memuat satu kolom rujukan, jalur AI **secara struktural tidak dapat memperoleh identitas siapa pun** — bahkan seandainya penyusun prompt keliru menyertakannya.

Ini mengubah kedudukan ketentuan "identitas siswa tidak pernah dikirim" pada [Techstack.md §6](Techstack.md) dan [ARCHITECTURE.md §10.1](ARCHITECTURE.md): dari disiplin penyusunan prompt menjadi **batas yang ditegakkan basis data**. Pembuktiannya setara dengan pembuktian AC-20:

```sql
-- dijalankan sebagai app_ro
SELECT nama FROM pengguna LIMIT 1;
-- ERROR: permission denied for table pengguna
```

Penyempitan cakupan **V6** pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md) dengan demikian dapat ditunjukkan kepada sekolah, bukan sekadar dijanjikan.

### 7.2 Hak bawaan objek baru

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE edutrack_owner IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
```

Tidak ada default privilege bagi `app_ro`. Setiap tabel baru karenanya **tertutup bagi jalur AI sampai diberikan secara sadar** — pilihan bawaan yang aman, sehingga penambahan tabel di kemudian hari tidak membuka akses tanpa seorang pun memutuskannya.

---

## 8. Indeks dan kueri utama

Seluruh indeks non-bawaan telah tercantum pada Pasal 4. Pasal ini menunjukkan kueri yang membenarkan keberadaannya.

### 8.1 Matriks nilai satu kelas

Layar pengisian nilai Guru: 30 siswa dikali 8 komponen ([ARCHITECTURE.md §4](ARCHITECTURE.md)).

```sql
SELECT n.siswa_ref, n.komponen_ref, n.nilai
FROM nilai n
WHERE n.penugasan_ref = $1;
```

Dilayani `idx_nilai_penugasan_siswa`. Paling banyak 240 baris; pemutaran menjadi matriks dilakukan di frontend, bukan di SQL.

### 8.2 Kelengkapan sebelum finalisasi

Penegakan **I-20** dan penghasil pesan AC-07. Mengembalikan nama mata pelajaran yang belum lengkap pada satu kelas.

```sql
SELECT DISTINCT m.nama
FROM penugasan pg
JOIN mapel       m  ON m.id = pg.mapel_ref
JOIN kelas_siswa ks ON ks.kelas_ref = pg.kelas_ref
CROSS JOIN komponen_penilaian k
LEFT JOIN nilai n
       ON n.penugasan_ref = pg.id
      AND n.komponen_ref  = k.id
      AND n.siswa_ref     = ks.siswa_ref
WHERE pg.kelas_ref = $1
  AND n.id IS NULL;
```

Kueri ini adalah penerapan langsung I-12: karena baris `nilai` hanya ada apabila terisi, "belum lengkap" cukup dinyatakan sebagai `n.id IS NULL`. Tidak diperlukan pembedaan antara nol dan kosong di dalam kueri.

Hasil kosong berarti kelengkapan terpenuhi dan finalisasi dapat dilanjutkan; hasil tidak kosong langsung menjadi daftar pesan *"Data Mapel &lt;nama&gt; belum ada, tolong hubungi guru yang bertanggung jawab."* Dijalankan di dalam transaksi finalisasi, sebelum penulisan `rapor_mapel`.

### 8.3 Persentase kehadiran per mata pelajaran

Penegakan **I-17** dan **I-18**.

```sql
SELECT pg.mapel_ref,
       round(100.0 * count(*) FILTER (WHERE p.status <> 'alpa') / count(*), 2)
           AS kehadiran_persen
FROM presensi  p
JOIN sesi      s  ON s.id = p.sesi_ref
JOIN penugasan pg ON pg.id = s.penugasan_ref
JOIN kelas     kl ON kl.id = pg.kelas_ref
WHERE p.siswa_ref = $1
  AND kl.periode_ref = $2
GROUP BY pg.mapel_ref;
```

Dilayani `idx_presensi_siswa`. Penyebutnya adalah `count(*)`, yaitu jumlah baris presensi milik siswa tersebut — yang menurut I-15 sama dengan jumlah sesi yang dibuka Guru ([RFC-001 §6.1](RFC-001-model-data-konseptual.md)). Pembilang memakai `FILTER` alih-alih `CASE`, sehingga ketentuan "hanya Alpa yang mengurangi" (P16, AC-29) terbaca langsung dari kuerinya.

Kueri yang sama dijalankan jalur AI melalui koneksi `app_ro`, dan seluruh tabel yang disentuhnya termasuk dalam daftar `GRANT SELECT` pada Pasal 7.

---

## 9. Migrasi dan data awal

### 9.1 Urutan

Migrasi berupa berkas SQL yang dibangkitkan `drizzle-kit` dan ditinjau sebelum diterapkan ([Techstack.md §4.2](Techstack.md)). Dijalankan role `edutrack_owner`.

| # | Berkas | Isi |
|:--:|---|---|
| 0001 | `identitas` | `pengguna`, `guru`, `siswa` beserta indeks keunikannya |
| 0002 | `periode` | `tahun_ajaran`, `periode`, `kelas`, `kelas_siswa` |
| 0003 | `kurikulum` | `mapel`, `penugasan`, `komponen_penilaian`, `penugasan_komponen` |
| 0004 | `pencatatan` | `nilai`, `sesi`, `presensi` |
| 0005 | `rapor` | `rapor`, `rapor_mapel` |
| 0006 | `jejak` | `audit_log` |
| 0007 | `penopang` | `sesi_masuk`, `pembatas_laju` |
| 0008 | `pemicu` | `trg_komponen_bobot` dan `trg_rapor_status_maju` beserta fungsinya — ditulis tangan |
| 0009 | `role` | `REVOKE`, `GRANT`, dan default privilege Pasal 7 — ditulis tangan |
| 0010 | `komponen-awal` | Delapan baris `komponen_penilaian` |

**Setiap berkas dijalankan di dalam satu transaksi.** PostgreSQL memiliki DDL transaksional ([Techstack.md §4.1](Techstack.md)), sehingga migrasi yang gagal di tengah jalan batal seluruhnya dan tidak meninggalkan skema separuh jadi pada basis data berisi data sekolah. Tidak ada satu pun pernyataan pada daftar di atas yang memerlukan `CREATE INDEX CONCURRENTLY` — pada volume [RFC-001 §8.1](RFC-001-model-data-konseptual.md), pembangunan indeks selesai dalam hitungan milidetik — sehingga sifat transaksional tersebut tidak perlu dikorbankan.

Migrasi 0010 wajib berada **setelah** 0008, karena `trg_komponen_bobot` akan menolak keadaan akhir transaksi yang jumlah bobotnya bukan 100. Penyisipan delapan baris dalam satu transaksi lolos; penyisipan sebagian tidak.

**Catatan penerapan berada di luar `public`.** [DEPLOYMENT.md §3.3](DEPLOYMENT.md) langkah 6 memanggil fungsi `migrate` pada **setiap** rilis, termasuk rilis yang tidak membawa migrasi baru. Penerapnya karenanya wajib mengetahui berkas mana yang sudah dijalankan, dan pengetahuan itu disimpan pada tabel `migrasi.diterapkan` di dalam skema `migrasi` tersendiri:

```sql
CREATE SCHEMA IF NOT EXISTS migrasi;

CREATE TABLE IF NOT EXISTS migrasi.diterapkan (
    berkas          text        PRIMARY KEY,
    sidik_jari      text        NOT NULL,
    diterapkan_pada timestamptz NOT NULL DEFAULT now()
);
```

Skema tersendiri, bukan tabel kedua puluh di `public`. Pasal 3 mengunci sembilan belas tabel sebagai bentuk data EduTrack, dan catatan penerapan bukan salah satunya — ia milik mekanisme penerapan, bukan model data. Pemisahan ini juga menjadikan `GRANT` Pasal 7 tidak perlu menyebutnya: `app_rw` dan `app_ro` tidak memperoleh `USAGE` atas skema `migrasi`, sehingga jalur aplikasi maupun jalur AI tidak dapat membacanya, apalagi mengubahnya (CK-S-10).

`sidik_jari` berupa SHA-256 atas isi berkas. Penerap menolak melanjutkan apabila berkas yang sudah tercatat ternyata berubah isinya — migrasi yang sudah berjalan di produksi tidak boleh disunting, dan penyuntingannya menjadi kegagalan yang berisik alih-alih perbedaan diam-diam antara skema yang dikira berlaku dan skema yang sungguh berlaku.

### 9.2 Data awal

Satu-satunya data awal adalah templat komponen penilaian ([PRD §8.3](PRD.md), [RFC-001 §5.1](RFC-001-model-data-konseptual.md)).

```sql
INSERT INTO komponen_penilaian (kode, nama, bobot, urutan) VALUES
    ('T1',  'Tugas 1',                6, 1),
    ('T2',  'Tugas 2',                6, 2),
    ('T3',  'Tugas 3',                6, 3),
    ('U1',  'Ulangan Harian 1',      10, 4),
    ('U2',  'Ulangan Harian 2',      10, 5),
    ('U3',  'Ulangan Harian 3',      10, 6),
    ('UTS', 'Ujian Tengah Semester', 26, 7),
    ('UAS', 'Ujian Akhir Semester',  26, 8);
-- 6×3 + 10×3 + 26 + 26 = 100
```

Angka ini **belum divalidasi sekolah** (V1 pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md), T-06 pada [RFC-001 §10](RFC-001-model-data-konseptual.md)). Karena komponen disimpan sebagai baris dan bukan kolom (D-01), hasil validasi yang mengubah jumlah maupun bobot komponen adalah **perubahan data**, bukan migrasi skema — bahkan pada basis data yang sudah berisi nilai sekolah sungguhan. Inilah manfaat D-01 yang paling nyata pada tingkat skema fisik.

Akun Administrator **tidak termasuk data awal**. Ia dibuat lewat perintah CLI ([ARCHITECTURE.md §9.3](ARCHITECTURE.md)), sehingga tidak ada kata sandi bawaan yang tertinggal di dalam repositori maupun berkas migrasi.

### 9.3 Perkiraan ukuran

Diturunkan dari volume [RFC-001 §8.1](RFC-001-model-data-konseptual.md), satu semester penuh.

| Tabel | Baris | Perkiraan ukuran termasuk indeks |
|---|--:|--:|
| `presensi` | ~34.600 | ~9 MB |
| `nilai` | ~17.300 | ~5 MB |
| `rapor_mapel` | ~2.160 | ~2 MB |
| `sesi` | ~1.150 | < 1 MB |
| Seluruh sisanya | ~1.400 | < 1 MB |
| **Total per semester** | **~57.000** | **< 20 MB** |

Volume tetap sesuai C-06: kecil, dan bukan pendorong pemilihan apa pun. Penyimpanan `db.t4g.micro` 20 GB gp3 menampung puluhan tahun data tanpa penyesuaian. `db.t4g.micro` memiliki sekitar 1 GB memori, sehingga **seluruh basis data muat di dalam cache** — perencanaan indeks di atas dimaksudkan menjaga kejelasan kueri, bukan mengatasi keterbatasan sumber daya.

---

## 10. Yang tidak dibangun

[RFC-001 §7](RFC-001-model-data-konseptual.md) mendaftar entitas dan atribut yang gugur karena penyempitan cakupan PRD v3.0. **Dokumen ini tidak membangun satu pun di antaranya**, termasuk `rumus`, `komponen_rumus`, `penugasan_rumus`, `penilaian_sikap`, `ai_insight`, `materi`, `rapor.versi`, `rapor_mapel.deskripsi_ai`, `guru.nip`, dan `siswa.nis`.

Apabila cakupan diperluas kemudian, baris yang bersangkutan dibuka kembali melalui **amandemen RFC-001**, bukan ditambahkan diam-diam pada migrasi.

Dua hal yang patut ditegaskan karena mudah terlewat:

**Tidak ada tabel bagi keluaran AI.** I-24 ditegakkan oleh ketiadaan tempat penyimpanan, bukan oleh disiplin kode. Tidak ada tabel, tidak ada kolom, dan tidak ada cache di dalam basis data yang dapat menampungnya.

**Tidak ada tabel riwayat.** P7 meniadakan riwayat koreksi nilai maupun presensi. `audit_log` melayani penelusuran administratif dan bukan fitur produk ([RFC-001 §5.7](RFC-001-model-data-konseptual.md)), serta tidak ditulis sepanjang MVP (CK-A-06).

---

## 11. Yang perlu dijamin lapisan aplikasi

Selain lima invarian pada §5.1 yang bertanda ⚙️, terdapat satu aturan yang **tidak tercantum pada [RFC-001 §6](RFC-001-model-data-konseptual.md)** tetapi tersirat pada [PRD §8.2](PRD.md) butir 5 — *"tidak mungkin memasukkan nilai siswa yang tidak diajarnya"*.

**Skema ini tidak menjamin `nilai.siswa_ref` merupakan anggota kelas dari `nilai.penugasan_ref`.** Foreign key hanya menjamin siswa tersebut ada, bukan bahwa ia berada di kelas yang bersangkutan. Body request `POST /api/penugasan/:id/nilai` berisi daftar `siswa_ref` yang berasal dari klien; klien yang keliru atau jahat dapat menyertakan siswa dari kelas lain.

| Aspek | Keterangan |
|---|---|
| **Akibat bila terjadi** | Baris sampah yang tidak pernah tampil, karena seluruh kueri tampilan menyaring lewat `kelas_siswa`. Tidak merusak nilai siswa lain dan tidak memengaruhi kelengkapan maupun rapor |
| **Penegakan yang berlaku** | Satu kueri pemeriksaan di dalam transaksi Simpan Nilai: seluruh `siswa_ref` pada body wajib merupakan anggota `kelas_siswa` dari kelas penugasan tersebut |
| **Jalur naik** | Menambahkan `kelas_ref` pada `nilai` beserta dua composite foreign key — ke `penugasan (id, kelas_ref)` dan ke `kelas_siswa (kelas_ref, siswa_ref)` — sehingga keadaan ini menjadi mustahil tersimpan |

Jalur naik tersebut **tidak diambil sekarang** karena menambah kolom pada entitas yang bentuknya ditetapkan RFC-001, sehingga menuntut amandemen RFC dan bukan keputusan skema. Persoalan ini diajukan sebagai temuan **S-01** pada Pasal 12; apabila RFC menerima invarian barunya, jalur naik di atas menjadi cara penegakannya.

---

## 12. Temuan

Celah yang ditemukan saat menurunkan skema fisik. Perlu ditanggapi tim.

| # | Temuan | Usulan tindakan |
|---|---|---|
| S-01 | [RFC-001 §6](RFC-001-model-data-konseptual.md) tidak memuat invarian "nilai hanya boleh dicatat bagi siswa yang terdaftar pada kelas penugasan", padahal [PRD §8.2](PRD.md) butir 5 menyiratkannya | Tambahkan sebagai invarian baru lewat amandemen RFC-001. Cara penegakannya sudah disiapkan pada Pasal 11 |
| S-02 | ~~`uq_kelas_wali_per_periode` menegakkan **asumsi** [aktor-role.md §12](aktor-role.md) butir 1, bukan ketentuan PRD~~ | **Ditutup 7 Agustus 2026.** Sekolah menyatakan satu Guru menjadi Wali Kelas paling banyak satu kelas. Indeks dipertahankan dan kini menegakkan ketentuan, bukan asumsi. Catatan kedua pada temuan ini tetap berlaku: [ARCHITECTURE.md §1.1](ARCHITECTURE.md) menyebut partial unique index sebagai penegak I-09, padahal I-09 sudah terjamin bentuk kolom `kelas.wali_kelas_ref` — indeks ini menegakkan ketentuan wali, bukan I-09 |
| S-03 | ~~Kredensial role pemilik tidak tercatat pada `Techstack.md` §7~~ | **Ditutup 7 Agustus 2026.** `Techstack.md` §7 kini memuat empat rahasia; `edutrack_owner` diperlakukan sama seperti `app_rw`, dan hanya dapat dibaca role `edutrack-lambda-migrate` ([DEPLOYMENT.md §9.5](DEPLOYMENT.md)) |
| S-04 | ~~`tingkat` dibatasi `CHECK (... IN ('X','XI','XII'))`, sehingga skema ini mengikat produk pada jenjang SMA/SMK~~ | **Ditutup 7 Agustus 2026.** Sekolah menyatakan MVP mencakup jenjang SMA saja. `CHECK` dipertahankan apa adanya. Apabila jenjang SMP masuk cakupan kelak, perluasannya tetap satu `ALTER TABLE` sesuai alasan CK-S-02 |
| S-05 | [ARCHITECTURE.md §11.2](ARCHITECTURE.md) mewajibkan penghapusan berkas rapor dari S3 pada koreksi Administrator, di dalam transaksi yang sama. Basis data tidak dapat menjamin keberhasilan operasi S3 di dalam transaksinya | Tetapkan urutannya pada `API.md`: hapus objek S3 lebih dahulu, baru `COMMIT`; kegagalan penghapusan membatalkan transaksi. Kosongkan `rapor.kunci_berkas` pada transaksi yang sama |
| S-06 | Migrasi 0009 membuat `app_rw` dan `app_ro` tanpa kata sandi (CK-S-09), sehingga penerapan pertama pada lingkungan baru memiliki satu langkah penetapan kata sandi yang tidak dilakukan migrasi | Catat langkah tersebut pada [DEPLOYMENT.md](DEPLOYMENT.md) sebagai bagian penerapan pertama, sebelum penerapan sungguhan. Tidak menghalangi pengembangan maupun uji lokal |

Temuan **T-01, T-04, T-05, dan T-06** pada [RFC-001 §10](RFC-001-model-data-konseptual.md) masih terbuka dan bersifat produk. Tidak satu pun terpengaruh keputusan pada dokumen ini.

**T-02 ditutup 7 Agustus 2026.** Sekolah menyatakan satu siswa berada pada tepat satu kelas per semester sepanjang MVP, tanpa perpindahan kelas di tengah semester. I-08 dengan demikian bukan lagi asumsi, dan penegakannya lewat `uq_kelas_siswa_periode` beserta `fk_kelas_siswa_kelas` komposit dipertahankan.

---

## 13. Yang belum diputuskan

| # | Item | Menunggu | Dampak apabila berubah |
|---|---|---|---|
| 1 | Apakah `nilai` menerima pecahan atau hanya bilangan bulat | Konfirmasi ke sekolah bersama V1 | `numeric(5,2)` menampung keduanya, sehingga jawabannya hanya memperketat validasi Zod. Tidak ada migrasi |
| 2 | Lama retensi `sesi_masuk` yang sudah kedaluwarsa dan `pembatas_laju` yang sudah lewat | Keputusan tim | Saat ini keduanya dibersihkan secara oportunistik. Bila diperlukan pembersihan terjadwal, jalur naiknya `pg_cron` atau satu perintah CLI, bukan pekerjaan latar |
| 3 | Retensi dan pencadangan data akademik lintas tahun ajaran | V6 pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md) | Menentukan apakah data tahun ajaran lampau tetap berada di basis data yang sama atau diarsipkan. Tidak menyentuh bentuk tabel |
| 4 | Bentuk baku `snapshot_komponen` sebagai kontrak antara finalisasi dan render PDF | `API.md` dan format rapor sekolah (V5) | Isi JSONB, bukan bentuk kolom. Berkas rapor lama tidak perlu dirender ulang karena selalu dirender dari salinan bekunya sendiri |

---

## Lampiran — Catatan Keputusan

Bernomor dan bertanggal. Entri tidak disunting; perubahan keputusan ditulis sebagai entri baru yang menyebut nomor yang digantikannya.

Penomoran memakai awalan `CK-S-` sehingga tidak bertabrakan dengan `CK-xx` pada [Techstack.md](Techstack.md), `CK-A-xx` pada [ARCHITECTURE.md](ARCHITECTURE.md), maupun `CK-D-xx` pada [DEPLOYMENT.md](DEPLOYMENT.md).

### CK-S-01 · 6 Agustus 2026 · Pengenal berupa UUID acak

**Diputuskan.** Seluruh pengenal berupa `uuid` dengan `DEFAULT gen_random_uuid()`.

**Alasan.** Pengenal muncul di dalam alamat URL — `POST /api/penugasan/:id/nilai`, unduhan rapor — pada sistem berisi data akademik anak di bawah umur. Pengenal berurutan memungkinkan seseorang menebak keberadaan sumber daya lain dan mengukur pertumbuhannya, sedangkan pemeriksaan kewenangan menjadi satu-satunya penghalang. UUID menghapus seluruh kelas persoalan itu dengan biaya nol pada volume C-06. Keuntungan sampingannya: tidak ada sequence di dalam skema, sehingga pemberian hak akses Pasal 7 menjadi lebih ringkas dan tidak menyisakan celah `USAGE ON SEQUENCE` yang terlupa.

**Alternatif yang ditolak.**

*`bigint GENERATED ALWAYS AS IDENTITY`.* Lebih ringkas delapan byte dan lebih ramah bagi indeks. Ditolak karena penghematannya tidak berarti pada ~57 ribu baris per semester, sedangkan sifat dapat ditebaknya berlaku selamanya.

*UUIDv7.* Terurut menurut waktu sehingga menghindari pemencaran penulisan indeks. Ditolak karena `uuidv7()` baru tersedia bawaan pada PostgreSQL 18, sedangkan [Techstack.md §2](Techstack.md) memaku PostgreSQL 17. Membangkitkannya di lapisan aplikasi memindahkan pembangkitan pengenal keluar dari basis data demi keuntungan kinerja yang tidak terasa pada volume ini.

**Konsekuensi yang diterima.** Pemencaran penulisan indeks pada `nilai` dan `presensi`. Pada dua tabel yang seluruhnya muat di dalam memori, ini tidak terukur.

### CK-S-02 · 6 Agustus 2026 · Himpunan tertutup memakai CHECK, bukan tipe ENUM

**Diputuskan.** `peran`, `semester`, `tingkat`, `status` presensi, `status` rapor, dan `severity` disimpan sebagai `text` beserta `CHECK (... IN (...))`.

**Alasan.** PostgreSQL tidak memiliki `ALTER TYPE ... DROP VALUE`. Nilai enum yang keliru bertahan selamanya, dan menghapusnya berarti membangun tipe baru, memindahkan seluruh kolom, lalu menjatuhkan tipe lama. `tingkat` adalah contoh yang paling mungkin bergerak: nilainya kini `X`, `XI`, dan `XII`, sedangkan jenjang SMP menuntut himpunan yang sama sekali berbeda (S-04). Dengan `CHECK`, perubahan himpunan berupa satu `ALTER TABLE ... DROP CONSTRAINT` diikuti `ADD CONSTRAINT`, dua arah, di dalam satu transaksi.

**Alternatif yang ditolak.** *`CREATE TYPE ... AS ENUM`.* Lebih hemat penyimpanan, memberi pengurutan alami, dan tercermin lebih rapi pada tipe Drizzle. Ditolak karena penghematannya tidak terukur pada volume ini, sedangkan sifat satu arahnya berlaku pada basis data yang sudah memuat data sekolah.

**Konsekuensi yang diterima.** Pesan kesalahan menyebut nama constraint, bukan nama tipe, sehingga penerjemahannya menjadi pesan bagi pengguna dilakukan lapisan aplikasi. Konvensi penamaan constraint pada Pasal 2 dimaksudkan untuk itu.

### CK-S-03 · 6 Agustus 2026 · Tiga composite foreign key pada `penugasan`

**Diputuskan.** `penugasan` memiliki tiga foreign key komposit: `(mapel_ref, tingkat)`, `(kelas_ref, tingkat)`, dan `(mapel_ref, guru_ref)`. `mapel` menyediakan `UNIQUE (id, tingkat)` dan `UNIQUE (id, guru_ref)`; `kelas` menyediakan `UNIQUE (id, tingkat)`.

**Alasan.** [RFC-001 §5.3](RFC-001-model-data-konseptual.md) menyatakan `penugasan.tingkat` redundan secara logika dan menuntut konsistensinya dijamin basis data, dengan cara penjaminannya ditetapkan dokumen ini. Berbagi satu kolom `tingkat` di antara dua foreign key adalah cara yang tepat: ketidakcocokan jenjang menjadi keadaan yang **tidak dapat tersimpan**, sehingga AC-24 tidak bergantung pada satu pun baris kode. Foreign key ketiga menutup celah yang setara pada `guru_ref`, yang tanpanya penugasan dapat menunjuk guru yang bukan pengampu mata pelajaran tersebut.

**Alternatif yang ditolak.**

*Pemeriksaan aplikasi sebelum menyisipkan.* Bentuk yang lazim dan tidak menuntut indeks tambahan. Ditolak oleh prinsip ④: pemeriksaan aplikasi hanya menjamin jalur yang ada hari ini, sedangkan perbaikan data manual dan endpoint yang ditulis kelak tidak melewatinya.

*Pemicu validasi.* Menjamin hal yang sama tanpa kolom redundan. Ditolak karena pemicu berjalan sebagai kode yang perlu dibaca dan diuji, sedangkan foreign key adalah pernyataan yang dijamin mesin dan terbaca langsung dari definisi tabel.

**Konsekuensi yang diterima.** Tiga unique index tambahan pada `mapel` dan `kelas`, seluruhnya pada tabel berukuran puluhan baris. Perubahan jenjang sebuah mata pelajaran memerlukan pembaruan `penugasan` yang bersangkutan pada transaksi yang sama.

### CK-S-04 · 6 Agustus 2026 · Kolom pembeda peran pada `guru` dan `siswa`

**Diputuskan.** `guru` dan `siswa` masing-masing memiliki kolom `peran` bernilai tetap, yang menjadi bagian dari foreign key komposit ke `pengguna (id, peran)`.

**Alasan.** [RFC-001 §7.1](RFC-001-model-data-konseptual.md) menyatakan penyusutan `guru` dan `siswa` menjadi entitas satu atribut disengaja, agar "menetapkan seorang siswa sebagai guru pengampu atau wali kelas menjadi mustahil di tingkat data". Foreign key sederhana ke `pengguna (id)` tidak mencapainya: baris `guru` tetap dapat dibuat bagi pengguna berperan `siswa`, dan seluruh rantai di atasnya ikut menjadi salah. Kolom pembeda menutup lingkaran itu dengan dua kolom konstan dan satu unique index.

**Alternatif yang ditolak.** *Pemeriksaan aplikasi pada saat pembuatan profil.* Menghasilkan jaminan yang sama pada jalur yang ada. Ditolak dengan alasan yang sama seperti CK-S-03. *Kolom `GENERATED ALWAYS AS ... STORED`.* Lebih menyatakan maksud, tetapi menambah ketentuan PostgreSQL mengenai kolom terbangkitkan di dalam foreign key tanpa memberi jaminan tambahan apa pun.

**Konsekuensi yang diterima.** Dua kolom yang tidak menyimpan informasi. Keduanya tidak pernah dibaca aplikasi dan tidak muncul pada respons API mana pun.

### CK-S-05 · 6 Agustus 2026 · Dua pemicu untuk I-10 dan I-21 — memperkuat ARCHITECTURE §1.2

**Diputuskan.** I-10 dan I-21 ditegakkan basis data melalui `trg_komponen_bobot` dan `trg_rapor_status_maju`. Validasi pada `domain/nilai.ts` dan `domain/rapor.ts` tetap ada.

**Yang berubah dari ARCHITECTURE §1.2.** Dokumen tersebut menempatkan I-10 di lapisan aplikasi dan tidak menyebut I-21 sama sekali. Keduanya ternyata **dapat** dijadikan keadaan yang mustahil tersimpan, sehingga prinsip ④ mengharuskan pemindahannya. Ini penguatan, bukan penggantian: lapisan aplikasi tetap memeriksa lebih dahulu agar pengguna memperoleh pesan yang dapat dipahami, sedangkan basis data menjadi penjamin terakhir.

**Alasan.** Perubahan bobot dan perubahan status rapor sama-sama jarang dan sama-sama merusak apabila keliru. Bobot yang tidak berjumlah 100 membuat seluruh nilai akhir salah tanpa satu pun kesalahan muncul. Status yang dapat mundur mencabut arti finalisasi, dan justru paling mungkin terjadi lewat endpoint yang ditulis kelak oleh orang yang belum membaca [PRD §9](PRD.md).

**Alternatif yang ditolak.** *Membiarkan keduanya di lapisan aplikasi sesuai ARCHITECTURE §1.2.* Konsisten dengan dokumen di atasnya, tetapi bertentangan dengan prinsip yang dinyatakan dokumen itu sendiri. *Menggantikan validasi aplikasi dengan pemicu saja.* Ditolak karena pesan kesalahan basis data bukan pesan yang layak ditampilkan kepada Administrator, sedangkan P21 mensyaratkan kegagalan menyebutkan alasannya.

**Konsekuensi yang diterima.** Dua fungsi PL/pgSQL yang perlu ditulis tangan dan diuji tersendiri, di luar pembangkitan Drizzle. Keduanya berjumlah kurang dari tiga puluh baris dan tidak menyentuh jalur panas mana pun.

### CK-S-06 · 6 Agustus 2026 · Token sesi disimpan sebagai hash

**Diputuskan.** `sesi_masuk` menyimpan SHA-256 atas token, bukan tokennya. Argon2id tidak dipakai di sini.

**Alasan.** CK-A-04 menetapkan sesi berupa baris di basis data. Menyimpan tokennya apa adanya berarti setiap salinan basis data — cadangan RDS, dump untuk pengembangan, berkas cadangan on-prem yang tersalin ke luar server (CK-15) — memuat sesi yang dapat langsung dipakai masuk sebagai pengguna mana pun yang sedang aktif. Dengan hash, salinan tersebut tidak bernilai bagi penyerang.

Argon2id tidak diperlukan karena masukannya berbeda sifat dari kata sandi. Token dibangkitkan acak 256 bit sehingga tidak dapat ditebak melalui daftar kata maupun percobaan beruntun; yang dibutuhkan hanya fungsi searah yang cepat. Memakai Argon2id di sini justru membebani setiap request dengan pekerjaan yang sengaja dibuat lambat.

**Konsekuensi yang diterima.** Token tidak dapat dipulihkan dari basis data, sehingga tidak ada jalur "tampilkan ulang sesi". Hal ini tidak diminta fitur mana pun.

### CK-S-07 · 6 Agustus 2026 · Tiga role basis data, bukan dua

**Diputuskan.** Ditambahkan `edutrack_owner` sebagai pemilik objek dan satu-satunya role yang boleh DDL, di samping `app_rw` dan `app_ro`.

**Alasan.** Migrasi memerlukan hak DDL. Memberikannya kepada `app_rw` berarti setiap request aplikasi berjalan dengan kewenangan mengubah skema — termasuk menjatuhkan tabel dan, yang lebih menentukan, **memberikan hak tulis kepada `app_ro`**. Jaminan I-23 pada [ARCHITECTURE.md §8](ARCHITECTURE.md) akan bergantung pada tidak adanya kode yang menyalahgunakan hak itu, padahal seluruh nilai jaminan tersebut justru terletak pada tidak bergantungnya ia pada kode.

**Alternatif yang ditolak.** *Menjalankan migrasi sebagai `app_rw`.* Menghindari kredensial keempat. Ditolak dengan alasan di atas. *Menjalankan migrasi sebagai master user RDS.* Bekerja, tetapi menempatkan kredensial paling berkuasa di dalam jalur penerapan rutin, dan tidak dapat berpindah apa adanya ke on-prem.

**Konsekuensi yang diterima.** Satu kredensial tambahan yang perlu ditetapkan tempat penyimpanannya. [Techstack.md §7](Techstack.md) belum mencantumkannya; diajukan sebagai temuan S-03.

### CK-S-08 · 6 Agustus 2026 · `numeric` untuk nilai, bukan bilangan pecahan biner

**Diputuskan.** `nilai`, `nilai_akhir`, dan `kehadiran_persen` bertipe `numeric(5,2)`. `bobot` dan `kkm` bertipe `smallint`.

**Alasan.** `real` dan `double precision` tidak dapat menyatakan sebagian pecahan desimal secara tepat, sehingga penjumlahan berbobot dapat menghasilkan selisih pada digit terakhir. Untuk angka yang tercetak pada rapor dan dibaca orang tua, selisih semacam itu tidak dapat dipertanggungjawabkan. `numeric` menghitung dalam basis sepuluh dan tepat.

Bobot dan KKM sengaja tetap bilangan bulat: [PRD §8.3](PRD.md) menyatakan keduanya sebagai persen bulat, dan `smallint` menjadikan bobot pecahan mustahil sekaligus membuat penjumlahan bobot pada `trg_komponen_bobot` bersifat pasti.

**Konsekuensi yang diterima.** `numeric` lebih lambat daripada aritmetika biner. Pada agregat atas belasan ribu baris yang seluruhnya berada di dalam memori, perbedaannya tidak terukur.

### CK-S-09 · 7 Agustus 2026 · Role aplikasi dibuat migrasi, kata sandinya tidak

**Diputuskan.** Migrasi 0009 membuat `app_rw` dan `app_ro` secara idempoten dengan `LOGIN` dan **tanpa kata sandi**. Kata sandi ditetapkan di luar migrasi. `edutrack_owner` tidak dibuat migrasi karena ia yang menjalankannya.

**Alasan.** Pasal 7 menetapkan `GRANT` kepada kedua role tetapi tidak menyatakan siapa yang membuatnya, sehingga migrasi 0009 tidak dapat dijalankan pada basis data kosong mana pun — celah yang baru terlihat saat menurunkan skema menjadi berkas migrasi. Menempatkan pembuatannya di dalam 0009 menjaga agar seluruh bentuk hak akses berada pada satu berkas yang sama, dan menjadikan basis data pengembangan lokal identik dengan produksi tanpa langkah manual yang mudah terlupa.

Kata sandinya tidak ikut karena [Techstack.md §7](Techstack.md) butir 1 menempatkan nilai rahasia di luar Terraform dan di luar repositori. Berkas migrasi berada di dalam repositori dan terbaca siapa pun yang memegang salinannya.

**Alternatif yang ditolak.** *Membuat role di luar migrasi seluruhnya — Terraform atau prosedur manual.* Sejalan dengan pemisahan cangkang dan isi CK-D-02, tetapi menjadikan migrasi 0009 gagal pada basis data yang belum disiapkan tangan, termasuk kontainer uji yang menyala dan mati pada setiap kali tes berjalan. *`CREATE ROLE` beserta kata sandi acak di dalam migrasi.* Menempatkan kata sandi ke dalam repositori, melanggar Techstack §7.

**Konsekuensi yang diterima.** Terdapat keadaan antara: role sudah ada tetapi belum dapat dipakai masuk sampai kata sandinya ditetapkan. Keadaan tersebut tidak membuka apa pun — role `LOGIN` tanpa kata sandi ditolak autentikasi scram — tetapi berarti penerapan pertama pada lingkungan baru memiliki satu langkah yang tidak dilakukan migrasi, dan langkah itu wajib tercatat pada [DEPLOYMENT.md](DEPLOYMENT.md) sebelum penerapan sungguhan. Diajukan sebagai temuan **S-06** pada Pasal 12.

### CK-S-10 · 7 Agustus 2026 · Catatan penerapan migrasi berada di skema tersendiri

**Diputuskan.** Berkas migrasi yang sudah dijalankan dicatat pada `migrasi.diterapkan`, di dalam skema `migrasi`, bukan pada tabel di `public`. Isinya nama berkas, sidik jari SHA-256 atas isinya, dan waktu penerapannya.

**Alasan.** [DEPLOYMENT.md §3.3](DEPLOYMENT.md) langkah 6 memanggil fungsi `migrate` pada setiap rilis, sehingga penerap yang tidak mengingat apa pun akan menjalankan ulang seluruh migrasi dan gagal pada rilis kedua. Penerap membutuhkan catatan, dan catatan itu harus berada di dalam basis data yang sama agar tetap benar ketika rilis gagal di tengah.

Penempatannya di luar `public` menjaga Pasal 3 tetap benar apa adanya — sembilan belas tabel, seluruhnya bentuk data EduTrack. Tabel kedua puluh yang bukan data akan menjadikan pernyataan itu perlu pengecualian, dan pengecualian pada pernyataan penghitung adalah awal dari dokumen yang tidak lagi dapat dipercaya angkanya.

Manfaat kedua bersifat hak akses: Pasal 7 memberikan `USAGE` atas `public` saja, sehingga `app_rw` dan `app_ro` tidak dapat menyentuh skema `migrasi` tanpa satu pun `REVOKE` tambahan. Ini sejalan dengan §7.2 — tertutup sampai diberikan secara sadar.

**Alternatif yang ditolak.** *Memakai `drizzle.__drizzle_migrations` bawaan Drizzle.* Sudah berada di skema tersendiri dan tidak perlu ditulis sendiri, tetapi menuntut penamaan berkas mengikuti `drizzle-kit` beserta `meta/_journal.json` yang dipelihara tangan — bertabrakan dengan penamaan `expand`/`contract` pada [DEPLOYMENT.md §6.5](DEPLOYMENT.md) lapis 1, yang tidak dapat dinyatakan `drizzle-kit`. *Menyimpulkan dari bentuk skema yang sedang berlaku.* Tidak memerlukan tabel sama sekali, tetapi menjadikan penerapan bergantung pada penebakan, dan migrasi yang hanya memindahkan data tidak meninggalkan jejak yang dapat ditebak.

**Konsekuensi yang diterima.** Penerap migrasi menjadi kode milik sendiri, bukan pustaka. Ukurannya kecil dan seluruh perilakunya diuji, tetapi ia tetap satu bagian tambahan yang dapat rusak — dan bagian yang rusaknya paling mahal, karena ia berjalan mendahului setiap rilis.

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 6 Agustus 2026 | **Versi 1.0 — dokumen dibuat.** Menurunkan [RFC-001](RFC-001-model-data-konseptual.md) menjadi skema fisik PostgreSQL 17 di atas [Techstack.md](Techstack.md) v2.0 dan [ARCHITECTURE.md](ARCHITECTURE.md) v1.0. Sembilan belas tabel: tujuh belas entitas RFC-001 ditambah `sesi_masuk` dan `pembatas_laju` sebagai wujud fisik CK-A-04 dan CK-A-03. Pasal 5 memenuhi tuntutan [RFC-001 §6](RFC-001-model-data-konseptual.md) dengan memetakan seluruh dua puluh lima invarian beserta cara penegakannya. I-10 dan I-21 dipindahkan ke basis data lewat dua pemicu, memperkuat [ARCHITECTURE.md §1.2](ARCHITECTURE.md) (**CK-S-05**). Ditetapkan role ketiga `edutrack_owner` (**CK-S-07**), dan ditetapkan bahwa `app_ro` tidak memiliki hak baca atas tabel identitas, sehingga "identitas siswa tidak pernah dikirim" menjadi batas yang ditegakkan basis data (§7.1). Lampiran Catatan Keputusan dibuka dengan **CK-S-01** sampai **CK-S-08**. Diajukan lima temuan **S-01** sampai **S-05** |
| 7 Agustus 2026 | **S-03 ditutup.** Kredensial `edutrack_owner` ditetapkan pada `Techstack.md` §7 dengan perlakuan sama seperti `app_rw`, dan pembacanya dibatasi role `edutrack-lambda-migrate` |
| 7 Agustus 2026 | **S-02, S-04, dan T-02 ditutup oleh jawaban sekolah.** MVP mencakup jenjang SMA saja, satu siswa berada pada tepat satu kelas per semester tanpa perpindahan, dan satu Guru menjadi Wali Kelas paling banyak satu kelas. Ketiganya sudah sesuai dengan skema v1.0, sehingga tidak ada satu pun constraint yang berubah — yang berubah adalah kedudukannya, dari asumsi menjadi ketentuan. §4.2 disesuaikan mengikutinya |
| 7 Agustus 2026 | §7 dilengkapi pembuatan role `app_rw` dan `app_ro` beserta alasan penjaga idempotennya (**CK-S-09**). Celah ini terlihat saat menurunkan Pasal 7 menjadi migrasi 0009: `GRANT` mustahil tanpa role-nya ada, sementara tidak ada dokumen yang menyatakan siapa yang membuatnya. Kata sandi tetap di luar migrasi sesuai `Techstack.md` §7 butir 1, dan konsekuensinya diajukan sebagai temuan **S-06** |
| 7 Agustus 2026 | §9.1 dilengkapi catatan penerapan migrasi `migrasi.diterapkan` pada skema tersendiri (**CK-S-10**). Celah ini terlihat dari `DEPLOYMENT.md` §3.3 langkah 6 yang memanggil fungsi `migrate` pada setiap rilis: penerap tanpa catatan gagal pada rilis kedua. Ditempatkan di luar `public` agar Pasal 3 tetap menyebut sembilan belas tabel tanpa pengecualian, dan agar `app_rw` maupun `app_ro` tidak memperolehnya |
