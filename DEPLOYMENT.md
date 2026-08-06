# Penerapan dan Operasional — EduTrack

| Keterangan | Isi |
|---|---|
| **Versi** | v0.2 — **Pasal 9 berisi, pasal lain masih rancangan** |
| **Tanggal** | 6 Agustus 2026 |
| **Disusun oleh** | Re:Code |
| **Sumber kebenaran** | [Techstack.md](Techstack.md) v2.0 dan [ARCHITECTURE.md](ARCHITECTURE.md) |
| **Kedudukan** | Menetapkan **bagaimana sistem dikirim dan dioperasikan**, baik di AWS maupun di server sekolah |

> **Status dokumen: sebagian.** **Pasal 9 — Identitas dan akses** sudah ditulis dan berlaku, karena identitas harus ada sebelum Terraform dapat dijalankan sama sekali. Pasal 1 sampai 8 masih berupa rancangan susunan; isinya menyusul ketika Terraform mulai ditulis.
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
| 2 | Terraform | Pemisahan `bootstrap/` beserta alasannya, lingkungan `dev` dan `prod`, provider alias `us-east-1` untuk ACM, pengelolaan state dan penguncian, konvensi penamaan sumber daya, serta **pembuatan rahasia di luar Terraform** sehingga hanya ARN yang masuk ke state | Techstack v1 §12 dan Techstack v2 §7, diperluas |
| 3 | CI/CD | Tahapan pada pull request dan pada merge, pertukaran token OIDC dengan IAM role, pembangunan image, urutan migrasi sebelum fungsi diperbarui, dan pemindahan alias | Techstack v1 §12 |
| 4 | Lingkungan pengembangan dan pengujian | `docker compose up`, adapter lokal, serta tiga tingkat pengujian — unit, integrasi, dan pembuktian penegakan oleh basis data | Techstack v1 §13 |
| 5 | **Urutan penaikan pertama** | **Baru.** Infrastruktur dinaikkan lebih dahulu dengan API yang hanya memuat `GET /healthz`, sehingga VPC, RDS, Function URL, CloudFront, dan pipeline terbukti hidup sebelum backend selesai. Termasuk **pembuktian penandatanganan OAC atas request ber-body**, yang wajib dilakukan pada tahap ini | sebagian dari Techstack v1 §12 |
| 6 | **Rollback** | **Baru — belum pernah ditulis.** Pengembalian aplikasi dengan memindahkan alias Lambda ke versi sebelumnya. Yang lebih penting: **migrasi basis data tidak dapat dikembalikan otomatis**, sehingga perlu aturan tersendiri — migrasi wajib kompatibel mundur, dan penghapusan kolom dipisahkan dari penambahannya ke rilis berikutnya | — |
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

**Kecuali Pasal 9.** Identitas dan akses tidak dapat menunggu Terraform, karena Terraform sendiri harus dijalankan sebagai sesuatu — dan sesuatu itu tidak boleh berupa pengguna root. Pasal 9 karenanya ditulis lebih dahulu, dan sudah diterapkan di akun AWS pada 6 Agustus 2026.

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
| `edutrack-gha-backend` | Repositori backend, ref `main` | Push ke satu repositori ECR · `lambda:UpdateFunctionCode`, `PublishVersion`, `UpdateAlias`, `GetFunction`, `InvokeFunction` pada dua fungsi | `CreateFunction`, `UpdateFunctionConfiguration`, `DeleteFunction`, `iam:PassRole`, apa pun pada bucket frontend |
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
    "token.actions.githubusercontent.com:sub": "repo:<ORG>/edutrack-backend:ref:refs/heads/main"
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

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 6 Agustus 2026 | Kerangka dibuat sebagai bagian dari pemecahan `Techstack.md` menjadi tiga dokumen. Isi belum ditulis |
| 6 Agustus 2026 | **Versi 0.2 — Pasal 9 Identitas dan akses ditulis.** Ditetapkan dua IAM user bernama orang, satu grup `Edutrack-dev` dengan dua customer managed policy, dan tujuh role: dua dipinjam manusia, dua dipinjam GitHub Actions lewat OIDC, dan tiga untuk fungsi Lambda serta NAT instance. Seluruh jalur mesin tanpa access key. Pasal ini ditulis mendahului pasal lain karena Terraform tidak dapat dijalankan tanpanya. Lampiran Catatan Keputusan dibuka dengan **CK-D-01**. Dicatat pula keadaan penerapan per 6 Agustus 2026 pada §9.9 |
