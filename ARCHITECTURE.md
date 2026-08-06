# Arsitektur — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v0.1 — **rancangan, belum berisi** |
| **Tanggal** | 6 Agustus 2026 |
| **Disusun oleh** | Re:Code |
| **Sumber kebenaran** | [PRD.md](PRD.md) v3.0, [RFC-001](RFC-001-model-data-konseptual.md), dan [Techstack.md](Techstack.md) v2.0 |
| **Kedudukan** | Menetapkan **bagaimana bagian-bagian sistem terhubung**. Menggantikan [ARCHITECTURE-2026-08-02.md](ARCHITECTURE-2026-08-02.md), yang diturunkan menjadi arsip |

> **Status dokumen: kerangka.** Isi setiap pasal belum ditulis. Yang tercantum di bawah hanyalah rancangan susunan beserta keterangan singkat mengenai apa yang akan dimuat masing-masing.
>
> Pilihan teknologi beserta alasannya berada pada [Techstack.md](Techstack.md); dokumen ini tidak mengulangnya. Prosedur penerapan dan operasional berada pada [DEPLOYMENT.md](DEPLOYMENT.md).

---

## Rancangan Susunan

| Pasal | Judul | Yang akan dimuat | Asal isi |
|:--:|---|---|---|
| 1 | Prinsip | Prinsip arsitektural yang menjadi alasan pilihan pada dokumen ini, terutama: yang dapat dijamin basis data tidak diserahkan kepada disiplin kode | Techstack v1 §1 prinsip ② |
| 2 | Peta besar | Diagram keseluruhan sistem — CloudFront, S3, Function URL, Lambda, RDS, NAT, layanan AI — beserta penjelasan bahwa tidak ada sumber daya yang dapat dihubungi langsung dari internet | Techstack v1 §3 |
| 3 | Pintu masuk dan pembagian path | Satu domain dua origin, pembagian `/*` dan `/api/*`, serta akibatnya: CORS hilang dan cookie `HttpOnly` menjadi mungkin | Techstack v1 §5 |
| 4 | Frontend | Bentuk aplikasi React, kebutuhan yang membentuknya — pemberitahuan berhasil atau gagal, penyimpanan hanya setelah tombol ditekan, pemutaran matriks nilai, tata letak responsif | Techstack v1 §5 |
| 5 | Struktur kode dan batas modul | Susunan `src/`, tanggung jawab tiap lapisan, dan tabel aturan impor yang menjaga `domain/` bebas I/O | Techstack v1 §6.1 dan §6.2 |
| 6 | Penerapan Lambda Web Adapter | Cara kerja adapter, Dockerfile, parameter fungsi, disiplin connection pool, reserved concurrency, dan cold start | Techstack v1 §6.3 |
| 7 | Pembatasan laju | Letaknya di dalam Express, penyandaran pada identitas pengguna, dan penyimpanan penghitung di PostgreSQL | Techstack v1 §6.4 |
| 8 | Dua role basis data | `app_rw` dan `app_ro`, pernyataan `GRANT`, dan cara penguji membuktikan AI tidak dapat menulis | Techstack v1 §7.2 |
| 9 | Autentikasi dan sesi | Alur masuk, bentuk token sesi, pemeriksaan kewenangan per request, dan perintah CLI pembuatan akun Administrator | Techstack v1 §8 |
| 10 | AI Insight | Alur endpoint Suggestion, susunan prompt, minimalisasi data, penanganan kegagalan lunak, dan batas waktu | Techstack v1 §9 dan §9.2 |
| 11 | Berkas rapor | Finalisasi sebagai pembekuan data, render saat unduh, dan penyajian lewat presigned URL | Techstack v1 §10 |
| 12 | Jaringan dan keamanan | Tabel lapisan jaringan, Function URL dengan `AWS_IAM`, header keamanan, pengelolaan rahasia, dan dua peringatan: sertifikat ACM di `us-east-1` serta pembuktian penandatanganan OAC atas request ber-body | Techstack v1 §11 |
| 13 | Portabilitas ke on-prem | Tabel lapisan beserta apa yang berubah ketika dipasang di server sekolah | Techstack v1 §14 |
| 14 | **Alur request** | **Baru — belum pernah ditulis.** Dua alur ditelusuri langkah demi langkah: **Simpan Nilai sekelas** dari klik sampai `COMMIT`, dan **tombol Suggestion** dari klik sampai teks tampil. Termasuk pemeriksaan kewenangan, validasi, transaksi, dan penanganan kegagalan di setiap langkah | — |
| — | Lampiran Catatan Keputusan | Bernomor `CK-A-01` dan seterusnya, sehingga tidak bertabrakan dengan `CK-xx` pada Techstack.md maupun `CK-D-xx` pada DEPLOYMENT.md. Dimulai kosong | — |

---

## Catatan penyusunan

**Pasal 14 adalah satu-satunya isi yang benar-benar baru.** Tiga belas pasal lainnya memindahkan isi yang sudah tertulis dan tervalidasi pada `Techstack.md` versi 1. Alur request belum pernah ditulis di dokumen mana pun, padahal itulah yang paling sering dicari saat implementasi dimulai.

**Rujukan pasal di dalam Catatan Keputusan pada `Techstack.md`** — misalnya §6.3 atau §11 — mengacu pada penomoran dokumen tersebut sebelum pemecahan. Isi yang dirujuk kini berada pada dokumen ini. Entri Catatan Keputusan tidak disunting, sesuai konvensi [README.md](README.md).

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 6 Agustus 2026 | Kerangka dibuat sebagai bagian dari pemecahan `Techstack.md` menjadi tiga dokumen. Isi belum ditulis. Menggantikan `ARCHITECTURE.md` versi 2 Agustus 2026, yang diturunkan menjadi arsip dengan nama `ARCHITECTURE-2026-08-02.md` |
