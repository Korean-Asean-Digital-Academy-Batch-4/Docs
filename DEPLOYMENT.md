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
| 5 | **Urutan penaikan pertama** | **Baru.** Infrastruktur dinaikkan lebih dahulu dengan API yang hanya memuat `GET /healthz`, sehingga VPC, RDS, Function URL, CloudFront, dan pipeline terbukti hidup sebelum backend selesai. Termasuk **pembuktian penandatanganan OAC atas request ber-body**, yang wajib dilakukan pada tahap ini. Termasuk pula penerbitan sertifikat ACM di `us-east-1` beserta dua record CNAME tanpa proxy di Cloudflare — satu untuk validasi ACM, satu untuk `app.edutrack.sch.id` (CK-17) | sebagian dari Techstack v1 §12 |
| **6** | **Rollback** | **Sudah ditulis.** Pengembalian aplikasi dengan memindahkan alias Lambda ke versi sebelumnya. Yang lebih penting: **migrasi basis data tidak dapat dikembalikan otomatis**, sehingga perlu aturan tersendiri — migrasi wajib kompatibel mundur, dan penghapusan kolom dipisahkan dari penambahannya ke rilis berikutnya | — |
| 7 | **Pemasangan on-prem** | **Baru.** Skrip `install.sh` tunggal dan idempoten (CK-15) beserta sepuluh langkahnya: pemeriksaan prasyarat, pemasangan Docker, penyusunan `.env` dengan kata sandi acak, **pendaftaran token Cloudflare Tunnel**, `docker compose up`, migrasi, pembuatan akun Administrator, pemasangan timer pencadangan, penyalaan pembaruan keamanan otomatis, dan verifikasi akhir. Disertai `docker-compose.yml` untuk on-prem — berisi `db`, `api`, `caddy`, dan `cloudflared` — serta runbook satu halaman. **Prasyarat yang dikerjakan tim lebih dahulu:** subdomain `<sekolah>.edutrack.sch.id` beserta tunnelnya dibuat sebelum skrip dijalankan, sehingga staf sekolah hanya menempelkan satu token (CK-17) | — |
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
  function_version = "$LATEST"           # hanya dipakai saat pembuatan pertama — CK-D-07

  lifecycle {
    # ke mana alias menunjuk dimiliki CI. Ini tindakan rilis itu sendiri.
    ignore_changes = [function_version]
  }
}
```

> Nilai awal `function_version` **diamandemen CK-D-07** dari `"1"` menjadi `"$LATEST"`. Keduanya tidak dapat berlaku bersamaan dengan `publish = false`: fungsi yang baru dibuat hanya memiliki `$LATEST`, dan version bernomor 1 tidak pernah ada untuk ditunjuk.

**Dua `ignore_changes`, bukan satu.** Melupakan yang kedua menghasilkan kegagalan yang lebih buruk daripada melupakan yang pertama: `terraform apply` untuk urusan yang tidak berhubungan akan mengembalikan alias ke version 1, yaitu image bootstrap yang hanya memuat `GET /healthz`. Seluruh aplikasi lenyap, dan penyebabnya adalah perintah yang tampaknya tidak menyentuh aplikasi sama sekali.

`publish = false` juga wajib. Dengan `publish = true`, Terraform menerbitkan version sendiri pada setiap apply, sehingga penomoran version menjadi rebutan dua sistem.

**`ignore_changes` bukan mengabaikan masalah.** Ia menuliskan batas kepemilikan di tempat yang dibaca orang berikutnya, sehingga tidak ada yang "memperbaiki" atribut yang memang bukan urusannya. Bentuknya sama dengan Prinsip ④ pada [ARCHITECTURE.md](ARCHITECTURE.md): yang dapat dijamin secara struktural tidak diserahkan kepada ingatan orang.

### 2.3 Pemisahan `bootstrap/`

Fungsi Lambda tidak dapat dibuat tanpa image, sedangkan image tidak dapat didorong sebelum ECR ada. Terraform karenanya dipecah dua.

| Konfigurasi | Isi | Dijalankan |
|---|---|---|
| `bootstrap/` | Bucket state beserta penguncian bawaannya, repositori ECR | Sekali, di awal |
| `infra/` | Seluruh sisanya | Setiap kali infrastruktur berubah |

Penguncian state **tidak lagi memakai tabel DynamoDB tersendiri** — lihat **CK-D-04**.

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

**Diamandemen CK-D-09.** Pengaman di atas berlaku bagi setiap `apply` sehari-hari, tetapi ia bukan lagi pengaman yang tidak berpintu. Pembongkaran lingkungan yang disengaja melewatinya lewat dua mekanisme yang sengaja dibedakan, karena Terraform memperlakukan keduanya secara berbeda:

| Pengaman | Bentuknya | Cara membukanya |
|---|---|---|
| `deletion_protection`, `skip_final_snapshot`, `recovery_window_in_days`, `force_destroy` | atribut biasa | variabel `izinkan_hapus`, bawaannya `false` |
| `prevent_destroy` | blok `lifecycle` | `skrip/izinkan-hapus.patch` pada repositori `infra` |

**Terraform melarang variabel di dalam blok `lifecycle`**, dan tidak ada berkas overlay yang dapat menggantikannya — sebuah resource tidak boleh didefinisikan dua kali. Karena itu `prevent_destroy` hanya dapat dibuka dengan mengubah berkasnya, dan perubahan itu dijadikan patch yang ikut ditinjau alih-alih penyuntingan yang tidak terlihat. `turunkan.sh` menolak berjalan apabila worktree tidak bersih, dan memasang `trap ... EXIT` yang mengembalikan berkasnya pada ketiga jalur keluar — selesai, galat, maupun Ctrl-C.

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
| 11 | `GET /api/healthz` lewat CloudFront | Menguji jalur nyata, bukan hanya fungsinya |
| 12 | Gagal → `update-alias live` → version sebelumnya | Rollback dalam hitungan detik |

**Alamatnya `/api/healthz`, bukan `/healthz`.** CloudFront hanya meneruskan `/api/*` ke Lambda ([ARCHITECTURE.md Pasal 3](ARCHITECTURE.md)); `/healthz` di akar akan dilayani bucket frontend, dan jawabannya `200` berisi `index.html` — pemeriksaan yang selalu lulus dan karenanya tidak memeriksa apa pun. Jalur `/healthz` tetap ada dan tetap dipakai readiness check Lambda Web Adapter, yang memanggilnya dari dalam container dan tidak melewati CloudFront sama sekali.

**Langkah 5 dan 8 paling mudah terlupa.** `update-function-code` mengembalikan jawaban sebelum AWS selesai memasang image. Menerbitkan version terlalu cepat menghasilkan version yang membeku pada image **lama**, dan gejalanya berupa rilis yang tampak berhasil tetapi tidak mengubah apa pun.

**Langkah 6 mendahului langkah 7 dengan sengaja**, sehingga terdapat jeda ketika kode lama berjalan di atas skema baru. Inilah alasan sesungguhnya aturan migrasi kompatibel mundur pada Pasal 6.

### 3.4 Yang tidak ada pada pipeline backend

`s3 sync` frontend, invalidasi CloudFront, dan `terraform apply`. Ketiganya bukan bagian backend; dua yang pertama milik pipeline frontend, dan yang ketiga dijalankan manusia.

### 3.5 Rilis yang memerlukan Terraform lebih dahulu

Karena variabel lingkungan dimiliki Terraform (§2.1), rilis yang memperkenalkan variabel baru menjadi **dua langkah**: Terraform lebih dahulu, CI menyusul. Melupakan urutan ini menghasilkan kode yang membaca variabel bernilai `undefined` di produksi.

Konsekuensinya: variabel lingkungan dijaga tetap sedikit — hanya ARN rahasia, `PORT`, dan pengaturan Lambda Web Adapter. Konfigurasi aplikasi yang berubah bersama kode ditempatkan **di dalam image**, bukan di variabel lingkungan.

## 5. Urutan penaikan pertama

**Sebagian.** §5.1 dan §5.2 sudah ditulis. Yang tersisa — penerbitan sertifikat ACM di `us-east-1` dan kedua record CNAME — menunggu domain dibeli (CK-17).

### 5.1 Pengisian rahasia

Keempat rahasia beserta tempat penyimpanannya ditetapkan [Techstack.md §7](Techstack.md). Yang ditetapkan di sini adalah **namanya, bentuk nilainya, dan cara memasukkannya** — CK-D-05, sebagaimana diamandemen **CK-D-06**.

| Rahasia | Layanan | Nama | Dibuat oleh |
|---|---|---|---|
| Kredensial `edutrack_owner` | Secrets Manager | dibangkitkan RDS, berbentuk `rds!db-…` | **RDS sendiri** — CK-D-06 |
| Kredensial `app_rw` | Secrets Manager | `edutrack/db/app_rw` | Manusia |
| Kredensial `app_ro` | Secrets Manager | `edutrack/db/app_ro` | Manusia |
| Kunci API Elice | SSM Parameter Store, `SecureString` | `/edutrack/ai/elice-api-key` | Manusia |

Bentuk penamaannya berbeda karena layanannya berbeda: SSM menuntut garis miring di depan untuk membentuk hierarki, Secrets Manager tidak.

**Rahasia `edutrack_owner` tidak memiliki nama yang dapat disepakati di muka**, karena ia dibangkitkan RDS beserta akhiran acak. Fungsi `migrate` karenanya menerima **ARN**-nya, bukan namanya; ARN itu dibaca Terraform lewat `master_user_secret` dan diteruskan sebagai variabel lingkungan (CK-D-06). Kedua rahasia yang dibuat manusia tetap diteruskan sebagai nama.

Ketiga kredensial basis data memakai bentuk nilai baku RDS, sehingga rotasi terjadwal pada Pasal 8 kelak tidak menuntut penulisan ulang:

```json
{ "username": "app_rw", "password": "…" }
```

#### Tiga aturan yang mengikat cara memasukkannya

**1. Nilai tidak pernah muncul pada baris perintah.** Argumen perintah terbaca seluruh pengguna mesin lewat `ps`, dan tersimpan pada riwayat shell. Nilainya diserahkan lewat berkas sementara berizin ketat, lalu berkasnya dihapus.

**2. Berkas sementara dibuat dengan `umask 077`.** Tanpa itu, berkas rahasia lahir dengan izin yang dapat dibaca pengguna lain pada mesin yang sama.

**3. Verifikasi tidak pernah mendekripsi.** Yang diperiksa keberadaan dan versinya, bukan isinya.

#### Kunci Elice — SSM Parameter Store

```bash
umask 077 && tmp=$(mktemp)
```

Tulis berkas permintaannya, lalu tempelkan kuncinya sebagai nilai `Value`:

```bash
cat > "$tmp" <<'JSON'
{
  "Name": "/edutrack/ai/elice-api-key",
  "Type": "SecureString",
  "Value": "TEMPELKAN_KUNCI_DI_SINI",
  "Description": "Kunci API Elice AI Cloud - Techstack sec 6",
  "Tier": "Standard"
}
JSON
```

```bash
aws ssm put-parameter --cli-input-json "file://$tmp" --region ap-southeast-3 && rm -f "$tmp"
```

`--cli-input-json` dipilih alih-alih `--value` karena bentuknya tidak menyisakan keraguan: nilainya berada di dalam berkas, bukan di dalam perintah.

**Kunci KMS dibiarkan bawaan** (`alias/aws/ssm`). Kunci yang dikelola sendiri menambah sekitar $1 per bulan tanpa menambah jaminan apa pun pada susunan satu akun ini, dan tier `Standard` menjadikan parameternya tidak berbiaya ([Techstack §8.2](Techstack.md)).

Verifikasi tanpa mendekripsi:

```bash
aws ssm get-parameter --name /edutrack/ai/elice-api-key --region ap-southeast-3 --query 'Parameter.{Type:Type,Version:Version,Diubah:LastModifiedDate}'
```

Penggantian kunci memakai perintah yang sama dengan `--overwrite`; versinya naik, dan versi lama tetap dapat dilihat untuk penelusuran.

#### Kredensial basis data — Secrets Manager

Berlaku bagi **`app_rw` dan `app_ro` saja**. Kredensial `edutrack_owner` tidak dibuat dengan cara ini — ia dibangkitkan RDS sendiri (CK-D-06), dan justru kredensial itulah yang dipakai untuk menyambung ke PostgreSQL guna membuat kedua role di bawah.

Dijalankan **sesudah** RDS menyala dan kata sandinya disetel pada PostgreSQL. Urutannya: bangkitkan kata sandi, setel pada basis data, baru simpan.

Kata sandi dibangkitkan **sekali** ke dalam variabel shell, lalu ditulis ke **dua** tempat dari variabel yang sama. Membangkitkannya dua kali — sekali untuk basis data, sekali untuk Secrets Manager — menghasilkan dua nilai berbeda, dan kegagalannya baru muncul pada rilis pertama sebagai `password authentication failed`.

```bash
umask 077 && tmp=$(mktemp -d) && SANDI_RW=$(openssl rand -base64 24 | tr -d '\n=/+')
```

Disetel pada PostgreSQL, lalu disimpan — keduanya tanpa melewati baris perintah:

```bash
printf "ALTER ROLE app_rw LOGIN PASSWORD '%s';\n" "$SANDI_RW" > "$tmp/role.sql"
```

```bash
printf '{"username":"app_rw","password":"%s"}' "$SANDI_RW" > "$tmp/app_rw.json"
```

**`put-secret-value`, bukan `create-secret`.** Wadahnya sudah dibuat Terraform (CK-D-05); `create-secret` dijawab `ResourceExistsException`.

```bash
aws secretsmanager put-secret-value --secret-id edutrack/db/app_rw --secret-string "file://$tmp/app_rw.json" --region ap-southeast-3
```

Diulang untuk `app_ro`, lalu `rm -rf "$tmp"` dan `unset`. Pemisahan siapa boleh membaca yang mana ditegakkan IAM (§9.5), bukan oleh penamaan.

#### ⚠️ `.pgpass` tidak dapat dipakai untuk kredensial pemilik

Menyambung sebagai `edutrack_owner` menuntut kata sandi yang dibangkitkan RDS, dan **kata sandi itu dapat memuat titik dua**. `.pgpass` memakai titik dua sebagai pemisah bidang, sehingga barisnya terurai salah **tanpa satu pun peringatan** — `psql` mengira kata sandinya adalah potongan sebelum titik dua pertama, dan menjawab `password authentication failed for user "edutrack_owner"`. Pesan itu menyesatkan ke arah kredensial yang keliru, padahal formatnya yang rusak.

Dipakai `PGPASSWORD`, yang tidak memiliki format sama sekali. Nilainya diserahkan sebagai lingkungan milik proses `psql` saja, sehingga tidak pernah masuk ke `argv` maupun riwayat shell:

```bash
PGPASSWORD="$(cat "$tmp/sandi-owner")" psql "host=127.0.0.1 port=15432 dbname=edutrack user=edutrack_owner sslmode=require" -f "$tmp/role.sql"
```

Ditemukan 12 Agustus 2026 saat prosedur ini dijalankan untuk pertama kalinya.

#### Verifikasi yang sesungguhnya

Memeriksa keberadaan versi rahasia **tidak cukup** — ia tidak membuktikan nilainya cocok dengan yang ada di PostgreSQL. Yang membuktikannya hanya satu: membaca kembali dari Secrets Manager, lalu memakainya untuk masuk.

```bash
aws secretsmanager get-secret-value --secret-id edutrack/db/app_rw --region ap-southeast-3 --query SecretString --output text | jq -r .password > "$tmp/uji" && PGPASSWORD="$(cat "$tmp/uji")" psql "host=127.0.0.1 port=15432 dbname=edutrack user=app_rw sslmode=require" -qtAc 'SELECT 1'
```

Tanpa langkah ini, selisih antara kedua tempat baru ketahuan pada langkah 11 §3.3 — yaitu sesudah migrasi terlanjur diterapkan.

#### Pembagian dengan Terraform

| Yang dibuat Terraform | Yang dibuat manusia |
|---|---|
| `aws_secretsmanager_secret` untuk `app_rw` dan `app_ro` — **wadahnya saja** | Isi kedua wadah itu, lewat perintah di atas |
| `manage_master_user_password = true` pada RDS — wadah **beserta isinya** dibuat RDS, bukan Terraform (CK-D-06) | — |
| Kebijakan IAM yang memberi izin baca per ARN | — |
| Variabel lingkungan Lambda berisi **nama** kedua rahasia buatan manusia, dan **ARN** rahasia terkelola RDS | — |

**`aws_secretsmanager_secret_version` tidak pernah dipakai**, karena resource itulah yang akan menaruh kata sandi ke dalam state.

**Parameter SSM tidak dikelola Terraform sama sekali.** Resource `aws_ssm_parameter` mewajibkan atribut `value`, sehingga tidak ada cara membuatnya lewat Terraform tanpa nilainya masuk ke state — dan `ignore_changes` tidak menolong, karena ia hanya mengabaikan perubahan sesudah nilai pertama tertulis. Terraform karenanya hanya menyusun ARN-nya dari nama yang sudah disepakati di atas, dan tidak pernah membacanya.

**Fungsi Lambda menerima nama, bukan nilai.** Pembacaannya terjadi saat container menyala, lewat interface `Secrets` di `ports/` ([ARCHITECTURE §12.1](ARCHITECTURE.md)).

### 5.2 Pembuktian penandatanganan OAC — B4

Dijalankan **selagi image `:bootstrap` masih terpasang**, karena alat ukurnya berada di dalam image itu. Prosedur dan skripnya pada repositori `infra` di `uji-oac/`. Dilaksanakan **12 Agustus 2026**; hasilnya menjadi **CK-A-12** pada [ARCHITECTURE.md](ARCHITECTURE.md).

**Hasilnya:** `GET` lolos tanpa syarat tambahan, sedangkan setiap request ber-body menuntut header `x-amz-content-sha256`. Rincian beserta akibatnya pada CK-A-12; yang dicatat di sini hanya dua jebakan yang ditemui saat menaikkannya, karena keduanya akan ditemui lagi oleh siapa pun yang membangun ulang lingkungan ini.

#### Jebakan 1 — OAC menuntut DUA izin Lambda, bukan satu

`lambda:InvokeFunctionUrl` **tidak cukup**. CloudFront juga memerlukan `lambda:InvokeFunction` pada fungsi yang sama. Dokumentasi OAC AWS menyebutkan keduanya sebagai dua perintah `add-permission` terpisah, dan yang kedua sangat mudah terbaca sebagai pengulangan yang pertama.

Gejalanya: **403 pada setiap request**, termasuk `GET` tanpa body, dengan kebijakan yang tampak persis benar di layar.

Yang membuatnya memakan waktu lama adalah pengujian yang menyesatkan: request bertanda tangan SigV4 memakai profil `edutrack` **lolos**, sehingga Function URL tampak sehat dan kecurigaan berpindah ke CloudFront. Sebabnya role `edutrack-terraform` ber-`AdministratorAccess` sehingga memiliki **kedua** izin, sementara principal CloudFront hanya diberi yang pertama.

**Cara membedakannya dalam hitungan detik.** Bandingkan badan jawaban 403 dari kedua jalur:

| Jalur | Badan jawaban | Artinya |
|---|---|---|
| Langsung ke Function URL, tanpa tanda tangan | `{"Message":"Forbidden"}` | Tidak ada tanda tangan sama sekali |
| Lewat CloudFront | `{"Message":"Forbidden. For troubleshooting Function URL authorization issues, see: …}` | Tanda tangan **ada** dan ditolak — persoalannya izin atau payload hash |

#### Jebakan 2 — `custom_error_response` berlaku se-distribusi

Blok itu semula dipasang untuk perutean React SPA. Ia **tidak dapat dibatasi pada satu cache behavior**: CloudFront hanya mengenal error response tingkat distribusi. Akibatnya setiap 403 dan 404 dari `/api/*` ikut dibelokkan ke `/index.html`, sehingga kontrak amplop `kesalahan` [API §2](API.md) rusak — 404 berubah menjadi 200 berisi HTML — dan galat yang sesungguhnya tertutup jawaban bucket frontend.

Perutean SPA karenanya diselesaikan lewat CloudFront Function pada perilaku bawaan saja, dan ditulis ketika repositori frontend ada.

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
| `edutrack-gha-backend` | Repositori `Korean-Asean-Digital-Academy-Batch-4/backend`, ref `main` | Push ke satu repositori ECR · `lambda:UpdateFunctionCode`, `PublishVersion`, `UpdateAlias`, `GetFunction`, **`GetFunctionConfiguration`**, **`GetAlias`**, `InvokeFunction` pada dua fungsi | `CreateFunction`, `UpdateFunctionConfiguration`, `DeleteFunction`, `iam:PassRole`, apa pun pada bucket frontend |

**Dua tindakan baca ditambahkan 12 Agustus 2026**, sesudah rilis pertama gagal karenanya. Keduanya tidak pernah didaftar semula karena pasal ini ditulis mendahului §3.3, dan yang menuntutnya adalah langkah pada pasal itu:

| Tindakan | Dituntut oleh |
|---|---|
| `lambda:GetFunctionConfiguration` | `aws lambda wait function-updated` — §3.3 langkah 5 dan 8. Waiter memanggilnya berulang, bukan `GetFunction` |
| `lambda:GetAlias` | Pembacaan version sebelumnya untuk jalur rollback — §3.3 langkah 12 |

**Keduanya hanya membaca, dan tidak melonggarkan apa pun.** Yang menjaga pembagian kepemilikan CK-D-02 adalah ketiadaan `UpdateFunctionConfiguration` — perhatikan bahwa namanya nyaris sama dengan `GetFunctionConfiguration` yang kini diizinkan. Yang satu mengubah cangkang yang dimiliki Terraform; yang lain hanya melihatnya.
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
| `edutrack-lambda-api` | `api` | ENI VPC · CloudWatch Logs · `secretsmanager:GetSecretValue` pada dua rahasia `app_rw` dan `app_ro` · `ssm:GetParameter` beserta `kms:Decrypt` untuk kunci API Elice · `s3:GetObject/PutObject/DeleteObject` pada isi bucket rapor · **`s3:ListBucket`** pada bucket rapor itu sendiri |
| `edutrack-lambda-migrate` | `migrate` | ENI VPC · CloudWatch Logs · `secretsmanager:GetSecretValue` **hanya** pada rahasia `edutrack_owner`, yaitu rahasia terkelola RDS yang ARN-nya dibaca dari `master_user_secret` (CK-D-06) |
| `edutrack-nat` | NAT instance | `AmazonSSMManagedInstanceCore`, sehingga tidak diperlukan kunci SSH |

**Kedua fungsi memakai image yang sama tetapi role yang berbeda, dan pemisahan itu menentukan.** Fungsi `migrate` menjalankan DDL sehingga membutuhkan kredensial `edutrack_owner`; fungsi `api` melayani request dan tidak boleh dapat membacanya. Tanpa pemisahan ini, satu kekeliruan kode pada jalur permintaan dapat mengambil kredensial pemilik dan melewati seluruh pemisahan `app_rw` dan `app_ro` yang dibangun CK-08.

IAM di sini **memperkuat jaminan basis data, bukan mengulanginya**: [SCHEMA.md §7](SCHEMA.md) memisahkan hak di dalam PostgreSQL, dan Pasal ini memastikan kredensial yang paling berkuasa tidak pernah berada dalam jangkauan jalur request.

`s3:DeleteObject` pada `edutrack-lambda-api` bukan kelebihan izin. Ia diperlukan CK-A-05: setiap koreksi Administrator atas data final menghapus berkas rapor terkait dari S3 di dalam transaksi yang sama.

**`s3:ListBucket` juga diperlukan, dan sebabnya tidak terlihat dari daftar tindakannya.** Aplikasi tidak pernah mendaftar isi bucket. Yang menuntutnya adalah perilaku S3 pada objek yang **belum ada**: tanpa `s3:ListBucket`, `HeadObject` menjawab **`403`** alih-alih `404`, karena S3 menolak membocorkan keberadaan objek kepada pemanggil yang tidak boleh mendaftarnya.

Akibatnya persis kebalikan dari yang diinginkan. Jalur render-saat-unduh memeriksa keberadaan berkas lebih dahulu ([API.md §8.4](API.md)); `403` itu bukan "belum ada" melainkan galat sungguhan, sehingga setiap perenderan gagal **sebelum satu pun berkas dibuat**. Gejalanya `berkas_terender: 0` tanpa satu pun pesan yang menyebut S3.

Izinnya dipatok pada bucket itu sendiri — `arn:…:edutrack-rapor-…`, tanpa `/*` — karena `ListBucket` adalah tindakan atas bucket, bukan atas objek. Ditemukan 12 Agustus 2026 saat B7 dijalankan.

### 9.6 Konfigurasi CLI

```ini
[profile andreas]
region = ap-southeast-3

[profile edutrack]
region           = ap-southeast-3
role_arn         = arn:aws:iam::<ID-AKUN>:role/edutrack-terraform
source_profile   = andreas
mfa_serial       = arn:aws:iam::<ID-AKUN>:mfa/Andreas
duration_seconds = 14400

[profile edutrack-ro]
region         = ap-southeast-3
role_arn       = arn:aws:iam::<ID-AKUN>:role/edutrack-readonly
source_profile = andreas
mfa_serial     = arn:aws:iam::<ID-AKUN>:mfa/Andreas
```

```bash
AWS_PROFILE=edutrack-ro  aws sts get-caller-identity   # sehari-hari
```

**Terraform tidak dapat memakai `AWS_PROFILE` di sini, dan ini bukan salah konfigurasi.** Profil `edutrack` memuat `mfa_serial`, sehingga peminjaman role menuntut kode enam angka. AWS CLI menanyakannya lewat prompt lalu menyimpan sesi hasilnya di `~/.aws/cli/cache/`. Terraform memakai AWS SDK for Go: ia membaca `~/.aws/config` yang sama, melihat `mfa_serial`, tetapi **tidak memiliki jalan untuk bertanya** — `terraform apply` bukan sesi interaktif, dan cache CLI tidak dibacanya. Yang muncul:

```
Error: assume role with MFA enabled, but AssumeRoleTokenProvider session option not set.
```

Jalan keluarnya bukan melemahkan MFA, melainkan **menyerahkan sesi yang sudah dipinjam CLI** kepada Terraform sebagai variabel lingkungan:

```bash
eval "$(aws configure export-credentials --profile edutrack --format env)"
```

```bash
terraform -chdir=bootstrap apply
```

Perintah pertama meminta kode MFA sekali — atau langsung memakai sesi yang masih hidup — lalu mengekspor `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, dan `AWS_SESSION_TOKEN` ke shell. Terraform kemudian memakai kredensial sementara itu apa adanya dan **tidak meminjam role sama sekali**, sehingga tidak pernah menyentuh MFA.

`AWS_PROFILE` sengaja **tidak** disertakan pada perintah kedua. Menyetel keduanya sekaligus membuat sumber kredensial menjadi ambigu bagi pembaca berikutnya, meskipun SDK memang mendahulukan variabel lingkungan.

Sesinya berumur `duration_seconds` di atas, yaitu **4 jam**. Sesudah itu `eval` diulang. Membutuhkan AWS CLI **2.9 atau lebih baru**.

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

### 9.9 Keadaan pada 11 Agustus 2026

| Sudah ada | Belum ada |
|---|---|
| Grup `Edutrack-dev` | Role `edutrack-readonly` |
| `EdutrackAssumeRoles`, `EdutrackSelfManageCredentials` | Pendaftaran MFA |
| User `Andreas`, sudah masuk grup | Ketiga role eksekusi Lambda dan NAT — dibuat `infra/` pada B3 |
| Role `edutrack-terraform` — terbukti oleh `terraform apply` pada `bootstrap/` (B1) | Role `edutrack-gha-frontend` |
| OIDC provider beserta role `edutrack-gha-backend` — terbukti oleh jabat tangan [Gitaction.md](Gitaction.md) (B0.5) | Penghapusan user `admin` bawaan |

**Role `edutrack-gha-backend` dibuat dengan tangan lewat konsol pada B0.5**, sebelum `infra/` ada. Ia karenanya **di-`import`** ke dalam state Terraform pada B5, bukan dibuat ulang — membuat ulang berarti menghapus role yang sedang dipercaya OIDC provider, dan jabat tangan yang sudah terbukti akan putus.

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

### CK-D-04 · 11 Agustus 2026 · Penguncian state memakai mekanisme bawaan S3 — mengamandemen CK-D-02 dan §2.3

**Diputuskan.** Bucket state memakai penguncian bawaan S3 lewat `use_lockfile = true` pada blok backend. **Tabel DynamoDB penguncian tidak dibuat.**

**Alasan.** Ketika §2.3 ditulis, satu-satunya cara mengunci state S3 memang tabel DynamoDB terpisah. Terraform 1.10 menambahkan penguncian berbasis berkas kunci di dalam bucket yang sama, dan Terraform 1.11 **menandai `dynamodb_table` sebagai usang**. Mempertahankan tabel berarti menambah satu sumber daya, satu izin IAM, dan satu tempat lagi yang dapat menyimpang — untuk memperoleh persis jaminan yang sama.

Yang menentukan bukan penghematan biayanya, melainkan bahwa argumen backend-nya sedang dalam jalur penghapusan. Konfigurasi yang dibangun di atas argumen usang akan berhenti bekerja pada peningkatan Terraform berikutnya, dan kegagalannya muncul pada saat paling buruk: ketika seseorang menjalankan `terraform init` untuk memperbaiki hal lain.

**Alternatif yang ditolak.** *Tetap memakai tabel DynamoDB sesuai §2.3 apa adanya.* Sesuai dokumen, tetapi memilih mekanisme yang sudah diumumkan akan dihapus. *Memakai keduanya.* Terraform memang mengizinkannya sebagai jalur perpindahan, tetapi mempertahankan dua mekanisme untuk satu jaminan adalah bentuk yang paling mudah ditinggalkan setengah jalan.

**Konsekuensi yang diterima.** Terraform yang dipakai wajib **1.10 atau lebih baru**; `required_version` pada `bootstrap/` dan `infra/` mematoknya. Penguncian menjadi bergantung pada operasi bersyarat S3, yang sudah bersifat konsisten kuat sejak 2020.

### CK-D-05 · 11 Agustus 2026 · Nama rahasia ditetapkan, dan parameter SSM berada di luar Terraform seluruhnya

**Diputuskan.** Keempat rahasia memakai nama pada §5.1. Kredensial basis data disimpan sebagai JSON `{username, password}`. Terraform membuat **wadah** rahasia Secrets Manager tetapi tidak pernah isinya, dan **tidak menyentuh parameter SSM sama sekali**.

**Alasan.** [Techstack §7](Techstack.md) sudah menetapkan nilai rahasia dibuat di luar Terraform sehingga hanya ARN yang masuk ke state. Yang belum ditetapkan adalah caranya, dan pada SSM caranya ternyata tidak dapat setengah-setengah: resource `aws_ssm_parameter` **mewajibkan** atribut `value`. Tidak ada bentuk penulisan yang membuatnya lahir tanpa nilai. `ignore_changes = [value]` sering diusulkan sebagai jalan keluar, tetapi ia hanya mengabaikan perubahan **sesudah** nilai pertama tertulis — nilai pertama itu sendiri tetap masuk ke state.

Secrets Manager berbeda: `aws_secretsmanager_secret` dan `aws_secretsmanager_secret_version` adalah dua resource terpisah, sehingga wadahnya dapat dikelola Terraform sementara isinya tidak pernah disentuh.

**Bentuk JSON `{username, password}` dipilih** karena itulah bentuk yang diharapkan rotasi terjadwal Secrets Manager, yang direncanakan Pasal 8. Menyimpan kata sandi telanjang berarti menulis ulang bentuknya ketika rotasi dinyalakan.

**Alternatif yang ditolak.** *Menaruh kunci Elice di Secrets Manager bersama ketiga kredensial basis data.* Menyeragamkan satu layanan, tetapi menambah $0,40 per bulan untuk rahasia yang paling ringan akibat kebocorannya, sedangkan SSM tier Standard tidak berbiaya. Pembedaan tempat penyimpanan pada [Techstack §7](Techstack.md) memang mengikuti akibat kebocoran, bukan kenyamanan. *Membangkitkan kata sandi basis data dari Terraform lewat `random_password`.* Ditolak: nilainya masuk ke state, yang persis dilarang.

**Konsekuensi yang diterima.** Satu langkah manual pada penaikan pertama dan pada setiap penggantian kunci. Ditukar dengan jaminan bahwa berkas state — yang disalin, dicadangkan, dan dibaca lebih banyak orang daripada yang disadari — tidak pernah memuat satu pun kata sandi.

### CK-D-06 · 11 Agustus 2026 · Kata sandi master RDS dikelola RDS sendiri — mengamandemen CK-D-05 dan §5.1

**Diputuskan.** Instance RDS memakai `manage_master_user_password = true` dengan nama pengguna master `edutrack_owner`. RDS membangkitkan kata sandinya, menyimpannya sendiri di Secrets Manager, dan merotasinya setiap 7 hari; Terraform hanya membaca ARN-nya lewat atribut `master_user_secret`. Akibatnya **rahasia yang dibuat manusia berkurang menjadi dua** — `edutrack/db/app_rw` dan `edutrack/db/app_ro` — dan nama `edutrack/db/owner` pada §5.1 gugur.

**Alasan.** CK-D-05 menetapkan ketiga kredensial basis data dibuat manusia di luar Terraform. Untuk `app_rw` dan `app_ro` itu dapat berjalan: keduanya adalah role PostgreSQL biasa yang dibuat dengan `CREATE ROLE` sesudah instance menyala, sehingga Terraform memang tidak perlu mengetahui kata sandinya.

**Untuk role master, urutannya tidak dapat dibalik.** Kata sandi master bukan sesuatu yang disetel sesudah instance ada — ia adalah masukan pembuatan instance itu sendiri. Bentuk `aws_db_instance` yang lazim mewajibkan atribut `password`, sehingga satu-satunya cara memenuhi §5.1 apa adanya adalah menaruh kata sandi pemilik seluruh objek ke dalam berkas state. Itu persis yang dilarang [Techstack §7](Techstack.md) butir 1, dan pada kredensial yang **paling berat akibat kebocorannya** — pemegangnya dapat menjatuhkan seluruh tabel.

`manage_master_user_password` membalik keadaan itu tanpa melemahkan satu pun ketentuan: nilainya tidak pernah melewati Terraform, tidak pernah melewati mesin siapa pun, dan tidak pernah muncul pada baris perintah maupun riwayat shell. Tiga aturan pemasukan pada §5.1 karenanya terpenuhi secara mutlak pada rahasia ini, bukan sekadar dipatuhi.

**Rotasi tujuh hari adalah tambahan, bukan biaya.** [Techstack §7](Techstack.md) sudah menghendaki rotasi terjadwal bagi ketiga kredensial basis data dan menempatkannya pada Pasal 8. Untuk `edutrack_owner`, Pasal 8 kini tidak perlu menuliskan apa pun. Yang membuatnya aman adalah rahasia dibaca pada saat container menyala ([ARCHITECTURE §12.1](ARCHITECTURE.md)): fungsi `migrate` yang dipanggil enam puluh kali setahun selalu membaca versi yang berlaku saat itu, dan tidak pernah memegang kata sandi lama.

**Alternatif yang ditolak.**

*Master berupa role antara, `edutrack_owner` dibuat manusia lewat SQL.* Mempertahankan daftar tiga rahasia §5.1 apa adanya. Ditolak karena menambah satu role PostgreSQL yang tidak dipakai siapa pun kecuali untuk membuat ketiga role lain, satu rahasia lagi yang harus dirotasi Pasal 8, dan satu langkah manual lagi pada penaikan pertama — seluruhnya demi mempertahankan sebuah nama.

*`random_password` pada Terraform.* Sudah ditolak CK-D-05 dengan alasan yang sama, dan tetap ditolak di sini.

*Kata sandi master sementara yang segera diganti manusia.* Nilai pertamanya tetap masuk ke state, dan state menyimpan riwayat. Mengganti kata sandi sesudahnya tidak menghapus apa yang sudah tercatat.

**Konsekuensi yang diterima.** Nama rahasia `edutrack_owner` tidak lagi dapat disepakati di muka; ia berbentuk `rds!db-…` beserta akhiran acak, sehingga fungsi `migrate` menerima ARN-nya alih-alih namanya (§5.1). Kunci KMS-nya juga bukan `aws/secretsmanager` melainkan kunci terkelola RDS, sehingga kebijakan IAM `edutrack-lambda-migrate` menyebut ARN rahasia itu apa adanya dan tidak dapat ditulis sebagai pola nama.

### CK-D-07 · 11 Agustus 2026 · Alias `live` lahir menunjuk `$LATEST` — mengamandemen §2.2

**Diputuskan.** Pada pembuatan pertama, `aws_lambda_alias.live` menunjuk `$LATEST`. Nilai `"1"` pada §2.2 diganti. `ignore_changes = [function_version]` dan `publish = false` tetap berlaku tanpa perubahan.

**Alasan.** Kedua ketentuan §2.2 tidak dapat berlaku bersamaan. `publish = false` berarti Terraform tidak pernah menerbitkan version, sehingga fungsi yang baru dibuat hanya memiliki `$LATEST` — version bernomor 1 tidak pernah ada. `terraform apply` gagal pada pembuatan alias dengan keluhan bahwa versionnya tidak ditemukan, yaitu pada langkah 3 [§2.3](#23-pemisahan-bootstrap) tepat ketika seluruh infrastruktur lain sudah terlanjur dibuat.

Kekeliruannya berasal dari penyusunan §2.2 yang menuliskan kedua atribut dari sudut pandang keadaan **sesudah** rilis pertama, ketika version 1 memang sudah ada.

**Alternatif yang ditolak.**

*`publish = true` pada pembuatan pertama saja.* Tidak dapat dinyatakan: Terraform tidak mengenal atribut yang hanya berlaku sekali. Menyalakannya secara tetap membuat penomoran version menjadi rebutan dua sistem, yang justru dilarang §2.2.

*Menerbitkan version pertama dengan `aws_lambda_function_version` tersendiri.* Menambah satu sumber daya yang dimiliki Terraform di wilayah yang sengaja diserahkan kepada CI (CK-D-02), demi angka yang akan segera ditinggalkan rilis pertama.

**Konsekuensi yang diterima.** Di antara `terraform apply` dan rilis pertama, alias `live` menunjuk `$LATEST` — sehingga `update-function-code` mengubah apa yang dilayani alias seketika, tanpa menunggu `update-alias`. Jendela itu berumur satu kali jalan pipeline dan hanya ada sekali seumur lingkungan: sejak rilis pertama, alias menunjuk version bernomor dan tidak pernah kembali.

### CK-D-08 · 11 Agustus 2026 · Fungsi `migrate` dipilih variabel lingkungan `PERAN`, bukan penggantian perintah image

**Diputuskan.** Kedua fungsi menjalankan image dan perintah yang **sama persis**. Yang membedakannya satu variabel lingkungan, `PERAN`, bernilai `api` (bawaan) atau `migrasi`. `image_config` pada Terraform dibiarkan kosong. Ketika `PERAN=migrasi`, proses menyajikan satu jalur HTTP yang menerapkan migrasi dan melaporkan hasilnya, alih-alih menyalakan aplikasi.

**Alasan.** Lambda Web Adapter menuntut **aplikasi yang mendengarkan HTTP**. Ia berjalan sebagai extension, mengambil alih perulangan invocation, lalu meneruskannya sebagai request ke `127.0.0.1:8080` — dan tidak mengalirkan trafik sebelum readiness check `GET /healthz` lulus ([ARCHITECTURE.md Pasal 6](ARCHITECTURE.md)).

Perintah migrasi yang lazim — berjalan sekali lalu keluar — karenanya tidak dapat dipasang sebagai fungsi Lambda pada image ini. Ia tidak pernah mendengarkan, readiness tidak pernah lulus, dan prosesnya keluar sehingga Lambda melaporkan runtime yang berhenti tanpa alasan. Gejalanya muncul pada langkah 6 §3.3, yaitu tepat ketika pipeline seharusnya menerapkan skema.

**Mengganti perintah image lewat `image_config` juga tidak dapat dipakai**, dan sebabnya lebih halus: `image_config` adalah bagian dari **cangkang** yang dimiliki Terraform (§2.1), sehingga nilainya berlaku bagi image mana pun yang sedang terpasang — termasuk image `:bootstrap` yang dipakai saat fungsi dibuat. Perintah yang menunjuk berkas milik image aplikasi membuat fungsi `migrate` gagal menyala pada `terraform apply` pertama, sebelum ada satu pun rilis.

Variabel lingkungan tidak memiliki persoalan itu: image `:bootstrap` mengabaikannya, dan image aplikasi membacanya.

**Bahwa jalur migrasi berada di dalam image yang sama dengan jalur request bukan pelonggaran pemisahan.** Yang memisahkan keduanya tidak pernah berupa berkas yang berbeda, melainkan **kredensial**: fungsi `api` tidak dapat membaca rahasia `edutrack_owner` sekalipun kodenya mencoba, karena IAM menolaknya (§9.5). Pemisahan itu tetap utuh, dan justru inilah bentuk yang dirancang [Techstack §7](Techstack.md) ketika menempatkan kolom "dibaca oleh" sebagai bagian yang menentukan.

**Alternatif yang ditolak.**

*Entry point kedua khusus AWS.* Bertentangan langsung dengan [ARCHITECTURE.md §5.1](ARCHITECTURE.md), yang menyatakan `entry/` tidak memiliki entry point terpisah untuk AWS karena AWS menjalankan container yang sama dengan on-prem.

*Image kedua khusus migrasi.* Membatalkan janji satu image dua lingkungan, menggandakan pipeline build, dan memunculkan pertanyaan baru yang tidak punya jawaban baik: image mana yang di-tag git SHA.

*Menyingkirkan Lambda Web Adapter dari fungsi `migrate`.* Adapter berada di dalam image, bukan pada konfigurasi fungsi. Menyingkirkannya berarti image kedua.

**Konsekuensi yang diterima.** Fungsi `migrate` menyala sebagai server HTTP berumur pendek, dan penerapan migrasi terjadi ketika ia menerima invocation — bukan ketika ia menyala. Pemanggilannya sinkron (§3.3 langkah 6), sehingga hasilnya tetap terbaca pipeline apa adanya. Batas waktu fungsi `migrate` disetel lebih longgar daripada fungsi `api`, karena yang dibatasi keduanya adalah hal yang berbeda.

---

### CK-D-09 · 13 Agustus 2026 · Lingkungan dapat dibongkar dan dibangun kembali dengan dua skrip — mengamandemen §2.5

**Persoalannya biaya, bukan kerapian.** Akun ini ditagih per jam atas sumber daya yang berdiri, dan pembuatannya sendiri tidak berbiaya. Lingkungan yang menganggur karenanya lebih murah dibongkar daripada dibiarkan hidup — asalkan membangunnya kembali dapat dipercaya, sebab pembongkaran yang tidak dapat dibalik bukan penghematan melainkan kehilangan.

Dua skrip pada repositori `infra`: `skrip/turunkan.sh` dan `skrip/naikkan.sh`.

**Yang menahan biaya sesungguhnya hanya dua sumber daya.** RDS `db.t4g.micro` beserta 20 GB gp3 dan NAT instance `t4g.micro` beserta EIP-nya menanggung sekitar 70% dari $32,41 per bulan pada [Techstack §8.2](Techstack.md). CloudFront, Lambda, S3, dan ECR pada trafik sekarang mendekati nol. NAT Gateway — pemborosan terbesar yang lazim — memang tidak pernah dipakai di sini.

**Cadangan berupa `pg_dump`, bukan snapshot akhir.** Keduanya sama-sama menyelamatkan data, tetapi hanya satu yang memenuhi maksud pembongkaran:

| | `pg_dump` | Snapshot RDS |
|---|---|---|
| Dapat dipulihkan ke | Postgres mana pun, termasuk Docker setempat | hanya instance RDS baru, di akun dan region yang sama |
| Biaya sesudah pembongkaran | nol — berkasnya di mesin operator | ~$0,10 per bulan, bertahan sampai dihapus sendiri |
| Nama | bebas | tetap, sehingga pembongkaran kedua gagal |

Snapshot karenanya justru bertentangan dengan tujuannya: ia adalah sisa yang tetap ditagih. `skip_final_snapshot` disetel `true` bersama `izinkan_hapus`.

**Sidik jari mendahului dump.** Dua dump dari data yang sama tidak menghasilkan berkas yang sama — format `-Fc` terkompresi dan memuat stempel waktu — sehingga membandingkan berkas dump selalu berkata "berubah", dan jawabannya baru diperoleh sesudah datanya terlanjur ditransfer. Yang dibandingkan karenanya **basis datanya**, lewat satu query `count(*)` dan `md5` agregat per tabel yang keluarannya sekitar 200 byte. Hasilnya disimpan berdampingan sebagai `edutrack.sidik`; dump dilewati seluruhnya apabila sidiknya sama.

Biayanya sendiri bukan alasannya — dump sebesar 5 MB berharga di bawah sepersepuluh sen, sedangkan RDS menyala satu jam lebih mahal daripada seribu dump. Yang dihindari adalah pekerjaan yang tidak perlu, bukan tagihannya.

**`pg_dump` tidak membawa role.** `app_rw` dan `app_ro` adalah objek tingkat cluster, dan `edutrack_owner` bukan superuser sehingga tidak dapat mengekspornya. Urutan pemulihan karenanya mengikat: **role dibuat lebih dahulu, dump dipulihkan sesudahnya.** Terbalik, seluruh `GRANT` di dalam dump gagal.

**Tiga hal sengaja tidak ikut dihapus, dan ketiganya berbiaya nol.** OIDC provider GitHub dibaca `iam-oidc.tf` sebagai *data source* — ia dibuat dengan tangan pada B0.5 dan tidak pernah dimiliki Terraform. Menghapusnya membuat `plan` pada pembangunan berikutnya gagal karena data source-nya tidak ditemukan, dan memulihkannya adalah titik henti manusia ([AGENTS §10](AGENTS.md), [RUNBOOK-OIDC.md](RUNBOOK-OIDC.md)). Bersamanya tetap berdiri user `Andreas`, grup `Edutrack-dev`, dan role `edutrack-terraform` — tanpa ketiganya tidak ada yang dapat meminjam apa pun.

**Domain CloudFront berubah setiap pembangunan.** `ALAMAT_PUBLIK` pada repositori backend karenanya diperbarui `naikkan.sh` lewat `gh`. Terlewat, langkah 11 §3.3 tetap lulus sambil menguji alamat yang sudah mati.

**Tunnel SSM menuntut jalan masuk yang dibuka sebentar.** `edutrack-rds` hanya menerima dari security group Lambda, dan NAT instance memakai security group tersendiri. Port forwarding lewatnya karenanya tersambung di sisi lokal — `session-manager-plugin` mendengarkan, `psql` terhubung kepadanya — tetapi tidak pernah sampai ke RDS: paketnya dibuang **tanpa penolakan**, sehingga `psql` menunggu selamanya alih-alih gagal. Gejalanya berupa skrip yang berhenti pada langkah pemeriksaan sidik jari tanpa satu pun pesan.

Kedua skrip karenanya membuka aturan itu sebelum tunnel dan mencabutnya lewat `trap`. Ia sengaja **tidak** dijadikan sumber daya Terraform: jalan masuk yang berdiri tetap melemahkan postur yang justru menjadi maksud subnet privat-data. Aturan yang sudah ada sebelumnya tidak pernah ikut dicabut — mencabut milik orang lain adalah perubahan yang tidak diminta. `connect_timeout=15` dipasang pada seluruh koneksi psql supaya kesunyian semacam ini berubah menjadi kegagalan yang terbaca dalam 15 detik.

**Konsekuensi yang diterima.** Pembongkaran memakan 35–45 menit dan pembangunan 25–35 menit, hampir seluruhnya menunggu CloudFront dan pelepasan ENI Lambda. Nama bucket S3 bersifat global, sehingga membuat ulang bucket state sesaat setelah menghapusnya kadang ditolak sampai propagasinya selesai; `naikkan.sh` mencoba ulang berjeda dan melaporkannya terang-terangan, karena jalan keluarnya hanya menunggu.

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 13 Agustus 2026 | **CK-D-09** — lingkungan dapat dibongkar dan dibangun kembali dengan `turunkan.sh` dan `naikkan.sh`. §2.5 diamandemen: `prevent_destroy` kini berpintu, lewat patch yang ditinjau, sedangkan keempat pengaman lain dibuka variabel `izinkan_hapus`. Cadangan berupa `pg_dump` yang didahului perbandingan sidik jari, bukan snapshot akhir |
| 12 Agustus 2026 | §9.5 menambahkan **`s3:ListBucket`** pada `edutrack-lambda-api`. Aplikasi tidak pernah mendaftar isi bucket; yang menuntutnya adalah perilaku S3 pada objek yang belum ada — tanpa izin itu `HeadObject` menjawab `403` alih-alih `404`, sehingga setiap perenderan gagal sebelum satu pun berkas dibuat. Ditemukan saat B7 dijalankan, dengan gejala `berkas_terender: 0` tanpa satu pun pesan yang menyebut S3 |
| 12 Agustus 2026 | §9.4 melengkapi izin role OIDC dengan `lambda:GetFunctionConfiguration` dan `lambda:GetAlias`. Keduanya tidak pernah didaftar karena Pasal 9 ditulis mendahului §3.3, sedangkan yang menuntutnya adalah `aws lambda wait function-updated` dan pembacaan version untuk rollback. Ditemukan ketika rilis pertama gagal pada langkah 5 |
| 12 Agustus 2026 | **§5.1 dikoreksi setelah dijalankan untuk pertama kalinya.** Tiga cacat: contohnya memakai `create-secret` padahal wadahnya sudah dibuat Terraform; kata sandinya dibangkitkan di dalam `printf` sehingga tidak dapat dipakai ulang untuk `ALTER ROLE`; dan `.pgpass` **tidak dapat dipakai** karena kata sandi bangkitan RDS dapat memuat titik dua, yaitu pemisah bidang formatnya. Ditambahkan langkah verifikasi yang benar-benar mencoba masuk, bukan sekadar memeriksa keberadaan versi |
| 12 Agustus 2026 | **§5.2 ditulis — B4 selesai.** Penandatanganan OAC atas request ber-body dibuktikan; hasilnya **CK-A-12** pada [ARCHITECTURE.md](ARCHITECTURE.md). Dicatat dua jebakan yang ditemui saat menaikkannya: OAC menuntut **dua** izin Lambda (`InvokeFunctionUrl` **dan** `InvokeFunction`), dan `custom_error_response` berlaku se-distribusi sehingga merusak kontrak amplop `kesalahan` pada `/api/*` |
| 11 Agustus 2026 | §3.3 langkah 11 dikoreksi dari `/healthz` menjadi `/api/healthz`. CloudFront hanya meneruskan `/api/*` ke Lambda, sehingga bentuk semula dilayani bucket frontend dan selalu lulus tanpa memeriksa apa pun. Ditemukan saat `infra/` dikodekan |
| 11 Agustus 2026 | **CK-D-08** — kedua fungsi menjalankan image dan perintah yang sama persis; yang membedakannya variabel lingkungan `PERAN`. Lambda Web Adapter menuntut aplikasi yang mendengarkan HTTP, sehingga perintah migrasi yang berjalan sekali lalu keluar tidak dapat dipasang sebagai fungsi. `image_config` juga tidak dapat dipakai karena ia bagian dari cangkang yang berlaku bagi image `:bootstrap` sekalipun |
| 11 Agustus 2026 | **CK-D-07** — nilai awal `function_version` pada alias `live` diamandemen dari `"1"` menjadi `"$LATEST"`. Keduanya tidak dapat berlaku bersamaan dengan `publish = false`, karena fungsi yang baru dibuat tidak memiliki version bernomor untuk ditunjuk. §2.2 disesuaikan |
| 11 Agustus 2026 | **CK-D-06** — kata sandi master RDS dikelola RDS sendiri lewat `manage_master_user_password`, sehingga rahasia yang dibuat manusia berkurang menjadi dua dan nama `edutrack/db/owner` gugur. §5.1 dan §9.5 disesuaikan. Sebabnya urutan: kata sandi master adalah masukan pembuatan instance, bukan sesuatu yang disetel sesudahnya, sehingga §5.1 sebagaimana ditulis semula tidak pernah dapat berjalan tanpa melanggar [Techstack §7](Techstack.md) butir 1 |
| 11 Agustus 2026 | §9.9 diperbarui dari keadaan 6 Agustus. Role `edutrack-terraform` dan jalur OIDC terbukti ada; dicatat pula bahwa `edutrack-gha-backend` dibuat dengan tangan sehingga wajib di-`import` pada B5, bukan dibuat ulang |
| 11 Agustus 2026 | **§5.1 dan CK-D-05** — nama keempat rahasia, bentuk nilainya, dan prosedur pengisiannya ditetapkan. Parameter SSM dinyatakan berada di luar Terraform seluruhnya, karena `aws_ssm_parameter` mewajibkan `value` sehingga tidak ada cara membuatnya tanpa nilainya masuk ke state |
| 11 Agustus 2026 | §9.6 dikoreksi. Baris `AWS_PROFILE=edutrack terraform apply` **tidak pernah dapat berjalan**: profil ber-`mfa_serial` menuntut prompt yang tidak dimiliki Terraform. Digantikan `aws configure export-credentials`, beserta penjelasan sebabnya. Ditemukan saat `terraform apply` pertama pada `bootstrap/` |
| 11 Agustus 2026 | **CK-D-04** — penguncian state berpindah ke mekanisme bawaan S3 (`use_lockfile`), tabel DynamoDB tidak dibuat. §2.3 disesuaikan. Ditulis sebelum `bootstrap/` dikodekan, karena `dynamodb_table` sudah usang sejak Terraform 1.11 |
| 6 Agustus 2026 | Kerangka dibuat sebagai bagian dari pemecahan `Techstack.md` menjadi tiga dokumen. Isi belum ditulis |
| 6 Agustus 2026 | **Versi 0.2 — Pasal 9 Identitas dan akses ditulis.** Ditetapkan dua IAM user bernama orang, satu grup `Edutrack-dev` dengan dua customer managed policy, dan tujuh role: dua dipinjam manusia, dua dipinjam GitHub Actions lewat OIDC, dan tiga untuk fungsi Lambda serta NAT instance. Seluruh jalur mesin tanpa access key. Pasal ini ditulis mendahului pasal lain karena Terraform tidak dapat dijalankan tanpanya. Lampiran Catatan Keputusan dibuka dengan **CK-D-01**. Dicatat pula keadaan penerapan per 6 Agustus 2026 pada §9.9 |
| 7 Agustus 2026 | **Versi 0.3 — Pasal 2, 3, dan 6 ditulis.** Ditetapkan **skema B** (**CK-D-02**): Terraform memiliki cangkang fungsi, CI memiliki isinya, dan `ignore_changes` dipasang di **dua** tempat — `image_uri` pada fungsi dan `function_version` pada alias. Yang kedua ditemukan belakangan dan lebih berbahaya, karena memindahkan alias adalah tindakan rilis itu sendiri. Ditolak: tag `:latest`, skema A, dan pemecahan Terraform menjadi `app/`. Dicatat delapan lubang yang diketahui beserta penutupnya. Pasal 6 menetapkan lima aturan migrasi kompatibel mundur, yang wajib berlaku sebelum migrasi 0001 ditulis. Nama repositori pada §9.4 dikoreksi menjadi `Korean-Asean-Digital-Academy-Batch-4/backend` |
| 7 Agustus 2026 | Ditambahkan **§6.5 Enam lapis penjagaan** beserta **CK-D-03**: header klasifikasi, penamaan `expand`/`contract`, linter `squawk`, konfirmasi pemakai sebelum `contract`, tes rilis sebelumnya terhadap skema baru, dan latihan rollback sungguhan. Tiga lapis di antaranya lahir dari peninjauan kumpulan skill data engineering pihak ketiga. Ditolak: compatibility view, dual write, dan migrasi turun |
| 7 Agustus 2026 | Region pada §9.6 disesuaikan menjadi `ap-southeast-3` mengikuti **CK-16** pada [Techstack.md](Techstack.md) |
