# Glosarium Teknis — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v1.0 |
| **Tanggal** | 7 Agustus 2026 |
| **Kedudukan** | Menjelaskan setiap singkatan dan istilah teknis yang dipakai di seluruh dokumen. **Tidak menetapkan apa pun** |
| **Padanannya** | [PRD.md Lampiran A](PRD.md) memuat glosarium **produk** — Wali Kelas, Finalisasi, KKM, dan sejenisnya. Dokumen ini memuat yang **teknis** |

> **Aturan yang berlaku sejak 7 Agustus 2026:** setiap singkatan dijelaskan pada pemakaian pertama di dokumen mana pun, **dan** terdaftar di sini. Singkatan yang muncul tanpa penjelasan adalah cacat dokumen, bukan pengetahuan yang diandaikan.

---

## 1. Awalan rujukan antar dokumen

Dipakai di seluruh dokumen untuk menunjuk satu butir tertentu.

| Awalan | Kepanjangan | Ada di |
|---|---|---|
| `P` | Aturan **P**roduk | [ATURAN-DAN-KRITERIA §1](ATURAN-DAN-KRITERIA.md) |
| `UC` | **U**se **C**ase — kegiatan yang dilakukan pengguna | [ATURAN-DAN-KRITERIA §2](ATURAN-DAN-KRITERIA.md) |
| `AC` | **A**cceptance **C**riteria — kriteria aplikasi dianggap siap | [ATURAN-DAN-KRITERIA §4](ATURAN-DAN-KRITERIA.md) |
| `V` | Butir **V**alidasi yang harus ditanyakan ke sekolah | [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md) |
| `NG` | **N**ot in **G**oal — di luar cakupan MVP | [PRD §4.2](PRD.md) |
| `M` | Cakupan yang **M**asuk MVP | [PRD §4.1](PRD.md) |
| `D` | **D**ecision — keputusan model data | [RFC-001 §1](RFC-001-model-data-konseptual.md) |
| `I` | **I**nvarian — pernyataan yang harus selalu benar | [RFC-001 §6](RFC-001-model-data-konseptual.md) |
| `C` | **C**onstraint — kendala bagi dokumen arsitektur | [RFC-001 §8](RFC-001-model-data-konseptual.md) |
| `K` | **K**eputusan terbuka | [RFC-001 §9](RFC-001-model-data-konseptual.md) |
| `T` | **T**emuan terhadap PRD | [RFC-001 §10](RFC-001-model-data-konseptual.md) |
| `S` | Temuan dari **S**kema fisik | [SCHEMA §12](SCHEMA.md) |
| `A` | Temuan dari **A**PI | [API §13](API.md) |
| `CK` | **C**atatan **K**eputusan | Lampiran tiap dokumen |
| `CK-A`, `CK-D`, `CK-S`, `CK-API` | Catatan Keputusan milik ARCHITECTURE, DEPLOYMENT, SCHEMA, API | Lampiran masing-masing |
| `RFC` | **R**equest **F**or **C**omments — usulan yang dibahas sebelum ditetapkan | Nama dokumen |
| `PRD` | **P**roduct **R**equirements **D**ocument | Nama dokumen |

---

## 2. Amazon Web Services

| Singkatan | Kepanjangan | Artinya di sini |
|---|---|---|
| **AWS** | **A**mazon **W**eb **S**ervices | Penyedia layanan komputasi awan yang dipakai EduTrack |
| **ARN** | **A**mazon **R**esource **N**ame | Alamat unik setiap sumber daya AWS. Lihat §2.1 |
| **IAM** | **I**dentity and **A**ccess **M**anagement | Layanan AWS yang mengatur siapa boleh memanggil apa |
| **MFA** | **M**ulti-**F**actor **A**uthentication | Lapis kedua saat masuk, berupa kode dari ponsel |
| **OIDC** | **O**pen **ID** **C**onnect | Cara pihak luar membuktikan identitas tanpa menyimpan kunci. Lihat [RUNBOOK-OIDC.md](RUNBOOK-OIDC.md) |
| **VPC** | **V**irtual **P**rivate **C**loud | Jaringan pribadi di dalam AWS. RDS dan Lambda berada di dalamnya |
| **RDS** | **R**elational **D**atabase **S**ervice | Layanan basis data terkelola. Menjalankan PostgreSQL milik EduTrack |
| **S3** | **S**imple **S**torage **S**ervice | Penyimpanan berkas. Menyimpan hasil build frontend dan berkas rapor |
| **ECR** | **E**lastic **C**ontainer **R**egistry | Gudang penyimpanan container image |
| **ECS** | **E**lastic **C**ontainer **S**ervice | Layanan menjalankan container. **Tidak dipakai** (CK-13), tercatat sebagai jalur naik |
| **ALB** | **A**pplication **L**oad **B**alancer | Pembagi trafik. **Tidak dipakai** (CK-13) |
| **SQS** | **S**imple **Q**ueue **S**ervice | Antrean pesan. **Tidak dipakai** (CK-07) |
| **NAT** | **N**etwork **A**ddress **T**ranslation | Perantara agar sumber daya di jaringan pribadi bisa menghubungi internet |
| **IGW** | **I**nternet **G**ate**w**ay | Pintu keluar internet. Bentuk *egress-only* atas IPv6 dipertimbangkan menggantikan NAT |
| **ACM** | **AWS** **C**ertificate **M**anager | Penerbit sertifikat TLS. Milik CloudFront **wajib** di region `us-east-1` |
| **OAC** | **O**rigin **A**ccess **C**ontrol | Mekanisme agar hanya CloudFront yang boleh menghubungi sumber di belakangnya |
| **SSM** | **S**ystems **M**anager | Layanan AWS. Dipakai untuk Parameter Store dan akses NAT instance tanpa kunci SSH |
| **KMS** | **K**ey **M**anagement **S**ervice | Pengelola kunci enkripsi. Diperlukan membaca rahasia bertipe SecureString |
| **ENI** | **E**lastic **N**etwork **I**nterface | Kartu jaringan maya. Lambda di dalam VPC membutuhkannya |
| **AZ** | **A**vailability **Z**one | Pusat data terpisah dalam satu region |
| **EBS** | **E**lastic **B**lock **S**tore | Cakram penyimpanan untuk EC2 |
| **EC2** | **E**lastic **C**ompute **C**loud | Mesin maya. Dipakai hanya untuk NAT instance |
| **SDK** | **S**oftware **D**evelopment **K**it | Pustaka pemrograman. SDK AWS hanya boleh muncul di `adapters/aws/` |
| **CLI** | **C**ommand **L**ine **I**nterface | Antarmuka baris perintah |
| **LWA** | **L**ambda **W**eb **A**dapter | Binary yang menerjemahkan panggilan Lambda menjadi request HTTP biasa |
| **SigV4** | **Sig**nature **V**ersion **4** | Cara AWS menandatangani request agar keasliannya terbukti |
| **DLQ** | **D**ead **L**etter **Q**ueue | Tempat pesan yang gagal diproses. **Tidak dipakai** |
| **WAF** | **W**eb **A**pplication **F**irewall | Penyaring trafik berbahaya |
| **AROA** | — | Awalan pengenal internal AWS untuk sebuah role. Muncul pada keluaran `sts get-caller-identity` |

### 2.1 Bentuk ARN

```
arn:aws:iam::274286556151:role/edutrack-terraform
 │   │   │  │        │            │
 │   │   │  │        │            └─ jenis dan nama sumber daya
 │   │   │  │        └─ pengenal akun
 │   │   │  └─ region — dikosongkan untuk layanan global seperti IAM
 │   │   └─ layanan
 │   └─ partisi — hampir selalu `aws`
 └─ awalan tetap
```

Setiap policy menyebut sumber daya lewat ARN-nya. Itulah cara "boleh membaca rahasia **ini saja**, bukan yang lain" dinyatakan secara tepat.

---

## 3. Web dan jaringan

| Singkatan | Kepanjangan | Artinya di sini |
|---|---|---|
| **API** | **A**pplication **P**rogramming **I**nterface | Kontrak yang dipanggil frontend. Ditetapkan [API.md](API.md) |
| **URL** | **U**niform **R**esource **L**ocator | Alamat sumber daya di internet |
| **URI** | **U**niform **R**esource **I**dentifier | Bentuk yang lebih umum dari URL |
| **HTTP** | **H**yper**t**ext **T**ransfer **P**rotocol | Aturan percakapan antara peramban dan server |
| **HTTPS** | HTTP **S**ecure | HTTP yang terenkripsi |
| **TLS** | **T**ransport **L**ayer **S**ecurity | Enkripsi yang membuat HTTPS aman |
| **CA** | **C**ertificate **A**uthority | Penerbit sertifikat yang dipercaya peramban |
| **DNS** | **D**omain **N**ame **S**ystem | Penerjemah nama domain menjadi alamat IP |
| **NS** | **N**ame **S**erver | Mesin yang menjawab pertanyaan DNS untuk sebuah domain. Didelegasikan ke Cloudflare, sekali saja di registrar (CK-17) |
| **CNAME** | **C**anonical **NAME** | Record DNS yang mengarahkan satu nama ke nama lain, bukan ke alamat IP |
| **Proxied** | — | Record CNAME yang trafiknya melewati jaringan Cloudflare. **Wajib** untuk Cloudflare Tunnel, **wajib mati** untuk CloudFront (CK-17) |
| **Cloudflare Tunnel** | — | Jalan masuk on-prem. Proses `cloudflared` membuka koneksi keluar ke Cloudflare, sehingga server sekolah tidak memerlukan IP publik maupun port terbuka (CK-17) |
| **IP** | **I**nternet **P**rotocol | Alamat numerik sebuah mesin di jaringan |
| **CORS** | **C**ross-**O**rigin **R**esource **S**haring | Aturan peramban saat halaman memanggil domain lain. **Hilang seluruhnya** karena frontend dan API satu domain |
| **CSRF** | **C**ross-**S**ite **R**equest **F**orgery | Serangan yang menumpang sesi pengguna dari situs lain |
| **XSS** | **C**ross-**S**ite **S**cripting | Serangan yang menyisipkan skrip ke halaman |
| **HSTS** | **HTTP** **S**trict **T**ransport **S**ecurity | Header yang memaksa peramban selalu memakai HTTPS |
| **CSP** | **C**ontent **S**ecurity **P**olicy | Header yang membatasi sumber skrip dan gaya yang boleh dimuat |
| **JWT** | **J**SON **W**eb **T**oken | Token yang isinya dapat dibaca tanpa basis data. **Tidak dipakai** (CK-A-04) |
| **JSON** | **J**ava**S**cript **O**bject **N**otation | Format pertukaran data yang dipakai seluruh endpoint |
| **REST** | **RE**presentational **S**tate **T**ransfer | Gaya perancangan API yang dipakai di sini |
| **SPA** | **S**ingle **P**age **A**pplication | Aplikasi web yang tidak memuat ulang halaman. Bentuk frontend EduTrack |
| **SSR** | **S**erver-**S**ide **R**endering | Perenderan halaman di server. **Tidak dipakai** (CK-03) |
| **SEO** | **S**earch **E**ngine **O**ptimization | Tidak relevan untuk aplikasi internal sekolah |

---

## 4. Basis data

| Singkatan | Kepanjangan | Artinya di sini |
|---|---|---|
| **SQL** | **S**tructured **Q**uery **L**anguage | Bahasa untuk berbicara dengan basis data |
| **DDL** | **D**ata **D**efinition **L**anguage | Bagian SQL yang **mengubah bentuk** — `CREATE`, `ALTER`, `DROP` |
| **ORM** | **O**bject-**R**elational **M**apping | Pustaka yang menjembatani kode dan basis data. Dipakai: Drizzle |
| **PK** | **P**rimary **K**ey | Kolom pengenal utama sebuah baris |
| **FK** | **F**oreign **K**ey | Kolom yang menunjuk baris di tabel lain |
| **UK** | **U**nique **K**ey | Kolom yang nilainya tidak boleh berulang |
| **UUID** | **U**niversally **U**nique **ID**entifier | Pengenal acak 128 bit. Dipakai sebagai seluruh pengenal (CK-S-01) |
| **JSONB** | **JSON** **B**inary | Tipe kolom PostgreSQL untuk menyimpan JSON secara efisien |
| **GIN** | **G**eneralized **IN**verted index | Jenis indeks untuk isi JSONB. **Tidak dipakai** |
| **ENUM** | **ENUM**eration | Tipe dengan daftar nilai tetap. **Tidak dipakai** (CK-S-02) |
| **CRUD** | **C**reate, **R**ead, **U**pdate, **D**elete | Empat operasi dasar terhadap data |
| **PITR** | **P**oint-**I**n-**T**ime **R**ecovery | Memulihkan basis data ke keadaan pada suatu saat |

---

## 5. Pengembangan dan penerapan

| Singkatan | Kepanjangan | Artinya di sini |
|---|---|---|
| **CI** | **C**ontinuous **I**ntegration | Pemeriksaan otomatis pada setiap perubahan kode |
| **CD** | **C**ontinuous **D**elivery | Penerapan otomatis setelah pemeriksaan lulus |
| **PR** | **P**ull **R**equest | Usulan perubahan kode yang ditinjau sebelum digabungkan |
| **SHA** | **S**ecure **H**ash **A**lgorithm | Sidik jari kriptografis. "git SHA" berarti pengenal unik satu commit |
| **TDD** | **T**est-**D**riven **D**evelopment | Menulis tes lebih dahulu, baru implementasinya |
| **E2E** | **E**nd-to-**E**nd | Pengujian yang menelusuri alur pengguna dari awal sampai akhir |
| **IaC** | **I**nfrastructure **a**s **C**ode | Infrastruktur ditulis sebagai berkas teks. Dipakai: Terraform |
| **LTS** | **L**ong-**T**erm **S**upport | Versi yang didukung lama. Node.js 24 LTS |
| **UAT** | **U**ser **A**cceptance **T**esting | Pengujian penerimaan oleh calon pengguna |
| **MVP** | **M**inimum **V**iable **P**roduct | Versi terkecil yang sudah dapat diuji dari awal sampai akhir |
| **UI** | **U**ser **I**nterface | Tampilan yang dilihat pengguna |
| **UX** | **U**ser **E**xperience | Rancangan pengalaman memakai aplikasi |
| **SSH** | **S**ecure **SH**ell | Cara masuk ke mesin dari jarak jauh. **Sengaja tidak dipakai** |
| **ECC** | — | Kumpulan aturan, agen, dan perintah yang dipakai agen. Lihat [AGENTS.md](AGENTS.md) |

---

## 6. Kinerja dan ukuran

| Singkatan | Kepanjangan | Sasaran |
|---|---|---|
| **CWV** | **C**ore **W**eb **V**itals | Kumpulan ukuran pengalaman memuat halaman |
| **LCP** | **L**argest **C**ontentful **P**aint | Waktu sampai bagian terbesar halaman tampil. Sasaran < 2,5 detik |
| **INP** | **I**nteraction to **N**ext **P**aint | Waktu halaman menanggapi tindakan. Sasaran < 200 milidetik |
| **CLS** | **C**umulative **L**ayout **S**hift | Seberapa banyak tata letak bergeser saat memuat. Sasaran < 0,1 |
| **FCP** | **F**irst **C**ontentful **P**aint | Waktu sampai isi pertama tampil |
| **TBT** | **T**otal **B**locking **T**ime | Lama halaman tidak menanggapi apa pun |
| **CPU** | **C**entral **P**rocessing **U**nit | Prosesor. Pada Lambda, porsinya ditentukan besar memori |
| **KB**, **MB**, **GB** | Kilo/Mega/Giga**byte** | Satuan ukuran berkas dan memori |

---

## 7. Format berkas

| Singkatan | Kepanjangan | Dipakai untuk |
|---|---|---|
| **CSV** | **C**omma-**S**eparated **V**alues | Unggah akun Guru dan Siswa; unduh kata sandi awal |
| **XLSX** | Excel Spreadsheet | Unggah daftar siswa per kelas |
| **PDF** | **P**ortable **D**ocument **F**ormat | Berkas rapor |
| **ZIP** | — | Arsip unduh rapor sekelas |
| **YAML** | **Y**AML **A**in't **M**arkup **L**anguage | Format berkas workflow GitHub Actions |
| **HCL** | **H**ashiCorp **C**onfiguration **L**anguage | Format berkas Terraform |
| **ISO 8601** | — | Standar penulisan waktu, misalnya `2026-08-07T14:30:00+07:00` |
| **UTC** | **C**oordinated **U**niversal **T**ime | Waktu acuan dunia. EduTrack memakai `Asia/Jakarta`, yaitu UTC+7 |

---

## 8. Produk dan pendidikan

Istilah produk selengkapnya pada [PRD.md Lampiran A](PRD.md). Yang berupa singkatan dikumpulkan di sini.

| Singkatan | Kepanjangan |
|---|---|
| **KKM** | **K**riteria **K**etuntasan **M**inimal — batas acuan ketuntasan, nilai awal 75 |
| **NIP** | **N**omor **I**nduk **P**egawai — pengenal masuk Guru |
| **NIS** | **N**omor **I**nduk **S**iswa — pengenal masuk Siswa |
| **UTS** | **U**jian **T**engah **S**emester — komponen penilaian, bobot 26 |
| **UAS** | **U**jian **A**khir **S**emester — komponen penilaian, bobot 26 |
| **T1**, **T2**, **T3** | Tugas — komponen penilaian, bobot 6 masing-masing |
| **U1**, **U2**, **U3** | Ulangan Harian — komponen penilaian, bobot 10 masing-masing |
| **RPS** | **R**encana **P**embelajaran **S**emester — di luar cakupan (NG6) |
| **SMA**, **SMK**, **SMP** | Jenjang sekolah menengah |
| **IPA**, **IPS** | Jurusan Ilmu Pengetahuan Alam dan Ilmu Pengetahuan Sosial |
| **X**, **XI**, **XII** | Tingkat kelas SMA, dalam angka Romawi: 10, 11, 12 |
| **KADA** | Program yang menyediakan kredit layanan AI Elice |

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 7 Agustus 2026 | Dokumen dibuat setelah ditemukan bahwa **ARN** dan puluhan singkatan lain dipakai di seluruh dokumen tanpa pernah dijelaskan. Ditetapkan pula aturan bahwa singkatan dijelaskan pada pemakaian pertama dan terdaftar di sini |
| 8 Agustus 2026 | §3 memperoleh empat istilah yang dibutuhkan **CK-17**: **NS**, **CNAME**, **Proxied**, dan **Cloudflare Tunnel**. Dua yang terakhir dicantumkan meski bukan singkatan, karena arah proxy yang berlawanan antara CloudFront dan Tunnel adalah sumber kekeliruan yang paling mudah terjadi |
