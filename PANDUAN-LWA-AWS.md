# Panduan Penerapan — Express di Lambda Web Adapter, CloudFront, dan RDS

| Keterangan | Isi |
|---|---|
| **Tanggal** | 12 Agustus 2026 |
| **Kedudukan** | **Bukan dokumen EduTrack.** Ia panduan lintas proyek, disarikan dari penaikan EduTrack, dan dimaksudkan **disalin ke proyek lain** yang memakai susunan serupa |
| **Berlaku bila** | Aplikasi HTTP di container Lambda + Lambda Web Adapter · CloudFront + Origin Access Control di depan Function URL · RDS di subnet privat · Terraform · GitHub Actions OIDC |
| **Sumbernya** | Setiap butir di bawah **benar-benar terjadi**, bukan disusun dari dokumentasi. Yang tidak pernah menggigit tidak ditulis di sini |

> **Cara membaca dokumen ini.** Ia disusun menurut **gejala**, bukan menurut layanan — karena yang Anda punya ketika terjebak adalah gejalanya. Setiap butir berbentuk: apa yang terlihat, apa sebabnya, apa perbaikannya.
>
> Yang paling mahal dari seluruh penaikan bukan hal yang sulit, melainkan hal yang **pesan galatnya menunjuk ke arah yang salah**. Butir-butir itu ditandai ⚠️.

---

## 1. AWS Free Plan — pagar, bukan potongan

Sejak 2025 akun baru memilih **Free Plan** atau **Paid Plan**. Perbedaannya tidak seperti free tier lama:

| | Free tier lama | **Free Plan** |
|---|---|---|
| Sifat | Potongan tagihan | **Pagar** |
| Di luar daftar | Ditagih | **Ditolak API** |

### Yang benar-benar diblokir

Hanya **konfigurasi**, bukan layanannya:

| Ditolak | Pesan | Sebab sesungguhnya |
|---|---|---|
| `t4g.nano` untuk NAT | `InvalidParameterCombination` | Tipe tidak ada pada daftar eligible region itu |
| `backup_retention_period = 7` | `FreeTierRestrictionError` | Melampaui batas; galatnya **tidak menyebut angka maksimumnya** |

Amazon RDS dan Amazon EC2 sendiri **tidak** terlarang. Periksa daftarnya, jangan tebak — ia berbeda antar region:

```bash
aws ec2 describe-instance-types --filters Name=free-tier-eligible,Values=true --query 'InstanceTypes[].InstanceType' --output text --region <region>
```

### Yang lolos meski bukan free tier

**Secrets Manager.** Ia berbiaya $0,40 per rahasia dan tetap dibuat tanpa penolakan. Kesimpulannya: Free Plan memblokir hal tertentu dan **menggerus kredit** untuk sisanya. Jangan mengasumsikan seluruh rancangan gugur hanya karena satu penolakan.

### Plafon concurrency Lambda = 10

Akibatnya `reserved_concurrent_executions` **tidak dapat disetel sama sekali**:

```
InvalidParameterValueException: … decreases account's UnreservedConcurrentExecution
below its minimum value of [10]
```

Reservasi **1** pun ditolak. Yang tersedia hanya `-1`, yaitu tidak menyetelnya.

Remnya berpindah ke plafon akun — dan itu **lebih ketat**, bukan lebih longgar. Bahayanya bukan sekarang melainkan nanti: plafon akun tidak tertulis di repositori mana pun, dan AWS dapat menaikkannya tanpa satu pun commit. **Catat pemicu peninjauannya**: pasang kembali reservasi begitu plafon mencapai 50 (di bawah itu reservasinya ditolak karena menyisakan kurang dari 10).

```bash
aws lambda get-account-settings --query 'AccountLimit.ConcurrentExecutions'
```

### ⚠️ Bulan ke-13

Cakupan 12 bulan berakhir **bersamaan** untuk RDS, EC2, dan alamat IPv4. Tagihan berpindah dari nol ke angka penuh dalam satu siklus. Pasang AWS Budgets sebelum bulan itu, bukan sesudahnya.

---

## 2. Lambda Web Adapter

### Bentuk yang bekerja

```dockerfile
FROM node:24-slim
# Versi WAJIB dipatok. Variabel tanpa prefiks AWS_LWA_ sudah usang.
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:1.0.1 /lambda-adapter /opt/extensions/lambda-adapter
ENV PORT=8080
ENV AWS_LWA_READINESS_CHECK_PATH=/healthz
CMD ["node", "dist/entry/server.js"]
```

Adapter berjalan sebagai extension, mengambil alih perulangan invocation, dan meneruskannya sebagai HTTP ke `127.0.0.1:8080`. **Aplikasi tidak mengetahui keberadaan Lambda** — tidak ada handler, tidak ada `event`.

### Konsekuensi yang menentukan: aplikasi WAJIB mendengarkan

Perintah sekali-jalan-lalu-keluar — migrasi, seeding, cron — **tidak dapat dipasang sebagai fungsi** pada image ini. Prosesnya keluar, readiness tidak pernah lulus, dan Lambda melaporkan runtime yang berhenti tanpa alasan.

### ⚠️ Satu image, dua fungsi: bedakan lewat variabel lingkungan, BUKAN `image_config`

Godaannya menyetel `image_config { command = [...] }` pada fungsi kedua. Jangan.

`image_config` adalah bagian dari **cangkang** yang dimiliki Terraform, sehingga berlaku bagi **image apa pun yang sedang terpasang** — termasuk image bootstrap yang dipakai saat fungsi pertama kali dibuat. Perintah yang menunjuk berkas milik image aplikasi membuat fungsi itu gagal menyala pada `terraform apply` pertama, **sebelum ada satu pun rilis**.

Yang bekerja: satu variabel lingkungan, misalnya `PERAN=api|migrasi`, dibaca entry point yang sama. Image bootstrap mengabaikannya; image aplikasi membacanya.

Untuk fungsi yang dipanggil `lambda:invoke` biasa (bukan HTTP), adapter meneruskan payload sebagai `POST` ke `AWS_LWA_PASS_THROUGH_PATH` (bawaan `/events`). Setel eksplisit, dan layani jalur itu.

### Image bootstrap harus TERPISAH dari image aplikasi

Fungsi tidak dapat dibuat tanpa image, dan image aplikasi biasanya menuntut konfigurasi yang belum ada pada `apply` pertama — URL basis data, kunci API. Containernya mati sebelum mendengarkan, readiness tidak pernah lulus, dan `apply` gagal pada langkah yang justru dipasang untuk membuktikan infrastrukturnya sehat.

Buat image minimal tanpa dependensi dan **tanpa satu pun variabel lingkungan yang wajib**. Sekalian pasang jalur diagnostik di dalamnya — lihat §3.

---

## 3. CloudFront + OAC + Function URL

Bagian tersulit dari seluruh susunan. Empat jebakan, dan tiga di antaranya pesan galatnya menyesatkan.

### ⚠️ 3.1 OAC menuntut DUA izin Lambda, bukan satu

**Gejala:** `403` pada **setiap** request lewat CloudFront, termasuk `GET` tanpa body, dengan kebijakan yang tampak persis benar di layar.

```hcl
resource "aws_lambda_permission" "cf_url" {
  action = "lambda:InvokeFunctionUrl"   # ← ini saja TIDAK CUKUP
  ...
}
resource "aws_lambda_permission" "cf_invoke" {
  action = "lambda:InvokeFunction"      # ← yang ini selalu terlupa
  ...
}
```

**Yang membuatnya memakan waktu berjam-jam:** menguji dengan `curl --aws-sigv4` memakai kredensial sendiri **lolos**, sehingga Function URL tampak sehat dan kecurigaan berpindah ke CloudFront. Sebabnya profil administrator memiliki **kedua** izin, sementara principal CloudFront hanya diberi satu.

> **Pelajaran umum:** menguji dengan kredensial manusia menyesatkan ketika kredensial itu jauh lebih berkuasa daripada pemanggil yang sesungguhnya.

### 3.2 Membedakan dua bentuk `403` dalam hitungan detik

| Badan jawaban | Artinya |
|---|---|
| `{"Message":"Forbidden"}` | Tidak ada tanda tangan sama sekali |
| `{"Message":"Forbidden. For troubleshooting Function URL authorization issues, see: …"}` | Tanda tangan **ada** dan ditolak — persoalannya izin atau sidik jari payload |

Perbedaan ini memisahkan "OAC tidak menandatangani" dari "otorisasi menolak", dan menghemat berjam-jam.

### ⚠️ 3.3 Request ber-body menuntut `x-amz-content-sha256`

**Gejala paling berbahaya di seluruh dokumen ini:** seluruh jalur **baca** berjalan sempurna, dan hanya jalur **tulis** yang mati.

Lambda Function URL **tidak menerima `UNSIGNED-PAYLOAD`**. OAC mengambil sidik jari payload dari header itu alih-alih menghitungnya sendiri. Kewajiban jatuh pada **pengirim**:

```js
const sidik = [...new Uint8Array(await crypto.subtle.digest("SHA-256", new TextEncoder().encode(badan)))]
  .map((b) => b.toString(16).padStart(2, "0")).join("");
```

**Akibat terberatnya ada pada unggahan.** `FormData` disusun peramban beserta *boundary* yang dibangkitkannya sendiri, sehingga bita persisnya tidak dapat diketahui dan sidik jarinya tidak dapat dihitung. Multipart **wajib disusun sendiri** sebagai `Blob`, dan sidik jarinya diambil dari `Blob` yang sama yang akan dikirim.

### 3.4 Origin request policy wajib `AllViewerExceptHostHeader`

Function URL menolak request yang header `Host`-nya menunjuk CloudFront. Penolakannya `403` dan **tidak menyebut `Host` sama sekali**.

### ⚠️ 3.5 `custom_error_response` berlaku SE-DISTRIBUSI

Dipasang untuk perutean SPA, dan **tidak dapat dibatasi pada satu cache behavior** — CloudFront hanya mengenal error response tingkat distribusi.

Akibatnya setiap `403` dan `404` dari `/api/*` dibelokkan ke `/index.html`: kontrak API rusak (404 menjadi 200 berisi HTML), dan galat yang sesungguhnya tertutup jawaban bucket frontend. Perutean SPA diselesaikan lewat CloudFront Function pada perilaku bawaan saja.

### 3.6 Hanya `/api/*` yang sampai ke Lambda

Jalur pemeriksaan kesehatan harus berada di bawah awalan itu. `/healthz` di akar akan dilayani bucket frontend dan menjawab `200` berisi `index.html` — pemeriksaan yang **selalu lulus dan karenanya tidak memeriksa apa pun**.

Sediakan dua alamat: `/healthz` untuk readiness adapter (dari dalam container), `/api/healthz` untuk pemeriksaan dari luar.

---

## 4. Terraform

### ⚠️ 4.1 `count` tidak boleh menunjuk sumber daya lain

```hcl
count = length(aws_subnet.publik)   # SALAH
count = length(var.zona)            # BENAR
```

Jumlahnya tidak diketahui saat plan, dan Terraform menolak **seluruh perintah** yang mengevaluasi konfigurasi — termasuk `import` yang tidak berhubungan sama sekali.

**`terraform validate` TIDAK menangkap ini.** Hanya `plan` yang menangkapnya, dan `plan` menuntut kredensial. Tutup dengan pemeriksaan sendiri:

```bash
grep -rnE '(count|for_each)\s*=.*\baws_' *.tf && echo "count/for_each menunjuk sumber daya — perbaiki"
```

### 4.2 Dua `ignore_changes`, bukan satu

Ketika CI memiliki isi fungsi dan Terraform memiliki cangkangnya:

```hcl
resource "aws_lambda_function" "api" {
  publish   = false
  lifecycle { ignore_changes = [image_uri] }
}
resource "aws_lambda_alias" "live" {
  function_version = "$LATEST"                      # BUKAN "1" — lihat 4.3
  lifecycle { ignore_changes = [function_version] } # yang ini lebih berbahaya bila lupa
}
```

Melupakan yang kedua membuat `apply` untuk urusan **yang sama sekali tidak berhubungan** mengembalikan alias ke image bootstrap. Seluruh aplikasi lenyap, dan penyebabnya perintah yang tampaknya tidak menyentuh aplikasi.

### 4.3 Alias tidak dapat menunjuk version `"1"` bersama `publish = false`

Fungsi yang baru dibuat hanya memiliki `$LATEST`. `apply` gagal pada pembuatan alias — yaitu **sesudah** seluruh infrastruktur lain terlanjur dibuat.

### 4.4 `apply_method` pada parameter statis basis data

Parameter statis disetel RDS menjadi `pending-reboot`. Konfigurasi yang tidak menyebutkannya berselisih pada **setiap** `plan`. Selisih abadi melumpuhkan deteksi drift — orang berhenti membaca `plan` yang tidak pernah bersih.

```hcl
parameter { name = "rds.force_ssl"  value = "1"  apply_method = "pending-reboot" }
```

### 4.5 Parameter group: `name_prefix`, bukan `name`

Dengan `create_before_destroy`, nama tetap membuat penggantian gagal karena namanya masih terpakai.

---

## 5. Rahasia — cara yang tidak gagal

### 5.1 Aturan yang mengikat

**Nilai rahasia tidak pernah dibuat Terraform.** Yang masuk ke state adalah ARN, bukan isinya.

| Layanan | Boleh dikelola Terraform? |
|---|---|
| `aws_secretsmanager_secret` | ✅ **wadahnya saja** |
| `aws_secretsmanager_secret_version` | ❌ inilah yang menaruh kata sandi ke state |
| `aws_ssm_parameter` | ❌ **mewajibkan `value`** — tidak ada bentuk penulisan yang membuatnya lahir tanpa nilai. `ignore_changes` tidak menolong: ia hanya mengabaikan perubahan **sesudah** nilai pertama tertulis |

### 5.2 Kata sandi master RDS: serahkan kepada RDS

```hcl
manage_master_user_password = true
```

Kata sandi master adalah **masukan pembuatan instance**, bukan sesuatu yang disetel sesudahnya. Satu-satunya cara membuatnya sendiri adalah menyerahkannya kepada Terraform lebih dahulu — yaitu memasukkan kredensial paling berkuasa ke dalam state.

Dengan `manage_master_user_password`, RDS membangkitkan, menyimpan, dan merotasi sendiri; ARN-nya dibaca lewat `master_user_secret[0].secret_arn`. **Namanya tidak dapat disepakati di muka** (`rds!db-…` beserta akhiran acak), sehingga aplikasi menerima ARN, bukan nama.

### ⚠️ 5.3 Bangkitkan SEKALI, tulis ke DUA tempat dari variabel yang sama

```bash
SANDI=$(openssl rand -base64 24 | tr -d '\n=/+')
# → ALTER/CREATE ROLE di PostgreSQL
# → put-secret-value ke Secrets Manager
```

Membangkitkannya dua kali menghasilkan dua nilai berbeda, dan kegagalannya baru muncul pada rilis pertama sebagai `password authentication failed`.

### 5.4 `put-secret-value`, bukan `create-secret`

Wadahnya sudah dibuat Terraform; `create-secret` dijawab `ResourceExistsException`.

### ⚠️ 5.5 `.pgpass` rusak oleh kata sandi bangkitan RDS

`.pgpass` memakai **titik dua sebagai pemisah bidang**, dan kata sandi bangkitan RDS dapat memuat titik dua. Barisnya terurai salah **tanpa satu pun peringatan**, dan `psql` menjawab `password authentication failed` — pesan yang menyesatkan ke arah kredensial yang keliru, padahal formatnya yang rusak.

Pakai `PGPASSWORD`, yang tidak memiliki format sama sekali:

```bash
PGPASSWORD="$(cat "$tmp/sandi")" psql "host=127.0.0.1 port=15432 dbname=… user=… sslmode=require"
```

### 5.6 Verifikasi yang sesungguhnya: coba masuk

Memeriksa keberadaan versi rahasia **tidak membuktikan apa pun** tentang kecocokannya dengan basis data. Yang membuktikannya hanya satu:

```bash
aws secretsmanager get-secret-value --secret-id … --query SecretString --output text | jq -r .password > "$tmp/uji"
PGPASSWORD="$(cat "$tmp/uji")" psql "…user=app_rw sslmode=require" -qtAc 'SELECT 1'
```

Tanpa ini, selisih antara kedua tempat baru ketahuan **sesudah migrasi terlanjur diterapkan**.

### ⚠️ 5.7 Tulis berkas kata sandi SESUDAH basis data berhasil

Urutan sebaliknya menghasilkan keadaan terburuk dari keduanya: perintah gagal dengan "nama pengguna sudah dipakai", sementara berkasnya **sudah terlanjur menimpa** kata sandi yang benar dengan kata sandi yang tidak cocok dengan akun mana pun. Akunnya menjadi tidak dapat dimasuki tanpa satu pun pesan yang menyebutkannya.

Dan: **satu berkas per akun**, bukan satu nama yang dipatok. Nama tetap membuat pembuatan akun kedua menimpa kata sandi akun pertama.

---

## 6. Menjangkau RDS di subnet privat

### 6.1 Security group tidak mengizinkan Anda

RDS hanya menerima dari security group aplikasi. NAT instance — satu-satunya perantara — **tidak termasuk**. Perlu aturan sementara, dan **wajib dicabut**:

```bash
aws ec2 authorize-security-group-ingress --group-id <sg-rds> --protocol tcp --port 5432 --source-group <sg-nat>
# … kerjakan …
aws ec2 revoke-security-group-ingress   --group-id <sg-rds> --protocol tcp --port 5432 --source-group <sg-nat>
```

⚠️ Aturan ini dibuat **di luar Terraform**, sehingga `terraform plan` tidak akan pernah mengingatkan apabila tertinggal terbuka. Bungkus seluruh prosedur dalam skrip ber-`trap` supaya pencabutannya terjadi juga ketika gagal di tengah.

### 6.2 Terowongan tanpa SSH

Beri NAT instance `AmazonSSMManagedInstanceCore`; tidak perlu kunci SSH sama sekali.

```bash
aws ssm start-session --target <id-instance> \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["<endpoint-rds>"],"portNumber":["5432"],"localPortNumber":["15432"]}'
```

Menuntut `session-manager-plugin` di mesin operator.

### 6.3 `sslmode` berbeda antara libpq dan node-postgres

| Klien | Tanpa verifikasi nama host |
|---|---|
| `psql` (libpq) | `sslmode=require` — **`no-verify` ditolak** sebagai nilai tidak sah |
| `pg` (node-postgres) | `sslmode=no-verify` |

Lewat terowongan, host-nya `127.0.0.1` sehingga nama pada sertifikat RDS tidak akan pernah cocok. Ini **hanya** untuk jalur pemeliharaan; aplikasi tetap memverifikasi — lihat 6.4.

### ⚠️ 6.4 `sslmode=require` pada `pg` BUKAN berarti tanpa verifikasi

`pg-connection-string` memperlakukan `require` sebagai **`verify-full`**. Gejalanya:

```
self-signed certificate in certificate chain
```

Verifikasinya sudah menyala sejak awal; yang tidak ada adalah CA-nya. Salin bundel CA Amazon RDS ke dalam image dan percayai lewat variabel lingkungan:

```dockerfile
COPY certs/rds-global-bundle.pem ./certs/
```
```hcl
NODE_EXTRA_CA_CERTS = "/app/certs/rds-global-bundle.pem"
```

**Patok `sslmode=verify-full` secara eksplisit.** `pg` memperingatkan bahwa pada v9 arti `require` akan **melemah** mengikuti libpq — konfigurasi yang bersandar pada arti lama akan kehilangan verifikasinya pada peningkatan pustaka, tanpa satu pun gejala.

Bundel **disalin ke repositori**, bukan diunduh saat build: unduhan saat build menambah ketergantungan jaringan pada setiap rilis dan satu permukaan rantai pasok yang tidak terlihat pada diff.

### 6.5 `s3:ListBucket` diperlukan meski Anda tidak pernah mendaftar isi bucket

**Gejala:** setiap penyimpanan berkas gagal, tanpa satu pun pesan yang menyebut S3.

Tanpa `s3:ListBucket`, `HeadObject` pada objek yang **belum ada** menjawab `403` alih-alih `404` — S3 menolak membocorkan keberadaan objek kepada pemanggil yang tidak boleh mendaftarnya. Kode yang memeriksa keberadaan berkas lebih dahulu memperlakukan `403` itu sebagai galat sungguhan.

Sasarannya bucket itu sendiri, **tanpa `/*`**.

---

## 7. CI/CD lewat OIDC

### 7.1 `sub` bisa saja bukan format baku

Organisasi yang menyalakan **custom subject claim** mengirim `sub` yang memuat id numerik:

```
repo:<ORG>@<id-org>/<repo>@<id-repo>:ref:refs/heads/main
```

Trust policy berformat baku tidak akan pernah cocok, dan AWS menolak tanpa petunjuk. Baca nilai aktualnya dari log diagnostik workflow **sebelum** menulis trust policy:

```yaml
- run: |
    TOKEN=$(curl -sS -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | jq -r '.value')
    echo "$TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq '{sub, aud}'
```

Pakai `StringEquals` dengan nilai utuh. `repo:<ORG>/*` membuat repositori mana pun di organisasi itu dapat menerapkan ke produksi.

### ⚠️ 7.2 Izin yang tidak terlihat dari perintah workflow

`aws lambda wait function-updated` memanggil **`lambda:GetFunctionConfiguration`** berulang — bukan `GetFunction`. Ia tidak muncul sebagai perintah tersendiri pada workflow, sehingga mudah luput dari daftar izin.

Yang diperlukan satu pipeline rilis lengkap:

```
ecr:GetAuthorizationToken            (resource "*", tidak dapat dibatasi)
ecr:* push pada satu repositori
lambda:UpdateFunctionCode
lambda:PublishVersion
lambda:UpdateAlias
lambda:GetFunction
lambda:GetFunctionConfiguration      ← waiter
lambda:GetAlias                      ← membaca version untuk rollback
lambda:InvokeFunction
```

**Yang justru penting adalah yang TIDAK ada:** `CreateFunction`, `UpdateFunctionConfiguration`, `DeleteFunction`, dan `iam:PassRole`. Perhatikan `GetFunctionConfiguration` diizinkan sedangkan `UpdateFunctionConfiguration` tidak — namanya nyaris sama, dan perbedaannya persis batas kepemilikan.

### 7.3 Dua `wait` yang paling mudah terlupa

`update-function-code` menjawab **sebelum** AWS selesai memasang image. Menerbitkan version terlalu cepat menghasilkan version yang membeku pada image **lama**, dan gejalanya rilis yang tampak berhasil tetapi tidak mengubah apa pun.

### 7.4 Invocation yang "berhasil" tidak berarti pekerjaannya berhasil

`aws lambda invoke` mengembalikan `StatusCode: 200` selama fungsinya tidak melempar. Pekerjaan yang gagal — misalnya migrasi — dilaporkan **di dalam payload**. Periksa keduanya:

```bash
jq -e '.FunctionError' ringkasan.json && exit 1
jq -e '(if has("body") then (.body|fromjson? // {}) else . end) | .data.berhasil == true' jawaban.json || exit 1
```

### 7.5 Membangun arm64 dari runner x86

```yaml
- uses: docker/setup-qemu-action@v3
- uses: docker/build-push-action@v6
  with:
    platforms: linux/arm64
    provenance: false   # Lambda menolak manifest list, dan pesannya tidak menyebut attestation
```

Tag = **git SHA**, bukan `:latest`. Lambda menerjemahkan tag menjadi digest lalu menguncinya, sehingga mendorong `:latest` baru tidak mengubah fungsi yang berjalan.

---

## 8. Pengembangan setempat

### 8.1 Proxy dev menggantikan CloudFront

Satu origin, sehingga CORS tidak ada dan cookie `SameSite=Strict` terkirim. Memanggil API lintas origin dari `localhost` gagal berlapis: tidak ada header CORS, dan cookie tidak pernah dikirim.

### ⚠️ 8.2 Cookie `Secure` dan Safari di `http://localhost`

| Peramban | Cookie `Secure` di `http://localhost` |
|---|---|
| Chrome, Edge, Firefox | diterima |
| **Safari** | **ditolak, tanpa peringatan** |

**Gejalanya menyesatkan sepenuhnya:** masuk **berhasil** — sehingga tidak ada percobaan gagal tercatat di mana pun — lalu request berikutnya berjalan tanpa sesi, dijawab `401`, dan pengguna dilempar balik ke halaman masuk. Kecurigaan mengarah ke kata sandi, yang sama sekali bukan sebabnya.

Lepas `Secure` **hanya pada proxy dev**, dan **pertahankan `HttpOnly`**:

```js
configure(agen) {
  agen.on("proxyRes", (jawaban) => {
    const kuki = jawaban.headers["set-cookie"];
    if (kuki) jawaban.headers["set-cookie"] = kuki.map((c) => c.replace(/;\s*Secure/gi, ""));
  });
}
```

### 8.3 Perkakas operator

| Alat | Untuk apa | Catatan macOS |
|---|---|---|
| `session-manager-plugin` | terowongan SSM | `brew install --cask session-manager-plugin` |
| `psql` | pemeliharaan basis data | `brew install libpq && brew link --force libpq` — keg-only |
| Docker | suite tes berbasis Testcontainers | Matinya muncul sebagai `Could not find a working container runtime strategy`, bukan sebagai tes yang gagal |

⚠️ **bash bawaan macOS adalah 3.2.** `${var^^}`, array asosiatif, dan `mapfile` tidak ada. Skrip yang memakainya gagal di tengah dengan `bad substitution` sementara sisanya tetap jalan — meninggalkan keadaan setengah jadi. Tulis untuk bash 3.2, dan **buat skrip basis data idempoten**.

---

## 9. Urutan penaikan

```
1  bootstrap: bucket state + ECR
2  dorong image :bootstrap  ← minimal, tanpa env wajib, beserta jalur diagnostik
3  import sumber daya yang sudah dibuat tangan   ← SEBELUM apply
4  terraform apply
5  buktikan OAC ber-body    ← selagi :bootstrap masih terpasang; jendelanya tertutup
                              begitu rilis pertama berjalan
6  isi rahasia + buat role basis data
7  setel secret/variable CI
8  rilis pertama
```

**Langkah 3 dan 5 paling sering terbalik atau terlewat.** Yang ke-3 karena `import` terasa seperti pekerjaan pembersihan yang bisa ditunda; yang ke-5 karena alat ukurnya hidup di dalam image yang akan segera diganti.

---

## 10. Daftar periksa

Sebelum `apply` pertama:

- [ ] `grep -rnE '(count|for_each)\s*=.*\baws_' *.tf` bersih
- [ ] `ignore_changes` pada `image_uri` **dan** `function_version`
- [ ] Alias menunjuk `$LATEST`, bukan version bernomor
- [ ] Tidak ada `aws_secretsmanager_secret_version` maupun `aws_ssm_parameter`
- [ ] `prevent_destroy` pada ECR, basis data, dan bucket berdata
- [ ] Provider alias `us-east-1` sudah ada bila kelak memakai ACM untuk CloudFront
- [ ] Sumber daya yang dibuat tangan sudah di-`import`

Sesudah infrastruktur naik:

- [ ] `terraform plan -detailed-exitcode` keluar **0** — tanpa selisih abadi
- [ ] `GET /api/<sehat>` lewat CloudFront menjawab 200
- [ ] `POST` ber-body **tanpa** `x-amz-content-sha256` dijawab **403**
- [ ] `POST` ber-body **dengan** header itu menjawab 200, **dan sidik jarinya cocok**
- [ ] Function URL dipanggil langsung dijawab **403**
- [ ] Rahasia diverifikasi dengan **benar-benar masuk**, bukan dengan memeriksa keberadaan versi
- [ ] Aturan security group sementara **sudah dicabut**

---

## 11. Tiga pelajaran yang berlaku di luar susunan ini

**Pesan galat menunjuk ke tempat kejadian, bukan ke sebabnya.** `password authentication failed` karena format `.pgpass`. `berkas_terender: 0` karena `s3:ListBucket`. Stuck di halaman login karena atribut `Secure`. Pada ketiganya, membaca pesannya secara harfiah membuang waktu paling banyak.

**Menguji dengan kredensial yang lebih berkuasa daripada pemanggil sesungguhnya adalah pengujian yang menyesatkan.** Ia menjawab "apakah jalurnya sehat", bukan "apakah izinnya cukup" — dan yang kedua itulah yang ditanyakan.

**Berhenti menebak lebih awal.** Lima `apply` terbuang menebak kebijakan IAM sebelum membaca dokumentasi resmi. Aturannya sederhana: dua tebakan gagal berturut-turut, berhenti dan baca sumbernya.

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 12 Agustus 2026 | Dokumen dibuat, disarikan dari penaikan EduTrack Jalur B — dari `terraform apply` pertama sampai rilis pertama berhasil dan frontend tersambung. Seluruh butirnya benar-benar terjadi |
