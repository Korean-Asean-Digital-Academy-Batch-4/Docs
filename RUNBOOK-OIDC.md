# Runbook — Menyiapkan OIDC GitHub Actions ke AWS

| Keterangan | Isi |
|---|---|
| **Versi** | v1.0 |
| **Tanggal** | 7 Agustus 2026 |
| **Kedudukan** | Prosedur yang dijalankan tangan. Keputusan yang mendasarinya pada [DEPLOYMENT.md §9.4](DEPLOYMENT.md) dan CK-D-01 |
| **Lama pengerjaan** | Sekitar 20 menit |
| **Hasil akhir** | GitHub Actions dapat memperoleh kredensial AWS sementara **tanpa satu pun kunci disimpan di mana pun** |

> Runbook ini **tidak menetapkan apa pun**. Ia menjalankan keputusan yang sudah diambil. Apabila berbeda dengan `DEPLOYMENT.md`, dokumen itu yang berlaku.
>
> Pengenal akun AWS ditulis `<ID-AKUN>`. Ganti dengan nilai sebenarnya saat mengerjakan, jangan menuliskannya kembali ke repositori.

---

## Yang dihasilkan runbook ini

```
GitHub Actions                          AWS
     │                                   │
     │  1. GitHub menerbitkan token       │
     │     "job ini dari repo X,          │
     │      branch main"                  │
     │                                   │
     ├──────── token ──────────────────► │  2. AWS memeriksa tanda tangan
     │                                   │     dan mencocokkan klaimnya
     │                                   │
     │ ◄────── kredensial sementara ──── │  3. Berlaku ±1 jam, mati
     │                                   │     bersama pekerjaannya
```

**Tidak ada access key yang dibuat, disalin, atau ditempel.**

---

## Prasyarat

| # | Yang dibutuhkan | Cara memastikan |
|:--:|---|---|
| 1 | Akses admin ke akun AWS | Masuk konsol dan bisa membuka IAM |
| 2 | Akses admin ke repositori GitHub | Tab **Settings** terlihat pada repositori |
| 3 | Repositori sudah punya branch `main` | Sudah — commit pertama sudah ada |
| 4 | Nama lengkap repositori | `Korean-Asean-Digital-Academy-Batch-4/backend` |

Bagian AWS pada runbook ini **wajib dikerjakan manusia**. Agen tidak dapat menggantikannya, karena setiap jalur menuju kuasa AWS menuntut kode MFA.

---

## Bagian 1 — AWS: daftarkan GitHub sebagai identity provider

Dilakukan **sekali per akun AWS**. Role backend dan role frontend nanti memakai provider yang sama.

| # | Langkah |
|:--:|---|
| 1.1 | Masuk ke AWS Console |
| 1.2 | Buka layanan **IAM**. Pemilih region di kanan atas boleh apa saja — IAM bersifat global |
| 1.3 | Menu kiri → **Identity providers** |
| 1.4 | Tombol **Add provider** |
| 1.5 | **Provider type**: pilih **OpenID Connect** |
| 1.6 | **Provider URL**: `https://token.actions.githubusercontent.com` |
| 1.7 | Klik **Get thumbprint** |
| 1.8 | **Audience**: `sts.amazonaws.com` |
| 1.9 | Tombol **Add provider** |
| 1.10 | Buka provider yang baru dibuat, **salin ARN-nya** — diperlukan pada Bagian 2 |

ARN-nya berbentuk:

```
arn:aws:iam::<ID-AKUN>:oidc-provider/token.actions.githubusercontent.com
```

**Soal thumbprint.** Sejak 2023 AWS memvalidasi sertifikat GitHub lewat CA tepercaya, sehingga thumbprint ini **tidak perlu dirotasi** dan tidak perlu dipantau. Isi sekali lewat tombolnya, lalu lupakan.

**Jangan membuat provider kedua** untuk frontend. Satu akun cukup satu, dan usaha membuat yang kedua dengan URL sama akan ditolak.

---

## Bagian 2 — AWS: buat role `edutrack-gha-backend`

| # | Langkah |
|:--:|---|
| 2.1 | IAM → menu kiri → **Roles** → **Create role** |
| 2.2 | **Trusted entity type**: pilih **Custom trust policy** |
| 2.3 | Timpa seluruh isi editor dengan JSON di bawah |
| 2.4 | **Next** |
| 2.5 | Halaman permissions: **jangan pilih apa pun**. Lewati, lalu **Next** |
| 2.6 | **Role name**: `edutrack-gha-backend` — huruf kecil semua |
| 2.7 | **Create role** |
| 2.8 | Buka role tersebut, **salin ARN-nya** |

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ID-AKUN>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:Korean-Asean-Digital-Academy-Batch-4/backend:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**Kenapa permissions dikosongkan.** Tahap ini hanya menguji jabat tangannya. Perintah `sts:GetCallerIdentity` yang dipakai menguji **tidak memerlukan izin apa pun**, sehingga role tanpa satu pun policy tetap membuktikan bahwa AWS memercayai token GitHub dan klaimnya cocok. Izin ECR dan Lambda ditambahkan belakangan, setelah keduanya ada.

**Kenapa Custom trust policy, bukan Web identity.** Pilihan **Web identity** menyediakan formulir GitHub yang lebih cepat, tetapi apabila kolom branch dibiarkan kosong ia menghasilkan `sub` dengan tanda bintang — dan itu berarti **branch mana pun** dapat memperoleh kredensial. Menempel JSON di atas membuat isinya terlihat apa adanya.

> ⚠️ Jangan pernah menulis `sub` sebagai `repo:<ORG>/*`. Itu memberi **setiap repositori di organisasi** hak menerapkan ke produksi. Ini kekeliruan OIDC yang paling sering terjadi, dan tidak menimbulkan gejala apa pun sampai disalahgunakan.

---

## Bagian 3 — GitHub: simpan ARN role

| # | Langkah |
|:--:|---|
| 3.1 | Buka `https://github.com/Korean-Asean-Digital-Academy-Batch-4/backend` |
| 3.2 | Tab **Settings** |
| 3.3 | Menu kiri → **Secrets and variables** → **Actions** |
| 3.4 | Tab **Secrets** → **New repository secret** |
| 3.5 | **Name**: `AWS_ROLE_ARN` |
| 3.6 | **Secret**: tempel ARN dari langkah 2.8 |
| 3.7 | **Add secret** |

**Kenapa Secret, bukan Variable.** ARN memuat pengenal akun AWS. Sebagai Secret, GitHub menyamarkannya di seluruh keluaran log; sebagai Variable, ia tampil apa adanya kepada siapa pun yang dapat membaca repositori. Ini sejalan dengan keputusan tidak menuliskan pengenal akun ke dalam repositori dokumen.

Konsekuensinya, saat menelusuri kegagalan Anda akan melihat `***` alih-alih ARN. Kalau itu menghambat, sementara ganti ke Variable, lalu kembalikan.

---

## Bagian 4 — GitHub: tulis workflow uji

Ditulis **di lokal**, lalu di-commit dan di-push. Jangan lewat editor web.

```bash
cd ~/Documents/Edudex/backend
mkdir -p .github/workflows
```

Isi `.github/workflows/oidc-smoke.yml`:

```yaml
name: Uji jabat tangan OIDC

on: workflow_dispatch

permissions:
  id-token: write
  contents: read

jobs:
  uji:
    runs-on: ubuntu-latest
    steps:
      - name: Pinjam role AWS lewat OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          role-session-name: uji-oidc
          aws-region: ap-southeast-1

      - name: Buktikan identitas yang diperoleh
        run: aws sts get-caller-identity
```

```bash
git add .github/workflows/oidc-smoke.yml
git commit -m "ci: tambah uji jabat tangan OIDC"
git push
```

**`permissions: id-token: write` tidak boleh dilewat.** Tanpa baris itu GitHub sama sekali tidak menerbitkan token, dan kegagalannya terjadi sebelum AWS tersentuh.

**Berkas wajib berada di branch `main`.** Tombol jalankan pada `workflow_dispatch` hanya muncul apabila berkasnya sudah ada di branch bawaan.

---

## Bagian 5 — Jalankan dan baca hasilnya

| # | Langkah |
|:--:|---|
| 5.1 | Buka repositori di GitHub → tab **Actions** |
| 5.2 | Menu kiri → **Uji jabat tangan OIDC** |
| 5.3 | Tombol **Run workflow** → pastikan branch **`main`** → **Run workflow** |
| 5.4 | Tunggu sekitar 20 detik, buka jalannya, buka langkah **Buktikan identitas yang diperoleh** |

**Lulus** apabila keluarannya berbentuk:

```json
{
    "UserId": "AROA…:uji-oidc",
    "Account": "…",
    "Arn": "arn:aws:sts::…:assumed-role/edutrack-gha-backend/uji-oidc"
}
```

Yang dibuktikan: `assumed-role/edutrack-gha-backend` — **bukan** `user/`. Kredensialnya dipinjam, bukan milik seseorang.

---

## Bagian 6 — Ketika gagal

| Pesan | Sebab | Perbaikan |
|---|---|---|
| `Unable to get ACTIONS_ID_TOKEN_REQUEST_URL` | Blok `permissions` tidak ada, atau `id-token: write` lupa ditulis | Tambahkan pada workflow. Perhatikan letaknya sejajar `on:`, bukan di dalam `jobs:` |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Klaim `sub` tidak cocok. **Penyebab tersering:** workflow dijalankan dari branch selain `main`, atau dari pull request | Pastikan langkah 5.3 memilih `main`. Bandingkan `sub` pada trust policy dengan nama repositori huruf demi huruf |
| Tombol **Run workflow** tidak muncul | Berkas workflow belum ada di branch bawaan | `git push` ke `main` |
| `Invalid identity token` | ARN provider pada trust policy salah ketik | Bandingkan dengan ARN dari langkah 1.10 |
| `No OpenIDConnect provider found` | Provider belum dibuat, atau URL-nya keliru | Ulangi Bagian 1. URL memakai `https://`, tanpa garis miring di akhir |
| `Could not load credentials from any providers` | Secret `AWS_ROLE_ARN` kosong atau salah nama | Periksa Bagian 3, perhatikan huruf besar-kecilnya |
| Lulus, tetapi ARN-nya `user/…` | Workflow tidak memakai OIDC | Pastikan `role-to-assume` terisi dan tidak ada `aws-access-key-id` di mana pun |

**Cara memastikan `sub` yang sebenarnya dikirim.** Klaimnya mengikuti pemicu:

| Dijalankan dari | `sub` yang dikirim |
|---|---|
| Push atau dispatch di `main` | `repo:ORG/REPO:ref:refs/heads/main` |
| Branch `fitur-x` | `repo:ORG/REPO:ref:refs/heads/fitur-x` |
| Pull request | `repo:ORG/REPO:pull_request` |
| Tag `v1` | `repo:ORG/REPO:ref:refs/tags/v1` |

Trust policy saat ini hanya menerima baris pertama. Itu disengaja: **pull request dari mana pun tidak boleh menyentuh AWS.**

---

## Bagian 7 — Setelah lulus

**Biarkan role tanpa permission.** Izin ECR dan Lambda ditambahkan setelah keduanya ada, sesuai [DEPLOYMENT §9.4](DEPLOYMENT.md). Menambahkannya sekarang berarti menulis ARN sumber daya yang belum lahir.

**Simpan berkas ujinya.** `oidc-smoke.yml` tetap berguna: setiap kali jalur rilis bermasalah, ia memisahkan "jabat tangannya rusak" dari "rilisnya rusak" dalam dua puluh detik.

**Role frontend menyusul belakangan** memakai provider yang sama, dengan `sub` menunjuk repositori frontend. Jangan membuat provider baru.

**Penguatan opsional.** Untuk repositori yang menerapkan ke produksi, `uses:` pada action pihak ketiga sebaiknya dipatok ke commit SHA, bukan ke `@v4`. Tag dapat dipindahkan; SHA tidak.

---

## Daftar periksa

- [ ] Identity provider dibuat, ARN disalin
- [ ] Role `edutrack-gha-backend` dibuat dengan trust policy tertempel, **tanpa permission**
- [ ] `sub` menyebut nama repositori lengkap dan `refs/heads/main`, **tanpa tanda bintang**
- [ ] Secret `AWS_ROLE_ARN` terisi
- [ ] `oidc-smoke.yml` ada di `main`, memuat `id-token: write`
- [ ] Workflow dijalankan dari `main` dan lulus
- [ ] Keluarannya `assumed-role/edutrack-gha-backend/uji-oidc`
- [ ] Tidak ada satu pun access key yang dibuat sepanjang prosedur ini

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 7 Agustus 2026 | Runbook dibuat. Menjalankan keputusan `DEPLOYMENT.md` §9.4 dan CK-D-01 |
