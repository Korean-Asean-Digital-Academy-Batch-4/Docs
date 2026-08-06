# Penerapan dan Operasional — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v0.1 — **rancangan, belum berisi** |
| **Tanggal** | 6 Agustus 2026 |
| **Disusun oleh** | Re:Code |
| **Sumber kebenaran** | [Techstack.md](Techstack.md) v2.0 dan [ARCHITECTURE.md](ARCHITECTURE.md) |
| **Kedudukan** | Menetapkan **bagaimana sistem dikirim dan dioperasikan**, baik di AWS maupun di server sekolah |

> **Status dokumen: kerangka.** Isi setiap pasal belum ditulis. Yang tercantum di bawah hanyalah rancangan susunan beserta keterangan singkat mengenai apa yang akan dimuat masing-masing.
>
> Pilihan teknologi berada pada [Techstack.md](Techstack.md); hubungan antar bagian berada pada [ARCHITECTURE.md](ARCHITECTURE.md). Dokumen ini tidak mengulang keduanya.
>
> Dokumen ini berubah jauh lebih sering daripada kedua dokumen di atas, karena isinya mengikuti keadaan infrastruktur yang berjalan. Itulah alasan pemisahannya.

---

## Rancangan Susunan

| Pasal | Judul | Yang akan dimuat | Asal isi |
|:--:|---|---|---|
| 1 | Susunan `infra/` | Pohon direktori Terraform beserta tanggung jawab tiap modul: `network`, `data`, `compute`, `frontend`, `observability` | Techstack v1 §12 |
| 2 | Terraform | Pemisahan `bootstrap/` beserta alasannya, lingkungan `dev` dan `prod`, provider alias `us-east-1` untuk ACM, pengelolaan state dan penguncian, konvensi penamaan sumber daya, serta **pembuatan rahasia di luar Terraform** sehingga hanya ARN yang masuk ke state | Techstack v1 §12 dan Techstack v2 §7, diperluas |
| 3 | CI/CD | Tahapan pada pull request dan pada merge, pertukaran token OIDC dengan IAM role, pembangunan image, urutan migrasi sebelum fungsi diperbarui, dan pemindahan alias | Techstack v1 §12 |
| 4 | Lingkungan pengembangan dan pengujian | `docker compose up`, adapter lokal, serta tiga tingkat pengujian — unit, integrasi, dan pembuktian penegakan oleh basis data | Techstack v1 §13 |
| 5 | **Urutan penaikan pertama** | **Baru.** Infrastruktur dinaikkan lebih dahulu dengan API yang hanya memuat `GET /healthz`, sehingga VPC, RDS, Function URL, CloudFront, dan pipeline terbukti hidup sebelum backend selesai. Termasuk **pembuktian penandatanganan OAC atas request ber-body**, yang wajib dilakukan pada tahap ini | sebagian dari Techstack v1 §12 |
| 6 | **Rollback** | **Baru — belum pernah ditulis.** Pengembalian aplikasi dengan memindahkan alias Lambda ke versi sebelumnya. Yang lebih penting: **migrasi basis data tidak dapat dikembalikan otomatis**, sehingga perlu aturan tersendiri — migrasi wajib kompatibel mundur, dan penghapusan kolom dipisahkan dari penambahannya ke rilis berikutnya | — |
| 7 | **Pemasangan on-prem** | **Baru.** Skrip `install.sh` tunggal dan idempoten (CK-15) beserta sembilan langkahnya: pemeriksaan prasyarat, pemasangan Docker, penyusunan `.env` dengan kata sandi acak, `docker compose up`, migrasi, pembuatan akun Administrator, pemasangan timer pencadangan, penyalaan pembaruan keamanan otomatis, dan verifikasi akhir. Disertai `docker-compose.yml` untuk on-prem dan runbook satu halaman | — |
| 8 | **Operasional** | **Baru.** Pencadangan beserta **prosedur pemulihannya yang sudah pernah diuji**, rotasi kredensial basis data di Secrets Manager dan penggantian kunci Elice di Parameter Store, alarm CloudWatch, AWS Budgets, pemantauan sisa kredit layanan AI, dan verifikasi biaya lewat AWS Pricing Calculator | — |
| — | Lampiran Catatan Keputusan | Bernomor `CK-D-01` dan seterusnya, sehingga tidak bertabrakan dengan `CK-xx` pada Techstack.md maupun `CK-A-xx` pada ARCHITECTURE.md. Dimulai kosong | — |

---

## Catatan penyusunan

**Empat dari delapan pasal adalah isi baru.** Pasal 1 sampai 4 memindahkan isi yang sudah tertulis pada `Techstack.md` versi 1; pasal 5 sampai 8 belum pernah ditulis di dokumen mana pun.

**Dua pasal yang paling menentukan dan paling mudah ditunda:**

**Pasal 6 — rollback.** Pengembalian aplikasi mudah karena artefaknya image. Yang tidak mudah adalah basis data: migrasi yang sudah berjalan tidak dapat dibatalkan begitu saja, terlebih pada basis data berisi data sekolah sungguhan. Aturannya perlu ditetapkan sebelum migrasi pertama ditulis, bukan sesudahnya.

**Pasal 8 — pencadangan beserta pemulihannya.** Pencadangan yang belum pernah diuji pemulihannya bukanlah pencadangan. Ini berlaku bagi RDS maupun bagi server sekolah, dan berkaitan langsung dengan **V6** pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md) yang masih menunggu validasi.

**Kapan dokumen ini mulai diisi.** Ketika Terraform mulai ditulis. Menuliskannya lebih awal berarti mengarang nama modul, prosedur, dan angka yang belum diketahui.

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 6 Agustus 2026 | Kerangka dibuat sebagai bagian dari pemecahan `Techstack.md` menjadi tiga dokumen. Isi belum ditulis |
