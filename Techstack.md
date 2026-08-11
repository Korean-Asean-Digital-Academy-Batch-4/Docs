# Techstack — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v2.0 |
| **Tanggal** | 6 Agustus 2026 |
| **Disusun oleh** | Re:Code |
| **Sumber kebenaran** | [PRD.md](PRD.md) v3.0, [ATURAN-DAN-KRITERIA.md](ATURAN-DAN-KRITERIA.md) v1.0, dan [RFC-001](RFC-001-model-data-konseptual.md) — terutama §8 Kendala bagi dokumen arsitektur |
| **Kedudukan** | Menetapkan **teknologi apa yang dipilih dan mengapa**. Menggantikan bagian stack pada [ARCHITECTURE.md](ARCHITECTURE.md) versi 2 Agustus 2026, yang diturunkan menjadi arsip |
| **Dokumen lanjutan** | [ARCHITECTURE.md](ARCHITECTURE.md) — bagaimana bagian-bagian terhubung · [DEPLOYMENT.md](DEPLOYMENT.md) — bagaimana sistem dikirim dan dioperasikan · `SCHEMA.md` · `API.md` |

> Dokumen ini menjawab **apa** yang dipilih dan **kenapa**. Dokumen ini **tidak** menjelaskan cara bagian-bagian terhubung, tidak memuat diagram jaringan, tidak memuat batas modul, dan tidak memuat prosedur penerapan. Ketiganya berada pada `ARCHITECTURE.md` dan `DEPLOYMENT.md`.
>
> Entitas dan invarian basis data ditetapkan [RFC-001](RFC-001-model-data-konseptual.md); aturan peran dan izin ditetapkan [aktor-role.md](aktor-role.md). Dokumen ini tidak mengulang keduanya.
>
> Mengikuti [konvensi dokumen](README.md), badan dokumen memuat **deskripsi keadaan yang berlaku** dan disunting langsung ketika berubah, sedangkan **Lampiran Catatan Keputusan** bernomor dan bertanggal serta hanya ditambah, tidak disunting.

---

## 1. Prinsip

Tiga prinsip berikut menjadi alasan hampir setiap pilihan pada dokumen ini.

**① Portabilitas dibuktikan, bukan diklaim.**
Artefak yang dijalankan di AWS adalah **image Docker yang sama persis** dengan yang dijalankan di laptop pengembang dan di server sekolah. Karena tim menjalankannya setiap hari, portabilitas terbukti sendiri tanpa pengujian khusus.

**② Kompleksitas hanya dibayar apabila ada kebutuhan produk yang membayarnya.**
RFC-001 C-07 mencabut kebutuhan antrean, dan volume pada C-06 kecil. Antrean pesan, worker terpisah, dan pekerjaan latar karenanya tidak dibangun.

**③ Ketergantungan pada AWS dikurung di satu tempat.**
SDK AWS hanya boleh muncul di `adapters/aws/`. Lapisan domain, rute, dan basis data tidak mengetahui keberadaan AWS.

Prinsip keempat — bahwa yang dapat dijamin basis data tidak diserahkan kepada disiplin kode — bersifat arsitektural dan berada pada [ARCHITECTURE.md](ARCHITECTURE.md).

---

## 2. Stack

| Lapisan | Pilihan | Versi |
|---|---|---|
| **Frontend** | React SPA, dibangun dengan Vite | React 19, Vite 7 |
| **Bahasa** | TypeScript, `strict` di frontend maupun backend | TypeScript 5.x |
| **Backend** | Express, dikemas sebagai container image | Express 5, Node.js 24 LTS |
| **ORM** | Drizzle | — |
| **Basis data** | PostgreSQL di Amazon RDS | PostgreSQL 17, `db.t4g.micro` |
| **Autentikasi** | Dikelola sendiri: Argon2id + sesi lewat cookie `HttpOnly` | — |
| **AI** | Elice AI Cloud (KADA), antarmuka setara OpenAI | `/v1/chat/completions` |
| **Penyajian frontend** | S3 (privat, OAC) + CloudFront | — |
| **Penyajian backend** | AWS Lambda dari container image, memakai **Lambda Web Adapter**, diakses lewat **Function URL** dengan CloudFront OAC | LWA 1.0.x, arm64 |
| **Penyimpanan berkas** | S3, diakses lewat presigned URL | — |
| **Validasi** | Zod, di batas HTTP maupun batas berkas unggahan | — |
| **Berkas rapor** | pdfmake, dirender saat finalisasi dengan render-saat-unduh sebagai cadangan (CK-A-07) | — |
| **IaC** | Terraform | — |
| **CI/CD** | GitHub Actions + OIDC | — |
| **Pemasangan on-prem** | Skrip `install.sh` tunggal, idempoten | — |
| **Nama domain dan DNS** | Satu domain, Cloudflare sebagai nameserver otoritatif untuk AWS maupun on-prem — CK-17 | — |
| **Jalan masuk on-prem** | Cloudflare Tunnel (`cloudflared`) — CK-17 | — |
| **Region** | **ap-southeast-3 (Jakarta)** — CK-16. Sertifikat ACM untuk CloudFront tetap `us-east-1` | — |

**Yang sengaja tidak dipakai:** API Gateway, Application Load Balancer, ECS, SQS, Cognito, Bedrock, RDS Proxy, dan Ansible. Alasan masing-masing tercatat pada Lampiran Catatan Keputusan.

**Catatan bentuk artefak.** Backend berjalan di Lambda, tetapi yang di-deploy tetap **image Docker berisi Express yang mendengarkan di sebuah port** — bukan fungsi bergaya Lambda. Aplikasi tidak mengetahui keberadaan Lambda, dan image yang sama dijalankan di laptop, di ECS Fargate, maupun di server sekolah tanpa perubahan. Mekanismenya diuraikan pada [ARCHITECTURE.md](ARCHITECTURE.md).

---

## 3. Pemenuhan kendala RFC-001 §8

Tabel ini adalah pertanggungjawaban langsung dokumen ini terhadap RFC-001. Setiap kendala harus memiliki jawaban yang dapat ditunjuk.

| # | Kendala | Cara dipenuhi |
|---|---|---|
| **C-01** | Data relasional sampai akar | PostgreSQL. Rata-rata berbobot, penggabungan lintas mata pelajaran, agregat kehadiran, dan pemeriksaan kelengkapan seluruhnya dikerjakan sebagai query SQL |
| **C-02** | Penulisan atomik atas banyak baris | Satu penekanan **Simpan Nilai** menjadi satu request, satu transaksi. Kegagalan membatalkan seluruh baris. Memenuhi P22 dan AC-15 |
| **C-03** | Penghapusan berantai | `ON DELETE CASCADE` dari `sesi` ke `presensi`, memenuhi I-16 dan AC-25 |
| **C-04** | Penyimpanan berkas rapor terpisah | S3 dengan presigned URL berumur pendek |
| **C-05** | Pembedaan hak baca dan tulis pada tingkat data | Dua role basis data, `app_rw` dan `app_ro`. Jalur AI memakai koneksi yang tidak memiliki hak tulis sama sekali |
| **C-06** | Volume kecil, bukan pendorong pemilihan | `db.t4g.micro` Single-AZ memadai dengan margin besar. Ukuran instance tidak menjadi pertimbangan |
| **C-07** | Jalur AI sinkron, sekali jalan, hanya membaca | Panggilan HTTP biasa di dalam request handler. **Tidak ada antrean, tidak ada worker, tidak ada pekerjaan latar** |

C-07 adalah pencabutan terbesar terhadap rancangan 2 Agustus 2026. Antrean pada rancangan tersebut dibenarkan oleh AI yang menulis deskripsi naratif ke rapor secara latar; PRD v3.0 mencabut seluruh dasar itu.

---

## 4. Basis data

### 4.1 Mesin — PostgreSQL

PostgreSQL 17 di Amazon RDS, `db.t4g.micro`, Single-AZ, terenkripsi at-rest, dengan pencadangan otomatis ~~7 hari~~ **1 hari — diamandemen CK-19**, dilengkapi snapshot manual sebelum tindakan berisiko.

Empat kemampuan PostgreSQL menjadi penentu pemilihan, dan seluruhnya berasal dari invarian RFC-001 §6:

| Kemampuan | Invarian yang ditegakkan |
|---|---|
| **Partial unique index** — `UNIQUE (...) WHERE aktif` | I-03 satu semester aktif per tahun ajaran; I-09 satu wali kelas per kelas |
| **Composite foreign key** — `(mapel_ref, tingkat) → mapel(id, tingkat)` | I-06 jenjang kelas wajib sama dengan jenjang mata pelajaran, sehingga ketidakcocokan **mustahil tersimpan** dan bukan sekadar dicegah pemeriksaan aplikasi (AC-24) |
| **JSONB** | `rapor_mapel.snapshot_komponen` dan `audit_log.sebelum`/`sesudah` |
| **DDL transaksional** | Migrasi yang gagal batal seluruhnya, tidak meninggalkan skema separuh jadi pada basis data berisi data sekolah sungguhan |

Dua di antaranya menentukan sendirian. Tanpa **partial unique index**, I-03 dan I-09 harus dijaga disiplin kode. Tanpa **DDL transaksional**, migrasi yang gagal di tengah jalan meninggalkan skema yang tidak diketahui bentuknya.

### 4.2 ORM — Drizzle

Skema, migrasi, dan query ditulis dekat dengan SQL, sehingga partial index, composite foreign key, dan `ON DELETE CASCADE` pada §4.1 dapat dinyatakan langsung tanpa jalan memutar. Migrasi berupa berkas SQL yang dapat dibaca dan ditinjau.

---

## 5. Autentikasi

Kredensial dikelola sendiri di dalam PostgreSQL. Tidak ada layanan identitas terkelola. Menutup **K-01** pada RFC-001 §9.

| Aspek | Pilihan | Dasar |
|---|---|---|
| Pengenal masuk | `pengguna.nama_pengguna` — NIP bagi Guru, NIS bagi Siswa, nama pengguna tersendiri bagi Administrator | P20, RFC-001 §7.1 |
| Penyimpanan kata sandi | Argon2id | — |
| Sesi | Token acak di dalam cookie `HttpOnly; Secure; SameSite=Strict` | — |
| Kata sandi awal | Dihasilkan sistem, ditampilkan sekali kepada Administrator | §6.1.3, P17 |
| Syarat kerumitan | **Tidak ada** | §6.1.3 |
| Kewajiban ganti saat masuk pertama | **Tidak ada** | §6.1.3 |
| Pemulihan mandiri | **Tidak ada** | §6.1.3, AC-33 |
| Akun Administrator | Dibuat langsung ke basis data lewat perintah CLI, bukan lewat antarmuka aplikasi | Menutup T-03 pada RFC-001 §10 |

Seluruh ketentuan §6.1.3 di atas merupakan **perilaku bawaan** dari pendekatan ini, bukan hasil mematikan fitur pada layanan pihak lain. Inilah alasan utama pemilihannya (CK-05).

Alur sesi, pemeriksaan kewenangan per request, dan bentuk perintah CLI berada pada [ARCHITECTURE.md](ARCHITECTURE.md).

---

## 6. Layanan AI

Tombol Suggestion (PRD §8.5) dilayani **Elice AI Cloud** melalui program KADA.

| Aspek | Pilihan |
|---|---|
| Antarmuka | **Setara OpenAI** — `POST /v1/chat/completions`, otorisasi `Bearer` |
| Model | **Gemini 3.6 Flash**, dipilih dari Model Library Elice (11 Agustus 2026) |
| Alamat | Dedicated endpoint dengan jalur OpenAI di belakangnya: `https://mlapi.run/{endpoint-id}/v1/chat/completions`. Satu endpoint melayani **satu model**; `{endpoint-id}` itulah yang membedakannya. Jalur lain yang tersedia pada endpoint yang sama: `GET /v1/models`, `GET /v1/models/{model_id}`, dan `POST /v1/responses` — tidak satu pun dipakai selain `/v1/models` untuk memastikan nilai bidang `model` |
| Bahasa keluaran | **Bahasa Indonesia**, sejalan dengan seluruh pesan sistem ([API.md §2.2](API.md)). [PRD §8.5](PRD.md) menetapkan gayanya profesional tetapi tidak menyebut bahasanya |
| Kunci API | SSM Parameter Store (SecureString) di AWS; berkas `.env` di on-prem dan pengembangan. Rinciannya pada §7 |
| Bentuk pemanggilan | Sinkron, sekali jalan, hanya membaca. Tidak ada antrean dan tidak ada penyimpanan keluaran (I-24, NG14) |
| Biaya | Kredit program KADA, **di luar tagihan AWS** |

Kesetaraan dengan antarmuka OpenAI inilah yang membuat pilihan ini tidak mengikat. Adapter yang ditulis adalah klien OpenAI-compatible biasa, sehingga berpindah penyedia — ke layanan lain, atau ke model yang dipasang sendiri di server sekolah lewat vLLM, Ollama, maupun LiteLLM — berarti **mengganti base URL dan kunci API**, bukan menulis ulang adapter.

**Identitas siswa tidak pernah dikirim.** Prompt hanya memuat nama mata pelajaran, nilai per komponen, KKM, kelengkapan, topik, dan persentase kehadiran. Nama dan NIS tidak disertakan. Ini menyempitkan cakupan **V6** pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md), yang mensyaratkan penggunaan data nyata untuk AI divalidasi dengan sekolah.

**Bentuk permintaannya ditetapkan [payload.md](payload.md)**, yang diverifikasi langsung terhadap endpoint pada 11 Agustus 2026. Tiga ketentuannya mengikat adapter:

| Ketentuan | Sebab |
|---|---|
| `max_tokens` **minimal 2000** | Gemini 3.6 Flash adalah model penalaran, dan token penalaran dihitung terhadap anggaran yang sama dengan token jawaban. Anggaran 100 menghasilkan jawaban terpenggal menjadi tiga huruf |
| `finish_reason: "length"` diperlakukan sebagai **kegagalan**, bukan jawaban sah | Jawaban terpotong di tengah kalimat tidak boleh sampai ke siswa sebagai rekomendasi |
| Dua bentuk galat ditangani | Gateway Elice menjawab `{"error":{…}}`; Google menjawab larik `[{"error":{…}}]`. Field yang tidak dikenal gateway diteruskan apa adanya ke Google |

`reasoning_effort: "low"` dipakai sebagai setelan bawaan — sekitar 220 token penalaran, cukup bagi permintaan sebentuk ini. Nilai `"none"` diterima tanpa galat tetapi diabaikan, sehingga tidak boleh diandalkan untuk mematikan penalaran.

Alur pemanggilan, penanganan kegagalan, dan susunan prompt berada pada [ARCHITECTURE.md](ARCHITECTURE.md).

---

## 7. Penyimpanan rahasia

Sistem memiliki empat rahasia infrastruktur. Keempatnya **tidak pernah dibangun ke dalam image** dan tidak pernah masuk ke repositori.

| Rahasia | Isinya | Di AWS | Di on-prem | Dibaca oleh |
|---|---|---|---|---|
| Kredensial `edutrack_owner` | Nama role dan kata sandi PostgreSQL pemilik seluruh objek, satu-satunya yang boleh DDL | **Secrets Manager** | `.env`, izin `600` | Fungsi `migrate` saja |
| Kredensial `app_rw` | Nama role dan kata sandi PostgreSQL untuk jalur tulis aplikasi | **Secrets Manager** | `.env`, izin `600` | Fungsi `api` |
| Kredensial `app_ro` | Nama role dan kata sandi PostgreSQL untuk jalur AI, tanpa hak tulis | **Secrets Manager** | `.env`, izin `600` | Fungsi `api` |
| Kunci API Elice | Bearer token ke `mlapi.run` | **SSM Parameter Store**, tipe SecureString | `.env`, izin `600` | Fungsi `api` |

**Kenapa dibedakan.** Akibat kebocorannya tidak setara, dan urutannya menaik.

Kunci Elice yang bocor hanya menghabiskan kredit program, tidak menyentuh data sekolah, dan cukup dibuat ulang. Kredensial `app_rw` yang bocor memungkinkan seseorang di dalam VPC menyambung langsung ke basis data dan mengubah nilai seluruh sekolah, melewati seluruh pemeriksaan kewenangan aplikasi. Kredensial **`edutrack_owner` yang bocor memungkinkan tabelnya dijatuhkan** — kerusakan yang tidak dapat diperbaiki tanpa pemulihan cadangan, beserta hilangnya seluruh penulisan sejak titik pemulihan.

Rotasi terjadwal karenanya dibayar untuk ketiga kredensial basis data, dan tidak untuk kunci Elice.

**Kolom "dibaca oleh" adalah bagian yang menentukan.** Ketiga kredensial basis data disimpan di tempat yang sama, tetapi IAM membatasi siapa yang boleh mengambil masing-masing ([DEPLOYMENT.md §9.5](DEPLOYMENT.md)). Fungsi `api` yang melayani setiap request dari internet **tidak dapat mengambil kredensial `edutrack_owner`**, sehingga kekeliruan kode pada jalur permintaan tidak akan pernah dapat menjatuhkan tabel — sekalipun kodenya mencoba.

**Ketentuan yang berlaku bagi keempatnya:**

1. **Nilai rahasia dibuat di luar Terraform.** Terraform hanya menyimpan ARN-nya, sehingga kata sandi basis data tidak pernah berada di dalam state.
2. **Tidak pernah dicetak ke log**, baik log aplikasi maupun log CI.
3. **Dibaca sekali pada saat container menyala**, lalu disimpan di memori selama container hidup — bukan pada setiap request.
4. Pembacaannya melewati interface `Secrets` di `ports/`, sehingga perbedaan antara AWS dan on-prem tidak menyentuh kode aplikasi.

Di on-prem, keempatnya berada dalam satu berkas `.env` berizin `600` yang dibuat `install.sh` beserta kata sandi acaknya (CK-15). Untuk satu server, tidak ada tempat yang lebih baik — dan penambahan pengelola rahasia tersendiri di sana hanya menambah bagian yang dapat rusak.

Pemisahan pembaca yang di AWS ditegakkan IAM **tidak tersedia di on-prem**, karena hanya ada satu berkas dan satu proses. Konsekuensi ini diterima: pada satu server milik sekolah, pemisahannya adalah pemisahan role PostgreSQL saja, tanpa lapis kedua.

**Yang bukan termasuk di sini:** kata sandi akun Guru, Siswa, dan Administrator. Ketiganya tidak pernah menjadi rahasia infrastruktur, melainkan hash Argon2id di dalam tabel `pengguna` (§5), dan tidak dapat dibaca siapa pun termasuk tim.

---

## 8. Perkiraan biaya

### 8.1 Asumsi beban

Diturunkan dari volume RFC-001 §8.1 — 360 siswa, 18 guru, 1 administrator.

| Sumber | Perkiraan request per bulan |
|---|--:|
| Guru — 18 orang × 20 hari × ~150 request | 54.000 |
| Siswa — 360 orang × ~12 hari × ~30 request | 130.000 |
| Administrator dan lain-lain | ~15.000 |
| **Dipakai untuk perhitungan** (dibulatkan naik sebagai margin) | **300.000** |

### 8.2 Rincian

**Diverifikasi 11 Agustus 2026 terhadap AWS Price List API untuk `ap-southeast-3`**, bukan diperkirakan. Setiap tarif di bawah ini dibaca dari berkas harga resmi region tersebut pada `pricing.us-east-1.amazonaws.com/offers/v1.0/aws/<layanan>/current/ap-southeast-3/index.json`.

| Komponen | Tarif Jakarta | Per bulan |
|---|---|--:|
| Lambda `edutrack-api` — 300 rb request, arm64 1024 MB, 120 ms | $2,0·10⁻⁷ per request · $1,33334·10⁻⁵ per GB-detik | $0,54 |
| Lambda `edutrack-migrate` — 60 kali, 5 detik | sama | $0,00 |
| Function URL | — | $0 |
| RDS `db.t4g.micro` Single-AZ PostgreSQL | $0,025 per jam | $18,25 |
| RDS penyimpanan gp3 20 GB | $0,138 per GB-bulan | $2,76 |
| NAT instance `t4g.micro` — **diamandemen CK-19**, semula `t4g.nano` | $0,0106 per jam † | $7,74 † |
| NAT — EBS gp3 8 GB | $0,096 per GB-bulan | $0,77 |
| NAT — satu alamat IPv4 publik | $0,005 per jam | $3,65 |
| Secrets Manager — tiga rahasia | $0,40 per rahasia | $1,20 |
| Secrets Manager — panggilan API | $0,000005 per panggilan | $0,10 |
| ECR — 10 GB image | $0,10 per GB-bulan | $1,00 |
| S3 — 5 GB Standard beserta request-nya | $0,025 per GB-bulan | $0,26 |
| CloudFront — 15 GB keluar, 500 rb request | di bawah free tier tetap 1 TB dan 10 juta request | $0 |
| SSM Parameter Store standar · S3 gateway endpoint | — | $0 |
| Elice AI Cloud — ~1.400 panggilan tombol Suggestion | kredit program KADA | $0, di luar tagihan AWS |
| **Total** | | **$36,28** † |

† **Baris NAT belum diverifikasi terhadap Price List API.** Tarifnya diturunkan secara aritmetika sebagai dua kali tarif `t4g.nano` yang sudah diverifikasi, mengikuti pola harga keluarga `t4g`. Seluruh baris lain tetap sebagaimana diverifikasi 11 Agustus 2026. Verifikasinya menuntut kredensial dan karenanya menunggu manusia — sampai itu terjadi, angka ini **perkiraan, bukan hasil pembacaan**.

Empat keadaan yang mungkin:

| Keadaan | Per bulan |
|---|--:|
| Tanpa free tier apa pun | **$36,28** † |
| Dengan free tier RDS 12 bulan | **$15,27** † |
| NAT instance digantikan Egress-only Internet Gateway | $24,12 |
| Keduanya | **$3,11** |

**Keadaan yang sesungguhnya berlaku tidak ada pada tabel di atas.** Akun berjalan pada **Free Plan** (CK-19), dan di sana RDS `db.t4g.micro`, EC2 `t4g.micro`, serta 750 jam alamat IPv4 publik per bulan seluruhnya tercakup selama 12 bulan pertama. Yang tersisa hanyalah Secrets Manager dua rahasia (~$0,80), ECR, dan S3 — dan ketiganya menggerus kredit Free Plan alih-alih menagih.

**Yang perlu diwaspadai justru bulan ke-13.** Ketika cakupan 12 bulan habis, tagihan melompat dari nol ke angka penuh di atas sekaligus. Itu bukan alasan mengubah rancangan sekarang, tetapi alasan memasang AWS Budgets sebelum bulan itu tiba — tercatat pada Pasal 8 [DEPLOYMENT.md](DEPLOYMENT.md).

**Perkiraan lama ternyata tidak meleset.** Angka $28–36 yang disusun untuk `ap-southeast-1` melingkupi $32,41 di Jakarta; yang berbeda hanya angka free tier, yang seharusnya $11,40 alih-alih $13–21. Kekhawatiran bahwa Jakarta jauh lebih mahal daripada Singapura tidak terbukti pada bauran layanan ini.

**Free tier Lambda tidak diperhitungkan.** Akun yang masih memakai model free tier lama memperoleh 1 juta request dan 400 ribu GB-detik per bulan secara tetap, yang menutup seluruh $0,54 di atas. Karena kelayakannya bergantung pada usia akun, angkanya dibiarkan penuh — meleset ke arah yang aman.

### 8.3 Yang perlu diperhatikan

**Compute bukan pos yang perlu dioptimalkan.** Lambda menyumbang 1,7% dari tagihan; sepuluh kali lipat trafik pun menambah kurang dari $5. Tagihan didominasi **RDS (65%)** dan **NAT (26%)**; keduanya bersama-sama adalah 91% dari total.

Dua tuas yang tersisa, keduanya perlu diperiksa lebih dahulu, bukan diasumsikan berhasil:

| Tuas | Hemat | Syarat |
|---|--:|---|
| Free tier RDS | −$21,01 | Status kelayakan akun AWS tim perlu diperiksa. Berlaku 12 bulan |
| Egress-only Internet Gateway lewat IPv6, menggantikan NAT instance | −$8,29 | Hanya berlaku apabila `mlapi.run` dapat dihubungi lewat IPv6. **Wajib diuji** |

**Alamat IPv4 publik ditagih $0,005 per jam** — $3,65 per bulan, hampir separuh biaya NAT. Inilah sebabnya mengganti NAT instance dengan Egress-only Internet Gateway menghemat lebih banyak daripada yang tampak dari harga instance-nya saja.

**Sebagai pembanding**, rancangan ECS Fargate dengan dua task di belakang Application Load Balancer berbiaya sekitar $60–71 per bulan. Perbandingan ini dicatat karena Fargate tetap menjadi jalur naik apabila cold start atau plafon koneksi terbukti mengganggu (CK-13).

**Biaya AI berada di luar tagihan AWS.** Kredit program KADA bersifat terbatas dan menipis, bukan biaya berulang. Pembatasan laju per siswa adalah pengendali pemakaiannya, dan habisnya kredit tidak menghentikan aplikasi karena kegagalan layanan AI ditangani sebagai kegagalan lunak (AC-21).

---

## 9. Yang belum diputuskan

| # | Item | Menunggu | Dampak apabila berubah |
|---|---|---|---|
| 3 | Kebijakan penyimpanan dan pencadangan data | V6 | Menentukan lama retensi cadangan RDS, aturan daur hidup bucket rapor, dan jadwal pencadangan on-prem |
| 4 | **Nama domain yang sesungguhnya beserta pembeliannya**, dan penerbitan sertifikat ACM di atasnya | Pihak sekolah dan pembelian domain | **Bentuknya sudah ditetapkan CK-17**; yang tersisa hanya namanya. Menentukan modul `frontend` pada Terraform, nilai record CNAME, dan subdomain per sekolah. **Dikerjakan paling akhir dengan sengaja** — seluruh susunan CK-17 dapat ditulis dan ditinjau tanpa domain, dan hanya penerapannya yang menunggu |
| 5 | Apakah `dev` memerlukan RDS tersendiri atau cukup PostgreSQL lokal | Keputusan tim | Menentukan biaya lingkungan `dev` |
| 6 | Apakah `mlapi.run` dapat dihubungi lewat IPv6, sehingga NAT instance dapat digantikan Egress-only Internet Gateway | Uji jaringan saat infrastruktur naik | Menghemat $8,29 per bulan, yaitu 26% tagihan (§8.3) |
| ~~7~~ | ~~Status kelayakan free tier akun AWS tim~~ — **terjawab 11 Agustus 2026, CK-19** | — | Akun berada pada **Free Plan** yang membatasi, bukan memberi potongan. Pertanyaannya ternyata bukan "berapa tagihannya" melainkan "apa yang boleh dibuat" |
| 8 | Apakah cold start ~0,8–1,5 detik dapat diterima pengguna | UAT | Apabila tidak, jalur naiknya provisioned concurrency atau ECS Fargate memakai image yang sama (CK-13) |

Temuan RFC-001 §10 yang masih terbuka — T-01, T-02, T-04, T-05, dan T-06 — bersifat produk dan tidak dipengaruhi pilihan teknologi mana pun pada dokumen ini. T-03 ditutup oleh §5.

---

## Lampiran — Catatan Keputusan

Bernomor dan bertanggal. Entri tidak disunting; perubahan keputusan ditulis sebagai entri baru yang menyebut nomor yang digantikannya.

> **Catatan pembacaan.** Rujukan pasal di dalam entri di bawah ini — misalnya §6.3 atau §15 — mengacu pada penomoran dokumen **pada saat entri ditulis**, yaitu sebelum pemecahan menjadi `ARCHITECTURE.md` dan `DEPLOYMENT.md` pada 6 Agustus 2026. Entri tidak disunting, sesuai konvensi. Isi yang dirujuk kini berada pada dokumen yang bersangkutan.
>
> Catatan Keputusan pada `ARCHITECTURE.md` dan `DEPLOYMENT.md` memakai awalan tersendiri — `CK-A-xx` dan `CK-D-xx` — sehingga tidak bertabrakan dengan penomoran di sini.

### CK-01 · 6 Agustus 2026 · ECS Fargate untuk backend

**Diputuskan.** Express dijalankan sebagai container di ECS Fargate di belakang Application Load Balancer.

**Alasan.** Express adalah server long-running, sedangkan Lambda adalah fungsi. Menjalankan Express di Lambda memerlukan adapter, disiplin `max: 1` pada connection pool, dan pengelolaan cold start. Yang lebih menentukan: batas **29 detik** API Gateway tidak dapat dinaikkan, sehingga pekerjaan panjang memaksa antrean — padahal RFC-001 C-07 baru saja mencabut satu-satunya pembenaran antrean yang tersisa. Fargate juga menjadikan **image Docker** sebagai artefak, yaitu bentuk yang sama persis dengan yang akan dijalankan di server sekolah.

**Alternatif yang ditolak.**

*Lambda dengan `serverless-http`.* Biaya di bawah $1 per bulan, jauh lebih murah. Ditolak karena menuntut SQS, worker, dan DLQ hanya untuk mengakali batas platform, serta memerlukan entry point terpisah dan penukaran adapter antrean yang harus ditulis dan diuji terus-menerus demi portabilitas.

*AWS App Runner.* **Tidak tersedia.** App Runner dipindahkan ke mode pemeliharaan pada 31 Maret 2026 dan tidak menerima pelanggan baru sejak 30 April 2026. Akun baru tidak dapat membuat layanan App Runner. Jalur pengganti resmi yang ditunjuk AWS adalah ECS Express Mode.

*ECS Express Mode.* Menghemat biaya ALB dengan menyatukan sampai 25 layanan di balik satu ALB. Ditolak untuk saat ini karena mengabstraksi ALB, subnet, dan penskalaan di balik satu sumber daya berkendali terbatas, sedangkan EduTrack memerlukan koneksi ke RDS di subnet privat. Layak ditinjau ulang apabila biaya ALB memberatkan (§15).

*Elastic Beanstalk.* Menyediakan ALB, penskalaan, dan rolling deploy dalam satu paket dengan biaya setara. Ditolak karena yang di-deploy adalah bundel kode, bukan image, sehingga konfigurasi `.ebextensions` dan `Procfile` tidak ikut berpindah ke on-prem. Beanstalk juga menyisakan EC2 yang harus dipatch dan platform branch yang memiliki tanggal pensiun berkala. Ringkasnya: Beanstalk mempermudah deploy ke AWS, sedangkan Fargate mempermudah pindah dari AWS — dan syarat kedualah yang ditetapkan.

**Konsekuensi yang diterima.** Biaya naik dari ~$18–22 menjadi ~$50–55 per bulan.

### CK-02 · 6 Agustus 2026 · Tanpa API Gateway

**Diputuskan.** ALB menjadi satu-satunya penerus trafik ke aplikasi.

**Alasan.** API Gateway pada rancangan sebelumnya bukan pilihan gaya melainkan keharusan, karena Lambda memerlukan pemicu yang mengubah request HTTP menjadi event. Express tidak memerlukannya. Menambahkannya di depan ALB berarti satu hop tanpa manfaat, sekaligus mengembalikan batas 29 detik yang justru menjadi alasan meninggalkan Lambda.

**Yang hilang beserta penggantinya.** Pembatasan laju dan validasi request berpindah ke dalam Express (§6.4), sehingga keduanya ikut berpindah ke on-prem alih-alih tertinggal di AWS.

### CK-03 · 6 Agustus 2026 · Frontend statis di S3 dan CloudFront

**Diputuskan.** React dibangun menjadi berkas statis di bucket S3 privat, disajikan CloudFront pada domain yang sama dengan API.

**Alasan.** Hasil build tidak memerlukan compute. Satu domain menghilangkan CORS dan memungkinkan cookie `HttpOnly`, yang untuk data nilai anak di bawah umur jauh lebih aman daripada menyimpan token di `localStorage` yang dapat dicuri lewat XSS.

**Alternatif yang ditolak.** *React di dalam container ECS.* Membayar compute untuk pekerjaan yang tidak memerlukan compute. *Next.js dengan SSR.* Menambahkan runtime server untuk lapisan yang seluruh datanya bersifat privat per pengguna dan tidak dapat di-cache; SEO tidak relevan bagi aplikasi internal sekolah.

### CK-04 · 6 Agustus 2026 · PostgreSQL

**Diputuskan.** PostgreSQL 17 di Amazon RDS.

**Alasan.** Empat kemampuan pada §7.1 menegakkan invarian RFC-001 langsung di basis data. Yang paling menentukan adalah **partial unique index**, yang tanpanya I-03 dan I-09 harus dijaga disiplin kode, serta **DDL transaksional**, yang membuat migrasi gagal batal seluruhnya pada basis data berisi data sekolah sungguhan.

**Alternatif yang ditolak.** *MySQL 8.* Tidak memiliki partial index dan DDL-nya tidak transaksional. *DynamoDB.* Data EduTrack relasional sampai akar (C-01), dan DynamoDB tidak memiliki versi on-prem, sehingga skenario pemasangan di server sekolah menjadi mustahil tanpa menulis ulang seluruh lapisan data.

### CK-05 · 6 Agustus 2026 · Kredensial dikelola sendiri, bukan Cognito

**Diputuskan.** Kata sandi disimpan sebagai hash Argon2id pada tabel `pengguna`; sesi memakai token acak di dalam cookie `HttpOnly`. Menutup K-01 pada RFC-001 §9.

**Alasan.** PRD §6.1.3 menetapkan tanpa syarat kerumitan, tanpa kewajiban penggantian, dan tanpa pemulihan mandiri. Pada pendekatan ini ketiganya adalah **perilaku bawaan**; pada layanan identitas terkelola ketiganya adalah fitur yang harus dimatikan satu per satu. Identitas juga tetap berada pada satu sumber data, sehingga tidak ada kolom penghubung yang dapat menyimpang.

**Alternatif yang ditolak.** *Amazon Cognito.* Memperkuat narasi native AWS, tetapi menghasilkan dua sumber data yang harus dijembatani, memaksa seluruh ketentuan §6.1.3 dijalankan sebagai penonaktifan fitur, dan memerlukan penggantian penyedia identitas ketika dipasang di server sekolah.

**Konsekuensi yang diterima.** Penyimpanan kata sandi menjadi tanggung jawab tim. Dikurangi dengan Argon2id, pembatasan laju pada endpoint masuk (§6.4), dan tidak adanya jalur pemulihan mandiri yang dapat disalahgunakan.

### CK-06 · 6 Agustus 2026 · OpenRouter, bukan Bedrock

**Diputuskan.** Tombol Suggestion dilayani OpenRouter melalui interface `AiAdvisor`.

**Alasan.** Penyedia yang sama dipakai di AWS maupun on-prem, sehingga perilaku keluaran tidak berbeda antar lingkungan. Penggantian model dapat dilakukan tanpa berpindah layanan.

**Alternatif yang ditolak.** *Amazon Bedrock.* Autentikasi lewat IAM role sehingga tidak ada kunci API untuk dirotasi. Ditolak karena mengikat jalur AI ke AWS, sehingga pemasangan on-prem memerlukan penyedia lain dan keluarannya belum tentu setara.

**Konsekuensi yang diterima.** Kunci API perlu disimpan di Secrets Manager dan dirotasi. Jalur keluar internet menjadi kebutuhan tetap, sehingga NAT instance tidak dapat dihapus — dan sejak alamat IPv4 publik ditagih, pos ini menjadi 28% tagihan (§15.3). Kemungkinan penggantiannya dengan Egress-only Internet Gateway lewat IPv6 dicatat sebagai butir 6 pada §16.

### CK-07 · 6 Agustus 2026 · Tanpa antrean dan tanpa worker

**Diputuskan.** Tidak ada SQS, tidak ada proses worker, dan tidak ada pekerjaan latar.

**Alasan.** RFC-001 C-07 menetapkan jalur AI bersifat sinkron, sekali jalan, dan hanya membaca. Pembuatan berkas rapor dipindahkan ke saat unduh (CK-09), sehingga tidak ada lagi pekerjaan panjang yang tersisa. Antrean pada rancangan 2 Agustus 2026 dibenarkan oleh AI yang menulis deskripsi naratif ke rapor secara latar; PRD v3.0 mencabut seluruh dasar tersebut.

**Konsekuensi yang diterima.** Apabila kelak muncul kebutuhan pekerjaan latar yang sungguhan, jalur naiknya adalah `pg-boss` — antrean di dalam PostgreSQL itu sendiri, tanpa infrastruktur baru dan tanpa kehilangan portabilitas.

### CK-08 · 6 Agustus 2026 · Dua role basis data

**Diputuskan.** `app_rw` untuk jalur tulis aplikasi, `app_ro` untuk jalur AI.

**Alasan.** I-23 dan AC-20 mensyaratkan AI tidak pernah menulis ke data akademik. RFC-001 C-05 menuntut pembedaan tersebut ditegakkan pada tingkat data. Dengan dua role, penguji dapat membuktikannya lewat satu query alih-alih membaca kode.

### CK-09 · 6 Agustus 2026 · Rapor dirender saat diunduh

**Diputuskan.** Finalisasi hanya membekukan data ke `rapor_mapel`. Berkas PDF dirender pada saat unduhan pertama, lalu disimpan di S3 dan dipakai ulang.

**Alasan.** Karena PDF dirender dari salinan beku, hasilnya selalu sama dengan data yang difinalisasi (AC-13). Pendekatan ini menghapus kebutuhan pekerjaan latar beserta pemantauan progres, pengulangan, dan pelaporan kegagalannya.

**Alternatif yang ditolak.** *Membuat seluruh PDF sekelas pada saat finalisasi.* Menghasilkan tiga puluh pekerjaan render dalam satu tindakan, yang menuntut pemrosesan latar dan tampilan progres. Kompleksitas ini dibayar tanpa manfaat produk, karena rapor tidak selalu diunduh seluruhnya.

**Konsekuensi yang diterima.** Unduhan pertama memerlukan waktu render. pdfmake dipilih karena murni JavaScript dan menjaga ukuran image tetap kecil; apabila format rapor sekolah (V5) menuntut tata letak yang tidak dapat dicapai pdfmake, keputusan ini ditinjau ulang lewat entri baru.

### CK-10 · 6 Agustus 2026 · CloudFront VPC Origin dengan ALB privat

**Diputuskan.** ALB ditempatkan di subnet privat dan hanya dapat dihubungi CloudFront melalui VPC Origin.

**Alasan.** CloudFront menjadi satu-satunya pintu masuk sistem, dan ALB tidak pernah terekspos ke internet. Untuk sistem berisi data akademik anak di bawah umur, penguatan ini berbiaya rendah. Fitur ini tersedia umum sejak 20 November 2024.

**Alternatif yang ditolak.** *ALB internet-facing dengan header rahasia dari CloudFront.* Bergantung pada kerahasiaan sebuah header, sedangkan ALB tetap dapat dihubungi siapa pun yang mengetahui alamatnya.

### CK-11 · 6 Agustus 2026 · Drizzle

**Diputuskan.** Drizzle sebagai ORM.

**Alasan.** Skema, migrasi, dan query ditulis dekat dengan SQL, sehingga partial index, composite foreign key, dan `ON DELETE CASCADE` yang menegakkan invarian RFC-001 dapat dinyatakan langsung. Migrasi berupa berkas SQL yang dapat dibaca dan ditinjau.

**Catatan.** Pada rancangan 2 Agustus 2026, Drizzle dipilih karena ukuran bundel dan cold start di Lambda. Alasan itu **berlaku kembali** setelah CK-13, tetapi tidak lagi menjadi alasan utamanya: yang menentukan adalah kedekatannya dengan SQL sebagaimana diuraikan di atas.

### CK-12 · 6 Agustus 2026 · Tanpa Ansible

**Diputuskan.** Seluruh infrastruktur dikelola Terraform.

**Alasan.** NAT instance adalah satu-satunya host di dalam sistem, dan konfigurasinya cukup ditangani user data. Tidak ada armada server untuk dikelola. Apabila kelak sistem dipasang di server sekolah, pengelolaan konfigurasi host di sana adalah pekerjaan yang berbeda dan ditetapkan tersendiri.

### CK-13 · 6 Agustus 2026 · Lambda Web Adapter dan Function URL — mengamandemen CK-01 dan CK-02

**Diputuskan.** Backend berjalan di AWS Lambda dari **container image** memakai **Lambda Web Adapter**, diakses lewat **Function URL** dengan CloudFront Origin Access Control. ECS Fargate dan Application Load Balancer **tidak dipakai**, dan tetap tercatat sebagai jalur naik.

**Yang berubah dari CK-01 dan CK-02.** Keduanya menolak Lambda dengan tiga alasan. Ketiganya ditinjau ulang, dan hanya satu yang bertahan:

| Alasan penolakan pada CK-01 dan CK-02 | Status setelah ditinjau |
|---|---|
| Express memerlukan `serverless-http`, sehingga tidak berjalan apa adanya | **Gugur.** Lambda Web Adapter menerjemahkan event menjadi request HTTP ke aplikasi yang mendengarkan di port 8080. Tidak ada adapter di dalam kode aplikasi, dan `entry/` tetap satu berkas |
| Batas 29 detik API Gateway tidak dapat dinaikkan | **Gugur dua kali.** Pertama, sejak Juni 2024 batas tersebut dapat dinaikkan untuk REST API regional dan privat, dengan konsekuensi penurunan account-level throttle quota. Kedua, Function URL tidak memakai API Gateway sama sekali dan berbatas 15 menit. Ditambah lagi, CK-09 sudah menghapus satu-satunya pekerjaan panjang yang ada |
| Portabilitas menuntut entry point terpisah dan penukaran adapter antrean | **Gugur.** Artefaknya adalah image Docker yang sama, yang oleh AWS dinyatakan dapat dijalankan di Lambda, EC2, Fargate, dan komputer lokal. Di luar Lambda, `AWS_LAMBDA_EXEC_WRAPPER` tidak ada sehingga adapter tidak pernah dipanggil |
| Satu instance melayani satu request, sehingga pool wajib `max: 1` | **Bertahan.** Ini konsekuensi yang diterima, ditangani dengan reserved concurrency 40 terhadap plafon ~106 koneksi (§6.3) |

**Alasan.** Setelah tiga dari empat keberatan gugur, yang tersisa adalah selisih biaya yang besar untuk manfaat yang tidak lagi ada: Fargate dengan dua task di belakang ALB berbiaya **$60–71 per bulan**, sedangkan susunan ini **$27–35** — dan compute di dalamnya hanya ~$1. Membayar ~$36 per bulan untuk penyeimbang beban terhadap dua container yang melayani 379 pengguna tidak sepadan.

**Alternatif yang ditolak.**

*ECS Fargate satu task tanpa redundansi AZ.* Menekan biaya menjadi $45–55. Ditolak karena masih membayar ALB penuh sambil mengorbankan ketersediaan — kombinasi terburuk dari kedua pilihan.

*EC2 dengan Docker, tanpa ALB.* Sekitar $33–40. Ditolak karena mengembalikan sistem operasi yang harus dipatch, tanpa rolling deploy dan tanpa penskalaan, demi penghematan yang lebih kecil daripada susunan ini.

*API Gateway di depan Lambda.* Ditolak karena TLS, domain kustom, dan titik pemasangan WAF sudah disediakan CloudFront, sedangkan pembatasan laju dan validasi request sengaja ditempatkan di dalam Express agar ikut berpindah ke on-prem (§6.4). HTTP API juga justru lebih ketat, terkunci di 30 detik. Akan ditinjau ulang apabila kelak ada konsumen API di luar frontend sendiri — hal yang saat ini dicoret NG2.

**Konsekuensi yang diterima.**

1. **Cold start ~0,8–1,5 detik** pada request pertama setelah masa senggang. Dicatat sebagai butir 8 pada §16 untuk dibuktikan saat UAT.
2. **Pool `max: 1`** beserta disiplin reserved concurrency.
3. **Ketergantungan pada proyek `awslabs/aws-lambda-web-adapter`**, yang merupakan open source milik AWS dan bukan layanan berdukungan formal. Versi image adapter **wajib dipatok** — variabel lingkungan tanpa prefiks `AWS_LWA_` sudah usang dan akan dihapus pada versi 2.0.
4. **Perilaku penandatanganan OAC atas request ber-body wajib dibuktikan** sejak hari pertama infrastruktur naik (§11).

**Yang membuat keputusan ini dapat dibalik.** Ketiga konsekuensi pertama diselesaikan dengan berpindah ke ECS Fargate memakai **image yang sama persis**: yang berubah hanya modul Terraform, ditambah `max: 1` menjadi `max: 10` yang dibaca dari variabel lingkungan. Nol perubahan kode aplikasi. Inilah yang membedakannya dari rancangan Lambda 2 Agustus 2026, yang mengikat kode ke Lambda lewat `serverless-http` dan entry point terpisah.

### CK-14 · 6 Agustus 2026 · Elice AI Cloud, bukan OpenRouter — mengamandemen CK-06

**Diputuskan.** Tombol Suggestion dilayani **Elice AI Cloud** melalui program KADA, dengan antarmuka setara OpenAI (`POST /v1/chat/completions`, otorisasi `Bearer`).

**Alasan.**

1. **Kredit sudah tersedia** melalui program KADA, sehingga biaya AI keluar dari tagihan berulang dan menjadi kredit terbatas yang perlu dijaga pemakaiannya.
2. **Antarmukanya setara OpenAI.** Ini yang menentukan, bukan penyedianya. Adapter yang ditulis adalah klien OpenAI-compatible biasa, sehingga tuntutan portabilitas pada CK-06 justru terpenuhi lebih baik daripada dengan OpenRouter: berpindah ke penyedia lain maupun ke model yang dipasang sendiri di server sekolah cukup dengan mengubah base URL dan kunci API.

**Yang tidak berubah dari CK-06.** Penolakan terhadap **Amazon Bedrock** tetap berlaku, dengan alasan yang sama: Bedrock mengikat jalur AI ke AWS sehingga pemasangan on-prem memerlukan penyedia lain yang keluarannya belum tentu setara. Interface `AiAdvisor` di `ports/` juga tetap, sehingga lapisan domain dan rute tidak mengetahui penyedia mana yang dipakai.

**Konsekuensi yang diterima.**

1. **Kredit terbatas dan terikat program.** Habisnya kredit tidak menghentikan aplikasi, karena kegagalan layanan AI ditangani sebagai kegagalan lunak (AC-21).
2. **Data akademik dikirim ke layanan pihak ketiga.** Dimitigasi dengan tidak pernah mengirim identitas siswa (§9.2), dan tetap memerlukan persetujuan sekolah sesuai V6.
3. **Jalur keluar internet tetap dibutuhkan**, sehingga NAT instance tidak dapat dihapus kecuali IPv6 terbukti bekerja (§16 butir 6).

### CK-15 · 6 Agustus 2026 · Skrip pemasangan tunggal untuk on-prem — melengkapi CK-12

**Diputuskan.** Pemasangan di server sekolah dilakukan lewat satu skrip `install.sh` yang idempoten, dikirim bersama repositori. Ansible tetap tidak dipakai.

**Alasan.** CK-12 melepas Ansible karena tidak ada host untuk dikelola di sisi AWS. Rencana pemasangan on-prem memunculkan host yang sungguhan, sehingga keputusan itu ditimbang ulang — dan tetap bertahan. Nilai Ansible berasal dari pengulangan atas banyak mesin, sedangkan pilot ini satu server yang dipasang satu kali. Skrip juga tidak menuntut Python maupun mesin kontrol tersendiri, dapat dibaca staf TI sekolah sebagai perintah Linux biasa, dan dapat dijalankan tanpa internet dari media penyimpanan.

**Cakupan skrip.** Pemeriksaan prasyarat, pemasangan Docker, penyusunan `.env` beserta kata sandi acak, `docker compose up`, migrasi, pembuatan akun Administrator, pemasangan timer pencadangan beserta salinannya ke luar server, penyalaan pembaruan keamanan otomatis, dan verifikasi akhir. Rinciannya berada pada [DEPLOYMENT.md](DEPLOYMENT.md).

**Konsekuensi yang diterima.** Idempotensi harus ditulis sendiri melalui pemeriksaan "apakah sudah ada" di setiap langkah, sedangkan Ansible menyediakannya secara bawaan. Deteksi penyimpangan konfigurasi juga tidak tersedia. Keduanya dapat diterima untuk satu server. Apabila pemasangan on-prem kelak menjadi banyak — misalnya sepuluh sekolah — keputusan ini ditinjau ulang lewat entri baru.

**Yang paling mudah terlupakan.** Pencadangan basis data beserta salinannya ke luar server. Tanpa itu, kerusakan disk berarti hilangnya nilai satu semester. Butir ini diperlakukan sebagai bagian wajib skrip, bukan langkah opsional.

---

### CK-16 · 7 Agustus 2026 · Region berpindah ke Jakarta — mengamandemen §2

**Diputuskan.** Seluruh sumber daya ditempatkan di **`ap-southeast-3` (Jakarta)**, menggantikan `ap-southeast-1` (Singapura). Sertifikat ACM untuk CloudFront tetap wajib di `us-east-1`, dan CloudFront sendiri bersifat global.

**Alasan.**

1. **Kedudukan data.** EduTrack menyimpan data akademik **anak di bawah umur**. Menempatkannya di dalam negeri menjawab persoalan lokasi data alih-alih menundanya. Ini menyentuh **V6** pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md), yang masih menunggu validasi sekolah mengenai kebijakan privasi dan penyimpanan — dan region Jakarta membuat butir itu lebih mudah dijawab, bukan lebih sulit.
2. **Latensi.** Seluruh pengguna berada di Indonesia. Singapura sekitar 30–50 milidetik, sedangkan in-region satu digit milidetik.

**Yang diperiksa sebelum diputuskan.** `db.t4g.micro` beserta penyimpanan gp3, dan `t4g.nano` untuk NAT instance, **seluruhnya diterima AWS Pricing Calculator pada `ap-southeast-3`** — sehingga ketersediaannya di Jakarta bukan lagi dugaan. Perkiraan pembandingnya tersimpan pada https://calculator.aws/#/estimate?id=5ec85f81287d121b6209e62fa01954076ba236a4.

**Yang belum diperiksa.** Besaran biayanya. MCP `aws-calculator` gagal menghitung karena cacat pada peramban headless-nya, dan angka §8.2 karenanya tetap belum terverifikasi — keadaan yang sudah berlaku sebelum keputusan ini, kini ditambah region yang berubah.

**Alternatif yang ditolak.** *Bertahan di `ap-southeast-1`.* Lebih murah, lebih matang, dan seluruh perkiraan biaya sudah disusun untuknya. Ditolak karena tidak menjawab kedudukan data, dan karena selisih biaya pada skala 379 pengguna berukuran beberapa dolar per bulan — tidak sebanding dengan pertanyaan yang ditinggalkannya terbuka.

**Konsekuensi yang diterima.**

1. **Biaya naik dengan besaran yang belum diketahui.** Region baru lazimnya lebih mahal.
2. **Region yang jauh lebih muda**, sehingga layanan baru kerap datang belakangan. Seluruh layanan yang dipakai EduTrack sudah tersedia di sana.
3. **Kelayakan free tier RDS perlu diperiksa ulang** untuk region ini, dan menjadi butir tersendiri pada §9.

---

### CK-17 · 8 Agustus 2026 · Cloudflare menjadi DNS untuk kedua lingkungan; on-prem dijangkau lewat Tunnel

**Diputuskan.** Satu domain dikelola **Cloudflare** sebagai nameserver otoritatif, melayani AWS dan seluruh pemasangan on-prem sekaligus. Delegasi nameserver dilakukan **satu kali** di registrar; sesudahnya setiap lingkungan cukup menambah satu record.

| Nama | Tipe | Menunjuk ke | Proxy Cloudflare |
|---|---|---|---|
| `app.edutrack.sch.id` | CNAME | distribusi CloudFront | **mati** — DNS only |
| record validasi ACM | CNAME | nilai yang diminta ACM `us-east-1` | **mati** — DNS only |
| `<sekolah>.edutrack.sch.id` | CNAME | `<uuid>.cfargotunnel.com` | **hidup** — Proxied |

**Sisi AWS tidak berubah sama sekali.** CloudFront tetap satu-satunya pintu masuk, OAC dan `auth_type = AWS_IAM` tetap berlaku, dan TLS tetap berasal dari sertifikat ACM. Cloudflare hanya menjawab pertanyaan DNS lalu menyingkir dari jalur trafik.

**Sisi on-prem memakai Cloudflare Tunnel.** Proses `cloudflared` di server sekolah membuka koneksi keluar ke Cloudflare, dan trafik masuk mengalir balik lewat koneksi itu.

**Arah proxy berlawanan antara CloudFront dan Tunnel, dan itu bukan pilihan.** CloudFront wajib DNS only karena memproxy CDN di depan CDN hanya menambah lompatan. Tunnel wajib Proxied karena `cfargotunnel.com` tidak memiliki alamat publik dan hanya dapat dijangkau melalui jaringan Cloudflare — disetel DNS only, record itu tidak akan pernah resolve. Keliru menyeragamkan keduanya adalah kekeliruan yang paling mudah terjadi pada susunan ini.

**Alasan.**

1. **Satu titik kendali.** Dua lingkungan dengan infrastruktur yang sengaja berbeda ([ARCHITECTURE.md §13](ARCHITECTURE.md)) tetap dikelola dari satu zona DNS.
2. **Server sekolah tidak memerlukan IP publik statis maupun port masuk yang terbuka.** Koneksi dibuka dari dalam ke luar, sehingga pemasangan tidak bergantung pada kerja sama admin jaringan sekolah dan tidak menambah permukaan serangan di router.
3. **TLS on-prem tanpa mengurus sertifikat.** Diperlukan karena cookie sesi `HttpOnly` menuntut HTTPS, dan sekolah tidak memiliki staf untuk memperbarui sertifikat.

**Penamaan.** Setiap sekolah memperoleh **subdomain di bawah domain tim**, bukan domain miliknya sendiri, sehingga `install.sh` cukup meminta nama sekolah dan zona DNS tetap dipegang satu pihak.

**Alternatif yang ditolak.**

*Cloudflare memproxy CloudFront (awan oranye).* Dua CDN berturut-turut: latensi bertambah dan header `Host` berpotensi tidak dikenali distribusi. Manfaatnya nol karena CloudFront sudah CDN.

*Cloudflare menggantikan CloudFront seluruhnya.* Cloudflare tidak dapat menandatangani SigV4, sehingga Function URL harus turun ke `auth_type = NONE` dan bucket frontend harus meninggalkan OAC. Ini membatalkan ketetapan [ARCHITECTURE.md §7](ARCHITECTURE.md) bahwa tidak ada sumber daya yang dapat dihubungi langsung dari internet, dan memindahkan pembatasan akses dari infrastruktur ke dalam kode aplikasi.

*`cloudflared` dimasukkan ke dalam image dan dijalankan di Lambda.* `cloudflared` menuntut proses yang hidup terus-menerus untuk memelihara koneksi keluarnya, sedangkan Lambda membekukan seluruh execution environment di antara invocation. Trafik tunnel juga bukan invocation, sehingga tidak ada yang membangunkan fungsi ketika request datang. Tunnel dan scale-to-zero saling meniadakan; susunan ini baru mungkin bila compute berpindah ke ECS Fargate (CK-13).

*Setiap sekolah memakai domain miliknya sendiri.* Memberi sekolah kepemilikan penuh, tetapi menuntut tiap sekolah mengurus zona DNS dan token tunnelnya sendiri — bertentangan dengan **CK-15** yang menetapkan pemasangan dapat dijalankan staf TI sekolah dari satu skrip.

**Konsekuensi yang diterima.**

1. **Domain wajib dibeli sebelum pemasangan on-prem pertama.** Selama sistem hanya berjalan di AWS, nama bawaan CloudFront masih cukup dan pembelian dapat ditunda.
2. **Zona DNS menjadi milik tim, bukan sekolah.** Setiap pemasangan baru menuntut satu record dibuat tim lebih dahulu, sehingga `install.sh` tidak sepenuhnya mandiri.
3. **Satu pihak ketiga berada di jalur masuk on-prem.** Tunnel yang putus membuat sekolah tidak dapat dijangkau dari luar, meski jaringan lokalnya tetap berjalan.

### CK-18 · 11 Agustus 2026 · Tanpa RDS Proxy

**Diputuskan.** Fungsi Lambda menghubungi RDS secara langsung. **RDS Proxy tidak dipakai.** Perlindungan terhadap kehabisan koneksi tetap berupa `max: 1` pada pool dan reserved concurrency 40 ([ARCHITECTURE §6](ARCHITECTURE.md)).

**Alasan.** RDS Proxy adalah jawaban baku bagi persoalan Lambda dan RDS, dan persoalannya nyata — tetapi tidak pada volume ini. Kriteria AWS sendiri dihadapkan pada angka EduTrack:

| Kriteria kandidat menurut AWS | Keadaan EduTrack |
|---|---|
| Menemui galat *too many connections* | Plafon 40 koneksi terhadap ~106 tersedia; tidak pernah mendekat |
| Kelas kecil T2/T3 **saat menangani koneksi dalam jumlah besar** | Concurrency 0,08 rata-rata; ~1,7 pada lonjakan dua puluh kali lipat |
| Fungsi Lambda dengan koneksi pendek yang sering | Benar sebagian — tetapi pada 0,7 request/detik container tetap hangat dan koneksinya dipakai ulang |
| Failover Multi-AZ hingga 66% lebih cepat | **Tidak berlaku.** Susunan ini Single-AZ |

**Biayanya melampaui yang dilindunginya.** $0,018 per vCPU-jam di `ap-southeast-3` × 2 vCPU × 730 jam = **$26,28 per bulan**, sedangkan RDS yang dilindunginya berbiaya $21,01. Tagihan naik 81% (§8.2) untuk menutup keadaan yang belum pernah terjadi.

**Dua hal yang menguranginya lebih jauh.** Jalur terberat sistem ini — finalisasi sekelas dan Simpan Nilai yang mengunci baris rapor — berjalan di dalam transaksi, dan selama transaksi berlangsung koneksinya tidak dapat dibagi. Multiplexing berkurang tepat di tempat ia paling diinginkan. Selain itu setiap query memperoleh satu lompatan jaringan tambahan.

**Alternatif yang ditolak.** *Memasang RDS Proxy sekarang sebagai pencegahan.* Ditolak dengan alasan di atas. *Menaikkan reserved concurrency tanpa proxy.* Ditolak karena justru menghapus rem yang membuat proxy tidak diperlukan.

**Tiga pemicu peninjauan ulang.** Keputusan ini **bukan "tidak pernah"**, melainkan "belum". Ia ditinjau ulang apabila salah satu terjadi:

1. Reserved concurrency perlu melampaui ~90, yaitu trafik tumbuh sekitar lima puluh kali lipat atau satu sistem melayani banyak sekolah.
2. Basis data berpindah ke Multi-AZ — sejak saat itu failover cepat menjadi manfaat yang nyata.
3. CloudWatch menunjukkan `DatabaseConnections` merayap naik atau `ConnectionAttempts` melonjak.

Sebelum ketiganya, jalur naik yang lebih murah adalah menaikkan kelas instance: `db.t4g.micro` → `db.t4g.small` memberi ~212 koneksi dengan biaya di bawah harga proxy.

### CK-19 · 11 Agustus 2026 · Akun bertahan pada Free Plan; rancangan disesuaikan agar seluruhnya eligible — mengamandemen §4.1 dan §8.2

**Diputuskan.** Akun AWS tetap pada **Free Plan** dan tidak dinaikkan ke Paid Plan. Dua tetapan disesuaikan supaya seluruh sumber daya masuk daftar free tier `ap-southeast-3`:

| Tetapan | Semula | Menjadi |
|---|---|---|
| NAT instance | `t4g.nano` | **`t4g.micro`** |
| Retensi cadangan otomatis RDS | 7 hari | **1 hari** — 0 apabila 1 pun ditolak |

Cadangan yang sesungguhnya berpindah ke **snapshot manual** yang dijalankan manusia sebelum tindakan berisiko.

**Alasan.** Free Plan bukan potongan harga, melainkan pagar: pembuatan sumber daya di luar daftar free tier **ditolak API**, bukan ditagih. Dua penolakan pada `terraform apply` pertama membuktikannya — `FreeTierRestrictionError` pada retensi cadangan, dan `InvalidParameterCombination` pada tipe instance.

Yang menentukan bentuk penyesuaiannya adalah daftar tipe yang eligible di region ini: `c7i-flex.large`, `t4g.small`, **`t4g.micro`**, `t3.micro`, `t3.small`, `m7i-flex.large`. Keberadaan `t4g.micro` di dalamnya menghapus seluruh perubahan yang semula diperkirakan perlu — **arsitektur arm64 tetap**, AMI tidak berganti, dan tidak ada satu baris kode pun yang berubah selain nilai bawaan dua variabel.

**Kelayakannya ternyata luas, dan itu berlawanan dengan dugaan awal.** Bukti dari `apply` yang sama: VPC beserta seluruh isinya, gateway endpoint S3, kedua bucket, role IAM, log group, dan **wadah Secrets Manager** semuanya lolos — padahal Secrets Manager bukan layanan free tier. Free Plan karenanya hanya memblokir hal tertentu dan menggerus kredit untuk sisanya. **Tidak ada satu layanan pun pada rancangan ini yang menuntut akun berbayar.**

**Alternatif yang ditolak.**

*Naik ke Paid Plan.* Membuat rancangan berjalan apa adanya tanpa satu perubahan pun, dan kredit yang ada umumnya ikut terbawa. Ditolak pemilik akun — keputusan uang, bukan keputusan teknis, dan bukan milik dokumen ini.

*Pindah ke `t3.micro`.* Juga eligible, tetapi x86 — menuntut AMI berganti arsitektur dan meninggalkan arm64 tanpa satu pun alasan teknis, karena `t4g.micro` sudah eligible.

*Menurunkan retensi langsung ke 0.* Pasti diterima, tetapi menghapus point-in-time recovery seluruhnya. Galat AWS menyebut "melampaui maksimum" tanpa menyebut angkanya, sehingga 1 patut dicoba lebih dahulu; kegagalannya murah dan muncul seketika.

*NAT Gateway terkelola.* Menghapus persoalan tipe instance, tetapi berbiaya sekitar $30 lebih mahal per bulan dan tidak termasuk free tier sama sekali.

**Konsekuensi yang diterima.**

1. **Jendela pemulihan menyusut dari 7 hari menjadi 1.** Kekeliruan yang baru disadari lusa tidak lagi dapat dikembalikan otomatis. Ini menaikkan bobot **V6** pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md), yang masih terbuka, dari "menentukan angka retensi" menjadi "menentukan apakah angka ini memadai bagi data sekolah".
2. **Cadangan bersandar pada disiplin manusia**, yaitu snapshot manual sebelum migrasi maupun koreksi massal. Snapshot manual tidak dibatasi retensi otomatis dan ikut kuota 20 GB penyimpanan cadangan free tier.
3. **NAT menjadi dua kali lebih mahal sesudah bulan ke-12** — `t4g.micro` kira-kira dua kali tarif `t4g.nano`. Selama 12 bulan pertama selisihnya nol, karena 750 jam per bulan sudah menutup satu instance yang menyala terus.
4. **Bulan ke-13 melompat sekaligus.** Cakupan 12 bulan berakhir bersamaan untuk RDS, EC2, dan alamat IPv4, sehingga tagihan berpindah dari nol ke angka penuh §8.2 dalam satu siklus.

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 11 Agustus 2026 | **CK-19** — akun bertahan pada **Free Plan**, dan rancangan disesuaikan agar seluruh sumber daya eligible: NAT `t4g.nano` → `t4g.micro`, retensi cadangan 7 → 1 hari. Butir 7 pada §9 **terjawab**, tetapi bukan sebagaimana ditanyakan: Free Plan membatasi apa yang boleh dibuat, bukan memberi potongan. §4.1 dan §8.2 disesuaikan. Dicatat bahwa **tidak ada satu layanan pun** pada rancangan ini yang menuntut akun berbayar |
| 11 Agustus 2026 | Butir 1 pada §9 **ditutup** — teks prompt sistem sudah diuji terhadap Gemini 3.6 Flash dan lulus AC-18 beserta AC-31 |
| 11 Agustus 2026 | §6 merujuk **[payload.md](payload.md)** sebagai kontrak permintaan yang sudah diverifikasi terhadap endpoint, beserta tiga ketentuan yang mengikat adapter: `max_tokens` minimal 2000, `finish_reason: "length"` sebagai kegagalan, dan dua bentuk galat |
| 11 Agustus 2026 | §6 — model ditetapkan **Gemini 3.6 Flash** lewat endpoint khusus Elice, bahasa keluaran ditetapkan **Bahasa Indonesia**, dan bentuk endpointnya diperjelas: satu endpoint satu model. Butir 1 pada §9 menyempit menjadi teks prompt sistem saja |
| 11 Agustus 2026 | **CK-18** — RDS Proxy ditolak beserta tiga pemicu peninjauan ulangnya. Ditambahkan ke daftar "yang sengaja tidak dipakai" pada §2 |
| 11 Agustus 2026 | Butir 2 pada §9 ditutup — format rapor resmi sekolah terjawab, dan bentuknya ditetapkan [ARCHITECTURE §11.3](ARCHITECTURE.md) beserta CK-A-10. Tata letaknya sederhana, sehingga peninjauan ulang CK-09 yang dikhawatirkan butir itu tidak diperlukan |
| 11 Agustus 2026 | §8.2 dan §8.3 ditulis ulang dengan tarif `ap-southeast-3` yang **diverifikasi terhadap AWS Price List API**, menggantikan perkiraan `ap-southeast-1` yang belum pernah diuji. Total $32,41 per bulan, atau $11,40 dengan free tier RDS. Butir 9 pada §9 ditutup |
| 6 Agustus 2026 | Dokumen dibuat. Menetapkan stack di atas PRD v3.0 dan RFC-001. Menggantikan bagian stack pada `ARCHITECTURE.md` versi 2 Agustus 2026. Menutup K-01 dan K-02 pada RFC-001 §9, serta menutup temuan T-03 |
| 6 Agustus 2026 | Compute berpindah dari ECS Fargate dengan ALB ke Lambda Web Adapter dengan Function URL (**CK-13**, mengamandemen CK-01 dan CK-02). Perkiraan biaya diperbaiki: NAT menjadi ~$8 karena alamat IPv4 publik kini ditagih, dan total turun menjadi $27–35 per bulan |
| 6 Agustus 2026 | Penyedia AI berpindah dari OpenRouter ke Elice AI Cloud melalui program KADA (**CK-14**, mengamandemen CK-06). Ditetapkan bahwa identitas siswa tidak pernah dikirim ke layanan AI |
| 6 Agustus 2026 | Penyimpanan rahasia ditetapkan pada **§7** yang baru: kredensial `app_rw` dan `app_ro` di Secrets Manager, kunci API Elice di SSM Parameter Store, dan ketiganya di berkas `.env` berizin `600` pada on-prem. Nilai rahasia dibuat di luar Terraform. Pasal biaya dan pasal keputusan terbuka bergeser menjadi §8 dan §9; total menjadi $28–36 per bulan |
| 6 Agustus 2026 | **Versi 2.0 — dokumen dipecah tiga.** Isi yang menjelaskan hubungan antar bagian dipindahkan ke [ARCHITECTURE.md](ARCHITECTURE.md), dan isi yang menjelaskan penerapan serta operasional dipindahkan ke [DEPLOYMENT.md](DEPLOYMENT.md). Dokumen ini menyusut menjadi pilihan teknologi beserta alasannya. Ditambahkan **CK-15** yang menetapkan skrip pemasangan tunggal untuk on-prem, melengkapi CK-12. Butir §16 mengenai sisa kredit KADA dan persetujuan sekolah dihapus atas keputusan tim; butir mengenai bentuk endpoint Elice ditutup dan dipindahkan ke §6 |
| 6 Agustus 2026 | Waktu render berkas rapor pada §2 disesuaikan mengikuti **CK-A-07** pada [ARCHITECTURE.md](ARCHITECTURE.md), yang mengamandemen **CK-09**. Amandemennya ditulis di sana, bukan di sini, karena isi yang dirujuk CK-09 sudah berpindah ke `ARCHITECTURE.md` pada pemecahan 6 Agustus 2026 |
| 7 Agustus 2026 | **§7 menjadi empat rahasia.** Ditambahkan kredensial `edutrack_owner` dengan perlakuan sama seperti `app_rw`, beserta kolom **dibaca oleh** yang menyatakan pembatasan IAM per rahasia. Menutup temuan **S-03** pada [SCHEMA.md](SCHEMA.md) §12. Dicatat pula bahwa pemisahan pembaca tidak tersedia di on-prem |
| 7 Agustus 2026 | **Region berpindah dari `ap-southeast-1` ke `ap-southeast-3` (Jakarta)** — **CK-16**, mengamandemen §2. Didorong kedudukan data akademik anak di bawah umur dan latensi pengguna. Ketersediaan `db.t4g.micro` dan `t4g.nano` di Jakarta terbukti lewat AWS Pricing Calculator; besaran biayanya belum. Peringatan §8.2 diperluas: angkanya kini salah region **dan** belum pernah diverifikasi |
| 8 Agustus 2026 | **Nama domain dan DNS ditetapkan pada CK-17.** Satu domain dikelola Cloudflare sebagai nameserver otoritatif untuk kedua lingkungan: CNAME tanpa proxy ke CloudFront di sisi AWS, dan CNAME ber-proxy ke Cloudflare Tunnel di sisi on-prem, dengan subdomain per sekolah di bawah domain tim. Susunan AWS tidak berubah — OAC, `auth_type = AWS_IAM`, dan sertifikat ACM tetap berlaku. Dicatat pula tiga alternatif yang ditolak, termasuk menjalankan `cloudflared` di dalam Lambda |
| 8 Agustus 2026 | §9 butir 4 disesuaikan mengikuti CK-17. Yang belum diputuskan bukan lagi bentuk DNS-nya melainkan **nama domainnya**, dan pembeliannya. Dicatat pula bahwa penerapannya **dikerjakan paling akhir dengan sengaja**: seluruh susunan CK-17 dapat ditulis dan ditinjau tanpa domain, sehingga penundaan ini tidak menahan satu pun pekerjaan lain |
