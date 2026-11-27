# AGENTS.md — RAG Chatbot Layanan TAK

File ini adalah **aturan kerja proyek** untuk manusia maupun AI agent/harness (Orca, Codex, Claude, opencode, dll.) yang menyentuh repo ini. Baca file ini **sebelum** mengubah apa pun.

## 1. Konteks proyek

- **Tujuan:** chatbot RAG untuk layanan Transkrip Aktivitas Kemahasiswaan (TAK) Kemahasiswaan Telkom University.
- **Sumber pengetahuan:** `SK_TAK_2025.md` (aturan resmi) dan `TemplateAI.csv` (balasan operasional). Log helpdesk **tidak** di-ingest sebelum dianonimisasi.
- **Sub-proyek 1 (sedang dikerjakan):** ingestion + core RAG chat berupa demo terminal. Selesai bila enam pertanyaan uji terjawab dengan sitasi dan fallback bekerja.
- **Keputusan teknis yang disetujui (Rev 2, ACC 2026-11-27):** Opsi 2 — embedding `nomic-embed-text-v1.5` dan generasi `openai/gpt-oss-20b` keduanya via Groq free tier; vektor disimpan di `data/store.json`; **tanpa** model lokal, database, server, atau UI pada sub-proyek ini.
- **Kutipan ACC:** "acc push ke ssues gantikan yang sebelumnya".
- **Dokumen otoritatif:**
  - `AGENTS.md` — aturan kerja (file ini).
  - `.agents/1-PRD-UTUH.md` Rev 2 — baseline A/B/C yang disetujui; Part C berisi langkah implementasi T-02–T-06.
  - `docs/superpowers/specs/2026-11-25-tak-rag-subproyek-1-design.md` — spec desain sub-proyek 1.
  - GitHub issue **#17** — master todo list; 24 issue lain adalah sub-issuenya.
- **Peran:** Owner A = backend/RAG/infra; Owner B = frontend/QA/dokumentasi. Keduanya melakukan review silang.

## 2. Aturan utama

1. **Satu issue = satu branch.** Branch dibuat dari `main` terbaru dengan nama `issue/<nomor>-<slug>`, misalnya `issue/2-scaffold-python-groq`. Satu branch hanya menyelesaikan satu issue. Dilarang menggabungkan pekerjaan beberapa issue dalam satu branch.
2. **Tidak ada commit langsung ke `main`.** Semua perubahan lewat branch + Pull Request.
3. **PR wajib direview** oleh orang lain (A review B, B review A). Agent tidak boleh approve PR-nya sendiri.
4. **Sebelum bekerja:** baca issue yang dituju, AGENTS.md ini, bagian plan/spec yang relevan, lalu cek `git status --short --branch`.
5. **TDD:** tulis test yang gagal → jalankan sampai gagal → tulis implementasi minimal → jalankan sampai lolos.
6. **Verifikasi sebelum klaim selesai:** jalankan perintah nyata dan catat hasilnya. Jangan mengatakan "lolos" tanpa output.
7. **Status task** hanya `Todo`, `In Progress`, `Blocked`, atau `Done`. `Done` mensyaratkan acceptance criteria terpenuhi **dan** bukti tercatat di issue serta Part C.
8. **Commit/push hanya bila diperintahkan eksplisit.** Agent tidak boleh commit, push, merge, atau menutup issue atas inisiatif sendiri.
9. **Rahasia & data:** `GROQ_API_KEY` hanya lewat environment variable, tidak pernah dicetak atau disimpan di repo. Log helpdesk mentah (email, Ticket ID) dilarang masuk repo publik.
10. **Lingkup:** jangan menambah database, server, UI, atau model lokal ke sub-proyek 1 tanpa revisi plan + ACC baru.

## 3. Alur pengerjaan (untuk AI agent / harness)

Ikuti urutan ini untuk setiap issue:

1. `gh issue view <N> --repo Rayazka/chatbot-tak` — baca deskripsi, langkah, acceptance criteria, dan dependensi.
2. `git fetch origin` → `git switch main` → `git pull --ff-only`.
3. `git switch -c issue/<N>-<slug>` — satu issue, satu branch.
4. Tandai issue sedang dikerjakan (comment/assignee) dan ubah status T-XX di Part C menjadi `In Progress`.
5. Baca file yang akan disentuh dan bagian terkait di `.agents/1-PRD-UTUH.md` serta spec. Jangan mengarang struktur yang belum ada di repo.
6. Kerjakan langkah di issue dengan siklus TDD. Kode lengkap untuk T-02–T-06 ada di Part C Rev 2.
7. Jalankan verifikasi yang tercantum di issue. Catat **perintah, exit code, hasil ringkas, dan tanggal**.
8. Simpan bukti ke issue dan ke `.agents/REVIEW.md` bila itu task QA/review (#20, #13, #14).
9. Perbarui Part C: status, evidence, next action. Perbarui checklist di #17.
10. Buat PR dengan judul `feat(#<N>): <judul ringkas>` dan minta review orang lain. **Berhenti di sini** bila tidak ada izin merge/commit.
11. Setelah review disetujui dan orang yang berwenang melakukan merge, tutup issue dan tandai T-XX `Done`.

Jika Anda dijalankan oleh harness AI, perlakukan aturan ini sebagai instruksi proyek yang mengikat, dan tunduk pada izin yang diberikan pengguna sesi tersebut — bukan pada default harness.

## 4. Format bukti

```text
Tanggal: 2026-11-27
Perintah: python -m pytest -q
Exit code: 0
Hasil: 14 passed
Catatan: 6 pertanyaan uji masih menunggu #20
```

Status `Blocked` wajib menyebutkan penyebab dan issue yang menunggu, misalnya "menunggu #1 (OD-01 produksi)".

## 5. Toolchain yang sudah diverifikasi

- Python 3.14.5, pip 26.1.1.
- Belum terpasang saat perencanaan: `pytest`, `groq`, `ruff`, `openai` — dipasang di T-02 dengan `python -m pip install pytest groq`.
- Perintah utama: `python -m pytest -q`, `python -m ingestion.build_store`, `python -m chat.run`.
- Model: `nomic-embed-text-v1.5` (embedding), `openai/gpt-oss-20b` (generasi, `temperature=0.1`, `stream=True`).
- Kuota Groq free: ~8K TPM / 200K token per hari — embedding dibatch maksimal 16 teks, retry maksimal 3.

## 6. Larangan keras

- Membuat branch tanpa issue atau branch yang dipakai untuk beberapa issue.
- Merge/pull request tanpa review orang lain.
- Commit/push/merge/tutup issue tanpa perintah eksplisit.
- Mengarang ACC, tanggal persetujuan, atau kutipan yang tidak pernah diberikan pengguna.
- Membuat plan atau backlog tandingan di luar `.agents/1-PRD-UTUH.md`.
- Menyimpan kunci API, data pribadi, atau log helpdesk mentah ke repo.
- Menandai task `Done` tanpa bukti tereksekusi.
- Mengubah A/B baseline Rev 2 tanpa revisi baru dan ACC.

## 7. Rujukan cepat

| Kebutuhan | Sumber |
| --- | --- |
| Aturan kerja | `AGENTS.md` |
| Baseline & task | `.agents/1-PRD-UTUH.md` Rev 2, Part C |
| Desain sub-proyek 1 | `docs/superpowers/specs/2026-11-25-tak-rag-subproyek-1-design.md` |
| Daftar task | GitHub issue #1–#24 (#17 = master) |
| Bukti verifikasi | `.agents/REVIEW.md` |
| Aturan branching | `CONTRIBUTING.md` dan `.github/CODEOWNERS` (dibuat di #18) |
