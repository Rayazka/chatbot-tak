# Project Requirements Document (PRD)

## 1. Project Overview

* **Project Name:** Sistem Layanan Informasi Kemahasiswaan Berbasis AI (*RAG Chatbot* Kemahasiswaan)

* **Project ID:** PRJ-KMS-RAG-2026-001

* **Date:** 25 November 2026

* **Version:** 1.1

* **Prepared By:** Tim Pengembang AI & Sistem Informasi Kemahasiswaan

* **Approved By:** Kepala Biro Administrasi Kemahasiswaan & Kepala Unit Teknologi Informasi

### 1.1 Purpose of the Project

Proyek ini bertujuan untuk membangun dan meluncurkan asisten virtual (*chatbot*) berbasis arsitektur *Retrieval-Augmented Generation* (RAG) menggunakan model bahasa `openai/gpt-oss-20b`. Tujuan utamanya adalah menyediakan layanan informasi dan bantuan pelanggan (*customer service*) kemahasiswaan secara otomatis, akurat, dan tersedia 24/7 berdasarkan dokumen resmi institusi.

### 1.2 Project Background

Unit Layanan Kemahasiswaan sering menghadapi lonjakan pertanyaan berulang dari mahasiswa terkait prosedur cuti akademik, pengajuan beasiswa, legalisir dokumen, surat keterangan aktif kuliah, hingga tata tertib kampus, terutama pada periode awal semester. Penanganan secara manual melalui loket fisik atau pesan admin menyebabkan waktu tunggu yang lama dan beban kerja staf administratif yang tinggi. Dengan memanfaatkan teknologi RAG, sistem dapat menjawab pertanyaan mahasiswa secara instan berdasarkan basis pengetahuan (*knowledge base*) resmi tanpa risiko AI mengarang jawaban (*hallucination*).

### 1.3 Scope of the Project

* **In-Scope:**

  * Pembuatan *pipeline* pengolahan dan pembersihan dokumen resmi kemahasiswaan (PDF, DOCX, Markdown) menjadi vektor informasi.

  * Integrasi model LLM `openai/gpt-oss-20b` dengan *Vector Database* dan mekanisme *Hybrid Search*.

  * Pengembangan *Backend API* dengan fitur *streaming response* dan manajemen konteks percakapan (*chat memory*).

  * Pengembangan antarmuka *Web Chat Widget* yang dapat disematkan pada portal kemahasiswaan serta *dashboard* admin sederhana untuk pembaruan dokumen SOP.

* **Out-of-Scope:**

  * Pemrosesan transaksi administratif secara langsung di dalam sistem akademik (misal: persetujuan otomatis pengajuan cuti atau pencairan dana beasiswa).

  * Akses terhadap data pribadi/sensitif mahasiswa secara spesifik (seperti nilai transkrip individu atau status tagihan SPP personal) pada fase rilis pertama ini.

  * Pengembangan aplikasi *mobile* *native* (Android/iOS) tersendiri.

## 2. Stakeholders

* **Primary Stakeholders:**

  * **Kepala Biro Kemahasiswaan (Project Sponsor):** Pemilik bisnis proses, penyedia dokumen SOP resmi, dan pemberi persetujuan akhir.

  * **Product Manager / Project Lead:** Penanggung jawab perencanaan, pengelolaan *timeline*, dan koordinasi lintas tim.

  * **AI & Backend Engineer:** Bertanggung jawab atas arsitektur RAG, *deployment* model `openai/gpt-oss-20b`, *ingestion pipeline*, dan API.

  * **Frontend Developer:** Bertanggung jawab atas antarmuka *chat* mahasiswa dan *dashboard* manajemen dokumen admin.

  * **Staf Admin Kemahasiswaan:** Kurator dokumen kebijakan dan validator akurasi jawaban *chatbot*.

* **Secondary Stakeholders:**

  * **Mahasiswa Aktif & Calon Mahasiswa:** Pengguna akhir (*end-user*) layanan *chatbot*.

  * **Unit Infrastruktur IT Kampus:** Penyedia server GPU/CPU dan keamanan jaringan.

* **Stakeholder Communication Plan:**

  * **Rapat Mingguan (*Weekly Sync*):** Setiap hari Senin untuk memantau progres pengembangan dan kendala teknis.

  * **Laporan Status (*Status Report*):** Dikirimkan setiap akhir pekan melalui email kepada *Project Sponsor*.

  * **Sesi UAT (*User Acceptance Testing*):** Dilakukan bersama perwakilan staf admin kemahasiswaan sebelum peluncuran.

## 3. Objectives and Goals

* **Key Project Objectives:**

  1. Mengurangi beban pertanyaan berulang (*repetitive inquiries*) yang masuk ke staf admin kemahasiswaan sebesar minimal $60\%$ dalam 3 bulan pertama setelah peluncuran.

  2. Menyediakan waktu respons pertama (*first response time*) di bawah $3$ detik untuk setiap pertanyaan mahasiswa.

  3. Memastikan seluruh jawaban *chatbot* bersumber dari dokumen SOP dan SK resmi yang berlaku.

* **Success Criteria:**

  * Tingkat akurasi jawaban (*Faithfulness* & *Context Recall*) mencapai $\ge 90\%$ berdasarkan pengujian terhadap 50 set pertanyaan standar kemahasiswaan.

  * Tingkat halusinasi (memberikan informasi di luar dokumen) di bawah $2\%$.

  * Tingkat kepuasan pengguna (*thumbs-up rating* pada fitur *feedback*) mencapai minimal $85\%$.

  * Sistem dirilis sesuai jadwal yang ditetapkan tanpa melebihi anggaran infrastruktur.

## 4. Functional Requirements

### 4.1 Modul Mahasiswa (*User Interface*)

* **FR-01 (Tanya Jawab Interaktif):** Mahasiswa dapat mengetikkan pertanyaan dalam bahasa Indonesia sehari-hari maupun formal dan menerima jawaban secara *real-time* (*streaming text*).

* **FR-02 (Sitasi Sumber Dokumen):** Sistem wajib menampilkan referensi nama dokumen, pasal, atau halaman sumber di bagian bawah setiap jawaban yang diberikan.

* **FR-03 (Konteks Percakapan / *Follow-up*):** Sistem mampu memahami pertanyaan lanjutan dalam satu sesi *chat* yang sama (misal: menanyakan rincian berkas setelah menanyakan jenis beasiswa).

* **FR-04 (Pertanyaan Cepat / *Quick Prompts*):** Tampilan awal menyediakan tombol topik populer (seperti *"Syarat Cuti Kuliah"*, *"Info Beasiswa"*, *"Surat Aktif Kuliah"*).

* **FR-05 (Eskalasi & *Fallback*):** Jika jawaban tidak ditemukan di dalam dokumen, sistem wajib menolak menjawab secara spekulatif dan menampilkan kontak resmi/tautan loket layanan admin.

* **FR-06 (Umpan Balik Jawaban):** Mahasiswa dapat memberikan penilaian jempol atas (*helpful*) atau jempol bawah (*not helpful*) pada setiap balasan bot.

### 4.2 Modul Admin (*Knowledge Management*)

* **FR-07 (Autentikasi Admin):** Staf kemahasiswaan harus melakukan *login* menggunakan akun terotorisasi untuk mengakses panel pengelolaan dokumen.

* **FR-08 (Unggah & Pemrosesan Dokumen):** Admin dapat mengunggah dokumen baru (PDF/DOCX/MD), menambahkan *metadata* (kategori dan tahun ajaran), dan memicu proses *chunking* serta *embedding* otomatis.

* **FR-09 (Pembaruan & Penghapusan Dokumen):** Admin dapat menonaktifkan atau menghapus dokumen kebijakan lama agar tidak lagi dijadikan referensi oleh LLM.

* **FR-10 (Log & Analitik Pertanyaan):** Admin dapat melihat daftar pertanyaan yang paling sering diajukan serta pertanyaan yang gagal dijawab oleh bot sebagai bahan evaluasi SOP.

## 5. Non-Functional Requirements

* **Performance Requirements:**

  * *Time to First Token* (TTFT) maksimal $2.5$ detik dan kecepatan generasi minimal $20$ token/detik menggunakan `openai/gpt-oss-20b`.

  * Mampu menangani minimal $50$ pengguna aktif secara bersamaan (*concurrent users*) tanpa penurunan performa signifikan.

* **Security Requirements:**

  * Seluruh komunikasi data antara *client* dan *server* wajib dienkripsi menggunakan protokol HTTPS (TLS 1.3).

  * Perlindungan terhadap serangan *Prompt Injection* melalui *system prompt guardrails* dan validasi input.

  * Kontrol akses berbasis peran (*Role-Based Access Control*) pada *dashboard* admin.

* **Usability Requirements:**

  * Antarmuka *chat* bersifat responsif (*mobile-friendly*) sehingga nyaman diakses melalui peramban ponsel pintar maupun desktop.

  * Desain mengikuti identitas visual resmi institusi dengan kontras warna yang mudah dibaca.

* **Availability Requirements:**

  * Target ketersediaan sistem (*uptime*) sebesar $99.5\%$, beroperasi 24 jam sehari, 7 hari seminggu.

  * Jadwal pemeliharaan (*maintenance window*) hanya dilakukan di luar jam sibuk (pukul 00.00 – 04.00 WIB).

* **Compliance Requirements:**

  * Kepatuhan terhadap Undang-Undang Perlindungan Data Pribadi (UU PDP) dengan tidak menyimpan data sensitif mahasiswa tanpa persetujuan dan melakukan anonimisasi pada log percakapan.

## 6. Technical Plan & System Architecture

Bagian ini mendefinisikan tumpukan teknologi (*tech stack*), arsitektur alur kerja data, dan spesifikasi infrastruktur yang digunakan untuk membangun sistem.

### 6.1 Tech Stack & Definisi Komponen

| Komponen | Definisi & Fungsi | Teknologi / Library | Alasan Pemilihan | 
| ----- | ----- | ----- | ----- | 
| **LLM (*Generator*)** | Otak utama AI yang bertugas memahami pertanyaan pengguna dan merangkai jawaban bahasa alami secara akurat berdasarkan potongan konteks yang diberikan. | `openai/gpt-oss-20b` (via **vLLM** atau **Ollama**) | Mendukung penalaran (*reasoning*) yang kuat, kapasitas *context window* besar (128k token), serta kompatibel dengan standar OpenAI API. | 
| **Model *Embedding*** | Model AI yang mengonversi potongan teks dokumen dan pertanyaan pengguna menjadi representasi angka vektor agar sistem dapat menghitung kemiripan makna (*semantic similarity*). | `BAAI/bge-m3` atau `nomic-embed-text` | Memiliki akurasi tinggi dalam menangkap konteks semantik bahasa Indonesia dan mendukung pencarian multibahasa (Indonesia-Inggris). | 
| ***Reranker*** **Model** | Komponen penyaring tahap kedua yang menilai dan mengurutkan ulang dokumen hasil pencarian awal agar hanya konteks yang paling relevan yang dikirim ke LLM. | `BAAI/bge-reranker-v2-m3` | Meminimalkan risiko salah konteks (*noise*) pada dokumen SOP yang mirip sehingga akurasi jawaban meningkat tajam dan mencegah halusinasi. | 
| ***Vector Database*** | Basis data khusus yang menyimpan vektor numerik beserta teks asli dan *metadata* dokumen untuk pencarian informasi berkecepatan tinggi. | **PostgreSQL (`pgvector`)** atau **Qdrant** | `pgvector` sangat efisien karena mampu menyatukan penyimpanan vektor, riwayat *chat*, dan data manajemen admin dalam satu database relasional. | 
| ***Backend*** **& Orkestrasi** | Tulang punggung sistem yang mengatur alur kerja RAG (*retrieval*, *reranking*, *prompting*), manajemen sesi, serta menyediakan jalur komunikasi API ke antarmuka pengguna. | **Python (FastAPI)** + **LlamaIndex** / **LangChain** | FastAPI mendukung pemrosesan *asynchronous* dan *streaming* respons secara *real-time*; LlamaIndex unggul dalam mengindeks dokumen administratif terstruktur. | 
| ***Document Parser*** | Alat ekstraksi yang membaca dokumen mentah (PDF, DOCX) dan mengubah isinya menjadi format teks terstruktur (Markdown) sebelum diproses lebih lanjut. | **Docling** atau **PyMuPDF4LLM** | Mampu mempertahankan struktur tabel, poin persyaratan, dan hierarki bab dari buku panduan akademik atau SK Rektor agar tidak berantakan. | 
| ***Cache & Memory*** | Sistem penyimpanan sementara berkecepatan tinggi di memori untuk menyimpan riwayat percakapan aktif dan hasil jawaban dari pertanyaan yang sering berulang. | **Redis** | Memungkinkan chatbot memahami konteks pertanyaan lanjutan (*multi-turn chat*) dan mempercepat waktu respons untuk FAQ populer tanpa membebani GPU. | 
| ***Frontend*** **(Antarmuka)** | Tampilan antarmuka visual tempat mahasiswa mengetik pertanyaan, melihat jawaban secara bertahap (*streaming*), serta memberikan umpan balik kualitas jawaban. | **Next.js (React)** + **Tailwind CSS** | Ringan, responsif untuk disematkan sebagai *widget* di portal web kampus, dan mendukung integrasi *Server-Sent Events* (SSE) untuk efek mengetik *real-time*. | 

### 6.2 Arsitektur Alur Data (*Data & Inference Pipeline*)

Sistem dibagi menjadi dua alur pemrosesan utama:

1. **Alur Pemrosesan Dokumen (*Offline Data Ingestion Pipeline*):**

   * **Ekstraksi:** Dokumen SOP/SK yang diunggah admin diekstrak oleh *Document Parser* (`Docling`) menjadi teks Markdown bersih dengan mempertahankan struktur tabel.

   * **Pemotongan (*Chunking*):** Teks dipecah menggunakan *Markdown Header / Recursive Splitter* berukuran $500 - 800$ token dengan *overlap* $100$ token, dilengkapi *metadata* (`nama_dokumen`, `kategori`, `tahun_berlaku`, `status_aktif`).

   * **Vektorisasi:** Model *Embedding* (`BAAI/bge-m3`) mengubah setiap *chunk* menjadi vektor dan menyimpannya ke dalam **PostgreSQL (`pgvector`)**.

2. **Alur Tanya Jawab (*Online Retrieval & Generation Pipeline*):**

   * **Pengecekan Cache & Memori:** *Backend* memeriksa riwayat percakapan di **Redis** untuk melakukan *Query Rewriting* (jika pertanyaan bersifat lanjutan) serta mengecek *Semantic Cache*.

   * **Pencarian Hibrida (*Hybrid Search*):** Sistem mencari kandidat dokumen teratas (*Top-20*) menggunakan kombinasi pencarian vektor (*dense retrieval*) dan kata kunci eksak (*BM25/sparse retrieval*) yang difilter berdasarkan *metadata* dokumen aktif.

   * **Penyaringan Ulang (*Reranking*):** Model `BAAI/bge-reranker-v2-m3` menilai ulang 20 kandidat dan memilih *Top-5* potongan konteks paling relevan.

   * **Generasi Jawaban (*Streaming*):** Konteks disuntikkan ke dalam *System Prompt* yang ketat dan dikirim ke `openai/gpt-oss-20b` (*temperature* $= 0.1$) untuk menghasilkan jawaban secara *streaming* (SSE) beserta sitasi dokumen.

### 6.3 Spesifikasi Infrastruktur & Deployment

Seluruh layanan dikemas dalam kontainer menggunakan **Docker & Docker Compose** di balik *reverse proxy* (**Nginx / Caddy** dengan SSL otomatis). Sistem dirancang mendukung dua skema *deployment* utama:

* **Skema Utama: *On-Premise / Self-Hosted GPU Server* (Rekomendasi Produksi)**

  * **Server Komputasi AI (GPU Node):** GPU dengan **VRAM minimal 24 GB** (misal: NVIDIA RTX 3090 / 4090 / L4 / A10) untuk menjalankan `openai/gpt-oss-20b` (kuantisasi FP8/AWQ via **vLLM**), model *embedding*, dan *reranker* secara lokal.

  * **Spesifikasi Sistem:** CPU 8–16 Core, RAM Sistem 32 GB, Penyimpanan NVMe SSD 256 GB.

  * **Keunggulan:** Privasi data internal terjaga 100% di dalam jaringan kampus dan bebas biaya variabel per token.

* **Skema Alternatif: *Hybrid Cloud Architecture***

  * **Server Aplikasi (VPS):** 4 vCPU, 8 GB RAM, 80 GB SSD untuk menjalankan kontainer FastAPI, Next.js, PostgreSQL (`pgvector`), dan Redis.

  * **LLM Inference Endpoint:** Menggunakan layanan *Cloud Inference API* kompatibel OpenAI (seperti Together AI, Fireworks, atau OpenRouter) untuk mengakses `openai/gpt-oss-20b` apabila perangkat keras GPU lokal belum tersedia.

## 7. Assumptions

* Seluruh dokumen SOP, buku panduan, dan SK kemahasiswaan yang diberikan oleh pihak biro merupakan versi terbaru dan sah secara hukum institusi.

* Infrastruktur *server* lokal/cloud dengan kapasitas GPU/VRAM minimal 16 GB – 24 GB tersedia sebelum fase pengembangan *backend* dimulai.

* Staf admin kemahasiswaan bersedia meluangkan waktu selama fase pengujian (UAT) untuk memvalidasi kebenaran jawaban *chatbot*.

## 8. Constraints

* **Keterbatasan Komputasi:** Pengoperasian model `openai/gpt-oss-20b` secara mandiri membatasi jumlah pemrosesan paralel sesuai kapasitas VRAM server yang tersedia.

* **Kualitas Dokumen Sumber:** Dokumen PDF hasil pemindaian (*scanned image*) yang buram memerlukan proses OCR tambahan dan berpotensi menurunkan akurasi ekstraksi teks.

* **Tenggat Waktu:** Sistem harus sudah siap digunakan sebelum dimulainya masa registrasi ulang semester baru.

## 9. Dependencies

* **Tim Biro Kemahasiswaan:** Ketergantungan pada kelengkapan dan penyerahan dokumen SOP pada minggu pertama proyek.

* **Tim Infrastruktur IT:** Ketergantungan pada penyediaan akses *server*, domain/subdomain resmi, dan sertifikat SSL.

* **Ekosistem Open-Source:** Ketergantungan pada stabilitas pustaka *inference* (seperti vLLM atau Ollama), *framework* orkestrasi (LlamaIndex/LangChain), serta *Vector Database* (Qdrant atau PostgreSQL `pgvector`).

## 10. Risks

| ID Risiko | Identifikasi Risiko | Tingkat Dampak | Rencana Mitigasi (*Risk Mitigation Plan*) | 
| ----- | ----- | ----- | ----- | 
| **RSK-01** | **Halusinasi Model:** LLM memberikan jawaban yang salah atau mengarang syarat administratif. | Tinggi | Menerapkan *system prompt* yang ketat, menggunakan *Reranker*, mewajibkan sitasi dokumen, dan mengatur *temperature* model mendekati $0.1$. | 
| **RSK-02** | **Informasi Kedaluwarsa:** Bot menjawab menggunakan aturan tahun ajaran lama. | Tinggi | Mewajibkan pengisian *metadata* tahun berlaku pada setiap dokumen dan menerapkan filter pencarian berdasarkan dokumen aktif terbaru. | 
| **RSK-03** | **Lonjakan Trafik (*Bottleneck*):** Server melambat saat masa pengumuman beasiswa atau registrasi. | Sedang | Mengimplementasikan *Semantic Caching* (Redis) untuk menyimpan jawaban dari pertanyaan yang identik/sering ditanyakan agar tidak membebani GPU. | 
| **RSK-04** | **Format Tabel PDF Rusak:** Syarat dalam bentuk tabel tidak terbaca utuh saat *chunking*. | Sedang | Menggunakan *document parser* khusus tata letak (seperti `Docling`) untuk mengonversi tabel PDF menjadi format Markdown terstruktur sebelum di-*embed*. | 

## 11. Deliverables

1. **Dokumen Spesifikasi & Arsitektur Sistem** (Minggu ke-1)

2. **Basis Data Vektor & *Data Ingestion Pipeline*** berisi seluruh SOP kemahasiswaan terstruktur (Minggu ke-3)

3. **Prototipe *Backend RAG API* (`openai/gpt-oss-20b`)** dengan fitur *hybrid search* dan *streaming* (Minggu ke-5)

4. **Antarmuka *Web Chat Widget* & *Admin Dashboard*** (Minggu ke-6)

5. **Laporan Hasil Pengujian Akurasi (*RAG Evaluation*) & UAT** (Minggu ke-7)

6. **Rilis Produksi (*Production Deployment*) & Panduan Penggunaan Admin** (Minggu ke-8)

## 12. Timeline

Total estimasi durasi pengerjaan adalah **8 Minggu**:

* **Minggu 1 – 2 (Fase Persiapan & Data):** Pengumpulan SOP kemahasiswaan, pembersihan data PDF ke Markdown, dan *setup* lingkungan *server* serta model `openai/gpt-oss-20b`.

* **Minggu 3 – 4 (Fase Pengembangan Core RAG):** Pembuatan *ingestion pipeline*, konfigurasi *Vector Database*, implementasi *hybrid search*, dan *prompt engineering*.

* **Minggu 5 – 6 (Fase Pengembangan Aplikasi & UI):** Pembuatan *endpoint* FastAPI, integrasi *memory/cache*, serta pengembangan UI *chat* mahasiswa dan *dashboard* admin.

* **Minggu 7 (Fase Pengujian & Evaluasi):** Uji akurasi dengan dataset pertanyaan kemahasiswaan, uji keamanan (*red teaming*), dan *User Acceptance Testing* (UAT) bersama admin.

* **Minggu 8 (Fase Peluncuran & Finishing):** *Deployment* menggunakan Docker, optimasi performa, pelatihan staf admin, dan peluncuran resmi (*Go-Live*).

## 13. Budget

* **Estimated Budget:** Rp 35.000.000 *(estimasi untuk deployment on-premise/cloud VPS mandiri selama tahap pengembangan hingga rilis)*

* **Cost Breakdown:**

  * **Infrastruktur Server GPU & Cloud Hosting (3 Bulan):** Rp 15.000.000 (Sewa GPU Cloud untuk *inference* `openai/gpt-oss-20b` + VPS untuk *Backend/Database* atau biaya listrik/perawatan server lokal).

  * **Pengembangan & Rekayasa Sistem (Labor):** Rp 16.000.000 (Alokasi tim *AI/Backend Engineer* dan *Frontend Developer* selama 2 bulan).

  * **Pengujian, Dokumentasi & Cadangan Operasional:** Rp 4.000.000 (Biaya *testing*, domain/SSL, dan dana tak terduga).

* **Budget Constraints:** Proyek memanfaatkan model *open-source* (`openai/gpt-oss-20b` dan model *embedding* lokal) untuk menghindari biaya langganan API berbayar per token (*pay-per-token*) yang berpotensi membengkak saat jumlah mahasiswa yang mengakses meningkat.

## 14. Approval & Sign-off

Dokumen ini telah ditinjau dan disetujui sebagai acuan resmi pelaksanaan proyek pengembangan *RAG Chatbot* Layanan Kemahasiswaan:

| Peran | Nama | Tanda Tangan | Tanggal | 
| ----- | ----- | ----- | ----- | 
| **Project Sponsor** (Kepala Biro Kemahasiswaan) | \________________________\_ | \________________________\_ | \_**\_ / \_**\_ / 2026 | 
| **Head of IT Infrastructure** | \________________________\_ | \________________________\_ | \_**\_ / \_**\_ / 2026 | 
| **Project Manager / Lead Developer** | \________________________\_ | \________________________\_ | \_**\_ / \_**\_ / 2026 | 

* **Approval Date:** \[Diisi saat dokumen ditandatangani\]

## Appendices

### Glossary of Terms

* **RAG (*Retrieval-Augmented Generation*):** Teknik arsitektur AI yang mengambil informasi relevan dari basis data eksternal sebelum menghasilkan jawaban menggunakan LLM.

* **LLM (*Large Language Model*):** Model kecerdasan buatan pemroses bahasa alami (dalam proyek ini menggunakan `openai/gpt-oss-20b`).

* **Embedding:** Proses mengubah potongan teks menjadi representasi angka (*vector*) agar komputer dapat mengukur kemiripan makna antar teks.

* **Reranker:** Model penyaring tahap kedua yang mengurutkan ulang dokumen berdasarkan relevansi spesifik terhadap pertanyaan sebelum diberikan kepada LLM.

* **Chunking:** Proses memecah dokumen panjang menjadi bagian-bagian teks yang lebih kecil agar efektif saat dicari dan diproses oleh LLM.

* **Hallucination:** Kondisi ketika model AI menghasilkan informasi yang terdengar meyakinkan namun salah atau tidak memiliki dasar pada dokumen sumber.

* **Hybrid Search:** Metode pencarian yang menggabungkan pencarian berbasis makna (*semantic/vector search*) dan pencarian kata kunci eksak (*keyword/BM25 search*).

### References

1. Dokumen Standar Operasional Prosedur (SOP) Layanan Administrasi Kemahasiswaan.

2. Buku Pedoman Akademik dan Peraturan Kemahasiswaan Institusi Tahun Akademik Berjalan.

3. Dokumentasi Teknis Model `openai/gpt-oss-20b` dan Arsitektur *Retrieval-Augmented Generation*.