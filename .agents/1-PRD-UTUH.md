# Development plan - RAG Chatbot Layanan TAK Telkom University

Revision: 2
Plan status: Approved for Rev 2 (sub-proyek 1)
Implementation authorization: Granted untuk T-02–T-06 sesuai Rev 2
Approved revision: 2
Approval date: 2026-11-27
Approval quotation/reference: "acc push ke ssues gantikan yang sebelumnya" — ACC diberikan pada sesi ini untuk Rev 2 dan penggantian konten 24 issue GitHub.
Authorized scope and delivery permissions: Setup repo (README commit, branch `main`, push ke `origin/main`) telah diperintahkan dan dilakukan. implementasi T-02–T-06 dan pembaruan issue GitHub diizinkan. Commit/push git berikutnya tetap hanya bila diperintahkan eksplisit.
Source brief/design: Percakapan ini — "pelajari konteks.pdf dan prd.md untuk mengetahui konteks ini"; "Project ini untuk membuat chatbot rag untuk layanan TAK ... dari kemahasiswaan telkom university"; "Adapaun rag nya dari SK-TAK dan list pertanyaan yang sudah dibuat"; "selanjutnya kita matengkan prd nya dengan skill dari opencode"; "matangkan dulu sub-proyek pertama = ingestion + core RAG chat"; "kita pake opsi 2". Bukti pendukung: `prd_rag_chatbot_layanan_kemahasiswaan.md` v1.1, `konteks.pdf`, `SK_TAK_2025.md`, `TemplateAI.csv`, log helpdesk CSV, dan spec `docs/superpowers/specs/2026-11-25-tak-rag-subproyek-1-design.md` (disetujui pengguna untuk dilanjutkan ke implementation plan).
Planning depth: Full

Rev 2 delta (disetujui 2026-11-27): sub-proyek 1 memakai Opsi 2 — embedding dan generasi keduanya via Groq (`nomic-embed-text-v1.5` + `openai/gpt-oss-20b`), tanpa model lokal dan tanpa pgvector/Redis pada tahap ini; bukti selesai berupa demo terminal enam pertanyaan uji; prioritas sumber diatur di prompt, bukan urutan kaku. T-02/T-03/T-04/T-05/T-06 pada Part C kini memiliki langkah implementasi terperinci di bawah. T-01 sebagian terjawab untuk runtime pengembangan; deployment produksi tetap terbuka.

Draft A/B/C together when inputs suffice. Approval locks the A/B baseline. Task progress may change within approved scope; material changes require an amendment and ACC. Keep unknowns explicit and do not use template values as evidence of authorization.

## PART A - Requirements

### A1. Problem, users and outcome

**Problem.** Layanan TAK (Transkrip Aktivitas Kemahasiswaan) Kemahasiswaan Telkom University menerima pertanyaan berulang dari mahasiswa tentang verifikasi pengajuan, TAK kolektif, SKPI, kategori/pemenuhan kegiatan, penolakan/revisi, dan error sistem. Log helpdesk yang tersedia berisi **4.028 tiket** dalam 9 kategori (Verifikasi TAK 1.099; TAK Kolektif 592; Pertanyaan Kategori Kegiatan TAK 528; Lain-Lain 503; SKPI 420; Banding Penolakan TAK 419; Eror Sistem TAK 355; TAK Kolektif UKM 97; TAK Kolektif UKM/Unit/Direktorat 15). Penanganan manual memicu waktu tunggu panjang dan beban staf tinggi, terutama menjelang registrasi ulang dan yudisium.

**Users.**
- Mahasiswa aktif S1/D4, D3, S1 Ekstensi, S1 PJJ, dan mahasiswa asing (pengguna utama; kebutuhan berbeda pada nilai minimal TAK dan kegiatan wajib).
- Calon mahasiswa / mahasiswa baru yang memahami skema TAK angkatan 2022 ke atas.
- Staf Admin Layanan TAK: kurator balasan, validator akurasi, pengelola dokumen SOP/SK.
- Unit IT kampus: penyedia server, domain, dan sertifikat SSL.

**Primary journey.** Mahasiswa mengetik pertanyaan bahasa Indonesia sehari-hari → sistem mengambil konteks relevan dari SK TAK 2025 dan/atau template balasan resmi → menjawab secara streaming dengan sitasi → mahasiswa mengajukan pertanyaan lanjutan → mahasiswa memberi feedback. Jika tidak ada dasar di dokumen, sistem menolak spekulasi dan mengarahkan ke kanal resmi (SITU TAK, Student Affairs, Service Desk PuTI).

**Goals and observable success measures** (target diwariskan dari PRD v1.1; perlu disahkan pada ACC):
- Menurunkan pertanyaan berulang ke staf minimal 60% dalam 3 bulan pertama.
- First response time < 3 detik; TTFT ≤ 2,5 detik dan ≥ 20 token/detik.
- Faithfulness & context recall ≥ 90% pada 50 pertanyaan standar; halusinasi < 2%; thumbs-up ≥ 85%.
- Seluruh jawaban kebijakan bersumber dari SK TAK 2025 yang berlaku; seluruh jawaban operasional konsisten dengan template balasan admin.

### A2. Scope and constraints

**MVP in-scope (P0).**
- Knowledge base: `SK_TAK_2025.md` (aturan/resmi), `TemplateAI.csv` (19 intent + balasan siap pakai), log helpdesk yang dianonimisasi untuk evaluasi dan analisis intent.
- Pipeline ingestion untuk Markdown dan CSV dengan metadata `nama_dokumen`, `kategori`, `tahun_berlaku`, `status_aktif`, `jenis_sumber`.
- Hybrid search (vector + keyword/BM25), reranking, system prompt guardrails, sitasi, fallback, chat memory, dan streaming SSE.
- Web chat widget + dashboard admin sederhana (unggah/nonaktifkan dokumen, lihat log pertanyaan & feedback).
- Anonimisasi data pribadi pada log sebelum dipakai evaluasi atau disimpan.

**P1 / P2.**
- P1: analitik intent paling sering dan pertanyaan gagal dijawab sebagai bahan evaluasi SOP.
- P2: integrasi Telegram/WhatsApp bila diputuskan pada T-01.

**Explicit exclusions.**
- Persetujuan otomatis pengajuan TAK/SKPI, pencairan dana, atau transaksi di sistem akademik.
- Akses data nilai transkrip individu, tagihan personal, atau identitas mahasiswa tertentu.
- Aplikasi native Android/iOS.
- Balasan yang mengarang prosedur di luar SK TAK dan template resmi.

**Constraints.**
- **Anggaran terbatas; pengguna lebih memilih solusi gratis bila memungkinkan** (pernyataan pengguna pada percakapan ini). Konsekuensi: opsi deployment berbayar (GPU/on-premise, VPS, cloud GPU) bukan default; biaya menjadi kriteria pemilihan utama pada OD-01.
- Model `openai/gpt-oss-20b` membutuhkan VRAM ±16–24 GB (lokal) atau endpoint inference cloud; kapasitas paralel terbatas. Untuk jalur gratis, model ini harus diganti dengan model yang tersedia di layanan gratis (ketersediaan `gpt-oss-20b` di Workers AI belum terkonfirmasi dari dokumentasi) — perubahan ini bersifat material dan memerlukan ACC.
- Dokumen PDF buram memerlukan OCR tambahan (di luar MVP; SK TAK 2025 sudah berupa teks).
- Tenggat: sistem siap sebelum registrasi ulang semester baru.
- Kepatuhan UU PDP: email dan Ticket ID pada log helpdesk tidak boleh disimpan mentah.

**Dependencies.**
- `SK_TAK_2025.md`, `TemplateAI.csv`, dan log CSV sudah tersedia di repo (belum ter-commit kecuali README).
- Keputusan skema deployment (on-premise / hybrid / full cloud) dan kanal rilis belum diambil.
- Ketersediaan server/GPU atau kredensial cloud inference dari Unit IT.
- Waktu staf admin untuk UAT dan validasi jawaban.

**Assumptions (label).**
- **AS-01** SK TAK 2025 (KR.1149/KMH08/KKA-PKM/2025, 4 Desember 2025) adalah aturan berlaku dan mencabut KR.362/MHS6/KMH-KKM/2022. Konsekuensi: filter `status_aktif`/`tahun_berlaku` wajib.
- **AS-02** `TemplateAI.csv` adalah balasan yang disetujui admin. Konsekuensi: template boleh dijadikan sumber jawaban operasional.
- **AS-03** Log helpdesk boleh dipakai sebagai dataset evaluasi setelah dianonimisasi. Konsekuensi: T-04/T-13 harus menghapus email & Ticket ID.
- **AS-04** Target metrik PRD v1.1 masih berlaku untuk proyek TAK. Konsekuensi: T-13/T-14 mengukur terhadap target tersebut.
- **AS-05** Tidak ada desain visual institusi yang harus diikuti selain identitas visual Telkom University; `.agents/2-DESIGN.md` belum diperlukan.
- **AS-06** Jika OD-01 memilih Opsi D, batas tier gratis Cloudflare yang tercatat saat perencanaan: Workers 100.000 request/hari, 10 ms CPU/request, 128 MB memori, 50 subrequest/request; Workers AI 10.000 neuron/hari; R2 10 GB-month + 1 juta operasi Class A + 10 juta Class B/bulan (Standard, egress gratis); D1 5 GB, 5 juta baris dibaca/hari, 100.000 baris ditulis/hari, 10 database; Vectorize 30 juta dimensi query/bulan dan 5 juta dimensi tersimpan. Angka dapat berubah; verifikasi ulang sebelum ACC, dan jangan menjanjikan "gratis selamanya" tanpa pengecekan. Konsekuensi: arsitektur harus streaming, ringan CPU, dan menghindari full-table scan.

### A3. Requirements and acceptance

| ID | Requirement / user outcome | Priority | Observable acceptance criteria |
| --- | --- | --- | --- |
| FR-01 | Mahasiswa bertanya bahasa Indonesia sehari-hari/formal dan menerima jawaban streaming | P0 | Pertanyaan "gimana cara lihat poin TAK saya?" menghasilkan jawaban bertahap (SSE) dalam bahasa Indonesia, bukan error; istilah teknis dijelaskan bila muncul |
| FR-02 | Sitasi sumber pada setiap jawaban | P0 | Jawaban kebijakan menyebut nama dokumen + pasal/lampiran (mis. "SK TAK 2025, Lampiran II"); jawaban operasional menyebut kategori template; tidak ada sitasi palsu |
| FR-03 | Konteks percakapan / follow-up | P0 | Setelah bertanya "syarat SKPI", pertanyaan "kalau angkatan 22 gimana?" dijawab menggunakan konteks sesi yang sama |
| FR-04 | Quick prompts topik populer TAK | P0 | Tampilan awal menampilkan 5 topik dari kategori log teratas; menekan tombol mengisi dan mengirim pertanyaan |
| FR-05 | Fallback & eskalasi | P0 | Pertanyaan di luar SK TAK/template (mis. "kapan beasiswa cair?") dijawab dengan penolakan sopan + tautan SITU TAK / kontak Student Affairs / Service Desk PuTI; tidak ada jawaban spekulatif |
| FR-06 | Umpan balik jempol atas/bawah | P0 | Setiap balasan memiliki tombol feedback; klik tercatat di log anonim dan tampil di dashboard admin |
| FR-07 | Autentikasi admin & RBAC | P0 | Endpoint admin menolak akses tanpa peran `admin`; pengguna biasa hanya bisa chat |
| FR-08 | Unggah & pemrosesan dokumen | P0 | Admin mengunggah PDF/DOCX/MD/CSV beserta metadata; sistem membuat chunk + embedding dan mengembalikan jumlah chunk yang masuk |
| FR-09 | Pembaruan/penghapusan dokumen | P0 | Admin dapat menonaktifkan SK/versi lama; pencarian berikutnya tidak mengembalikan chunk dokumen tersebut |
| FR-10 | Log & analitik pertanyaan | P1 | Dashboard menampilkan top-N pertanyaan, pertanyaan gagal dijawab, dan ringkasan feedback; seluruh entri tanpa email/Ticket ID |
| FR-11 | Cakupan dua sumber pengetahuan | P0 | Pertanyaan kebijakan (nilai minimal, kegiatan wajib, sanksi) dijawab dari SK TAK; pertanyaan operasional (verifikasi, error, SKPI, kolektif) dijawab dari template; bentrok diselesaikan dengan prioritas SK TAK untuk kebijakan |
| FR-12 | Anonimisasi log helpdesk | P0 | Dataset evaluasi dan log aplikasi tidak mengandung alamat email atau pola Ticket ID (`A271125-2`); pemeriksaan otomatis lolos |
| FR-13 | Ketersediaan & Performa | P1 | 50 pengguna bersamaan tanpa degradasi signifikan; TTFT ≤2,5s; uptime 99,5% dengan maintenance 00.00–04.00 WIB |
| FR-14 | Keamanan & kepatuhan | P0 | HTTPS/TLS 1.3; input divalidasi; prompt injection ditolak; tidak ada data sensitif mahasiswa di log |
| FR-15 | Antarmuka responsif & identitas visual | P1 | Widget nyaman di browser ponsel dan desktop; kontras teks memadai; mengikuti identitas visual Telkom University |
| FR-16 | Deployment & pemulihan | P1 | Seluruh layanan dalam Docker Compose di balik reverse proxy HTTPS; dokumen dapat dinonaktifkan/dipulihkan tanpa rebuild |

Catatan: FR-01–FR-10 diadaptasi dari PRD v1.1; FR-11–FR-16 ditambahkan oleh penyesuaian scope TAK dan hasil inspeksi data.

### A4. Open decisions and risks

**Open decisions (material, blocking).**
- **OD-01** Skema deployment. **Untuk runtime pengembangan/sub-proyek 1 sudah diputuskan: Groq free tier + `openai/gpt-oss-20b` + embedding `nomic-embed-text-v1.5` (Opsi 2)**, disetujui pengguna. Untuk **deployment produksi** masih terbuka: on-premise GPU, hybrid, full cloud, atau stack gratis Cloudflare. Yang masih memengaruhi T-15/T-16 adalah keputusan produksi tersebut; T-03 tidak lagi menunggu OD-01 untuk tahap ini.
- **OD-02** Kanal rilis pertama: web portal saja, atau sekalian WhatsApp/Telegram. Memengaruhi T-01/T-12 dan scope P2.
- **OD-03** Apakah 19 template di `TemplateAI.csv` sudah final dan ditandatangani admin (AS-02 perlu konfirmasi). Memengaruhi FR-11/T-04.

**Risks and mitigations.**
- RSK-01 Halusinasi (diwariskan): system prompt ketat, reranker, sitasi wajib, temperature 0,1.
- RSK-02 Informasi kedaluwarsa: metadata `tahun_berlaku`/`status_aktif`, filter dokumen aktif; SK TAK 2025 mencabut aturan 2022.
- RSK-03 Lonjakan trafik: semantic cache Redis untuk pertanyaan berulang.
- RSK-04 Tabel PDF rusak: SK TAK sudah dikonversi ke Markdown terstruktur; Docling/PyMuPDF4LLM untuk dokumen baru.
- RSK-05 (baru) PII pada log: anonimisasi sebelum evaluasi, pemeriksaan otomatis pada T-04/T-13, tidak commit log mentah ke repo publik.
- RSK-06 (baru) Inkonsistensi sumber: Lampiran II vs IV SK TAK berbeda pada nilai TAK pilihan D3 (5 vs 4). Mitigasi: T-04 mencatat sebagai catatan kurasi; T-09 memungkinkan admin mengoreksi/menonaktifkan potongan; validasi staf pada UAT.
- RSK-07 (baru) Tidak ada test framework yang dikonfirmasi di repo. Mitigasi: T-02 menetapkan toolchain sebelum perintah verifikasi ditulis; tidak mengarang perintah.

## PART B - Technical approach

### B1. Context and architecture

**Verified repo state.** `D:\Project\chatbot-tak` adalah repo git baru; branch `main` tracking `origin/main` (commit README). Belum ada kode aplikasi, `pyproject.toml`, atau `package.json`. File yang ada: `README.md`, `konteks.pdf`, `prd_rag_chatbot_layanan_kemahasiswaan.md`, `SK TAK 2025.pdf`, `SK_TAK_2025.md`, `TemplateAI.csv`, log helpdesk `.csv` dan `.xlsx`. Tool Python tersedia: `pymupdf`, `pymupdf4llm`, `pypdf`; `docling` dan `openpyxl` belum terpasang.

**Stack (diwariskan PRD v1.1, belum dikunci oleh ACC).**
LLM `openai/gpt-oss-20b` via vLLM/Ollama atau cloud inference; embedding `BAAI/bge-m3`; reranker `BAAI/bge-reranker-v2-m3`; PostgreSQL + pgvector; Redis; FastAPI + LlamaIndex/LangChain; Next.js + Tailwind CSS; Docker Compose + Nginx/Caddy.

**Stack yang dipakai sub-proyek 1 (Opsi 2, disetujui):** Python 3.14 + SDK `groq`, embedding `nomic-embed-text-v1.5` via `POST /openai/v1/embeddings`, generasi `openai/gpt-oss-20b` via `POST /openai/v1/chat/completions` dengan `stream=True`, penyimpanan vektor sementara di `data/store.json`, antarmuka terminal. Tidak ada model lokal, database, atau server.

**Stack alternatif untuk deployment produksi (masih menunggu OD-01):** Workers untuk API/SSE, Workers AI/endpoint gratis, Vectorize, D1/KV/Durable Objects, R2 untuk dokumen/aset, Pages untuk UI, Wrangler untuk deploy; atau opsi A/B/C dari PRD. R2 saja tidak dapat menjalankan backend/LLM (object storage statis).

**Data flow.**
- *Offline ingestion:* `SK_TAK_2025.md` + `TemplateAI.csv` + log anonim → parser/loader per `jenis_sumber` → chunking (500–800 token, overlap 100, potong per header/pasal/baris template) → metadata → embedding → pgvector.
- *Online inference:* pertanyaan → cek Redis (sesi + semantic cache) → query rewriting bila follow-up → hybrid search Top-20 (dense + BM25, filter dokumen aktif) → reranker Top-5 → system prompt guardrail + sitasi → `gpt-oss-20b` streaming SSE → feedback ke log anonim.

**Component responsibilities (additions/edits yang direncanakan).**
- `ingestion/`: loader Markdown/CSV, normalisasi metadata, anonymizer, embedding & upsert ke pgvector.
- `backend/`: retrieval, reranking, prompting, session/memory, endpoint SSE & feedback, admin API.
- `frontend/`: chat widget, quick prompts, render Markdown, feedback, dashboard admin.
- `docs/`: panduan admin dan evaluasi. Folder ini berasal dari rencana `konteks.pdf`, bukan nama yang mengarang.

**Planned file structure (proposal, dikunci setelah ACC).** `docker-compose.yml` (DB/cache/LLM + reverse proxy); `pyproject.toml` & `.gitignore` (toolchain backend, pengecualian log mentah); `ingestion/` berisi `load_sources.py`, `anonymize.py`, `chunk_and_embed.py`; `backend/` berisi modul `retrieval.py`, `prompt.py`, `session.py`, `api/chat.py`, `api/admin.py`; `frontend/` berisi halaman chat & halaman admin; `eval/` berisi dataset JSONL anonim; `docs/` berisi panduan admin. Nama file ini adalah rencana berdasarkan struktur yang sudah disebut di `konteks.pdf`, bukan klaim bahwa file sudah ada.

### B2. Interfaces, data and security

**Interfaces (kontrak ringkas; detail request/response dirinci di T-08/T-09 setelah toolchain dipilih).**
- `POST /api/chat` — SSE streaming, body memuat `session_id` dan `message`; respons memuat token + sitasi.
- `POST /api/feedback` — `session_id`, `message_id`, `rating` (up/down).
- `GET/POST/PATCH /api/documents` — daftar, unggah, nonaktifkan dokumen + metadata (khusus `admin`).
- `GET /api/logs` — ringkasan pertanyaan & feedback anonim (khusus `admin`).

**Data / schema implication.**
- `documents`: id, nama, kategori, tahun_berlaku, status_aktif, jenis_sumber, checksum.
- `chunks`: id, document_id, content, embedding, metadata (pasal/lampiran/kategori template/halaman).
- `sessions` & `messages`: riwayat chat; `feedback`: rating per message; `query_logs`: pertanyaan anonim + status dijawab/gagal.
- Tidak ada migrasi produksi (belum ada database). Anonimisasi menjadi precondition sebelum log CSV dijadikan dataset.

**Auth, validation, failure handling.**
- RBAC dua peran (`mahasiswa`, `admin`); endpoint admin menolak peran lain (401/403).
- Validasi panjang input, sanitasi, dan guardrail prompt injection; instruksi sistem mengikat jawaban pada konteks.
- Jika retrieval kosong atau confidence rendah → jawaban fallback + kontak resmi; tidak mengarang.
- Error LLM/vector DB → pesan ramah + log internal (tanpa PII).

**Omission reason.** Skema tabel SQL lengkap dan kontrak JSON detail tidak ditulis sekarang karena belum ada kode/service yang disepakati; penulisan awal akan mengarang struktur. Akan dihasilkan pada T-02/T-08 setelah keputusan OD-01/OD-02.

### B3. Decisions and dependencies

| Decision | Chosen approach | Alternatives / reason | Risk / mitigation |
| --- | --- | --- | --- |
| D-01 Format knowledge base | SK TAK → Markdown; template & log → CSV/JSONL langsung (tanpa `.md`) | Mengubah semua ke `.md` menambah duplikasi dan kehilangan struktur kolom `Kategori`/`Template Balasan` | Loader CSV harus menangani delimiter `;` dan field multi-baris; diuji di T-04 |
| D-02 Sumber kebijakan | SK TAK 2025 sebagai aturan berlaku; metadata `status_aktif`/`tahun_berlaku` | Tanpa filter, aturan 2022 ikut terambil | RSK-02; filter aktif + validasi admin T-09 |
| D-03 Retrieval | Hybrid search (pgvector + BM25) + `bge-reranker-v2-m3` Top-5 | Vector-only gagal pada kode/formulir/SITU/iGracias/iSAM | Biaya komputasi; semantic cache Redis |
| D-04 Guardrails | Jawab hanya dari konteks, wajib sitasi, fallback ke kontak resmi, temperature 0,1 | Jawaban bebas → halusinasi | RSK-01; red teaming T-14 |
| D-05 Privasi log | Anonimisasi email & Ticket ID sebelum evaluasi; log aplikasi tanpa PII | Menyimpan log mentah melanggar UU PDP | RSK-05; pemeriksaan otomatis T-13 |
| D-06 Deployment | Belum dipilih (OD-01); kandidat utama Opsi D tier gratis Cloudflare karena preferensi anggaran pengguna | Opsi A/B/C dari PRD (berbayar); Opsi D mengganti FastAPI→Workers, pgvector/Redis→Vectorize/D1/KV, LLM→Workers AI atau endpoint gratis, Docker→Wrangler | Blokir T-03/T-15; keputusan di T-01; perubahan model LLM bersifat material → ACC |
| D-07 Kanal rilis | Web widget sebagai MVP; WA/Telegram P2 (OD-02) | Bot pesan sejak awal menambah scope integrasi | Tidak ada; keputusan di T-01 |
| D-08 Penanganan beda nilai Lampiran II/IV | Simpan apa adanya + catatan kurasi; admin validasi saat UAT | Mengoreksi diam-diam mengubah dokumen resmi | RSK-06; T-04 mencatat, T-09/T-13 memvalidasi |
| D-09 Mode anggaran | Utamakan tier gratis/near-zero; biaya sebagai kriteria OD-01 | Opsi A/B/C berbayar; tetap bisa dipilih bila manfaatnya melebihi biaya dan disetujui | Batas gratis Workers (10 ms CPU, 100k req/hari) dan Workers AI (10k neuron/hari) dapat membatasi kualitas; validasi beban di T-14 |
| D-10 Sub-proyek 1 | Embedding + generasi via Groq; vektor di `data/store.json`; prioritas sumber di prompt | Embedding lokal (Opsi 1) atau keyword-only; pgvector sejak awal | Kuota 8K TPM; kualitas embedding ID belum teruji; `MIN_SCORE` perlu tuning; diverifikasi oleh 6 pertanyaan uji |

External dependencies and environment variable names (names/purpose only, no values): `DATABASE_URL` (PostgreSQL+pgvector), `REDIS_URL` (session & cache), `LLM_BASE_URL` (endpoint vLLM/Ollama/cloud), `LLM_API_KEY` (bila cloud inference), `EMBEDDING_MODEL`, `RERANKER_MODEL`, `ADMIN_PASSWORD_HASH` (autentikasi admin), `SITU_TAK_URL` / `SUPPORT_CONTACT` (tautan fallback). Nilai tidak disimpan di repo.

### B4. Verification and recovery

**Checks yang tersedia saat ini.** `git status --branch` untuk keadaan kerja; inspeksi file dengan Python (`pymupdf`, `pymupdf4llm`, `csv`) sudah digunakan untuk memvalidasi SK TAK dan data. Belum ada perintah test/lint karena belum ada toolchain; T-02 akan menetapkannya dan hanya perintah nyata yang akan dicatat sebagai verifikasi.

**Expected behavioral outcomes.** Jawaban mengandung sitasi yang bisa ditelusur; fallback muncul untuk pertanyaan di luar cakupan; tidak ada PII di log; pencarian tidak mengembalikan dokumen nonaktif; SSE menampilkan token bertahap.

**Manual checks.** Uji 5 quick prompts di desktop & ponsel; cek kontras dan fokus keyboard; uji feedback; uji unggah/nonaktifkan dokumen oleh admin dan penolakan oleh non-admin; uji red teaming (prompt injection, topik di luar TAK, aturan tahun lama).

**Unavailable tooling.** `docling` dan `openpyxl` belum terpasang — konversi SK TAK memakai `pymupdf4llm`, data memakai CSV (bukan xlsx langsung). `rapidocr` tidak ada, tetapi PDF SK TAK berisi teks sehingga OCR tidak diperlukan.

**Rollback.** Nonaktifkan dokumen lewat `status_aktif` (tanpa rebuild), kembalikan berkas dari git, dan backup tabel pgvector sebelum perubahan massal. Untuk deployment, reverse proxy dapat dialihkan ke versi sebelumnya bila rilis gagal.

## PART C - Tasks and progress

Statuses: Todo / In Progress / Blocked / Done. A task is Done only with acceptance evidence. Update details and the summary together. Dependency IDs refer to tasks below; no cycles.

| ID | Outcome | Requirement links | Dependencies | Status |
| --- | --- | --- | --- | --- |
| T-01 | Putuskan skema deployment & kanal rilis, catat di plan | A4, D-06, D-07 | None | Blocked |
| T-02 | Scaffold Python + pytest + SDK Groq (tanpa docker untuk sub-proyek 1) | FR-08, RSK-07 | None | Todo |
| T-03 | Konektivitas Groq & konfigurasi model (tanpa model lokal) | FR-01, D-10 | T-02 | Todo |
| T-04 | Ingestion SK TAK MD + TemplateAI.csv → `data/store.json` | FR-02, FR-08, FR-11, D-10 | T-02, T-03 | Todo |
| T-05 | Retrieval cosine Top-5 + filter skor/status (belum BM25/reranker) | FR-02, FR-11, D-10 | T-04 | Todo |
| T-06 | Prompt guardrail + sitasi + fallback + demo terminal streaming | FR-01, FR-02, FR-05, FR-11, FR-14 | T-05 | Todo |

Langkah implementasi terperinci untuk T-02–T-06 ada di bagian **"Detailed implementation plan — sub-proyek 1"** di bawah. T-01 dan T-07–T-16 tetap berupa ringkasan sampai sub-proyeknya masing-masing dibahas.
| T-07 | Chat memory & query rewriting via Redis | FR-03 | T-02, T-05 | Todo |
| T-08 | FastAPI `/api/chat` SSE + `/api/feedback` | FR-01, FR-06, FR-14 | T-06, T-07 | Todo |
| T-09 | Autentikasi admin + kelola dokumen (unggah/nonaktifkan) | FR-07, FR-08, FR-09 | T-02, T-04 | Todo |
| T-10 | Log & analitik pertanyaan anonim + ringkasan feedback | FR-10, FR-12 | T-08, T-09 | Todo |
| T-11 | Web chat widget: quick prompts, streaming Markdown, feedback, responsif | FR-01, FR-04, FR-06, FR-15 | T-08 | Todo |
| T-12 | Dashboard admin UI | FR-07, FR-09, FR-10 | T-09, T-10 | Todo |
| T-13 | Dataset evaluasi 30–50 pertanyaan + uji akurasi & anonimisasi | FR-02, FR-11, FR-12, AS-04 | T-04, T-06 | Todo |
| T-14 | Red teaming, uji performa & privasi | FR-13, FR-14, RSK-01, RSK-05 | T-08, T-13 | Todo |
| T-15 | Finalisasi Docker Compose + reverse proxy HTTPS | FR-16 | T-11, T-12, T-14, T-01 | Todo |
| T-16 | Dokumentasi admin & go-live | FR-16 | T-15 | Todo |

### T-01 - Putuskan skema deployment & kanal rilis
- Status: Blocked
- Requirements: A4, D-06, D-07
- Dependencies: None
- Files/components: `.agents/1-PRD-UTUH.md` (bagian A4/B3), tidak ada kode
- Acceptance criteria: OD-01 dan OD-02 terjawab dengan mempertimbangkan preferensi anggaran gratis; keputusan dan alasannya tercatat; revisi plan dibuat bila mengubah A/B secara material (termasuk penggantian model LLM bila memilih Opsi D)
- Verification: baca ulang A4/B3 dan cocokkan dengan pernyataan keputusan pengguna
- Evidence: Not run
- Blocker: OD-01 (on-premise/hybrid/full cloud) dan OD-02 (web saja / + WA/Telegram) belum dijawab
- Next action: Tanyakan OD-01/OD-02 setelah ACC; jangan mulai T-03/T-15 sebelum keputusan

T-02–T-06 untuk sub-proyek 1 tidak diuraikan di sini lagi. Detail langkah, file, test, dan acceptance criteria ada di bagian **"Detailed implementation plan — sub-proyek 1 (Rev 2)"** di bawah; bagian itu menggantikan uraian lama yang masih memakai docker-compose, pgvector, dan model lokal.

### T-07 - Chat memory & query rewriting
- Status: Todo
- Requirements: FR-03
- Dependencies: T-02, T-05
- Files/components: `backend/` modul session & Redis
- Acceptance criteria: Dua pertanyaan berurutan dalam satu `session_id` menghasilkan jawaban yang memakai konteks sebelumnya; sesi kedapatan tidak tercampur
- Verification: simulasi dua giliran chat; periksa query yang terekam
- Evidence: Not run
- Blocker: None
- Next action: Setelah T-02 & T-05

### T-08 - Endpoint FastAPI SSE & feedback
- Status: Todo
- Requirements: FR-01, FR-06, FR-14
- Dependencies: T-06, T-07
- Files/components: `backend/app` endpoint `/api/chat`, `/api/feedback`
- Acceptance criteria: `/api/chat` mengirim token bertahap via SSE beserta sitasi; `/api/feedback` mencatat rating; input tidak valid ditolak; tidak ada PII di respons
- Verification: uji endpoint dengan klien SSE; periksa entri feedback
- Evidence: Not run
- Blocker: None
- Next action: Setelah T-06 & T-07

### T-09 - Autentikasi admin & kelola dokumen
- Status: Todo
- Requirements: FR-07, FR-08, FR-09
- Dependencies: T-02, T-04
- Files/components: `backend/` auth + endpoint `/api/documents`
- Acceptance criteria: Non-admin ditolak (401/403); unggah menghasilkan chunk baru; menonaktifkan dokumen menghilangkannya dari hasil retrieval berikutnya
- Verification: uji dua peran; unggah lalu nonaktifkan satu dokumen uji; ulangi query
- Evidence: Not run
- Blocker: None
- Next action: Setelah T-02 & T-04

### T-10 - Log & analitik anonim
- Status: Todo
- Requirements: FR-10, FR-12
- Dependencies: T-08, T-09
- Files/components: `backend/` endpoint `/api/logs`, tabel `query_logs`
- Acceptance criteria: Dashboard/endpoint menampilkan top-N intent, pertanyaan gagal, dan ringkasan feedback; tidak ada email/Ticket ID
- Verification: query endpoint; pemeriksaan regex PII
- Evidence: Not run
- Blocker: None
- Next action: Setelah T-08 & T-09

### T-11 - Web chat widget
- Status: Todo
- Requirements: FR-01, FR-04, FR-06, FR-15
- Dependencies: T-08
- Files/components: `frontend/` (Next.js + Tailwind)
- Acceptance criteria: 5 quick prompts tampil dan berfungsi; jawaban Markdown (daftar/tabel) terbaca; indikator loading; tombol feedback; nyaman di layar ponsel; fokus keyboard terlihat; reduced motion dihormati
- Verification: uji manual desktop & ponsel; periksa fokus dan kontras
- Evidence: Not run
- Blocker: None
- Next action: Setelah T-08

### T-12 - Dashboard admin UI
- Status: Todo
- Requirements: FR-07, FR-09, FR-10
- Dependencies: T-09, T-10
- Files/components: `frontend/` halaman admin
- Acceptance criteria: Login admin; daftar dokumen + status aktif; unggah/nonaktifkan; tampilan analitik anonim
- Verification: uji alur login → unggah → nonaktifkan → lihat log
- Evidence: Not run
- Blocker: None
- Next action: Setelah T-09 & T-10

### T-13 - Dataset evaluasi & uji akurasi
- Status: Todo
- Requirements: FR-02, FR-11, FR-12, AS-04
- Dependencies: T-04, T-06
- Files/components: `docs/` atau `eval/` dataset JSONL anonim + skrip evaluasi
- Acceptance criteria: 30–50 pasangan pertanyaan/jawaban kunci dari SK TAK & template; metrik faithfulness/context recall ≥90%, halusinasi <2% (atau deviasi dicatat bila belum tercapai); nol PII
- Verification: jalankan evaluasi; simpan hasil ke `.agents/REVIEW.md` saat verifikasi
- Evidence: Not run
- Blocker: None
- Next action: Setelah T-04 & T-06

### T-14 - Red teaming, performa & privasi
- Status: Todo
- Requirements: FR-13, FR-14, RSK-01, RSK-05
- Dependencies: T-08, T-13
- Files/components: `backend/` pengamanan tambahan; laporan di `.agents/REVIEW.md`
- Acceptance criteria: Prompt injection, topik di luar TAK, dan aturan 2022 tidak menghasilkan jawaban salah; 50 pengguna bersamaan stabil; TTFT ≤2,5s; tidak ada PII
- Verification: uji red team; uji beban; tinjau log
- Evidence: Not run
- Blocker: None
- Next action: Setelah T-08 & T-13

### T-15 - Docker Compose final & reverse proxy HTTPS
- Status: Todo
- Requirements: FR-16
- Dependencies: T-11, T-12, T-14, T-01
- Files/components: `docker-compose.yml`, konfigurasi Nginx/Caddy
- Acceptance criteria: Seluruh layanan naik dari satu compose; HTTPS aktif; prosedur rollback teruji
- Verification: `docker compose up`; kunjungi domain; simulasi rollback
- Evidence: Not run
- Blocker: Menunggu OD-01
- Next action: Setelah T-11, T-12, T-14 & T-01

### T-16 - Dokumentasi admin & go-live
- Status: Todo
- Requirements: FR-16
- Dependencies: T-15
- Files/components: `docs/` panduan admin + catatan rilis
- Acceptance criteria: Panduan unggah/nonaktifkan dokumen dan baca analitik tersedia; staf admin terlatih; rilis dicatat
- Verification: UAT bersama staf admin; tanda tangan/serah terima dicatat sesuai bukti aktual
- Evidence: Not run
- Blocker: None
- Next action: Setelah T-15

## Detailed implementation plan — sub-proyek 1 (Rev 2)

> **For agentic workers:** REQUIRED SUB-SKILL: Gunakan `subagent-driven-development` (disarankan) atau `executing-plans` untuk mengerjakan tiap task. Langkah memakai checkbox `- [ ]`.

**Goal:** Demo terminal RAG TAK yang menjawab enam pertanyaan uji dengan sitasi dan fallback, memakai Groq untuk embedding dan generasi.

**Architecture:** `ingestion` membangun `data/store.json` dari `SK_TAK_2025.md` dan `TemplateAI.csv`; `retriever` mencari 5 potongan teratas dengan cosine similarity; `prompter` menyusun guardrail dan fallback; `chat` menjalankan loop terminal dengan streaming. Tanpa database, server, model lokal, atau UI.

**Tech Stack:** Python 3.14, `pytest`, `groq` SDK, `nomic-embed-text-v1.5`, `openai/gpt-oss-20b`.

### Global Constraints

- Python 3.14.5; `pip` 26.1.1; `pytest`, `groq`, `ruff`, `openai` belum terpasang pada saat perencanaan.
- Model embedding `nomic-embed-text-v1.5`; model LLM `openai/gpt-oss-20b`; `temperature = 0,1`; `stream = True`.
- `MIN_SCORE = 0,30`; `TOP_K = 5`; hanya record dengan `status_aktif = true` yang diambil.
- Kuota Groq free: 8K TPM / 200K token per hari; batch embedding maksimal 16 teks per permintaan; retry maksimal 3 kali.
- Kunci API hanya lewat environment variable `GROQ_API_KEY`; tidak pernah dicetak atau disimpan di repo.
- Tanpa model lokal, database, server API, dan UI pada sub-proyek ini.
- Commit/push hanya bila diperintahkan eksplisit; jangan otomatis.

---

### T-02 - Scaffold Python, pytest, dan SDK Groq

**Files:**
- Create: `pyproject.toml`, `.gitignore`, `config/__init__.py`, `config/settings.py`, `tests/test_smoke.py`, `data/.gitkeep`

**Interfaces:**
- Produces: `config.settings.EMBED_MODEL`, `LLM_MODEL`, `MIN_SCORE`, `TOP_K`, `TEMPERATURE`, `MAX_RETRIES`, `SOURCE_MD`, `SOURCE_CSV`, `STORE_PATH`.

- [ ] **Step 1: Tulis test yang gagal**

`tests/test_smoke.py`:

```python
from config.settings import EMBED_MODEL, LLM_MODEL, MIN_SCORE, STORE_PATH


def test_settings_constants():
    assert EMBED_MODEL == "nomic-embed-text-v1.5"
    assert LLM_MODEL == "openai/gpt-oss-20b"
    assert MIN_SCORE == 0.30
    assert STORE_PATH.as_posix() == "data/store.json"
```

- [ ] **Step 2: Pasang toolchain**

Run: `python -m pip install pytest groq`
Expected: `Successfully installed ... pytest ... groq ...`

- [ ] **Step 3: Jalankan test hingga gagal**

Run: `python -m pytest -q tests/test_smoke.py`
Expected: FAIL dengan `ModuleNotFoundError: No module named 'config'`

- [ ] **Step 4: Tulis implementasi minimal**

`pyproject.toml`:

```toml
[project]
name = "chatbot-tak"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["groq>=0.11"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

`.gitignore`:

```text
.venv/
__pycache__/
*.pyc
.env
.pytest_cache/
```

`config/__init__.py`: kosong.

`config/settings.py`:

```python
import os
from pathlib import Path

EMBED_MODEL = "nomic-embed-text-v1.5"
LLM_MODEL = "openai/gpt-oss-20b"
MIN_SCORE = 0.30
TOP_K = 5
TEMPERATURE = 0.1
MAX_RETRIES = 3
SOURCE_MD = Path("SK_TAK_2025.md")
SOURCE_CSV = Path("TemplateAI.csv")
STORE_PATH = Path("data/store.json")
GROQ_API_KEY = os.environ.get("GROQ_API_KEY")
```

`data/.gitkeep`: kosong.

- [ ] **Step 5: Jalankan test hingga lolos**

Run: `python -m pytest -q`
Expected: `1 passed`

- [ ] **Step 6: Cek manual konstanta**

Run: `python -c "from config.settings import EMBED_MODEL; print(EMBED_MODEL)"`
Expected: `nomic-embed-text-v1.5`

Acceptance: struktur folder terbentuk, pytest dan groq terpasang, konstanta sesuai spec, tidak ada kunci API di file.

---

### T-03 - Konektivitas Groq dan konfigurasi model

**Files:**
- Create: `config/groq_client.py`, `tests/test_groq_client.py`

**Interfaces:**
- Consumes: `config.settings.MAX_RETRIES`
- Produces: `make_client(api_key: str | None = None) -> Groq` — menaikkan `RuntimeError` berisi `GROQ_API_KEY` bila kunci tidak ada.

- [ ] **Step 1: Tulis test yang gagal**

`tests/test_groq_client.py`:

```python
import pytest
from config.groq_client import make_client


def test_make_client_raises_when_no_key(monkeypatch):
    monkeypatch.delenv("GROQ_API_KEY", raising=False)
    with pytest.raises(RuntimeError, match="GROQ_API_KEY"):
        make_client()


def test_make_client_uses_env(monkeypatch):
    monkeypatch.setenv("GROQ_API_KEY", "test-key")
    assert make_client().api_key == "test-key"
```

- [ ] **Step 2: Jalankan test hingga gagal**

Run: `python -m pytest -q tests/test_groq_client.py`
Expected: FAIL dengan `ModuleNotFoundError: No module named 'config.groq_client'`

- [ ] **Step 3: Tulis implementasi minimal**

`config/groq_client.py`:

```python
import os

from groq import Groq

from config.settings import MAX_RETRIES


def make_client(api_key=None):
    key = api_key or os.environ.get("GROQ_API_KEY")
    if not key:
        raise RuntimeError(
            "GROQ_API_KEY belum diset. Isi environment variable GROQ_API_KEY "
            "dengan kunci dari https://console.groq.com sebelum menjalankan program."
        )
    return Groq(api_key=key, max_retries=MAX_RETRIES)
```

- [ ] **Step 4: Jalankan test hingga lolos**

Run: `python -m pytest -q`
Expected: `3 passed`

- [ ] **Step 5: Uji koneksi nyata (manual, butuh kunci pengguna)**

Run: `python -c "from config.groq_client import make_client; print(make_client().models.list())"`
Expected: daftar model memuat id `openai/gpt-oss-20b` dan `nomic-embed-text-v1.5`. Bila tidak ada, catat sebagai blocker dan jangan lanjut ke T-04 sebelum model dikonfirmasi.

Acceptance: kunci tidak ada → pesan jelas; kunci ada → client Groq terbentuk dengan retry 3; daftar model sesuai konfirmasi manual.

---

### T-04 - Ingestion ke `data/store.json`

**Files:**
- Create: `ingestion/__init__.py`, `ingestion/chunker.py`, `ingestion/build_store.py`, `tests/test_chunker.py`, `tests/test_build_store.py`

**Interfaces:**
- Consumes: `config.settings` (`SOURCE_MD`, `SOURCE_CSV`, `STORE_PATH`, `EMBED_MODEL`)
- Produces: `chunk_markdown(text, source) -> list[dict]`, `chunk_templates(text) -> tuple[list[dict], int]`, `build_store(embed_fn=None, md_path=None, csv_path=None, out_path=None) -> dict` dengan `embed_fn(texts: list[str]) -> list[list[float]]`.

- [ ] **Step 1: Tulis test chunker yang gagal**

`tests/test_chunker.py`:

```python
from ingestion.chunker import chunk_markdown, chunk_templates


def test_chunk_markdown_keeps_heading():
    md = "# Judul\n\n## Pasal 4\nNilai minimal 60.\n\n### Lampiran II\nTabel syarat.\n"
    chunks = chunk_markdown(md)
    assert len(chunks) == 2
    assert chunks[0]["metadata"]["bagian"] == "Pasal 4"
    assert chunks[0]["metadata"]["sumber"] == "SK TAK 2025"
    assert "Nilai minimal 60" in chunks[0]["teks"]
    assert chunks[1]["metadata"]["bagian"] == "Lampiran II"


def test_chunk_templates_skips_header_and_empty():
    text = (
        "No.;Kategori Kendala Paling Umum;Template Balasan (Siap Digunakan)\n"
        "1;SKPI;Balasan A\n"
        "2;Terpenuhi;\n"
    )
    records, skipped = chunk_templates(text)
    assert len(records) == 1
    assert skipped == 1
    assert "Balasan A" in records[0]["teks"]
    assert records[0]["metadata"]["jenis"] == "template"


def test_chunk_markdown_splits_long_section_with_overlap():
    md = "## Pasal 4\n" + ("baris syarat\n" * 400)
    chunks = chunk_markdown(md)
    assert len(chunks) > 1
    assert all(c["metadata"]["bagian"] == "Pasal 4" for c in chunks)
    assert chunks[0]["teks"].startswith("Pasal 4")
```

- [ ] **Step 2: Tulis test build_store yang gagal**

`tests/test_build_store.py`:

```python
import json

from ingestion.build_store import build_store


def test_build_store_writes_records(tmp_path):
    md = tmp_path / "doc.md"
    md.write_text("## Pasal 4\nNilai 60.", encoding="utf-8")
    csvp = tmp_path / "t.csv"
    csvp.write_text(
        "No.;Kategori Kendala Paling Umum;Template Balasan (Siap Digunakan)\n"
        "1;SKPI;Balasan A\n",
        encoding="utf-8",
    )
    out = tmp_path / "store.json"

    stats = build_store(
        embed_fn=lambda texts: [[float(i)] for i in range(len(texts))],
        md_path=md,
        csv_path=csvp,
        out_path=out,
    )
    data = json.loads(out.read_text(encoding="utf-8"))

    assert stats == {"total": 2, "sk_tak": 1, "template": 1, "skipped": 0, "errors": 0}
    assert all("embedding" in r and "sumber" in r["metadata"] for r in data)
    assert "@" not in out.read_text(encoding="utf-8")


def test_build_store_skips_failed_embedding(tmp_path):
    md = tmp_path / "doc.md"
    md.write_text("## Pasal 4\nNilai 60.", encoding="utf-8")
    csvp = tmp_path / "t.csv"
    csvp.write_text(
        "No.;Kategori Kendala Paling Umum;Template Balasan (Siap Digunakan)\n"
        "1;SKPI;Balasan A\n",
        encoding="utf-8",
    )
    out = tmp_path / "store.json"

    def flaky(texts):
        return [None if "Pasal" in t else [1.0] for t in texts]

    stats = build_store(embed_fn=flaky, md_path=md, csv_path=csvp, out_path=out)
    data = json.loads(out.read_text(encoding="utf-8"))

    assert stats["errors"] == 1
    assert stats["total"] == 1
    assert [r["metadata"]["jenis"] for r in data] == ["template"]
```

- [ ] **Step 3: Jalankan test hingga gagal**

Run: `python -m pytest -q tests/test_chunker.py tests/test_build_store.py`
Expected: FAIL dengan `ModuleNotFoundError: No module named 'ingestion'`

- [ ] **Step 4: Tulis chunker**

`ingestion/__init__.py`: kosong.

`ingestion/chunker.py`:

```python
import csv
import io
import re

MAX_CHARS = 2400
OVERLAP_CHARS = 400


def _split_body(body):
    if len(body) <= MAX_CHARS:
        return [body]
    lines = body.splitlines(keepends=True)
    header = next((line.rstrip("\n") for line in lines if "|" in line), "")
    parts, current, length = [], [], 0
    for line in lines:
        if current and length + len(line) > MAX_CHARS:
            text = "".join(current)
            parts.append(text)
            overlap = text[-OVERLAP_CHARS:]
            current = [f"{header}\n{overlap}"] if header else [overlap]
            length = len(current[0])
        current.append(line)
        length += len(line)
    if current:
        parts.append("".join(current))
    return parts


def chunk_markdown(text, source="SK TAK 2025"):
    chunks, heading, buf = [], "", []

    def flush():
        body = "\n".join(buf).strip()
        for part in _split_body(body) if body else []:
            chunks.append({
                "id": f"sk-{len(chunks) + 1:04d}",
                "teks": f"{heading}\n{part}" if heading else part,
                "metadata": {
                    "jenis": "sk_tak",
                    "sumber": source,
                    "bagian": heading,
                    "tahun_berlaku": 2025,
                    "status_aktif": True,
                },
            })

    for line in text.splitlines():
        match = re.match(r"^(#{1,3})\s+(.*)$", line)
        if match:
            flush()
            buf = []
            heading = match.group(2).strip()
        else:
            buf.append(line)
    flush()
    return chunks


def chunk_templates(text):
    records, skipped = [], 0
    for row in csv.reader(io.StringIO(text), delimiter=";"):
        if len(row) < 3:
            continue
        kategori, template = row[1].strip(), row[2].strip()
        if kategori == "Kategori Kendala Paling Umum":
            continue
        if not kategori or not template:
            skipped += 1
            continue
        records.append({
            "id": f"tpl-{len(records) + 1:04d}",
            "teks": f"Kategori: {kategori}\nBalasan: {template}",
            "metadata": {
                "jenis": "template",
                "sumber": "TemplateAI.csv",
                "bagian": kategori,
                "tahun_berlaku": 2025,
                "status_aktif": True,
            },
        })
    return records, skipped
```

- [ ] **Step 5: Tulis build_store**

`ingestion/build_store.py`:

```python
import json
from pathlib import Path

from config.groq_client import make_client
from config.settings import EMBED_MODEL, SOURCE_CSV, SOURCE_MD, STORE_PATH
from ingestion.chunker import chunk_markdown, chunk_templates


def embed_with_groq(texts, client=None):
    client = client or make_client()
    vectors = []
    for start in range(0, len(texts), 16):
        batch = texts[start:start + 16]
        response = client.embeddings.create(model=EMBED_MODEL, input=batch)
        vectors.extend(item.embedding for item in response.data)
    return vectors


def build_store(embed_fn=None, md_path=None, csv_path=None, out_path=None):
    embed_fn = embed_fn or embed_with_groq
    md_path = Path(md_path or SOURCE_MD)
    csv_path = Path(csv_path or SOURCE_CSV)
    out_path = Path(out_path or STORE_PATH)

    chunks = chunk_markdown(md_path.read_text(encoding="utf-8"))
    templates, skipped = chunk_templates(csv_path.read_text(encoding="utf-8-sig"))
    records = chunks + templates

    errors = 0
    for record, vector in zip(records, embed_fn([r["teks"] for r in records])):
        if vector is None:
            errors += 1
            continue
        record["embedding"] = vector
    records = [r for r in records if "embedding" in r]

    out_path.parent.mkdir(parents=True, exist_ok=True)
    tmp = out_path.with_suffix(".tmp")
    tmp.write_text(json.dumps(records, ensure_ascii=False), encoding="utf-8")
    tmp.replace(out_path)
    return {
        "total": len(records),
        "sk_tak": len(chunks) - errors,
        "template": len(templates),
        "skipped": skipped,
        "errors": errors,
    }


if __name__ == "__main__":
    print(build_store())
```

- [ ] **Step 6: Jalankan test hingga lolos**

Run: `python -m pytest -q`
Expected: `8 passed`

- [ ] **Step 7: Jalankan ingestion nyata (manual, butuh kunci dan kuota)**

Run: `python -m ingestion.build_store`
Expected: cetakan berisi kunci `total`, `sk_tak`, `template`, `skipped`, `errors` dengan angka mengikuti isi dokumen yang sebenarnya (jangan mengandaikan jumlah tertentu). Periksa `data/store.json` ada dan bisa dibuka.

Acceptance: jumlah chunk per sumber tercetak, `skipped` tercetak, tidak ada `@` pada `store.json`, penulisan file bersifat atomic.

---

### T-05 - Retrieval cosine Top-5

**Files:**
- Create: `retriever/__init__.py`, `retriever/search.py`, `tests/test_search.py`

**Interfaces:**
- Consumes: `config.settings.STORE_PATH`, `MIN_SCORE`, `TOP_K`
- Produces: `cosine(a: list[float], b: list[float]) -> float`, `search(question: str, embed_fn, store=None, top_k=5, min_score=0.30) -> list[dict]` (record ditambah kunci `skor`).

- [ ] **Step 1: Tulis test yang gagal**

`tests/test_search.py`:

```python
from retriever.search import cosine, search


def test_cosine_identical_is_one():
    assert round(cosine([1, 0], [1, 0]), 3) == 1.0


def test_search_filters_low_score_and_inactive():
    store = [
        {"id": "a", "teks": "A", "embedding": [1, 0], "metadata": {"status_aktif": True}},
        {"id": "b", "teks": "B", "embedding": [0, 1], "metadata": {"status_aktif": True}},
        {"id": "c", "teks": "C", "embedding": [1, 0], "metadata": {"status_aktif": False}},
    ]
    result = search("q", lambda texts: [[1, 0] for _ in texts], store=store, min_score=0.5)
    assert [r["id"] for r in result] == ["a"]
    assert result[0]["skor"] == 1.0
```

- [ ] **Step 2: Jalankan test hingga gagal**

Run: `python -m pytest -q tests/test_search.py`
Expected: FAIL dengan `ModuleNotFoundError: No module named 'retriever'`

- [ ] **Step 3: Tulis implementasi minimal**

`retriever/__init__.py`: kosong.

`retriever/search.py`:

```python
import json
import math
from pathlib import Path

from config.settings import MIN_SCORE, STORE_PATH, TOP_K


def cosine(a, b):
    dot = sum(x * y for x, y in zip(a, b))
    norm_a = math.sqrt(sum(x * x for x in a))
    norm_b = math.sqrt(sum(y * y for y in b))
    return dot / (norm_a * norm_b) if norm_a and norm_b else 0.0


def load_store(path=None):
    return json.loads(Path(path or STORE_PATH).read_text(encoding="utf-8"))


def search(question, embed_fn, store=None, top_k=TOP_K, min_score=MIN_SCORE):
    records = store if store is not None else load_store()
    query_vector = embed_fn([question])[0]
    scored = []
    for record in records:
        if not record["metadata"].get("status_aktif", True):
            continue
        score = cosine(query_vector, record["embedding"])
        if score >= min_score:
            scored.append({**record, "skor": round(score, 4)})
    scored.sort(key=lambda r: r["skor"], reverse=True)
    return scored[:top_k]
```

- [ ] **Step 4: Jalankan test hingga lolos**

Run: `python -m pytest -q`
Expected: `10 passed`

Acceptance: Top-5 terurut, potongan di bawah ambang dan berstatus nonaktif tidak muncul, `skor` tercantum.

---

### T-06 - Prompt, fallback, dan demo terminal

**Files:**
- Create: `prompter/__init__.py`, `prompter/build_prompt.py`, `chat/__init__.py`, `chat/answer.py`, `chat/run.py`, `tests/test_prompt.py`, `tests/test_answer.py`

**Interfaces:**
- Consumes: `T-05.search`, `config.groq_client.make_client`, `config.settings`
- Produces: `build_prompt(question, chunks) -> list[dict] | None`, `FALLBACK_TEXT`, `answer(question, embed_fn, client=None, store=None) -> Iterator[str]`.

- [ ] **Step 1: Tulis test prompt yang gagal**

`tests/test_prompt.py`:

```python
from prompter.build_prompt import FALLBACK_TEXT, build_prompt


def test_no_context_uses_fallback():
    assert build_prompt("q", []) is None
    assert "SITU TAK" in FALLBACK_TEXT


def test_prompt_contains_citation_instruction():
    chunks = [{
        "teks": "Nilai minimal 60.",
        "metadata": {"sumber": "SK TAK 2025", "bagian": "Pasal 4"},
    }]
    messages = build_prompt("q", chunks)
    assert "sitasi" in messages[0]["content"].lower()
    assert "Sumber: SK TAK 2025" in messages[1]["content"]
```

- [ ] **Step 2: Tulis test answer yang gagal**

`tests/test_answer.py`:

```python
from types import SimpleNamespace

from chat.answer import answer


class FakeCompletions:
    def create(self, **kwargs):
        return [
            SimpleNamespace(choices=[SimpleNamespace(delta=SimpleNamespace(content=t))])
            for t in ["Nilai", " minimal", " 60."]
        ]


def test_answer_streams_tokens():
    client = SimpleNamespace(chat=SimpleNamespace(completions=FakeCompletions()))
    store = [{
        "id": "a",
        "teks": "Nilai minimal TAK 60.",
        "embedding": [1, 0],
        "metadata": {"sumber": "SK TAK 2025", "bagian": "Pasal 4", "status_aktif": True},
    }]
    out = "".join(answer("q", lambda texts: [[1, 0] for _ in texts], client=client, store=store))
    assert out == "Nilai minimal 60."


def test_answer_fallback_without_llm_call():
    class NoCall:
        def create(self, **kwargs):
            raise AssertionError("LLM tidak boleh dipanggil saat tanpa konteks")

    client = SimpleNamespace(chat=SimpleNamespace(completions=NoCall()))
    out = "".join(answer("q", lambda texts: [[0, 1] for _ in texts], client=client, store=[]))
    assert "belum tersedia" in out
```

- [ ] **Step 3: Jalankan test hingga gagal**

Run: `python -m pytest -q tests/test_prompt.py tests/test_answer.py`
Expected: FAIL dengan `ModuleNotFoundError: No module named 'prompter'`

- [ ] **Step 4: Tulis prompter**

`prompter/__init__.py`: kosong.

`prompter/build_prompt.py`:

```python
FALLBACK_TEXT = (
    "Maaf, informasi ini belum tersedia di dokumen layanan TAK. "
    "Silakan cek SITU TAK (https://situ-kem.telkomuniversity.ac.id/tak) "
    "atau hubungi Student Affairs / Service Desk PuTI."
)

SYSTEM_PROMPT = (
    "Kamu asisten layanan TAK Telkom University. Jawab HANYA berdasarkan KONTEKS. "
    "Untuk aturan, nilai, pasal, atau syarat, utamakan sumber SK TAK 2025; "
    "untuk cara, verifikasi, error, atau prosedur internal, utamakan TemplateAI.csv. "
    "Sertakan sitasi [Sumber: ..., bagian: ...] di akhir jawaban. "
    "Jangan mengarang informasi di luar konteks."
)


def build_prompt(question, chunks):
    if not chunks:
        return None
    context = "\n\n".join(
        f"[{i + 1}] {c['teks']}\n"
        f"(Sumber: {c['metadata']['sumber']}, {c['metadata']['bagian']})"
        for i, c in enumerate(chunks)
    )
    return [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": f"KONTEKS:\n{context}\n\nPERTANYAAN:\n{question}"},
    ]
```

- [ ] **Step 5: Tulis answer dan run**

`chat/__init__.py`: kosong.

`chat/answer.py`:

```python
from config.groq_client import make_client
from config.settings import LLM_MODEL, TEMPERATURE
from prompter.build_prompt import FALLBACK_TEXT, build_prompt
from retriever.search import search


def answer(question, embed_fn, client=None, store=None):
    chunks = search(question, embed_fn, store=store)
    messages = build_prompt(question, chunks)
    if messages is None:
        yield FALLBACK_TEXT
        return
    client = client or make_client()
    stream = client.chat.completions.create(
        model=LLM_MODEL,
        messages=messages,
        temperature=TEMPERATURE,
        stream=True,
    )
    for part in stream:
        token = part.choices[0].delta.content or ""
        if token:
            yield token
```

`chat/run.py`:

```python
from chat.answer import answer
from config.groq_client import make_client
from config.settings import EMBED_MODEL


def run():
    client = make_client()

    def embed_fn(texts):
        response = client.embeddings.create(model=EMBED_MODEL, input=texts)
        return [item.embedding for item in response.data]

    print("Chatbot TAK. Ketik 'keluar' untuk berhenti.")
    while True:
        question = input("\nKamu: ").strip()
        if not question:
            continue
        if question.lower() in {"keluar", "exit", "quit"}:
            break
        if len(question) > 1000:
            print("Pertanyaan terlalu panjang.")
            continue
        print("Bot: ", end="", flush=True)
        for token in answer(question, embed_fn, client=client):
            print(token, end="", flush=True)
        print()


if __name__ == "__main__":
    run()
```

- [ ] **Step 6: Jalankan test hingga lolos**

Run: `python -m pytest -q`
Expected: `14 passed`

- [ ] **Step 7: Uji enam pertanyaan (manual)**

Prasyarat: `data/store.json` sudah dibuat pada T-04 dan `GROQ_API_KEY` terset.

Run: `python -m chat.run`, lalu kirim enam pertanyaan dari bagian 5 spec.

Expected: pertanyaan 1–5 terjawab dengan sitasi yang sesuai; pertanyaan 6 menghasilkan teks fallback tanpa jawaban karangan; token tampil bertahap.

Acceptance: seluruh checklist penerimaan pada spec terpenuhi, termasuk tidak ada PII di `store.json` dan tidak ada kunci API yang tercetak.

### Handoff
- Current task: None
- Next ready task: T-02 (ACC Rev 2 sudah diberikan 2026-11-27); T-03 menunggu konfirmasi daftar model Groq; T-04–T-06 berurutan setelahnya
- Latest verification: `python -c` menunjukkan `pytest`, `groq`, `ruff`, `openai` belum terpasang; `pip` 26.1.1 dan Python 3.14.5 tersedia. Belum ada test yang dijalankan.
- Remaining blockers: OD-02 (kanal), OD-03 (finalisasi template), dan OD-01 untuk produksi. Runtime pengembangan sudah ditetapkan (Opsi 2) dan tidak memblokir T-02/T-03.

## Amendments
Rev 2 (Draft, belum ACC): mengikuti spec `docs/superpowers/specs/2026-11-25-tak-rag-subproyek-1-design.md` yang disetujui pengguna untuk dilanjutkan ke implementation plan. Delta: Opsi 2 (Groq untuk embedding dan generasi) menggantikan embedding lokal untuk sub-proyek 1; `data/store.json` menggantikan pgvector pada tahap ini; kriteria selesai berupa demo terminal enam pertanyaan; T-02/T-03 diubah dari docker-compose/runtime lokal menjadi scaffold Python + konektivitas Groq; langkah terperinci ditambahkan di Part C. Dampak: T-03 tidak lagi menunggu OD-01 untuk tahap ini, T-15 masih menunggu keputusan deployment produksi. Status approval: **Approved 2026-11-27** dengan kutipan persis "acc push ke ssues gantikan yang sebelumnya". A/B Rev 2 menjadi baseline yang disetujui; perubahan material berikutnya memerlukan revisi baru dan ACC.
