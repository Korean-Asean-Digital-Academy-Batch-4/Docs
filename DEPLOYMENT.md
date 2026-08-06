# Penerapan dan Operasional — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v0.3 — **Pasal 2, 3, 6, dan 9 berisi; sisanya masih rancangan** |
| **Tanggal** | 7 Agustus 2026 |
| **Disusun oleh** | Re:Code |
| **Sumber kebenaran** | [Techstack.md](Techstack.md) v2.0 dan [ARCHITECTURE.md](ARCHITECTURE.md) |
| **Kedudukan** | Menetapkan **bagaimana sistem dikirim dan dioperasikan**, baik di AWS maupun di server sekolah |

> **Status dokumen: sebagian.** **Pasal 2, 3, 6, dan 9 sudah ditulis dan berlaku.** Pasal 1, 4, 5, 7, dan 8 masih berupa rancangan susunan; isinya menyusul ketika Terraform mulai ditulis.
>
> Pengenal akun AWS ditulis sebagai `<ID-AKUN>` di seluruh dokumen ini. Nilainya tidak dicantumkan di repositori.
>
> Pilihan teknologi berada pada [Techstack.md](Techstack.md); hubungan antar bagian berada pada [ARCHITECTURE.md](ARCHITECTURE.md). Dokumen ini tidak mengulang keduanya.
>
> Dokumen ini berubah jauh lebih sering daripada kedua dokumen di atas, karena isinya mengikuti keadaan infrastruktur yang berjalan. Itulah alasan pemisahannya.

---

## Rancangan Susunan

| Pasal | Judul | Yang akan dimuat | Asal isi |
|:--:|---|---|---|
| 1 | Susunan `infra/` | Pohon direktori Terraform beserta tanggung jawab tiap modul: `network`, `data`, `compute`, `frontend`, `observability` | Techstack v1 §12 |
| **2** | **Terraform** | **Sudah ditulis.** Pembagian kepemilikan, skema B, pemisahan `bootstrap/` beserta alasannya, lingkungan `dev` dan `prod`, provider alias `us-east-1` untuk ACM, pengelolaan state dan penguncian, konvensi penamaan sumber daya, serta **pembuatan rahasia di luar Terraform** sehingga hanya ARN yang masuk ke state | Techstack v1 §12 dan Techstack v2 §7, diperluas |
| **3** | **CI/CD** | **Sudah ditulis.** Tahapan pada pull request dan pada merge, pertukaran token OIDC dengan IAM role, pembangunan image, urutan migrasi sebelum fungsi diperbarui, dan pemindahan alias | Techstack v1 §12 |
| 4 | Lingkungan pengembangan dan pengujian | `docker compose up`, adapter lokal, serta tiga tingkat pengujian — unit, integrasi, dan pembuktian penegakan oleh basis data | Techstack v1 §13 |
| 5 | **Urutan penaikan pertama** | **Baru.** Infrastruktur dinaikkan lebih dahulu dengan API yang hanya memuat `GET /healthz`, sehingga VPC, RDS, Function URL, CloudFront, dan pipeline terbukti hidup sebelum backend selesai. Termasuk **pembuktian penandatanganan OAC atas request ber-body**, yang wajib dilakukan pada tahap ini | sebagian dari Techstack v1 §12 |
| **6** | **Rollback** | **Sudah ditulis.** Pengembalian aplikasi dengan memindahkan alias Lambda ke versi sebelumnya. Yang lebih penting: **migrasi basis data tidak dapat dikembalikan otomatis**, sehingga perlu aturan tersendiri — migrasi wajib kompatibel mundur, dan penghapusan kolom dipisahkan dari penambahannya ke rilis berikutnya | — |
| 7 | **Pemasangan on-prem** | **Baru.** Skrip `install.sh` tunggal dan idempoten (CK-15) beserta sembilan langkahnya: pemeriksaan prasyarat, pemasangan Docker, penyusunan `.env` dengan kata sandi acak, `docker compose up`, migrasi, pembuatan akun Administrator, pemasangan timer pencadangan, penyalaan pembaruan keamanan otomatis, dan verifikasi akhir. Disertai `docker-compose.yml` untuk on-prem dan runbook satu halaman | — |
| 8 | **Operasional** | **Baru.** Pencadangan beserta **prosedur pemulihannya yang sudah pernah diuji**, rotasi kredensial basis data di Secrets Manager dan penggantian kunci Elice di Parameter Store, alarm CloudWatch, AWS Budgets, pemantauan sisa kredit layanan AI, dan verifikasi biaya lewat AWS Pricing Calculator | — |
| **9** | **Identitas dan akses** | **Sudah ditulis — lihat di bawah.** Dua IAM user, satu grup, tujuh role, dan nol access key pada jalur mesin. Ditulis lebih dahulu karena Terraform tidak dapat dijalankan tanpanya | — |
| — | Lampiran Catatan Keputusan | Bernomor `CK-D-01` dan seterusnya, sehingga tidak bertabrakan dengan `CK-xx` pada Techstack.md maupun `CK-A-xx` pada ARCHITECTURE.md | — |

---

## Catatan penyusunan

**Empat dari delapan pasal adalah isi baru.** Pasal 1 sampai 4 memindahkan isi yang sudah tertulis pada `Techstack.md` versi 1; pasal 5 sampai 8 belum pernah ditulis di dokumen mana pun.

**Dua pasal yang paling menentukan dan paling mudah ditunda:**

**Pasal 6 — rollback.** Pengembalian aplikasi mudah karena artefaknya image. Yang tidak mudah adalah basis data: migrasi yang sudah berjalan tidak dapat dibatalkan begitu saja, terlebih pada basis data berisi data sekolah sungguhan. Aturannya perlu ditetapkan sebelum migrasi pertama ditulis, bukan sesudahnya.

**Pasal 8 — pencadangan beserta pemulihannya.** Pencadangan yang belum pernah diuji pemulihannya bukanlah pencadangan. Ini berlaku bagi RDS maupun bagi server sekolah, dan berkaitan langsung dengan **V6** pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md) yang masih menunggu validasi.

**Kapan dokumen ini mulai diisi.** Ketika Terraform mulai ditulis. Menuliskannya lebih awal berarti mengarang nama modul, prosedur, dan angka yang belum diketahui.

**Kecuali Pasal 2, 3, 6, dan 9.** Keempatnya memuat keputusan yang sudah diambil dan tidak boleh menunggu, karena masing-masing menjadi prasyarat pekerjaan yang segera dimulai: identitas mendahului Terraform, pembagian kepemilikan mendahului modul Terraform pertama, dan aturan rollback mendahului migrasi pertama.

---

## 2. Terraform

### 2.1 Pembagian kepemilikan

Terraform dan CI memiliki daur hidup yang jauh berbeda. Terraform dijalankan sesekali; CI dijalankan tiap merge. Keduanya menyentuh fungsi Lambda yang sama, sehingga pembagian kepemilikannya **wajib dinyatakan, bukan disepakati lisan**.

| Terraform memiliki **cangkang** | CI memiliki **isi** |
|---|---|
| Memori, batas waktu, arsitektur | `image_uri` |
| Execution role | Version yang diterbitkan |
| Pengaturan VPC dan security group | Ke mana alias `live` menunjuk |
| Variabel lingkungan | — |
| Reserved concurrency | — |
| Function URL, keberadaan alias | — |
| ECR, RDS, S3, CloudFront, IAM | — |

**Cangkang** berarti seluruh bagian fungsi yang bukan kode. **Isi** berarti kode yang dijalankan, yaitu image-nya.

### 2.2 Skema B — `ignore_changes` di dua tempat

```hcl
resource "aws_lambda_function" "api" {
  image_uri = "${aws_ecr_repository.app.repository_url}:bootstrap"
  publish   = false                      # version diterbitkan CI, bukan Terraform

  lifecycle {
    # image_uri dimiliki CI. Lihat Pasal 3 dan CK-D-02.
    ignore_changes = [image_uri]
  }
}

resource "aws_lambda_alias" "live" {
  name             = "live"
  function_name    = aws_lambda_function.api.function_name
  function_version = "1"                 # hanya dipakai saat pembuatan pertama

  lifecycle {
    # ke mana alias menunjuk dimiliki CI. Ini tindakan rilis itu sendiri.
    ignore_changes = [function_version]
  }
}
```

**Dua `ignore_changes`, bukan satu.** Melupakan yang kedua menghasilkan kegagalan yang lebih buruk daripada melupakan yang pertama: `terraform apply` untuk urusan yang tidak berhubungan akan mengembalikan alias ke version 1, yaitu image bootstrap yang hanya memuat `GET /healthz`. Seluruh aplikasi lenyap, dan penyebabnya adalah perintah yang tampaknya tidak menyentuh aplikasi sama sekali.

`publish = false` juga wajib. Dengan `publish = true`, Terraform menerbitkan version sendiri pada setiap apply, sehingga penomoran version menjadi rebutan dua sistem.

**`ignore_changes` bukan mengabaikan masalah.** Ia menuliskan batas kepemilikan di tempat yang dibaca orang berikutnya, sehingga tidak ada yang "memperbaiki" atribut yang memang bukan urusannya. Bentuknya sama dengan Prinsip ④ pada [ARCHITECTURE.md](ARCHITECTURE.md): yang dapat dijamin secara struktural tidak diserahkan kepada ingatan orang.

### 2.3 Pemisahan `bootstrap/`

Fungsi Lambda tidak dapat dibuat tanpa image, sedangkan image tidak dapat didorong sebelum ECR ada. Terraform karenanya dipecah dua.

| Konfigurasi | Isi | Dijalankan |
|---|---|---|
| `bootstrap/` | Bucket state, tabel penguncian, repositori ECR | Sekali, di awal |
| `infra/` | Seluruh sisanya | Setiap kali infrastruktur berubah |

```
1  terraform apply pada bootstrap/     → ECR dan penyimpanan state
2  build & push image :bootstrap       → aplikasi minimal, hanya GET /healthz
3  terraform apply pada infra/         → fungsi dibuat memakai image itu
4  CI mengambil alih sejak sini
```

Tag `:bootstrap` sengaja dipilih agar tampak sementara. Tag seperti `:v1` mengundang orang mengira angkanya berarti sesuatu dan perlu dinaikkan.

### 2.4 Penamaan yang disepakati dua sistem

Empat nama ditulis di dua tempat — modul Terraform dan berkas workflow. Salah ketik di salah satunya menghasilkan rilis yang gagal tanpa petunjuk jelas.

| Nama | Nilai |
|---|---|
| Repositori ECR | `edutrack` |
| Fungsi aplikasi | `edutrack-api` |
| Fungsi migrasi | `edutrack-migrate` |
| Alias | `live` |

Keempatnya diterbitkan sebagai `output` Terraform, dan workflow membacanya dari sana bila memungkinkan.

### 2.5 Perlindungan terhadap penghapusan

```hcl
lifecycle { prevent_destroy = true }
```

Dipasang pada **repositori ECR**, **RDS**, dan **bucket rapor**.

ECR memerlukannya karena Lambda version mengunci digest image: menghapus image yang masih dirujuk sebuah version membuat rollback ke version itu tidak dapat menyala. Aturan daur hidup ECR karenanya juga **tidak boleh menghapus image bertag** — hanya image tanpa tag hasil percobaan yang boleh dibersihkan.

### 2.6 Deteksi drift

Terraform berhenti mengawasi `image_uri` dan `function_version` — **bukan berhenti mengawasi sisanya**. Perubahan memori, security group, atau aturan bucket yang dilakukan lewat konsol tetap merupakan drift sungguhan.

`terraform plan` dijalankan pada setiap pull request infrastruktur, dan terjadwal seminggu sekali. Selisih yang muncul di situ wajib ditindak, bukan diabaikan.

### 2.7 Menjawab "yang jalan sekarang commit mana?"

Karena Terraform tidak lagi mengetahuinya, jawabannya diambil dari AWS:

```bash
aws lambda get-function --function-name edutrack-api --qualifier live \
  --query 'Code.ImageUri' --output text
```

Karena tag image berupa git SHA, keluarannya langsung menunjuk satu commit persis. **Inilah alasan sesungguhnya tag git SHA diwajibkan** — bukan kerapian, melainkan penutup bagi lubang yang ditinggalkan `ignore_changes`.

---

## 3. CI/CD

### 3.1 Dua workflow

| Berkas | Pemicu | Menyentuh AWS? |
|---|---|---|
| `.github/workflows/pr.yml` | Pull request | **Tidak sama sekali** |
| `.github/workflows/deploy.yml` | Merge ke `main` | Ya, lewat OIDC |

### 3.2 Pemeriksaan pada pull request

```
npm ci → prettier --check → eslint → tsc --noEmit
       → vitest unit          domain/, tanpa I/O
       → vitest integration   service container postgres:17
             ├─ jalankan migrasi 0001–0010
             └─ jalankan bukti penegakan basis data (AGENTS.md §4.2)
       → docker build         tanpa push, membuktikan image jadi
```

Tidak ada kredensial AWS pada jalur ini. Pull request dari mana pun karenanya tidak dapat menyentuh infrastruktur.

### 3.3 Urutan rilis

```yaml
permissions:
  id-token: write        # tanpa ini, GitHub tidak menerbitkan token OIDC
  contents: read
```

| # | Langkah | Catatan |
|:--:|---|---|
| 1 | `aws-actions/configure-aws-credentials@v4` dengan `role-to-assume` | Tanpa access key, tanpa secret |
| 2 | Bangun image, tag = **git SHA** | Bukan `:latest` |
| 3 | Dorong ke ECR | Tag bersifat immutable |
| 4 | `update-function-code` pada `edutrack-migrate` | — |
| 5 | `aws lambda wait function-updated` | **Wajib.** Langkah 4 asinkron |
| 6 | `invoke` `edutrack-migrate`, sinkron | Gagal → pipeline berhenti, aplikasi tidak disentuh |
| 7 | `update-function-code` pada `edutrack-api` | — |
| 8 | `aws lambda wait function-updated` | **Wajib** |
| 9 | `publish-version` | Menghasilkan version bernomor |
| 10 | `update-alias live` → version baru | **Momen rilis sesungguhnya** |
| 11 | `GET /healthz` lewat CloudFront | Menguji jalur nyata, bukan hanya fungsinya |
| 12 | Gagal → `update-alias live` → version sebelumnya | Rollback dalam hitungan detik |

**Langkah 5 dan 8 paling mudah terlupa.** `update-function-code` mengembalikan jawaban sebelum AWS selesai memasang image. Menerbitkan version terlalu cepat menghasilkan version yang membeku pada image **lama**, dan gejalanya berupa rilis yang tampak berhasil tetapi tidak mengubah apa pun.

**Langkah 6 mendahului langkah 7 dengan sengaja**, sehingga terdapat jeda ketika kode lama berjalan di atas skema baru. Inilah alasan sesungguhnya aturan migrasi kompatibel mundur pada Pasal 6.

### 3.4 Yang tidak ada pada pipeline backend

`s3 sync` frontend, invalidasi CloudFront, dan `terraform apply`. Ketiganya bukan bagian backend; dua yang pertama milik pipeline frontend, dan yang ketiga dijalankan manusia.

### 3.5 Rilis yang memerlukan Terraform lebih dahulu

Karena variabel lingkungan dimiliki Terraform (§2.1), rilis yang memperkenalkan variabel baru menjadi **dua langkah**: Terraform lebih dahulu, CI menyusul. Melupakan urutan ini menghasilkan kode yang membaca variabel bernilai `undefined` di produksi.

Konsekuensinya: variabel lingkungan dijaga tetap sedikit — hanya ARN rahasia, `PORT`, dan pengaturan Lambda Web Adapter. Konfigurasi aplikasi yang berubah bersama kode ditempatkan **di dalam image**, bukan di variabel lingkungan.

---

## 6. Rollback

### 6.1 Aplikasi — mudah

Pindahkan alias `live` ke version sebelumnya. Selesai dalam hitungan detik, tanpa membangun ulang, tanpa menyentuh ECR.

```bash
aws lambda update-alias --function-name edutrack-api \
  --name live --function-version <nomor-sebelumnya>
```

Inilah manfaat utama version dan alias. Tanpa keduanya, rollback berarti mencari commit lama, membangun ulang image, lalu mendorongnya — sepuluh menit atau lebih, tepat ketika sistem sedang bermasalah.

### 6.2 Basis data — tidak bisa

**Migrasi yang sudah berjalan tidak dapat dikembalikan otomatis.** Tidak ada `terraform destroy` maupun pemindahan alias yang menolong. Pada basis data berisi nilai sekolah sungguhan, migrasi turun yang ditulis terburu-buru lebih berbahaya daripada tidak ada sama sekali.

Asimetri inilah yang membentuk seluruh aturan berikutnya.

### 6.3 Aturan migrasi

| # | Aturan |
|:--:|---|
| 1 | **Migrasi wajib kompatibel mundur.** Kode versi sebelumnya harus tetap berjalan di atas skema baru, karena Pasal 3 langkah 6 memang menciptakan jeda itu |
| 2 | **Penghapusan kolom dipisahkan ke rilis berikutnya**, setelah kode yang memakainya tidak lagi berjalan |
| 3 | **Penambahan kolom `NOT NULL` wajib disertai nilai bawaan**, atau dipecah menjadi tiga rilis: tambah nullable, isi, baru ketatkan |
| 4 | **Penggantian nama kolom dilarang.** Tambah kolom baru, salin, hapus pada rilis berikutnya |
| 5 | Setiap berkas migrasi dijalankan di dalam satu transaksi ([SCHEMA.md §9.1](SCHEMA.md)) |

Aturan 2 sampai 4 adalah penerapan aturan 1 pada tiga bentuk perubahan yang paling sering muncul.

### 6.4 Ketika rollback aplikasi tidak cukup

Apabila rilis yang rusak sudah menulis data yang salah, memindahkan alias **tidak memperbaiki datanya**. Perbaikan data adalah pekerjaan tersendiri: perbaiki lewat migrasi baru atau perintah CLI, bukan dengan mengembalikan skema.

### 6.5 Enam lapis penjagaan

Aturan pada §6.3 tidak boleh bergantung pada ingatan orang yang menulis migrasi. Enam lapis berikut menegakkannya, disusun dari yang paling murah (CK-D-03).

| # | Lapis | Biaya | Dipasang sebelum |
|:--:|---|---|---|
| 0 | Header klasifikasi pada tiap berkas migrasi | ~0 | Migrasi 0001 |
| 1 | Nama berkas `expand` dan `contract` | ~0 | Migrasi 0001 |
| 2 | Linter migrasi di CI | 1 baris | Migrasi 0001 |
| 3 | Bukti tidak ada yang memakai, sebelum `contract` | 1 perintah | Migrasi 0001 |
| 4 | Tes rilis sebelumnya dijalankan terhadap skema baru | 1 job CI | Data sekolah dimuat |
| 5 | Latihan rollback sungguhan | 1 sesi | Data sekolah dimuat |

**Lapis 0 — klasifikasi.** Setiap berkas migrasi dibuka dengan header wajib. Yang dipaksa bukan formatnya, melainkan **keputusannya ditulis alih-alih disimpulkan**.

```sql
-- migrasi : 0011
-- jenis   : additive | backward-compatible | breaking | dual-schema
-- mundur  : ya | tidak — beserta alasannya
-- dibaca  : api, migrate, app_ro
-- penutup : nomor migrasi contract yang kelak menutupnya, atau —
```

Migrasi `contract` mengisi baris `penutup` dengan nomor migrasi `expand` yang ditutupnya. Dengan begitu, `expand` yang belum pernah ditutup **terlihat** — dan tidak menumpuk menjadi kolom mati yang tidak berani disentuh siapa pun.

**Lapis 1 — penamaan.**

```
0011_expand_tambah_deskripsi.sql
0012_expand_isi_deskripsi.sql
0013_contract_hapus_kkm.sql
```

Berkas `contract` yang muncul tanpa `expand` pendahulunya adalah tanda bahaya yang terlihat sejak daftar berkas, sebelum isinya dibaca.

**Lapis 2 — linter.** `squawk` membaca berkas SQL dan menolak pola berbahaya sebelum sampai ke peninjau manusia.

```yaml
- run: npx squawk migrations/*.sql
```

Menangkap `DROP COLUMN`, `DROP TABLE`, `RENAME COLUMN`, dan `NOT NULL` tanpa `DEFAULT` — yaitu aturan 2, 3, dan 4 pada §6.3, seluruhnya secara otomatis.

**Lapis 3 — konfirmasi, bukan harapan.** Sebelum migrasi `contract` dijalankan, keberadaan pemakai dibuktikan, bukan diperkirakan:

```bash
grep -rn "kkm" backend/src/ && echo "MASIH DIPAKAI — jangan dihapus"
```

Pemeriksaan ini murah **karena konsumennya hanya tiga dan seluruhnya diketahui**: fungsi `api`, fungsi `migrate`, dan jalur AI lewat `app_ro`. NG2 mencabut konsumen pihak ketiga, sehingga tidak ada pemakai tak terdaftar yang perlu dikhawatirkan.

**Lapis 4 — pembuktian, bukan pelarangan.** Ketiga lapis sebelumnya melarang pola yang **diketahui** berbahaya. Lapis ini membuktikan hal yang sebenarnya dijanjikan §6.3: kode rilis sebelumnya masih berjalan di atas skema baru.

```
Satu job tambahan pada pr.yml:

1  nyalakan postgres kosong
2  jalankan seluruh migrasi, termasuk yang baru        → skema baru
3  ambil suite tes dari commit yang berjalan di produksi
4  jalankan tes lama itu terhadap skema baru
5  lulus → kompatibilitas mundur terbukti
```

Commit yang berjalan di produksi diambil lewat perintah pada §2.7. Lapis ini menangkap yang tidak terpikir masuk daftar larangan — misalnya `CHECK` baru yang menolak nilai yang masih dikirim kode lama.

**Lapis 5 — rollback yang dijalankan, bukan dituliskan.** §6.1 menyatakan rollback cukup memindahkan alias. Pernyataan itu belum pernah diuji.

Sebelum data sekolah sungguhan dimuat, dilakukan **satu latihan**: rilis versi cacat dengan sengaja, mundurkan alias, catat waktunya, dan simpan hasilnya di runbook.

Alasannya sama persis dengan yang sudah dinyatakan dokumen ini tentang pencadangan pada Pasal 8 — *pencadangan yang belum pernah diuji pemulihannya bukanlah pencadangan.* Hal yang sama berlaku bagi rollback.

---

## 9. Identitas dan akses

### 9.1 Prinsip

**Manusia memiliki IAM user. Mesin tidak memiliki apa pun.**

| Jenis identitas | Bentuk | Kredensial |
|---|---|---|
| Manusia | IAM user, bernama orang | Satu access key, berdaya nyaris nol (§9.2) |
| GitHub Actions | Role, dipinjam lewat OIDC | Tidak ada — token berumur satu kali jalan |
| Fungsi Lambda | Execution role | Tidak ada — diberikan runtime |
| NAT instance | Instance profile | Tidak ada, dan tanpa kunci SSH |

Kekuatan tidak pernah melekat pada identitas manusia. Ia **dipinjam** untuk waktu terbatas, dijaga MFA, dan tercatat per sesi di CloudTrail. Akibatnya, kebocoran kredensial di laptop tidak dengan sendirinya menjadi kebocoran akun.

Pengguna **root** dipakai hanya untuk urusan tingkat akun — penagihan, dukungan, dan penutupan akun — serta untuk bootstrap awal. Root wajib ber-MFA dan tidak boleh memiliki access key.

### 9.2 Identitas manusia

Satu IAM user per orang, **bernama orang**, bukan nama peran. Dengan dua orang di proyek ini, CloudTrail yang hanya mencatat `admin` tidak dapat menjawab siapa yang melakukan apa.

```
IAM user  Andreas        ──┐
IAM user  <frontend>     ──┴─► IAM group  Edutrack-dev
                                     │
                                     │  EdutrackAssumeRoles
                                     │  EdutrackSelfManageCredentials
                                     ▼
                    ┌────────────────┴────────────────┐
           edutrack-terraform                edutrack-readonly
```

| Ketentuan | Nilai |
|---|---|
| Akses konsol | Ya |
| **MFA** | **Wajib.** `EdutrackAssumeRoles` menolak peminjaman tanpanya |
| Access key | **Tepat satu**, diperlukan agar CLI dapat memanggil `sts:AssumeRole` |
| Izin langsung pada user | **Tidak ada.** Seluruhnya lewat grup |

**Dua policy grup, keduanya customer managed:**

`EdutrackAssumeRoles` — inti rancangan. Satu-satunya kekuatan yang dimiliki anggota grup.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AsumsiRoleEdutrack",
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": [
      "arn:aws:iam::<ID-AKUN>:role/edutrack-terraform",
      "arn:aws:iam::<ID-AKUN>:role/edutrack-readonly"
    ],
    "Condition": { "Bool": { "aws:MultiFactorAuthPresent": "true" } }
  }]
}
```

`EdutrackSelfManageCredentials` — mengizinkan setiap orang mengurus kata sandi dan perangkat MFA **miliknya sendiri**, dibatasi variabel `${aws:username}`. Tanpa policy ini, pengguna baru tidak dapat mendaftarkan MFA, dan karena policy pertama mensyaratkan MFA, ia terkunci total. Ini jebakan yang paling mudah terjadi pada rancangan semacam ini.

**Access key pada user manusia diterima secara sadar.** Ia tidak dapat dihindari selama `source_profile` pada AWS CLI membutuhkan kredensial permanen untuk memanggil `sts:AssumeRole`. Yang membuatnya dapat diterima adalah kekuatannya sendiri: kunci itu hanya dapat mengurus kredensial pemiliknya dan meminjam role — dan peminjaman ditolak tanpa kode MFA. Kunci yang bocor tidak memberi apa pun kepada penemunya. Jalur naik menuju nol kunci permanen adalah IAM Identity Center, yang berlebihan untuk dua orang.

### 9.3 Role yang dipinjam manusia

Keduanya memiliki **trust policy yang sama** — peminjamnya memang orang yang sama — dan **permission yang berbeda**.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::<ID-AKUN>:root" },
    "Action": "sts:AssumeRole",
    "Condition": { "Bool": { "aws:MultiFactorAuthPresent": "true" } }
  }]
}
```

`Principal` bernilai `:root` berarti "identitas mana pun di dalam akun ini", **bukan** pengguna root. Yang benar-benar membatasi adalah `EdutrackAssumeRoles` pada grup.

| Role | Permission | Umur sesi | Dipakai untuk |
|---|---|--:|---|
| `edutrack-terraform` | `AdministratorAccess` | **4 jam** | `terraform apply` |
| `edutrack-readonly` | `ReadOnlyAccess` | 1 jam | Pekerjaan sehari-hari, penelusuran kegagalan |

**Umur sesi 4 jam bukan kelonggaran, melainkan keharusan.** Bawaan satu jam dapat habis di tengah `terraform apply` yang membuat CloudFront dan RDS. Karena peminjaman diikat MFA, SDK tidak dapat memperbarui kredensial tanpa bertanya, sehingga apply gagal di tengah jalan — keadaan yang jauh lebih merepotkan daripada gagal di awal.

**`AdministratorAccess` pada `edutrack-terraform` terwajarkan bukan karena Terraform butuh sedikit kuasa**, melainkan karena kuasa itu kini **sementara, dijaga MFA, dan tercatat per sesi**. Terraform memang perlu membuat IAM role, yang secara praktis setara admin; menyempitkannya menghasilkan daftar izin panjang yang tetap setara admin sambil menyulitkan penelusuran.

**`ReadOnlyAccess` mencakup `s3:GetObject`**, sehingga berkas rapor siswa terbaca. Untuk sistem berisi data akademik anak di bawah umur, role ini perlu dilengkapi satu pernyataan `Deny` pada bucket rapor. Penelusuran kegagalan tidak pernah memerlukan isi rapor seseorang. Belum dapat diterapkan karena bucket-nya belum ada; masuk bersama Terraform.

### 9.4 Role mesin — penerapan

Dipinjam GitHub Actions lewat OIDC. **Tidak ada access key, dan tidak ada IAM user.**

| Role | Dipinjam oleh | Izin | Yang justru penting: tidak boleh |
|---|---|---|---|
| `edutrack-gha-backend` | Repositori `Korean-Asean-Digital-Academy-Batch-4/backend`, ref `main` | Push ke satu repositori ECR · `lambda:UpdateFunctionCode`, `PublishVersion`, `UpdateAlias`, `GetFunction`, `InvokeFunction` pada dua fungsi | `CreateFunction`, `UpdateFunctionConfiguration`, `DeleteFunction`, `iam:PassRole`, apa pun pada bucket frontend |
| `edutrack-gha-frontend` | Repositori frontend, ref `main` | `s3:PutObject/DeleteObject/ListBucket` pada bucket frontend · `cloudfront:CreateInvalidation` pada satu distribusi | Lambda, ECR, RDS, bucket rapor |

**Ketiadaan `iam:PassRole` adalah akibat langsung dari pembagian kepemilikan pada Pasal 2 dan 3**: Terraform memiliki cangkang fungsi, CI hanya menukar isinya. Karena CI tidak pernah membuat maupun mengonfigurasi ulang fungsi, izin paling berbahaya itu dapat dihilangkan sepenuhnya.

**Trust policy wajib dipatok ke repositori dan ref sekaligus:**

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::<ID-AKUN>:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": { "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:Korean-Asean-Digital-Academy-Batch-4/backend:ref:refs/heads/main"
  }}
}
```

Menulis `repo:<ORG>/*` pada `sub` membuat **repositori mana pun di organisasi itu dapat menerapkan ke produksi**. Ini kekeliruan OIDC yang paling sering terjadi, dan tidak menimbulkan gejala apa pun sampai disalahgunakan.

Satu OIDC provider per akun, dipakai bersama kedua role.

### 9.5 Role eksekusi Lambda

| Role | Fungsi | Izin |
|---|---|---|
| `edutrack-lambda-api` | `api` | ENI VPC · CloudWatch Logs · `secretsmanager:GetSecretValue` pada dua rahasia `app_rw` dan `app_ro` · `ssm:GetParameter` beserta `kms:Decrypt` untuk kunci API Elice · `s3:GetObject/PutObject/DeleteObject` pada bucket rapor |
| `edutrack-lambda-migrate` | `migrate` | ENI VPC · CloudWatch Logs · `secretsmanager:GetSecretValue` **hanya** pada rahasia `edutrack_owner` |
| `edutrack-nat` | NAT instance | `AmazonSSMManagedInstanceCore`, sehingga tidak diperlukan kunci SSH |

**Kedua fungsi memakai image yang sama tetapi role yang berbeda, dan pemisahan itu menentukan.** Fungsi `migrate` menjalankan DDL sehingga membutuhkan kredensial `edutrack_owner`; fungsi `api` melayani request dan tidak boleh dapat membacanya. Tanpa pemisahan ini, satu kekeliruan kode pada jalur permintaan dapat mengambil kredensial pemilik dan melewati seluruh pemisahan `app_rw` dan `app_ro` yang dibangun CK-08.

IAM di sini **memperkuat jaminan basis data, bukan mengulanginya**: [SCHEMA.md §7](SCHEMA.md) memisahkan hak di dalam PostgreSQL, dan Pasal ini memastikan kredensial yang paling berkuasa tidak pernah berada dalam jangkauan jalur request.

`s3:DeleteObject` pada `edutrack-lambda-api` bukan kelebihan izin. Ia diperlukan CK-A-05: setiap koreksi Administrator atas data final menghapus berkas rapor terkait dari S3 di dalam transaksi yang sama.

### 9.6 Konfigurasi CLI

```ini
[profile andreas]
region = ap-southeast-1

[profile edutrack]
region           = ap-southeast-1
role_arn         = arn:aws:iam::<ID-AKUN>:role/edutrack-terraform
source_profile   = andreas
mfa_serial       = arn:aws:iam::<ID-AKUN>:mfa/Andreas
duration_seconds = 14400

[profile edutrack-ro]
region         = ap-southeast-1
role_arn       = arn:aws:iam::<ID-AKUN>:role/edutrack-readonly
source_profile = andreas
mfa_serial     = arn:aws:iam::<ID-AKUN>:mfa/Andreas
```

```bash
AWS_PROFILE=edutrack-ro  aws sts get-caller-identity   # sehari-hari
AWS_PROFILE=edutrack     terraform apply               # saat mengubah infrastruktur
```

Bekerja dengan `edutrack-ro` sebagai kebiasaan **bukan tembok** — orang yang sama tetap dapat meminjam keduanya. Gunanya membuat jalur berbahaya menjadi jalur yang dipilih secara sadar, bukan jalur bawaan.

### 9.7 Ringkasan

**Dua IAM user, satu grup, tujuh role, dan nol access key pada seluruh jalur mesin.**

Nama role bersifat **case-sensitive** dan seluruhnya huruf kecil. Ketidakcocokan huruf besar-kecil terhadap ARN di dalam `EdutrackAssumeRoles` menghasilkan `AccessDenied` yang membingungkan, karena semuanya tampak benar di layar.

### 9.8 Gerbang penerapan

Selama pemeriksaan berikut belum lulus, **pengguna root belum boleh ditinggalkan** — root adalah satu-satunya jalan memperbaiki apabila ada yang keliru.

```bash
AWS_PROFILE=edutrack aws sts get-caller-identity
```

Keluarannya wajib memuat `assumed-role/edutrack-terraform/...`, **bukan** `user/Andreas`.

Sesudahnya: periksa **Credential report**, pastikan user `admin` bawaan tidak memiliki access key aktif, lalu hapus.

### 9.9 Keadaan pada 6 Agustus 2026

| Sudah ada | Belum ada |
|---|---|
| Grup `Edutrack-dev` | Role `edutrack-terraform` dan `edutrack-readonly` |
| `EdutrackAssumeRoles`, `EdutrackSelfManageCredentials` | Pendaftaran MFA |
| User `Andreas`, sudah masuk grup | Kedua role OIDC dan ketiga role eksekusi |
| — | Penghapusan user `admin` bawaan |

---

## Lampiran — Catatan Keputusan

Bernomor dan bertanggal. Entri tidak disunting; perubahan keputusan ditulis sebagai entri baru yang menyebut nomor yang digantikannya.

### CK-D-01 · 6 Agustus 2026 · Manusia memakai IAM user yang meminjam role; mesin tidak memiliki kredensial

**Diputuskan.** Dua IAM user bernama orang, satu grup, dan tujuh role. Seluruh kekuatan dipinjam melalui `sts:AssumeRole` yang dijaga MFA. GitHub Actions memakai OIDC, dan fungsi Lambda memakai execution role; keduanya tanpa access key.

**Alasan.** Terraform harus dijalankan sebagai sesuatu, dan sesuatu itu tidak boleh berupa root. Menjadikannya IAM user beraksesan penuh memindahkan persoalan alih-alih menyelesaikannya: kredensial permanen berkekuatan penuh berada di laptop, tanpa batas waktu dan tanpa MFA pada tindakan yang berbahaya.

Peminjaman role membalik keadaan itu. Kredensial yang tersimpan di laptop nyaris tidak berdaya; yang berdaya hanya sesi berumur empat jam yang menuntut kode MFA untuk dibuat. Setiap tindakan juga tercatat atas nama sesi di CloudTrail, sehingga "siapa yang menghapus itu" memiliki jawaban.

**Alternatif yang ditolak.**

*IAM user dengan `AdministratorAccess` langsung.* Bentuk yang paling sedikit langkahnya. Ditolak karena menghasilkan kredensial permanen berkekuatan penuh tanpa syarat MFA, dan karena izin yang menempel pada user jauh lebih sulit diaudit daripada izin yang menempel pada role.

*IAM Identity Center.* Menghapus kebutuhan access key sepenuhnya dan merupakan jalur naik yang benar. Ditolak untuk saat ini karena menambah lapisan penyiapan yang tidak sepadan untuk dua orang pada satu akun.

*Satu role gabungan tanpa `edutrack-readonly`.* Menghemat satu role. Ditolak karena membuat seluruh pekerjaan sehari-hari — membaca log, memeriksa keadaan sumber daya — berjalan dengan kewenangan penuh, sehingga tindakan merusak menjadi jalur bawaan alih-alih jalur yang dipilih.

**Konsekuensi yang diterima.**

1. **Satu access key permanen per manusia.** Tidak dapat dihindari selama `source_profile` AWS CLI membutuhkannya. Dikurangi dengan menjadikan kunci itu nyaris tanpa kekuatan (§9.2).
2. **Kode MFA diminta setiap kali sesi baru dibuat.** Umur empat jam pada `edutrack-terraform` membuatnya paling banyak sekali per sesi kerja.
3. **`AdministratorAccess` pada role Terraform.** Diterima karena sifatnya kini sementara dan tercatat, bukan karena kuasanya kecil.

### CK-D-02 · 7 Agustus 2026 · Terraform memiliki cangkang, CI memiliki isi

**Diputuskan.** Terraform membuat dan memiliki seluruh bagian fungsi Lambda **kecuali** image dan penunjuk alias. Kedua atribut itu dinyatakan `ignore_changes`, dan dimiliki CI. Disebut **skema B** sepanjang penyusunannya.

**Alasan.** Terraform dan CI memiliki daur hidup yang jauh berbeda — sesekali berbanding beberapa kali sehari — tetapi menyentuh sumber daya yang sama. Tanpa pembagian yang dinyatakan, keduanya sama-sama merasa memiliki `image_uri`, dan `terraform apply` yang dijalankan untuk urusan **yang sama sekali tidak berhubungan** akan mengembalikan aplikasi ke versi lama tanpa satu pun pesan kesalahan.

Sembilan alasan Terraform akan dijalankan kembali sudah tertulis di dokumen proyek sebelum satu baris kode pun ada: aturan daur hidup S3 untuk arsip ZIP, `Deny` bucket rapor pada `edutrack-readonly`, penyesuaian memori setelah lama render diukur, nama domain dan ACM, penggantian NAT dengan Egress-only IGW, provisioned concurrency, jalur naik ke ECS Fargate, alarm dan anggaran, serta sumber daya frontend. Anggapan bahwa Terraform "hanya sekali jalan" karenanya tidak bertahan.

**Justru jarangnya yang berbahaya.** Bila Terraform dijalankan tiap rilis, kemunduran diam-diam akan ketahuan dalam sehari. Karena ia dijalankan berminggu-minggu sekali untuk urusan lain, tidak akan ada yang menghubungkan "aplikasi mundur tiga rilis" dengan "kemarin saya memasang alarm biaya".

**Alternatif yang ditolak.**

*Tag tetap `:latest` di Terraform.* Menghapus drift tanpa `ignore_changes`, sehingga terlihat lebih sederhana. Ditolak karena **tidak merilis apa pun**: Lambda menerjemahkan tag menjadi digest pada saat pemasangan lalu menguncinya, sehingga mendorong `:latest` baru tidak mengubah fungsi yang berjalan. `update-function-code` tetap diperlukan, sementara penelusuran hilang — setiap version berbunyi `:latest` — immutable tag harus dimatikan, dan aturan daur hidup yang menghapus image tanpa tag dapat membuat rollback gagal karena image-nya sudah lenyap.

*Skema A — CI menjalankan `terraform apply -var image_tag=…`.* Satu sumber kebenaran dan mustahil drift. Ditolak karena setiap rilis aplikasi kemudian memerlukan kewenangan penuh Terraform di dalam CI, mengunci state, berjalan lebih lambat, dan membuat modul infrastruktur yang rusak ikut memblokir rilis aplikasi yang sehat.

*Terraform dipecah dua, `app/` khusus fungsi dan dijalankan CI.* Bentuk yang sah dan dipakai sebagian tim. Ditolak karena mengembalikan `iam:PassRole` ke role CI — izin menyerahkan role kepada sumber daya lain, yang justru berhasil dihilangkan skema ini (§9.4) — sekaligus tetap mengunci state pada tiap rilis.

**Lubang yang diketahui beserta penutupnya.**

| Lubang | Penutup |
|---|---|
| Alias dikembalikan Terraform ke version bootstrap | `ignore_changes = [function_version]` pada `aws_lambda_alias` (§2.2) |
| Terraform menerbitkan version sendiri | `publish = false` |
| Fungsi dibuat ulang setelah `destroy` dan kembali ke image bootstrap | `prevent_destroy` pada sumber daya berdata; rilis diulang untuk memulihkan |
| Image yang masih dirujuk version terhapus, sehingga rollback tidak menyala | `prevent_destroy` pada ECR; aturan daur hidup tidak menghapus image bertag (§2.5) |
| Drift pada atribut lain tidak lagi diperhatikan | `terraform plan` pada tiap PR infrastruktur dan terjadwal mingguan (§2.6) |
| Nama tidak cocok antara Terraform dan workflow | Diterbitkan sebagai `output` Terraform (§2.4) |
| Rilis yang memerlukan variabel lingkungan baru | Dua langkah, Terraform lebih dahulu (§3.5) |
| Terraform tidak lagi tahu versi yang berjalan | Perintah `get-function --qualifier live` (§2.7) |

**Konsekuensi yang diterima.** Terraform sengaja tidak mengetahui image mana yang berjalan, dan dua nama sumber daya hidup di dua tempat.

### CK-D-03 · 7 Agustus 2026 · Aturan migrasi ditegakkan enam lapis, bukan disiplin

**Diputuskan.** Lima aturan migrasi pada §6.3 ditegakkan enam lapis pada §6.5: header klasifikasi, penamaan `expand`/`contract`, linter `squawk` di CI, konfirmasi pemakai sebelum `contract`, tes rilis sebelumnya terhadap skema baru, dan latihan rollback sungguhan.

**Alasan.** Aturan yang hanya tertulis akan dilanggar orang yang belum membacanya, dan pelanggarannya baru terasa saat rollback — yaitu saat paling buruk. Ini penerapan Prinsip ④ [ARCHITECTURE.md](ARCHITECTURE.md) pada migrasi: yang dapat ditegakkan mesin tidak diserahkan kepada ingatan.

Tiga lapis berasal dari peninjauan `schema-evolution-and-contract-migrations` dan `data-migration-and-platform-cutover` pada kumpulan skill data engineering pihak ketiga. **Lapis 0** menjawab tuntutan bahwa jenis perubahan digolongkan lebih dahulu, bukan disimpulkan. **Lapis 3** menjawab peringatan bahwa penghapusan sering dijadwalkan *"based on hope instead of confirmation"*. **Lapis 5** menjawab prinsip *"rollback should be executable, not a sentence in a plan"*, yang menohok §6.1 sebagaimana ia ditulis semula.

**Alternatif yang ditolak.**

*Compatibility view dan dual write.* Ditawarkan kumpulan skill yang sama sebagai jalan agar penggantian nama kolom tidak memerlukan tiga rilis. Ditolak karena di PostgreSQL menuntut tabel diganti nama, view dibuat memakai nama lama, dan trigger ditulis agar view dapat ditulisi — menukar tiga migrasi sederhana dengan satu mekanisme yang harus diuji tersendiri. Bertentangan dengan Prinsip ② [Techstack.md §1](Techstack.md): kompleksitas hanya dibayar apabila ada kebutuhan produk yang membayarnya, dan rilis EduTrack tidak dikejar waktu.

*Migrasi turun untuk setiap migrasi naik.* Bentuk yang lazim pada banyak alat migrasi. Ditolak karena mengembalikan **bentuk**, bukan **isi**: kebalikan `ADD COLUMN` adalah `DROP COLUMN`, yang menghapus seluruh data yang sudah terisi. Migrasi turun juga merupakan migrasi yang dapat gagal, dijalankan pada basis data yang sudah bermasalah, dan hampir tidak pernah diuji.

*Mengandalkan §6.3 sebagai aturan tertulis saja.* Ditolak dengan alasan utama di atas.

**Konsekuensi yang diterima.** Satu dependensi pengembangan baru (`squawk`), satu job CI tambahan, dan satu sesi latihan rollback. Lapis 4 dan 5 baru wajib sebelum data sekolah sungguhan dimuat — batas yang sama dengan V1 pada [ATURAN-DAN-KRITERIA §5](ATURAN-DAN-KRITERIA.md).

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 6 Agustus 2026 | Kerangka dibuat sebagai bagian dari pemecahan `Techstack.md` menjadi tiga dokumen. Isi belum ditulis |
| 6 Agustus 2026 | **Versi 0.2 — Pasal 9 Identitas dan akses ditulis.** Ditetapkan dua IAM user bernama orang, satu grup `Edutrack-dev` dengan dua customer managed policy, dan tujuh role: dua dipinjam manusia, dua dipinjam GitHub Actions lewat OIDC, dan tiga untuk fungsi Lambda serta NAT instance. Seluruh jalur mesin tanpa access key. Pasal ini ditulis mendahului pasal lain karena Terraform tidak dapat dijalankan tanpanya. Lampiran Catatan Keputusan dibuka dengan **CK-D-01**. Dicatat pula keadaan penerapan per 6 Agustus 2026 pada §9.9 |
| 7 Agustus 2026 | **Versi 0.3 — Pasal 2, 3, dan 6 ditulis.** Ditetapkan **skema B** (**CK-D-02**): Terraform memiliki cangkang fungsi, CI memiliki isinya, dan `ignore_changes` dipasang di **dua** tempat — `image_uri` pada fungsi dan `function_version` pada alias. Yang kedua ditemukan belakangan dan lebih berbahaya, karena memindahkan alias adalah tindakan rilis itu sendiri. Ditolak: tag `:latest`, skema A, dan pemecahan Terraform menjadi `app/`. Dicatat delapan lubang yang diketahui beserta penutupnya. Pasal 6 menetapkan lima aturan migrasi kompatibel mundur, yang wajib berlaku sebelum migrasi 0001 ditulis. Nama repositori pada §9.4 dikoreksi menjadi `Korean-Asean-Digital-Academy-Batch-4/backend` |
| 7 Agustus 2026 | Ditambahkan **§6.5 Enam lapis penjagaan** beserta **CK-D-03**: header klasifikasi, penamaan `expand`/`contract`, linter `squawk`, konfirmasi pemakai sebelum `contract`, tes rilis sebelumnya terhadap skema baru, dan latihan rollback sungguhan. Tiga lapis di antaranya lahir dari peninjauan kumpulan skill data engineering pihak ketiga. Ditolak: compatibility view, dual write, dan migrasi turun |
