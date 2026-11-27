# Design Spec — Sub-proyek 1: Ingestion + Core RAG Chat (Layanan TAK)

Date: 25 November 2026
Status: Draft — menunggu review pengguna
Source: Sesi brainstorming pada percakapan ini; rencana `.agents/1-PRD-UTUH.md` Rev 1 masih Draft / ACC Pending.

## 1. Scope dan keputusan

Sub-proyek pertama mencakup dua hal saja: **ingestion knowledge base** dan **core RAG chat dalam bentuk demo terminal**. Tidak termasuk database, API server, SSE, UI web, dashboard admin, autentikasi, riwayat percakapan, BM25/reranker penuh, dan evaluasi 50 pertanyaan. Semua itu sub-proyek berikutnya.

Keputusan yang disepakati dalam brainstorming:

| Keputusan | Pilihan | Alasan |
| --- | --- | --- |
| Lingkup | Ingestion + core RAG chat | Jantung produk; bisa diuji sebelum UI dibangun |
| LLM | `openai/gpt-oss-20b` via Groq free tier | Sesuai PRD, tanpa GPU lokal, biaya nol pada kuota gratis |
| Embedding | `nomic-embed-text-v1.5` via Groq `/openai/v1/embeddings` (Opsi 2) | Tidak ada model lokal yang diunduh; sesuai preferensi pengguna |
| Bukti selesai | Demo terminal: 5–10 pertanyaan nyata dijawab benar + sitasi + fallback (pilihan A) | Paling cepat dirasakan hasilnya untuk pemula |
| Prioritas sumber | Tanpa urutan kaku; prompt menginstruksikan LLM memilih: aturan/nilai/pasal/syarat → SK TAK; cara/verifikasi/error/prosedur internal → template | Menghindari false fallback pada pertanyaan kebijakan dan menghindari template mengalahkan aturan spesifik |

Batasan kuota Groq yang tercatat saat perencanaan: `openai/gpt-oss-20b` sekitar 30 RPM, 1.000 request/hari, 8K TPM, 200K token/hari. Bila kuota terlampaui, utilisasi naik kelas berbayar dengan harga $0,075/M token input dan $0,30/M token output; *prompt caching* otomatis memberi penghematan 50% pada input yang di-cache. Angka ini dapat berubah dan diverifikasi ulang sebelum produksi.

## 2. Arsitektur dan komponen

Lima unit dengan satu tanggung jawab masing-masing:

| Unit | Tanggung jawab | Antarmuka |
| --- | --- | --- |
| `ingestion` | Membaca `SK_TAK_2025.md` dan `TemplateAI.csv`, memotong menjadi chunk, memanggil embedding Groq, menulis `data/store.json` | `build_store() -> None` |
| `retriever` | Meng-embedding pertanyaan, menghitung cosine similarity, mengambil 5 potongan teratas | `search(question: str, top_k: int = 5) -> list[Chunk]` |
| `prompter` | Menyusun system prompt berisi guardrails, instruksi pemilihan sumber, sitasi, dan fallback | `build_prompt(question: str, chunks: list[Chunk]) -> messages` |
| `chat` | Loop terminal: baca input → `search` → `build_prompt` → panggil Groq streaming → tampilkan jawaban | `run() -> None` |
| `config` | Menyimpan nama environment variable dan konstanta tanpa nilai | `GROQ_API_KEY`, `EMBED_MODEL`, `LLM_MODEL`, `MIN_SCORE` |

Bentuk `Chunk`: `{id, teks, metadata, skor}`. Metadata minimal memuat `jenis`, `sumber`, `bagian`, `tahun_berlaku`, `status_aktif`.

Alur data:

```
[SK_TAK_2025.md] ─┐
                  ├─ ingestion ─ embed(Groq) ─> data/store.json
[TemplateAI.csv] ─┘

[pertanyaan] ─ embed(Groq) ─ search(store.json) ─ Top-5
                                                   ↓
                      prompt + sitasi + fallback ─> Groq gpt-oss-20b
                                                   ↓
                                          jawaban di terminal
```

Konsekuensi: setiap perubahan dokumen berarti `build_store()` dijalankan ulang dan menghabiskan sebagian kuota embedding harian. File `store.json` dipilih agar hasil ingestion bisa dibaca langsung saat belajar.

## 3. Chunking dan format data

**`SK_TAK_2025.md`.** Pemotongan mengikuti judul Markdown (`##` / `###`), bukan batas karakter sembarangan. Setiap chunk menyimpan jejak judul pada `bagian`, misalnya `Lampiran III > 5. Kompetisi > Kompetisi Belmawa`. Target 500–800 token dengan overlap sekitar 100 token, dipotong hanya pada batas paragraf atau baris tabel agar baris tabel tidak terbelah. Untuk tabel panjang, header tabel disalin ke awal tiap potongan. Metadata: `jenis = "sk_tak"`, `sumber = "SK TAK 2025"`, `tahun_berlaku = 2025`, `status_aktif = true`.

**`TemplateAI.csv`.** Delimiter `;`, field multi-baris dibaca dengan `csv.reader`. Satu baris menjadi satu chunk dengan teks `Kategori: {kategori}\nBalasan: {template}`. Baris tanpa template dilewati dan dihitung sebagai `skipped`. Metadata: `jenis = "template"`, `sumber = "TemplateAI.csv"`, `bagian = kategori`.

**`data/store.json`.** Array of record, sengaja polos:

```json
{
  "id": "sk-0042",
  "teks": "...",
  "metadata": {"jenis": "sk_tak", "sumber": "SK TAK 2025", "bagian": "Pasal 4"},
  "embedding": [0.012, -0.331]
}
```

Dokumen dan pertanyaan wajib memakai model embedding yang sama agar vektornya sejajar. Permintaan embedding dikirim per batch kecil untuk menjaga kuota 8K TPM, dengan retry bila kena rate limit. Log helpdesk tidak di-ingest pada sub-proyek ini karena masih memerlukan anonimisasi.

## 4. Retrieval, prompt, dan error handling

Pertanyaan di-embedding dengan model yang sama, lalu dibandingkan dengan seluruh record memakai cosine similarity. Diambil 5 teratas yang `status_aktif = true`; potongan dengan skor di bawah `MIN_SCORE` (nilai awal 0,30, di-tuning lewat uji) dianggap tidak ada konteks. Setiap potongan membawa `jenis`/`sumber`/`bagian` agar prompt dapat menginstruksikan pemilihan sumber seperti tercantum pada bagian 1.

System prompt berisi empat aturan: jawab hanya dari konteks; sertakan sitasi `[Sumber: SK TAK 2025, Pasal 4]`; arahkan ke admin bila tidak tersedia; jangan mengarang. User message memuat pertanyaan dan 5 potongan bernomor. Bila tidak ada potongan yang lolos ambang, fallback dikembalikan langsung tanpa memanggil LLM.

Groq dipanggil dengan `stream=True` dan `temperature = 0,1`; token ditampilkan begitu tiba.

| Kondisi | Perlakuan |
| --- | --- |
| `GROQ_API_KEY` tidak ada | Pesan jelas, program berhenti sebelum proses apa pun |
| Rate limit / 429 | Retry maksimal 3 kali dengan jeda meningkat |
| Embedding sebagian gagal | Chunk dilewati dan dihitung sebagai error; `store.json` ditulis ke file sementara lalu diganti agar tidak korup |
| Pertanyaan kosong atau terlalu panjang | Ditolak sebelum embedding |
| Tidak ada konteks | Fallback + kontak resmi, tanpa panggilan LLM |

## 5. Kriteria selesai dan pengujian

Enam pertanyaan uji dipakai sebagai contoh dasar; target akhir 5–10 pertanyaan nyata:

| # | Pertanyaan | Sumber yang diharapkan |
| --- | --- | --- |
| 1 | "syarat SKPI angkatan 22 gimana?" | Template: SKPI Angkatan 22 |
| 2 | "nilai minimal TAK buat sarjana berapa?" | SK TAK 2025, Pasal 4 → 60 |
| 3 | "TAK wawasan kebangsaan bisa didapat dari mana?" | Template: TAK Wajib |
| 4 | "gimana cara nonaktifkan TAK?" | Template: nonaktifkan TAK |
| 5 | "apa sanksi pemalsuan dokumen TAK?" | SK TAK 2025, Pasal 7 |
| 6 | "kapan jadwal beasiswa cair?" | Fallback, bukan jawaban karangan |

Checklist penerimaan:

- [ ] `build_store()` selesai; jumlah chunk per sumber dan jumlah `skipped` tercetak
- [ ] Lima pertanyaan pertama terjawab tepat dengan sitasi yang benar
- [ ] Pertanyaan ke-6 menghasilkan fallback dengan kontak resmi dan tanpa panggilan LLM
- [ ] Jawaban muncul bertahap (streaming) di terminal
- [ ] Tidak ada email atau Ticket ID di `store.json`; API key tidak tercetak dan tidak masuk repo
- [ ] Tidak ada model embedding lokal yang diunduh

Perintah eksekusi mengikuti toolchain yang dipilih pada task T-02 di rencana; spec ini tidak mengandaikan nama perintah atau framework test tertentu.

## 6. Batasan dan risiko

Batasan sengaja di luar sub-proyek ini: riwayat percakapan dan query rewriting (#7), BM25 dan reranker versi penuh (#5), endpoint API/SSE (#8), widget web (#11), modul admin (#9/#12), dan evaluasi 50 pertanyaan dengan metrik ≥90% (#13).

Risiko: kuota 8K TPM membatasi jumlah chunk dan pertanyaan per hari; kualitas `nomic-embed-text-v1.5` untuk bahasa Indonesia belum teruji sampai enam pertanyaan dijalankan; daftar model Groq dapat berubah sehingga `EMBED_MODEL` perlu diverifikasi terhadap `GET /openai/v1/models`; ambang `MIN_SCORE` perlu disetel dari hasil uji agar tidak terlalu banyak fallback atau terlalu banyak jawaban salah.

## 7. Hubungan dengan rencana yang sudah ada

Spec ini menyempurnakan bagian awal `.agents/1-PRD-UTUH.md` Rev 1 (ingestion + core RAG). Setelah spec disetujui, rencana perlu direvisi untuk mencatat Opsi 2 (Groq untuk embedding dan generasi), penghapusan model embedding lokal dari sub-proyek pertama, dan definisi selesai berupa demo terminal. Revisi rencana dikerjakan terpisah setelah review spec ini; belum ada perubahan baseline yang disetujui.
