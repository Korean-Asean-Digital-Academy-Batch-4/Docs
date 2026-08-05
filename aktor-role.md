# Aktor dan Peran — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v3.0 |
| **Tanggal** | 5 Agustus 2026 |
| **Kedudukan** | Turunan dari [PRD.md](PRD.md) v3.0. Menggantikan seluruh versi dokumen ini sebelumnya |
| **Cakupan** | Menjawab pertanyaan "siapa boleh melihat dan mengubah apa" pada MVP |

> Dokumen ini merupakan penjabaran dari PRD §5, §6, §8, dan §9. Apabila terdapat perbedaan, **PRD yang berlaku**.
> Seluruh isi disusun dari sisi aplikasi. Rancangan basis data, kontrak antarmuka program, dan mekanisme autentikasi tidak dibahas di sini.

---

## 1. Ringkasan

Sistem memiliki **tiga jenis akun**: Administrator, Guru, dan Siswa.

**Wali Kelas bukan jenis akun keempat.** Wali Kelas adalah **kewenangan tambahan** yang diberikan Administrator kepada salah satu Guru pada saat kelas dibuat. Seluruh Guru merupakan Guru Mata Pelajaran; sebagian di antaranya juga memegang kewenangan Wali Kelas.

| Aktor | Jenis | Cakupan kerja | Terhadap nilai akademik |
|---|---|---|---|
| **Administrator** | Jenis akun | Seluruh sekolah | Akses penuh, termasuk mengisi dan mengubah nilai |
| **Guru Mata Pelajaran** | Jenis akun | Kelas yang ditugaskan kepadanya, pada satu mata pelajaran | Mengisi dan menyunting, terbatas pada penugasannya |
| **Wali Kelas** | Kewenangan tambahan pada akun Guru | Satu kelas asuhan, seluruh mata pelajaran | Hanya membaca; tidak dapat mengubah nilai Guru lain |
| **Siswa** | Jenis akun | Dirinya sendiri | Hanya membaca |

---

## 2. Model mental: dua arah pemotongan data

Guru Mata Pelajaran dan Wali Kelas memotong data yang sama pada **arah yang berbeda**.

```
                    X IPA 1    X IPA 2    X IPA 3
                  ┌──────────┬──────────┬──────────┐
     Biologi X    │  ██████  │  ██████  │  ██████  │  ← GURU MAPEL: vertikal
                  ├──────────┼──────────┼──────────┤     1 mata pelajaran × N kelas
     Matematika X │  ██████  │          │          │        (seluruhnya satu jenjang)
                  ├──────────┤          │          │
     Fisika X     │  ██████  │          │          │
                  ├──────────┤          │          │
     B. Indo. X   │  ██████  │          │          │
                  └──────────┴──────────┴──────────┘
                       ▲
                       │  WALI KELAS: horizontal
                       │  1 kelas × seluruh mata pelajaran
```

Apabila kedua fungsi dipegang orang yang sama, kedua potongan tersebut **bersilangan pada satu kotak**, dan hanya pada kotak itulah yang bersangkutan memiliki kewenangan menulis sekaligus kewenangan memantau.

**Asimetri yang perlu diingat:**

> **Guru Mata Pelajaran: cakupan sempit, tetapi berwenang menulis.**
> **Wali Kelas: cakupan luas, tetapi hanya membaca** — dengan satu pengecualian, yaitu finalisasi dan distribusi rapor semester.

Wali Kelas tidak pernah mengetik nilai. Wali Kelas memeriksa kelengkapan nilai yang telah diisi Guru, menuliskan catatan umum, lalu bertanggung jawab atas penerbitan rapor semester.

---

## 3. Struktur penugasan Guru

Penugasan disusun dalam tiga lapis ([PRD §8.2](PRD.md)). Setiap lapis menjadi dasar bagi lapis berikutnya.

| Lapis | Yang dihubungkan | Contoh |
|:--:|---|---|
| 1 | Mata pelajaran dibuat **per jenjang**, beserta KKM | Biologi X, Matematika XI |
| 2 | Mata pelajaran-jenjang dihubungkan dengan **satu Guru** | Pak Cahyo — Biologi X |
| 3 | **Kelas** dihubungkan dengan Guru pada saat kelas dibuat | Kelas X IPA 3 — Pak Cahyo |

**Ketentuan yang membentuk lingkup kewenangan Guru:**

1. Satu mata pelajaran memiliki tepat satu jenjang dan tepat satu Guru pengampu.
2. Satu Guru mengampu **tepat satu mata pelajaran pada tepat satu jenjang**.
3. Satu Guru boleh mengajar lebih dari satu kelas, sepanjang seluruh kelas berada pada jenjang yang sama.
4. Karena Guru hanya mengampu satu mata pelajaran, **mata pelajaran tidak pernah perlu dipilih**. Kelas sudah cukup untuk menentukan lingkup pengisian nilai maupun sesi presensi.
5. Sistem menolak penghubungan apabila jenjang kelas berbeda dengan jenjang mata pelajaran Guru.
6. Guru hanya melihat dan mengisi data pada kelas yang dihubungkan kepadanya, sehingga tidak mungkin memasukkan nilai siswa yang tidak diajarnya.

**Guru tanpa penugasan.** Akun Guru yang baru dibuat belum memiliki peran apa pun sampai dihubungkan dengan mata pelajaran ([PRD §6.1.1](PRD.md)). Sebelum penghubungan tersebut, akun dapat masuk aplikasi tetapi tidak memiliki kelas, nilai, maupun sesi presensi yang dapat dikelola.

**Penetapan Wali Kelas.** Wali Kelas ditetapkan Administrator dari salah satu Guru yang telah dihubungkan dengan kelas tersebut ([PRD §6.1.5](PRD.md)). Dengan demikian, Wali Kelas selalu merupakan Guru yang juga mengajar di kelas asuhannya.

---

## 4. Contoh konkret

**Pak Cahyo** — pengampu Biologi X, sekaligus Wali Kelas X IPA 1.

| Di kelas | Sebagai Guru Mata Pelajaran | Sebagai Wali Kelas |
|---|---|---|
| **X IPA 1** | Mengisi nilai dan presensi **Biologi X** | Membaca seluruh mata pelajaran · memeriksa kelengkapan · menulis catatan · finalisasi · distribusi · unduh rapor |
| **X IPA 2** | Mengisi nilai dan presensi **Biologi X** | — |
| **X IPA 3** | Mengisi nilai dan presensi **Biologi X** | — |

**Yang tidak dapat dilakukan Pak Cahyo:**

- Pada X IPA 1, Pak Cahyo **melihat** nilai Matematika X karena ia Wali Kelas, tetapi **tidak dapat mengubahnya** — nilai tersebut wilayah Guru pengampu Matematika X.
- Pada X IPA 2, Pak Cahyo **tidak dapat melihat** nilai Matematika X sama sekali, karena pada kelas itu ia hanya Guru Mata Pelajaran.
- Pak Cahyo **tidak dapat memfinalisasi rapor** X IPA 2 dan X IPA 3, meskipun ia mengajar di sana.
- Pak Cahyo **tidak dapat mengampu mata pelajaran lain** maupun jenjang lain.

---

## 5. Kewenangan per aktor

### 5.1 Administrator

Administrator menyiapkan data dan aturan sekolah, serta memiliki akses penuh terhadap seluruh data termasuk nilai akademik.

1. Membuat akun Guru dan Siswa melalui unggah berkas CSV atau pengisian manual, dengan kata sandi awal yang dibuat sistem.
2. Menyerahkan kata sandi awal kepada pengguna dan melakukan penggantian kata sandi apabila diperlukan. **Tidak tersedia pemulihan kata sandi secara mandiri.**
3. Membuat mata pelajaran per jenjang beserta KKM, dan menghubungkannya dengan Guru pengampu.
4. Membuat kelas beserta periode akademiknya, mengunggah daftar siswa, menghubungkan mata pelajaran dan Guru pengampunya, serta menetapkan Wali Kelas.
5. Mengisi dan mengubah nilai akademik apabila diperlukan, termasuk pada data yang sudah difinalisasi.
6. Membuka, menyunting, dan menghapus seluruh sesi presensi beserta isinya.

### 5.2 Guru Mata Pelajaran

Guru hanya dapat bekerja pada kelas yang ditugaskan Administrator kepadanya.

1. Mengisi dan menyunting nilai, lalu menyimpannya melalui tombol **Simpan Nilai**.
2. Membuka sesi presensi, menetapkan dan menyunting status kehadiran, serta menghapus sesi yang keliru.
3. Melihat progres siswa pada penugasannya.
4. **Tidak melakukan finalisasi rapor.**
5. **Tidak memiliki jalur pengunduhan rapor.**

### 5.3 Wali Kelas

Wali Kelas merupakan kewenangan tambahan pada akun Guru.

1. Melihat nilai seluruh mata pelajaran milik siswa di kelas walinya.
2. Melihat ringkasan presensi seluruh siswa pada kelas walinya.
3. Memeriksa apakah seluruh Guru telah menginput semua nilai.
4. Menulis atau menyunting catatan umum rapor semester.
5. Memfinalisasi rapor semester setelah seluruh mata pelajaran lengkap.
6. Mendistribusikan rapor kepada siswa.
7. Mengunduh rapor kelas walinya setelah difinalisasi.
8. **Tidak dapat mengubah nilai yang dibuat oleh Guru Mata Pelajaran lain.**
9. **Tidak dapat membuka, mengubah, maupun menghapus sesi presensi Guru lain.**

### 5.4 Siswa

1. Melihat nilai dan bobot penilaian miliknya segera setelah Guru menginputnya.
2. Melihat persentase presensi miliknya sendiri per mata pelajaran.
3. Memperoleh rangkuman capaian dan rekomendasi belajar melalui tombol **Suggestion**.
4. Melihat dan mengunduh rapor setelah Wali Kelas mendistribusikannya.
5. **Tidak dapat menulis data apa pun.**

---

## 6. Matriks kewenangan

Sumber: [PRD §8.1](PRD.md), dilengkapi ketentuan presensi pada [PRD §8.4](PRD.md).

| Kemampuan | Administrator | Guru Mapel | Wali Kelas | Siswa |
|---|:--:|:--:|:--:|:--:|
| Membuat tahun ajaran, semester, kelas, mata pelajaran | ✅ | ❌ | ❌ | ❌ |
| Membuat dan mengelola akun Guru dan Siswa | ✅ | ❌ | ❌ | ❌ |
| Mengganti kata sandi pengguna | ✅ | ❌ | ❌ | ❌ |
| Menempatkan siswa ke kelas | ✅ | ❌ | ❌ | ❌ |
| Menghubungkan mata pelajaran-jenjang dengan Guru, lalu dengan kelas | ✅ | ❌ | ❌ | ❌ |
| Menetapkan Wali Kelas | ✅ | ❌ | ❌ | ❌ |
| Menetapkan KKM per mata pelajaran-jenjang | ✅ | ❌ | ❌ | ❌ |
| **Mengisi dan mengubah nilai akademik** | ✅ penuh | ✅ penugasannya | ❌ | ❌ |
| **Membuka dan menghapus sesi presensi** | ✅ | ✅ penugasannya | ❌ | ❌ |
| Mengisi dan menyunting presensi | ✅ | ✅ penugasannya | ❌ | ❌ |
| Melihat ringkasan presensi lintas mata pelajaran | ✅ | ❌ | ✅ kelas walinya | ✅ dirinya |
| Melihat nilai lintas mata pelajaran | ✅ | ❌ | ✅ kelas walinya | ✅ dirinya |
| Menulis catatan umum rapor semester | ✅ | ❌ | ✅ | ❌ |
| **Memfinalisasi rapor semester** | ✅ | ❌ | ✅ | ❌ |
| Mendistribusikan rapor | ✅ | ❌ | ✅ | ❌ |
| Mengunduh rapor | ✅ | ❌ | ✅ kelas walinya | ✅ setelah distribusi |
| Menggunakan tombol Suggestion | ❌ | ❌ | ❌ | ✅ dirinya |

Tanda ❌ berarti kemampuan tersebut tidak tersedia bagi aktor yang bersangkutan, termasuk dalam bentuk baca saja apabila kemampuannya bersifat membaca.

---

## 7. Lingkup presensi

Presensi dicatat melalui **sesi**, yaitu satu pertemuan pada satu kelas, satu mata pelajaran, dan satu tanggal ([PRD §8.4](PRD.md)).

| Pelaku | Yang dapat dilakukan | Batas kewenangan |
|---|---|---|
| **Administrator** | Membuka, menyunting, dan menghapus seluruh sesi beserta presensinya | — |
| **Guru Mapel** | Membuka sesi, menetapkan status Hadir/Izin/Sakit/Alpa, menekan **Hadir Semua**, menambah catatan, menyunting, dan menghapus sesi | Hanya untuk kelas yang diajarnya |
| **Wali Kelas** | Melihat ringkasan presensi seluruh siswa pada kelas walinya | Tidak membuka, mengubah, maupun menghapus sesi Guru lain |
| **Siswa** | Melihat persentase presensi miliknya sendiri per mata pelajaran | Tidak dapat membuka rincian presensi per tanggal, maupun presensi siswa lain |

**Ketentuan yang memengaruhi kewenangan:**

1. Sesi melekat pada Guru yang membukanya. Satu Guru, satu kelas, dan satu tanggal hanya boleh memiliki **satu sesi**; dua Guru mata pelajaran berbeda dapat membuka sesi pada kelas dan tanggal yang sama tanpa bertabrakan.
2. Sesi tidak perlu ditutup dan tidak dapat dikunci. Sesi yang keliru dihapus oleh Guru yang membukanya.
3. Koreksi dan penghapusan sesi berlaku langsung, **tanpa pencatatan riwayat perubahan**.
4. Sistem hanya menampilkan persentase kehadiran, tanpa peringatan, anjuran, maupun tindak lanjut.

---

## 8. Peran pada daur hidup rapor semester

Status berikut berlaku untuk rapor semester. Nilai dan presensi tidak memiliki status: keduanya tersimpan ketika Guru menekan tombol simpan, dan **nilai langsung terlihat siswa** sejak saat itu.

```
        Wali Kelas                 Wali Kelas
[Draft] ──finalisasi──> [Finalized] ──distribusi──> [Distributed]
```

| Status | Siapa yang dapat melihat | Dapat diubah? |
|---|---|---|
| **Draft** | Administrator, Guru pada penugasannya, Wali Kelas pada kelas walinya | ✅ Guru masih dapat mengubah nilai dan presensi |
| **Finalized** | Administrator, Wali Kelas | ❌ Terkunci bagi Guru maupun Wali Kelas. Hanya Administrator yang dapat mengubah datanya |
| **Distributed** | Ditambah Siswa yang bersangkutan | ❌ Sama seperti Finalized |

**Ketentuan finalisasi:**

1. Finalisasi dilakukan **satu kali**, oleh Wali Kelas, setelah seluruh mata pelajaran pada kelas asuhannya lengkap.
2. Apabila masih terdapat mata pelajaran yang belum lengkap, sistem menolak finalisasi dan menampilkan pesan untuk setiap mata pelajaran tersebut: *"Data Mapel &lt;nama mata pelajaran&gt; belum ada, tolong hubungi guru yang bertanggung jawab."*
3. Di luar momen finalisasi, sistem **tidak** menampilkan pemberitahuan kelengkapan data dalam bentuk apa pun.
4. **Tidak tersedia mekanisme buka kembali.** Tidak ada tahap peninjauan maupun persetujuan oleh pihak lain — Wali Kelas adalah pihak terakhir dalam alur ini.
5. Rapor dapat diunduh Wali Kelas setelah difinalisasi, dan oleh Siswa setelah didistribusikan.

---

## 9. Batas kewenangan AI

AI Insight dijalankan melalui tombol **Suggestion** dan **hanya tersedia bagi Siswa** ([PRD §8.5](PRD.md)).

| Aspek | Ketentuan |
|---|---|
| Pemicu | Ditekan siswa. Tidak berjalan otomatis |
| Data yang dibaca | Nilai seluruh mata pelajaran beserta topiknya pada semester berjalan, KKM, kelengkapan penilaian, dan persentase kehadiran per mata pelajaran — **hanya milik siswa yang menekan tombol** |
| Keluaran | Satu paragraf rekomendasi diikuti poin-poin ringkas, memuat rekomendasi belajar, alasan rekomendasi, dan dua pilihan tindakan yang realistis |
| Penyimpanan | Tidak disimpan. Tidak tersedia riwayat rekomendasi |

**Larangan bagi AI** ([PRD §8.6](PRD.md)):

1. Tidak menghitung atau menentukan nilai resmi.
2. Tidak mengubah KKM, bobot, nilai, atau presensi.
3. Tidak memfinalisasi atau mendistribusikan rapor.
4. Tidak membuat prediksi kelulusan, diagnosis psikologis, atau keputusan sanksi.
5. Tidak menampilkan rata-rata final apabila data belum lengkap.
6. Tidak membaca data siswa selain siswa yang menekan tombol.
7. Kegagalan layanan AI tidak menghambat input nilai, presensi, finalisasi, maupun distribusi rapor.

---

## 10. Umpan balik proses

Berlaku bagi seluruh aktor: **setiap interaksi yang memasukkan atau mengubah data wajib menampilkan pemberitahuan berhasil atau gagal** ([PRD §6.1.6](PRD.md)). Termasuk di dalamnya unggah berkas, pembuatan akun, penetapan peran, pembuatan kelas, penyimpanan nilai, penyimpanan presensi, penghapusan sesi, finalisasi, dan distribusi rapor. Pemberitahuan kegagalan menyebutkan alasannya.

---

## 11. Kesalahpahaman yang sering terjadi

| Anggapan keliru | Ketentuan yang benar |
|---|---|
| Wali Kelas adalah jenis akun tersendiri | Wali Kelas adalah kewenangan tambahan pada akun Guru |
| Guru Mata Pelajaran memfinalisasi rapornya sendiri | Finalisasi dilakukan Wali Kelas, satu kali untuk seluruh mata pelajaran |
| Administrator meninjau dan menyetujui rapor | Tidak ada tahap peninjauan. Wali Kelas memfinalisasi dan langsung mendistribusikan |
| Wali Kelas dapat memperbaiki nilai mata pelajaran lain menjelang rapor terbit | Tidak dapat. Perbaikan dilakukan Guru pengampu, atau Administrator |
| Guru memilih mata pelajaran saat mengisi nilai atau membuka sesi | Tidak perlu dipilih. Satu Guru mengampu tepat satu mata pelajaran |
| Guru dapat mengunduh rapor kelas yang diajarnya | Pengunduhan hanya tersedia bagi Administrator, Wali Kelas, dan Siswa |
| Menu Wali Kelas tampil untuk seluruh Guru | Menu finalisasi dan distribusi hanya tampil bagi Guru yang memegang kewenangan Wali Kelas |
| Nilai baru terlihat siswa setelah rapor terbit | Nilai langsung terlihat siswa begitu Guru menekan Simpan Nilai |

---

## 12. Pertanyaan terbuka

Belum ditetapkan PRD v3.0 dan perlu dikonfirmasi sebelum implementasi kewenangan dikunci.

| # | Pertanyaan | Dampak apabila belum dijawab |
|---|---|---|
| 1 | Dapatkah satu Guru menjadi Wali Kelas pada lebih dari satu kelas dalam satu periode? | Saat ini diasumsikan paling banyak satu kelas per periode |
| 2 | Apabila Administrator mengubah nilai setelah rapor berstatus Distributed, apakah berkas rapor yang telah diterima siswa ikut diperbarui? | Berpotensi menimbulkan selisih antara nilai pada sistem dan berkas rapor yang sudah diunduh |
| 3 | Apakah tindakan Administrator terhadap nilai perlu dibedakan dari tindakan Guru bagi pihak sekolah? | MVP tidak mencatat riwayat perubahan nilai maupun presensi |
| 4 | Apakah Wali Kelas perlu melihat rincian presensi per tanggal, atau cukup ringkasan persentase? | PRD hanya menyebut ringkasan; rincian per tanggal tidak tersedia |

---

## 13. Dokumen terkait

| Dokumen | Isi |
|---|---|
| [PRD.md](PRD.md) | Sumber kebenaran untuk seluruh ketentuan pada dokumen ini |
| [ATURAN-DAN-KRITERIA.md](ATURAN-DAN-KRITERIA.md) | Aturan produk, use case, layar minimum, kriteria kesiapan, dan butir validasi sekolah |
