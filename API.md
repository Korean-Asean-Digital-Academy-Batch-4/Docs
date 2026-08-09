# Kontrak API — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v1.0 |
| **Tanggal** | 6 Agustus 2026 |
| **Disusun oleh** | Re:Code |
| **Sumber kebenaran** | [PRD.md](PRD.md) v3.0, [ATURAN-DAN-KRITERIA.md](ATURAN-DAN-KRITERIA.md) v1.0, [aktor-role.md](aktor-role.md) v3.0, [RFC-001](RFC-001-model-data-konseptual.md), [Techstack.md](Techstack.md) v2.0, [ARCHITECTURE.md](ARCHITECTURE.md) v1.0, dan [SCHEMA.md](SCHEMA.md) v1.0 |
| **Kedudukan** | Menetapkan **kontrak endpoint**: alamat, bentuk permintaan dan respons, kode status, serta katalog kesalahan. Dokumen terakhir pada rantai penguncian |
| **Dokumen lanjutan** | — |

> Dokumen ini menjawab **apa yang dipanggil frontend dan apa yang dikembalikan backend**. Alur request beserta urutan pemeriksaannya ditetapkan [ARCHITECTURE.md §14](ARCHITECTURE.md); bentuk penyimpanan ditetapkan [SCHEMA.md](SCHEMA.md); kewenangan ditetapkan [aktor-role.md](aktor-role.md). Dokumen ini tidak mengulang ketiganya.
>
> Bentuk endpoint yang disebut pada [ARCHITECTURE.md](ARCHITECTURE.md) bersifat penjelas alur dan **bukan kontrak**. Apabila berbeda, **dokumen ini yang berlaku**. Perbedaan yang ada tercatat pada §2.1.
>
> Mengikuti [konvensi dokumen](README.md), badan dokumen memuat **deskripsi keadaan yang berlaku** dan disunting langsung ketika berubah, sedangkan **Lampiran Catatan Keputusan** bernomor dan bertanggal serta hanya ditambah, tidak disunting.

---

## 1. Cakupan

### 1.1 Termasuk

Seluruh endpoint yang dipanggil frontend, bentuk permintaan dan responsnya, kode status, katalog kesalahan beserta teks yang diwajibkan PRD, dan daftar hal yang **sengaja tidak memiliki endpoint**.

### 1.2 Tidak termasuk

| Hal | Ditetapkan pada |
|---|---|
| Urutan pemeriksaan per request | [ARCHITECTURE.md §14](ARCHITECTURE.md) |
| Bentuk tabel, constraint, dan kueri | [SCHEMA.md](SCHEMA.md) |
| Susunan prompt AI | [ARCHITECTURE.md §10.1](ARCHITECTURE.md) |
| Tata letak layar | UI/UX, di atas [ATURAN-DAN-KRITERIA §3](ATURAN-DAN-KRITERIA.md) |

### 1.3 Enam keputusan yang dibuka dokumen ini

Enam hal berikut belum ditetapkan dokumen mana pun sebelum ini, dan masing-masing menentukan **bentuk endpoint**, bukan sekadar penamaannya. Seluruhnya bersifat teknis dan tidak mengubah satu pun ketetapan PRD.

| # | Keputusan | Catatan |
|---|---|---|
| 1 | Sesi presensi beserta seluruh statusnya ditulis oleh **satu** request Simpan Presensi | CK-API-11, mengamandemen CK-API-05 |
| 2 | Unggah berkas bersifat **tolak seluruhnya**; tidak ada penerimaan sebagian | CK-API-02 |
| 3 | Pembuatan kelas berupa **satu endpoint atomik**, didahului pratinjau yang tidak menulis | CK-API-03 |
| 4 | Kata sandi awal diserahkan sebagai **berkas CSV sekali unduh** | CK-API-04 |
| 5 | Guru dan Siswa **dapat** mengganti kata sandinya sendiri | CK-API-06 |
| 6 | Baris `rapor` berstatus `draft` dibuat massal pada saat kelas dibuat | CK-API-07 |

---

## 2. Konvensi

### 2.1 Alamat

Seluruh endpoint berada di bawah `/api`, pada domain yang sama dengan frontend ([ARCHITECTURE.md §3](ARCHITECTURE.md)).

**Tidak ada awalan versi.** NG2 mencabut integrasi dengan sistem pihak ketiga, sehingga satu-satunya konsumen API adalah frontend yang di-deploy bersamanya. Penambahan `/v1` akan menjanjikan kestabilan kepada pihak yang tidak ada (CK-API-09).

**Penamaan memakai Bahasa Indonesia**, mengikuti nama tabel dan kolom pada [SCHEMA.md §2](SCHEMA.md), sehingga jalur dari endpoint sampai kolom dapat ditelusuri tanpa kamus penerjemah. [ARCHITECTURE.md §10](ARCHITECTURE.md) dan [§14.2](ARCHITECTURE.md) menulis `POST /api/me/suggestion` sebagai penjelas alur; alamat yang berlaku adalah **`POST /api/saya/suggestion`**.

### 2.2 Amplop respons

```jsonc
// Berhasil
{ "data": { } }

// Gagal
{ "kesalahan": { "kode": "JENJANG_TIDAK_COCOK", "pesan": "...", "rincian": [ ] } }
```

`data` dan `kesalahan` tidak pernah muncul bersamaan. `rincian` bersifat opsional dan dipakai ketika satu kegagalan memiliki banyak sebab — baris berkas yang tidak terbaca, atau daftar mata pelajaran yang belum lengkap.

**`pesan` selalu siap tampil kepada pengguna akhir** dalam Bahasa Indonesia. Ini pemenuhan langsung P21 dan AC-27: kegagalan wajib menyebutkan alasannya. `kode` dipakai frontend untuk menentukan perlakuan, bukan untuk diterjemahkan ulang (CK-API-01).

Respons `204 No Content` tidak memiliki badan.

### 2.3 Kode status

| Kode | Dipakai untuk |
|---|---|
| `200` | Pembacaan berhasil, atau mutasi yang mengembalikan ringkasan |
| `201` | Sumber daya baru terbentuk |
| `204` | Mutasi berhasil tanpa badan respons |
| `400` | Bentuk permintaan tidak sah menurut Zod, atau isi berkas tidak terbaca |
| `401` | Sesi tidak ada, kedaluwarsa, atau sudah dicabut |
| `403` | Lulus autentikasi tetapi gagal lapis peran atau lapis baris |
| `409` | Permintaan sah tetapi bertentangan dengan keadaan data |
| `413` | Berkas melampaui 2 MB |
| `429` | Batas laju terlampaui |
| `500` | Kegagalan tak terduga |
| `503` | Layanan AI tidak dapat dihubungi — **hanya** pada jalur Suggestion |

Kegagalan kewenangan dijawab **`403`, bukan `404`**, sesuai [ARCHITECTURE.md §9.2](ARCHITECTURE.md).

### 2.4 Waktu, angka, dan pengenal

| Aspek | Ketentuan |
|---|---|
| Zona waktu | **`Asia/Jakarta`** untuk seluruh penafsiran tanggal, termasuk penentuan "hari ini" pada sesi presensi. Region AWS `ap-southeast-3` kebetulan juga berzona UTC+7, tetapi kesamaan itu **tidak dijadikan sandaran** — zona waktu ditulis eksplisit agar tidak bergantung pada setelan server |
| Momen | ISO 8601 beserta offset, misalnya `2026-08-06T14:30:00+07:00` |
| Tanggal kalender | `YYYY-MM-DD`, tanpa waktu dan tanpa zona |
| Nilai dan persentase | Angka JSON dengan paling banyak dua desimal |
| Bobot dan KKM | Bilangan bulat |
| Pengenal | String UUID |
| Nilai kosong | `null`. Ketiadaan baris nilai diwakili `null`, bukan `0` — penegakan I-12 dan AC-06 sampai ke bentuk respons |

### 2.5 Autentikasi dan CSRF

Setiap endpoint selain `POST /api/auth/masuk` mensyaratkan cookie sesi:

```
Set-Cookie: edutrack_sesi=<token>; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=43200
```

**Tidak ada token CSRF tersendiri.** Frontend dan API berada pada satu domain ([ARCHITECTURE.md §3](ARCHITECTURE.md)) dan cookie memakai `SameSite=Strict`, sehingga situs lain tidak dapat menyertakannya pada request apa pun. Menambahkan token CSRF di atas itu tidak menutup celah yang tersisa.

### 2.6 Paginasi

**Tidak ada paginasi.** Daftar terbesar di seluruh sistem adalah 379 pengguna ([RFC-001 §8.1](RFC-001-model-data-konseptual.md)), dan daftar yang paling sering dipanggil — matriks nilai satu kelas — berisi paling banyak 240 sel. Menambahkan paginasi berarti membangun mekanisme beserta pengujiannya untuk keadaan yang tidak akan terjadi.

Apabila volume kelak berubah, jalur naiknya adalah parameter `kursor` dan `batas` pada endpoint daftar, tanpa mengubah bentuk amplop (CK-API-09).

### 2.7 Validasi

Seluruh badan permintaan divalidasi Zod di batas HTTP ([ARCHITECTURE.md §12](ARCHITECTURE.md)). Bidang yang tidak dikenal **ditolak**, bukan diabaikan, sehingga salah ketik nama bidang muncul sebagai kegagalan dan bukan sebagai perubahan yang diam-diam tidak tersimpan.

Kata sandi **tidak memiliki syarat kerumitan** ([PRD §6.1.3](PRD.md)). Validasinya hanya panjang 1 sampai 128 karakter. Tidak ada pemeriksaan huruf besar, angka, maupun simbol — dan ketiadaannya disengaja, bukan terlupa.

---

## 3. Autentikasi dan akun sendiri

### 3.1 Masuk

```http
POST /api/auth/masuk
{ "nama_pengguna": "198501012010011002", "kata_sandi": "..." }
```

```jsonc
// 200
{ "data": {
    "id": "…", "nama": "Cahyo Nugroho", "peran": "guru",
    "penugasan": [ { "id": "…", "kelas_nama": "X IPA 1", "mapel_nama": "Biologi X" } ],
    "wali_kelas": [ { "kelas_ref": "…", "kelas_nama": "X IPA 1" } ]
} }
```

Dibatasi **dua lapis**: 5 kegagalan per 15 menit per akun, dan 30 kegagalan per 15 menit per alamat IP. Ambangnya sengaja berbeda; alasannya pada **CK-A-08** ([ARCHITECTURE.md §7](ARCHITECTURE.md)). Akun dengan `aktif = false` ditolak `401` dengan kode yang sama seperti kata sandi salah, agar keberadaan akun tidak terungkap.

`penugasan` dan `wali_kelas` disertakan sejak masuk karena keduanya menentukan menu yang boleh tampil (UC-01), dan `wali_kelas` yang kosong adalah satu-satunya penanda bahwa menu finalisasi tidak boleh dirender — sesuai CK-A-01, kewenangan Wali Kelas tidak berupa peran.

### 3.2 Endpoint lain

| Metode | Alamat | Isi |
|---|---|---|
| `POST` | `/api/auth/keluar` | Menghapus baris sesi. `204` |
| `GET` | `/api/saya` | Bentuk yang sama dengan respons masuk. Dipanggil saat halaman dimuat ulang |
| `PATCH` | `/api/saya/kata-sandi` | `{ kata_sandi_lama, kata_sandi_baru }` → `204` |

**`PATCH /api/saya/kata-sandi` mencabut seluruh sesi lain milik pengguna tersebut** dan mempertahankan sesi yang sedang dipakai. Tersedia bagi ketiga peran (CK-API-06).

**Tidak ada endpoint lupa kata sandi.** Tombolnya ada di halaman masuk tetapi hanya menampilkan pesan statis di frontend, tanpa memanggil apa pun — pemenuhan AC-33 dan §6.1.3. Endpoint yang menerima nama pengguna, sekalipun hanya menjawab pesan tetap, akan menjadi jalur pengungkapan akun yang tidak diminta siapa pun.

---

## 4. Peta endpoint

| Kelompok | Metode dan alamat | Peran |
|---|---|---|
| **Autentikasi** | `POST /api/auth/masuk` · `POST /api/auth/keluar` · `GET /api/saya` · `PATCH /api/saya/kata-sandi` | Semua |
| **Akun** | `POST /api/pengguna` · `POST /api/pengguna/unggah` · `GET /api/pengguna` · `POST /api/pengguna/:id/kata-sandi` | Administrator |
| **Templat** | `GET /api/templat/pengguna.csv?peran=guru\|siswa` · `GET /api/templat/daftar-siswa.xlsx` | Administrator |
| **Periode** | `POST /api/tahun-ajaran` · `GET /api/tahun-ajaran` · `POST /api/tahun-ajaran/:id/periode` · `PATCH /api/periode/:id/aktif` | Administrator |
| **Mata pelajaran** | `POST /api/mapel` · `GET /api/mapel` · `PATCH /api/mapel/:id` | Administrator |
| **Komponen** | `GET /api/komponen-penilaian` · `PUT /api/komponen-penilaian` | Baca: semua · Tulis: Administrator |
| **Kelas** | `POST /api/kelas/pratinjau` · `POST /api/kelas` · `GET /api/kelas` · `GET /api/kelas/:id` | Administrator |
| **Nilai** | `GET /api/penugasan/:id/nilai` · `POST /api/penugasan/:id/nilai` | Guru pengampu · Administrator |
| **Presensi** | `GET /api/penugasan/:id/siswa` · `POST /api/penugasan/:id/sesi` · `GET /api/penugasan/:id/sesi` · `GET /api/sesi/:id` · `PUT /api/sesi/:id/presensi` · `DELETE /api/sesi/:id` | Guru pengampu · Administrator |
| **Pantauan kelas** | `GET /api/kelas/:id/nilai` · `GET /api/kelas/:id/presensi` | Wali Kelas · Administrator |
| **Rapor** | `GET /api/kelas/:id/rapor` · `PATCH /api/rapor/:id` · `POST /api/kelas/:id/rapor/finalisasi` · `POST /api/kelas/:id/rapor/distribusi` · `GET /api/rapor/:id/berkas` · `GET /api/kelas/:id/rapor/berkas` | Wali Kelas · Administrator · Siswa (unduh per siswa saja) |
| **Siswa** | `GET /api/saya/nilai` · `GET /api/saya/presensi` · `GET /api/saya/rapor` · `POST /api/saya/suggestion` | Siswa |

Empat puluh tiga endpoint. Lapis peran dan lapis baris untuk masing-masing mengikuti [ARCHITECTURE.md §9.2](ARCHITECTURE.md) tanpa pengecualian.

---

## 5. Administrasi

### 5.1 Pembuatan akun

```http
POST /api/pengguna
{ "nama": "Cahyo Nugroho", "nama_pengguna": "198501012010011002", "peran": "guru" }
```

```jsonc
// 201
{ "data": { "id": "…", "nama_pengguna": "…", "kata_sandi_awal": "k7mQx2vT" } }
```

`peran` hanya menerima `guru` atau `siswa`. Nilai `administrator` ditolak `400`: akun Administrator dibuat lewat perintah CLI dan **tidak memiliki endpoint dalam bentuk apa pun** ([ARCHITECTURE.md §9.3](ARCHITECTURE.md)).

`kata_sandi_awal` muncul **hanya pada respons ini** dan tidak dapat dibaca ulang. Responsnya menyertakan `Cache-Control: no-store` dan tidak pernah dicatat ke log.

`GET /api/pengguna` mengembalikan seluruh akun Guru dan Siswa tanpa paginasi dalam bentuk `{ "data": [ { "id": "…", "nama": "Cahyo Nugroho", "nama_pengguna": "198501012010011002", "peran": "guru", "aktif": true } ] }`, terurut `nama`. Akun Administrator dan `kata_sandi_hash` tidak pernah muncul pada daftar ini (CK-API-17).

### 5.2 Unggah akun

```http
POST /api/pengguna/unggah
Content-Type: multipart/form-data
  peran=guru
  berkas=<CSV: Nama, NIP>          // atau <CSV: Nama, NIS> bagi siswa
```

Berhasil — respons berupa **berkas CSV**, bukan JSON:

```http
200 OK
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="kredensial-guru-2026-08-06.csv"
Cache-Control: no-store

nama,nama_pengguna,kata_sandi_awal
Cahyo Nugroho,198501012010011002,k7mQx2vT
```

Gagal — respons JSON menyebutkan setiap baris yang bermasalah, dan **tidak satu akun pun terbentuk**:

```jsonc
// 400
{ "kesalahan": {
    "kode": "BERKAS_TIDAK_SAH",
    "pesan": "Berkas tidak dapat diproses. 3 dari 42 baris bermasalah dan tidak ada akun yang dibuat.",
    "rincian": [
      { "baris": 7,  "sebab": "NIP 198501012010011002 sudah terdaftar" },
      { "baris": 19, "sebab": "Kolom Nama kosong" },
      { "baris": 31, "sebab": "NIP hanya boleh berisi angka" }
    ]
} }
```

**Satu baris gagal membatalkan seluruh berkas** (CK-API-02). Ini pemenuhan AC-26 setelah redaksinya diselaraskan: Administrator memperbaiki berkasnya lalu mengunggah ulang, alih-alih menelusuri akun mana yang sudah terlanjur ada. Penutupan temuan **A-03** dicatat pada §13.2.

Dibatasi **10 unggahan per jam per pengguna**, dengan **batas 2 MB per berkas** ([ARCHITECTURE.md §7](ARCHITECTURE.md)). Berkas yang lebih besar dijawab `413` tanpa dibaca isinya.

Sebagai batas pengamanan parser, berkas memuat paling banyak 360 baris data; satu record CSV paling banyak 4 KiB. Pelanggaran dijawab `400 BERKAS_TIDAK_SAH`, bukan dipotong diam-diam (CK-API-18).

Templat unggahan akun diunduh dari:

```http
GET /api/templat/pengguna.csv?peran=guru
```

Parameter query `peran` wajib bernilai `guru` atau `siswa`; parameter yang tidak ada maupun nilai lain ditolak `400 PERMINTAAN_TIDAK_SAH`. Respons berhasil berupa berkas `text/csv; charset=utf-8` dengan `Content-Disposition: attachment; filename="templat-pengguna-guru.csv"` atau `filename="templat-pengguna-siswa.csv"`. Templat Guru hanya memuat kepala `Nama,NIP`, sedangkan templat Siswa hanya memuat kepala `Nama,NIS` (CK-API-13).

Templat daftar siswa diunduh dari `GET /api/templat/daftar-siswa.xlsx`. Respons berhasil berupa berkas `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` dengan `Content-Disposition: attachment; filename="templat-daftar-siswa.xlsx"`; lembar pertamanya hanya memuat kepala `Kelas`, `NIS`, dan `Nama` yang dipakai §5.7.

### 5.3 Reset kata sandi oleh Administrator

```http
POST /api/pengguna/:id/kata-sandi
```

```jsonc
// 200
{ "data": { "kata_sandi_awal": "p3Rw9naL" } }
```

Menghasilkan kata sandi baru dan **mencabut seluruh sesi pengguna tersebut seketika** (CK-A-04). Respons menyertakan `Cache-Control: no-store`. Inilah satu-satunya jalur pemulihan yang tersedia ketika sebuah akun Guru atau Siswa diduga disalahgunakan, sekaligus satu-satunya jalur ketika berkas kredensial pada §5.2 hilang.

Sasaran endpoint ini hanya akun Guru dan Siswa. Pengenal Administrator dijawab `404 TIDAK_DITEMUKAN`: Administrator mengganti kata sandinya sendiri lewat `/api/saya/kata-sandi`, sedangkan pembuatan atau pemulihan Administrator di luar sesi berjalan tetap melalui CLI §9.3 arsitektur (CK-API-17).

### 5.4 Periode akademik

```http
POST /api/tahun-ajaran
{ "nama": "2026/2027", "tgl_mulai": "2026-07-13", "tgl_selesai": "2027-06-18" }
```

```jsonc
// 201
{ "data": { "id": "…", "nama": "2026/2027", "tgl_mulai": "2026-07-13", "tgl_selesai": "2027-06-18", "aktif": false } }
```

```http
POST /api/tahun-ajaran/:id/periode
{ "semester": "ganjil", "tgl_mulai": "2026-07-13", "tgl_selesai": "2026-12-18" }
```

`semester` hanya menerima `ganjil` atau `genap`. Tahun ajaran yang tidak ada dijawab `404 TIDAK_DITEMUKAN`; benturan nama tahun ajaran atau semester pada tahun yang sama dijawab `409 DATA_SUDAH_ADA`.

```jsonc
// 201
{ "data": { "id": "…", "tahun_ajaran_ref": "…", "semester": "ganjil", "tgl_mulai": "2026-07-13", "tgl_selesai": "2026-12-18", "aktif": false } }
```

`GET /api/tahun-ajaran` mengembalikan seluruh tahun ajaran terurut tanggal mulai terbaru, dengan periode masing-masing terurut tanggal mulai:

```jsonc
// 200
{ "data": [ {
    "id": "…", "nama": "2026/2027", "tgl_mulai": "2026-07-13", "tgl_selesai": "2027-06-18", "aktif": false,
    "periode": [ { "id": "…", "semester": "ganjil", "tgl_mulai": "2026-07-13", "tgl_selesai": "2026-12-18", "aktif": true } ]
} ] }
```

`PATCH /api/periode/:id/aktif` menjawab `204`, mengaktifkan satu periode, dan menonaktifkan periode lain pada tahun ajaran yang sama **dalam satu transaksi**. Periode yang tidak ada dijawab `404 TIDAK_DITEMUKAN`.

Penonaktifan yang menyertai pengaktifan bukan kemudahan melainkan keharusan: `uq_periode_aktif_per_tahun` ([SCHEMA.md §4.2](SCHEMA.md)) menolak dua periode aktif, sehingga pengaktifan tanpa penonaktifan akan gagal di tingkat basis data. Ini penegakan I-03 dan P19 yang terlihat sampai ke kontrak.

### 5.5 Mata pelajaran

```http
POST /api/mapel
{ "kode": "BIO-X", "nama": "Biologi", "tingkat": "X", "kkm": 75, "guru_ref": "…" }
```

`kkm` boleh dihilangkan dan berdefault **75** (AC-22, I-11). `guru_ref` menunjuk guru yang belum mengampu mata pelajaran mana pun; guru yang sudah mengampu ditolak `409 GURU_SUDAH_MENGAMPU`, penegakan I-05.

```jsonc
// 201 — bentuk yang sama dipakai elemen GET dan hasil PATCH
{ "data": {
    "id": "…", "kode": "BIO-X", "nama": "Biologi", "tingkat": "X", "kkm": 75,
    "guru": { "id": "…", "nama": "Cahyo Nugroho", "nama_pengguna": "198501012010011002" }
} }
```

`GET /api/mapel` mengembalikan `{ "data": [ … ] }` tanpa paginasi, terurut `tingkat` lalu `kode`. Guru yang tidak ada dijawab `404 TIDAK_DITEMUKAN`; kode duplikat dijawab `409 DATA_SUDAH_ADA`.

`PATCH /api/mapel/:id` menerima `nama` dan `kkm` saja dan menjawab `200` dengan bentuk mapel di atas. **`tingkat` dan `guru_ref` tidak dapat diubah**: keduanya menjadi bagian dari composite foreign key pada `penugasan` ([SCHEMA.md §4.3](SCHEMA.md)), sehingga perubahannya menuntut pembaruan seluruh penugasan terkait — alur yang tidak dimiliki PRD dan tercatat sebagai T-05 pada [RFC-001 §10](RFC-001-model-data-konseptual.md). Mapel yang tidak ada dijawab `404 TIDAK_DITEMUKAN`.

### 5.6 Komponen penilaian

`GET /api/komponen-penilaian` tersedia bagi seluruh pengguna terautentikasi dan mengembalikan daftar terurut `urutan`:

```jsonc
// 200
{ "data": [ { "id": "…", "kode": "T1", "nama": "Tugas 1", "bobot": 6, "urutan": 1 } ] }
```

```http
PUT /api/komponen-penilaian
{ "komponen": [ { "kode": "T1", "nama": "Tugas 1", "bobot": 6, "urutan": 1 }, … ] }
```

Mengganti seluruh templat dalam satu transaksi. Jumlah bobot yang bukan 100 ditolak dengan total saat ini disebutkan, sesuai AC-04:

```jsonc
// 400
{ "kesalahan": {
    "kode": "BOBOT_TIDAK_SERATUS",
    "pesan": "Jumlah bobot komponen penilaian harus tepat 100%, saat ini 98%."
} }
```

Bentuk `PUT` atas seluruh daftar dipilih karena penyesuaian bobot selalu menyentuh beberapa komponen sekaligus — menaikkan UTS berarti menurunkan yang lain. Endpoint per komponen akan menolak langkah pertama dari perubahan yang sah, persis persoalan yang diselesaikan `DEFERRABLE INITIALLY DEFERRED` pada [SCHEMA.md §5.2](SCHEMA.md).

Perubahan yang berhasil menjawab `200` dengan bentuk daftar yang sama seperti `GET`.

`kode` menjadi identitas stabil setelah templat pernah dipakai oleh sebuah penugasan. Selama belum ada `penugasan_komponen`, seluruh daftar boleh diganti termasuk menambah atau menghapus kode. Setelah dipakai, daftar `kode` pada permintaan wajib sama dengan daftar yang tersimpan; `nama`, `bobot`, dan `urutan` tetap dapat diubah atomik, tetapi penambahan, penghapusan, atau penggantian kode ditolak `409 KOMPONEN_SUDAH_DIPAKAI` (CK-API-15). Dengan demikian foreign key atas nilai dan topik tidak pernah diputus.

### 5.7 Pembuatan kelas

Dua endpoint: satu **pratinjau yang tidak menulis apa pun**, dan satu **pembuatan atomik** (CK-API-03).

```http
POST /api/kelas/pratinjau
Content-Type: multipart/form-data
  periode_ref=…
  berkas=<XLSX: Kelas, NIS, Nama>
```

```jsonc
// 200 — tidak ada satu baris pun yang tertulis ke basis data
{ "data": {
    "kelas_berkas": "X IPA 1",
    "cocok": [ { "baris": 2, "nis": "0071234567", "nama_berkas": "Andi Pratama", "nama_sistem": "Andi Pratama", "siswa_ref": "…" } ],
    "bermasalah": [
      { "baris": 5,  "nis": "0079999999", "sebab": "NIS tidak terdaftar sebagai akun siswa" },
      { "baris": 8,  "nis": "0071234567", "sebab": "NIS ganda di dalam berkas" },
      { "baris": 12, "nis": "0071112223", "sebab": "Sudah terdaftar pada kelas X IPA 2 pada semester ini" }
    ]
} }
```

`periode_ref` wajib menunjuk periode sasaran yang juga akan dikirim pada `POST /api/kelas`; nilai yang tidak ada atau bukan UUID ditolak `400 PERMINTAAN_TIDAK_SAH`. Periode tidak boleh ditebak dari periode aktif karena Administrator dapat menyiapkan semester yang belum aktif (CK-API-14).

`kelas_berkas` adalah satu-satunya nilai non-kosong kolom `Kelas` di seluruh baris. Apabila tidak ada tepat satu nilai unik, bidang ini bernilai `null` dan setiap baris yang kosong atau berbeda masuk `bermasalah`; lebih dari satu nama kelas tidak pernah dipilihkan diam-diam. `nama_berkas` dan `nama_sistem` ditampilkan berdampingan karena **NIS adalah kunci pencocokan dan nama hanya pemeriksaan** ([PRD §6.1.5](PRD.md) butir 3). Selisih nama tidak menggagalkan pencocokan, tetapi Administrator perlu melihatnya.

Baris "sudah terdaftar pada kelas lain" adalah pemeriksaan awal terhadap I-08, yang pada akhirnya ditegakkan `uq_kelas_siswa_periode`. Menampilkannya di pratinjau mengubah kegagalan basis data menjadi keterangan yang dapat ditindaklanjuti.

```http
POST /api/kelas
Content-Type: multipart/form-data
  data={ "periode_ref": "…", "nama": "X IPA 1", "tingkat": "X", "jurusan": "IPA",
         "guru_ref": [ "…", "…", "…" ], "wali_kelas_ref": "…" }
  berkas=<XLSX yang sama>
```

Nilai `Kelas` pada seluruh baris XLSX wajib sama dengan `data.nama`; perbedaan dilaporkan sebagai `400 BERKAS_TIDAK_SAH`. XLSX wajib memiliki tepat satu worksheet, paling banyak 360 baris data dan 64 entry ZIP, serta jumlah ukuran entry sebelum kompresi paling banyak 16 MiB. Pelanggaran batas pengamanan ini dijawab `400 BERKAS_TIDAK_SAH` (CK-API-18).

**Satu request, satu transaksi, satu kelas** (P18). Yang terbentuk sekaligus:

| Yang dibuat | Dasar |
|---|---|
| Satu baris `kelas` | [PRD §6.1.5](PRD.md) |
| Satu baris `kelas_siswa` per siswa yang cocok | §6.1.5 butir 3 |
| Satu baris `penugasan` per guru pada `guru_ref` | §8.2 lapis 3 |
| Satu baris `penugasan_komponen` untuk setiap komponen pada templat yang berlaku, `topik` masih kosong — delapan pada data V1 saat ini | D-06, CK-API-15 |
| Satu baris `rapor` berstatus `draft` per siswa | CK-API-07 |

**Mata pelajaran tidak disertakan dalam permintaan.** Server menurunkannya dari `mapel.guru_ref` masing-masing guru, sesuai [PRD §8.2](PRD.md): mata pelajaran sudah melekat pada guru sejak lapis kedua dan tidak pernah dipilih ulang.

Setiap anggota `guru_ref` wajib sudah memiliki mata pelajaran. Guru yang akunnya sah tetapi belum dihubungkan dengan mata pelajaran ditolak `409 GURU_BELUM_MENGAMPU`, dengan rincian `guru_ref` dan nama Guru. Ini membuat keadaan Guru tanpa penugasan pada AC-28 tetap sah tanpa memaksa server menebak `mapel_ref` (CK-API-16).

Penolakan yang mungkin terjadi:

```jsonc
// 409
{ "kesalahan": {
    "kode": "JENJANG_TIDAK_COCOK",
    "pesan": "Kelas berjenjang X tidak dapat dihubungkan dengan Bu Fifi yang mengampu Matematika berjenjang XI.",
    "rincian": [ { "guru_ref": "…", "guru_nama": "Fifi Handayani",
                   "mapel_nama": "Matematika", "jenjang_mapel": "XI", "jenjang_kelas": "X" } ]
} }
```

Pesan menyebutkan **kedua jenjang**, sesuai AC-24. Pemeriksaan ini dilakukan aplikasi agar pesannya dapat dipahami, sedangkan penjaminannya tetap berada pada composite foreign key — kekeliruan kode di jalur mana pun tetap ditolak PostgreSQL ([SCHEMA.md §4.3](SCHEMA.md)).

`wali_kelas_ref` wajib merupakan salah satu anggota `guru_ref`; di luar itu ditolak `400`. Ini penerapan [aktor-role.md §3](aktor-role.md): Wali Kelas selalu merupakan Guru yang juga mengajar di kelas asuhannya.

Pembuatan berhasil menjawab:

```jsonc
// 201
{ "data": { "id": "…", "nama": "X IPA 1", "periode_ref": "…", "jumlah_siswa": 30, "jumlah_penugasan": 6 } }
```

`GET /api/kelas` mengembalikan ringkasan tanpa paginasi, terurut periode terbaru lalu nama kelas:

```jsonc
// 200
{ "data": [ {
    "id": "…", "nama": "X IPA 1", "tingkat": "X", "jurusan": "IPA",
    "periode": { "id": "…", "semester": "ganjil", "tahun_ajaran_nama": "2026/2027" },
    "wali_kelas": { "id": "…", "nama": "Cahyo Nugroho" },
    "jumlah_siswa": 30, "jumlah_penugasan": 6
} ] }
```

`GET /api/kelas/:id` mengembalikan ringkasan yang sama beserta anggota dan penugasan:

```jsonc
// 200
{ "data": {
    "id": "…", "nama": "X IPA 1", "tingkat": "X", "jurusan": "IPA",
    "periode": { "id": "…", "semester": "ganjil", "tahun_ajaran_nama": "2026/2027" },
    "wali_kelas": { "id": "…", "nama": "Cahyo Nugroho" },
    "jumlah_siswa": 30, "jumlah_penugasan": 6,
    "siswa": [ { "id": "…", "nama": "Andi Pratama", "nama_pengguna": "0071234567" } ],
    "penugasan": [ {
      "id": "…", "guru": { "id": "…", "nama": "Cahyo Nugroho" },
      "mapel": { "id": "…", "kode": "BIO-X", "nama": "Biologi", "tingkat": "X", "kkm": 75 }
    } ]
} }
```

Larik `siswa` terurut nama; larik `penugasan` terurut kode mapel. Kelas yang tidak ada dijawab `404 TIDAK_DITEMUKAN`.

**Pembatalan tidak memiliki endpoint.** Proses yang ditinggalkan sebelum `POST /api/kelas` tidak menyisakan apa pun karena belum ada yang tertulis — pemenuhan "proses dapat dibatalkan" ([PRD §6.1.5](PRD.md) butir 2) sekaligus "tidak menyisakan kelas setengah jadi" (AC-26), tanpa memerlukan kelas berstatus draf.

---

## 6. Nilai

### 6.1 Membaca matriks

```http
GET /api/penugasan/:id/nilai
```

```jsonc
// 200
{ "data": {
    "penugasan": { "id": "…", "kelas_nama": "X IPA 1", "mapel_nama": "Biologi", "kkm": 75 },
    "komponen": [ { "id": "…", "kode": "T1", "nama": "Tugas 1", "bobot": 6, "urutan": 1,
                    "topik": "Sel dan jaringan" }, … ],
    "siswa": [ { "siswa_ref": "…", "nama": "Andi Pratama" }, … ],
    "nilai": [ { "siswa_ref": "…", "komponen_ref": "…", "nilai": 85 }, … ]
} }
```

`nilai` berbentuk daftar memanjang, **bukan matriks**. Pemutaran menjadi matriks 30 × 8 dilakukan frontend ([ARCHITECTURE.md §4](ARCHITECTURE.md)). Pasangan yang tidak muncul pada daftar berarti **belum diisi** — bukan nol, sesuai I-12 dan AC-06.

### 6.2 Simpan Nilai

```http
POST /api/penugasan/:id/nilai
{
  "nilai": [
    { "siswa_ref": "…", "komponen_ref": "…", "nilai": 85 },
    { "siswa_ref": "…", "komponen_ref": "…", "nilai": null }
  ],
  "topik": [ { "komponen_ref": "…", "topik": "Sel dan jaringan" } ]
}
```

```jsonc
// 200
{ "data": { "tersimpan": 238, "terhapus": 2 } }
```

| Ketentuan | Dasar |
|---|---|
| Satu request, satu transaksi. Kegagalan membatalkan seluruh baris | C-02, P22, AC-15 |
| `nilai: null` **menghapus barisnya**; ketiadaan baris adalah satu-satunya wujud nilai kosong | I-12, [ARCHITECTURE.md §14.1](ARCHITECTURE.md) langkah 10 |
| `topik` menumpang pada payload yang sama | CK-API-08, menutup T-01 dari sisi API |
| Guru ditolak `409` apabila rapor kelas sudah final; Administrator dilanjutkan | I-22, P13, AC-14 |
| Tidak ada pembatasan laju pada jalur ini | [ARCHITECTURE.md §14.1](ARCHITECTURE.md) langkah 4 |
| Seluruh `siswa_ref` wajib merupakan anggota kelas penugasan | [SCHEMA.md §11](SCHEMA.md) |

**Topik ikut pada payload Simpan Nilai** karena topik melekat pada pasangan penugasan-komponen (D-06) dan diisi pada layar yang sama. Endpoint tersendiri akan menuntut Guru menekan dua tombol simpan untuk satu layar, yang bertentangan dengan bentuk P22.

**Tidak ada langkah publikasi.** Setelah `COMMIT`, nilai langsung terlihat siswa yang bersangkutan (UC-09).

### 6.3 Koreksi Administrator atas data final

Endpoint yang sama dipakai Administrator, dengan lapis baris dilewati (P14). Apabila rapor terkait sudah `finalized` atau `distributed`, transaksi yang sama juga:

1. memperbarui baris `rapor_mapel` yang terpengaruh;
2. menghapus objek berkas rapor dari S3 **sebelum `COMMIT`**;
3. mengosongkan `rapor.kunci_berkas`.

Kegagalan penghapusan objek S3 membatalkan seluruh transaksi. Urutan ini menutup temuan S-05 pada [SCHEMA.md §12](SCHEMA.md): penghapusan yang gagal setelah `COMMIT` akan meninggalkan berkas usang yang tetap disajikan selamanya, melanggar AC-13 tanpa memunculkan kesalahan apa pun. Unduhan berikutnya merender ulang dari salinan beku yang sudah diperbarui (CK-A-05).

---

## 7. Presensi

### 7.1 Menyiapkan layar

**Membuka sesi adalah keadaan layar, bukan tindakan yang tersimpan** (CK-API-11). Guru memilih kelas dan tanggal; frontend memuat daftar siswa dan menampilkannya serba-Alpa. Belum ada satu baris pun yang tertulis.

```http
GET /api/penugasan/:id/siswa
```

```jsonc
// 200
{ "data": { "siswa": [ { "siswa_ref": "…", "nama": "Andi Pratama" }, … ] } }
```

Frontend juga memanggil `GET /api/penugasan/:id/sesi` untuk mengetahui tanggal mana yang sudah memiliki sesi. Tanggal yang sudah ada sesinya **dimuat untuk disunting** (§7.3), bukan dibuat ulang.

### 7.2 Simpan Presensi — membuat sesi

```http
POST /api/penugasan/:id/sesi
{
  "tanggal": "2026-08-06",
  "presensi": [ { "siswa_ref": "…", "status": "hadir", "catatan": null }, … ]
}
```

```jsonc
// 201
{ "data": {
    "sesi": { "id": "…", "tanggal": "2026-08-06" },
    "tersimpan": 30
} }
```

**Satu request, satu transaksi, dan tidak ada yang tertulis sebelumnya.** Baris `sesi` beserta seluruh baris `presensi` terbentuk bersamaan ketika Guru menekan Simpan Presensi — pemenuhan P22 dan AC-15 secara harfiah: meninggalkan layar tanpa menekan tombol tidak mengubah data apa pun, termasuk tidak meninggalkan sesi kosong.

| Ketentuan | Dasar |
|---|---|
| Server menurunkan daftar siswa dari `kelas_siswa`, **bukan dari payload**. Siswa yang tidak disebut permintaan tetap disisipkan berstatus `alpa` | I-15, AC-11 |
| `siswa_ref` yang bukan anggota kelas ditolak `400`, bukan diabaikan | [SCHEMA.md §11](SCHEMA.md) |
| Sesi kedua pada penugasan dan tanggal yang sama ditolak `409 SESI_SUDAH_ADA` | I-14, P6 |
| Guru ditolak `409 RAPOR_TERKUNCI` apabila rapor kelas sudah final; Administrator dilanjutkan | I-22, P13 |

Kelengkapan I-15 dengan demikian **tidak pernah bergantung pada klien**: apa pun yang dikirim frontend, jumlah baris presensi yang terbentuk selalu sama dengan jumlah anggota kelas. Status `alpa` sebagai bawaan kolom ([SCHEMA.md §4.4](SCHEMA.md)) yang membuatnya cukup dinyatakan sekali.

Dua Guru mata pelajaran berbeda tetap dapat mencatat presensi pada kelas dan tanggal yang sama tanpa bertabrakan, karena keunikan melekat pada penugasan (P6).

**Tombol Hadir Semua adalah tindakan frontend** yang mengubah seluruh baris pada layar menjadi `hadir` sebelum tombol simpan ditekan; tidak ada endpoint tersendiri untuknya (AC-11).

### 7.3 Menyunting sesi yang sudah ada

```http
PUT /api/sesi/:id/presensi
{ "presensi": [ { "siswa_ref": "…", "status": "izin", "catatan": "Surat dokter" }, … ] }
```

```jsonc
// 200
{ "data": { "diperbarui": 30 } }
```

Bentuk `PUT` atas seluruh daftar, dalam satu transaksi. Endpoint ini **hanya memperbarui status**; jumlah baris presensi tidak pernah berubah karena keanggotaannya sudah ditetapkan pada §7.2.

Perubahan berlaku langsung **tanpa pencatatan riwayat** (P7). Tidak ada endpoint riwayat presensi dalam bentuk apa pun.

### 7.4 Menghapus sesi dan melihat ringkasan

| Metode | Alamat | Isi |
|---|---|---|
| `GET` | `/api/penugasan/:id/sesi` | Daftar sesi beserta ringkasan jumlah per status |
| `GET` | `/api/sesi/:id` | Satu sesi beserta seluruh status siswanya |
| `DELETE` | `/api/sesi/:id` | `204`. Seluruh presensi ikut terhapus lewat `ON DELETE CASCADE`, dan persentase kehadiran menyesuaikan sendiri (I-16, AC-25) |
| `GET` | `/api/kelas/:id/presensi` | Ringkasan persentase seluruh siswa lintas mata pelajaran. **Wali Kelas dan Administrator saja** |

`GET /api/kelas/:id/presensi` mengembalikan persentase per siswa per mata pelajaran, **tanpa rincian per tanggal**. Wali Kelas tidak membuka, mengubah, maupun menghapus sesi Guru lain ([aktor-role.md §7](aktor-role.md)), sehingga rinciannya tidak diperlukan.

---

## 8. Rapor

### 8.1 Kesiapan kelas

```http
GET /api/kelas/:id/rapor
```

```jsonc
// 200
{ "data": {
    "kelas": { "id": "…", "nama": "X IPA 1", "periode_nama": "2026/2027 Ganjil" },
    "status": "draft",
    "kelengkapan": [
      { "mapel_nama": "Biologi",    "lengkap": true,  "nilai_terisi": 240, "nilai_diperlukan": 240 },
      { "mapel_nama": "Matematika", "lengkap": false, "nilai_terisi": 180, "nilai_diperlukan": 240 }
    ],
    "rapor": [ { "id": "…", "siswa_ref": "…", "siswa_nama": "Andi Pratama",
                 "catatan_wali": null, "status": "draft" }, … ]
} }
```

Ini layar kesiapan Wali Kelas (UC-11). Kelengkapan dihitung dengan kueri pada [SCHEMA.md §8.2](SCHEMA.md).

**Di luar momen finalisasi, tidak ada pemberitahuan kelengkapan dalam bentuk apa pun** ([PRD §6.3](PRD.md)). Endpoint ini dipanggil ketika Wali Kelas membuka layarnya sendiri; ia bukan sumber notifikasi, dan tidak ada mekanisme dorong ke arah Guru.

### 8.2 Catatan wali

```http
PATCH /api/rapor/:id
{ "catatan_wali": "Andi menunjukkan perkembangan yang baik pada semester ini." }
```

**Catatan bersifat per siswa**, mengikuti `rapor.catatan_wali` pada [RFC-001 §4](RFC-001-model-data-konseptual.md). "Catatan umum" pada [PRD §6.3](PRD.md) dibaca sebagai catatan yang berlaku umum atas **seluruh mata pelajaran** satu siswa — sebagai lawan dari catatan per mata pelajaran — bukan satu catatan untuk seluruh kelas.

Hanya dapat diubah selama rapor berstatus `draft`; sesudahnya `409 RAPOR_TERKUNCI` (I-22, AC-14). Beban penulisan tiga puluh catatan dicatat sebagai temuan **A-02**.

### 8.3 Finalisasi

```http
POST /api/kelas/:id/rapor/finalisasi
```

Berhasil:

```jsonc
// 200
{ "data": { "difinalisasi": 30, "difinalisasi_pada": "2026-12-18T10:15:00+07:00",
            "berkas_terender": 30 } }
```

Ditolak — teks pesan mengikuti AC-07 **kata demi kata**, satu entri per mata pelajaran yang belum lengkap:

```jsonc
// 409
{ "kesalahan": {
    "kode": "MAPEL_BELUM_LENGKAP",
    "pesan": "Rapor belum dapat difinalisasi karena 2 mata pelajaran belum lengkap.",
    "rincian": [
      { "mapel_nama": "Matematika", "pesan": "Data Mapel Matematika belum ada, tolong hubungi guru yang bertanggung jawab." },
      { "mapel_nama": "Fisika",     "pesan": "Data Mapel Fisika belum ada, tolong hubungi guru yang bertanggung jawab." }
    ]
} }
```

**Finalisasi bersifat sekelas, satu transaksi** ([ARCHITECTURE.md §11](ARCHITECTURE.md)). Yang terjadi di dalamnya: pemeriksaan kelengkapan, penulisan `rapor_mapel` beserta `snapshot_komponen` sebagai salinan beku, lalu perubahan status seluruh rapor kelas menjadi `finalized`.

**Setelah `COMMIT`, berkas PDF dirender di dalam request yang sama** (CK-API-12, mengamandemen CK-API-10). Sifatnya *best effort*:

| Ketentuan | Alasan |
|---|---|
| Render berjalan **sesudah** `COMMIT`, tidak di dalam transaksi | Transaksi tidak boleh menggantung selama pekerjaan render |
| Anggaran lunak **20 detik**. Berkas yang belum sempat dirender dilewati | Batas waktu fungsi 30 detik ([ARCHITECTURE.md §6](ARCHITECTURE.md)) tidak boleh terlampaui |
| `berkas_terender` menyebutkan berapa yang jadi. Angka di bawah `difinalisasi` **bukan kegagalan** | Sisanya dirender pada saat diunduh, lewat jalur yang tetap ada (§8.4) |
| Kegagalan render tidak pernah membatalkan finalisasi | Finalisasi sudah `COMMIT` dan sah; berkas hanyalah turunannya |

Dilakukan **satu kali** dan tidak dapat dibatalkan. Tidak ada endpoint buka kembali; upaya menurunkan status juga ditolak `trg_rapor_status_maju` di tingkat basis data ([SCHEMA.md §5.2](SCHEMA.md)).

### 8.4 Distribusi dan unduh

```http
POST /api/kelas/:id/rapor/distribusi
```

Mengubah status seluruh rapor kelas menjadi `distributed`. Sejak saat itu Siswa yang bersangkutan dapat melihat dan mengunduhnya (UC-16, AC-10).

```http
GET /api/rapor/:id/berkas
```

```jsonc
// 200
{ "data": { "url": "https://…", "kedaluwarsa_pada": "2026-12-18T10:20:00+07:00" } }
```

Berkas biasanya **sudah ada** karena dirender pada saat finalisasi (§8.3). Apabila belum ada — karena anggaran render terlampaui, atau karena Administrator mengoreksi data final sehingga berkasnya dihapus (CK-A-05) — PDF dirender saat itu juga lalu disimpan. Yang dikembalikan selalu **presigned URL berumur 5 menit**, bukan isi berkasnya ([ARCHITECTURE.md §11.1](ARCHITECTURE.md)).

Lapis baris: Administrator tanpa batas; Wali Kelas hanya kelas walinya; Siswa hanya rapor miliknya **dan** berstatus `distributed`. Guru Mata Pelajaran ditolak lapis baris pada setiap rapor — tanpa aturan terpisah, sesuai [ARCHITECTURE.md §9.2](ARCHITECTURE.md) dan AC-32.

### 8.5 Unduh sekelas

```http
GET /api/kelas/:id/rapor/berkas
```

```jsonc
// 200
{ "data": { "url": "https://…", "kedaluwarsa_pada": "2026-12-18T10:20:00+07:00",
            "jumlah_rapor": 30 } }
```

Mengembalikan satu arsip ZIP berisi seluruh rapor kelas. **Wali Kelas dan Administrator saja**; Siswa dan Guru Mata Pelajaran ditolak lapis baris.

**Endpoint ini tidak merender apa pun.** Ia mengambil berkas yang sudah ada di S3, menyusunnya menjadi ZIP, mengunggahnya, lalu mengembalikan presigned URL — murni pekerjaan I/O. Inilah yang dimungkinkan oleh pra-render pada §8.3, dan yang membuat CK-API-10 tidak lagi berlaku.

Rapor yang berkasnya belum ada dirender lebih dahulu, dengan anggaran lunak yang sama seperti §8.3. Apabila anggaran terlampaui, permintaan dijawab `409 BERKAS_BELUM_SIAP` beserta jumlah yang sudah siap, dan permintaan berikutnya melanjutkan dari sana — bukan mengulang dari awal, karena berkas yang sudah jadi tetap tersimpan.

**Arsip ZIP tidak pernah dipakai ulang.** Ia dibangun ulang setiap permintaan dan disimpan di bawah awalan `sementara/` yang dihapus aturan daur hidup S3. Dengan begitu arsip tidak dapat menjadi usang setelah Administrator mengoreksi data final — persoalan yang pada berkas per siswa harus diselesaikan CK-A-05.

---

## 9. Siswa

| Metode | Alamat | Isi |
|---|---|---|
| `GET` | `/api/saya/nilai` | Seluruh mata pelajaran pada semester berjalan |
| `GET` | `/api/saya/presensi` | Persentase kehadiran per mata pelajaran |
| `GET` | `/api/saya/rapor` | Rapor yang **sudah didistribusikan** saja |
| `POST` | `/api/saya/suggestion` | Rangkuman dan rekomendasi belajar |

```jsonc
// GET /api/saya/nilai — 200
{ "data": { "periode_nama": "2026/2027 Ganjil", "mapel": [ {
    "mapel_nama": "Biologi", "kkm": 75,
    "komponen": [ { "kode": "T1", "nama": "Tugas 1", "bobot": 6, "nilai": 85, "topik": "Sel dan jaringan" },
                  { "kode": "UAS", "nama": "Ujian Akhir Semester", "bobot": 26, "nilai": null, "topik": null } ],
    "lengkap": false,
    "nilai_akhir": null
} ] } }
```

**`nilai_akhir` bernilai `null` selama data belum lengkap**, dan `lengkap` menyatakan sebabnya. Ini pemenuhan [PRD §8.3](PRD.md) — nilai final tidak ditampilkan sebagai hasil final sebelum data lengkap — dan diputuskan di sisi API, bukan diserahkan kepada frontend, agar angka setengah jadi tidak pernah sampai ke perangkat siswa.

Kelengkapan pada endpoint ini dihitung **per siswa**, berbeda dari kelengkapan sekelas pada §8.1 yang menjadi syarat finalisasi. Keduanya menjawab pertanyaan berbeda dan sengaja dipisahkan.

`GET /api/saya/presensi` mengembalikan persentase per mata pelajaran **tanpa jalur apa pun menuju rincian per tanggal** (AC-30). Izin dan Sakit terhitung sebagai kehadiran; hanya Alpa yang menguranginya (P16, AC-29). Tidak ada peringatan, anjuran, maupun ambang minimum pada respons (NG12, NG16, AC-12).

### 9.1 Tombol Suggestion

```http
POST /api/saya/suggestion
```

```jsonc
// 200
{ "data": {
    "teks": "…satu paragraf rekomendasi diikuti poin-poin ringkas…",
    "periode_nama": "2026/2027 Ganjil",
    "data_sementara": true,
    "dibuat_pada": "2026-08-06T14:30:00+07:00"
} }
```

| Ketentuan | Dasar |
|---|---|
| **Hanya peran `siswa`.** Administrator, Guru, dan Wali Kelas ditolak `403` | [aktor-role.md §6](aktor-role.md) |
| Paling banyak **5 kali per jam per siswa** | [ARCHITECTURE.md §7](ARCHITECTURE.md) |
| Data dibaca lewat koneksi `app_ro` yang tidak dapat menulis dan tidak dapat membaca identitas | I-23, [SCHEMA.md §7.1](SCHEMA.md) |
| Keluaran **tidak disimpan**. Tidak ada `GET` pasangannya dan tidak ada riwayat | I-24, NG14, AC-16 |
| Batas waktu 20 detik | [ARCHITECTURE.md §10](ARCHITECTURE.md) |

`data_sementara` dan `periode_nama` dikembalikan agar halaman dapat menampilkan penanda Data Sementara beserta fakta sumbernya **di sekitar** keluaran AI, bukan menuntutnya menjadi bagian teks model yang susunannya bebas (AC-19).

Kegagalan bersifat **lunak**:

```jsonc
// 503
{ "kesalahan": { "kode": "LAYANAN_AI_GAGAL",
    "pesan": "Rekomendasi tidak dapat dibuat saat ini. Silakan coba beberapa saat lagi." } }
```

Kegagalan ini tidak menghambat apa pun. Nilai, presensi, finalisasi, dan distribusi tetap berjalan (AC-21, [PRD §8.6](PRD.md) butir 7).

---

## 10. Katalog kesalahan

| Kode | HTTP | Kapan | Teks diwajibkan |
|---|:--:|---|:--:|
| `KREDENSIAL_SALAH` | 401 | Nama pengguna atau kata sandi keliru, atau akun tidak aktif | — |
| `SESI_TIDAK_SAH` | 401 | Cookie tidak ada, kedaluwarsa, atau sudah dicabut | — |
| `KEWENANGAN_DITOLAK` | 403 | Gagal lapis peran atau lapis baris | — |
| `PERMINTAAN_TIDAK_SAH` | 400 | Zod menolak bentuk permintaan, termasuk badan maupun parameter query | — |
| `TIDAK_DITEMUKAN` | 404 | Sumber daya yang dirujuk pengenal tidak ada | — |
| `BERKAS_TIDAK_SAH` | 400 | Satu baris atau lebih bermasalah; tidak ada yang tersimpan | AC-26 |
| `BERKAS_TERLALU_BESAR` | 413 | Melampaui 2 MB | — |
| `BOBOT_TIDAK_SERATUS` | 400 | Jumlah bobot bukan 100 | **AC-04** |
| `DATA_SUDAH_ADA` | 409 | Nama pengguna, kode, nama kelas, tahun ajaran, atau semester bertentangan dengan data yang sudah ada | — |
| `GURU_SUDAH_MENGAMPU` | 409 | Guru sudah memiliki mata pelajaran | — |
| `GURU_BELUM_MENGAMPU` | 409 | Guru yang dipilih untuk kelas belum memiliki mata pelajaran | AC-28 |
| `KOMPONEN_SUDAH_DIPAKAI` | 409 | Daftar kode komponen diubah setelah templat dipakai penugasan | — |
| `JENJANG_TIDAK_COCOK` | 409 | Jenjang kelas berbeda dari jenjang mata pelajaran | **AC-24** |
| `SESI_SUDAH_ADA` | 409 | Penugasan dan tanggal yang sama sudah memiliki sesi | — |
| `RAPOR_TERKUNCI` | 409 | Guru atau Wali Kelas mengubah data yang sudah final | AC-14 |
| `MAPEL_BELUM_LENGKAP` | 409 | Finalisasi sebelum seluruh mata pelajaran lengkap | **AC-07** |
| `BERKAS_BELUM_SIAP` | 409 | Unduh sekelas ketika sebagian berkas belum selesai dirender | — |
| `BATAS_LAJU_TERLAMPAUI` | 429 | Salah satu dari tiga jalur terbatas | — |
| `LAYANAN_AI_GAGAL` | 503 | Batas waktu, `5xx`, atau kredit habis | — |
| `KESALAHAN_SERVER` | 500 | Kegagalan tak terduga | — |

Tiga kode bertanda tebal memiliki **teks yang ditetapkan PRD atau kriteria kesiapan** dan tidak boleh diubah redaksinya. Sisanya bebas selama menyebutkan alasan yang dapat dipahami (P21, AC-27).

`429` menyertakan waktu percobaan berikutnya:

```jsonc
{ "kesalahan": { "kode": "BATAS_LAJU_TERLAMPAUI",
    "pesan": "Terlalu banyak percobaan. Silakan coba lagi pada pukul 14.45.",
    "rincian": [ { "coba_lagi_pada": "2026-08-06T14:45:00+07:00" } ] } }
```

`500` **tidak pernah membocorkan pesan asli**, jejak tumpukan, maupun kueri SQL. Rinciannya masuk ke log server, sedangkan pengguna menerima pesan tetap.

---

## 11. Yang sengaja tidak memiliki endpoint

Daftar ini sama mengikatnya dengan daftar endpoint. Ketiadaannya adalah keputusan, bukan kelalaian.

| Tidak tersedia | Dasar |
|---|---|
| Pembuatan dan penggantian kata sandi akun **Administrator** | Perintah CLI, [ARCHITECTURE.md §9.3](ARCHITECTURE.md) |
| Pemulihan kata sandi mandiri, termasuk endpoint yang hanya menjawab pesan | §6.1.3, AC-33 |
| Unduh nilai maupun rapor bagi Guru Mata Pelajaran | P23, AC-32 |
| Finalisasi oleh Guru Mata Pelajaran dalam bentuk apa pun | P9, AC-08 |
| Buka kembali atau terbitkan ulang rapor | [PRD §9](PRD.md), I-21 |
| Riwayat rekomendasi AI | NG14, I-24 |
| Riwayat perubahan nilai maupun presensi | P7, CK-A-06 |
| Rincian presensi per tanggal bagi Siswa | AC-30 |
| Penulisan apa pun oleh jalur AI | I-23, AC-20 |
| Pergantian guru pengampu di tengah semester | T-05, belum ada alurnya di PRD |
| Penonaktifan akun | Kolom `pengguna.aktif` ada, alurnya belum ditetapkan — temuan **A-04** |
| Penambahan atau pemindahan siswa setelah kelas terbentuk | Ditetapkan di luar cakupan MVP — §13.2 **A-05** |
| Kenaikan kelas dan perpindahan tahun ajaran | NG8 |

---

## 12. Pemenuhan use case

Pertanggungjawaban langsung terhadap [ATURAN-DAN-KRITERIA §2](ATURAN-DAN-KRITERIA.md). Setiap use case harus memiliki endpoint yang dapat ditunjuk.

| UC | Endpoint |
|---|---|
| UC-01 | `POST /api/auth/masuk` · `POST /api/auth/keluar` · `GET /api/saya` |
| UC-02 | `POST /api/kelas/pratinjau` · `POST /api/kelas` |
| UC-03 | `POST /api/pengguna` · `POST /api/pengguna/unggah` |
| UC-04 | `POST /api/mapel` · `POST /api/kelas` |
| UC-05 | `POST /api/mapel` · `PATCH /api/mapel/:id` |
| UC-06 | `POST /api/penugasan/:id/nilai` sebagai Administrator |
| UC-07 | `POST /api/penugasan/:id/nilai` |
| UC-08 | `GET /api/penugasan/:id/siswa` · `POST /api/penugasan/:id/sesi` · `PUT /api/sesi/:id/presensi` · `DELETE /api/sesi/:id` |
| UC-09 | `GET /api/saya/nilai` — tanpa langkah publikasi di antaranya |
| UC-10 | `GET /api/kelas/:id/nilai` |
| UC-11 | `GET /api/kelas/:id/rapor` |
| UC-12 | `POST /api/kelas/:id/rapor/finalisasi` |
| UC-13 | `POST /api/kelas/:id/rapor/distribusi` · `GET /api/rapor/:id/berkas` · `GET /api/kelas/:id/rapor/berkas` |
| UC-14 | `GET /api/saya/nilai` · `GET /api/saya/presensi` |
| UC-15 | `POST /api/saya/suggestion` |
| UC-16 | `GET /api/saya/rapor` · `GET /api/rapor/:id/berkas` |

Seluruh enam belas use case tertutup. Seluruh layar pada [ATURAN-DAN-KRITERIA §3](ATURAN-DAN-KRITERIA.md) memperoleh datanya dari endpoint di atas, kecuali halaman lupa kata sandi dan halaman akses ditolak yang sepenuhnya statis.

---

## 13. Temuan

Celah yang ditemukan saat menurunkan kontrak.

### 13.1 Masih terbuka

| # | Temuan | Usulan tindakan |
|---|---|---|
| A-04 | `pengguna.aktif` ada pada model data dan diperiksa saat masuk, tetapi **tidak ada aktor, alur, maupun layar yang mengubahnya menjadi `false`**. [aktor-role.md §5.1](aktor-role.md) tidak memuat kewenangan penonaktifan, dan [ATURAN-DAN-KRITERIA §3](ATURAN-DAN-KRITERIA.md) tidak memuat layarnya. Akibatnya kolom itu tidak akan pernah bernilai `false`, dan akun Guru yang keluar dari sekolah tetap dapat masuk | Nyatakan pada PRD bahwa kolom tersebut **belum dipakai pada MVP**. Pemeriksaan saat masuk tetap dipertahankan sebagai pertahanan. Penanganan sementara bagi akun yang perlu ditutup: Administrator mereset kata sandinya dan tidak menyerahkannya. Menambah endpoint penonaktifan berarti menambah kewenangan Administrator yang tidak diberikan PRD |

### 13.2 Sudah ditutup

| # | Temuan | Penutupan |
|---|---|---|
| A-01 | Dugaan pertentangan antara P22 dan [ARCHITECTURE.md §1.2](ARCHITECTURE.md) mengenai kapan sesi presensi tersimpan | **Gugur.** Pertentangan itu berasal dari CK-API-05, bukan dari dokumen sumber. CK-API-11 menempatkan pembuatan sesi pada transaksi Simpan Presensi, sehingga P22, AC-15, [PRD §8.4](PRD.md), dan ARCHITECTURE §1.2 terpenuhi seluruhnya tanpa satu pun perlu ditafsir ulang. Tidak ada usulan perubahan bagi PRD |
| A-02 | Beban penulisan tiga puluh catatan wali per kelas | **Ditetapkan.** Catatan wali bersifat per siswa karena melekat pada rapor siswa yang bersangkutan. Sesuai [RFC-001 §4](RFC-001-model-data-konseptual.md); tidak memerlukan amandemen |
| A-03 | AC-26 menyebut "berhasil sebagian" sekaligus melarang data setengah jadi | **Ditutup 10 Agustus 2026.** [ATURAN-DAN-KRITERIA.md §4](ATURAN-DAN-KRITERIA.md) kini menetapkan unggahan dengan baris bermasalah ditolak seluruhnya. Ini selaras dengan CK-API-02 dan kontrak §5.2 |
| A-06 | ~~§3.1 dan ARCHITECTURE Pasal 7 bertentangan mengenai ambang pembatas laju per alamat IP~~ | **Ditutup 8 Agustus 2026 oleh CK-A-08.** Keduanya tidak benar-benar bertentangan: ARCHITECTURE menuntut ambang IP yang **berbeda** dari ambang per akun, bukan meniadakannya. Ditetapkan dua lapis — 5 per akun, 30 per alamat IP — dan §3.1 disesuaikan. Tinjauan keamanan A4 menegaskan bahwa lapis per akun saja tidak menahan password spraying atas pengenal NIP dan NIS yang berpola |
| A-05 | Siswa pindahan di tengah semester tidak memiliki jalur masuk | **Ditetapkan sebagai asumsi.** Tidak ada perpindahan siswa di tengah semester selama pilot, dan penanganannya berada di luar cakupan MVP bersama NG8. Berkedudukan sederajat dengan asumsi I-08 |

Temuan yang masih terbuka pada dokumen sebelumnya — T-02, T-04, T-05, dan T-06 pada [RFC-001 §10](RFC-001-model-data-konseptual.md), serta S-01 sampai S-04 pada [SCHEMA.md §12](SCHEMA.md) — tidak dipengaruhi keputusan pada dokumen ini. **S-05 ditutup** oleh §6.3.

### 13.3 Yang wajib diukur

Satu angka menentukan apakah CK-API-12 bertahan, dan **belum pernah diukur siapa pun**: lama render tiga puluh PDF pdfmake berurutan beserta unggahannya ke S3, di dalam fungsi Lambda 1024 MB arm64.

Diuji sejak berkas rapor pertama dapat dirender, bukan ditemukan saat Wali Kelas pertama memfinalisasi kelas sungguhan. Apabila melampaui anggaran lunak 20 detik secara konsisten, yang berubah hanya **jumlah berkas yang sempat dirender** — jalur render-saat-unduh tetap menutupinya, dan tidak ada satu pun endpoint yang perlu diubah.

---

## Lampiran — Catatan Keputusan

Bernomor dan bertanggal. Entri tidak disunting; perubahan keputusan ditulis sebagai entri baru yang menyebut nomor yang digantikannya.

Penomoran memakai awalan `CK-API-` sehingga tidak bertabrakan dengan `CK-xx`, `CK-A-xx`, `CK-D-xx`, maupun `CK-S-xx`.

### CK-API-01 · 6 Agustus 2026 · Amplop `data` dan `kesalahan`, pesan siap tampil

**Diputuskan.** Seluruh respons memakai amplop `{ "data": … }` atau `{ "kesalahan": { kode, pesan, rincian } }`. `pesan` selalu berbahasa Indonesia dan siap ditampilkan kepada pengguna akhir.

**Alasan.** P21 dan AC-27 mewajibkan setiap kegagalan menyebutkan alasannya, dan tiga di antaranya memiliki teks yang ditetapkan PRD kata demi kata. Menempatkan penyusunan pesan di frontend berarti teks wajib tersebut hidup di dua tempat dan dapat menyimpang. Dengan pesan disusun server, `kode` cukup dipakai frontend untuk menentukan perlakuan — misalnya menyorot baris berkas yang gagal — bukan untuk menerjemahkan ulang.

**Alternatif yang ditolak.** *RFC 7807 `application/problem+json`.* Bentuk baku yang lazim. Ditolak karena bidang `type` berupa URI mengandaikan dokumentasi kesalahan yang dapat dihubungi, sedangkan NG2 mencabut konsumen di luar frontend sendiri. *Respons telanjang tanpa amplop.* Menghemat satu lapis, tetapi membuat penanganan berhasil dan gagal tidak seragam di sisi frontend.

### CK-API-02 · 6 Agustus 2026 · Unggah bersifat tolak seluruhnya

**Diputuskan.** Satu baris bermasalah membatalkan seluruh berkas. Tidak ada akun maupun anggota kelas yang terbentuk sebagian.

**Alasan.** AC-26 memuat dua tuntutan yang tidak dapat dipenuhi bersamaan: melaporkan baris yang gagal pada unggahan yang "berhasil sebagian", dan tidak menyisakan akun setengah jadi. Larangan yang kedua dimenangkan karena akibatnya lebih berat. Unggahan yang menerima sebagian memaksa Administrator menelusuri akun mana yang sudah ada sebelum mengunggah ulang, dan setiap pengulangan menghasilkan kata sandi awal baru bagi akun yang sudah terlanjur dibuat — persoalan yang tidak dimiliki pendekatan tolak seluruhnya.

**Alternatif yang ditolak.** *Menerima baris yang sah dan melaporkan sisanya.* Membaca AC-26 secara harfiah pada bagian pertamanya. Ditolak dengan alasan di atas. *Parameter `lanjutkan_meski_gagal`.* Menyerahkan pilihan kepada Administrator, tetapi menghadirkan dua perilaku yang harus diuji dan didokumentasikan untuk kebutuhan yang belum pernah dinyatakan.

**Konsekuensi yang diterima.** Berkas 360 baris dengan satu NIS ganda harus diperbaiki dan diunggah ulang seluruhnya. Endpoint pratinjau pada §5.7 mengurangi biaya ini untuk daftar siswa; unggahan akun belum memiliki padanannya.

### CK-API-03 · 6 Agustus 2026 · Pembuatan kelas berupa satu endpoint atomik

**Diputuskan.** `POST /api/kelas` membuat kelas, keanggotaan siswa, penugasan, pasangan penugasan-komponen, dan rapor draf dalam satu transaksi. Didahului `POST /api/kelas/pratinjau` yang tidak menulis apa pun.

**Alasan.** [PRD §6.1.5](PRD.md) menggambarkan empat langkah yang "dapat dibatalkan atau dilanjutkan", sedangkan P18 dan AC-26 melarang kelas setengah jadi. Keduanya terpenuhi sekaligus apabila langkah-langkah tersebut adalah **langkah antarmuka**, bukan langkah penyimpanan: proses yang ditinggalkan tidak menyisakan apa pun karena belum ada yang tertulis.

**Alternatif yang ditolak.** *Kelas berstatus draf yang diselesaikan bertahap.* Mengikuti urutan PRD secara harfiah. Ditolak karena menghadirkan kelas yang ada tetapi belum sah, sehingga setiap kueri di seluruh sistem perlu menyaringnya, dan draf yang tertinggal menjadi persoalan pembersihan yang tidak dimiliki siapa pun. *Empat endpoint tanpa draf.* Menghasilkan kelas tanpa siswa apabila langkah kedua gagal — persis yang dilarang AC-26.

**Konsekuensi yang diterima.** Antarmuka wajib mengumpulkan seluruh masukan sebelum mengirim, dan berkas daftar siswa dikirim dua kali — sekali untuk pratinjau, sekali untuk pembuatan. Pada berkas berukuran puluhan kilobyte, ini tidak berarti.

### CK-API-04 · 6 Agustus 2026 · Kata sandi awal diserahkan sebagai berkas CSV sekali unduh

**Diputuskan.** Unggahan akun yang berhasil menjawab dengan `text/csv` berisi nama, nama pengguna, dan kata sandi awal, disertai `Cache-Control: no-store`. Pembuatan manual satu akun menjawab dengan JSON biasa.

**Alasan.** [Techstack.md §5](Techstack.md) menetapkan kata sandi awal "ditampilkan sekali kepada Administrator" tanpa menetapkan bentuknya. Untuk 360 akun sekaligus, menampilkannya di layar berarti Administrator menyalinnya satu per satu, dan menutup halaman berarti kehilangan seluruhnya. Berkas dapat disimpan, dicetak, dan dipotong per kelas untuk diserahkan.

Pemisahan bentuk respons juga membuat kedua keadaan tidak dapat tertukar: berhasil selalu `text/csv`, gagal selalu JSON. Frontend membedakannya dari `Content-Type`, bukan dari isi.

**Alternatif yang ditolak.** *JSON berisi larik 360 kata sandi.* Konsisten dengan endpoint lain, tetapi memaksa frontend menyusun sendiri berkas yang akhirnya tetap diperlukan. *Tautan unduh sekali pakai.* Menuntut penyimpanan sementara berisi kata sandi teks terang beserta masa berlakunya — tempat penyimpanan baru yang justru ingin dihindari.

**Konsekuensi yang diterima.** Berkas berisi kata sandi teks terang berada di perangkat Administrator dan tidak dapat ditarik kembali. Dikurangi dengan tidak adanya salinan di sisi server, dan dengan tersedianya reset per akun pada §5.3. Kewajiban penanganannya termasuk yang perlu divalidasi bersama sekolah pada V6.

### CK-API-05 · 6 Agustus 2026 · Sesi presensi ditulis pada saat dibuka

**Diputuskan.** `POST /api/penugasan/:id/sesi` menulis baris `sesi` beserta satu baris `presensi` berstatus `alpa` untuk setiap siswa kelas, dalam satu transaksi. `PUT /api/sesi/:id/presensi` hanya memperbarui status.

**Alasan.** [PRD §8.4](PRD.md) butir 2 menyatakan "status kosong tidak mungkin terjadi, karena sesi selalu terbuka dengan seluruh siswa berstatus Alpa", dan AC-11 menjadikannya kriteria kesiapan. Pernyataan itu hanya benar apabila baris presensi sudah ada sejak sesi terbuka; menundanya sampai tombol simpan menghasilkan sesi tanpa satu pun status — persis keadaan yang dinyatakan mustahil. [ARCHITECTURE.md §1.2](ARCHITECTURE.md) sudah menempatkan kelengkapan I-15 pada transaksi pembukaan sesi.

P22 karenanya dibaca berlaku atas **status kehadiran**, bukan atas keberadaan sesi. Pembukaan sesi adalah tindakan sadar Guru yang memilih kelas dan tanggal, bukan penyimpanan otomatis atas perubahan yang belum dikonfirmasi — dan penyimpanan otomatis itulah yang dilarang P22 beserta AC-15.

**Alternatif yang ditolak.** *Sesi bersifat lokal sampai Simpan Presensi ditekan.* Membaca P22 secara harfiah. Ditolak karena mencabut satu-satunya cara menjamin I-15, dan menghadirkan keadaan "sesi ada tetapi tanpa status" yang dilarang PRD. *Menyisipkan baris presensi hanya bagi siswa yang statusnya diubah.* Menghasilkan penyebut persentase yang berbeda antar siswa dalam satu sesi, sehingga I-18 kehilangan arti.

**Konsekuensi yang diterima.** Sesi yang dibuka lalu ditinggalkan masuk ke penyebut kehadiran dengan seluruh siswa berstatus Alpa. Diselesaikan Guru dengan menghapus sesi tersebut, jalur yang memang tersedia dan tidak memerlukan siapa pun selain dirinya. Penafsiran P22 ini diajukan ke PRD sebagai temuan A-01.

### CK-API-06 · 6 Agustus 2026 · Penggantian kata sandi mandiri tersedia bagi seluruh peran

**Diputuskan.** `PATCH /api/saya/kata-sandi` tersedia bagi Administrator, Guru, dan Siswa, mensyaratkan kata sandi lama, dan mencabut seluruh sesi lain milik pengguna tersebut.

**Alasan.** [PRD §6.1.3](PRD.md) meniadakan **pemulihan** kata sandi secara mandiri — jalur yang dapat dipakai seseorang tanpa mengetahui kata sandi sebelumnya. Penggantian oleh pemilik yang sudah masuk dan mengetahui kata sandi lamanya bukan pemulihan, tidak membuka jalur yang dapat disalahgunakan, dan tidak bertentangan dengan ketentuan mana pun. Layar "profil" pada [ATURAN-DAN-KRITERIA §3](ATURAN-DAN-KRITERIA.md) tersedia bagi seluruh peran tanpa isi yang ditetapkan; inilah isinya yang paling jelas diperlukan.

Ketersediaannya juga mengurangi beban Administrator, yang tanpa ini menjadi satu-satunya jalur bagi setiap pengguna yang ingin mengganti kata sandi awal buatan sistem.

**Alternatif yang ditolak.** *Tidak menyediakannya sama sekali.* Membaca §6.1.3 seluas mungkin. Ditolak karena menempatkan seluruh penggantian kata sandi 379 pengguna pada satu orang, tanpa satu pun ketentuan yang menuntutnya. *Menyediakannya tanpa kata sandi lama.* Menjadikan sesi yang dicuri sebagai jalur pengambilalihan akun permanen.

**Konsekuensi yang diterima.** Administrator tidak lagi mengetahui kata sandi pengguna yang sudah menggantinya sendiri. Ini justru yang dikehendaki; reset pada §5.3 tetap menjadi jalur ketika akun perlu dipulihkan.

### CK-API-07 · 6 Agustus 2026 · Baris rapor dibuat massal pada saat kelas dibuat

**Diputuskan.** `POST /api/kelas` membuat satu baris `rapor` berstatus `draft` per siswa. Finalisasi memperbarui baris yang sudah ada, bukan membuatnya.

**Alasan.** [PRD §9](PRD.md) menetapkan `draft` sebagai status yang berlaku, bukan sebagai ketiadaan baris. Tanpa baris yang sudah ada, `PATCH /api/rapor/:id` untuk catatan wali tidak memiliki sasaran, dan layar kesiapan pada §8.1 harus membedakan siswa yang rapornya belum ada dari yang catatannya belum ditulis — pembedaan yang tidak berarti apa pun bagi pengguna.

Pembuatan massal juga membuat I-19 ditegakkan sejak awal: `uq_rapor_siswa_periode` memastikan tidak ada rapor kedua, dan keanggotaan kelas sudah pasti karena dibuat pada transaksi yang sama.

**Alternatif yang ditolak.** *Membuat baris rapor pada saat finalisasi.* Menghemat 360 baris kosong per semester. Ditolak karena menjadikan `draft` status yang tidak pernah tersimpan, sehingga catatan wali tidak memiliki tempat sebelum finalisasi — padahal urutan pada [PRD §6.3](PRD.md) menempatkan penulisan catatan **sebelum** finalisasi. *Membuatnya secara malas ketika catatan pertama ditulis.* Menyembunyikan pembuatan sumber daya di dalam operasi pembaruan.

**Konsekuensi yang diterima.** Rapor berstatus `draft` ada bagi siswa yang datanya belum lengkap. Ini tidak terlihat siswa: `GET /api/saya/rapor` hanya mengembalikan yang berstatus `distributed`.

### CK-API-08 · 6 Agustus 2026 · Topik menumpang pada payload Simpan Nilai

**Diputuskan.** `POST /api/penugasan/:id/nilai` menerima bidang `topik` di samping `nilai`, dan keduanya tersimpan dalam satu transaksi.

**Alasan.** T-01 pada [RFC-001 §10](RFC-001-model-data-konseptual.md) mencatat bahwa [PRD §8.5](PRD.md) menuntut AI membaca topik, tetapi tidak ada layar yang mengisinya, dan mengusulkan penambahan kolom topik pada layar pengisian nilai. Karena keduanya berada pada satu layar, keduanya harus tersimpan oleh satu tombol: P22 menetapkan penyimpanan terjadi ketika tombol simpan ditekan, dan dua tombol pada satu layar bertentangan dengan bentuk itu.

**Alternatif yang ditolak.** *`PUT /api/penugasan/:id/komponen/:kode/topik` tersendiri.* Lebih rapi sebagai sumber daya, tetapi menuntut penyimpanan kedua yang tidak terlihat pengguna, dan membuka keadaan topik tersimpan sementara nilai gagal.

**Konsekuensi yang diterima.** Payload Simpan Nilai memuat dua jenis data. Ditandai jelas sebagai dua bidang terpisah, dan keduanya tetap berada dalam satu transaksi.

### CK-API-09 · 6 Agustus 2026 · Tanpa awalan versi dan tanpa paginasi

**Diputuskan.** Alamat tidak memuat `/v1`, dan tidak ada endpoint daftar yang berparameter halaman.

**Alasan.** Keduanya berasal dari sebab yang sama: NG2 mencabut konsumen di luar frontend yang di-deploy bersama API ini, dan [RFC-001 §8.1](RFC-001-model-data-konseptual.md) menetapkan volume yang seluruhnya muat dalam satu respons. Awalan versi menjanjikan kestabilan kepada pihak yang tidak ada; paginasi membangun mekanisme beserta pengujiannya untuk keadaan yang tidak terjadi.

**Konsekuensi yang diterima.** Apabila kelak muncul konsumen di luar frontend — hal yang saat ini dicoret NG2 — versi diperkenalkan lewat entri baru. Paginasi ditambahkan sebagai parameter `kursor` dan `batas` yang bersifat opsional, tanpa mengubah bentuk amplop maupun endpoint yang sudah ada.

### CK-API-10 · 6 Agustus 2026 · Unduh rapor hanya per siswa

**Diputuskan.** Tidak ada endpoint yang mengembalikan seluruh rapor satu kelas dalam satu berkas.

**Alasan.** CK-09 memindahkan render PDF ke saat unduhan justru untuk menghapus kebutuhan pekerjaan latar. Endpoint unduh sekelas menghadirkan kembali tiga puluh pekerjaan render dalam satu tindakan, beserta pemantauan progres, pengulangan, dan pelaporan kegagalannya — kompleksitas yang sama persis dengan yang ditolak, hanya berpindah tempat.

**Alternatif yang ditolak.** *Arsip ZIP yang dirender saat diminta.* Ditolak dengan alasan di atas, ditambah batas waktu fungsi 30 detik pada [ARCHITECTURE.md §6](ARCHITECTURE.md) yang tidak memadai untuk tiga puluh render berurutan.

**Konsekuensi yang diterima.** Wali Kelas menekan unduh tiga puluh kali apabila memerlukan seluruh kelas. Perlu diperiksa saat UAT apakah ini mengganggu; apabila ya, jalur naiknya adalah pekerjaan latar berbasis `pg-boss` (CK-07) yang ditetapkan lewat entri baru.

### CK-API-11 · 6 Agustus 2026 · Sesi presensi ditulis oleh transaksi Simpan Presensi — mengamandemen CK-API-05

**Diputuskan.** `POST /api/penugasan/:id/sesi` menerima tanggal **beserta seluruh status kehadiran**, dan membuat baris `sesi` bersama seluruh baris `presensi` dalam satu transaksi. Tidak ada endpoint yang menulis pada saat Guru "membuka" sesi; pembukaan sesi adalah keadaan layar.

**Yang berubah dari CK-API-05.** Entri tersebut membaca "sesi terbuka dan seluruh siswa langsung memperoleh status Alpa" ([PRD §8.4](PRD.md) langkah 3) sebagai penulisan ke basis data, sehingga P22 harus ditafsirkan berlaku atas status saja. Penafsiran itu **tidak diperlukan**. Membaca langkah 3 sebagai keadaan layar memenuhi seluruh ketentuan tanpa satu pun perlu ditafsir ulang:

| Ketentuan | Terpenuhi karena |
|---|---|
| P22 dan AC-15 — tidak ada yang tersimpan sebelum tombol ditekan | Benar secara harfiah. Layar yang ditinggalkan tidak meninggalkan satu baris pun |
| I-15 dan AC-11 — setiap siswa tepat satu status | Server menurunkan daftar siswa dari `kelas_siswa`, bukan dari payload. Kelengkapan tidak pernah bergantung pada klien |
| I-14 dan P6 — satu sesi per penugasan per tanggal | Unique constraint menolak pada saat penyisipan |
| [ARCHITECTURE.md §1.2](ARCHITECTURE.md) — kelengkapan dijamin transaksi pembukaan sesi | Tetap akurat. Transaksi yang membuat baris `sesi` memang menyisipkan seluruh siswa sekaligus |

**Alasan.** Selain menghapus kebutuhan menafsir ulang PRD, bentuk ini menghilangkan satu cacat nyata pada CK-API-05: sesi yang terlanjur dibuka lalu ditinggalkan tidak lagi masuk ke penyebut persentase kehadiran (I-18) dengan seluruh siswa berstatus Alpa. Pada CK-API-05 keadaan itu hanya dapat diperbaiki dengan menghapus sesi; di sini keadaan itu tidak pernah terjadi.

**Alternatif yang ditolak.** *Mempertahankan CK-API-05.* Duplikat tanggal ketahuan lebih awal, yaitu pada saat membuka alih-alih pada saat menyimpan. Ditolak karena keuntungan itu dapat diperoleh dari `GET /api/penugasan/:id/sesi` di sisi layar, sedangkan harganya adalah penafsiran ulang P22 beserta cacat sesi terlantar.

**Konsekuensi yang diterima.** Bentrokan tanggal dilaporkan pada saat menyimpan, sesudah Guru mengisi layar. Dikurangi dengan memuat sesi yang sudah ada untuk disunting ketika tanggalnya dipilih (§7.1), sehingga bentrokan sungguhan hanya terjadi pada perlombaan dua peramban milik Guru yang sama.

### CK-API-12 · 6 Agustus 2026 · Berkas rapor dirender pada saat finalisasi, dan unduh sekelas tersedia — mengamandemen CK-API-10 dan CK-09

**Diputuskan.** Finalisasi merender seluruh berkas rapor kelas sesudah `COMMIT`, di dalam request yang sama, dengan anggaran lunak 20 detik. Ditambahkan `GET /api/kelas/:id/rapor/berkas` yang mengembalikan arsip ZIP sekelas. Jalur render-saat-unduh **tetap ada** dan tidak berubah.

**Yang berubah dari CK-09 dan CK-API-10.** Keduanya menolak pembuatan berkas sekelas dengan dua alasan. Keduanya ditinjau ulang:

| Alasan penolakan | Status setelah ditinjau |
|---|---|
| Menuntut pemrosesan latar beserta tampilan progres | **Gugur, dengan syarat.** Tiga puluh render berurutan diperkirakan muat di dalam satu request berbatas 30 detik, sehingga tidak memerlukan antrean maupun worker. Angkanya belum diukur dan menjadi kewajiban pengukuran pada §13.3. Anggaran lunak 20 detik membuat kegagalan perkiraan ini tidak berakibat apa pun selain berkas yang dirender belakangan |
| Tidak memberi manfaat produk karena rapor tidak selalu diunduh seluruhnya | **Gugur.** Manfaatnya bukan pada kecepatan unduhan per siswa, melainkan pada terbukanya unduh sekelas: dengan berkas sudah tersedia, endpoint ZIP menjadi **murni I/O** dan muat di dalam anggaran request. Tanpa pra-render, endpoint yang sama berarti tiga puluh render dalam satu permintaan — persis yang ditolak CK-09 |

**Alasan.** CK-API-10 menerima konsekuensi bahwa Wali Kelas menekan unduh tiga puluh kali, dan mencatatnya untuk diperiksa saat UAT. Pemeriksaan itu tidak perlu ditunggu: tiga puluh tindakan berulang untuk satu pekerjaan tunggal adalah beban yang sudah dapat dinilai sekarang. Yang membuatnya mahal pada rancangan sebelumnya adalah render, dan render itulah yang dipindahkan.

**Yang tidak berubah.** CK-A-05 tetap berlaku: koreksi Administrator atas data final menghapus berkas terkait, dan unduhan berikutnya merender ulang. Karena itu jalur render-saat-unduh tidak pernah dapat dihapus, dan pra-render **selalu bersifat tambahan, tidak pernah menggantikan**. CK-07 juga tetap berlaku penuh — tidak ada antrean, tidak ada worker, dan tidak ada pekerjaan latar yang ditambahkan.

**Alternatif yang ditolak.** *Pra-render tanpa endpoint ZIP.* Mempercepat unduhan pertama, tetapi Wali Kelas tetap menekan unduh tiga puluh kali — masalah yang sesungguhnya tidak tersentuh. *Endpoint ZIP tanpa pra-render.* Menghadirkan tiga puluh render di dalam satu permintaan, yaitu keadaan yang ditolak CK-09 tanpa perubahan keadaan apa pun yang membenarkannya. *Arsip ZIP yang disimpan dan dipakai ulang.* Menghemat penyusunan berulang, tetapi menghadirkan berkas kedua yang dapat menjadi usang setelah koreksi Administrator, sehingga CK-A-05 harus diperluas untuk menghapusnya juga.

**Konsekuensi yang diterima.**

1. **Request finalisasi menjadi panjang**, dari ratusan milidetik menjadi belasan detik. Terjadi sekali per kelas per semester dan menghasilkan hasil yang dapat langsung dipakai, sehingga tunggu itu dibayar pada saat yang tepat.
2. **Berkas dirender bagi rapor yang mungkin tidak pernah diunduh.** Biayanya adalah waktu compute yang menyumbang sekitar 3% tagihan ([Techstack.md §8.3](Techstack.md)) dan penyimpanan S3 yang tidak berarti pada volume ini.
3. **Perkiraan lama render belum terbukti.** Ditangani anggaran lunak, bukan diasumsikan aman.

### CK-API-13 · 10 Agustus 2026 · Templat akun dipilih dengan parameter peran wajib

**Diputuskan.** `GET /api/templat/pengguna.csv` mensyaratkan parameter query `peran=guru|siswa`. Guru menerima kepala `Nama,NIP`, sedangkan Siswa menerima kepala `Nama,NIS`. Parameter yang hilang maupun nilai lain ditolak; server tidak memilihkan peran bawaan.

**Alasan.** Unggahan akun pada §5.2 sudah membedakan kolom pengenal berdasarkan peran. Templat yang tidak membawa pilihan peran tidak memiliki satu bentuk CSV yang benar, sedangkan memilih salah satu secara bawaan membuat Administrator dapat mengunduh templat yang salah tanpa menyadarinya.

**Alternatif yang ditolak.** *Dua alamat `/guru.csv` dan `/siswa.csv`.* Menambah alamat untuk perbedaan yang sudah dinyatakan oleh parameter `peran` pada endpoint unggah. *Satu templat `Nama,NIP,NIS`.* Selalu menyisakan satu kolom yang tidak berlaku dan mengaburkan pemeriksaan kepala yang ketat. *Default Guru.* Menebak maksud pemanggil dan menghasilkan kegagalan baru ketika templat itu dipakai untuk Siswa.

### CK-API-14 · 10 Agustus 2026 · Pratinjau kelas membawa periode sasaran

**Diputuskan.** `POST /api/kelas/pratinjau` mensyaratkan bidang multipart `periode_ref` di samping berkas XLSX. Pengenal itu adalah periode yang sama dengan `data.periode_ref` pada `POST /api/kelas` berikutnya.

**Alasan.** Pratinjau pada §5.7 wajib melaporkan siswa yang sudah berada di kelas lain pada semester sasaran, sedangkan I-08 membatasi keanggotaan berdasarkan periode. Berkas hanya memuat Kelas, NIS, dan Nama; tanpa `periode_ref` server tidak memiliki dasar untuk memilih semester yang diperiksa.

**Alternatif yang ditolak.** *Memakai periode aktif.* Administrator dapat menyiapkan semester berikutnya sebelum diaktifkan, sehingga periode aktif dapat berbeda dari sasaran. *Menunda pemeriksaan I-08 sampai pembuatan.* Menghilangkan salah satu masalah yang secara eksplisit dijanjikan respons pratinjau dan memindahkan kegagalan ke langkah terakhir.

### CK-API-15 · 10 Agustus 2026 · Kode komponen stabil setelah dipakai

**Diputuskan.** Sebelum ada satu pun `penugasan_komponen`, `PUT /api/komponen-penilaian` boleh mengganti seluruh daftar termasuk kode. Sesudah templat dipakai, himpunan kode membeku; nama, bobot, dan urutan tetap dapat diubah selama total bobot 100. Pembuatan kelas menyalin sejumlah komponen yang berlaku saat itu, bukan angka delapan yang ditulis tetap di kode.

**Alasan.** `penugasan_komponen` dan `nilai` memiliki foreign key `ON DELETE RESTRICT` ke komponen. Menghapus baris yang sudah dipakai akan memutus topik dan nilai yang ada, sedangkan mengubah bobot dan nama di tempat mempertahankan identitas referensial serta memenuhi kewenangan Administrator pada AC-04.

**Alternatif yang ditolak.** *Selalu menghapus lalu menyisipkan ulang.* Gagal segera setelah komponen direferensikan. *Menghapus nilai dan topik terkait.* Menghilangkan data akademik tanpa alur maupun kewenangan pada PRD. *Mengunci seluruh perubahan setelah kelas ada.* Terlalu luas karena bobot dan nama masih dapat diperbarui aman dengan identitas yang sama.

### CK-API-16 · 10 Agustus 2026 · Guru tanpa mata pelajaran ditolak saat kelas dibuat

**Diputuskan.** Setiap Guru pada `guru_ref` yang belum memiliki baris `mapel` membuat `POST /api/kelas` ditolak seluruhnya dengan `409 GURU_BELUM_MENGAMPU`. Tidak ada `mapel_ref` bawaan dan Guru tersebut tidak dilewati diam-diam.

**Alasan.** Mata pelajaran sengaja tidak ada pada payload kelas karena harus diturunkan dari Guru. Bagi Guru yang belum dihubungkan dengan mata pelajaran, sumber itu memang belum ada; keadaan akunnya tetap sah menurut PRD §6.1.1 dan AC-28, tetapi belum sah dipilih untuk membentuk penugasan.

**Alternatif yang ditolak.** *Melewati Guru tersebut.* Menghasilkan kelas yang berbeda dari permintaan Administrator. *Meminta `mapel_ref` pada payload.* Mengulang pilihan yang menurut PRD §8.2 tidak pernah dilakukan pada lapis kelas. *Menjawab 404.* Gurunya ada; yang belum terpenuhi adalah prasyarat penugasannya.

### CK-API-17 · 10 Agustus 2026 · Administrasi akun HTTP hanya mencakup Guru dan Siswa

**Diputuskan.** `GET /api/pengguna` hanya mendaftarkan Guru dan Siswa. `POST /api/pengguna/:id/kata-sandi` hanya mereset kedua peran itu dan menjawab 404 bagi pengenal Administrator. Hash kata sandi tidak pernah masuk bentuk daftar.

**Alasan.** Kewenangan pengelolaan akun pada PRD §8.1 dan aktor-role §5.1 menyebut akun Guru dan Siswa. Akun Administrator sengaja tidak memiliki endpoint pembuatan; membuka reset antarsesama Administrator melalui endpoint umum akan menciptakan kewenangan baru yang tidak diberikan dokumen sumber.

**Alternatif yang ditolak.** *Mendaftarkan seluruh pengguna lalu menyembunyikan tombol.* Menyerahkan batas kewenangan kepada frontend. *Mengizinkan reset Administrator lain.* Menambahkan jalur pengambilalihan akun istimewa tanpa alur maupun layar. *Menjawab 403 bagi ID Administrator.* Mengungkap bahwa ID tersebut Administrator, padahal sumber dayanya memang berada di luar koleksi yang dikelola endpoint ini.

### CK-API-18 · 10 Agustus 2026 · Batas parser berkas gagal tertutup

**Diputuskan.** CSV dan XLSX memuat paling banyak 360 baris data. Record CSV paling banyak 4 KiB. XLSX memiliki tepat satu worksheet, paling banyak 64 entry ZIP, dan total ukuran entry sebelum kompresi paling banyak 16 MiB. Pelanggaran dijawab `400 BERKAS_TIDAK_SAH`; batas 2 MiB terkompresi tetap dijawab 413 sebelum isi dibaca.

**Alasan.** Batas 2 MiB hanya membatasi ukuran kiriman. XLSX adalah ZIP sehingga berkas kecil dapat mengembang jauh lebih besar di memori; batas entry dan ukuran sebelum kompresi harus diperiksa dari central directory sebelum parser membuka worksheet. Angka 360 berasal dari volume terbesar RFC-001 §8.1 dan dasar batas berkas ARCHITECTURE §7, sedangkan 4 KiB per record, 64 entry, dan 16 MiB memberi margin jauh di atas tiga kolom yang sah tanpa membiarkan struktur arsip tak terbatas.

**Alternatif yang ditolak.** *Mengandalkan 2 MiB saja.* Tidak membatasi zip bomb. *Memotong baris setelah batas.* Menghasilkan unggahan berhasil sebagian yang dilarang AC-26. *Menerima worksheet tambahan dan membaca yang pertama.* Menyembunyikan data yang dikira Administrator ikut diproses.

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 6 Agustus 2026 | **Versi 1.0 — dokumen dibuat.** Menetapkan tiga puluh endpoint di atas [SCHEMA.md](SCHEMA.md) v1.0 dan [ARCHITECTURE.md](ARCHITECTURE.md) v1.0. Membuka enam keputusan yang belum ditetapkan dokumen mana pun: sesi presensi ditulis saat dibuka (**CK-API-05**), unggah bersifat tolak seluruhnya (**CK-API-02**), pembuatan kelas atomik dengan pratinjau (**CK-API-03**), kata sandi awal berupa berkas CSV sekali unduh (**CK-API-04**), penggantian kata sandi mandiri tersedia (**CK-API-06**), dan baris rapor dibuat massal saat kelas dibuat (**CK-API-07**). Ditetapkan pula amplop respons, katalog lima belas kesalahan beserta tiga teks yang diwajibkan PRD, ketiadaan versi dan paginasi, serta daftar empat belas hal yang sengaja tidak memiliki endpoint. Menutup **T-01** lewat CK-API-08 dan **S-05** lewat §6.3. Diajukan lima temuan **A-01** sampai **A-05** |
| 6 Agustus 2026 | Sesi presensi berpindah dari ditulis-saat-dibuka menjadi ditulis oleh transaksi Simpan Presensi (**CK-API-11**, mengamandemen CK-API-05). Perpindahan ini menutup **A-01**: pertentangan yang dilaporkannya berasal dari CK-API-05, bukan dari dokumen sumber, dan tidak ada usulan perubahan bagi PRD. Ditambahkan `GET /api/penugasan/:id/siswa` |
| 6 Agustus 2026 | Berkas rapor dirender pada saat finalisasi dengan anggaran lunak 20 detik, dan ditambahkan `GET /api/kelas/:id/rapor/berkas` yang mengembalikan arsip ZIP sekelas (**CK-API-12**, mengamandemen CK-API-10 dan CK-09). Jalur render-saat-unduh tetap ada dan tidak berubah, karena CK-A-05 menuntutnya. Ditambahkan §13.3 yang mewajibkan pengukuran lama render sebelum keputusan ini dianggap terbukti |
| 6 Agustus 2026 | **A-02** ditetapkan: catatan wali bersifat per siswa karena melekat pada rapor siswa. **A-05** ditetapkan sebagai asumsi: tidak ada perpindahan siswa di tengah semester selama pilot. Jumlah endpoint dikoreksi dari tiga puluh menjadi **empat puluh tiga**, sesuai peta pada §4, dan daftar §11 menyusut menjadi tiga belas butir setelah unduh sekelas dipindahkan menjadi endpoint |
| 7 Agustus 2026 | Catatan zona waktu pada §2.4 disesuaikan mengikuti perpindahan region ke `ap-southeast-3` (**CK-16**). Ketentuannya tidak berubah: `Asia/Jakarta` tetap ditulis eksplisit dan tidak menyandar pada zona waktu server |
| 10 Agustus 2026 | **A-03 ditutup**: redaksi AC-26 diselaraskan dengan CK-API-02 sehingga unggahan CSV maupun Excel yang memuat baris bermasalah ditolak seluruhnya. Kontrak `GET /api/templat/pengguna.csv` diperjelas dengan parameter wajib `peran=guru\|siswa` (**CK-API-13**). Katalog kesalahan melengkapi `TIDAK_DITEMUKAN` yang sudah dipakai rute dan menambahkan `DATA_SUDAH_ADA` bagi benturan unik administrasi. Pratinjau kelas kini membawa `periode_ref` agar pemeriksaan I-08 memiliki periode sasaran (**CK-API-14**). Identitas kode komponen dibekukan setelah dipakai (**CK-API-15**), Guru tanpa mata pelajaran memperoleh penolakan eksplisit saat dipilih untuk kelas (**CK-API-16**), endpoint akun HTTP ditegaskan hanya mengelola Guru serta Siswa (**CK-API-17**), dan parser berkas memperoleh batas gagal-tertutup terhadap arsip terkompresi (**CK-API-18**) |
