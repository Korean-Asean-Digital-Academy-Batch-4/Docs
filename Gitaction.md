# Catatan Penelusuran — OIDC GitHub Actions ke AWS

| Keterangan | Isi |
|---|---|
| **Versi** | v1.0 |
| **Tanggal** | 7 Agustus 2026 |
| **Status** | 🔴 **Belum selesai** — jabat tangan OIDC masih ditolak |
| **Kedudukan** | Catatan penelusuran. **Tidak menetapkan apa pun.** Prosedurnya pada [RUNBOOK-OIDC.md](RUNBOOK-OIDC.md), keputusannya pada [DEPLOYMENT.md §9.4](DEPLOYMENT.md) |
| **Gejala** | `Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity` |

> Dokumen ini mencatat **apa yang sudah terbukti, apa yang sudah dicoba, dan apa yang belum pernah dilihat**, supaya penelusuran tidak mengulang langkah yang sama. Ditutup dan diringkas menjadi satu baris pada `RUNBOOK-OIDC.md` begitu penyebabnya ditemukan.

---

## 1. Sasaran

GitHub Actions memperoleh kredensial AWS sementara tanpa satu pun kunci disimpan, dibuktikan lewat `aws sts get-caller-identity` yang mengembalikan `assumed-role/edutrack-gha-backend/uji-oidc`.

Ini **B0.5** pada urutan tahap, dan menjadi prasyarat seluruh jalur rilis.

---

## 2. Keadaan sekarang

| | |
|---|---|
| Percobaan | 4 kali, seluruhnya gagal |
| Kegagalan terakhir | 1 menit 32 detik, 12 kali percobaan ulang |
| Langkah yang gagal | **Pinjam role AWS lewat OIDC** |
| Langkah yang belum pernah jalan | **Buktikan identitas yang diperoleh** |
| `Last activity` pada role | `-` — belum pernah sekali pun berhasil dipinjam |

Dua belas baris `Assuming role with OIDC` berturut-turut adalah **percobaan ulang**, bukan kemajuan. Jabat tangan yang berhasil mencatat baris itu sekali saja dan selesai dalam dua detik.

---

## 3. Yang sudah terbukti benar

Seluruhnya diperiksa langsung di konsol, bukan diasumsikan.

| # | Yang diperiksa | Nilai | Bukti |
|:--:|---|---|---|
| 1 | Identity provider ada | `token.actions.githubusercontent.com`, tipe OpenID Connect | IAM → Identity providers, dibuat 7 Agu 02:05 |
| 2 | ARN provider | `arn:aws:iam::274286556151:oidc-provider/token.actions.githubusercontent.com` | Panel Summary provider |
| 3 | **Audiences pada provider** | `sts.amazonaws.com` — tepat satu entri | Tab Audiences |
| 4 | Role ada | `edutrack-gha-backend`, dibuat 7 Agu 02:07 | IAM → Roles |
| 5 | ARN role | `arn:aws:iam::274286556151:role/edutrack-gha-backend` | Panel Summary role |
| 6 | Permission policy kosong | 0 policy — **memang disengaja** | Tab Permissions |
| 7 | Principal pada trust policy | `arn:aws:iam::274286556151:o…` — pengenal akun sungguhan | Kolom Trusted entities |
| 8 | Secret ada di repositori | `AWS_ROLE_ARN` | Settings → Secrets and variables → Actions |
| 9 | Workflow memuat `id-token: write` | ada | `.github/workflows/oidc-smoke.yml` |
| 10 | Dijalankan dari branch `main` | ya, oleh `andreasTimo` | Halaman ringkasan run |

Butir 6 perlu ditegaskan: **permission kosong bukan penyebab kegagalan.** `sts:GetCallerIdentity` tidak memerlukan izin apa pun, dan kegagalannya terjadi pada tahap peminjaman role — sebelum izin apa pun sempat diperiksa.

---

## 4. Yang sudah dicoba

| # | Percobaan | Yang diubah | Hasil |
|:--:|---|---|---|
| 1 | Run pertama | — | Gagal |
| 2 | Run kedua | — | Gagal |
| 3 | **Perbaikan pertama** | Principal pada trust policy masih memuat teks pengganti `<ID-AKUN>` secara harfiah, diganti menjadi `274286556151` | Terbukti dari kolom Trusted entities. **Kegagalan tetap** |
| 4 | Run ketiga, commit `2d17482` | — | Gagal, pesan sama |
| 5 | **Diagnostik ditambahkan**, commit `2c8134a` | Langkah pencetak klaim `sub` dan `aud`, dijalankan **sebelum** peminjaman role agar hasilnya tetap terlihat meski gagal | Langkah berjalan; **keluarannya belum dibaca** |
| 6 | Run keempat | — | Gagal, pesan sama |

Perbaikan nomor 3 juga ditutup di sisi dokumen: teks pengganti pada `RUNBOOK-OIDC.md` diubah menjadi `GANTI-DENGAN-ID-AKUN` dan peringatannya dipindahkan tepat di bawah blok JSON, agar penempelan apa adanya tidak terulang.

---

## 5. Yang belum pernah dilihat nilainya

Tiga hal, dan penyebabnya pasti salah satu di antaranya.

| # | Yang belum terlihat | Kenapa penting |
|:--:|---|---|
| A | **Keluaran langkah pencetak klaim** | Menunjukkan bentuk `sub` dan `aud` yang **sebenarnya** dikirim GitHub |
| B | **Blok `Condition` pada trust policy** | Kolom Trusted entities hanya menampilkan Principal. Satu karakter yang bergeser saat menyunting di konsol tidak terlihat dari mana pun |
| C | **Isi Secret `AWS_ROLE_ARN`** | Nilainya tidak dapat dibaca ulang setelah disimpan, hanya dapat ditimpa |

---

## 6. Kenapa sulit dipersempit

**AWS mengembalikan pesan yang sama persis** untuk tiga keadaan yang berbeda:

1. Role yang dituju tidak ada
2. Trust policy menolak
3. Klaim `sub` tidak cocok

Ini disengaja — AWS tidak memberi tahu mana yang terjadi, karena itu akan membocorkan keberadaan sumber daya kepada pihak yang tidak berhak. Akibatnya, penelusuran harus dilakukan dengan **membelah kemungkinan**, bukan dengan membaca pesan.

---

## 7. Dugaan yang tersisa

| # | Dugaan | Alasan |
|:--:|---|---|
| 1 | **Organisasi GitHub mengubah bentuk klaim `sub`** | GitHub mengizinkan penyetelan *custom subject claim* di tingkat organisasi maupun repositori. Bila aktif, `sub` bukan lagi `repo:ORG/REPO:ref:refs/heads/main`, sehingga tidak akan pernah cocok betapa pun benarnya sisi AWS. Organisasi bootcamp kerap memiliki setelan tingkat organisasi yang tidak terlihat dari sisi repositori |
| 2 | **Isi Secret keliru** | Salah ketik atau spasi ikut tertempel menghasilkan gejala yang identik (§6 butir 1) |
| 3 | **Blok `Condition` rusak** saat disunting di konsol | Penyuntingan JSON di editor konsol mudah menggeser atau menghapus baris |

Dugaan 1 naik menjadi yang teratas karena **seluruh sisi AWS sudah terbukti benar** — butir 1 sampai 7 pada §3 — sehingga yang tersisa hanya kemungkinan bahwa GitHub mengirim sesuatu yang berbeda dari yang diandaikan.

---

## 8. Langkah berikutnya

**Yang paling menentukan** — buka langkah **Tampilkan klaim token OIDC yang dikirim GitHub** pada run terakhir, klik tanda panah di sebelah kirinya. Dua baris `sub` dan `aud` menjawab dugaan 1 sekaligus.

**Bila log sulit dibaca**, membelah kemungkinan dengan mengubah trust policy **sementara**:

```json
"Condition": {
  "StringLike": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "*"
  }
}
```

| Hasil | Kesimpulan |
|---|---|
| Berhasil | Terbukti murni soal bentuk `sub` — dugaan 1 atau 3 |
| Tetap gagal | Bukan `sub` — tersisa dugaan 2, Secret ditimpa |

> 🛑 Bentuk ini menerima **repositori mana pun di internet** yang memakai OIDC GitHub. Hanya boleh hidup beberapa menit untuk diagnosis, dan **wajib dikembalikan** ke `StringEquals` segera setelah penyebabnya ketahuan. Jangan ditinggalkan semalam.

**Langkah murah yang bisa dikerjakan kapan saja** — timpa Secret `AWS_ROLE_ARN` dengan nilai berikut, tanpa spasi di depan maupun belakang:

```
arn:aws:iam::274286556151:role/edutrack-gha-backend
```

Menimpa lebih murah daripada memastikan, karena nilainya memang tidak dapat dibaca.

---

## 9. Dampak terhadap pekerjaan lain

**Tidak menghalangi apa pun selain jalur rilis.** Seluruh Jalur A — kerangka repositori, Docker, migrasi, `domain/`, sampai jalur AI — dikerjakan lokal dan tidak menyentuh AWS sama sekali.

Yang tertahan hanya B5, yaitu penambahan izin ECR dan Lambda pada role ini, dan itu memang baru relevan setelah keduanya ada.

---

## 10. Yang menandai selesai

- [ ] `aws sts get-caller-identity` mengembalikan `assumed-role/edutrack-gha-backend/uji-oidc`
- [ ] Trust policy kembali memakai `StringEquals`, bukan `StringLike` dengan bintang
- [ ] Penyebabnya ditambahkan ke tabel diagnosa [RUNBOOK-OIDC.md](RUNBOOK-OIDC.md) Bagian 6
- [ ] Dokumen ini diringkas menjadi satu baris riwayat, lalu dihapus

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 7 Agustus 2026 | Dokumen dibuat setelah empat percobaan gagal. Mencatat sepuluh hal yang sudah terbukti benar, enam percobaan yang sudah dilakukan, tiga nilai yang belum pernah terlihat, dan tiga dugaan yang tersisa |
