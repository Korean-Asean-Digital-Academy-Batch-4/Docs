# Aturan, Use Case, Layar, dan Kriteria — EduTrack

| Keterangan | Isi |
|---|---|
| **Kedudukan** | Lampiran operasional dari [PRD.md](PRD.md). Memuat rincian yang sebelumnya berada pada §10 sampai §14 PRD |
| **Versi** | v1.0 |
| **Tanggal** | 5 Agustus 2026 |
| **Disusun oleh** | Re:Code |

> Dokumen ini menurunkan ketetapan PRD menjadi aturan, use case, daftar layar, dan kriteria kesiapan yang dapat diuji.
> Apabila terdapat perbedaan, **PRD.md yang berlaku**.

---

## 1. Aturan Produk

| # | Aturan |
|---|---|
| P1 | Seluruh Guru merupakan Guru Mata Pelajaran. Kewenangan Wali Kelas hanya tambahan pada Guru tertentu |
| P2 | Mata pelajaran dibuat per jenjang dengan tepat satu Guru pengampu. Satu Guru mengampu tepat satu mata pelajaran pada tepat satu jenjang, tetapi boleh mengajar banyak kelas pada jenjang tersebut. Sistem menolak penghubungan kelas apabila jenjang kelas tidak sama dengan jenjang mata pelajaran Guru |
| P3 | Satu kelas hanya memiliki satu Wali Kelas aktif dalam satu semester |
| P4 | Administrator menetapkan KKM per mata pelajaran-jenjang pada saat mata pelajaran dibuat. Komponen penilaian dan bobot mengikuti templat bawaan sistem; jumlah seluruh bobot harus tepat 100% |
| P5 | Nilai kosong tidak boleh dianggap sebagai nilai nol; sistem menandainya sebagai belum lengkap |
| P6 | Presensi hanya dicatat melalui sesi. Satu Guru, kelas, dan tanggal hanya memiliki satu sesi, dan setiap siswa memiliki tepat satu status pada sesi tersebut dengan status awal Alpa |
| P7 | Koreksi presensi dan penghapusan sesi hanya dapat dilakukan pada lingkup penugasan Guru, dan berlaku langsung tanpa pencatatan riwayat |
| P8 | Guru Mata Pelajaran hanya dapat melihat dan mengubah nilai pada penugasannya sendiri |
| P9 | **Finalisasi rapor semester hanya dilakukan Wali Kelas.** Guru Mata Pelajaran tidak melakukan finalisasi |
| P10 | Wali Kelas tidak dapat mengubah nilai Guru lain; Wali Kelas hanya dapat meminta koreksi |
| P11 | Rapor semester tidak dapat difinalisasi sebelum seluruh mata pelajaran pada kelas tersebut lengkap |
| P12 | Siswa hanya dapat melihat datanya sendiri dan hanya dapat membuka rapor yang sudah didistribusikan |
| P13 | Setelah finalisasi, rapor terkunci bagi Guru dan Wali Kelas. Hanya Administrator yang dapat mengubah data yang sudah final |
| P14 | **Administrator memiliki akses penuh terhadap seluruh data, termasuk nilai akademik** |
| P15 | AI hanya membaca data milik siswa yang bersangkutan dan tidak menulis ke data akademik |
| P16 | Presensi hanya disajikan sebagai persentase kehadiran terhadap jumlah sesi yang dibuka Guru, tanpa ambang minimum, peringatan, maupun anjuran. Izin dan Sakit dihitung sebagai kehadiran |
| P17 | Akun Guru dan Siswa dibuat dengan kata sandi awal yang dihasilkan sistem. Guru tidak memiliki peran apa pun sampai dihubungkan dengan mata pelajaran, dan Siswa tidak berada di kelas mana pun sampai kelas dibuat |
| P18 | Satu proses pembuatan kelas menghasilkan tepat satu kelas |
| P19 | Satu tahun ajaran hanya memiliki **satu semester aktif** pada satu waktu |
| P20 | Guru masuk aplikasi menggunakan NIP, Siswa menggunakan NIS, keduanya dengan kata sandi awal yang dibuat sistem |
| P21 | **Setiap interaksi yang memasukkan atau mengubah data wajib menampilkan pemberitahuan berhasil atau gagal.** Pemberitahuan gagal menyebutkan alasannya |
| P22 | Nilai dan presensi baru tersimpan setelah Guru menekan tombol simpan. Tidak ada penyimpanan otomatis |
| P23 | Guru Mata Pelajaran tidak dapat mengunduh nilai maupun rapor. Pengunduhan rapor hanya tersedia bagi Administrator, Wali Kelas, dan Siswa |

---

## 2. Use Case Utama

| ID | Pelaku | Kegiatan | Berhasil apabila |
|---|---|---|---|
| UC-01 | Semua pengguna | Masuk dan keluar aplikasi | Pengguna hanya melihat menu sesuai kewenangannya |
| UC-02 | Administrator | Membuat kelas beserta periode akademik dan unggah daftar siswa | Satu kelas terbentuk, siswa termuat, Guru mata pelajaran terhubung, dan Wali Kelas tertetapkan |
| UC-03 | Administrator | Membuat akun Guru dan Siswa melalui unggah CSV atau manual | Akun unik dan aktif, kata sandi awal terbentuk otomatis, dan hasil proses diberitahukan |
| UC-04 | Administrator | Menghubungkan Guru ke mata pelajaran-jenjang lalu kelas ke Guru | Kelas memperoleh mata pelajaran secara otomatis, dan penghubungan dengan jenjang yang tidak cocok ditolak |
| UC-05 | Administrator | Menetapkan KKM per mata pelajaran-jenjang | KKM tersimpan pada mata pelajaran yang bersangkutan |
| UC-06 | Administrator | Mengoreksi nilai akademik | Perubahan tersimpan dan berlaku pada perhitungan nilai |
| UC-07 | Guru Mapel | Mengisi nilai lalu menekan Simpan Nilai | Nilai tersimpan pada kelas tugas Guru dan hasilnya diberitahukan |
| UC-08 | Guru Mapel | Membuka sesi presensi, menetapkan status, lalu menekan Simpan Presensi | Seluruh siswa memperoleh tepat satu status sejak sesi terbuka, dan sesi yang keliru dapat dihapus |
| UC-09 | Guru Mapel | Menginput nilai yang langsung terlihat siswa | Nilai tampil pada halaman siswa yang bersangkutan tanpa langkah publikasi |
| UC-10 | Wali Kelas | Melihat seluruh nilai mata pelajaran kelas walinya | Semua mata pelajaran terlihat tanpa izin mengubah nilai Guru lain |
| UC-11 | Wali Kelas | Memeriksa kelengkapan seluruh mata pelajaran | Mata pelajaran yang belum lengkap ditampilkan secara jelas |
| UC-12 | Wali Kelas | Memfinalisasi rapor semester | Berhasil hanya apabila seluruh mata pelajaran sudah lengkap |
| UC-13 | Wali Kelas | Mendistribusikan dan mengunduh rapor | Status dan waktu distribusi tersimpan, dan berkas rapor dapat diunduh |
| UC-14 | Siswa | Melihat nilai dan persentase presensi sendiri | Data siswa lain tidak dapat dibuka |
| UC-15 | Siswa | Menekan tombol Suggestion | Keluaran disusun dari seluruh data semester berjalan miliknya, memuat rekomendasi, alasan, dan dua pilihan tindakan, serta tidak tersimpan setelah halaman dimuat ulang |
| UC-16 | Siswa | Melihat dan mengunduh rapor | Rapor hanya tersedia setelah didistribusikan |

> **Penomoran:** identitas UC pada dokumen ini **tidak sama** dengan baseline PRD MVP Final. UC-06 baseline adalah *Guru mengisi nilai*, sedangkan UC-06 di sini adalah *Administrator mengoreksi nilai*. Dokumen turunan seperti Test Case dan UAT wajib merujuk penomoran dokumen ini.
>
> **Prinsip use case:** setiap kegiatan harus memiliki pelaku yang jelas, batas akses yang jelas, hasil yang dapat diperiksa, dan pesan kesalahan yang dapat dipahami apabila proses tidak dapat dilanjutkan.

---

## 3. Layar Minimum

| Pengguna | Layar yang wajib tersedia |
|---|---|
| **Semua** | Masuk, tombol lupa kata sandi yang menampilkan pesan menghubungi Wali Kelas atau Administrator, profil, serta halaman akses ditolak |
| **Administrator** | Dasbor, pembuatan akun beserta unggah berkas dan unduh templat, mata pelajaran per jenjang beserta KKM dan Guru pengampu, pembuatan kelas beserta periode akademik dan unggah daftar siswa, penghubungan Guru mata pelajaran, penetapan Wali Kelas, dan pengelolaan nilai |
| **Guru Mapel** | Dasbor kelas yang diajar, daftar siswa, pengisian nilai beserta tombol Simpan Nilai, daftar dan pembukaan sesi presensi, pengisian status beserta tombol Simpan Presensi, dan ringkasan presensi |
| **Wali Kelas** | Dasbor kelas wali, ringkasan presensi kelas, kesiapan setiap mata pelajaran, catatan rapor, finalisasi, distribusi, dan unduh rapor |
| **Siswa** | Dasbor, nilai dan bobot, persentase presensi sendiri per mata pelajaran, tombol Suggestion, serta rapor |

---

## 4. Kriteria Aplikasi Dianggap Siap

| ID | Kriteria |
|---|---|
| AC-01 | Administrator dapat menyiapkan satu semester lengkap tanpa data ganda maupun penugasan yang keliru |
| AC-02 | Kelas memperoleh mata pelajaran secara otomatis dari Guru yang dihubungkan kepadanya, dan Guru hanya melihat penugasannya sendiri |
| AC-03 | Guru tidak dapat membuka kelas atau mata pelajaran yang bukan tugasnya |
| AC-04 | Templat komponen penilaian dan bobot tersedia sejak awal, dan perubahan yang membuat jumlah bobot tidak sama dengan 100% ditolak disertai pesan yang menyebutkan total saat ini |
| AC-05 | Perhitungan nilai sesuai KKM mata pelajaran dan templat bobot yang berlaku |
| AC-06 | Nilai kosong ditampilkan sebagai belum lengkap, bukan sebagai nol |
| AC-07 | Sistem menolak finalisasi rapor semester apabila masih terdapat mata pelajaran yang belum lengkap, dan menampilkan pesan "Data Mapel *nama* belum ada, tolong hubungi guru yang bertanggung jawab" untuk setiap mata pelajaran tersebut |
| AC-08 | Guru Mata Pelajaran tidak memiliki jalur apa pun untuk memfinalisasi rapor |
| AC-09 | Wali Kelas dapat melihat seluruh mata pelajaran kelasnya tetapi tidak dapat mengubah nilai Guru lain |
| AC-10 | Siswa tidak dapat membuka data atau rapor siswa lain |
| AC-11 | Sesi yang baru dibuka langsung memuat seluruh siswa kelas dengan status Alpa, tombol Hadir Semua mengubah seluruhnya menjadi Hadir, dan satu sesi tidak memiliki status ganda untuk siswa yang sama |
| AC-12 | Presensi hanya menampilkan persentase kehadiran terhadap jumlah sesi yang dibuka Guru, tanpa peringatan maupun anjuran |
| AC-13 | Rapor yang diterima siswa sama dengan data yang sudah difinalisasi |
| AC-14 | Rapor yang sudah final tidak dapat diubah Guru maupun Wali Kelas |
| AC-15 | Nilai dan presensi hanya tersimpan setelah tombol simpan ditekan; meninggalkan halaman tanpa menyimpan tidak mengubah data |
| AC-16 | Rekomendasi hanya muncul setelah tombol Suggestion ditekan, dan hilang ketika halaman dimuat ulang |
| AC-17 | Data yang dibaca AI hanya milik siswa yang bersangkutan pada semester berjalan, mencakup nilai, KKM, kelengkapan, dan presensi |
| AC-18 | Keluaran AI menggunakan bahasa profesional dan tidak memuat prakiraan kelulusan maupun perbandingan antarsiswa |
| AC-19 | Fakta sumber, periode data, dan penanda Data Sementara tetap terlihat pada halaman meskipun susunan jawaban AI berbeda-beda |
| AC-20 | AI tidak pernah mengubah nilai, presensi, status final, maupun distribusi |
| AC-21 | Kegagalan layanan AI tidak menghambat input nilai, presensi, finalisasi, maupun distribusi |
| AC-22 | KKM bernilai awal 75 dan dapat diubah Administrator |
| AC-24 | Penghubungan kelas dengan Guru yang jenjang mata pelajarannya tidak sesuai ditolak disertai pesan yang menyebutkan kedua jenjang |
| AC-25 | Penghapusan sesi menghapus seluruh status di dalamnya dan persentase kehadiran menyesuaikan |
| AC-26 | Unggah CSV maupun Excel yang berhasil sebagian melaporkan baris yang gagal beserta alasannya, dan tidak menyisakan akun atau kelas setengah jadi |
| AC-27 | Setiap interaksi yang memasukkan atau mengubah data menampilkan pemberitahuan berhasil atau gagal, dan pemberitahuan gagal menyebutkan alasannya |
| AC-28 | Guru yang belum dihubungkan dengan mata pelajaran tidak memiliki menu mengajar, dan Siswa yang belum masuk kelas tidak memiliki data akademik |
| AC-29 | Izin dan Sakit terhitung sebagai kehadiran pada persentase presensi, dan hanya Alpa yang menguranginya |
| AC-30 | Siswa hanya melihat persentase presensi per mata pelajaran, tanpa jalur apa pun untuk membuka rincian per tanggal |
| AC-31 | Setiap keluaran AI memuat rekomendasi belajar, alasan rekomendasi, dan dua pilihan tindakan yang realistis |
| AC-32 | Guru Mata Pelajaran tidak memiliki jalur mengunduh nilai maupun rapor |
| AC-33 | Tombol lupa kata sandi hanya menampilkan pesan menghubungi Wali Kelas atau Administrator, tanpa mengirim tautan maupun mengubah kata sandi |
| AC-23 | **Kriteria penutup.** Minimal 90% skenario uji pengguna berhasil dan tidak terdapat kesalahan kritis yang masih terbuka |

---

## 5. Hal yang Harus Divalidasi dengan Sekolah

| # | Yang harus divalidasi |
|---|---|
| V1 | Komponen penilaian dan bobot pada templat bawaan, apakah sesuai dengan yang benar-benar digunakan |
| V2 | Nilai KKM untuk setiap mata pelajaran-jenjang |
| V3 | Aturan remedial dan cara mengganti nilai setelah remedial |
| V5 | Format rapor resmi sekolah dan data wajib yang harus tercantum |
| V6 | Kebijakan privasi, lama penyimpanan data, pencadangan, dan penggunaan data nyata untuk AI |
