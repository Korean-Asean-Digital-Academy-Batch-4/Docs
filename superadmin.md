# Superadmin — Peran & Kekurangan

> **Status:** 2 Agustus 2026
> **Sumber:** prototype Figma *Superadmin Edutrack* (9 layar) + PRD + Project Charter
> **Scope file ini:** memahami peran Superadmin. Implikasi ke skema/API **belum** dituangkan — lihat [Kekurangan](#kekurangan--pertanyaan-terbuka) dulu.
> Aturan izin lintas peran ada di [`aktor-role.md`](./aktor-role.md).

---

## 1. Ringkasan

Superadmin adalah **penyiap sistem**, bukan pengguna harian. Pekerjaannya menumpuk di awal semester: buat akun, impor data, tetapkan wali kelas, kunci rumus penilaian. Setelah semester berjalan, perannya berubah jadi pemantau.

---

## 2. Peta menu

```
Dashboard                    → Log Aktivitas Sistem
Penugasan
  ├── Daftar Nama Guru       → direktori guru + import Excel
  ├── Kelas X
  ├── Kelas XI               → wizard penugasan 4 langkah
  └── Kelas XII
Rumus Nilai
  ├── Rumus Utama            → komponen + bobot, wajib total 100%
  └── Rumus Lainnya          → rumus alternatif (Keterampilan/Praktikum)
Database
  ├── DB Presensi Siswa
  ├── DB Mata Pelajaran
  ├── DB Nilai
  └── DB Rapor
Pembuatan Akun
  ├── Akun Guru
  └── Akun Siswa
Pengaturan Sistem
```

---

## 3. Wewenang

| # | Wewenang | Layar |
|---|---|---|
| 1 | Pantau log aktivitas: aktor, aksi, waktu, IP, status | Dashboard |
| 2 | Buat akun **Guru** (email + NIP) | Pembuatan Akun › Akun Guru |
| 3 | Buat akun **Siswa** (NIS, tanpa email) | Pembuatan Akun › Akun Siswa |
| 4 | Import massal guru via Excel/CSV + unduh template | Daftar Nama Guru |
| 5 | Import massal siswa + pembagian kelas via Excel | Wizard langkah 2 |
| 6 | Lihat direktori guru per periode (Role 1 & Role 2) | Daftar Nama Guru |
| 7 | **Tetapkan Wali Kelas** per rombongan belajar | Wizard langkah 3 |
| 8 | Susun komponen & bobot **Rumus Utama** (tambah, hapus, urutkan, validasi 100%) | Rumus Utama |
| 9 | Buat **rumus alternatif** di luar rumus utama | Rumus Lainnya |
| 10 | **Sinkronkan rumus** = setujui & kunci sebelum semester dimulai | Rumus Utama |
| 11 | Akses data mentah: presensi, mapel, nilai, rapor | Database |
| 12 | Pengaturan sistem | Pengaturan Sistem |

---

## 4. Alur kunci: Wizard Penugasan (4 langkah)

Dipicu dari `Penugasan › Kelas X/XI/XII`, tombol **"Mulai Penugasan Baru"**.

```
① Set Periode        → pilih Semester + Tahun Ajaran
② Siswa & Kelas      → unggah Excel berisi daftar siswa + pembagian kelasnya
③ Wali Kelas         → tetapkan 1 guru per rombongan belajar
④ Selesai
```

**Langkah ③ — teks aslinya:**
> *"Data rombongan belajar berhasil diekstrak. Silakan tetapkan tenaga pendidik yang akan bertugas sebagai wali kelas untuk masing-masing rombongan belajar."*

| Rombongan Belajar | Tenaga Pendidik (Wali Kelas) |
|---|---|
| X-A · Kelas X-MIPA-1 · 36 Siswa | Budi Santoso, M.Pd. |
| X-B · Kelas X-MIPA-2 · 35 Siswa | *(pencarian: ketik nama guru → Siti Rahmawati, S.Si. NIP 198507232010012004)* |
| X-C · Kelas X-IPS-1 · 34 Siswa | *(kosong)* |

**Implikasi penting:** kelas (rombel) **lahir dari isi file Excel**, bukan dibuat manual. Parser Excel adalah satu-satunya jalur masuk data kelas & roster — tidak ada jalur cadangan.

Istilah resmi yang dipakai: **rombongan belajar (rombel)**.

### Hasil setelah wizard selesai

Halaman `Penugasan Guru & Siswa Kelas X`:
- Filter: Semester · Tahun Ajaran · Kelas → **Tampilkan**
- Kartu **WALI KELAS**: nama + NIP
- Tabel **Daftar Siswa**: `NO · NIS · NAMA SISWA · JENIS KELAMIN (L/P) · AKSI`

---

## 5. Direktori Guru — model dua topi terkonfirmasi

Filter: `Semester Ganjil | 2023/2024` · `Total 4 Guru Terdaftar` · cari nama/email.

| Nama Guru | Email | **Role 1** | **Role 2** |
|---|---|---|---|
| Budi Santoso, S.Pd | budi.santoso@sekolah.sch.id | Wali Kelas | Guru Mapel Matematika |
| Siti Aminah, M.Pd | siti.aminah@sekolah.sch.id | Wali Kelas | Guru Mapel IPA |
| Ahmad Fauzi, S.T | ahmad.fauzi@sekolah.sch.id | **–** | Guru Mapel Fisika |
| Ratna Sari, S.Pd | ratna.sari@sekolah.sch.id | Wali Kelas | Guru Mapel Matematika |

✅ **Memvalidasi [`aktor-role.md`](./aktor-role.md) tanpa perubahan.** Ahmad Fauzi dapat `–` di Role 1 karena dia guru mapel murni. Wali Kelas memang bukan role login, melainkan penugasan yang bisa kosong.

Direktori guru **di-scope per periode** — konsisten dengan `penugasan` yang juga per periode.

---

## 6. Rumus Nilai — ternyata jamak

### Rumus Utama

`Standard Curriculum (2023)` · `• AKTIF` · *Diterapkan ke: Kelas X, Kelas XI, Kelas XII*

| Kode | Komponen | Bobot |
|---|---|---|
| T1 | Tugas 1 (Penugasan) | 10% |
| T2 | Tugas 2 (Proyek) | 15% |
| UH | Ulangan Harian (Kuis) | 15% |
| UTS | Ujian Tengah Semester | 25% |
| UAS | Ujian Akhir Semester | 35% |
| | **Total** | **100% ✓** |

- Baris bisa **di-drag untuk diurutkan**
- Tombol **+ Tambahkan Komponen**
- Panel kanan: donut **Distribusi Bobot Valid**, validasi total 100%

### Rumus Lainnya

> *"Atur bobot perhitungan nilai untuk jenis evaluasi alternatif di luar rumus utama. Pastikan total bobot setiap rumus mencapai 100%."*

Contoh yang ada: **Rumus Penilaian Keterampilan (Praktikum)** — punya komponen & bobot sendiri. Ada tombol **+ Tambah Rumus**.

### Artinya: model Kurikulum 2013 utuh

Digabung dengan kolom Sikap Spiritual/Sosial di layar guru, polanya lengkap:

```
Sikap          → Spiritual + Sosial   (kualitatif A/B/C + keterangan teks)
Pengetahuan    → Rumus Utama          (T1, T2, UH, UTS, UAS)
Keterampilan   → Rumus Praktikum      (komponen & bobot sendiri)
```

**Rumus adalah entitas tersendiri**, bukan atribut yang melekat ke penugasan:

```
rumus (id, nama, jenis, status, diterapkan_ke[])
  └── komponen (kode, nama, bobot, urutan)

penugasan ──> rumus_id
```

Satu mapel bisa dinilai lebih dari satu rumus, dan nilai akhirnya terpisah.

### "Sinkronkan ke Database" = persetujuan pra-semester

> *"Kirim perubahan rumus terbaru untuk menghitung ulang nilai siswa di DB Nilai."* → **[Sinkronkan Sekarang]**

**Ini bukan hitung-ulang di tengah semester.** Ini tindakan **menyetujui & mengunci rumus sebelum semester dimulai**:

```
Superadmin susun rumus  →  [Sinkronkan]  →  rumus terkunci untuk semester itu
                                          →  guru mulai input nilai
```

Jadi `rumus` punya lifecycle sendiri, mirip rapor. Badge `• AKTIF` menandai rumus yang sudah disinkronkan dan berlaku.

---

## 7. Log Aktivitas Sistem

> *"Pantau semua perubahan dan aktivitas dalam sistem EduTrack."*

Contoh entri nyata dari prototype:

| Aktivitas | Detail | Status |
|---|---|---|
| Pembuatan Akun Guru Pak Budi | Admin sistem membuat akun guru baru untuk 'Budi Santoso' dengan NIP 198203152006041002 | Success |
| Upload Data Kelas X-MIPA-1 | Import data massal sebanyak 36 siswa dari file CSV berhasil diselesaikan | Success |
| Update Rumus Nilai Utama | Parameter bobot diubah dari (UTS 30% / UAS 70%) menjadi (UTS 40% / UAS 60%) | **Warning** |
| Gagal Sinkronisasi Database Presensi | Koneksi ke server presensi eksternal terputus. Sistem akan mencoba kembali dalam 15 menit | **Failed** |
| Login Superadmin | Superadmin berhasil login dari IP Address 192.168.1.45 (Jakarta) | — |

Ada paginasi **"Muat Lebih Banyak..."**.

✅ Ini **`audit_log` yang sudah dirancang**, dan lebih kaya dari rancangan awal. Tambahan yang perlu masuk skema: `ip_address`, `severity` (success/warning/failed), `judul`, `deskripsi`.

---

## Kekurangan & Pertanyaan Terbuka

### A. Layar yang belum ada

| # | Hilang | Kenapa penting |
|---|---|---|
| **A1** | **Penugasan Guru Mapel ke Kelas** | 🔴 **Paling besar.** Wizard hanya menugaskan siswa→kelas dan wali kelas. Tidak ada langkah yang menetapkan *"Bu Rina mengajar Matematika di X-MIPA-1"* — padahal itu tabel `penugasan`, inti seluruh otorisasi guru. Direktori guru sudah menampilkan "Guru Mapel Matematika", tapi **tidak ada layar tempat itu diisi** |
| **A2** | **Persetujuan rapor** (`review` → `distributed`) | Lifecycle 3 tahap yang tim putuskan tidak punya rumah. Yang ada cuma "DB Rapor" (lihat data), bukan alur persetujuan |
| **A3** | **Reset password siswa** | Siswa login pakai NIS tanpa email → tidak bisa reset sendiri. Di sekolah nyata ini kejadian mingguan. Wajib ada, belum digambar |
| **A4** | **Edit / hapus akun** guru & siswa | Menunya bernama "Pembuatan Akun" — hanya buat. Update & delete tidak terlihat |
| **A5** | **CRUD Mata Pelajaran** | Hanya ada "DB Mata Pelajaran" (lihat). Bagaimana menambah mapel baru? |
| **A6** | **CRUD Semester & Tahun Ajaran** | Muncul di dropdown hampir semua layar, tapi tidak ada layar pembuatannya. Mungkin di Pengaturan Sistem — belum dilihat |
| **A7** | **Isi Pengaturan Sistem** | Belum pernah terlihat sama sekali |

### B. Aturan yang belum ditentukan

| # | Pertanyaan | Kalau salah tebak |
|---|---|---|
| **B1** | Apa yang terjadi kalau **Sinkronkan ditekan di tengah semester**? | Nilai yang sudah masuk bisa berubah tanpa jejak. Usulan: rumus terkunci setelah semester berjalan; perubahan wajib versi baru + tercatat di audit |
| **B2** | Satu guru = **satu mapel** saja, atau bisa banyak? | Role 2 hanya menampilkan satu. Menentukan bentuk tabel `penugasan` |
| **B3** | **Rumus mana berlaku untuk mapel mana?** Apakah semua mapel punya Pengetahuan + Keterampilan, atau hanya sebagian? | Menentukan relasi `penugasan ↔ rumus` — satu-ke-satu atau satu-ke-banyak |
| **B4** | **Format kolom template Excel** guru & siswa | Parser adalah satu-satunya jalur masuk data. Tanpa spesifikasi kolom, tidak bisa dibangun |
| **B5** | Excel diunggah **dua kali** — duplikat, timpa, atau tolak? | Risiko roster ganda |
| **B6** | Kelas lahir dari Excel — **bagaimana memperbaiki kalau salah**? | Tidak ada jalur koreksi manual |
| **B7** | Boleh ada **lebih dari satu superadmin**? | Menentukan aturan pendaftaran |
| **B8** | Ambang batas badge kehadiran (`85%` → "Perlu Ditingkatkan") | Aturan bisnis, belum tertulis di mana pun |

### C. Kontradiksi dengan dokumen

| # | Kontradiksi |
|---|---|
| **C1** | **FR-02** menyatakan superadmin "membuat dan mengelola kelas". Nyatanya kelas **diekstrak dari Excel** |
| **C2** | PRD menaruh **integrasi pihak ketiga sebagai out-of-scope**, tapi log menyebut *"server presensi eksternal"*. Kalau ini nyata (mis. mesin fingerprint), itu integrasi yang belum pernah dibahas |
| **C3** | Charter mencoret **notifikasi**, tapi ikon lonceng ada di header guru & siswa |
| **C4** | Rapor 3 tahap (keputusan tim 2 Agu) tidak punya UI di superadmin — lihat A2 |
| **C5** | Bobot di layar guru (`T1 10 · T2 10 · UH1 15 · UH2 15 · UTS 20 · UAS 30`) **berbeda** dari Rumus Utama superadmin (`T1 10 · T2 15 · UH 15 · UTS 25 · UAS 35`). Layar guru punya UH1 **dan** UH2; superadmin cuma satu UH. Apakah guru boleh menambah komponen sendiri? |

### D. Keamanan

| # | Isu | Tingkat |
|---|---|---|
| **D1** | **Superadmin mendaftar sendiri** tanpa pembatas terlihat. Siapa pun yang menemukan URL bisa membuat akun superadmin ke seluruh data nilai sekolah | 🔴 Tinggi |
| **D2** | Dropdown **"Login Sebagai"** di layar login. Role tidak boleh dipilih pengguna — harus dari akun. Kalau backend mempercayainya, akun siswa bisa masuk sebagai admin | 🔴 Tinggi |
| **D3** | **Pertanyaan keamanan** untuk pemulihan password. Cognito **tidak punya fitur ini** — harus dibangun sendiri di Postgres (jawaban wajib di-hash) + alur `AdminSetUserPassword`. ~1 hari kerja, padahal email sekolah sudah wajib diisi di formulir yang sama | 🟡 Sedang |
| **D4** | Siswa tanpa email → pemulihan hanya bisa lewat superadmin (lihat A3) | 🟡 Sedang |

---

## Yang sudah cocok dengan desain — tidak perlu diubah

| Hal | Bukti |
|---|---|
| Model dua topi Guru (mapel vs wali kelas) | Kolom Role 1 / Role 2, dengan `–` untuk yang bukan wali |
| Wali kelas = satu guru per kelas | Wizard langkah 3 |
| `audit_log` | Log Aktivitas Sistem (bahkan lebih kaya) |
| Komponen nilai sebagai data, bukan enum | Rumus Utama bisa tambah/hapus/urutkan komponen |
| Validasi Σ bobot = 100% | Ditegakkan di UI, ada indikator donut |
| Periode akademik sebagai sumbu utama | Semua layar difilter Semester + Tahun Ajaran |
| NIS sebagai identitas siswa | Tabel Daftar Siswa |

---

## Berkas terkait

- [`aktor-role.md`](./aktor-role.md) — peran & matriks izin lintas aktor
- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — stack, batas modul, alur data
- `SCHEMA.md` *(belum ditulis — tunggu B1–B6 terjawab)*
