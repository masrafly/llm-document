# ML PIPELINE — DocuGen AI (Implementasi Teknis)

Status: Dokumen turunan #6 dari 7. Level pembahasan: **"bagaimana caranya"** — tools konkret, config YAML/JSON nyata, struktur folder, error handling, monitoring per pipeline. Dokumen ini TIDAK mengulang diagram alur besar (lihat `ML_Flow_Diagram.md` untuk level "apa yang terjadi"); dokumen ini adalah spesifikasi implementasi yang siap dieksekusi oleh AI/ML Team.

Referensi wajib (anchor, lihat `master_spec.md`):
- Decision Log §4 (PostgreSQL, vector DB pgvector default MVP/Qdrant Phase 2, FastAPI backend, Next.js frontend).
- Hardware Reality §7 (dev lokal CPU-only 8GB RAM; training/eval GPU-berat wajib di Colab/Kaggle → GPU rental RunPod/Vast.ai/Lambda).
- Riset Teknis §8 (kandidat model Qwen2.5-7B-Instruct sebagai kandidat utama uji pertama, VRAM requirement QLoRA 7B ~8-14GB, harga GPU rental).
- Security & Data Governance Baseline §9 (tidak ada data training otomatis dari percakapan/upload, approval eksplisit, RAG permission-aware, audit log, deny-by-default API eksternal).
- Fine-tuning vs RAG §3 (fine-tuning = behavior statis; RAG = knowledge dinamis; tidak boleh ditukar perannya).

Stack Python wajib (sesuai master_spec §Decision Log — Python training stack): **HF Transformers + PEFT (LoRA/QLoRA) + TRL (SFTTrainer) + MLflow (tracking & registry)**.

---

## 1. Pipeline Architecture

### 1.1 Objective
Menyediakan struktur proyek ML yang konsisten, reproducible, dan dapat dieksekusi lintas environment (dev lokal CPU-only → Colab/Kaggle GPU gratis → GPU rental) tanpa perubahan kode, hanya perubahan config.

### 1.2 Prinsip Desain
1. **Config-driven**: setiap pipeline dikendalikan oleh file YAML di `configs/`, bukan hardcode di script.
2. **Environment-agnostic**: script sama jalan di CPU (smoke test, dataset kecil) maupun GPU (training/eval penuh); device dan precision ditentukan oleh config.
3. **Separation of concerns**: data, kode, eksperimen, model, dan script orchestration terpisah folder agar bisa diversikan independen (DVC untuk data/model, Git untuk kode).
4. **Jejak audit penuh**: setiap run tercatat di MLflow dan setiap dataset/model bertanda versi (lihat Pipeline 6 & 8).

### 1.3 Struktur Folder Proyek ML Lengkap

```
docugen-ai-ml/
├── README.md
├── pyproject.toml                  # dependency pin: transformers, peft, trl, mlflow, dvc, bitsandbytes, etc.
├── .dvc/                            # DVC metadata (remote storage pointer)
├── .dvcignore
├── .gitignore                      # data/raw, data/processed, models/*.safetensors di-ignore git, dikelola DVC
│
├── configs/
│   ├── data/
│   │   ├── ingestion.yaml
│   │   ├── cleaning.yaml
│   │   └── validation_schema.yaml
│   ├── training/
│   │   ├── qlora_qwen2.5_7b_run01.yaml
│   │   └── qlora_qwen2.5_7b_run02.yaml
│   ├── eval/
│   │   └── eval_suite_v1.yaml
│   ├── rag/
│   │   ├── ingestion_chunking.yaml
│   │   └── retrieval.yaml
│   └── deployment/
│       └── vllm_serving.yaml
│
├── data/
│   ├── raw/                        # hasil Data Ingestion, immutable, versi via DVC
│   │   ├── public_corpus/
│   │   └── synthetic_generated/
│   ├── interim/                    # hasil Data Cleaning (sebelum anotasi)
│   ├── processed/                  # hasil Annotation + Validation, siap training (format JSONL chat)
│   │   ├── train.jsonl
│   │   ├── val.jsonl
│   │   └── test.jsonl
│   └── candidates/                 # Feedback Pipeline: kandidat dataset dari user feedback, BELUM approved
│       └── feedback_candidates_2026-11.jsonl
│
├── src/
│   └── docugen_ml/
│       ├── __init__.py
│       ├── ingestion/
│       │   ├── scrapers.py
│       │   └── synthetic_generator.py
│       ├── cleaning/
│       │   ├── dedup.py
│       │   ├── pii_redaction.py
│       │   └── normalizer.py
│       ├── annotation/
│       │   └── label_studio_export_parser.py
│       ├── validation/
│       │   └── schema_validator.py          # Pydantic + Great Expectations
│       ├── training/
│       │   ├── train_qlora.py               # entrypoint: HF Transformers + PEFT + TRL SFTTrainer
│       │   └── dataset_formatter.py
│       ├── evaluation/
│       │   ├── run_eval.py
│       │   └── metrics.py                   # ROUGE, BLEU, LLM-as-judge, format compliance
│       ├── rag/
│       │   ├── chunker.py
│       │   ├── embedder.py
│       │   └── retriever.py
│       ├── serving/
│       │   └── vllm_client.py
│       └── monitoring/
│           └── exporters.py                 # Prometheus custom exporter
│
├── scripts/
│   ├── run_ingestion.sh
│   ├── run_cleaning.sh
│   ├── run_training_colab.ipynb             # notebook untuk eksekusi di Colab/Kaggle (GPU)
│   ├── run_eval.sh
│   ├── register_model.py                    # push ke MLflow Model Registry
│   └── promote_model.py                     # Staging -> Production (dengan approval gate manual)
│
├── experiments/
│   └── mlruns/                     # MLflow local tracking store (atau remote tracking URI, lihat Pipeline 8)
│
├── models/
│   ├── adapters/                   # LoRA/QLoRA adapter weights (kecil, ratusan MB), versi via DVC/Git LFS
│   │   └── qwen2.5-7b-docugen-v0.1/
│   └── merged/                     # (opsional) merged full weight untuk deployment, via DVC saja (besar)
│
├── tests/
│   ├── test_data_validation.py
│   ├── test_rag_chunking.py
│   └── test_eval_metrics.py
│
└── .github/workflows/
    └── ml_ci.yml                   # lint + unit test + schema validation (CPU-only, bukan training)
```

### 1.4 Tools
Python 3.11, Poetry/`pyproject.toml`, DVC (remote: S3-compatible/GCS bucket), Git + Git LFS (alternatif ringan untuk adapter kecil), MLflow, HF Transformers/PEFT/TRL/bitsandbytes, Great Expectations, Pydantic, GitHub Actions.

### 1.5 Dependencies
Tidak ada dependency upstream (pipeline paling dasar). Seluruh pipeline 2–17 mengikuti struktur folder ini.

### 1.6 Error Handling
- `pyproject.toml` dengan version pinning ketat (bukan `>=`) untuk mencegah drift dependency antara dev lokal, Colab, dan GPU rental.
- CI (`ml_ci.yml`) gagal build jika ada import path yang menyimpang dari struktur `src/docugen_ml/`.

### 1.7 Validation
Review arsitektur folder oleh AI/ML Team lead sebelum pipeline data pertama dijalankan; checklist: apakah data besar sudah di-`.gitignore` dan diarahkan ke DVC, apakah config dan kode terpisah bersih.

### 1.8 Monitoring
Tidak applicable langsung (bukan pipeline runtime) — dipantau secara struktural lewat CI lint/structure check di setiap PR.

---

## 2. Data Ingestion Pipeline

### 2.1 Objective
Mengumpulkan data mentah dari sumber publik dan sintetik (sesuai master_spec §4 — dataset awal tidak ada dokumen perusahaan riil) ke `data/raw/`, dengan desain yang siap menyerap dokumen historis perusahaan begitu tersedia.

### 2.2 Input
- Sumber publik: dataset dokumen Bahasa Indonesia (misal corpus berita, dokumen pemerintah terbuka, template surat resmi publik), lisensi harus jelas dan dicatat.
- Sumber sintetik: prompt-generated document examples (surat, laporan, memo) via LLM generator (self-hosted atau API eksternal dengan guardrail §9.7 — hanya untuk generasi data **sintetik non-sensitif**, tidak mengirim data internal).
- (Masa depan) Dokumen historis perusahaan — masuk lewat jalur upload admin dengan proses approval terpisah, TIDAK melalui scraper otomatis.

### 2.3 Processing Steps
1. Definisikan sumber di `configs/data/ingestion.yaml` (lihat contoh config §2.9).
2. Jalankan scraper/loader per sumber (`src/docugen_ml/ingestion/scrapers.py`) → simpan mentah apa adanya ke `data/raw/public_corpus/<source_name>/<date>/`.
3. Jalankan `synthetic_generator.py` untuk menghasilkan N dokumen sintetik per kategori (surat resmi, laporan, memo, SOP dummy) dengan template prompt terkontrol.
4. Catat metadata sumber (URL/API, tanggal ambil, lisensi, jumlah record) ke `data/raw/<source>/manifest.json`.
5. Commit hasil ke DVC (`dvc add data/raw/... && dvc push`).

### 2.4 Tools
`requests`/`httpx` untuk scraping sumber terbuka, HF `datasets` library untuk load dataset publik dari Hugging Face Hub, LLM API (internal self-hosted model atau API eksternal berlabel TBD per master_spec §4) untuk generasi sintetik, DVC untuk versioning.

### 2.5 Output
`data/raw/public_corpus/**`, `data/raw/synthetic_generated/**`, masing-masing dengan `manifest.json` (jumlah dokumen, sumber, lisensi, checksum).

### 2.6 Dependencies
Upstream: Pipeline Architecture (1). Downstream: Data Cleaning (3).

### 2.7 Error Handling
- Retry otomatis (exponential backoff, max 3x) untuk scraping sumber web yang gagal/timeout.
- Validasi lisensi wajib: jika field `license` di manifest kosong/tidak jelas, dokumen ditandai `quarantine` dan tidak lanjut ke Cleaning sampai ditinjau manual.
- Checksum (SHA-256) disimpan per file untuk deteksi korupsi saat transfer antar environment (lokal ↔ Colab ↔ GPU rental).

### 2.8 Validation
Great Expectations suite dasar: jumlah file > 0, tidak ada file kosong (0 byte), encoding UTF-8 valid.

### 2.9 Contoh Config — `configs/data/ingestion.yaml`
```yaml
ingestion:
  run_id: "ingest-2026-11-15"
  sources:
    - name: "public_gov_documents_id"
      type: "huggingface_dataset"
      dataset_id: "indonlp/indo-gov-documents"  # contoh ilustratif
      license: "CC-BY-4.0"
      split: "train"
      max_records: 5000
    - name: "synthetic_formal_letters"
      type: "synthetic_generation"
      generator_model: "self_hosted_qwen2.5_7b_base"
      categories: ["surat_resmi", "laporan_internal", "memo", "sop_template"]
      records_per_category: 1500
      prompt_template_path: "configs/data/synthetic_prompts/formal_doc_id.jinja"
  output_dir: "data/raw"
  manifest_required_fields: ["source_name", "license", "retrieved_at", "record_count", "sha256"]
  quarantine_on_missing_license: true
```

### 2.10 Monitoring
Metrik run: `ingestion_records_total{source=...}`, `ingestion_failures_total{source=...}`, `ingestion_duration_seconds`. Dikirim ke log terstruktur (JSON) yang nantinya diagregasi di Pipeline 15 (Monitoring).

---

## 3. Data Cleaning Pipeline

### 3.1 Objective
Mengubah data mentah menjadi data bersih dan konsisten, bebas duplikasi dan PII yang tidak relevan, siap untuk anotasi.

### 3.2 Input
`data/raw/**` (output Pipeline 2).

### 3.3 Processing Steps
1. **Deduplication**: exact match (hash) + near-duplicate (MinHash/SimHash) via `dedup.py`.
2. **Normalisasi teks**: unicode normalization (NFKC), penghapusan HTML/markup sisa scraping, standardisasi whitespace.
3. **PII redaction**: deteksi & masking NIK, nomor telepon, email pribadi pada data publik/sintetik yang tidak sengaja mengandungnya (regex + NER model `id`), sebelum digunakan sebagai contoh training — selaras dengan baseline governance §9 meski ini bukan dokumen internal perusahaan.
4. **Filtering kualitas**: buang dokumen terlalu pendek (<50 token) atau dominan non-Bahasa Indonesia (langdetect threshold).
5. Simpan hasil ke `data/interim/cleaned/<batch_id>.jsonl` dengan skema `{doc_id, source, text, lang, length_tokens}`.

### 3.4 Tools
`src/docugen_ml/cleaning/*.py` — `datasketch` (MinHash), `regex`, `unicodedata`, `langdetect`, model NER ringan (`flair`/`spacy` id) untuk PII. Semua CPU-bound, cocok dijalankan di dev lokal (sesuai hardware §7 — tahap ini memang didesain untuk CPU 8GB RAM).

### 3.5 Output
`data/interim/cleaned/*.jsonl` + `cleaning_report.json` (jumlah duplikat dibuang, jumlah PII dimask, jumlah dibuang karena kualitas).

### 3.6 Dependencies
Upstream: Data Ingestion (2). Downstream: Dataset Annotation (4).

### 3.7 Error Handling
- Jika >30% dokumen suatu batch gagal lolos filter kualitas, pipeline berhenti (fail-fast) dan memberi alert — indikasi sumber data bermasalah, bukan di-skip diam-diam.
- Dokumen yang gagal PII redaction (error parsing) ditandai `needs_manual_review`, tidak dibuang otomatis.

### 3.8 Validation
Great Expectations: tidak ada duplikat exact-hash tersisa, tidak ada baris JSONL invalid, distribusi panjang token dalam rentang wajar (log histogram ke report).

### 3.9 Monitoring
`cleaning_dedup_ratio`, `cleaning_pii_redacted_count`, `cleaning_rejected_ratio`, `cleaning_duration_seconds`.

---

## 4. Dataset Annotation Pipeline

### 4.1 Objective
Mengubah data bersih menjadi dataset instruction-tuning berformat chat (`system/user/assistant`) sesuai gaya penulisan dokumen formal Bahasa Indonesia yang ingin diajarkan ke model (selaras master_spec §3 — fine-tuning mengubah *behavior*, bukan menyuntik fakta).

### 4.2 Input
`data/interim/cleaned/*.jsonl`.

### 4.3 Processing Steps
1. Import batch ke **Label Studio** (self-hosted) dengan template anotasi kustom: anotator menulis instruksi (`instruction`) yang relevan untuk setiap dokumen bersih (misal "Buatkan surat permohonan cuti dengan format baku perusahaan") dan menandai `document_type`, `format_compliance_notes`.
2. Anotator/AI-assisted pre-labeling: gunakan model LLM yang ada untuk draft instruksi awal, anotator manusia mengoreksi (human-in-the-loop, bukan full-auto).
3. Export hasil anotasi Label Studio (JSON) → parser `label_studio_export_parser.py` → format chat JSONL: `{"messages": [{"role":"system","content":...},{"role":"user","content":...},{"role":"assistant","content":...}]}`.
4. Dual-review minimal untuk 10% sample (quality audit) oleh anotator kedua.

### 4.4 Tools
Label Studio (self-hosted, Docker), Python parser kustom, inter-annotator agreement check (Cohen's kappa untuk field kategorikal seperti `document_type`).

### 4.5 Output
`data/processed/annotated_raw.jsonl` (belum displit train/val/test — split dilakukan di Pipeline 5 setelah validasi).

### 4.6 Dependencies
Upstream: Data Cleaning (3). Downstream: Dataset Validation (5).

### 4.7 Error Handling
- Record dengan field wajib kosong (`instruction` atau `assistant_response` kosong) otomatis di-reject dari export, dicatat di `annotation_rejects.log`.
- Jika inter-annotator agreement < 0.6 (kappa) pada sample audit, batch ditahan dan anotator diberi kalibrasi ulang sebelum lanjut.

### 4.8 Validation
Schema check (lihat Pipeline 5) + manual spot-check 5% random sample oleh AI/ML Team lead sebelum dataset dianggap "annotation-complete".

### 4.9 Monitoring
`annotation_throughput_per_day`, `annotation_reject_rate`, `inter_annotator_agreement_score`.

---

## 5. Dataset Validation Pipeline

### 5.1 Objective
Menjamin dataset hasil anotasi memenuhi skema, bebas kebocoran data (leakage train/val/test), dan representatif sebelum digunakan training.

### 5.2 Input
`data/processed/annotated_raw.jsonl`.

### 5.3 Processing Steps
1. Validasi skema per record dengan Pydantic model (`schema_validator.py`) — struktur `messages`, role valid (`system|user|assistant`), panjang minimum/maksimum token.
2. Validasi via Great Expectations suite: distribusi `document_type` tidak timpang ekstrem (>80% satu kelas), tidak ada duplikat instruksi-respons exact match antar split.
3. Split train/val/test (default 85/10/5) dengan stratifikasi per `document_type`, seed tetap untuk reproducibility.
4. Leakage check: pastikan tidak ada dokumen sumber (`doc_id`) yang sama muncul di lebih dari satu split.
5. Simpan hasil final ke `data/processed/{train,val,test}.jsonl`.

### 5.4 Tools
Pydantic v2, Great Expectations, `scikit-learn` (`train_test_split` dengan stratify).

### 5.5 Output
`data/processed/train.jsonl`, `val.jsonl`, `test.jsonl`, `validation_report.html` (Great Expectations Data Docs).

### 5.6 Dependencies
Upstream: Dataset Annotation (4). Downstream: Dataset Versioning (6).

### 5.7 Error Handling
Pipeline exit non-zero (fail CI) jika: schema validation error > 0 record, leakage terdeteksi, atau kelas minoritas < 20 sample (terlalu sedikit untuk belajar pola format).

### 5.8 Validation
Great Expectations suite (`configs/data/validation_schema.yaml`) dijalankan sebagai gate wajib — dataset yang gagal validasi TIDAK dapat lanjut ke Pipeline 6 (enforced via script exit code di CI).

### 5.9 Contoh Config — `configs/data/validation_schema.yaml`
```yaml
validation:
  schema:
    required_fields: ["messages"]
    message_roles_allowed: ["system", "user", "assistant"]
    min_tokens_per_record: 20
    max_tokens_per_record: 4096
  checks:
    - name: "no_exact_duplicate_across_splits"
      type: "cross_split_hash_check"
      fail_on_violation: true
    - name: "class_balance_document_type"
      type: "distribution_check"
      max_class_ratio: 0.80
      min_class_count: 20
    - name: "no_doc_id_leakage"
      type: "group_leakage_check"
      group_key: "source_doc_id"
  split:
    ratios: { train: 0.85, val: 0.10, test: 0.05 }
    stratify_by: "document_type"
    random_seed: 42
  on_failure: "block_pipeline"
```

### 5.10 Monitoring
`validation_pass_rate`, `validation_leakage_count`, `class_balance_report` (diarsipkan per run untuk audit trail dataset).

---

## 6. Dataset Versioning Pipeline

### 6.1 Objective
Menjamin setiap dataset yang pernah dipakai training dapat ditelusuri ulang persis (reproducibility) dan tidak hilang/tertimpa.

### 6.2 Input
`data/processed/{train,val,test}.jsonl` yang sudah lolos validasi (Pipeline 5).

### 6.3 Processing Steps
1. `dvc add data/processed/train.jsonl data/processed/val.jsonl data/processed/test.jsonl`.
2. Commit `.dvc` pointer files ke Git dengan tag versi dataset (konvensi penamaan §6.6).
3. `dvc push` ke remote storage (bucket S3-compatible/GCS — detail provider di `ML_Deployment_Strategy.md`).
4. Tag Git: `git tag dataset-v<MAJOR>.<MINOR>-<YYYYMMDD>` lalu `git push --tags`.
5. Catat metadata versi dataset (jumlah record, sumber, hash DVC) ke tabel `dataset_versions` di PostgreSQL (agar bisa dirujuk dari MLflow run params dan dari aplikasi admin dashboard — lihat master_spec §4 Database: PostgreSQL).

### 6.4 Tools
DVC (dengan remote GCS/S3), Git tag, PostgreSQL untuk katalog metadata lintas sistem.

### 6.5 Output
Git tag `dataset-v1.0-20261115`, DVC remote objects, baris baru di tabel `dataset_versions`.

### 6.6 Konvensi Penamaan Versi Dataset
```
dataset-v<MAJOR>.<MINOR>-<YYYYMMDD>[-<label>]
```
- **MAJOR** bertambah: perubahan sumber data signifikan (sumber baru ditambah/dihapus, skema berubah).
- **MINOR** bertambah: penambahan volume record dari sumber yang sama, perbaikan anotasi non-breaking.
- `<label>` opsional: `-synthetic-only`, `-with-company-docs` (begitu dokumen perusahaan riil masuk, Phase 2).

Contoh riwayat:
| Tag | Isi | Catatan |
|---|---|---|
| `dataset-v1.0-20261115` | Public + synthetic awal, 8.000 record | Baseline MVP |
| `dataset-v1.1-20261201` | +1.200 record sintetik kategori SOP | Minor, sumber sama |
| `dataset-v2.0-20270215` | + dokumen historis perusahaan (setelah approval) | Major — sumber baru, butuh re-validasi penuh (Pipeline 5) |

### 6.7 Dependencies
Upstream: Dataset Validation (5). Downstream: Training Pipeline (7), Retraining Pipeline (17).

### 6.8 Error Handling
- `dvc push` gagal (network/remote auth) → retry 3x, jika tetap gagal, versi TIDAK dianggap final (tag Git tidak dibuat sampai push remote sukses) agar tidak ada tag menunjuk data yang tidak bisa direstore.
- Tag Git yang sudah dipush bersifat immutable — perbaikan dataset wajib versi baru, tidak overwrite tag lama.

### 6.9 Validation
Verifikasi `dvc checkout` dari environment bersih (misal Colab baru) berhasil merekonstruksi dataset identik (checksum match) sebelum tag dianggap valid.

### 6.10 Monitoring
Katalog `dataset_versions` di PostgreSQL dipantau via admin dashboard: jumlah versi aktif, ukuran storage DVC remote, versi mana yang sedang dipakai model Production (join ke `model_registry`, lihat Pipeline 10).

---

## 7. Training Pipeline

### 7.1 Objective
Melakukan supervised fine-tuning (SFT) dengan QLoRA pada model kandidat (default kandidat uji pertama: **Qwen2.5-7B-Instruct**, per master_spec §8.1) agar model konsisten pada gaya, struktur, dan format dokumen formal Bahasa Indonesia — TANPA menyuntik pengetahuan dinamis (itu peran RAG, §3).

### 7.2 Input
`data/processed/{train,val}.jsonl` versi tertentu (dirujuk via dataset tag, misal `dataset-v1.0-20261115`), base model dari Hugging Face Hub.

### 7.3 Processing Steps
1. Muat base model dalam 4-bit (NF4) via `bitsandbytes` + `BitsAndBytesConfig`.
2. Siapkan `LoraConfig` (PEFT) sesuai hyperparameter di config run (§7.8).
3. Format dataset jadi chat template model (`dataset_formatter.py`, pakai `tokenizer.apply_chat_template`).
4. Jalankan `TRL SFTTrainer` dengan `peft_config` terpasang — training HANYA dieksekusi di Colab Pro/Kaggle (free/low-cost tier) atau GPU rental (RunPod dkk.), **tidak pernah di dev lokal** (sesuai hardware §7 — CPU 8GB RAM tidak mampu).
5. Checkpoint adapter LoRA disimpan tiap N step + di akhir epoch ke `models/adapters/<run_name>/`.
6. Setiap run otomatis logging ke MLflow (lihat Pipeline 8) — params, metrics (loss train/eval), artifacts (adapter checkpoint).

### 7.4 Tools
HF `transformers`, `peft`, `trl` (`SFTTrainer`), `bitsandbytes`, `accelerate`, MLflow autolog, dijalankan via notebook `scripts/run_training_colab.ipynb` (GPU) — script Python inti (`train_qlora.py`) ditulis & diuji sintaksnya di dev lokal tanpa eksekusi penuh, lalu dieksekusi di Colab/GPU rental.

### 7.5 Output
Adapter LoRA (`adapter_model.safetensors` + `adapter_config.json`) di `models/adapters/qwen2.5-7b-docugen-run01/`, MLflow run record, training log.

### 7.6 Dependencies
Upstream: Dataset Versioning (6), Pipeline Architecture (1). Downstream: Experiment Tracking (8), Model Evaluation (9).

### 7.7 Error Handling
- OOM GPU → config punya fallback otomatis: turunkan `per_device_train_batch_size`, naikkan `gradient_accumulation_steps` agar effective batch size tetap sama (dicatat di config, bukan tebakan manual tiap kali).
- Checkpoint resume: `resume_from_checkpoint: true` jika sesi Colab/Kaggle terputus (umum pada free-tier yang punya limit sesi) — training tidak restart dari nol.
- Loss NaN/divergen → trainer dikonfigurasi `max_grad_norm` (gradient clipping) dan early stop jika eval loss naik 3 evaluasi berturut-turut.

### 7.8 Contoh Config — `configs/training/qlora_qwen2.5_7b_run01.yaml`
```yaml
run:
  run_name: "qwen2.5-7b-docugen-run01"
  base_model: "Qwen/Qwen2.5-7B-Instruct"
  dataset_version: "dataset-v1.0-20261115"
  seed: 42
  output_dir: "models/adapters/qwen2.5-7b-docugen-run01"
  mlflow_tracking_uri: "http://mlflow.internal.docugen.local:5000"
  mlflow_experiment: "docugen-finetune-qwen2.5-7b"

quantization:
  load_in_4bit: true
  bnb_4bit_quant_type: "nf4"
  bnb_4bit_compute_dtype: "bfloat16"
  bnb_4bit_use_double_quant: true

lora:
  r: 16
  lora_alpha: 32
  lora_dropout: 0.05
  bias: "none"
  task_type: "CAUSAL_LM"
  target_modules: ["q_proj", "k_proj", "v_proj", "o_proj", "gate_proj", "up_proj", "down_proj"]

training:
  num_train_epochs: 3
  per_device_train_batch_size: 2
  per_device_eval_batch_size: 2
  gradient_accumulation_steps: 8        # effective batch size = 16
  learning_rate: 2.0e-4
  lr_scheduler_type: "cosine"
  warmup_ratio: 0.03
  weight_decay: 0.01
  max_grad_norm: 0.3
  optim: "paged_adamw_8bit"
  max_seq_length: 2048
  logging_steps: 10
  eval_strategy: "steps"
  eval_steps: 100
  save_strategy: "steps"
  save_steps: 100
  save_total_limit: 3
  resume_from_checkpoint: true
  bf16: true
  packing: false

compute_target:
  environment: "colab_pro"            # colab_pro | kaggle | runpod_rtx4090 | runpod_a100
  expected_vram_gb: 16                 # align dengan riset §8.2: QLoRA 7B ~8-14GB, +buffer evaluasi
```

### 7.9 Validation
Sanity check pre-training: load 1 batch, jalankan 1 forward-backward step dummy di CPU (dev lokal, model kecil pengganti/smoke-model) untuk memastikan kode tidak error sebelum konsumsi kuota GPU Colab/Kaggle yang terbatas.

### 7.10 Monitoring
MLflow metrics real-time: `train/loss`, `eval/loss`, `learning_rate`, `gpu_memory_allocated_gb`, `train/epoch`. Lihat detail integrasi di Pipeline 8.

---

## 8. Experiment Tracking Pipeline

### 8.1 Objective
Mencatat setiap eksperimen training/evaluasi secara terstruktur agar dapat dibandingkan dan direproduksi, menjadi basis keputusan model mana yang naik ke Model Registry.

### 8.2 Input
Semua run dari Training Pipeline (7) dan Evaluation Pipeline (9) via MLflow autolog/manual logging.

### 8.3 Processing Steps
1. Set `MLFLOW_TRACKING_URI` ke server MLflow terpusat (bukan file lokal `mlruns/` per environment — supaya run dari Colab, Kaggle, dan GPU rental semuanya tercatat ke satu tempat).
2. Setiap training run membuat MLflow Run baru di bawah Experiment `docugen-finetune-qwen2.5-7b`, dengan `run_name` = nama run config.
3. Log params (semua field dari `training.yaml`), metrics (loss per step), artifacts (adapter checkpoint, sample generation output), dan tags (`dataset_version`, `base_model`, `compute_target`).
4. Setelah evaluasi (Pipeline 9), metrik eval digabung ke run yang sama (`mlflow.log_metrics` pada run_id yang sama) agar satu run = satu paket lengkap train+eval.

### 8.4 Tools
MLflow Tracking Server (backend store: PostgreSQL — konsisten dengan master_spec §4 Database: PostgreSQL; artifact store: bucket S3-compatible/GCS).

### 8.5 Output
Run tercatat di MLflow UI, dapat di-query via MLflow API untuk perbandingan antar run.

### 8.6 Dependencies
Upstream: Training Pipeline (7). Downstream: Model Evaluation (9), Model Registry (10).

### 8.7 Error Handling
- Jika koneksi ke tracking server putus saat training di Colab (sesi free-tier tidak stabil), metric di-buffer lokal dan di-flush ulang saat koneksi tersedia (`mlflow.autolog` dengan retry wrapper kustom), training tidak berhenti karena logging gagal.
- Artifact besar (merged full-weight model) tidak dikirim ke MLflow artifact store langsung — hanya adapter kecil; full merge disimpan di DVC (lihat Pipeline 6/11) dan direferensikan via path, menghindari artifact store MLflow penuh.

### 8.8 Contoh Konfigurasi — Tracking URI & Struktur Run
```yaml
# configs/training/mlflow_tracking.yaml
mlflow:
  tracking_uri: "postgresql://mlflow_user:***@mlflow-db.internal:5432/mlflow_tracking"
  artifact_location: "s3://docugen-ml-artifacts/mlflow-artifacts/"
  experiment_name: "docugen-finetune-qwen2.5-7b"
  run_naming_convention: "<base_model_short>-<purpose>-run<NN>"
  required_tags:
    - "dataset_version"
    - "base_model"
    - "compute_target"
    - "git_commit_sha"
  autolog:
    transformers: true
    log_models: false     # adapter besar: simpan manual via log_artifact terarah, bukan log_model otomatis
```

Struktur run di MLflow (ilustrasi):
```
Experiment: docugen-finetune-qwen2.5-7b
 ├─ Run: qwen2.5-7b-docugen-run01
 │   params: {r:16, lora_alpha:32, lr:2e-4, epochs:3, dataset_version:"dataset-v1.0-20261115"}
 │   metrics: {train/loss:0.42, eval/loss:0.51, rougeL:0.38, format_compliance_score:0.91}
 │   artifacts: adapter_model.safetensors, sample_generations.json, eval_report.html
 │   tags: {base_model:"Qwen2.5-7B-Instruct", compute_target:"colab_pro", git_commit_sha:"a1b2c3d"}
 └─ Run: qwen2.5-7b-docugen-run02 (hyperparameter berbeda, untuk perbandingan)
```

### 8.9 Validation
Setiap run wajib punya `dataset_version` dan `git_commit_sha` ter-tag — run tanpa dua field ini ditandai tidak valid untuk dipromosikan ke Registry (gate manual saat review).

### 8.10 Monitoring
Dashboard MLflow UI dipakai AI/ML Team untuk membandingkan run (parallel coordinates plot untuk hyperparameter vs metric eval).

---

## 9. Model Evaluation Pipeline

### 9.1 Objective
Mengukur kualitas model hasil fine-tuning secara objektif sebelum dipertimbangkan masuk Model Registry — mencegah model yang belum layak masuk ke jalur deployment.

### 9.2 Input
Adapter LoRA hasil Training Pipeline (7) + base model + `data/processed/test.jsonl` (test set yang TIDAK pernah dilihat saat training, lihat Pipeline 5).

### 9.3 Processing Steps
1. Load base model + adapter (merge sementara di memori untuk inference eval, tidak perlu merge permanen).
2. Jalankan generasi pada seluruh test set dengan parameter decoding tetap (temperature, max_tokens) agar hasil comparable antar run.
3. Hitung metrik otomatis: ROUGE-L/BLEU (kedekatan terhadap referensi), **format compliance score** (custom rule-based checker: heading terdeteksi, numbering konsisten, penutup surat formal ada) — ini metrik kunci karena tujuan fine-tuning adalah konsistensi format (§3).
4. LLM-as-judge: model evaluator (bisa model lebih besar/API eksternal ber-guardrail) menilai kualitas gaya Bahasa Indonesia formal pada skala 1-5, dengan rubric tertulis.
5. Bandingkan terhadap baseline (base model tanpa fine-tuning) dan terhadap run sebelumnya (regression check).
6. Simpan hasil ke MLflow run yang sama (Pipeline 8) + `eval_report.html`.

### 9.4 Tools
`evaluate` (HF), custom `metrics.py` (format compliance rule-based), LLM-as-judge via model self-hosted, `pandas`+`matplotlib`/`plotly` untuk report HTML.

### 9.5 Output
`eval_report.html`, metrics tercatat di MLflow, keputusan tertulis "lolos kandidat Staging / tidak lolos" oleh AI/ML Team (manusia, bukan otomatis).

### 9.6 Dependencies
Upstream: Training Pipeline (7), Dataset Validation (5) (test set). Downstream: Model Registry (10).

### 9.7 Error Handling
Jika skor format compliance turun dibanding baseline sebelumnya (regression), run otomatis ditandai `regression_flagged: true` di MLflow tag dan TIDAK boleh dipromosikan sampai direview manual — mencegah model buruk naik ke registry hanya karena skor agregat lain naik.

### 9.8 Contoh Config — `configs/eval/eval_suite_v1.yaml`
```yaml
evaluation:
  test_set: "data/processed/test.jsonl"
  decoding:
    temperature: 0.3
    top_p: 0.9
    max_new_tokens: 1024
  metrics:
    - name: "rouge_l"
      weight: 0.2
    - name: "format_compliance_score"
      weight: 0.4
      rules: ["has_heading", "numbering_consistent", "formal_closing_present", "no_placeholder_leftover"]
    - name: "llm_judge_quality"
      weight: 0.4
      judge_model: "self_hosted_qwen2.5_7b_base"
      rubric_path: "configs/eval/rubric_formal_id.md"
  regression_check:
    compare_against: "last_production_model"
    fail_if_metric_drop_pct:
      format_compliance_score: 5
  promotion_threshold:
    format_compliance_score_min: 0.85
    llm_judge_quality_min: 3.5
```

### 9.9 Validation
Dua anggota AI/ML Team mereview `eval_report.html` secara independen sebelum keputusan promosi final — bukan otomatis dari skor saja (konsisten dengan prinsip human review master_spec §4 "Human review wajib").

### 9.10 Monitoring
`eval_format_compliance_score_trend` (antar run, time series di MLflow), `eval_regression_flag_count`.

---

## 10. Model Registry

### 10.1 Objective
Menjadi satu sumber kebenaran untuk status siklus hidup setiap versi model (Staging/Production/Archived) agar deployment tidak ambigu model mana yang aktif.

### 10.2 Input
Model/adapter yang sudah lolos Evaluation Pipeline (9) dengan keputusan manual "lolos kandidat".

### 10.3 Processing Steps
1. `scripts/register_model.py` mendaftarkan adapter sebagai Model Version baru di MLflow Model Registry, nama model: `docugen-document-generator`.
2. Model baru otomatis masuk stage **`None`** → setelah review eval disetujui, dipindah ke **`Staging`**.
3. Di `Staging`: model dideploy ke environment staging aplikasi (terhubung ke backend FastAPI staging) untuk uji integrasi end-to-end (bukan hanya metrik offline) — termasuk uji manual oleh AI/ML Team + sample Authorized User (UAT terbatas).
4. Jika UAT staging lolos DAN ada approval formal (lihat Business Process — Model Deployment Approval), `scripts/promote_model.py` memindah stage ke **`Production`** — approval gate ini manual, bukan otomatis.
5. Model Production sebelumnya dipindah ke **`Archived`** (tidak dihapus — tetap bisa rollback).

### 10.4 Tools
MLflow Model Registry (API `mlflow.register_model`, `MlflowClient.transition_model_version_stage`), terhubung ke backend store PostgreSQL yang sama dengan Pipeline 8.

### 10.5 Output
Model Version terdaftar dengan stage jelas; tabel `model_registry` yang bisa di-query admin dashboard untuk menampilkan model mana yang aktif di Production.

### 10.6 Dependencies
Upstream: Model Evaluation (9). Downstream: Model Deployment Pipeline (11), Retraining Pipeline (17).

### 10.7 Error Handling
- Transisi stage ke `Production` divalidasi oleh script: menolak transisi jika model version yang sama belum pernah lewat stage `Staging` (tidak boleh lompat langsung None→Production).
- Rollback: jika model Production baru menimbulkan masalah pasca-deploy (lihat Monitoring Pipeline 15), stage dikembalikan ke `Archived` dan model Production sebelumnya (`Archived` → `Production`) diaktifkan ulang — proses manual dengan approval darurat dicatat di audit log.

### 10.8 Contoh Status Stage (ilustrasi MLflow Model Registry)
```yaml
model_name: "docugen-document-generator"
versions:
  - version: 3
    stage: "Production"
    source_run_id: "a1b2c3d4-run01"
    dataset_version: "dataset-v1.0-20261115"
    promoted_by: "admin_mlteam_lead"
    promoted_at: "2026-11-20T09:00:00+07:00"
  - version: 2
    stage: "Archived"
    source_run_id: "x9y8z7-run00"
    archived_reason: "superseded_by_v3_better_format_compliance"
  - version: 4
    stage: "Staging"
    source_run_id: "e5f6g7h8-run02"
    note: "sedang UAT staging, belum approval produksi"
```

### 10.9 Validation
Audit periodik: pastikan hanya ADA SATU model version berstatus `Production` per nama model pada satu waktu (invariant yang dicek via script `scripts/validate_registry_invariants.py`).

### 10.10 Monitoring
Alert jika ada lebih dari 1 versi `Production` simultan (harus tidak pernah terjadi), dashboard menampilkan riwayat promosi/rollback.

---

## 11. Model Deployment Pipeline

### 11.1 Objective
Memindahkan model dari Model Registry stage `Production` ke infrastruktur serving nyata secara terkontrol, dengan jalur CI/CD yang auditable.

### 11.2 Input
Model Version dengan stage `Production` di Model Registry (10).

### 11.3 Processing Steps (tahapan CI/CD)
1. **Trigger**: manual approval trigger (bukan otomatis saat commit) — dipicu setelah stage promotion ke `Production` disetujui.
2. **Build**: merge adapter LoRA ke base model (`peft_model.merge_and_unload()`) menghasilkan full-weight model siap serving, atau tetap pakai adapter terpisah jika serving engine mendukung LoRA hot-swap.
3. **Package**: build container image (serving engine + model weight reference, bukan weight di-bake ke image — weight diambil dari model storage saat startup untuk image tetap ringan).
4. **Test (staging smoke test)**: deploy ke staging cluster, jalankan smoke test request (contoh §12.6) dan health check endpoint `/health`.
5. **Canary**: rollout ke sebagian kecil traffic produksi (misal 10%) dengan monitoring ketat latency/error rate selama window observasi (misal 30 menit–2 jam).
6. **Full rollout**: jika canary sehat, naikkan ke 100% traffic produksi.
7. **Rollback plan**: jika canary/metrik produksi memburuk, otomatis rollback ke model Production versi sebelumnya (dari stage `Archived`), tanpa menunggu intervensi manual untuk tahap rollback teknis (meski keputusan *root cause* tetap direview manusia).

### 11.4 Tools
GitHub Actions/GitLab CI (pipeline deployment), Docker, Kubernetes (atau Docker Compose untuk skala awal self-hosted sesuai hardware reality — detail infrastruktur produksi ada di `ML_Deployment_Strategy.md`), vLLM sebagai serving engine (Pipeline 12), Prometheus untuk canary metric gate.

### 11.5 Output
Model aktif melayani traffic produksi di balik endpoint inference internal (dikonsumsi backend FastAPI, tidak langsung oleh frontend Next.js — sesuai boundary arsitektur master_spec §4/§6).

### 11.6 Dependencies
Upstream: Model Registry (10). Downstream: Inference Pipeline (12), Monitoring Pipeline (15).

### 11.7 Error Handling
- Gate otomatis di tahap Canary: jika `error_rate_5xx > 2%` atau `p95_latency` naik >50% dibanding baseline selama window observasi, deployment otomatis dihentikan dan rollback dieksekusi, alert terkirim ke AI/ML Team.
- Deployment tidak dapat dijalankan tanpa approval record tercatat (lihat Business Process — Model Deployment Approval); CI/CD job memverifikasi field approval di tiket/registry sebelum lanjut ke tahap Build.

### 11.8 Validation
Health check + smoke test wajib hijau sebelum Canary dimulai; checklist deployment (model version, dataset version, eval report link, approval ticket ID) dilampirkan otomatis ke deployment record.

### 11.9 Monitoring
`deployment_canary_error_rate`, `deployment_canary_latency_p95`, `deployment_rollback_count`, status pipeline CI/CD (lihat Pipeline 15 untuk integrasi penuh Prometheus/Grafana).

---

## 12. Inference Pipeline

### 12.1 Objective
Menyajikan model Production untuk konsumsi backend FastAPI dengan latency rendah dan throughput memadai untuk conversational use case.

### 12.2 Input
Request generasi dari backend FastAPI (hasil orkestrasi chat + context RAG yang sudah di-assemble, lihat Pipeline 14), berisi prompt final + parameter decoding.

### 12.3 Processing Steps
1. Serving engine **vLLM** dijalankan sebagai service terpisah dari backend aplikasi (konsisten master_spec §4 "backend Python FastAPI terpisah dari inference service").
2. vLLM meng-host model Production (full-weight merged, atau base+adapter via vLLM LoRA multi-adapter serving jika beberapa varian aktif).
3. Backend FastAPI memanggil endpoint OpenAI-compatible vLLM (`/v1/chat/completions`) secara internal (service-to-service, tidak exposed publik).
4. Response di-stream (SSE/token streaming) balik ke backend → frontend untuk UX chat yang responsif.

### 12.4 Tools
vLLM (serving engine, OpenAI-compatible API), FastAPI (backend klien internal ke vLLM), Pydantic untuk validasi skema request/response di layer backend.

### 12.5 Output
Response JSON (atau stream SSE) berisi teks dokumen/jawaban yang dihasilkan model, dikonsumsi backend untuk disusun ke format chat/preview dokumen.

### 12.6 Contoh Request/Response API (vLLM OpenAI-compatible endpoint)

**Request** (`POST http://inference.internal.docugen.local:8000/v1/chat/completions`):
```json
{
  "model": "docugen-document-generator-v3",
  "messages": [
    {"role": "system", "content": "Anda adalah asisten penulisan dokumen resmi perusahaan. Gunakan format baku: salam pembuka, isi terstruktur dengan heading, penutup formal."},
    {"role": "user", "content": "Buatkan draft surat permohonan cuti tahunan untuk karyawan bernama Budi, 5 hari kerja mulai 2 Desember 2026."}
  ],
  "temperature": 0.3,
  "top_p": 0.9,
  "max_tokens": 800,
  "stream": true
}
```

**Response** (contoh non-stream untuk keterbacaan; produksi memakai `stream: true`):
```json
{
  "id": "chatcmpl-internal-7f3a",
  "object": "chat.completion",
  "model": "docugen-document-generator-v3",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "SURAT PERMOHONAN CUTI TAHUNAN\n\nKepada Yth. ... [draf lengkap sesuai format baku] ..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {"prompt_tokens": 142, "completion_tokens": 310, "total_tokens": 452}
}
```

### 12.7 Dependencies
Upstream: Model Deployment Pipeline (11). Downstream: dikonsumsi langsung oleh backend aplikasi (di luar scope dokumen ini — lihat PRD LLM Document Generator), dan terhubung ke RAG Retrieval Pipeline (14) untuk context injection sebelum request dikirim ke sini.

### 12.8 Error Handling
- Timeout request (misal >30s) → backend menampilkan fallback message, request dicatat untuk audit kapasitas.
- vLLM engine crash/OOM → health check `/health` gagal → orchestrator (Kubernetes/Compose) restart otomatis + alert; traffic dialihkan ke instance lain jika replika >1.
- Rate limiting per user/department diterapkan di layer backend FastAPI (bukan di vLLM), sejalan dengan prinsip authorization ditegakkan di backend (§9.8).

### 12.9 Validation
Load test berkala (`k6`/`locust`) terhadap target latency SLA (didefinisikan di `ML_Deployment_Strategy.md`) sebelum dan sesudah setiap deployment model baru.

### 12.10 Monitoring
`inference_latency_p50_p95_p99`, `inference_tokens_per_second`, `inference_requests_total`, `inference_error_rate`, `gpu_utilization_percent` — detail dashboard di Pipeline 15.

---

## 13. RAG Ingestion Pipeline

### 13.1 Objective
Mengubah dokumen internal (SOP, kebijakan, dokumen referensi) menjadi representasi vektor yang dapat di-retrieve, dengan metadata ACL melekat di setiap chunk agar retrieval permission-aware (wajib, master_spec §9.4).

### 13.2 Input
Dokumen yang diupload/diapprove masuk Knowledge Base oleh Admin, lengkap dengan klasifikasi sensitivitas (`Public/Internal/Confidential/Restricted`, §9.5) dan daftar permission (role/user/department yang boleh akses).

### 13.3 Processing Steps
1. Ekstraksi teks dari dokumen (PDF/DOCX) via parser (`unstructured`/`pymupdf`).
2. **Chunking strategy**: recursive character/token-based chunking dengan overlap, ukuran chunk disesuaikan konteks dokumen formal (lihat config §13.8) — bukan fixed-size naif, mempertimbangkan boundary heading/paragraf agar chunk tetap koheren secara semantik.
3. **Embedding**: model embedding multilingual yang mendukung Bahasa Indonesia dengan baik, dipilih `BAAI/bge-m3` (multilingual, mendukung dense+sparse hybrid, dimensi 1024) sebagai pilihan default — dapat dievaluasi ulang terhadap alternatif (`intfloat/multilingual-e5-large`) berdasarkan benchmark retrieval internal.
4. Setiap chunk disimpan bersama metadata ACL lengkap (§13.9) ke vector DB — **pgvector (default MVP)** sesuai master_spec §4, dengan jalur migrasi ke Qdrant di Phase 2 bila dibutuhkan skala/fitur lanjutan.
5. Indexing ulang (re-ingest) dijadwalkan atau dipicu event saat dokumen sumber diupdate/di-revoke aksesnya oleh Admin.

### 13.4 Tools
`unstructured`/`pymupdf` (ekstraksi), `langchain`/custom chunker (`src/docugen_ml/rag/chunker.py`), `sentence-transformers` untuk load model embedding, pgvector extension on PostgreSQL (atau Qdrant client jika Phase 2).

### 13.5 Output
Baris-baris chunk di tabel `kb_chunks` (PostgreSQL+pgvector) berisi `embedding vector(1024)`, `content`, `metadata jsonb`.

### 13.6 Dependencies
Upstream: Admin document upload & approval (di luar scope ML, lihat PRD Document Control). Downstream: RAG Retrieval Pipeline (14).

### 13.7 Error Handling
- Dokumen tanpa klasifikasi sensitivitas atau tanpa daftar ACL eksplisit **ditolak masuk ingestion** (fail closed, bukan default-allow) — konsisten §9.5.
- Jika model embedding gagal memproses chunk (misal karakter rusak/encoding error), chunk tersebut di-skip dan dicatat di `ingestion_errors` tabel untuk retry manual, bukan membuat embedding kosong/placeholder.

### 13.8 Contoh Config — `configs/rag/ingestion_chunking.yaml`
```yaml
rag_ingestion:
  chunking:
    strategy: "recursive_structure_aware"
    chunk_size_tokens: 400
    chunk_overlap_tokens: 60
    respect_boundaries: ["heading", "paragraph", "table"]
  embedding:
    model: "BAAI/bge-m3"
    dimension: 1024
    batch_size: 32
    normalize: true
  vector_store:
    backend: "pgvector"            # default MVP per master_spec §4; alternatif: "qdrant" (Phase 2)
    table: "kb_chunks"
    distance_metric: "cosine"
  acl_required_fields: ["classification", "allowed_roles", "allowed_departments", "allowed_user_ids", "document_owner"]
  reject_on_missing_acl: true
```

### 13.9 Contoh Metadata ACL per Chunk (disimpan di kolom `metadata jsonb`)
```json
{
  "document_id": "kb-doc-00123",
  "chunk_index": 4,
  "classification": "Internal",
  "allowed_roles": ["authorized_user", "admin"],
  "allowed_departments": ["Finance", "HR"],
  "allowed_user_ids": [],
  "document_owner": "admin_user_42",
  "source_filename": "SOP_Pengajuan_Reimbursement_2026.pdf",
  "ingested_at": "2026-11-10T14:22:00+07:00",
  "document_version": "v2",
  "revoked": false
}
```

### 13.10 Validation
Query test wajib dijalankan post-ingestion: retrieval dengan user context "tanpa permission" terhadap dokumen baru harus mengembalikan nol hasil (negative test ACL), sebelum ingestion batch dianggap selesai.

### 13.11 Monitoring
`rag_ingestion_chunks_total`, `rag_ingestion_acl_rejects_total`, `rag_embedding_latency_ms`, `kb_storage_size_mb`.

---

## 14. RAG Retrieval Pipeline

### 14.1 Objective
Mengambil chunk relevan dari Knowledge Base saat inference, dengan filter ACL diterapkan SEBELUM konten masuk context model — bukan disaring belakangan atau hanya diandalkan pada system prompt (wajib, §9.4 & §9.8).

### 14.2 Input
Query user (hasil chat turn), identitas & permission user yang terautentikasi dari backend FastAPI (role, department, user_id).

### 14.3 Processing Steps
1. Backend menerima pesan user, mengambil **permission context** user dari session (bukan dari input user — mencegah spoofing).
2. Query di-embed dengan model embedding yang sama dengan ingestion (`BAAI/bge-m3`).
3. **Filter ACL diterapkan sebagai predicate WHERE pada query vector DB** (bukan post-filter di aplikasi) — chunk yang tidak match permission user tidak pernah keluar dari database sama sekali.
4. Similarity search (top-k, default k=5) + reranking opsional (cross-encoder) untuk presisi lebih tinggi pada kandidat teratas.
5. Chunk hasil retrieval disusun jadi context block, disisipkan ke prompt sebelum dikirim ke Inference Pipeline (12).
6. Setiap retrieval dicatat ke audit log: user_id, query, document_id yang diakses (untuk accountability §9.6).

### 14.4 Tools
pgvector `<=>` cosine distance operator dalam SQL query berfilter ACL, (opsional Phase 2) Qdrant filtered search dengan payload filter, cross-encoder reranker (`BAAI/bge-reranker-v2-m3`) opsional.

### 14.5 Output
Context block (teks chunk terpilih + sumber) yang disuntikkan ke prompt inference; log retrieval di tabel audit.

### 14.6 Contoh Query dengan Filter ACL (SQL, pgvector)
```sql
-- user_context: role='authorized_user', department='Finance', user_id='u-9981'
SELECT
  chunk_id, content, metadata,
  1 - (embedding <=> :query_embedding) AS similarity_score
FROM kb_chunks
WHERE
  (metadata->>'revoked')::boolean = false
  AND (
    metadata->'allowed_roles' ? :user_role
    OR metadata->'allowed_departments' ? :user_department
    OR metadata->'allowed_user_ids' ? :user_id
  )
ORDER BY embedding <=> :query_embedding
LIMIT 5;
```
Parameter `:query_embedding`, `:user_role`, `:user_department`, `:user_id` diisi oleh backend FastAPI dari session terautentikasi — tidak pernah dari input mentah frontend.

### 14.7 Dependencies
Upstream: RAG Ingestion Pipeline (13), autentikasi/RBAC backend (di luar scope ML, lihat PRD). Downstream: Inference Pipeline (12).

### 14.8 Error Handling
- Jika hasil retrieval kosong (user tidak punya akses ke apa pun yang relevan), model tetap merespons tanpa context KB dan secara eksplisit memberi tahu user bahwa tidak ada dokumen yang dapat diakses terkait topik tersebut — tidak diam-diam memakai pengetahuan parametrik model sebagai pengganti tanpa disclosure.
- Query embedding gagal (service timeout) → fallback: lanjut tanpa RAG context + flag di response metadata `rag_retrieval_status: "failed"` agar backend/UI bisa menampilkan notice, bukan gagal total.

### 14.9 Validation
Unit test reguler: skenario user lintas role/department terhadap set dokumen uji dengan ACL beragam, memverifikasi tidak ada kebocoran (false positive akses).

### 14.10 Monitoring
`rag_retrieval_latency_ms`, `rag_retrieval_empty_result_rate`, `rag_acl_denied_count`, `rag_top1_similarity_score_distribution` (sinyal kualitas retrieval, dipakai juga untuk deteksi drift KB di Pipeline 15).

---

## 15. Monitoring Pipeline

### 15.1 Objective
Memberi visibilitas real-time terhadap kesehatan sistem AI (inference, RAG, model) agar degradasi terdeteksi sebelum berdampak luas ke user.

### 15.2 Input
Metrik dari semua pipeline runtime: Inference (12), RAG Retrieval (14), Deployment canary (11), serta log aplikasi backend.

### 15.3 Processing Steps
1. Setiap service (vLLM, backend FastAPI, RAG retriever) expose endpoint `/metrics` format Prometheus (custom exporter di `src/docugen_ml/monitoring/exporters.py` untuk metrik yang tidak native).
2. Prometheus scrape berkala (interval 15s) dari seluruh target.
3. Grafana dashboard menampilkan panel: latency, throughput, error rate, dan **drift signal**.
4. Alerting rules (Alertmanager) terpicu pada threshold yang didefinisikan (§15.6), notifikasi ke channel AI/ML Team (Slack/email/webhook).

### 15.4 Tools
Prometheus, Grafana, Alertmanager, custom Python exporter (`prometheus_client` library).

### 15.5 Output
Dashboard Grafana, alert notifications, metrik historis untuk analisis tren.

### 15.6 Contoh Metrik Prometheus (nama metrik + deskripsi)
| Metrik | Tipe | Deskripsi |
|---|---|---|
| `inference_request_latency_seconds` | Histogram | Latency end-to-end request inference (label: `model_version`) |
| `inference_tokens_per_second` | Gauge | Throughput generasi token saat ini |
| `inference_error_rate_5xx` | Counter (rate) | Proporsi request gagal 5xx |
| `rag_retrieval_latency_seconds` | Histogram | Latency pencarian vector DB |
| `rag_empty_result_rate` | Gauge | Proporsi retrieval tanpa hasil (indikasi KB kurang/ACL terlalu ketat) |
| `model_output_embedding_drift_score` | Gauge | **Drift signal**: jarak distribusi embedding output model periode saat ini vs periode baseline (minggu peluncuran) — kenaikan signifikan mengindikasikan perilaku model bergeser dari ekspektasi, trigger investigasi (bukan otomatis retraining, lihat Pipeline 17) |
| `gpu_utilization_percent` | Gauge | Utilisasi GPU server inference |
| `document_format_compliance_violation_rate` | Gauge | Sampling periodik output produksi dicek rule-based format compliance (sama rule seperti Pipeline 9), naik = sinyal kualitas menurun pasca-deploy |

Contoh alert rule (`prometheus/alerts.yml`):
```yaml
groups:
  - name: docugen-ai-alerts
    rules:
      - alert: HighInferenceErrorRate
        expr: rate(inference_error_rate_5xx[5m]) > 0.02
        for: 10m
        labels: { severity: critical }
        annotations:
          summary: "Error rate 5xx inference > 2% selama 10 menit"
      - alert: ModelDriftDetected
        expr: model_output_embedding_drift_score > 0.35
        for: 1h
        labels: { severity: warning }
        annotations:
          summary: "Drift signal tinggi — investigasi manual diperlukan, bukan trigger retraining otomatis"
```

### 15.7 Dependencies
Upstream: semua pipeline runtime (11, 12, 14). Downstream: Feedback Pipeline (16, sinyal kualitatif), Retraining Pipeline (17, sinyal kuantitatif sebagai salah satu trigger kandidat).

### 15.8 Error Handling
Prometheus down → Alertmanager fallback healthcheck eksternal (dead man's switch) memberi tahu jika monitoring sendiri mati — mencegah "silent blind spot".

### 15.9 Validation
Uji berkala (chaos-light): simulasi error rate tinggi di staging untuk memastikan alert benar-benar terpicu dan diterima channel yang benar.

### 15.10 Monitoring (meta)
Dashboard "monitoring of monitoring": uptime Prometheus/Grafana/Alertmanager sendiri.

---

## 16. Feedback Pipeline

### 16.1 Objective
Mengumpulkan sinyal kualitas dari user nyata (thumbs up/down + komentar opsional) sebagai bahan evaluasi berkelanjutan — **BUKAN jalur otomatis masuk training set** (tegas sesuai master_spec §9.3: dataset training butuh approval eksplisit).

### 16.2 Input
Interaksi user: klik thumbs up/down pada output AI di UI chat, opsional free-text alasan.

### 16.3 Processing Steps
1. Backend FastAPI mencatat feedback ke tabel `ai_feedback` (PostgreSQL): `{conversation_id, message_id, rating, comment, user_id, timestamp}` — TIDAK menyentuh dataset training sama sekali di tahap ini.
2. Job berkala (misal mingguan) mengagregasi feedback: filter rating negatif + komentar substantif menjadi kandidat review (`data/candidates/feedback_candidates_<periode>.jsonl`), berisi `{prompt, model_output, user_feedback, conversation_context}`.
3. AI/ML Team mereview kandidat secara manual: menentukan apakah ini **bug model** (masuk kandidat dataset perbaikan, lanjut ke langkah 4) atau **bukan masalah model** (misal ekspektasi user di luar scope, ditutup tanpa tindakan).
4. Kandidat yang disetujui diberi anotasi perbaikan (respons ideal ditulis ulang oleh reviewer/anotator) → masuk ke jalur **Dataset Annotation Pipeline (4)** sebagai batch baru berlabel `source: feedback_derived`, lalu mengikuti seluruh gate Validasi (5) dan approval sebelum menjadi bagian dataset training resmi.
5. Tidak ada jalur kode yang menyuntik `feedback_candidates_*.jsonl` langsung ke `data/processed/train.jsonl` — dua direktori ini sengaja dipisah untuk mencegah bypass approval secara teknis.

### 16.4 Tools
Endpoint FastAPI feedback (di luar scope ML tapi relevan), `pandas` untuk agregasi, Label Studio (re-gunakan Pipeline 4) untuk anotasi perbaikan.

### 16.5 Output
`data/candidates/feedback_candidates_<periode>.jsonl` (kandidat, belum approved), keputusan review tertulis per kandidat (approve/reject + alasan), batch baru ke Annotation Pipeline jika disetujui.

### 16.6 Dependencies
Upstream: Monitoring Pipeline (15, sinyal kualitatif tambahan), aplikasi chat (di luar scope ML). Downstream: Dataset Annotation Pipeline (4) — hanya untuk kandidat yang disetujui; Retraining Pipeline (17) secara tidak langsung.

### 16.7 Error Handling
Feedback dengan konten mencurigakan (misal berisi data pribadi user lain yang ter-paste tanpa sengaja) ditandai untuk redaksi manual sebelum masuk file kandidat — tidak diproses otomatis apa adanya.

### 16.8 Validation
Setiap entri di `feedback_candidates_*.jsonl` yang masuk ke Annotation Pipeline wajib punya field `review_decision: "approved"` dan `reviewed_by` terisi — script validasi menolak batch yang tidak lengkap field ini.

### 16.9 Monitoring
`feedback_thumbs_down_rate_by_document_type`, `feedback_review_backlog_count`, `feedback_to_dataset_conversion_rate`.

---

## 17. Retraining Pipeline

### 17.1 Objective
Mendefinisikan kapan dan bagaimana model dilatih ulang, dengan approval gate eksplisit — model Production **tidak pernah diganti otomatis** tanpa evaluasi dan persetujuan manusia (tegas, konsisten §9.3 dan Model Registry §10).

### 17.2 Input
Sinyal trigger (kuantitatif dari Monitoring Pipeline 15, kualitatif dari Feedback Pipeline 16, atau terjadwal), dataset versi baru hasil akumulasi kandidat yang disetujui.

### 17.3 Kondisi Trigger (kandidat, bukan otomatis eksekusi)
1. **Terjadwal**: review rutin (misal triwulanan) — apakah cukup data baru terkumpul untuk dipertimbangkan retraining.
2. **Drift signal tinggi** dari Monitoring (`model_output_embedding_drift_score` di atas threshold berkepanjangan) — trigger *investigasi*, keputusan retraining tetap manual.
3. **Volume dataset kandidat approved** dari Feedback Pipeline melewati ambang (misal >500 record baru terkumpul sejak versi dataset terakhir) — trigger *proposal* retraining ke AI/ML Team lead, bukan auto-run.
4. **Perubahan kebijakan/format dokumen perusahaan** (business-driven, di luar sinyal teknis) — permintaan eksplisit dari Admin/stakeholder.

Semua trigger di atas hanya menghasilkan **proposal retraining** yang ditinjau manusia — tidak ada cron job yang langsung menjalankan training end-to-end tanpa checkpoint keputusan manusia.

### 17.4 Processing Steps
1. Proposal retraining diajukan (otomatis sebagai notifikasi, atau manual oleh AI/ML Team) dengan justifikasi trigger.
2. **Approval Gate 1**: AI/ML Team lead menyetujui proposal → dataset baru dirakit (gabungan dataset versi sebelumnya + kandidat approved baru) → masuk Dataset Versioning (6) sebagai tag MAJOR/MINOR baru.
3. Training Pipeline (7) dijalankan ulang dengan dataset versi baru, menghasilkan model candidate baru.
4. Model Evaluation Pipeline (9) wajib dijalankan penuh, termasuk regression check terhadap model Production saat ini.
5. **Approval Gate 2**: hasil evaluasi direview manusia (minimal 2 reviewer, sama seperti §9.9) — hanya model yang **lebih baik atau setara** pada metrik kunci yang boleh lanjut.
6. **Approval Gate 3** (approval formal deployment, sama seperti Pipeline 11): model candidate dipromosikan ke `Staging` → UAT → baru opsi promosi ke `Production` dengan tanda tangan approval tercatat (Business Process — Model Deployment Approval).
7. Model Production lama otomatis dipindah `Archived` (tidak dihapus) sebagai jalur rollback instan jika diperlukan.

### 17.5 Tools
Sama seperti Pipeline 6, 7, 9, 10, 11 — Retraining Pipeline adalah *orchestration* ulang dari pipeline yang sudah ada, bukan sistem terpisah, ditambah lapisan approval-gate tracking (tabel `retraining_proposals` di PostgreSQL untuk mencatat siapa approve apa di tahap mana).

### 17.6 Output
Model version baru (jika disetujui penuh) di Model Registry, atau proposal yang ditolak dengan alasan tercatat (retraining juga bisa berakhir "tidak dilanjutkan" — ini outcome valid, bukan kegagalan sistem).

### 17.7 Dependencies
Upstream: Monitoring (15), Feedback (16), Dataset Versioning (6). Downstream: seluruh ulang Training(7)→Eval(9)→Registry(10)→Deployment(11).

### 17.8 Error Handling
- Jika model candidate retraining TIDAK lebih baik dari Production (regression di metrik kunci), proses berhenti di Approval Gate 2 — Production tidak tersentuh, tidak ada downtime/risiko dari upaya retraining yang gagal.
- Jika salah satu Approval Gate ditolak, seluruh rantai berhenti di titik itu dan dicatat (`retraining_proposals.status = "rejected_at_gate_2"` misalnya) untuk audit — tidak ada jalur "retry otomatis tanpa approval ulang".

### 17.9 Contoh Pencatatan Approval Gate (ilustrasi tabel `retraining_proposals`)
```json
{
  "proposal_id": "rtp-2027-03-001",
  "trigger_reason": "feedback_candidate_volume_threshold_exceeded",
  "proposed_at": "2027-03-01T10:00:00+07:00",
  "gate_1_dataset_approval": {"approved_by": "mlteam_lead", "approved_at": "2027-03-03", "dataset_version": "dataset-v2.1-20270303"},
  "gate_2_eval_approval": {"approved_by": ["reviewer_a", "reviewer_b"], "approved_at": "2027-03-10", "regression_check": "passed", "format_compliance_delta": "+0.03"},
  "gate_3_deployment_approval": {"approved_by": "admin_doc_control", "approved_at": "2027-03-15", "approval_ticket": "CHG-0098"},
  "final_status": "promoted_to_production",
  "resulting_model_version": 5
}
```

### 17.10 Validation
Audit tahunan terhadap seluruh `retraining_proposals`: memastikan tidak ada proposal yang "lompat gate" (promoted tanpa salah satu dari tiga approval tercatat) — invariant yang sama pentingnya dengan invariant single-Production-version di Pipeline 10.

### 17.11 Monitoring
`retraining_proposals_total{status=...}`, `retraining_cycle_duration_days` (dari proposal sampai keputusan final), `retraining_rejection_rate_by_gate`.

---

## Lampiran — Ringkasan Keterhubungan Antar Pipeline

```
2 Ingestion → 3 Cleaning → 4 Annotation → 5 Validation → 6 Versioning
                                                              │
                                                              ▼
                                                      7 Training ──► 8 Tracking
                                                              │          │
                                                              ▼          ▼
                                                      9 Evaluation ──► 10 Registry
                                                                          │
                                                                          ▼
                                                                  11 Deployment ──► 12 Inference
                                                                                        ▲
13 RAG Ingestion ──► 14 RAG Retrieval ─────────────────────────────────────────────────┘
                                                                          │
                                                                          ▼
                                                                  15 Monitoring
                                                                     │        │
                                                                     ▼        ▼
                                                            16 Feedback   17 Retraining
                                                              (approval        (3x approval
                                                               gate wajib)      gate wajib)
                                                                     │              │
                                                                     └──► 4 Annotation (batch baru)
```

Catatan penutup: dokumen ini konsisten dengan `master_spec.md` — tidak ada otomasi yang memotong jalur approval manusia untuk training set (§9.3), tidak ada fine-tuning yang menggantikan peran RAG untuk pengetahuan dinamis (§3), vector DB default MVP adalah pgvector dengan Qdrant sebagai opsi Phase 2 (§4), dan seluruh beban komputasi GPU-berat (training/eval) dijalankan di Colab/Kaggle lalu GPU rental — tidak pernah di dev lokal CPU-only (§7).
