# payload.md — Kontrak Permintaan ke Elice ML API

Dokumen ini menetapkan bentuk permintaan yang sah ke endpoint model Elice AI Cloud (AKCF Korea-ASEAN Digital Academy). Isinya diverifikasi langsung terhadap endpoint pada 11 Agustus 2026; setiap klaim di bawah berasal dari respons nyata, bukan dari dokumentasi vendor.

## 1. Alamat

Elice memberikan sebuah **base URL**, bukan endpoint lengkap:

```
https://mlapi.run/{ENDPOINT_ID}
```

Base URL tidak menerima permintaan apa pun. POST langsung ke alamat tersebut menghasilkan `404` dengan `{"error":{"code":"model_not_found"}}`, yang menyesatkan karena kesalahan sebenarnya ada pada path, bukan pada nama model. Path OpenAI-compatible wajib ditambahkan.

| Path | Metode | Status | Keterangan |
|---|---|---|---|
| `/v1/chat/completions` | POST | Berlaku | Jalur utama. Gunakan ini kecuali ada alasan khusus |
| `/v1/responses` | POST | Berlaku | Bentuk ringkas: `input` berupa string, jawaban di `output_text` |
| `/v1/models` | GET | Berlaku | Mendaftar model yang diizinkan pada endpoint |
| `/v1/models/{model_id}` | GET | Berlaku | Rincian satu model |

## 2. Autentikasi

Kunci dikirim sebagai bearer token pada header `Authorization`. Kunci berbentuk JWT dan diterbitkan dari menu **Issue API Key** pada halaman model.

```
accept: application/json
content-type: application/json
Authorization: Bearer {ELICE_API_KEY}
```

Kunci tidak boleh ditulis di dalam kode sumber. Baca dari variabel lingkungan `ELICE_API_KEY`. Kunci yang pernah muncul di transkrip, log, atau riwayat percakapan dianggap bocor dan harus dicabut.

## 3. Model yang diizinkan

Endpoint menolak nama model di luar daftar yang terikat padanya, dengan `400` dan pesan yang menyebutkan daftar sahnya. Untuk endpoint Gemini 3.6 Flash:

| Nilai `model` | Keterangan |
|---|---|
| `gemini-3.6-flash` | Bentuk pendek, dianjurkan |
| `google/gemini-3.6-flash` | Bentuk berawalan penyedia, setara |

Daftar ini bersifat per-endpoint. Endpoint lain memiliki daftarnya sendiri, dan `GET /v1/models` adalah sumber kebenarannya.

## 4. Bentuk payload

```json
{
  "model": "gemini-3.6-flash",
  "messages": [
    { "role": "system", "content": "Jawab ringkas dalam bahasa Indonesia." },
    { "role": "user", "content": "Sebutkan 3 warna primer." }
  ],
  "max_tokens": 2000,
  "temperature": 0.3,
  "reasoning_effort": "low"
}
```

| Field | Wajib | Keterangan |
|---|---|---|
| `model` | Ya | Harus salah satu nilai pada Bagian 3 |
| `messages` | Ya | Format OpenAI. Peran `system`, `user`, dan `assistant` diterima |
| `max_tokens` | Praktis wajib | Lihat Bagian 5. Nilai kecil merusak jawaban |
| `temperature` | Tidak | Diterima |
| `reasoning_effort` | Tidak | `low`, `medium`, atau `high`. Lihat Bagian 5 |
| `stream` | Tidak | `true` menghasilkan SSE dengan `object: "chat.completion.chunk"` |
| `tools` | Tidak | Function calling. Jawaban memakai `finish_reason: "tool_calls"` |
| `response_format` | Tidak | `{"type":"json_object"}` menghasilkan JSON valid |

## 5. Anggaran token dan penalaran

Gemini 3.6 Flash adalah model penalaran. Token penalaran dihitung terhadap `max_tokens` yang sama dengan token jawaban, sehingga anggaran yang terlalu kecil menghabiskan seluruh kuota sebelum satu kalimat pun selesai ditulis.

| `max_tokens` | `finish_reason` | Hasil |
|---|---|---|
| 100 | `length` | Isi terpotong menjadi `"Mer"`; 96 token habis untuk penalaran |
| 300 | `length` | Jawaban terpotong di tengah kalimat |
| 2000 | `stop` | Jawaban utuh; 246 token terpakai |

Ketetapan: **`max_tokens` minimal 2000**, bahkan untuk pertanyaan yang jawabannya satu baris. Gejala khas anggaran kurang adalah `finish_reason: "length"` disertai `content` yang terpenggal beberapa huruf.

`reasoning_effort: "low"` menekan pemakaian menjadi sekitar 220 token dan merupakan setelan yang dianjurkan untuk permintaan sederhana. Nilai `"none"` **tidak didukung**: nilai tersebut diterima tanpa galat namun diabaikan, dan model tetap menalar. Jangan mengandalkannya untuk mematikan penalaran.

## 6. Bentuk respons

```json
{
  "id": "xed6avaQLNGhvr0PhKHYuQk",
  "object": "chat.completion",
  "model": "gemini-3.6-flash",
  "choices": [
    {
      "index": 0,
      "finish_reason": "stop",
      "message": {
        "role": "assistant",
        "content": "3 warna primer adalah: merah, kuning, dan biru.",
        "extra_content": { "google": { "thought_signature": "..." } }
      }
    }
  ],
  "usage": {
    "prompt_tokens": 18,
    "completion_tokens": 246,
    "total_tokens": 264,
    "cached_prompt_tokens": 0,
    "cache_write_tokens": 0
  }
}
```

Setiap balasan menyertakan `message.extra_content.google.thought_signature`. Pada percakapan berlanjut yang memakai function calling, nilai ini harus dikirim kembali apa adanya di dalam riwayat pesan. Menghapusnya memutus rantai penalaran antar giliran.

Blok `usage` memuat dua penamaan sekaligus: `prompt_tokens`/`completion_tokens` untuk kompatibilitas OpenAI, dan `input_tokens`/`output_tokens` untuk penamaan Google. Keduanya bernilai sama.

## 7. Perilaku galat

Gateway meneruskan field yang tidak dikenalnya langsung ke Google. Akibatnya galat muncul dalam dua format berbeda, dan keduanya harus ditangani:

| Sumber | Bentuk | Contoh pemicu |
|---|---|---|
| Gateway Elice | `{"error":{"message":..., "type":"invalid_request_error", "code":400}}` | Nama model di luar daftar |
| Google | `[{"error":{"code":400,"status":"INVALID_ARGUMENT",...}}]` — perhatikan pembungkus array | `thinking_config` yang tidak sah |

Contoh yang ditolak Google: `extra_body.google.thinking_config.thinking_budget: 0` menghasilkan `INVALID_ARGUMENT`, sedangkan `extra_body.google.thinking_config.thinking_level: "low"` diterima dan setara dengan `reasoning_effort: "low"`.

## 8. Acuan implementasi

```python
import os
import requests

BASE = "https://mlapi.run/{ENDPOINT_ID}"

payload = {
    "model": "gemini-3.6-flash",
    "messages": [
        {"role": "system", "content": "Jawab ringkas dalam bahasa Indonesia."},
        {"role": "user", "content": "Sebutkan 3 warna primer."},
    ],
    "max_tokens": 2000,
    "temperature": 0.3,
    "reasoning_effort": "low",
}

response = requests.post(
    f"{BASE}/v1/chat/completions",
    json=payload,
    headers={
        "accept": "application/json",
        "content-type": "application/json",
        "Authorization": f"Bearer {os.environ['ELICE_API_KEY']}",
    },
    timeout=60,
)
response.raise_for_status()

choice = response.json()["choices"][0]
if choice["finish_reason"] == "length":
    raise RuntimeError("max_tokens terlalu kecil; jawaban terpotong")

print(choice["message"]["content"])
```

Karena endpoint mengikuti kontrak OpenAI, klien resmi OpenAI juga dapat dipakai dengan `base_url=f"{BASE}/v1"`.

## 9. Daftar periksa sebelum integrasi

- [ ] Path `/v1/chat/completions` ditambahkan pada base URL
- [ ] Kunci dibaca dari `ELICE_API_KEY`, bukan ditulis di kode
- [ ] `max_tokens` minimal 2000
- [ ] `finish_reason: "length"` diperlakukan sebagai kegagalan, bukan jawaban sah
- [ ] Galat ditangani untuk kedua format pada Bagian 7
- [ ] `thought_signature` diteruskan kembali bila percakapan berlanjut dengan tools
