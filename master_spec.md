# MASTER PROJECT SPECIFICATION — DocuGen AI (LLM Document Generator)

Status: Phase 2 output. Ini adalah SINGLE SOURCE OF TRUTH yang WAJIB dipatuhi oleh seluruh 7 dokumen turunan. Jangan kontradiksi dokumen ini. Jika sebuah dokumen turunan butuh detail lebih dalam, detail itu harus konsisten dengan keputusan di sini.

## 1. Product Identity
- Nama kerja: **DocuGen AI**.
- Tipe produk: Private Enterprise AI Application, closed deployment, no public registration, admin-managed accounts only.
- Status proyek: RIIL — akan dibangun bertahap sesuai hardware yang tersedia (bukan sekadar blueprint portfolio).

## 2. Product Vision & Scope (tidak boleh diubah tanpa justifikasi kuat)
General-purpose conversational LLM application dengan **document generation sebagai kapabilitas utama yang dioptimalkan**, bukan wrapper API eksternal murni. Dua komponen wajib terintegrasi dalam satu ekosistem:
- **A. LLM Model Development**: foundation model selection → dataset engineering → SFT (LoRA/QLoRA evaluasi) → evaluation → optimization → serving → MLOps → deployment.
- **B. LLM Application Development**: conversational UI, document generation/understanding/editor/management, knowledge base + RAG, user & RBAC management, document export, AI monitoring.

Out of scope MVP: image generation (permanen out of scope), voice interaction (out of scope MVP, boleh di-reassess Phase 2+).

## 3. Fine-tuning vs RAG — pembedaan fungsi (wajib dijelaskan identik di semua dokumen terkait)
- **Fine-tuning (SFT + LoRA/QLoRA)**: mengubah *behavior* model — instruction-following, gaya penulisan dokumen, konsistensi struktur (heading/numbering/format baku perusahaan), kepatuhan terhadap format Bahasa Indonesia formal. Bersifat statis sampai retraining berikutnya; TIDAK untuk menyimpan fakta/dokumen yang sering berubah.
- **RAG**: menyuntikkan *knowledge* yang dinamis dan sering berubah (SOP, kebijakan, dokumen referensi terkini) saat inference, melalui retrieval dari vector DB yang permission-aware. Knowledge base selalu up-to-date tanpa retraining.
- **Aturan tegas**: dokumen perusahaan yang terus berubah → RAG. Fine-tuning tidak pernah dipakai sebagai pengganti knowledge base dinamis. Ini adalah keputusan final, bukan rekomendasi.

## 4. Keputusan Proyek Final (Decision Log)
| Parameter | Keputusan | Sumber |
|---|---|---|
| Product type | Private Enterprise AI Application | Prompt §3 |
| AI capability | General-purpose LLM, document generation sebagai spesialisasi utama | Prompt §3 |
| Model strategy | Fine-tuning pretrained open-weight model (bukan train from scratch) | Prompt §3 |
| Fine-tuning method | Evaluasi LoRA vs QLoRA berbasis benchmark, bukan diasumsikan | Prompt §3 |
| Knowledge integration | RAG wajib untuk dokumen dinamis | Prompt §3 |
| UI paradigm | ChatGPT-like conversational UI | Prompt §3 |
| Target users | Admin dan Authorized User saja, admin-managed account | Prompt §4 |
| Authorization | RBAC (MVP) → Department-Based Access Control + Resource-Level Permission (Phase 2) | Prompt §4, klarifikasi |
| Document Control role | **Admin merangkap fungsi Document Control di MVP**; role/department terpisah di Phase 2 | Klarifikasi user |
| Main output | Chat preview + export DOCX/PDF | Prompt §3 |
| Human review | Wajib untuk dokumen bisnis penting; AI output BUKAN otomatis dokumen resmi | Prompt §3, §5 Doc2 |
| Training from scratch | Bukan tujuan MVP maupun Phase 2 | Prompt §3 |
| External AI API | Tidak diasumsikan; model self-hosted adalah default; API eksternal hanya opsi evaluatif bertanda TBD dengan guardrail data governance | Prompt §3 |
| Backend language | **Python FastAPI** untuk backend aplikasi, terpisah dari inference service | Klarifikasi user |
| Cloud GPU approach | **Free/low-cost tier dulu (Google Colab Pro / Kaggle)** untuk eksperimen awal → GPU rental on-demand (RunPod/Vast.ai/Lambda) saat perlu scale | Klarifikasi user |
| Dataset awal | Tidak ada dokumen perusahaan riil tersedia saat ini → dataset awal public + synthetic; pipeline tetap dirancang untuk menyerap dokumen historis perusahaan begitu tersedia | Klarifikasi user |
| Database | PostgreSQL | Prompt §8 |
| Vector DB | Dievaluasi di ML Deployment Strategy: pgvector (default MVP, minim moving parts) vs Qdrant (Phase 2 bila butuh scale/fitur retrieval lanjutan) — ditandai sebagai rekomendasi, konfirmasi final ada di ML Deployment Strategy §A disertai justifikasi | Prompt §8 |
| Frontend | Next.js (React-based, SSR untuk performa awal, ekosistem komponen chat UI matang) | Evaluasi arsitektur |
| Image/voice generation | Out of scope (image: permanen; voice: MVP) | Prompt §2.4 |

## 5. Actors
1. **Administrator** — user/role/permission/department management, dataset management, model management, knowledge base management, template management, audit log, usage analytics, AI configuration, system monitoring, (MVP) document control/approval.
2. **Authorized User** — login, conversational AI, document generation/editing/upload, conversation & document history, export, template usage, KB access sesuai permission yang diberikan admin.
3. **AI/ML Team** (internal engineering, bukan role aplikasi) — dataset curation, fine-tuning, evaluation, model deployment approval flow.
4. **System/AI Model** — aktor non-manusia: menghasilkan draft, tidak punya otoritas approval apa pun.

## 6. System Boundaries
- Di dalam scope: chat app, document generation/editor/export, RAG atas internal KB, RBAC, admin dashboard, model fine-tuning pipeline, model serving self-hosted, MLOps dasar (tracking, registry, monitoring).
- Di luar scope MVP: image gen, voice, department-based ACL granular, resource-level ACL granular, multi-tenant SaaS komersial ke pelanggan eksternal, foundation model pretraining dari nol.
- Document Control sebagai **role terpisah** adalah Phase 2; MVP: fungsi approval ada tapi dijalankan oleh Admin.

## 7. Hardware Reality (wajib direfleksikan di ML Deployment Strategy & Project Requirements)
- Local dev machine: AMD Ryzen 3 3250U (2C/4T), 8GB RAM, integrated Radeon graphics, no dedicated GPU/CUDA.
- Implikasi: local dev hanya untuk dataset prep/cleaning/annotation (CPU-bound, dataset kecil-menengah), backend/frontend dev, DB dev, code untuk training/evaluation scripts (tanpa eksekusi penuh), dan inference *hanya* model sangat kecil terkuantisasi (misal <1B/CPU) untuk smoke test pipeline — bukan untuk benchmarking model kandidat 7B/8B.
- Semua benchmarking model kandidat, fine-tuning (LoRA/QLoRA), dan evaluasi GPU-berat **wajib** di Google Colab/Kaggle (free/low-cost tier dulu) lalu GPU rental (RunPod dkk.) bila perlu scale/durasi lebih panjang.
- Production inference GPU: disediakan terpisah sebagai infrastruktur produksi (cloud GPU rental persisten atau on-prem GPU bila sudah ada traksi) — detail sizing di ML Deployment Strategy §J/K.

## 8. Riset Teknis Terverifikasi (dipakai di ML Deployment Strategy, jangan dikarang ulang)
### 8.1 Kandidat model open-weight (per riset web, retrieved Nov 2026)
| Model | License | Context | Catatan Benchmark (sumber: Qwen2.5 Technical Report, HF model cards) |
|---|---|---|---|
| Qwen2.5-7B-Instruct | Apache 2.0 | 128K | Unggul di MMLU-Pro (56.3 vs Llama3.1-8B 48.3), coding (HumanEval 84.8), math (GSM8K 91.6); IFEval strict-prompt **lebih rendah** dari Llama3.1-8B (71.2 vs 75.9); IndoMMLU 56.4 (indikasi performa Bahasa Indonesia lebih baik dari Llama3.1-8B 53.9 dan Mistral-7B tidak diukur di tabel ini). |
| Llama-3.1-8B-Instruct | Llama 3.1 Community License (bukan Apache/MIT — ada batasan penggunaan komersial di skala besar & larangan tertentu, cek ketentuan lisensi resmi Meta sebelum deploy produksi) | 128K | IFEval strict-prompt tertinggi di kelas 7-9B (75.9); kalah di coding/math vs Qwen2.5-7B. |
| Mistral-7B-Instruct-v0.3 | Apache 2.0 | — | Lebih lemah di MMLU/HumanEval dibanding Qwen2.5-7B pada tabel base model (MMLU 64.2 vs 74.2); belum ada data IndoMMLU langsung dari sumber yang ditemukan — **TBD, perlu benchmark internal**. |

Catatan jujur: ini BUKAN benchmark internal proyek — ini kompilasi dari laporan teknis publik (Qwen2.5 Technical Report arXiv:2412.15115, HF model cards). Keputusan final model harus melalui benchmarking internal terhadap representative document-generation prompts (lihat ML Deployment Strategy §A). Qwen2.5-7B-Instruct direkomendasikan sebagai **kandidat utama untuk diuji pertama** karena kombinasi lisensi Apache 2.0 paling permisif + performa Indonesia (IndoMMLU) kompetitif + context length 128K, TAPI ini rekomendasi awal untuk urutan eksperimen, bukan keputusan final model.

### 8.2 VRAM requirement (sumber: Spheron Network GPU VRAM guide 2026, io.net fine-tuning guide, retrieved Nov 2026)
| Model size | Full FT | LoRA (16-bit) | QLoRA (4-bit) | GPU minimum QLoRA |
|---|---|---|---|---|
| 7B/8B | ~88 GB | ~19-20 GB | ~8-14 GB | 1x GPU 12-24GB (RTX 3090/4090/A100) |
| 14B | ~174 GB | ~35 GB | ~14 GB | 1x GPU 24GB+ |
| 32B | ~394 GB | ~76 GB | ~28 GB | 1x GPU 32GB+ |
| 70B/72B | ~860 GB | ~159 GB | ~52 GB | 1x A100 80GB |

Implikasi: untuk kandidat 7B/8B, QLoRA pada 1x GPU 12-24GB (RTX 3090/4090, atau A100 40GB) adalah jalur realistis pertama — align dengan keputusan "Colab/Kaggle dulu, lalu GPU rental". Full fine-tuning 7B (butuh ~88GB) TIDAK realistis untuk MVP — tidak direkomendasikan.

### 8.3 Harga GPU rental (sumber: RunPod pricing page, UsagePricing.com, Northflank blog, retrieved Nov 2026 — harga berubah, selalu cek ulang sebelum eksekusi budget riil)
| GPU | VRAM | RunPod Secure Cloud | RunPod Community Cloud |
|---|---|---|---|
| RTX 3090 | 24GB | $0.50/hr | $0.22/hr |
| RTX 4090 | 24GB | $0.74/hr | $0.34/hr |
| A100 PCIe | 80GB | $1.59/hr | $1.19/hr |
| A100 SXM | 80GB | $1.59/hr | $1.39/hr |
| H100 PCIe | 80GB | $2.89/hr | $1.99/hr |

Google Colab Pro (~$9.99/bulan) dan Kaggle (gratis, GPU T4/P100 dengan kuota jam mingguan terbatas) adalah titik masuk eksperimen awal sebelum commit ke GPU rental berbayar per jam. Angka-angka ini harus di-recheck di halaman resmi provider sebelum dipakai sebagai dasar budget riil karena harga cloud GPU berubah cepat.

## 9. Security & Data Governance Baseline (berlaku di semua dokumen)
1. Percakapan user tidak otomatis dipakai untuk training.
2. Dokumen upload tidak otomatis menjadi training data.
3. Dataset training butuh approval eksplisit (lihat Business Process §Model Training Approval).
4. RAG retrieval WAJIB permission-aware (filter berbasis ACL dokumen sebelum masuk context, bukan hanya system prompt).
5. Dokumen sensitif wajib punya klasifikasi (misal Public/Internal/Confidential/Restricted).
6. Semua akses sensitif Admin tercatat di audit log — tidak ada akses tanpa jejak.
7. Data internal tidak boleh dikirim ke API eksternal pihak ketiga tanpa persetujuan kebijakan perusahaan (default: deny, override butuh approval formal).
8. Authorization ditegakkan di backend (API layer), bukan hanya UI/system prompt. System prompt TIDAK PERNAH jadi pengganti backend authorization.

## 10. Dokumen Turunan & Traceability
7 dokumen wajib saling merujuk ID/section berikut ini sebagai anchor, bukan deskripsi lepas:
1. `business_process.md` — proses bisnis, actor, approval, dataset/model governance.
2. `PRD_Document_Control_SaaS.md` — lifecycle status dokumen (Draft→...→Archived) yang DIRUJUK oleh PRD LLM Document Generator saat AI output masuk workflow.
3. `PRD_LLM_Document_Generator.md` — FR-xxx/NFR-xxx requirement aplikasi, merujuk status lifecycle dari Document 2.
4. `ML_Deployment_Strategy.md` — merujuk Decision Log §4, riset §8, hardware §7.
5. `ML_Flow_Diagram.md` — visual dari proses yang dijelaskan di ML Deployment Strategy, level "apa yang terjadi".
6. `ML_Pipeline.md` — implementasi teknis dari flow di Document 5, level "bagaimana caranya" (tools, config, folder structure).
7. `Project_Requirements.md` — dokumen induk, Requirement Traceability Matrix menghubungkan ID requirement dari Document 1,2,3,4.

Semua requirement ID: FR-xxx (Functional), NFR-xxx (Non-Functional) untuk PRD LLM Document Generator; BR-xxx untuk Business Requirements di Document 1; DC-xxx untuk Document Control requirement di Document 2. Jangan menduplikasi ID across dokumen — Project Requirements MERUJUK ID yang sama, tidak membuat ID baru untuk requirement yang sudah ada.
