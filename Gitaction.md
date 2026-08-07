# Catatan Penelusuran — OIDC GitHub Actions ke AWS

| Keterangan | Isi |
|---|---|
| **Versi** | v2.0 |
| **Tanggal** | 7 Agustus 2026 |
| **Status** | ✅ **Selesai** — jabat tangan OIDC berhasil |
| **Kedudukan** | Catatan penelusuran. **Tidak menetapkan apa pun.** Prosedurnya pada [RUNBOOK-OIDC.md](RUNBOOK-OIDC.md), keputusannya pada [DEPLOYMENT.md §9.4](DEPLOYMENT.md) |
| **Gejala awal** | `Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity` |

---

## Penyebab

Organisasi `Korean-Asean-Digital-Academy-Batch-4` mengaktifkan **custom subject claim** di tingkat organisasi. Format `sub` yang dikirim GitHub bukan format baku:

```
# Yang dikirim GitHub (custom claim aktif)
repo:Korean-Asean-Digital-Academy-Batch-4@307598735/backend@1325655840:ref:refs/heads/main

# Format baku (tanpa custom claim)
repo:Korean-Asean-Digital-Academy-Batch-4/backend:ref:refs/heads/main
```

Trust policy memakai format baku, sehingga `Condition` tidak pernah cocok dan AWS menolak seluruh permintaan.

---

## Solusi

Ubah nilai `sub` pada blok `Condition` di trust policy role `edutrack-gha-backend` menjadi nilai aktual yang dikirim GitHub:

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:Korean-Asean-Digital-Academy-Batch-4@307598735/backend@1325655840:ref:refs/heads/main"
  }
}
```

Cara mendapat nilai `sub` aktual: tambahkan langkah diagnostik di workflow **sebelum** langkah peminjaman role:

```yaml
- name: Tampilkan klaim token OIDC yang dikirim GitHub
  run: |
    TOKEN=$(curl -sS \
      -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" \
      | jq -r '.value')
    PAYLOAD=$(echo "$TOKEN" | cut -d. -f2 | tr '_-' '/+')
    while [ $(( ${#PAYLOAD} % 4 )) -ne 0 ]; do PAYLOAD="${PAYLOAD}="; done
    echo "$PAYLOAD" | base64 -d | jq '{sub, aud}'
```

Nilai yang tercetak itulah yang harus diisi ke `Condition`.

---

## Yang terbukti benar (tidak berubah)

| # | Yang diperiksa | Nilai |
|:--:|---|---|
| 1 | Identity provider | `token.actions.githubusercontent.com`, tipe OpenID Connect |
| 2 | ARN provider | `arn:aws:iam::274286556151:oidc-provider/token.actions.githubusercontent.com` |
| 3 | Audiences pada provider | `sts.amazonaws.com` |
| 4 | ARN role | `arn:aws:iam::274286556151:role/edutrack-gha-backend` |
| 5 | Secret `AWS_ROLE_ARN` | berisi ARN role yang benar |
| 6 | `id-token: write` pada workflow | ada |

---

## Checklist selesai

- [x] Trust policy memakai `StringEquals` dengan nilai `sub` aktual dari log diagnostik
- [x] Workflow berhasil — tidak ada 12 percobaan ulang
- [x] Penyebabnya: custom subject claim organisasi mengubah format `sub`
- [ ] Penyebabnya ditambahkan ke tabel diagnosa [RUNBOOK-OIDC.md](RUNBOOK-OIDC.md) Bagian 6

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 7 Agustus 2026 | Dokumen dibuat setelah empat percobaan gagal. Mencatat sepuluh hal yang sudah terbukti benar, enam percobaan yang sudah dilakukan, tiga nilai yang belum pernah terlihat, dan tiga dugaan yang tersisa |
| 7 Agustus 2026 | **Selesai.** Penyebab ditemukan: custom subject claim organisasi. Solusi: perbarui nilai `sub` di `Condition` trust policy sesuai log diagnostik |
