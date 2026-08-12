# Panduan Integrasi API — EduTrack

| Keterangan | Isi |
|---|---|
| **Tanggal** | 12 Agustus 2026 |
| **Kedudukan** | Menjelaskan **cara memanggil API yang sudah berjalan**. Berada di luar rantai penguncian dan **tidak menetapkan apa pun** |
| **Kontraknya** | [API.md](./API.md) — bentuk permintaan dan jawaban setiap endpoint. Dokumen ini tidak mengulanginya |
| **Untuk siapa** | Siapa pun yang menulis klien: frontend React, skrip runbook, maupun uji yang menembak lingkungan sungguhan |

> Apabila dokumen ini bertentangan dengan [API.md](./API.md) atau [ARCHITECTURE.md](./ARCHITECTURE.md), **keduanya yang berlaku**. Yang ada di sini hanyalah cara memakai apa yang sudah ditetapkan di sana.
>
> Seluruh contoh di bawah **dijalankan terhadap lingkungan yang berjalan pada 12 Agustus 2026**, bukan disusun dari pembacaan kode.

---

## 1. Alamat

```
https://d2mw289fm4g0eo.cloudfront.net
```

Sementara, sampai domain dibeli (**CK-17**). Frontend **tidak menuliskannya** — ia memanggil `/api/...` secara relatif, karena frontend dan backend berada pada satu domain ([ARCHITECTURE Pasal 3](./ARCHITECTURE.md)). Akibatnya tidak ada CORS, tidak ada preflight, dan tidak ada base URL yang berbeda antar lingkungan.

Alamat penuh di atas hanya dipakai klien di luar peramban.

---

## 2. ⚠️ Setiap request ber-body wajib membawa `x-amz-content-sha256`

**Ini bagian yang akan menggigit lebih dahulu daripada yang lain, dan gejalanya menyesatkan.**

| Request | Tanpa header | Dengan header |
|---|:--:|:--:|
| `GET` apa pun | **200** | 200 |
| `POST`, `PATCH`, `PUT` | **403** | 200 |

Seluruh jalur **baca** berjalan sempurna tanpa header ini. Yang mati hanya jalur **tulis** — masuk, simpan nilai, presensi, finalisasi. Kalau aplikasi Anda dapat menampilkan data tetapi tidak dapat menyimpan apa pun, mulailah dari sini.

Nilainya **SHA-256 heksadesimal huruf kecil dari body persis seperti yang dikirim**. Sebabnya bukan pilihan rancangan kami: Lambda Function URL tidak menerima `UNSIGNED-PAYLOAD`, dan Origin Access Control mengambil sidik jari payload dari header itu alih-alih menghitungnya sendiri (**CK-A-12**).

`403` itu datang dari Lambda, bukan dari aplikasi, sehingga **badannya bukan amplop `kesalahan`** dan pesannya berbahasa Inggris:

```json
{"Message":"Forbidden. For troubleshooting Function URL authorization issues, see: …"}
```

### Bungkus sekali, pakai di mana-mana

Jangan menghitungnya di setiap pemanggilan. Satu pembungkus, dan seluruh mutasi lewat situ:

```ts
async function sidikJari(badan: string): Promise<string> {
  const bita = new TextEncoder().encode(badan);
  const cerna = await crypto.subtle.digest("SHA-256", bita);
  return [...new Uint8Array(cerna)].map((b) => b.toString(16).padStart(2, "0")).join("");
}

export async function panggil(jalur: string, pilihan: RequestInit = {}): Promise<Response> {
  const kepala = new Headers(pilihan.headers);
  const badan = pilihan.body;

  if (typeof badan === "string") {
    kepala.set("content-type", "application/json");
    kepala.set("x-amz-content-sha256", await sidikJari(badan));
  }

  // `same-origin` sudah bawaan, tetapi ditulis supaya cookie sesi tidak hilang
  // ketika seseorang menyalin pembungkus ini ke tempat lain.
  return fetch(jalur, { ...pilihan, headers: kepala, credentials: "same-origin" });
}
```

`crypto.subtle` hanya tersedia pada **secure context**. Terpenuhi di produksi karena seluruh trafik HTTPS; pada pengembangan lokal, `http://localhost` juga dihitung secure oleh peramban.

### Di on-prem header ini tidak diperlukan — dan tidak mengganggu

Caddy menggantikan CloudFront dan tidak memeriksa header apa pun. Pembungkus yang sama berjalan di kedua lingkungan **tanpa percabangan**; jangan menambahkan pemeriksaan lingkungan untuk melewatinya.

### Unggah berkas: hitung sendiri body multipart-nya

Ini yang paling merepotkan, dan tidak ada jalan pintasnya. `FormData` yang diserahkan ke `fetch` disusun peramban beserta *boundary* yang dibangkitkannya sendiri — Anda **tidak dapat mengetahui bita persisnya**, sehingga tidak dapat menghitung sidik jarinya.

Susun sendiri:

```ts
export async function unggah(jalur: string, berkas: File, bidang: Record<string, string> = {}) {
  const batas = `----edutrack${crypto.randomUUID()}`;
  const potongan: BlobPart[] = [];

  for (const [nama, nilai] of Object.entries(bidang)) {
    potongan.push(`--${batas}\r\nContent-Disposition: form-data; name="${nama}"\r\n\r\n${nilai}\r\n`);
  }
  potongan.push(
    `--${batas}\r\nContent-Disposition: form-data; name="berkas"; filename="${berkas.name}"\r\n` +
      `Content-Type: ${berkas.type || "application/octet-stream"}\r\n\r\n`,
  );
  potongan.push(berkas, `\r\n--${batas}--\r\n`);

  // Bita persisnya diambil dari Blob yang sama yang akan dikirim — bukan
  // disusun ulang. Menyusunnya dua kali adalah cara termudah menghasilkan
  // sidik jari yang tidak cocok.
  const badan = new Blob(potongan);
  const cerna = await crypto.subtle.digest("SHA-256", await badan.arrayBuffer());

  return fetch(jalur, {
    method: "POST",
    body: badan,
    credentials: "same-origin",
    headers: {
      "content-type": `multipart/form-data; boundary=${batas}`,
      "x-amz-content-sha256": [...new Uint8Array(cerna)]
        .map((b) => b.toString(16).padStart(2, "0"))
        .join(""),
    },
  });
}
```

Batas ukuran unggahan **2 MB** ([ARCHITECTURE Pasal 7](./ARCHITECTURE.md)).

---

## 3. Autentikasi

Cookie sesi, bukan token di header. Frontend tidak pernah menyentuhnya.

```bash
curl -c kuki.txt -X POST "$DASAR/api/auth/masuk" \
  -H 'content-type: application/json' \
  -H "x-amz-content-sha256: $(printf '%s' "$BADAN" | shasum -a 256 | cut -d' ' -f1)" \
  -d "$BADAN"
```

| Ketentuan | Nilai |
|---|---|
| Nama cookie | `edutrack_sesi` |
| Sifat | `HttpOnly; Secure; SameSite=Strict` — **tidak terbaca JavaScript** |
| Umur | **12 jam, tanpa perpanjangan otomatis** |
| Pencabutan | Berlaku seketika pada request berikutnya |
| Keluar | `POST /api/auth/keluar` |

Karena `HttpOnly`, tidak ada gunanya mencoba membaca sesi dari JavaScript — dan itu memang maksudnya ([CK-A-04](./ARCHITECTURE.md)). Cara mengetahui siapa yang sedang masuk adalah memanggil `GET /api/saya`.

Tanpa sesi:

```json
{"kesalahan":{"kode":"SESI_TIDAK_SAH","pesan":"Sesi tidak ditemukan. Silakan masuk kembali."}}
```

Kredensial salah — perhatikan pesannya **tidak** membedakan nama pengguna yang tidak ada dari kata sandi yang keliru:

```json
{"kesalahan":{"kode":"KREDENSIAL_SALAH","pesan":"Nama pengguna atau kata sandi tidak sesuai."}}
```

**Tidak ada pemulihan mandiri.** Tombol lupa kata sandi hanya menampilkan pesan menghubungi Wali Kelas atau Administrator ([PRD §6.1.3](./PRD.md), AC-33).

---

## 4. Amplop jawaban

Dua bentuk, tidak pernah yang lain:

```json
{ "data": … }
```

```json
{ "kesalahan": { "kode": "…", "pesan": "…", "rincian": [ … ] } }
```

**`pesan` selalu berbahasa Indonesia dan siap ditampilkan apa adanya** ([CK-API-01](./API.md)). Jangan menerjemahkan ulang di frontend — tiga di antaranya ditetapkan PRD kata demi kata, dan menyusunnya di dua tempat membuat keduanya menyimpang.

`kode` dipakai menentukan **perlakuan**, bukan teks. Misalnya menyorot baris berkas yang gagal, atau memutuskan apakah perlu mengarahkan ke layar masuk.

| Kode | Status | Artinya bagi klien |
|---|:--:|---|
| `KREDENSIAL_SALAH` | 401 | Tampilkan pesannya; jangan bedakan sebabnya |
| `SESI_TIDAK_SAH` | 401 | Arahkan ke layar masuk |
| `KEWENANGAN_DITOLAK` | 403 | **403, bukan 404** — sumber dayanya ada, penggunanya yang tidak berhak |
| `PERMINTAAN_TIDAK_SAH` | 400 | Gagal validasi Zod; `rincian` menyebut bidangnya |
| `BATAS_LAJU_TERLAMPAUI` | 429 | Pesannya menyebut kapan dapat dicoba lagi |
| `TIDAK_DITEMUKAN` | 404 | — |
| `BERKAS_TIDAK_SAH`, `BERKAS_TERLALU_BESAR` | 400 | Unggahan ditolak **seluruhnya** ([CK-API-02](./API.md)) |
| `BOBOT_TIDAK_SERATUS` | 400 | Bobot komponen wajib berjumlah 100 |
| `RAPOR_TERKUNCI` | 409 | Rapor sudah final; statusnya tidak pernah mundur (I-21) |
| `MAPEL_BELUM_LENGKAP` | 409 | `rincian` menyebut mata pelajaran mana |
| `BERKAS_BELUM_SIAP` | 409 | Berkas rapor belum dirender; coba unduh lagi |
| `LAYANAN_AI_GAGAL` | 503 | **Kegagalan lunak** — jangan menghalangi apa pun (AC-21) |
| `KESALAHAN_SERVER` | 500 | — |

Setiap mutasi wajib menampilkan pemberitahuan berhasil atau gagal, dan kegagalannya menyebutkan alasannya (P21, AC-27).

---

## 5. Dua bentuk `429`, dan hanya satu yang beramplop

| Sumber | Bentuk | Kapan |
|---|---|---|
| Aplikasi | Amplop `kesalahan`, kode `BATAS_LAJU_TERLAMPAUI`, Bahasa Indonesia | Batas per pengguna terlampaui |
| **Lambda** | Bukan amplop, Bahasa Inggris | Lebih dari sepuluh request berjalan **serentak** di seluruh sistem |

Yang kedua akibat plafon concurrency akun (**CK-A-11**). Praktis mustahil pada 18 guru, tetapi klien tetap wajib menanganinya sebagai kegagalan beralasan alih-alih menabrak `undefined` saat mencari `.kesalahan.pesan`.

Batas per pengguna yang ditetapkan [ARCHITECTURE Pasal 7](./ARCHITECTURE.md):

| Jalur | Batas |
|---|---|
| Masuk | 5 kegagalan per akun per 15 menit · 30 kegagalan per alamat IP per 15 menit |
| Tombol Suggestion | 5 kali per jam per siswa |
| Unggah berkas | 10 unggahan per jam per pengguna |

---

## 6. Berkas rapor tidak dikembalikan sebagai isi respons

Jawabannya **tautan bertanda tangan berumur 5 menit**, dan peramban mengunduh langsung dari S3 — isinya tidak pernah melewati proses aplikasi ([ARCHITECTURE §11.1](./ARCHITECTURE.md)).

Akibatnya bagi klien:

1. Tautan **kedaluwarsa**. Jangan disimpan, jangan dijadikan `href` yang bertahan lama. Kalau pengguna menekan unduh lagi, minta tautan baru.
2. Unduhan **tidak membawa cookie sesi**, dan memang tidak perlu — tanda tangannya yang menjadi kewenangan.
3. Berkas yang belum pernah dirender akan dirender saat itu juga, sehingga permintaan pertama lebih lambat.

---

## 7. Yang paling mudah salah dipahami

**`GET /api/healthz` melewati CloudFront, `/healthz` tidak.** Hanya `/api/*` yang diteruskan ke Lambda; `/healthz` di akar dilayani bucket frontend dan akan menjawab `200` berisi HTML — pemeriksaan yang selalu lulus tanpa memeriksa apa pun.

**Kewenangan diperiksa dua lapis**, dan lapis kedua tidak terlihat dari peran ([ARCHITECTURE §9.2](./ARCHITECTURE.md)). Guru yang sama boleh mengisi nilai pada kelas yang diampunya, dan ditolak pada kelas lain. Tidak ada peran bernama `wali_kelas`: kewenangan Wali Kelas seluruhnya diturunkan dari `kelas.wali_kelas_ref` (CK-A-01).

**Matriks nilai diputar di frontend.** Server menyimpannya memanjang, satu baris per nilai; 30 siswa × 8 komponen berarti 240 sel ([ARCHITECTURE Pasal 4](./ARCHITECTURE.md)).

**Simpan Nilai bersifat atomik.** Satu penekanan tombol = satu request = satu transaksi. Kegagalan membatalkan seluruh barisnya, bukan sebagian (C-02, AC-15). Jangan memecahnya menjadi banyak request kecil.

**Tidak ada penyimpanan otomatis.** Keadaan formulir bersifat lokal sampai tombol simpan ditekan; meninggalkan halaman tanpa menyimpan tidak mengubah data (P22, AC-15).

**Keluaran AI tidak pernah disimpan.** Ia ditampilkan sekali dan hilang. Jangan meng-cache-nya di klien maupun mengirimkannya kembali ke server (I-24, NG14, AC-16).

---

## 8. Daftar alamat

Bentuk permintaan dan jawaban setiap alamat ada pada [API.md](./API.md); yang di bawah hanya peta.

| Kelompok | Alamat |
|---|---|
| Kesehatan | `GET /api/healthz` |
| Sesi | `POST /api/auth/masuk` · `POST /api/auth/keluar` |
| Diri sendiri | `GET /api/saya` · `PATCH /api/saya/kata-sandi` · `GET /api/saya/nilai` · `GET /api/saya/presensi` · `POST /api/saya/suggestion` |
| Akun | `GET POST /api/pengguna` · `POST /api/pengguna/unggah` · `PATCH /api/pengguna/:id/kata-sandi` |
| Periode | `GET POST /api/tahun-ajaran` · `POST /api/tahun-ajaran/:id/periode` · `PATCH /api/periode/:id/aktif` |
| Kurikulum | `GET POST /api/mapel` · `PATCH /api/mapel/:id` · `GET POST /api/komponen-penilaian` |
| Kelas | `GET POST /api/kelas` · `GET /api/kelas/:id` · `POST /api/kelas/pratinjau` |
| Nilai | `GET /api/penugasan/:id/siswa` · `PUT /api/penugasan/:id/nilai` · `GET /api/kelas/:id/nilai` |
| Presensi | `POST /api/penugasan/:id/sesi` · `GET PATCH DELETE /api/sesi/:id` · `PUT /api/sesi/:id/presensi` · `GET /api/kelas/:id/presensi` |
| Rapor | `GET /api/kelas/:id/rapor` · `POST /api/kelas/:id/rapor/finalisasi` · `POST /api/kelas/:id/rapor/distribusi` · `GET /api/kelas/:id/rapor/berkas` · `GET PATCH /api/rapor/:id` · `GET /api/rapor/:id/berkas` |
| Templat | `GET /api/templat/pengguna.csv?peran=guru\|siswa` · `GET /api/templat/daftar-siswa.xlsx` |

---

## 9. Contoh utuh yang benar-benar berjalan

Finalisasi sekelas, dari masuk sampai hasilnya — persis yang dipakai mengukur **B7**:

```bash
DASAR=https://d2mw289fm4g0eo.cloudfront.net
sha() { printf '%s' "$1" | shasum -a 256 | cut -d' ' -f1; }

BADAN=$(jq -nc --arg u "$PENGGUNA" --arg p "$SANDI" '{nama_pengguna:$u,kata_sandi:$p}')
curl -sS -c kuki.txt -X POST "$DASAR/api/auth/masuk" \
  -H 'content-type: application/json' -H "x-amz-content-sha256: $(sha "$BADAN")" -d "$BADAN"

curl -sS -b kuki.txt -X POST "$DASAR/api/kelas/$KELAS/rapor/finalisasi" \
  -H 'content-type: application/json' -H "x-amz-content-sha256: $(sha '{}')" -d '{}'
```

Jawabannya:

```json
{"data":{"difinalisasi":30,"difinalisasi_pada":"2026-08-12T07:21:27.900Z","berkas_terender":30}}
```

**`berkas_terender` boleh lebih kecil dari `difinalisasi`, dan itu bukan kegagalan.** Perenderan dibatasi anggaran lunak 20 detik; berkas yang belum sempat dirender diselesaikan jalur unduh ([API §8.3](./API.md)). Klien menampilkannya sebagai keterangan, bukan sebagai galat.

---

## Riwayat

| Tanggal | Perubahan |
|---|---|
| 12 Agustus 2026 | Dokumen dibuat sesudah rilis pertama dan pengukuran B7. Isinya lahir dari tiga hal yang tidak terbaca dari [API.md](./API.md) karena baru muncul ketika API benar-benar dipanggil lewat CloudFront: kewajiban `x-amz-content-sha256` (**CK-A-12**) beserta akibatnya pada unggahan multipart, dua bentuk `429` yang berbeda (**CK-A-11**), dan sifat tautan berkas rapor yang kedaluwarsa |
