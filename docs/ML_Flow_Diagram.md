# ML_Flow_Diagram.md — DocuGen AI

**Dokumen turunan #5** dari `master_spec.md` (single source of truth). Dokumen ini adalah **visual level "apa yang terjadi"** untuk seluruh alur ML/MLOps DocuGen AI, menggunakan diagram Mermaid. Detail implementasi teknis (tools, config, folder structure, script) **TIDAK** dibahas di sini — itu domain `ML_Pipeline.md` ("bagaimana caranya"). Nama tool disebut singkat hanya jika perlu untuk konteks alur (misal "Colab/Kaggle", "pgvector/Qdrant"), tanpa detail konfigurasi.

Referensi keputusan yang mengikat seluruh diagram di dokumen ini (lihat `master_spec.md`):
- RAG wajib **permission-aware** — filter ACL dokumen terjadi di backend sebelum masuk context, bukan di system prompt (§9.4, §9.8).
- Dataset training **wajib approval eksplisit** sebelum dipakai fine-tuning (§9.3, §4 Decision Log).
- **Admin merangkap fungsi Document Control** di MVP — tidak ada role Document Control terpisah (§4, §6).
- **Hardware lokal terbatas** (CPU-only, 8GB RAM) — dataset prep/cleaning/annotation & kode training/eval jalan lokal, tapi **eksekusi training/evaluasi GPU-berat wajib di cloud GPU terpisah** (Colab/Kaggle dulu → GPU rental bila perlu scale) (§7).
- **Model registry, canary release, dan rollback** adalah bagian dari MLOps dan wajib ada sebelum model menyentuh traffic produksi penuh (§6, Out of scope section tidak mengecualikan ini).
- Dataset upload/percakapan user **tidak otomatis** jadi training data (§9.1–9.2).

---

## 1. End-to-End ML Lifecycle

Alur besar dari pemilihan foundation model sampai model berjalan di produksi dan siklus retraining-nya, menunjukkan pemisahan tegas antara **zona lokal (CPU)** dan **zona cloud GPU**.

```mermaid
flowchart TD
    A[Foundation Model Selection\nQwen2.5-7B-Instruct kandidat utama] --> B[Dataset Engineering\n#diagram 2]
    B --> C{Dataset approved?}
    C -- Tidak --> B
    C -- Ya --> D[Fine-tuning\nLoRA/QLoRA — Cloud GPU\n#diagram 4]
    D --> E[Model Evaluation\nbenchmark + human review\n#diagram 5]
    E --> F{Lolos threshold?}
    F -- Tidak --> D
    F -- Ya --> G[Model Registry\n#diagram 6]
    G --> H[Model Deployment\ncanary release\n#diagram 7]
    H --> I[Model Monitoring\nproduksi\n#diagram 8]
    I --> J{Drift / degradasi\nterdeteksi?}
    J -- Ya --> K[Model Retraining\n#diagram 10]
    K --> B
    J -- Tidak --> I

    subgraph ZONA_LOKAL["Zona Lokal (CPU, dev machine)"]
        A
        B
    end
    subgraph ZONA_CLOUD["Zona Cloud GPU (Colab/Kaggle -> GPU rental)"]
        D
        E
    end
    subgraph ZONA_PRODUKSI["Zona Produksi (MLOps)"]
        G
        H
        I
        K
    end
```

### Input
Kandidat foundation model open-weight (hasil riset §8 master_spec), kebutuhan bisnis dokumen generation (gaya, format, bahasa).

### Process
Seleksi model → rekayasa dataset → fine-tuning di cloud GPU → evaluasi → registrasi model → deployment bertahap (canary) → monitoring berkelanjutan → retraining saat terdeteksi penurunan performa.

### Output
Model fine-tuned yang terverifikasi, teregistrasi, dan berjalan stabil di produksi dengan jalur retraining yang terdefinisi.

### Actor
AI/ML Team (eksekusi teknis), Administrator (approval dataset & model, karena merangkap Document Control di MVP), System/AI Model (non-approval).

### Dependencies
Hasil setiap diagram turunan (2–10); ketersediaan hardware cloud GPU; dataset yang sudah disetujui.

### Decision Points
- Dataset disetujui atau tidak (gate approval).
- Model lolos threshold evaluasi atau tidak.
- Drift/degradasi performa terdeteksi di monitoring atau tidak → trigger retraining.

### Failure Handling
Dataset ditolak → kembali ke rekayasa dataset. Model gagal evaluasi → kembali ke fine-tuning dengan dataset/hyperparameter revisi. Degradasi di produksi → retraining loop, dengan model lama tetap serving (lihat rollback di diagram 7).

---

## 2. Dataset Engineering Flow

```mermaid
flowchart TD
    A[Sumber Data] --> A1[Dataset publik]
    A --> A2[Dataset sintetis]
    A --> A3[Dokumen historis perusahaan\n_jika sudah tersedia_]
    A1 --> B[Koleksi & Penggabungan]
    A2 --> B
    A3 --> B
    B --> C[Cleaning & Normalisasi\n_CPU lokal_]
    C --> D[Annotasi / Labeling\n_CPU lokal_]
    D --> E[Klasifikasi Sensitivitas\nPublic/Internal/Confidential/Restricted]
    E --> F[Dataset Validation\n#diagram 3]
    F --> G{Valid?}
    G -- Tidak --> C
    G -- Ya --> H[Pengajuan Approval Dataset\nke Admin]
    H --> I{Admin approve?}
    I -- Tidak --> C
    I -- Ya --> J[Dataset Versioning\n& Registrasi di Dataset Store]
    J --> K[Siap dipakai Fine-tuning]
```

### Input
Dokumen publik, data sintetis hasil generasi/template, dokumen historis perusahaan (bila ada), hasil feedback dari retraining loop.

### Process
Koleksi dari berbagai sumber → cleaning/normalisasi → anotasi → klasifikasi sensitivitas → validasi kualitas → pengajuan approval formal ke Admin → versioning dataset yang disetujui.

### Output
Dataset bersih, terklasifikasi, tervalidasi, dan **berstatus approved** dengan versi tercatat, siap dikonsumsi pipeline fine-tuning.

### Actor
AI/ML Team (eksekusi engineering dataset), Administrator (approval — merangkap fungsi Document Control di MVP).

### Dependencies
Diagram 3 (Data Validation Flow) sebagai gate wajib sebelum approval; kebijakan klasifikasi sensitivitas dokumen (§9.5 master_spec).

### Decision Points
- Apakah dataset lolos validasi kualitas/kepatuhan.
- Apakah Admin menyetujui dataset untuk dipakai training (gate wajib, §9.3 master_spec — tidak boleh dilewati).

### Failure Handling
Dataset gagal validasi → kembali ke cleaning/annotasi. Dataset ditolak Admin → revisi atau dikembalikan ke sumber data; dataset tidak pernah masuk ke fine-tuning tanpa approval eksplisit tercatat.

---

## 3. Data Validation Flow

```mermaid
flowchart TD
    A[Dataset kandidat masuk] --> B[Cek Skema & Format]
    B --> C{Skema valid?}
    C -- Tidak --> X1[Reject -> kembali ke Dataset Engineering]
    C -- Ya --> D[Cek Duplikasi]
    D --> E[Cek Data Sensitif / PII]
    E --> F{Ditemukan data sensitif\ntanpa klasifikasi?}
    F -- Ya --> X2[Reject -> wajib klasifikasi ulang]
    F -- Tidak --> G[Quality Scoring\n_konsistensi gaya, bahasa, struktur_]
    G --> H{Skor memenuhi\nambang minimum?}
    H -- Tidak --> X3[Reject -> kembali ke cleaning/annotasi]
    H -- Ya --> I[Tandai Dataset: VALID]
    I --> J[Lanjut ke Approval Dataset\n#diagram 2]
```

### Input
Dataset mentah hasil cleaning/annotasi dari Dataset Engineering Flow.

### Process
Validasi skema/format → deduplikasi → pemindaian data sensitif/PII → scoring kualitas (konsistensi bahasa, struktur, gaya) → penandaan status valid/tidak valid.

### Output
Dataset dengan status **VALID** (siap diajukan approval) atau **REJECTED** (dengan alasan tercatat) yang dikembalikan ke tahap sebelumnya.

### Actor
AI/ML Team (menjalankan dan meninjau hasil validasi), Administrator (menerima notifikasi bila ditemukan data sensitif tak terklasifikasi, karena merangkap Document Control).

### Dependencies
Kebijakan klasifikasi sensitivitas dokumen (§9.5); hasil cleaning/annotasi dari diagram 2.

### Decision Points
- Validitas skema/format.
- Keberadaan data sensitif tanpa klasifikasi yang benar.
- Pemenuhan ambang skor kualitas minimum.

### Failure Handling
Setiap kegagalan pada titik keputusan mengembalikan dataset ke tahap Dataset Engineering (cleaning/annotasi/klasifikasi ulang) — dataset tidak pernah lanjut ke approval dalam status tidak valid.

---

## 4. Fine-tuning Workflow

```mermaid
sequenceDiagram
    actor MLTeam as AI/ML Team
    participant Registry as Dataset Store (approved)
    participant Local as Lingkungan Lokal (CPU)
    participant Cloud as Cloud GPU (Colab/Kaggle -> GPU rental)
    participant Tracker as Experiment Tracking

    MLTeam->>Local: Siapkan script & konfigurasi training
    Local->>Registry: Tarik dataset berstatus APPROVED
    Registry-->>Local: Dataset approved
    MLTeam->>Cloud: Jalankan job fine-tuning (LoRA/QLoRA)
    Cloud->>Cloud: Training berjalan (GPU-berat)
    Cloud->>Tracker: Log metrik & checkpoint tiap tahap
    alt Training gagal / resource habis
        Cloud-->>MLTeam: Notifikasi gagal
        MLTeam->>Cloud: Resume dari checkpoint terakhir / ulang
    else Training selesai
        Cloud->>Tracker: Simpan checkpoint final & metadata
        Tracker-->>MLTeam: Model candidate siap
        MLTeam->>MLTeam: Lanjut ke Model Evaluation (#diagram 5)
    end
```

### Input
Dataset berstatus APPROVED dari Dataset Store, foundation model terpilih, konfigurasi hyperparameter (LoRA vs QLoRA sesuai evaluasi benchmark, bukan asumsi).

### Process
Persiapan job training di lokal → eksekusi training (fine-tuning) di cloud GPU → logging metrik dan checkpoint ke experiment tracking → penyimpanan model kandidat.

### Output
Model kandidat (checkpoint adapter LoRA/QLoRA) beserta metadata eksperimen, siap untuk evaluasi.

### Actor
AI/ML Team (penuh eksekusi), System (proses training otomatis di cloud GPU).

### Dependencies
Dataset Engineering & Validation (hanya dataset approved yang boleh ditarik); ketersediaan kuota/kredit cloud GPU; hasil riset benchmark LoRA vs QLoRA (§8 master_spec).

### Decision Points
- Pilihan metode: LoRA vs QLoRA (ditentukan oleh hasil benchmark, bukan di dalam flow ini — hanya dieksekusi).
- Resume dari checkpoint vs mulai ulang saat training gagal/terputus.

### Failure Handling
Job gagal/resource habis di tengah jalan → resume dari checkpoint terakhir yang tersimpan di tracker, atau ulangi dengan konfigurasi lebih kecil (Colab/Kaggle free-tier limit) sebelum eskalasi ke GPU rental berbayar.

---

## 5. Model Evaluation Workflow

```mermaid
flowchart TD
    A[Model Kandidat\ndari Fine-tuning] --> B[Automated Benchmark\n_Cloud GPU_]
    B --> C[Human Review\n_kualitas dokumen: struktur, gaya, Bahasa Indonesia formal_]
    C --> D{Lolos ambang\nbenchmark + review?}
    D -- Tidak --> E[Catat gap & rekomendasi perbaikan]
    E --> F[Kembali ke Fine-tuning\n#diagram 4]
    D -- Ya --> G[Tandai Model: APPROVED FOR REGISTRY]
    G --> H[Lanjut ke Model Registry\n#diagram 6]
```

### Input
Model kandidat (checkpoint) dari Fine-tuning Workflow, benchmark set representatif (prompt document-generation), kriteria ambang kelulusan.

### Process
Jalankan automated benchmark di cloud GPU → human review kualitas output terkait kepatuhan format/gaya/Bahasa Indonesia formal → keputusan lolos/tidak.

### Output
Status model: **APPROVED FOR REGISTRY** (lanjut) atau **REJECTED** (kembali ke fine-tuning dengan catatan gap).

### Actor
AI/ML Team (menjalankan benchmark & memimpin review), Administrator (terlibat pada review kualitas dokumen final jika relevan dengan kepatuhan bisnis).

### Dependencies
Model kandidat dari diagram 4; benchmark set & ambang kelulusan yang ditetapkan tim; hasil riset model pembanding (§8 master_spec) sebagai acuan konteks, bukan keputusan final.

### Decision Points
- Apakah model lolos ambang benchmark otomatis **dan** human review kualitas dokumen.

### Failure Handling
Gagal lolos → gap dicatat (misal konsistensi heading, akurasi instruksi) dan dikembalikan ke fine-tuning untuk iterasi (dataset/hyperparameter revisi) — model tidak pernah masuk registry tanpa lolos kedua kriteria.

---

## 6. Model Registry Workflow

```mermaid
stateDiagram-v2
    [*] --> Registered: Model disubmit dari Evaluation (APPROVED)
    Registered --> Staged: Metadata & lineage lengkap tercatat
    Staged --> Approved_for_Deployment: Administrator menyetujui rilis
    Staged --> Rejected: Metadata tidak lengkap / gagal cek lineage
    Rejected --> Registered: Perbaikan metadata
    Approved_for_Deployment --> In_Production: Deployment dieksekusi\n(#diagram 7)
    In_Production --> Deprecated: Versi baru menggantikan
    In_Production --> RolledBack: Rollback dari produksi\n(#diagram 7)
    RolledBack --> Staged: Dikembalikan untuk investigasi
    Deprecated --> Archived: Retensi selesai
    Archived --> [*]
```

### Input
Model APPROVED FOR REGISTRY dari Evaluation Workflow, beserta metadata (dataset version, hyperparameter, metrik evaluasi, lineage).

### Process
Registrasi versi model → verifikasi kelengkapan metadata/lineage → approval rilis oleh Administrator → transisi status sepanjang siklus hidup model (staged → production → deprecated/rolled back → archived).

### Output
Model versi tercatat dengan status jelas dan histori lineage lengkap, menjadi rujukan tunggal untuk deployment dan audit.

### Actor
Administrator (approval rilis — bagian dari fungsi Document Control yang dirangkap di MVP, diperluas ke governance model), AI/ML Team (registrasi & pelengkapan metadata).

### Dependencies
Model Evaluation Workflow (diagram 5); Deployment Workflow (diagram 7) untuk transisi ke produksi/rollback.

### Decision Points
- Kelengkapan metadata/lineage sebelum status Staged → Approved.
- Approval rilis oleh Administrator.

### Failure Handling
Metadata/lineage tidak lengkap → status dikembalikan ke Registered untuk perbaikan sebelum lanjut, model tidak bisa mencapai status Approved for Deployment tanpa lineage lengkap.

---

## 7. Model Deployment Workflow

```mermaid
flowchart TD
    A[Model APPROVED\n_dari Registry_] --> B[Deploy sebagai Canary\n_porsi kecil traffic_]
    B --> C[Monitoring Canary\n_latency, error rate, kualitas output_]
    C --> D{Metrik canary\nsesuai ambang?}
    D -- Tidak --> E[Rollback\nkembalikan 100% traffic ke model sebelumnya]
    E --> F[Update status di Registry: RolledBack]
    D -- Ya --> G[Rollout Bertahap\n_naikkan persentase traffic_]
    G --> H{Full rollout\nstabil?}
    H -- Tidak --> E
    H -- Ya --> I[Model jadi versi Production penuh]
    I --> J[Lanjut ke Model Monitoring\n#diagram 8]
```

### Input
Model berstatus Approved for Deployment dari Model Registry.

### Process
Rilis canary ke sebagian kecil traffic → observasi metrik → keputusan lanjut rollout bertahap atau rollback → status final production penuh.

### Output
Model berjalan sebagai versi production (full rollout) atau rollback sukses ke versi sebelumnya tanpa gangguan layanan signifikan.

### Actor
AI/ML Team (eksekusi deployment teknis), Administrator (approval rilis skala penuh), System (monitoring otomatis metrik canary).

### Dependencies
Model Registry Workflow (diagram 6); Model Monitoring Workflow (diagram 8) untuk metrik produksi berkelanjutan.

### Decision Points
- Apakah metrik canary memenuhi ambang sebelum rollout diperluas.
- Apakah full rollout stabil sebelum ditetapkan sebagai versi production utama.

### Failure Handling
Metrik canary atau rollout tidak memenuhi ambang pada tahap apa pun → rollback otomatis ke versi model sebelumnya yang masih serving, status model baru diubah menjadi RolledBack di registry untuk investigasi — tidak ada periode tanpa layanan.

---

## 8. Model Monitoring Workflow

```mermaid
flowchart TD
    A[Model Production] --> B[Koleksi Metrik Real-time]
    B --> B1[Latency & Error Rate]
    B --> B2[Data/Concept Drift]
    B --> B3[Feedback Kualitas dari User/Admin]
    B1 --> C[Evaluasi terhadap Ambang]
    B2 --> C
    B3 --> C
    C --> D{Dalam ambang\nnormal?}
    D -- Ya --> A
    D -- Tidak --> E{Insiden operasional\natau degradasi kualitas model?}
    E -- Insiden operasional --> F[Alert ke AI/ML Team\n_tindakan cepat/rollback_]
    F --> G[Lanjut ke Deployment Workflow\n#diagram 7 — rollback]
    E -- Degradasi kualitas/drift --> H[Trigger Model Retraining\n#diagram 10]
```

### Input
Trafik inference produksi, log latency/error, sinyal drift data/konsep, feedback kualitas dari pengguna/Administrator.

### Process
Koleksi metrik berkelanjutan → evaluasi terhadap ambang yang ditetapkan → klasifikasi anomali (insiden operasional vs degradasi kualitas model) → routing ke tindakan yang sesuai.

### Output
Status kesehatan model terkini; trigger otomatis ke rollback (insiden operasional) atau ke retraining (degradasi kualitas/drift).

### Actor
System (koleksi & evaluasi metrik otomatis), AI/ML Team (menindaklanjuti alert), Administrator (menerima laporan degradasi kualitas terkait kepatuhan dokumen).

### Dependencies
Model Deployment Workflow (diagram 7) sebagai sumber model yang dimonitor; Model Retraining Workflow (diagram 10) sebagai tujuan trigger.

### Decision Points
- Apakah metrik berada dalam ambang normal.
- Jika tidak, apakah ini insiden operasional (rollback) atau degradasi kualitas/drift model (retraining).

### Failure Handling
Insiden operasional → rollback cepat via diagram 7. Drift/degradasi kualitas bertahap → tidak langsung rollback, melainkan trigger siklus retraining terkontrol (diagram 10) agar tidak mengganggu layanan yang masih berjalan.

---

## 9. RAG Pipeline

```mermaid
flowchart TD
    subgraph INGEST["Ingestion (Offline)"]
        A[Dokumen Internal\n_SOP, kebijakan, referensi_] --> B[Klasifikasi Sensitivitas\n+ Tag ACL/Permission]
        B --> C[Chunking & Embedding]
        C --> D[Simpan ke Vector DB\n_pgvector MVP / Qdrant Phase 2_]
    end

    subgraph RUNTIME["Query-time (Online)"]
        E[User mengirim prompt] --> F[Ambil Identitas & Permission User]
        F --> G[Retrieval dari Vector DB\n_similarity search_]
        D -. basis pencarian .-> G
        G --> H[Filter ACL di Backend\n_wajib: hanya dokumen yang\nuser berhak akses_]
        H --> I{Ada konteks\nyang lolos filter?}
        I -- Tidak --> J[Jawab tanpa konteks RAG\natau beri tahu tidak ada akses]
        I -- Ya --> K[Augmentasi Prompt\n_konteks tervalidasi + query_]
        K --> L[LLM Generation\n_model fine-tuned_]
        L --> M[Response ke User]
        J --> M
    end
```

### Input
Dokumen internal terklasifikasi dengan metadata ACL, query/prompt dari Authorized User beserta identitas & hak aksesnya.

### Process
**Offline**: klasifikasi sensitivitas dan penandaan ACL dokumen → chunking & embedding → penyimpanan di vector DB. **Online**: retrieval kandidat konteks berdasarkan similarity → **filter ACL di backend** (bukan hanya system prompt) → augmentasi prompt dengan konteks yang lolos filter → generasi jawaban oleh model fine-tuned.

### Output
Jawaban yang disusun berdasarkan konteks yang memang berhak diakses user, atau jawaban tanpa konteks RAG bila tidak ada dokumen yang lolos filter izin.

### Actor
Authorized User (pengguna query), Administrator (mengelola klasifikasi & hak akses dokumen), System/AI Model (retrieval & generation).

### Dependencies
Knowledge base yang sudah terklasifikasi dan diindeks; mekanisme RBAC/ACL backend (§9.4, §9.8 master_spec); model fine-tuned yang sudah production.

### Decision Points
- Apakah ada konteks yang lolos filter ACL setelah retrieval — menentukan apakah jawaban memakai augmentasi RAG atau tidak.

### Failure Handling
Tidak ada dokumen relevan yang lolos filter izin → sistem tetap menjawab tanpa konteks RAG (fallback ke pengetahuan umum model) atau secara eksplisit memberi tahu user bahwa tidak ada dokumen yang dapat diakses — **tidak pernah** meloloskan dokumen di luar hak akses user hanya karena relevan secara semantik.

---

## 10. Model Retraining Workflow

```mermaid
flowchart TD
    A{Trigger Retraining} --> A1[Monitoring: drift/degradasi\nterdeteksi #diagram 8]
    A --> A2[Dataset baru/revisi tersedia\ndan disetujui]
    A --> A3[Jadwal retraining berkala]
    A1 --> B[Keputusan Retraining\n_AI/ML Team + Administrator_]
    A2 --> B
    A3 --> B
    B --> C{Retraining\ndisetujui?}
    C -- Tidak --> D[Catat alasan, pantau terus\n_tetap di diagram 8_]
    C -- Ya --> E[Dataset Engineering ulang/update\n#diagram 2 — termasuk approval]
    E --> F[Fine-tuning\n#diagram 4]
    F --> G[Model Evaluation\n#diagram 5]
    G --> H{Model baru lebih baik\ndari versi production saat ini?}
    H -- Tidak --> D
    H -- Ya --> I[Model Registry\n#diagram 6]
    I --> J[Deployment Canary\n#diagram 7]
    J --> K{Canary sukses?}
    K -- Tidak --> L[Rollback ke versi production lama]
    K -- Ya --> M[Model baru menggantikan versi production]
```

### Input
Sinyal trigger (drift/degradasi dari monitoring, dataset baru yang sudah disetujui, atau jadwal retraining berkala), model production saat ini sebagai baseline pembanding.

### Process
Evaluasi kelayakan retraining → (bila disetujui) ulangi siklus Dataset Engineering → Fine-tuning → Evaluation → bandingkan dengan model production saat ini → jika lebih baik, lanjut ke Registry dan Deployment canary.

### Output
Model production baru yang terbukti lebih baik dari versi sebelumnya, atau keputusan untuk menunda retraining dengan versi production lama tetap berjalan.

### Actor
AI/ML Team (eksekusi siklus retraining), Administrator (approval dataset & keputusan retraining, approval rilis).

### Dependencies
Model Monitoring Workflow (diagram 8) sebagai sumber trigger utama; seluruh siklus diagram 2, 4, 5, 6, 7 dijalankan ulang di sini.

### Decision Points
- Apakah retraining disetujui untuk dijalankan.
- Apakah model baru hasil retraining lebih baik dari model production saat ini.
- Apakah canary deployment model baru sukses.

### Failure Handling
Retraining tidak disetujui atau model baru tidak lebih baik → versi production lama tetap berjalan tanpa gangguan, alasan dicatat untuk evaluasi berikutnya. Canary model baru gagal → rollback ke versi production lama (konsisten dengan diagram 7) — model lama tidak pernah digantikan tanpa bukti model baru lebih baik dan stabil di canary.

---

## Ringkasan Keterkaitan Antar-Diagram

| Diagram | Menerima dari | Mengirim ke |
|---|---|---|
| 1. End-to-End ML Lifecycle | — (overview) | Semua diagram 2–10 |
| 2. Dataset Engineering | 10 (retraining loop) | 3, 4 |
| 3. Data Validation | 2 | 2 (approval gate) |
| 4. Fine-tuning | 2 (dataset approved) | 5 |
| 5. Model Evaluation | 4 | 4 (reject) / 6 (approve) |
| 6. Model Registry | 5 | 7 |
| 7. Model Deployment | 6 | 8; rollback balik ke registry (6) |
| 8. Model Monitoring | 7 | 7 (insiden) / 10 (drift) |
| 9. RAG Pipeline | KB internal (independen dari siklus training) | Response ke user (runtime aplikasi) |
| 10. Model Retraining | 8 (trigger) | 2, lalu ulang 4–7 |

**Catatan traceability**: dokumen ini merujuk keputusan di `master_spec.md` §4 (Decision Log), §7 (Hardware Reality), §8 (Riset Teknis), dan §9 (Security & Data Governance). Detail implementasi teknis (tools, konfigurasi, struktur folder, perintah eksekusi) didokumentasikan di `ML_Pipeline.md` agar tidak terjadi duplikasi.
