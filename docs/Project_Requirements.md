# Project Requirements — DocuGen AI

**Dokumen:** `Project_Requirements.md` (**Document 7 dari 7** dokumen turunan)
**Status:** Dokumen Induk (Master Consolidated Requirements Document)
**Sumber Acuan Tunggal:** `master_spec.md` — dokumen ini TIDAK membuat keputusan baru, TIDAK mengubah, dan TIDAK mengontradiksi `master_spec.md` maupun 5 dokumen turunan lainnya.

> **Catatan traceability wajib dibaca dulu:** Dokumen ini adalah **dokumen induk (umbrella document)** yang MENGGABUNGKAN dan MERUJUK seluruh requirement riil yang telah ditetapkan di:
> 1. `business_process.md` (Document 1) — ID `BR-xxx` (Business Requirements).
> 2. `PRD_Document_Control_SaaS.md` (Document 2) — ID `DC-xxx` (Document Control Requirements), lifecycle status Draft→...→Archived.
> 3. `PRD_LLM_Document_Generator.md` (Document 3) — ID `FR-xxx` (Functional) dan `NFR-xxx` (Non-Functional).
> 4. `ML_Deployment_Strategy.md` (Document 4) — strategi model, dataset, training, evaluasi, serving, RAG, MLOps, deployment, biaya.
> 5. `ML_Flow_Diagram.md` (Document 5) — visual alur ML level "apa yang terjadi".
> 6. `ML_Pipeline.md` (Document 6) — implementasi teknis ML level "bagaimana caranya" (17 pipeline konkret).
> 7. `System_Architecture_and_Data_Model.md` (dokumen referensi pendukung) — arsitektur sistem dan ERD 25 entitas.
>
> **Tidak ada ID requirement baru dibuat di dokumen ini.** Setiap kali sebuah requirement disebut, ID aslinya dicantumkan dan dirujuk ke dokumen sumber. Bagian §24 (Requirement Traceability Matrix) adalah bagian **terpenting** dari dokumen ini.

---

## 1. Project Overview

**DocuGen AI** adalah **Private Enterprise AI Application** — aplikasi AI konversasional internal dengan closed deployment (no public registration, admin-managed accounts only) yang menggabungkan dua kapabilitas inti dalam satu ekosistem (`master_spec.md` §1–§2):

- **A. LLM Application Development** — antarmuka chat bergaya ChatGPT/Claude/Gemini untuk generasi, pemahaman, editing, dan manajemen dokumen perusahaan berbasis RAG, dengan RBAC, document lifecycle governance, dan audit penuh.
- **B. LLM Model Development** — fine-tuning model open-weight (evaluasi LoRA vs QLoRA) agar model konsisten pada gaya penulisan dan format dokumen baku perusahaan dalam Bahasa Indonesia formal, dilengkapi RAG untuk menyuntikkan pengetahuan dinamis (SOP, kebijakan) tanpa retraining.

Proyek ini bersifat **riil** (akan dibangun bertahap sesuai ketersediaan hardware, `master_spec.md` §7), bukan sekadar blueprint portofolio. Prinsip paling fundamental yang mengikat seluruh dokumen: **AI output bukan otomatis menjadi dokumen resmi** — setiap dokumen bisnis penting wajib melalui human review (Administrator merangkap fungsi Document Control di MVP) sebelum berstatus final (`master_spec.md` §4, §9; `business_process.md` §11; `PRD_Document_Control_SaaS.md` §20).

Dokumen ini (Document 7) **tidak mendefinisikan ulang** detail apa pun yang sudah final di 6 dokumen sumber — ia menyatukan requirement tersebut ke dalam satu pandangan end-to-end proyek, lengkap dengan traceability, risiko, milestone, roadmap, dan pembagian tanggung jawab tim.

---

## 2. Project Scope

### 2.1 Dalam Scope (MVP)
Mengacu `master_spec.md` §2 dan §6, `business_process.md` §5.1, `PRD_LLM_Document_Generator.md` §31:
- Conversational AI (chat multi-turn, streaming, riwayat, pencarian riwayat).
- Document generation, document editor terstruktur penuh (heading/paragraf/bullet/numbered/tabel/undo-redo/autosave/version history/selective regeneration), export DOCX/PDF.
- Document upload dan document understanding dalam lingkup sesi chat.
- Knowledge Base dan RAG permission-aware di atas pgvector.
- Template Management dasar.
- User Management dan RBAC dua-role (Administrator, Authorized User), admin-managed account.
- Document Control penuh (7 status lifecycle: Draft → Under Review → Revision Required → Approved → Published → Archived, dengan Rejected sebagai status terminal alternatif) yang dijalankan oleh Administrator.
- Admin Dashboard: User Management, Role & Permission, KB Management, Template Management, Audit Log, Usage Analytics, System Monitoring ringkas.
- Fine-tuning model open-weight (kandidat 7B/8B, LoRA/QLoRA) + pipeline dataset publik/sintetis + evaluasi + MLOps dasar (tracking, registry, canary, rollback) + model serving self-hosted (vLLM).
- Keamanan baseline penuh (`master_spec.md` §9): RAG permission-aware, klasifikasi dokumen 4-tingkat, audit log append-only, default-deny API eksternal, otorisasi backend.

### 2.2 Luar Scope (Permanen atau Phase 2+)
- **Permanen out of scope**: image generation; foundation model pretraining dari nol; multi-tenant SaaS komersial ke pelanggan eksternal.
- **Out of scope MVP (Phase 2 kandidat)**: voice interaction; Department-Based Access Control (DBAC) granular; Resource-Level Permission granular; role Document Control terpisah dari Administrator; migrasi vector DB pgvector → Qdrant; self-service password reset; segregation of duties reviewer vs approver; multi-level approval workflow; diff/version comparison viewer kaya; notifikasi eksternal (email/SMS); eskalasi model size ke 14B+.

### 2.3 Batasan Proyek
- Hardware lokal terbatas (AMD Ryzen 3 3250U, 8GB RAM, tanpa GPU dedicated) — hanya untuk dataset prep CPU-bound, dev backend/frontend, penulisan script tanpa eksekusi penuh (`master_spec.md` §7).
- Belum ada dokumen perusahaan riil tersedia saat ini — dataset awal public + synthetic, pipeline dirancang menyerap dokumen historis begitu tersedia (`master_spec.md` §4; `ML_Deployment_Strategy.md` §B).
- Model final (Qwen2.5-7B-Instruct / Llama-3.1-8B-Instruct / Mistral-7B-Instruct-v0.3), LoRA vs QLoRA, context window produksi, skema quantization, dan beberapa parameter lain masih **TBD** — ditentukan lewat benchmarking internal, tidak diasumsikan di muka (`ML_Deployment_Strategy.md` Ringkasan TBD).

---

## 3. Business Requirements

Seluruh Business Requirement (`BR-001` s.d. `BR-018`) didefinisikan **secara final** di `business_process.md` §6 dan **tidak diduplikasi** di sini. Ringkasan rujukan (detail lengkap + Acceptance Criteria bisnis ada di `business_process.md` §6 dan §20):

| ID | Ringkasan | Prioritas |
|---|---|---|
| BR-001 | Admin-managed account provisioning (tidak ada registrasi mandiri) | Must |
| BR-002 | RBAC dasar (Administrator vs Authorized User) | Must |
| BR-003 | Conversational document generation | Must |
| BR-004 | Export dokumen (DOCX/PDF) | Must |
| BR-005 | Human review wajib sebelum status resmi | Must |
| BR-006 | Document Control oleh Admin (MVP) | Must |
| BR-007 | Klasifikasi dokumen (Public/Internal/Confidential/Restricted) | Must |
| BR-008 | RAG permission-aware | Must |
| BR-009 | Approval dataset training eksplisit | Must |
| BR-010 | Approval model sebelum deployment produksi | Must |
| BR-011 | Audit log akses sensitif | Must |
| BR-012 | Larangan default kirim data ke API eksternal | Must |
| BR-013 | Versioning dan riwayat dokumen | Should |
| BR-014 | Template dokumen perusahaan | Should |
| BR-015 | Notifikasi status approval | Should |
| BR-016 | Department-Based Access Control | Could (Phase 2, TBD) |
| BR-017 | Resource-Level Permission | Could (Phase 2, TBD) |
| BR-018 | Role Document Control terpisah | Could (Phase 2, TBD) |

Proses bisnis pendukung (Admin Process, User Process, Document Generation/Review/Management, Dataset Management, Model Training/Deployment Approval, Security & Governance) dijabarkan penuh di `business_process.md` §7–§19, termasuk diagram Mermaid dan daftar TBD operasional (SLA approval, admin backup, konflik approval bersamaan, rollback produksi) — **tidak diulang di sini**.

---

## 4. Functional Requirements

Seluruh Functional Requirement (`FR-001` s.d. `FR-037`, total **37 FR**) didefinisikan final di `PRD_LLM_Document_Generator.md` §9 dan `PRD_Document_Control_SaaS.md` §21 (`DC-001` s.d. `DC-020`, total **20 DC**). Pengelompokan ringkas:

| Kelompok Kapabilitas | Requirement ID | Sumber |
|---|---|---|
| Autentikasi & sesi | FR-001 | Document 3 §9 |
| Chat & percakapan | FR-002 s.d. FR-007 | Document 3 §9 |
| Document generation & regenerasi | FR-008 s.d. FR-011 | Document 3 §9 |
| Document editor (heading/paragraf/list/tabel/undo-redo/autosave/version) | FR-012 s.d. FR-020 | Document 3 §9 |
| Upload & document understanding | FR-021 s.d. FR-024 | Document 3 §9 |
| Knowledge Base & RAG | FR-025 s.d. FR-028 | Document 3 §9 |
| Template management | FR-029, FR-030 | Document 3 §9 |
| Export dokumen | FR-031 | Document 3 §9 |
| User & role/permission management | FR-032 s.d. FR-034 | Document 3 §9 |
| Admin dashboard (analytics, audit log) | FR-035, FR-036 | Document 3 §9 |
| Pengajuan approval dokumen | FR-037 | Document 3 §9 |
| Document Control lifecycle (registrasi, edit, submit, review, approve, publish, reject, reopen, klasifikasi, akses, versi, revisi, notifikasi, arsip, retensi, audit, pencarian, integrasi AI, diff versi) | DC-001 s.d. DC-020 | Document 2 §21 |

Detail penuh per-ID (Description, Priority, Actor, Preconditions, Expected Result, Acceptance Criteria) **tidak diduplikasi** — rujuk `PRD_LLM_Document_Generator.md` §9 dan `PRD_Document_Control_SaaS.md` §21. Lihat §24 (Traceability Matrix) untuk keterkaitan FR/DC terhadap BR dan ML requirement.

---

## 5. Non-Functional Requirements

Seluruh Non-Functional Requirement (`NFR-001` s.d. `NFR-013`, total **13 NFR**) didefinisikan final di `PRD_LLM_Document_Generator.md` §10, ditambah `NFR-DC-01` s.d. `NFR-DC-08` di `PRD_Document_Control_SaaS.md` §22. Ringkasan kategori:

| Kategori | Requirement ID | Sumber |
|---|---|---|
| Performance (latensi chat, generate dokumen, transisi status dokumen) | NFR-001, NFR-002, NFR-DC-01, NFR-DC-02 | Document 2 §22, Document 3 §10 |
| Scalability (concurrent user, pertumbuhan dokumen) | NFR-003, NFR-DC-03 | Document 2 §22, Document 3 §10 |
| Availability | NFR-004 | Document 3 §10 |
| Security/Authorization backend | NFR-005, NFR-DC-08 | Document 2 §22, Document 3 §10 |
| Privacy & data governance | NFR-006, NFR-007 | Document 3 §10 |
| Auditability | NFR-008, NFR-DC-04, NFR-DC-07 | Document 2 §22, Document 3 §10 |
| Usability | NFR-009, NFR-DC-05 | Document 2 §22, Document 3 §10 |
| Dukungan Bahasa Indonesia formal | NFR-010 | Document 3 §10 |
| Observability/Monitoring | NFR-011 | Document 3 §10 |
| Portabilitas inference | NFR-012 | Document 3 §10 |
| Disaster recovery/backup | NFR-013 | Document 3 §10 |
| Maintainability (config-driven) | NFR-DC-06 | Document 2 §22 |

Detail penuh (Description, Priority, Acceptance Criteria) rujuk dokumen sumber masing-masing.

---

## 6. Software Requirements

Mengacu `master_spec.md` §4 (Decision Log) dan `System_Architecture_and_Data_Model.md` §2:

| Komponen | Teknologi | Peran |
|---|---|---|
| Frontend | **Next.js** (React, SSR) | Chat UI, Document Workspace/Editor, Admin Dashboard |
| Backend aplikasi | **Python FastAPI** | REST/HTTP API (+WebSocket/SSE streaming), autentikasi, RBAC enforcement, business logic, orkestrasi RAG, audit logging — **terpisah dari inference service** |
| Database relasional | **PostgreSQL** | Seluruh operational data (user, chat, dokumen, template, audit log) |
| Vector store | **pgvector** (extension PostgreSQL, default MVP) → evaluasi **Qdrant** di Phase 2 bila volume/fitur retrieval lanjutan dibutuhkan | Indexing embedding knowledge base |
| Object storage | S3-compatible (MinIO self-hosted MVP, atau layanan S3/GCS kompatibel) | File upload asli, hasil export DOCX/PDF, artefak model besar |
| Model serving | **vLLM** (OpenAI-compatible API) | Serving model fine-tuned, continuous batching, PagedAttention, streaming |
| Experiment tracking & registry | **MLflow** (backend store PostgreSQL) | Tracking eksperimen, model registry (Staging/Production/Archived) |
| Dataset/model versioning | **DVC** (+ Git/Git LFS untuk adapter kecil) | Reproducibility dataset dan model artifact |
| Training stack Python | HF Transformers + PEFT (LoRA/QLoRA) + TRL (SFTTrainer) + bitsandbytes + accelerate | Fine-tuning |
| Orkestrasi container | Docker Compose (MVP) → Kubernetes (Phase 2+ dievaluasi ulang berdasarkan traksi) | Deployment Application Platform |
| Monitoring | Prometheus + Grafana + Alertmanager | Observability aplikasi dan inference |
| CI/CD | GitHub Actions/GitLab CI | Validasi dataset, build image, deployment bertahap (staging → canary → full rollout) |

Rujukan: `ML_Deployment_Strategy.md` §G.1 (justifikasi vLLM), `System_Architecture_and_Data_Model.md` §2.2 (justifikasi pemisahan backend-inference), `ML_Pipeline.md` §1.4 (tools lengkap).

---

## 7. AI/ML Requirements

Mengacu `ML_Deployment_Strategy.md` §A–§D dan `master_spec.md` §3, §8:

1. **Strategi model**: fine-tuning pretrained open-weight model (bukan training from scratch); evaluasi LoRA vs QLoRA berbasis benchmark internal, bukan diasumsikan.
2. **Pembedaan fungsi fine-tuning vs RAG** (final, tidak boleh tertukar — `master_spec.md` §3): fine-tuning mengubah *behavior* (gaya, struktur, instruction-following) dan bersifat statis; RAG menyuntikkan *knowledge* dinamis (SOP, kebijakan) saat inference. Dokumen perusahaan yang terus berubah **wajib** masuk RAG, tidak pernah via fine-tuning.
3. **Kandidat model** (`master_spec.md` §8.1, `ML_Deployment_Strategy.md` §A.1): Qwen2.5-7B-Instruct (kandidat utama, Apache 2.0, IndoMMLU 56.4), Llama-3.1-8B-Instruct (kandidat kedua, IFEval tertinggi, lisensi Community License perlu review legal), Mistral-7B-Instruct-v0.3 (kandidat ketiga, Apache 2.0, IndoMMLU TBD). Model final **TBD**, diputuskan lewat benchmarking internal (§E.2 ML Deployment Strategy), tidak dari skor publik saja.
4. **Ukuran model**: kelas 7B–8B, sejalan VRAM QLoRA ~8-14GB (feasible di Colab/Kaggle free tier → GPU rental 24GB). Eskalasi ke 14B+ adalah opsi Phase 2+ TBD.
5. **Arsitektur**: decoder-only transformer standar, tidak ada modifikasi arsitektur — murni fine-tuning di atas bobot pretrained.
6. **Serving**: self-hosted via vLLM, OpenAI-compatible API, wajib streaming, bukan API eksternal pihak ketiga (default), sesuai data governance `master_spec.md` §9.7.
7. **RAG architecture**: ingestion → text extraction → OCR fallback → structure-aware chunking → embedding multilingual ID-capable → pgvector (MVP) → retrieval+reranking+context construction+generation dengan source citation; permission-aware retrieval wajib via filter ACL pre-query (`ML_Deployment_Strategy.md` §H).
8. **MLOps**: experiment tracking (MLflow), dataset/model versioning, model registry dengan status approval gate, CI/CD, canary release, rollback, monitoring, drift detection, feedback loop (tanpa otomatis masuk training set) — detail penuh §I `ML_Deployment_Strategy.md` dan Pipeline 1–17 `ML_Pipeline.md`.

---

## 8. Dataset Requirements

Mengacu `ML_Deployment_Strategy.md` §B dan `ML_Pipeline.md` Pipeline 2–6:

1. **Kondisi awal**: tidak ada dokumen perusahaan riil tersedia → dataset awal **public + synthetic**; pipeline dirancang menyerap dokumen historis approved perusahaan begitu tersedia (`master_spec.md` §4).
2. **Kategori dataset**: general language, instruction, document generation, document editing, document structure, Indonesian language quality, evaluation (held-out, tidak pernah dicampur training).
3. **Pemisahan tegas jenis data** (wajib secara teknis, bukan hanya konvensi): operational data (tidak otomatis jadi training data), RAG knowledge data (vector DB, tidak melalui fine-tuning), fine-tuning training data (butuh approval eksplisit), evaluation data (locked, anti-leakage).
4. **PII detection & redaction wajib** sebelum masuk pipeline lebih lanjut (regex + NER Bahasa Indonesia), dengan flag `pii_checked` untuk audit.
5. **Approval dataset eksplisit** (mandatory gate, merujuk `business_process.md` §13 dan BR-009) sebelum dataset berstatus "Approved for Training" dan dapat dipakai fine-tuning.
6. **Pipeline konkret (10 tahap)**: Raw Data Collection → Source Validation → Cleaning → Deduplication → PII Detection → Annotation → Quality Validation → Train/Val/Test Split (85/10/5 stratified) → Dataset Versioning (DVC + Git tag) → Approved Dataset — tahap 1-8 feasible CPU-only di local dev (`master_spec.md` §7).
7. **Konvensi versioning**: `dataset-v<MAJOR>.<MINOR>-<YYYYMMDD>[-<label>]`, immutable setelah tag dipush (`ML_Pipeline.md` §6.6).
8. **Konversi dokumen historis approved**: hanya dokumen berstatus Approved/Published (lifecycle Document Control, `PRD_Document_Control_SaaS.md`) boleh dikonversi jadi instruction-response pair, dengan human review wajib dan PII pass sebelum masuk kandidat dataset (`ML_Deployment_Strategy.md` §B.3).

---

## 9. Model Requirements

Mengacu `ML_Deployment_Strategy.md` §D–§G dan `ML_Pipeline.md` Pipeline 7–11:

1. **Metode fine-tuning**: QLoRA (4-bit NF4) sebagai jalur utama eksperimen awal (VRAM ~8-14GB); LoRA (16-bit) sebagai pembanding kualitas bila QLoRA menunjukkan degradasi; Full FT **tidak dipakai** di Phase 1 (VRAM ~88GB tidak realistis).
2. **Konfigurasi eksperimen awal** (starting point, bukan final): LoRA rank 16, alpha 32, dropout 0.05, target modules attention projections (+MLP di config konkret Pipeline), learning rate 2e-4 cosine scheduler, batch size efektif 16, 3 epoch dengan early stopping, max sequence length 2048, optimizer paged_adamw_8bit.
3. **Lingkungan eksekusi training/eval GPU-berat**: wajib Google Colab Pro/Kaggle (free/low-cost tier) dahulu, eskalasi ke GPU rental (RunPod/Vast.ai/Lambda) bila perlu scale — **tidak pernah** di dev lokal CPU-only (`master_spec.md` §7).
4. **Evaluasi berjenjang** (tidak dicampur): base model → fine-tuned (tanpa RAG) → fine-tuned+RAG, mengukur dimensi: instruction following, document completeness, structural consistency, factual accuracy, hallucination, Bahasa Indonesia quality, format compliance, RAG faithfulness, latency, security (prompt injection/ACL bypass).
5. **Model registry**: status lifecycle `None → Staging → Production → Archived/Retired`, dengan invariant hanya satu versi `Production` aktif per model, promosi wajib lewat UAT staging + approval formal (merujuk BR-010, Model Training Approval `business_process.md` §14).
6. **Deployment**: canary release (porsi kecil traffic) → monitoring ketat → rollout bertahap atau rollback otomatis berbasis threshold (`error_rate_5xx > 2%` atau `p95 latency naik >50%`) — detail konkret `ML_Pipeline.md` Pipeline 11.
7. **Quantization serving**: skema konkret (AWQ/GPTQ/bitsandbytes 4-bit/8-bit) **TBD**, ditentukan kompatibilitas vLLM dan hasil benchmark.

---

## 10. Infrastructure Requirements

Mengacu `System_Architecture_and_Data_Model.md` §1, §6 dan `ML_Deployment_Strategy.md` §J:

1. **Pemisahan platform**: Application Platform (CPU-bound, backend FastAPI + worker, scale horizontal murah) dipisah tegas dari ML/Inference Platform (GPU-bound, mahal, scale independen) dan Training Pipeline (offline, bursty, Colab/Kaggle/GPU rental).
2. **Orkestrasi MVP**: **Docker Compose** (single-host/VM) cukup untuk skala MVP — Kubernetes baru dievaluasi di Phase 2+ bila ada kebutuhan konkret (multi-node, autoscaling real-time, GPU scheduling dinamis multi-tenant) yang tidak terpenuhi Compose.
3. **Komponen Compose MVP**: container frontend (Next.js), backend (FastAPI), worker, PostgreSQL (+pgvector), object storage (MinIO) — semuanya di host CPU; Inference Service digambar **terpisah** di infrastruktur GPU (cloud rental persisten atau on-prem bila sudah ada traksi).
4. **Containerization**: setiap komponen (backend, vLLM serving, embedding service) dikemas container Docker terpisah untuk deployment dan rollback konsisten.
5. **Backup & DR**: PostgreSQL (termasuk pgvector) di-backup rutin terjadwal; model artifact dan dataset versioned di MLflow registry + object storage, bukan hanya disk lokal. Strategi DR formal (RPO/RTO, multi-region) **TBD untuk MVP** — rekomendasi minimal: backup terjadwal + dokumentasi recovery manual, eskalasi formal di Phase 2.
6. **Monitoring stack**: Prometheus (scrape 15s) + Grafana (dashboard latency/throughput/error rate/drift) + Alertmanager (threshold alert ke AI/ML Team).

---

## 11. Hardware Requirements

**Wajib merujuk `master_spec.md` §7 secara langsung, tidak mengarang ulang angka:**

| Tier | Spesifikasi | Peruntukan |
|---|---|---|
| **Local dev machine** | AMD Ryzen 3 3250U (2C/4T), 8GB RAM, integrated Radeon graphics, **tanpa GPU/CUDA dedicated** | Dataset prep/cleaning/annotation (CPU-bound, dataset kecil-menengah), development backend/frontend, development database, penulisan script training/evaluation (**tanpa eksekusi penuh**), smoke test inference model sangat kecil terkuantisasi (<1B/CPU) — **bukan** untuk benchmarking model kandidat 7B/8B. |
| **Cloud GPU — tahap eksperimen** | Google Colab Pro (~$9.99/bulan) / Kaggle (gratis, GPU T4/P100, kuota jam mingguan terbatas) **dicoba dahulu** | Seluruh benchmarking model kandidat, fine-tuning LoRA/QLoRA, dan evaluasi GPU-berat — **wajib**, tidak boleh dilakukan di local dev. |
| **Cloud GPU — eskalasi scale** | GPU rental on-demand: RunPod/Vast.ai/Lambda — kelas RTX 3090/4090 (24GB) hingga A100 (80GB) sesuai tabel harga `master_spec.md` §8.3 | Dipakai bila kuota Colab/Kaggle tidak cukup untuk durasi/ukuran eksperimen. |
| **Production inference GPU** | Infrastruktur produksi **terpisah** (cloud GPU rental persisten, atau on-prem GPU bila sudah ada traksi nyata) | Serving model fine-tuned ke traffic produksi — detail sizing di `ML_Deployment_Strategy.md` §J–§K. |

**VRAM requirement** (sumber: Spheron Network GPU VRAM guide 2026, io.net fine-tuning guide, `master_spec.md` §8.2): QLoRA 7B/8B ~8-14GB (GPU minimum 12-24GB kelas RTX 3090/4090/A100); Full FT 7B ~88GB (**tidak direkomendasikan**, tidak realistis untuk MVP).

**Harga GPU rental** (`master_spec.md` §8.3, harus dicek ulang di halaman resmi provider sebelum eksekusi budget riil — harga berubah cepat): RTX 3090 24GB $0.22–$0.50/jam, RTX 4090 24GB $0.34–$0.74/jam, A100 PCIe/SXM 80GB $1.19–$1.59/jam, H100 PCIe 80GB $1.99–$2.89/jam (RunPod Community vs Secure Cloud).

---

## 12. Security Requirements

Mengacu `master_spec.md` §9 (Security & Data Governance Baseline, final, berlaku di seluruh dokumen), ditegakkan ulang di `PRD_LLM_Document_Generator.md` §25 dan `PRD_Document_Control_SaaS.md` §23:

1. Percakapan user tidak otomatis dipakai untuk training (NFR-006).
2. Dokumen upload tidak otomatis menjadi training data (NFR-006, SEC-DC-05).
3. Dataset training butuh approval eksplisit (BR-009, `business_process.md` §13).
4. RAG retrieval **wajib permission-aware** — filter ACL sebelum masuk context, bukan hanya system prompt (BR-008, FR-027, SEC-DC-03, NFR-005; mekanisme konkret `ML_Deployment_Strategy.md` §H.8 dan SQL contoh `ML_Pipeline.md` §14.6).
5. Dokumen sensitif wajib punya klasifikasi (BR-007, FR-022, DC-009, SEC-DC-01).
6. Semua akses sensitif Admin tercatat di audit log append-only, tidak dapat dihapus/diedit lewat UI (BR-011, FR-036, NFR-008, DC-015, SEC-DC-04/SEC-DC-07).
7. Data internal tidak dikirim ke API eksternal pihak ketiga tanpa approval formal berjenjang (default: deny) (BR-012, NFR-007, SEC-DC-06).
8. **Otorisasi ditegakkan di backend (API layer)**, tidak pernah hanya UI/system prompt — system prompt tidak pernah menjadi pengganti backend authorization (NFR-005, SEC-DC-02; detail flow `System_Architecture_and_Data_Model.md` §5.2).
9. Resistensi terhadap prompt injection, jailbreak dasar, dan RAG ACL bypass diuji sebagai bagian evaluasi model (Security eval, `ML_Deployment_Strategy.md` §E.1).
10. Rate limiting per user ditegakkan di layer backend FastAPI, bukan di serving engine (`ML_Pipeline.md` §12.8).

---

## 13. Privacy Requirements

1. PII (NIK, nomor telepon, email, nama, alamat) wajib dideteksi dan diredaksi pada dataset training sebelum digunakan sebagai contoh fine-tuning — rule-based regex + model NER Bahasa Indonesia (`ML_Deployment_Strategy.md` §B.5, `ML_Pipeline.md` §3.3).
2. Operational data (percakapan, dokumen upload) secara teknis terisolasi dari training data pipeline (folder/skema/kontrol akses terpisah), bukan hanya konvensi penamaan — mencegah kebocoran data user tanpa consent (`master_spec.md` §9.1-§9.2).
3. Feedback user (thumbs up/down) **tidak otomatis** masuk training set — harus melalui review manual dan gate approval dataset yang sama dengan jalur normal (`ML_Pipeline.md` Pipeline 16).
4. Dokumen Confidential/Restricted mewajibkan audit log atas setiap pembacaan, termasuk oleh Administrator (`master_spec.md` §9.6, SEC-DC-04).
5. Klasifikasi dokumen default "Internal" untuk upload tanpa niat berbagi, hanya terlihat bagi pengunggah kecuali diubah eksplisit (`PRD_LLM_Document_Generator.md` §16).
6. Periode retensi dokumen **belum ditetapkan** (TBD, menunggu kebijakan legal/compliance) — sistem wajib tetap menyediakan field retensi konfigurabel tanpa nilai hard-coded sejak MVP (DC-016, `PRD_Document_Control_SaaS.md` §17).

---

## 14. Integration Requirements

1. **Integrasi Document Generator ↔ Document Control**: setiap output AI Document Generator otomatis diregistrasi ke Document Control dengan status awal **Draft**, metadata sumber `ai-generated`, referensi sesi generasi untuk traceability — tidak ada jalur pintas ke status resmi (DC-018, FR-037, `PRD_Document_Control_SaaS.md` §20).
2. **Integrasi Document Control ↔ RAG**: hanya dokumen berstatus **Published** yang masuk index retrieval RAG (DC-019; mekanisme pengecualian indexing).
3. **Integrasi Backend ↔ Inference Service**: kontrak API internal stabil (OpenAI-compatible via vLLM), backend tidak pernah memanggil model langsung sebagai library in-process, sehingga model dapat diganti tanpa mengubah kontrak API aplikasi (NFR-012, T2).
4. **Integrasi RAG Pipeline ↔ Vector DB ↔ Object Storage**: alur ingestion (Admin upload → klasifikasi/ACL → chunking/embedding → vector DB) dan retrieval (query → embed → filter ACL pre-search → similarity search → context → generation) — lihat sequence diagram `System_Architecture_and_Data_Model.md` §7.2.
5. **Integrasi Training Pipeline ↔ Model Registry ↔ Inference Service**: Model Registry adalah **satu-satunya jembatan resmi** — tidak ada jalur langsung training→serving tanpa registry dan approval gate (`System_Architecture_and_Data_Model.md` §3.1).
6. **Integrasi API eksternal pihak ketiga**: default deny, hanya opsi evaluatif TBD dengan guardrail data governance dan approval formal tertulis (`master_spec.md` §9.7, BR-012).

---

## 15. Deployment Requirements

Mengacu `ML_Pipeline.md` Pipeline 11 dan `ML_Deployment_Strategy.md` §J:

1. **Trigger deployment**: manual approval, bukan otomatis saat commit — dipicu setelah model/dokumen stage promotion disetujui.
2. **Staging-first wajib**: tidak ada deployment langsung ke produksi tanpa smoke test staging (`business_process.md` §15, BR-010).
3. **Canary release**: rollout ke porsi kecil traffic (misal 10%) dengan monitoring ketat selama window observasi, sebelum full rollout.
4. **Rollback otomatis**: gate otomatis di tahap canary (`error_rate_5xx > 2%` atau `p95 latency` naik >50%) memicu rollback ke versi Production sebelumnya tanpa menunggu intervensi manual untuk tahap teknis; investigasi root cause tetap manual.
5. **Model lama tetap tersedia sebagai fallback** (status `Archived`, bukan dihapus) — jalur rollback instan.
6. **Deployment checklist**: model version, dataset version, eval report link, approval ticket ID dilampirkan otomatis ke deployment record untuk audit.
7. **Pemisahan siklus deployment**: backend app (frekuensi tinggi, ringan) vs Inference Service (frekuensi rendah, berat — loading weight ke VRAM) — tidak boleh digabung dalam satu siklus rilis (`System_Architecture_and_Data_Model.md` §2.2).

---

## 16. Monitoring Requirements

Mengacu `ML_Pipeline.md` Pipeline 15 dan `PRD_LLM_Document_Generator.md` NFR-011:

1. **Metrik aplikasi**: latency, error rate, usage analytics (jumlah percakapan, dokumen dibuat, user aktif) — update interval ≤5 menit (NFR-011).
2. **Metrik inference**: `inference_request_latency_seconds`, `inference_tokens_per_second`, `inference_error_rate_5xx`, `gpu_utilization_percent`.
3. **Metrik RAG**: `rag_retrieval_latency_seconds`, `rag_empty_result_rate`, `rag_acl_denied_count`.
4. **Drift signal**: `model_output_embedding_drift_score` — kenaikan signifikan men-trigger investigasi manual, **bukan** retraining otomatis.
5. **Format compliance monitoring produksi**: `document_format_compliance_violation_rate` — sampling periodik output produksi dengan rule yang sama seperti evaluasi training.
6. **Stack**: Prometheus (scrape 15s) + Grafana (dashboard) + Alertmanager (threshold alert ke AI/ML Team via Slack/email/webhook), dengan dead man's switch untuk mendeteksi monitoring itu sendiri down.
7. **Audit log sebagai komponen monitoring governance**: 100% aksi sensitif tercatat dan dapat diekspor untuk audit eksternal (NFR-DC-07).

---

## 17. Testing Requirements

1. **Functional testing**: seluruh FR/DC berpriority Must diuji sesuai Acceptance Criteria masing-masing di dokumen sumber (§9 PRD LLM Document Generator, §21 PRD Document Control).
2. **Non-functional testing**: load test dasar (minimal 50 concurrent sessions tanpa error rate >1%, NFR-003), security test otorisasi backend (permission test menghasilkan 403 untuk akses tidak sah, NFR-005), uji restore backup (NFR-013).
3. **RAG permission-aware test**: skenario minimal dua user dengan permission berbeda terhadap entri KB yang sama **wajib** menghasilkan context retrieval yang berbeda secara terbukti (FR-027, BR-008) — termasuk negative test ACL pasca-ingestion (`ML_Pipeline.md` §13.10).
4. **Model evaluation test**: automated benchmark (ROUGE-L/BLEU, format compliance score, LLM-as-judge) + human review minimal 2 reviewer independen sebelum promosi model (`ML_Pipeline.md` §9.9); regression check terhadap model Production saat ini wajib tidak boleh turun >5% pada format compliance.
5. **Export test**: dokumen hasil ekspor DOCX/PDF harus dapat dibuka tanpa corrupt di Microsoft Word/Adobe Reader dengan struktur visual identik editor.
6. **User journey end-to-end test**: Journey A (generate dokumen baru) dan Journey B (revisi berdasarkan instruksi) dapat dijalankan tanpa intervensi developer (`PRD_LLM_Document_Generator.md` §8, §30).
7. **Document lifecycle test**: tidak ada jalur UI/API yang memungkinkan dokumen berpindah status langsung dari Draft/Under Review ke Published tanpa melalui Approved (`PRD_Document_Control_SaaS.md` §27).
8. **Audit invariant test**: hanya satu versi model berstatus `Production` per nama model pada satu waktu; tidak ada proposal retraining yang "lompat gate" approval (`ML_Pipeline.md` §10.9, §17.10).

---

## 18. Acceptance Criteria

Proyek dinyatakan memenuhi kriteria penerimaan MVP apabila **seluruh** kondisi berikut terpenuhi (dikompilasi dari `business_process.md` §20, `PRD_Document_Control_SaaS.md` §27, dan `PRD_LLM_Document_Generator.md` §30 — tidak ada kriteria baru dikarang):

1. Tidak ada mekanisme registrasi akun mandiri; seluruh akun dibuat Administrator (BR-001).
2. Dokumen berstatus "resmi/final" hanya setelah lolos proses review/approval Administrator; tidak ada jalur AI output langsung menjadi dokumen resmi (BR-005, BR-006, DC integrasi §20).
3. Seluruh FR/NFR berpriority Must terimplementasi dan lulus pengujian sesuai Acceptance Criteria masing-masing (§9–§10 PRD LLM Document Generator).
4. Seluruh 20 DC (Document Control) berstatus "implemented & tested" dengan bukti test case (§21 PRD Document Control).
5. RAG permission-aware terverifikasi dengan skenario dua user berbeda permission menghasilkan context berbeda secara terbukti (BR-008, FR-027, DC-019).
6. Setiap dataset dipakai fine-tuning memiliki catatan status "Approved for Training" dengan identitas approver dan timestamp auditable (BR-009).
7. Setiap model live produksi memiliki catatan "Approved for Deployment" dengan approver, hasil evaluasi, dan riwayat staging auditable (BR-010).
8. Audit log mencatat 100% aksi sensitif selama periode UAT, tervalidasi sampling manual (BR-011, NFR-DC-07).
9. Tidak ada payload data internal terkirim ke API eksternal tanpa approval formal tercatat (BR-012).
10. Journey A dan Journey B dapat dijalankan end-to-end oleh pengguna uji tanpa intervensi developer.
11. Ekspor DOCX/PDF menghasilkan file valid, struktur identik editor.
12. Tidak ada jalur bypass RBAC/permission di seluruh endpoint API (NFR-005).

---

## 19. Project Dependencies

| Dependency | Jenis | Dampak bila tidak terpenuhi |
|---|---|---|
| Ketersediaan kuota GPU Colab Pro/Kaggle | Eksternal (platform pihak ketiga) | Eksperimen fine-tuning/evaluasi tertunda; perlu eskalasi lebih awal ke GPU rental berbayar |
| Ketersediaan budget GPU rental (RunPod/Vast.ai/Lambda) | Finansial | Scale eksperimen dan production serving terbatas; perlu keputusan budget sebelum eksekusi |
| Ketersediaan dokumen historis perusahaan berstatus Approved/Published | Data bisnis internal | Dataset fine-tuning tetap bertumpu pada public+synthetic, kualitas domain-specific terbatas |
| Approval formal dari Administrator (dataset, model, dokumen) | Proses governance internal | Pipeline training/deployment/document publishing terhenti di gate approval (ini **disengaja**, bukan bug) |
| Keputusan benchmarking internal model kandidat | Teknis (AI/ML Team) | Model final belum bisa ditetapkan; pipeline tetap beroperasi dengan kandidat pertama (Qwen2.5-7B-Instruct) sebagai default eksperimen |
| Review legal/compliance lisensi Llama 3.1 Community License (jika kandidat ini dipilih) | Legal | Potensi blocker deploy produksi bila lisensi tidak sesuai kebutuhan komersial internal |
| Kebijakan retensi dokumen final dari tim legal/compliance | Organisasi | Field retensi tetap kosong/TBD secara fungsional tanpa blocking go-live (by design, DC-016) |
| Infrastruktur PostgreSQL, object storage, MLflow server | Teknis internal | Seluruh operational data dan MLOps tracking bergantung pada ketersediaan ini |
| Kesiapan tim AI/ML (kurasi dataset, evaluasi, fine-tuning) | Organisasi/SDM | Siklus model development melambat tanpa kapasitas tim yang cukup |

---

## 20. Risk Assessment

| # | Risiko | Kategori | Dampak | Probabilitas | Mitigasi |
|---|---|---|---|---|---|
| 1 | Hardware lokal terbatas (CPU-only, 8GB RAM) tidak mampu eksekusi training/eval model 7B/8B | Teknis/Hardware | Tinggi — blocker eksperimen bila tidak dimitigasi | Pasti terjadi (realita yang sudah diketahui) | Seluruh eksekusi GPU-berat dipindah ke Colab Pro/Kaggle (free/low-cost tier) lalu GPU rental sesuai kebutuhan skala (`master_spec.md` §7); local dev dibatasi untuk tugas CPU-bound (dataset prep, kode, smoke test model kecil) |
| 2 | Ketergantungan pada GPU cloud pihak ketiga (quota terbatas, harga berubah, ketersediaan tidak terjamin) | Operasional/Finansial | Menengah — dapat memperlambat iterasi eksperimen | Menengah | Pemisahan anggaran eksperimen vs produksi (`ML_Deployment_Strategy.md` §K.4); cek ulang harga provider sebelum eksekusi budget riil; checkpoint resume untuk sesi free-tier yang terputus |
| 3 | Dataset perusahaan riil belum tersedia — model fine-tuned mungkin kurang relevan dengan gaya/format aktual perusahaan | Data/Kualitas Model | Tinggi untuk kualitas output domain-specific | Pasti terjadi di fase awal | Pipeline dirancang menyerap dokumen historis approved begitu tersedia (`ML_Deployment_Strategy.md` §B.3); dataset publik+synthetic dikurasi ketat dengan human review; evaluasi berjenjang base vs fine-tuned vs +RAG untuk mengukur kontribusi riil |
| 4 | Model kandidat (Qwen2.5-7B/Llama-3.1-8B/Mistral-7B) ternyata tidak memenuhi target kualitas Bahasa Indonesia formal setelah fine-tuning | Teknis/ML | Tinggi — scope creep ke model lebih besar (14B+) yang di luar budget hardware | Menengah | Benchmarking internal berjenjang (§E.2 ML Deployment Strategy) sebelum keputusan final; eskalasi 14B+ ditandai TBD terkondisi, bukan default |
| 5 | Lisensi Llama 3.1 Community License berisiko secara legal untuk deployment komersial skala besar | Legal/Compliance | Menengah-Tinggi bila kandidat ini dipilih tanpa review | Rendah-Menengah | Wajib review legal/compliance sebelum deploy produksi jika Llama dipilih; Qwen2.5 (Apache 2.0) dan Mistral (Apache 2.0) sebagai alternatif permisif |
| 6 | RAG retrieval bug menyebabkan kebocoran dokumen di luar permission user (ACL bypass) | Security | Sangat Tinggi — insiden keamanan data | Rendah (bila mitigasi diterapkan), Tinggi bila diabaikan | Filter ACL wajib sebagai predicate query vector DB (pre-filter, bukan post-filter), negative test ACL rutin, audit log setiap retrieval dokumen Confidential/Restricted (`ML_Deployment_Strategy.md` §H.8) |
| 7 | AI output "bocor" menjadi dokumen resmi tanpa human review (bypass governance) | Governance/Bisnis | Sangat Tinggi — risiko reputasi dan kepatuhan | Rendah (bila mitigasi diterapkan) | Status lifecycle Draft→...→Published ditegakkan di backend, tidak ada endpoint yang mengizinkan skip status (DC-018, DC-019, `PRD_Document_Control_SaaS.md` §20) |
| 8 | Admin (merangkap Document Control) menjadi single point of failure/bottleneck approval | Operasional/Governance | Menengah — menghambat kecepatan proses bisnis | Menengah | Ditandai TBD operasional di `business_process.md` (SLA approval, admin backup) — mitigasi sementara: reminder otomatis; solusi permanen di Phase 2 (role Document Control terpisah) |
| 9 | Dataset training tercampur dengan operational data pengguna tanpa consent (kebocoran privasi) | Privacy/Compliance | Sangat Tinggi | Rendah (bila mitigasi diterapkan) | Pemisahan teknis folder/skema/kontrol akses antara operational data dan training data (`master_spec.md` §9.1-§9.2, `ML_Deployment_Strategy.md` §B.4) |
| 10 | Model drift pasca-deployment tidak terdeteksi tepat waktu, kualitas output menurun diam-diam | Operasional/ML | Menengah | Menengah | Monitoring drift signal (`model_output_embedding_drift_score`) dan format compliance violation rate secara berkelanjutan, trigger investigasi manual (`ML_Pipeline.md` Pipeline 15, 17) |
| 11 | Biaya GPU rental produksi membengkak tanpa auto-scaling/idle shutdown | Finansial | Menengah | Menengah | Keputusan pola scaling (24/7 vs scheduled vs auto-scaling) ditandai TBD, diputuskan berdasarkan data pola pakai aktual pasca-MVP live (`ML_Deployment_Strategy.md` §K.3) |
| 12 | Kubernetes diadopsi prematur sebelum ada kebutuhan konkret, menambah kompleksitas operasional tanpa manfaat sepadan | Teknis/Operasional | Rendah-Menengah | Rendah (bila mitigasi diterapkan) | MVP eksplisit menggunakan Docker Compose; migrasi K8s dievaluasi ulang berdasarkan trigger konkret (multi-node, autoscaling nyata), bukan "best practice" generik (`System_Architecture_and_Data_Model.md` §6.1) |

---

## 21. Project Milestones

Milestone berikut disusun selaras dengan urutan gate approval dan dependency teknis yang telah ditetapkan di `business_process.md` dan `ML_Flow_Diagram.md`, **bukan tanggal kalender pasti** (proyek dibangun bertahap sesuai ketersediaan hardware, `master_spec.md` §1):

| Milestone | Kondisi Selesai | Dependency |
|---|---|---|
| M0 — Fondasi Teknis | Struktur folder ML (`ML_Pipeline.md` §1) siap; skema database PostgreSQL inti (ERD §8 System Architecture) diimplementasikan; backend FastAPI dan frontend Next.js skeleton berjalan | Tidak ada (titik mulai) |
| M1 — Dataset Awal Siap | Dataset public+synthetic versi `dataset-v1.0` lolos Validation Pipeline dan **Approved for Training** oleh Administrator | M0; proses approval `business_process.md` §13 |
| M2 — Fine-tuning Eksperimen Pertama | Minimal 1 model kandidat (Qwen2.5-7B-Instruct) berhasil di-fine-tune (QLoRA) di Colab/Kaggle dan dievaluasi (base vs fine-tuned) | M1; kuota GPU Colab/Kaggle tersedia |
| M3 — Keputusan Model Final | Benchmarking internal berjenjang selesai untuk minimal 1-2 kandidat; model final dipilih dengan dokumentasi keputusan (bukan skor publik saja) | M2 |
| M4 — Aplikasi Inti Berfungsi | Seluruh FR berpriority Must (chat, document generation/editor/export) berjalan end-to-end terhadap model fine-tuned pertama yang **Approved for Deployment** | M3; RBAC & auth dasar (FR-001, FR-032-034) siap |
| M5 — Document Control & RAG Terintegrasi | Lifecycle Document Control 7-status aktif; RAG permission-aware berjalan di atas pgvector dengan KB awal terisi | M4; DC-001 s.d. DC-020 terimplementasi |
| M6 — Model Live Produksi (MVP) | Model melewati staging, canary, dan full rollout ke infrastruktur GPU produksi terpisah; monitoring aktif | M3, M5; infrastruktur GPU produksi tersedia |
| M7 — UAT Internal & Go-Live MVP | Seluruh Acceptance Criteria §18 terpenuhi, UAT internal selesai dengan pengguna terbatas | M4, M5, M6 |
| M8 — Observasi Pasca-Live & Iterasi Awal | Monitoring produksi berjalan ≥1 siklus observasi penuh; feedback awal terkumpul; keputusan retraining pertama (jika perlu) diambil | M7 |

---

## 22. Development Roadmap

Roadmap berikut **selaras penuh** dengan `PRD_LLM_Document_Generator.md` §32 (Product Roadmap) dan `PRD_Document_Control_SaaS.md` §29 (Future Development) — dibedakan tegas antar fase, tidak dicampur:

### 22.1 MVP (Fokus: Aplikasi Inti + Model Fine-tuned Pertama Live)
- Conversational UI lengkap, document generation/editor/export (FR-001 s.d. FR-031).
- Document Control 7-status lifecycle penuh dijalankan Administrator (DC-001 s.d. DC-020, kecuali item Should/Could seperti diff viewer DC-020 yang dapat ditunda bila kapasitas terbatas).
- RBAC dua-role, admin-managed account (FR-032 s.d. FR-034).
- RAG permission-aware di atas pgvector (FR-025 s.d. FR-028).
- Keamanan baseline penuh (§25 PRD LLM Document Generator).
- Model inference awal: hasil fine-tuning kandidat pertama yang lolos evaluasi internal minimum (bukan model base generik).
- Infrastruktur: Docker Compose, backup terjadwal, monitoring dasar.

### 22.2 Phase 2 (Fokus: Governance Lebih Dalam + Kontrol Akses Granular)
- Department-Based Access Control (DBAC) dan Resource-Level Permission granular (BR-016, BR-017).
- Role Document Control terpisah dari Administrator, termasuk pemisahan Reviewer vs Approver (segregation of duties) (BR-018, DC §29 poin 1-2).
- Evaluasi migrasi vector DB pgvector → Qdrant bila volume/kompleksitas retrieval terbukti melampaui kapasitas pgvector (dipantau via trigger skala KB, `ML_Deployment_Strategy.md` §H.6).
- Self-service password reset.
- Multi-level approval workflow untuk dokumen berdampak bisnis tinggi.
- Diff/version comparison viewer kaya (visual diff).
- Notifikasi eksternal (email) melengkapi notifikasi in-app.
- Finalisasi kebijakan retensi dokumen (setelah keputusan legal/compliance).
- Peningkatan Usage Analytics dan System Monitoring.
- Evaluasi migrasi orkestrasi Docker Compose → Kubernetes bila traksi/skala nyata membutuhkan.

### 22.3 Future (Belum Committed — Kandidat Evaluasi, Bukan Rencana Pasti)
- Reassessment voice interaction (bukan committed).
- Perluasan integrasi API eksternal dengan guardrail data governance penuh.
- Perluasan multi-model inference (routing antar model fine-tuned untuk use case berbeda).
- Eskalasi model size ke 14B+ bila hasil evaluasi 7B/8B tidak memenuhi target kualitas.
- **Tidak termasuk kapanpun**: image generation (permanen out of scope sesuai keputusan final `master_spec.md` §2); foundation model pretraining dari nol; multi-tenant SaaS komersial eksternal.

---

## 23. Team Responsibilities

Pemetaan tanggung jawab ke 8 peran yang ditetapkan di master prompt proyek, selaras dengan Actor di `master_spec.md` §5 dan `business_process.md` §4:

| Peran | Tanggung Jawab Utama | Requirement/Dokumen Terkait |
|---|---|---|
| **Project Manager (PM)** | Koordinasi milestone (§21), pengelolaan dependency (§19) dan risiko (§20), memastikan 7 dokumen turunan konsisten dengan `master_spec.md`, pelaporan progress bertahap sesuai realita hardware | Seluruh dokumen; khusus §19–§22 dokumen ini |
| **AI/ML Architect** | Strategi model (kandidat, LoRA/QLoRA, context window), arsitektur RAG, keputusan vector DB, desain evaluasi berjenjang | `ML_Deployment_Strategy.md` §A, §D, §E, §H; §7, §9 dokumen ini |
| **Data Engineer** | Pipeline ingestion-cleaning-annotation-validation-versioning dataset (Pipeline 2-6), PII detection, RAG ingestion (chunking/embedding), kualitas data | `ML_Pipeline.md` Pipeline 2-6, 13; §8 dokumen ini |
| **ML Engineer** | Eksekusi fine-tuning (Training Pipeline 7), evaluasi model (Pipeline 9), eksperimen hyperparameter, benchmarking kandidat model | `ML_Pipeline.md` Pipeline 7-9; `ML_Deployment_Strategy.md` §D-§E; §9 dokumen ini |
| **MLOps Engineer** | Experiment tracking (MLflow), model registry, CI/CD deployment, canary/rollback, monitoring produksi, retraining pipeline, cost & resource planning GPU | `ML_Pipeline.md` Pipeline 8, 10-11, 15, 17; `ML_Deployment_Strategy.md` §I-§K; §15-§16 dokumen ini |
| **Software Architect** | Arsitektur sistem (pemisahan backend-inference), skema database, API design, integrasi RAG-backend, orkestrasi deployment (Docker Compose), pemilihan stack Next.js/FastAPI | `System_Architecture_and_Data_Model.md` §1-§4, §6-§7; `PRD_LLM_Document_Generator.md` §26-§28; §6, §10 dokumen ini |
| **Security Architect** | Penegakan RBAC backend, permission-aware RAG, audit log append-only, klasifikasi dokumen, PII governance, default-deny API eksternal | `master_spec.md` §9; `System_Architecture_and_Data_Model.md` §5; §12-§13 dokumen ini |
| **Business Analyst** | Business requirement (BR-xxx), proses bisnis end-to-end, stakeholder & actor mapping, approval workflow dokumen/dataset/model, acceptance criteria bisnis | `business_process.md` keseluruhan; §3, §18 dokumen ini |

---

## 24. Requirement Traceability Matrix

**Bagian terpenting dokumen ini.** Matrix berikut menghubungkan **BR-xxx** (`business_process.md`) → **DC-xxx** (`PRD_Document_Control_SaaS.md`, bila relevan) → **FR-xxx/NFR-xxx** (`PRD_LLM_Document_Generator.md`) → ML requirement/pipeline terkait (`ML_Deployment_Strategy.md` / `ML_Pipeline.md` / `ML_Flow_Diagram.md`) → ringkasan Acceptance Criteria. **Seluruh ID yang dirujuk sudah ada di dokumen sumber — tidak ada ID baru dibuat di sini.**

| # | BR-xxx | DC-xxx (bila relevan) | FR-xxx / NFR-xxx | ML Requirement/Pipeline Terkait | Ringkasan Acceptance Criteria |
|---|---|---|---|---|---|
| 1 | BR-001 (Admin-managed account) | — | FR-001, FR-032, FR-033, FR-034 | — | Tidak ada endpoint self-registration; akun hanya dibuat Admin; akun nonaktif tidak bisa login tapi riwayat tersimpan |
| 2 | BR-002 (RBAC dasar) | §6 Permission Matrix | FR-034, NFR-005 | — | Request tanpa permission memadai → HTTP 403 terverifikasi automated test di seluruh endpoint |
| 3 | BR-003 (Conversational document generation) | DC-001, DC-018 (registrasi otomatis draft AI) | FR-002, FR-003, FR-008, FR-009 | `ML_Pipeline.md` Pipeline 12 (Inference), Pipeline 14 (RAG Retrieval) | Dokumen baru muncul berstatus Draft dengan Document ID unik ≤2 detik; version 1 tercatat |
| 4 | BR-004 (Export dokumen) | DC §13 Distribution (kontekstual) | FR-031 | — | File DOCX/PDF terbuka tanpa corrupt, struktur identik editor |
| 5 | BR-005 (Human review wajib) | DC-004, DC-005, DC-006 | FR-037 | — | Tidak ada jalur Draft/Under Review → Published tanpa melalui Approved |
| 6 | BR-006 (Document Control oleh Admin, MVP) | §4-§5, §11-§12, §26 (Workflow transisi) | FR-037, §24 Admin Dashboard | — | Hanya Administrator yang dapat melakukan approve/publish/archive/reopen (diverifikasi test akses) |
| 7 | BR-007 (Klasifikasi dokumen) | DC-009 | FR-022 | `ML_Pipeline.md` §13.7 (reject ingestion tanpa klasifikasi/ACL) | Dokumen tanpa klasifikasi tidak dapat disubmit ke Under Review atau masuk index RAG |
| 8 | BR-008 (RAG permission-aware) | DC-010, DC-017, DC-019; SEC-DC-03 | FR-027, FR-028, NFR-005 | `ML_Deployment_Strategy.md` §H.8; `ML_Pipeline.md` Pipeline 13-14 (ACL predicate query) | Uji 2 user beda permission terhadap entri KB sama → hasil retrieval berbeda terbukti; negative test ACL nol hasil |
| 9 | BR-009 (Approval dataset training) | — | NFR-006 | `ML_Deployment_Strategy.md` §B.6; `ML_Flow_Diagram.md` Diagram 2-3; `ML_Pipeline.md` Pipeline 2-6 (gate Approved Dataset) | Setiap dataset dipakai training punya status "Approved for Training" dengan approver & timestamp auditable |
| 10 | BR-010 (Approval model deployment) | — | NFR-006 | `ML_Deployment_Strategy.md` §I; `ML_Flow_Diagram.md` Diagram 6-7; `ML_Pipeline.md` Pipeline 10-11 (approval gate Staging→Production) | Model live produksi punya status "Approved for Deployment", riwayat staging, approver tercatat; tidak ada model lompat None→Production |
| 11 | BR-011 (Audit log akses sensitif) | DC-015; SEC-DC-04, SEC-DC-07 | FR-036, NFR-008 | `ML_Pipeline.md` §14.3 (log retrieval RAG per akses) | 100% aksi sensitif tercatat append-only; tidak ada endpoint update/delete tabel audit log |
| 12 | BR-012 (Larangan default API eksternal) | SEC-DC-06 | NFR-007 | `ML_Deployment_Strategy.md` §A.8 (self-hosted default) | Percobaan kirim dokumen Confidential/Restricted ke integrasi eksternal gagal tanpa approval policy tercatat |
| 13 | BR-013 (Versioning dokumen) | DC-011, DC-020 | FR-019, FR-020 | — | Seluruh versi historis dapat ditelusuri dan dibandingkan; restore menambah entri versi baru |
| 14 | BR-014 (Template dokumen) | — | FR-009, FR-029, FR-030 | — | Template nonaktif tidak muncul sebagai pilihan baru tapi dokumen existing tetap valid |
| 15 | BR-015 (Notifikasi status approval) | DC-013 | — | — | Notifikasi muncul ≤1 menit dari perubahan status di dalam aplikasi |
| 16 | BR-016 (DBAC — Phase 2, TBD) | §5, §6 (catatan Phase 2), §29 poin 3 | — (Phase 2) | — | Dideferred ke Phase 2 — skema `department_id` sudah disediakan nullable sejak MVP tanpa migrasi destruktif |
| 17 | BR-017 (Resource-Level Permission — Phase 2, TBD) | §29 poin 4 | — (Phase 2) | — | Dideferred ke Phase 2 |
| 18 | BR-018 (Role Document Control terpisah — Phase 2, TBD) | §5, §29 poin 1-2 | — (Phase 2) | — | Dideferred ke Phase 2 — MVP eksplisit menjalankan fungsi via Admin, divalidasi lewat RBAC permission check bukan hardcode |

**Baris tambahan — Document Control detail (DC-xxx) yang tidak secara langsung dipetakan ke satu BR spesifik di atas, dikelompokkan di bawah payung BR-005/BR-006 (Document Control & Human Review):**

| # | BR-xxx (payung) | DC-xxx | FR-xxx terkait | ML Requirement/Pipeline Terkait | Ringkasan Acceptance Criteria |
|---|---|---|---|---|---|
| 19 | BR-006 | DC-002 (Edit Draft) | FR-018 (autosave) | — | Setiap simpan menghasilkan versi baru tercatat timestamp |
| 20 | BR-006 | DC-003 (Submit Review) | FR-037 | — | Status berubah ke Under Review dalam satu transaksi, tercatat audit log |
| 21 | BR-005 | DC-007 (Reject) | FR-037 | — | Author menerima notifikasi alasan; dokumen tidak dapat diedit tanpa reopen eksplisit |
| 22 | BR-006 | DC-008 (Reopen Rejected→Draft) | — | — | Transisi tercatat audit log dengan referensi record Rejected sebelumnya |
| 23 | BR-013 | DC-014 (Archiving) | — | — | Dokumen Archived tidak muncul pencarian Reader, tetap dapat diakses Admin |
| 24 | BR-007 (privasi/retensi) | DC-016 (Retention field) | — | — | Field retensi fungsional kosong/TBD tanpa error; dapat diisi kapan pun tanpa migrasi skema |

**Baris tambahan — Technical Objectives (T1–T7, `PRD_LLM_Document_Generator.md` §5) yang mendasari NFR tanpa BR spesifik, untuk kelengkapan cakupan:**

| # | Technical Objective | NFR-xxx terkait | ML/Arsitektur Terkait | Ringkasan Acceptance Criteria |
|---|---|---|---|---|
| 25 | T1 (Conversational UI responsif) | NFR-001, NFR-009 | — | First token ≤3 detik beban normal; 80% pengguna uji selesaikan Journey A tanpa bantuan |
| 26 | T2 (Pemisahan backend-inference) | NFR-012 | `System_Architecture_and_Data_Model.md` §2.2; `ML_Pipeline.md` Pipeline 12 | Penggantian model inference tidak memerlukan perubahan endpoint chat/document API |
| 27 | T3 (RAG permission-aware di backend) | NFR-005 | `ML_Deployment_Strategy.md` §H.8 | Sama dengan baris #8 di atas (BR-008) |
| 28 | T4 (RBAC API layer) | NFR-005 | `System_Architecture_and_Data_Model.md` §5.2 | Sama dengan baris #2 di atas (BR-002) |
| 29 | T5 (Document editor terstruktur) | FR-012 s.d. FR-017 | — | Heading/list/tabel/undo-redo berfungsi konsisten di editor dan ekspor |
| 30 | T6 (Audit trail) | NFR-008, NFR-DC-04 | — | Sama dengan baris #11 di atas (BR-011) |
| 31 | T7 (Skema Phase 2-ready) | — | `System_Architecture_and_Data_Model.md` §8.1 (kolom `department_id` nullable) | Skema tidak butuh migrasi destruktif saat Phase 2 mengaktifkan DBAC |

**Catatan keterbacaan matrix:**
- Baris 1–18 adalah mapping utama per Business Requirement (seluruh 18 BR tercakup).
- Baris 19–24 melengkapi detail Document Control (DC-002, DC-003, DC-007, DC-008, DC-014, DC-016) yang secara substansi berada di bawah payung governance BR-005/BR-006/BR-007/BR-013, memastikan seluruh 20 DC (DC-001 s.d. DC-020) tertelusur baik secara langsung (baris 1-18: DC-001, DC-004 s.d. DC-006, DC-009 s.d. DC-011, DC-013, DC-015, DC-017 s.d. DC-020) maupun di baris tambahan (DC-002, DC-003, DC-007, DC-008, DC-014, DC-016) — total 20 DC seluruhnya tercakup.
- Baris 25–31 melengkapi NFR dan Technical Objective yang mendasari kualitas sistem secara umum, tidak terikat satu BR spesifik namun tetap merujuk ID resmi di dokumen sumber.
- Total baris traceability: **31 baris**, mencakup 18 BR, 20 DC, seluruh FR kategori utama (FR-001 s.d. FR-037 tersentuh baik langsung maupun melalui kelompok §4), 13 NFR + 8 NFR-DC, dan seluruh pipeline ML kunci (dataset approval, model approval, RAG permission-aware, deployment governance).

---

**Penutup:** Dokumen ini (Document 7 dari 7) menutup rangkaian dokumentasi requirement DocuGen AI. Seluruh keputusan teknis, bisnis, keamanan, dan ML yang dirujuk di sini bersumber dari `master_spec.md` dan 5 dokumen turunan (Document 1–6), ditambah dokumen referensi `System_Architecture_and_Data_Model.md`. Perubahan terhadap requirement apa pun **wajib** dilakukan di dokumen sumber aslinya, kemudian direfleksikan ulang di Traceability Matrix §24 — dokumen ini tidak pernah menjadi sumber kebenaran baru yang berdiri sendiri.
