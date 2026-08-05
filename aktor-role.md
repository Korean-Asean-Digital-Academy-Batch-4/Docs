# Aktor & Role — EduTrack

> **Status:** Disepakati 2 Agustus 2026
> **Sumber kebenaran** untuk semua pertanyaan "siapa boleh lihat/ubah apa".
> Kalau PRD, Project Charter, atau use-case diagram bertentangan dengan file ini, **file ini yang menang** — kecuali ada keputusan tim baru yang ditulis di bagian [Riwayat Keputusan](#riwayat-keputusan).

---

## 1. Ringkasan

Sistem punya **3 role login**: `superadmin`, `guru`, `siswa`.

Tapi **"Guru" bukan satu peran** — ada dua topi yang bisa dipakai satu orang **sekaligus**:

| Topi | Wujudnya di data |
|---|---|
| **Guru Mapel** | Banyak baris di tabel `penugasan` |
| **Wali Kelas** | Satu kolom `kelas.wali_kelas_user_id` |

Satu guru bisa punya dua-duanya. Ini normal, bukan kasus pinggiran.

---

## 2. Model mental: dua arah potongan

Guru Mapel dan Wali Kelas memotong data yang sama di **arah yang berbeda**.

```
                    X-MIPA-1   X-MIPA-2   XII-IPA-3
                  ┌──────────┬──────────┬──────────┐
     Matematika   │  ██████  │  ██████  │  ██████  │  ← GURU MAPEL: vertikal
                  ├──────────┼──────────┼──────────┤     1 mapel × N kelas
     Biologi      │  ██████  │          │          │
                  ├──────────┤          │          │
     B. Indonesia │  ██████  │          │          │
                  ├──────────┤          │          │
     Fisika       │  ██████  │          │          │
                  └──────────┴──────────┴──────────┘
                       ▲
                       │  WALI KELAS: horizontal
                       │  1 kelas × semua mapel
```

Kalau orangnya sama, dua potongan itu **bersilangan di satu kotak** — dan di kotak itu dia punya dua wewenang sekaligus.

**Asimetri yang harus diingat:**

> **Guru Mapel: sempit tapi boleh menulis.**
> **Wali Kelas: luas tapi hanya membaca** — kecuali satu hal, dia yang menerbitkan rapor.

Wali kelas **tidak pernah mengetik nilai**. Dia mengumpulkan yang sudah ada, lalu bertanggung jawab atas hasil akhirnya — persis seperti wali kelas sungguhan.

---

## 3. Contoh konkret

**Bu Rina Kartika** — guru Matematika, sekaligus wali kelas X-MIPA-1.

| Di kelas | Sebagai Guru Mapel | Sebagai Wali Kelas |
|---|---|---|
| **X-MIPA-1** | Tulis nilai & presensi **Matematika** | Baca **semua mapel** · Generate rapor |
| **X-MIPA-2** | Tulis nilai & presensi **Matematika** | — |
| **XII-IPA-3** | Tulis nilai & presensi **Matematika** | — |

**Yang TIDAK boleh** (ini yang paling menjelaskan bedanya):

- ❌ Di X-MIPA-1, Bu Rina **melihat** nilai Biologi (dia wali), tapi **tidak bisa mengubahnya** — itu wilayah Pak Deni
- ❌ Di X-MIPA-2, Bu Rina **tidak bisa melihat** nilai Biologi sama sekali — di kelas itu dia cuma guru mapel
- ❌ Bu Rina **tidak bisa generate rapor** X-MIPA-2, walaupun dia mengajar di sana

---

## 4. Murid

Yang dilihat murid **bentuknya sama persis** dengan yang dilihat wali kelas tentang murid itu: rincian per mapel, dipecah per komponen nilai.

Bedanya cuma **cakupan baris**:

```
Wali kelas X-MIPA-1  →  30 siswa × semua mapel
Naomi (siswa)        →   1 siswa × semua mapel   (dirinya sendiri)
```

**Konsekuensi backend:** ini **bukan dua fitur**. Satu query, satu bentuk response, yang berbeda hanya klausa `WHERE`-nya.

```
GET /api/siswa/{id}/rincian?tahunAjaran=2026/2027&semester=ganjil

boleh kalau:  siswa  DAN  id == diri sendiri
        atau: guru   DAN  kelas.wali_kelas_user_id = saya
                     DAN  siswa ada di roster kelas itu
```

Murid tidak bisa menulis apa pun. Nol.

---

## 5. Matriks izin

| | Superadmin | Guru Mapel | Wali Kelas | Murid |
|---|---|---|---|---|
| **Lingkup** | seluruh sekolah | 1 mapel × N kelas | 1 kelas × semua mapel | dirinya sendiri |
| CRUD akun guru & siswa | ✅ | ❌ | ❌ | ❌ |
| CRUD kelas, mapel, semester, tahun ajaran | ✅ | ❌ | ❌ | ❌ |
| Assign guru/siswa ke kelas | ✅ | ❌ | ❌ | ❌ |
| Atur bobot nilai default | ✅ | ❌ | ❌ | ❌ |
| Atur bobot per penugasan | ✅ | ✅ (miliknya) | ❌ | ❌ |
| Tulis nilai | ❌ | ✅ mapelnya saja | ❌ | ❌ |
| Tulis presensi | ❌ | ✅ mapelnya saja | ❌ | ❌ |
| Tulis sikap spiritual/sosial | ❌ | ✅ mapelnya saja | ❌ | ❌ |
| Import nilai via Excel | ❌ | ✅ mapelnya saja | ❌ | ❌ |
| Baca lintas mapel | ✅ | ❌ | ✅ kelasnya saja | ✅ dirinya saja |
| Generate rapor (→ `draft`) | ❌ | ❌ | ✅ | ❌ |
| Ajukan rapor ke review | ❌ | ❌ | ✅ | ❌ |
| Setujui / terbitkan rapor | ✅ | ❌ | ❌ | ❌ |
| Lihat rapor | ✅ | ❌ | ✅ semua siswanya | ✅ miliknya, setelah `distributed` |
| Lihat AI insight | ✅ | ❌ | ✅ siswanya | ✅ miliknya |

---

## 6. Aturan otorisasi

Hampir semua izin guru berujung ke satu tabel: `penugasan`.

```sql
-- Guru mapel boleh tulis nilai/presensi/sikap?
EXISTS (
  SELECT 1 FROM penugasan
  WHERE id = :penugasanId AND guru_user_id = :me
)

-- Wali kelas boleh baca lintas mapel / generate rapor?
EXISTS (
  SELECT 1 FROM kelas
  WHERE id = :kelasId AND wali_kelas_user_id = :me
)

-- Murid boleh baca data siswa?
:siswaId = :me
```

**Kenapa digantung ke `penugasan_id`:** izin dicek **sekali di pintu masuk**, bukan per baris nilai. Begitu lolos, semua data di bawahnya (nilai, komponen, presensi, sikap, materi) otomatis dalam lingkup. Ini yang membuat RBAC-nya tidak berantakan saat fitur bertambah.

---

## 7. Batas Cognito ↔ Postgres

Ini **tidak boleh dilanggar**:

| Cognito jawab | Postgres jawab |
|---|---|
| "Siapa orang ini?" | "Orang ini boleh sentuh baris yang mana?" |
| Group: `superadmin` / `guru` / `siswa` | Wali kelas mana, mengajar apa, kelas siapa |

**Cognito User Pool hanya punya 3 group.** Cognito tidak akan pernah tahu Bu Rina wali kelas X-MIPA-1 — itu relasi data yang berubah tiap tahun ajaran, bukan atribut identitas.

Jembatannya satu kolom: `users.cognito_sub`.

Jangan pernah menaruh `wali_kelas` sebagai Cognito group atau custom attribute. Kalau itu terjadi, ada dua sumber kebenaran yang akan berbeda dan tidak ada yang tahu mana yang benar.

**Validasi token** dilakukan di middleware Hono (JWKS Cognito, di-cache), **bukan** di JWT authorizer API Gateway — karena authorizer bawaan hanya bisa membaca header `Authorization`, tidak bisa membaca HttpOnly cookie.

---

## 8. Wujud di database

Tiga peran, tiga bentuk penyimpanan yang berbeda — dan **tidak satu pun berupa kolom `role` biasa**.

```sql
-- GURU MAPEL = banyak baris
penugasan (id, guru_user_id, mapel_id, kelas_id, periode_id)
  UNIQUE (mapel_id, kelas_id, periode_id)

  (rina, matematika, X-MIPA-1,  ganjil-2026)
  (rina, matematika, X-MIPA-2,  ganjil-2026)
  (rina, matematika, XII-IPA-3, ganjil-2026)

-- WALI KELAS = satu kolom
kelas (id, nama, tingkat, periode_id, wali_kelas_user_id)

  X-MIPA-1  →  wali_kelas_user_id = rina
  X-MIPA-2  →  wali_kelas_user_id = deni

-- MURID = satu baris roster
kelas_siswa (kelas_id, siswa_user_id, status)

  (X-MIPA-1, naomi, 'aktif')
```

`UNIQUE (mapel_id, kelas_id, periode_id)` menjamin satu mapel di satu kelas hanya diajar **satu** guru per periode. Kalau sekolah ternyata pakai team teaching, constraint ini harus dilonggarkan — lihat [Pertanyaan Terbuka](#pertanyaan-terbuka).

---

## 9. Kontrak `/api/me`

Dipanggil sekali setelah login. Frontend menyusun menu dari response ini.

```jsonc
GET /api/me

{
  "id": "…",
  "nama": "Rina Kartika",
  "role": "guru",
  "penugasan": [
    { "id": "…", "mapel": "Matematika Wajib", "kelas": "X-MIPA 1" },
    { "id": "…", "mapel": "Matematika Wajib", "kelas": "X-MIPA 2" },
    { "id": "…", "mapel": "Matematika Wajib", "kelas": "XII-IPA 3" }
  ],
  "waliKelas": { "kelasId": "…", "kelas": "X-MIPA 1" }   // null kalau bukan wali
}
```

**Aturan menu:**

| Kondisi | Menu yang tampil |
|---|---|
| `penugasan` tidak kosong | Input Nilai, Presensi |
| `waliKelas !== null` | Generate Rapor AI |
| `role === "siswa"` | Dashboard, Nilai, Rapor |
| `role === "superadmin"` | Manajemen + Persetujuan Rapor |

> ⚠️ **Perlu diperbaiki di wireframe:** sidebar guru saat ini menampilkan "Generate Rapor AI" untuk **semua** guru. Guru yang bukan wali kelas seharusnya tidak melihat menu itu sama sekali. Di sekolah nyata mayoritas guru bukan wali kelas — ini bukan kasus pinggiran.

---

## 10. Lifecycle rapor & siapa yang berwenang

```
      wali kelas              superadmin              superadmin
[draft] ──ajukan──> [review] ──setujui──> [distributed]
   ▲                    │                        │
   └────kembalikan──────┘                   terbit ulang
                                       (versi baru, lama diarsip)
```

| Status | Siapa yang lihat | Bisa diubah? |
|---|---|---|
| `draft` | Wali kelas saja | ✅ regenerate, edit deskripsi AI |
| `review` | Wali kelas + Superadmin | ❌ beku — hanya setujui / kembalikan |
| `distributed` | + Siswa | ❌ perubahan wajib versi baru |

Yang mereview adalah **Superadmin**, mewakili Kepala Sekolah (di PRD berstatus "pengguna tidak langsung", jadi bukan aktor sistem).

---

## 11. AI tidak bisa menyentuh nilai

Charter mensyaratkan AI *"cannot alter grade data"*. Dijamin di **level database**, bukan disiplin kode:

```sql
-- Lambda `api`
GRANT SELECT, INSERT, UPDATE ON nilai, presensi, penilaian_sikap TO app_rw;

-- Lambda `worker` (jalur AI & generate rapor)
GRANT SELECT ON nilai, presensi TO app_ai;
GRANT INSERT, UPDATE ON rapor, rapor_mapel, ai_insight TO app_ai;
-- TIDAK ADA UPDATE pada nilai. Sama sekali.
```

Kalau ada bug yang membuat worker mencoba menulis nilai, **Postgres yang menolak**. QA bisa membuktikannya dengan satu query, bukan dengan membaca kode.

---

## Riwayat Keputusan

| Tanggal | Keputusan | Alasan / Dampak |
|---|---|---|
| 2026-08-02 | **Orang Tua digabung ke akun Siswa** | Tidak ada layar Orang Tua di wireframe, dan 12 hari tersisa. Efektif jadi 3 role — sesuai use-case diagram. **FR-14 gugur** (1 akun tidak bisa pantau >1 anak). Menambahkannya nanti murah: cukup tabel join `wali_siswa` baru, tidak menyentuh tabel siswa |
| 2026-08-02 | **Auth pakai Cognito User Pool** | Memperkuat narasi full-native-AWS. Konsekuensi: dua sumber data (Cognito + Postgres), disempitkan lewat satu kolom `cognito_sub` |
| 2026-08-02 | **Lifecycle rapor 3 tahap** (bukan 2 seperti wireframe) | Mengikuti success criteria charter: *controlled raport process*. Superadmin jadi reviewer |
| 2026-08-02 | **Wali kelas read-only atas nilai** | Wali kelas mengumpulkan & bertanggung jawab, tidak mengetik. Mencerminkan proses sekolah sungguhan |

### Known limitations (ditulis sadar, bukan didiamkan)

- **Audit jadi buram** — log tidak bisa membedakan siswa atau orang tua yang login, karena akunnya sama. Untuk pilot satu kelas ini wajar; harus disebut di laporan akhir
- PRD 5.5 mensyaratkan akses orang tua dibatasi pada anak terverifikasi. Dengan akun digabung, syarat ini terpenuhi secara tidak langsung (hanya bisa lihat 1 siswa), tapi bukan lewat mekanisme verifikasi eksplisit

---

## Pertanyaan Terbuka

Perlu dikonfirmasi ke pihak sekolah / PM. **Jangan ditebak sendiri** — semuanya mengubah tabel atau aturan izin.

| # | Pertanyaan | Terganjal apa |
|---|---|---|
| 1 | Bolehkah wali kelas **memperbaiki** nilai mapel lain saat rapor mau terbit? | Kalau boleh, matriks izin berubah dan butuh jejak audit khusus |
| 2 | Apakah ada **team teaching** (1 mapel 1 kelas diajar >1 guru)? | Kalau ya, `UNIQUE (mapel, kelas, periode)` di `penugasan` harus dilonggarkan |
| 3 | Bisakah satu guru jadi wali kelas di **lebih dari satu** kelas? | Saat ini diasumsikan maksimal satu per periode |
| 4 | Kalau guru mapel juga wali kelas di kelas yang sama, apakah dia boleh **menyetujui rapor siswanya sendiri**? | Saat ini tidak — persetujuan ada di Superadmin. Perlu dipastikan sekolah setuju |

---

## Berkas terkait

- `context/SCHEMA.md` — definisi tabel lengkap
- `context/API.md` — kontrak endpoint & bentuk response
- `context/PRD.md` — scope, requirement, known limitations
- `context/ARCHITECTURE.md` — Lambda, Cognito, VPC, batas modul
