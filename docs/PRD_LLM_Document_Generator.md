# PRD — DocuGen AI: LLM Document Generator Application

**Dokumen:** PRD_LLM_Document_Generator.md (Dokumen Turunan #3)
**Status:** Draft Final — mengacu penuh pada `master_spec.md` sebagai single source of truth.
**Traceability:** Requirement ID pada dokumen ini (`FR-xxx`, `NFR-xxx`) dirujuk oleh `Project_Requirements.md` (Dokumen #7) dan tidak diduplikasi di dokumen lain. Status lifecycle dokumen (Draft → ... → Archived) mengacu pada `PRD_Document_Control_SaaS.md` (Dokumen #2); detail arsitektur ML mengacu pada `ML_Deployment_Strategy.md` (Dokumen #4).

---

## 1. Product Overview

DocuGen AI adalah **aplikasi AI percakapan internal (private enterprise application)** dengan antarmuka mirip ChatGPT/Claude/Gemini, di mana **kapabilitas utama yang dioptimalkan adalah pembuatan, pemahaman, dan pengeditan dokumen berbasis LLM**, bukan sekadar chatbot tanya-jawab umum. Aplikasi ini bukan wrapper API eksternal murni — model inference dapat berjalan self-hosted (fine-tuned open-weight model) sesuai keputusan pada `master_spec.md` §4.

Pengguna berinteraksi melalui chat untuk meminta pembuatan dokumen (misal: SOP, surat, laporan, proposal), mereview hasil generate di document workspace, melakukan revisi melalui instruksi bahasa natural atau editor langsung, lalu mengekspor dokumen final ke DOCX/PDF. Dokumen hasil AI tidak pernah otomatis menjadi dokumen resmi — tetap melalui human review sebelum disetujui (lihat §29 Integration Requirements).

## 2. Product Vision

Menjadi *AI document co-pilot* internal yang memungkinkan karyawan membuat dokumen formal berbahasa Indonesia secara konsisten, cepat, dan sesuai format baku perusahaan, dengan jaminan bahwa pengetahuan internal (SOP, kebijakan, template) yang digunakan AI selalu up-to-date melalui RAG, dan bahwa perilaku/gaya penulisan model konsisten melalui fine-tuning — bukan sebaliknya (lihat pembedaan fine-tuning vs RAG di `master_spec.md` §3). Visi ini tidak boleh direduksi menjadi "chatbot generik ditambah fitur ekspor"; document generation adalah nilai inti produk.

## 3. Product Background

Organisasi saat ini tidak memiliki alat internal untuk menghasilkan dokumen formal secara konsisten dan cepat; proses penulisan dokumen (SOP, surat, laporan) manual dan tidak terstandardisasi. Dataset dokumen historis perusahaan belum tersedia dalam volume cukup untuk fine-tuning (lihat `master_spec.md` §4 baris "Dataset awal"), sehingga pipeline data dirancang untuk mulai dari dataset publik + sintetis, namun tetap siap menyerap dokumen historis internal begitu tersedia. Produk ini dibangun bertahap sesuai ketersediaan hardware (lihat `master_spec.md` §7), bukan sebagai blueprint portofolio semata — MVP harus benar-benar dapat dioperasikan oleh pengguna internal terbatas.

## 4. Business Objectives

| # | Objective |
|---|---|
| B1 | Mengurangi waktu pembuatan dokumen formal internal (SOP, surat, laporan, proposal) melalui generate-dan-revisi berbasis AI dibanding penulisan manual penuh. |
| B2 | Menstandardisasi format dan gaya penulisan dokumen Bahasa Indonesia formal lintas departemen. |
| B3 | Menjadikan pengetahuan internal (SOP, kebijakan, dokumen referensi) dapat diakses secara kontekstual oleh AI tanpa mengekspos data ke pihak eksternal, sesuai baseline keamanan `master_spec.md` §9. |
| B4 | Menjaga akuntabilitas: setiap dokumen hasil AI harus tertelusur (siapa generate, revisi apa, kapan disetujui) sebelum menjadi dokumen resmi. |
| B5 | Membangun dasar aplikasi yang dapat diperluas ke Phase 2 (department-based access, resource-level permission, Document Control sebagai role terpisah) tanpa perlu re-arsitektur besar. |

## 5. Technical Objectives

| # | Objective |
|---|---|
| T1 | Menyediakan conversational UI (Next.js) yang responsif dan familiar bagi pengguna yang sudah terbiasa dengan ChatGPT/Claude/Gemini. |
| T2 | Memisahkan backend aplikasi (FastAPI) dari inference service model, agar model dapat diganti/di-upgrade tanpa mengubah logika aplikasi (lihat `master_spec.md` §4 "Backend language"). |
| T3 | Mengimplementasikan RAG permission-aware di atas vector DB (pgvector MVP), sehingga retrieval dokumen internal selalu memfilter berdasarkan hak akses user di level backend, bukan hanya prompt. |
| T4 | Menegakkan RBAC di API layer (backend authorization wajib, system prompt tidak pernah menjadi pengganti otorisasi — `master_spec.md` §9 poin 8). |
| T5 | Menyediakan document editor terstruktur (bukan plain text) yang mendukung format dokumen bisnis standar (heading, paragraf, list, tabel) dengan autosave dan version history. |
| T6 | Menyediakan audit trail penuh atas akses sensitif Admin dan perubahan dokumen. |
| T7 | Merancang skema data dan API agar kompatibel dengan ekspansi Phase 2 (department-based ACL, resource-level permission, role Document Control terpisah) tanpa breaking change mayor. |

## 6. Target Users

Sesuai `master_spec.md` §5, produk ini **private, closed deployment, no public registration** — seluruh akun dibuat dan dikelola oleh Admin. Target pengguna:
- Karyawan internal lintas departemen yang rutin membuat/mereview dokumen formal (SOP, surat, laporan, proposal) — disebut **Authorized User**.
- Staf administratif/IT internal yang mengelola akun, konfigurasi AI, knowledge base, template, dan (pada MVP) persetujuan dokumen — disebut **Administrator**.
- Tidak ada target pengguna eksternal/publik pada MVP maupun Phase 2 yang direncanakan saat ini.

## 7. User Roles

| Role | Deskripsi | Sumber |
|---|---|---|
| **Administrator** | Mengelola user/role/permission, dataset & model (governance level), knowledge base, template, audit log, usage analytics, konfigurasi AI, monitoring sistem; pada MVP juga merangkap fungsi Document Control (approval dokumen). | `master_spec.md` §5.1, §4 |
| **Authorized User** | Login, menggunakan conversational AI, generate/edit/upload dokumen, mengakses riwayat percakapan & dokumen miliknya, mengekspor dokumen, menggunakan template, mengakses knowledge base sesuai permission yang diberikan Admin. | `master_spec.md` §5.2 |
| **System/AI Model** (aktor non-manusia) | Menghasilkan draft dokumen dan respons chat; tidak memiliki otoritas approval apa pun. | `master_spec.md` §5.4 |

Catatan Phase 2 (bukan scope MVP): Department-Based Access Control dan role Document Control terpisah dari Administrator — lihat §32 Product Roadmap.

## 8. User Journey

### 8.1 Journey A — Generate Dokumen Baru dari Prompt

1. Authorized User login, klik **"New Chat"** di sidebar.
2. User menuliskan prompt di chat input, misal: *"Buatkan draft SOP pengajuan cuti karyawan, format sesuai template HR."*
3. Sistem (opsional) menyarankan template relevan dari Template Management; user dapat memilih atau mengabaikan.
4. Backend melakukan retrieval RAG (jika relevan: SOP/kebijakan terkait yang user punya akses) dan mengirim context + prompt ke inference service.
5. AI menghasilkan draft dokumen; respons ditampilkan di dua tempat sekaligus: ringkasan percakapan di chat panel, dan dokumen terstruktur penuh di **Document Workspace** di sisi kanan.
6. Dokumen otomatis tersimpan sebagai draft baru dengan status lifecycle **"Draft"** (lihat §29) dan version 1 tercatat di version history.
7. User mereview di Document Workspace; dapat langsung mengedit manual, atau kembali ke chat untuk meminta revisi (lanjut ke Journey B).
8. User mengekspor ke DOCX/PDF kapan saja, atau menyerahkan dokumen untuk approval (fungsi Document Control, dijalankan Admin pada MVP).

### 8.2 Journey B — Revisi Dokumen Berdasarkan Instruksi

1. User membuka dokumen existing dari **Conversation History** atau **Document Workspace**.
2. User mengetik instruksi revisi di chat input, misal: *"Ubah bagian 'Prosedur Pengajuan' jadi numbered list, dan perpendek bagian 'Latar Belakang' jadi 2 paragraf."*
3. Sistem mengidentifikasi bagian dokumen yang dirujuk (selective regeneration — lihat §15), dan hanya meregenerasi bagian tersebut, bukan seluruh dokumen.
4. Perubahan ditampilkan di Document Workspace dengan indikasi bagian yang berubah; chat panel menampilkan ringkasan perubahan yang dilakukan AI.
5. Sistem menyimpan versi baru di version history (increment version), autosave berjalan otomatis.
6. User dapat membandingkan versi sebelumnya vs sekarang, melakukan undo bila hasil revisi tidak sesuai, atau melanjutkan iterasi revisi lain.
7. Setelah user puas, dokumen tetap berstatus "Draft" sampai diajukan dan disetujui melalui proses Document Control (§29).

## 9. Functional Requirements

| Requirement ID | Requirement Name | Description | Priority | Actor | Preconditions | Expected Result | Acceptance Criteria |
|---|---|---|---|---|---|---|---|
| FR-001 | Login Pengguna | Authorized User dan Admin login menggunakan akun yang dibuat Admin (no self-registration). | Must | User, Admin | Akun sudah dibuat Admin | User berhasil masuk ke dashboard chat sesuai role | Login sukses mengarahkan ke home chat; login gagal menampilkan error tanpa membocorkan apakah username/password yang salah |
| FR-002 | Memulai Chat Baru | User dapat memulai sesi chat baru kapan saja via tombol "New Chat". | Must | User | User sudah login | Sesi chat baru terbuka, kosong, siap menerima prompt | Klik "New Chat" membuat conversation baru dengan ID unik dan muncul di Conversation History |
| FR-003 | Mengirim Prompt Percakapan | User mengirim pesan teks ke AI dan menerima respons percakapan umum (non-dokumen) maupun terkait dokumen. | Must | User | Sesi chat aktif | AI merespons dalam sesi yang sama, respons tersimpan di chat history | Respons AI muncul di UI dalam batas waktu yang ditentukan NFR-001; pesan tersimpan permanen di DB |
| FR-004 | Streaming Respons AI | Respons AI ditampilkan secara streaming (token-by-token), bukan menunggu respons penuh. | Should | User | Sesi chat aktif | User melihat respons muncul bertahap, dapat menghentikan generasi di tengah jalan | Tombol "Stop generating" menghentikan stream dan menyimpan output parsial |
| FR-005 | Menyimpan Riwayat Percakapan | Setiap percakapan tersimpan otomatis dan dapat diakses kembali dari sidebar. | Must | User | User sudah login | Daftar percakapan muncul di sidebar, diurutkan terbaru di atas | Riwayat tetap ada setelah logout/login ulang; percakapan dapat dibuka dan melanjutkan konteks |
| FR-006 | Pencarian Riwayat Percakapan | User dapat mencari percakapan berdasarkan judul/isi. | Should | User | Minimal 1 percakapan tersimpan | Hasil pencarian menampilkan percakapan relevan | Query pencarian mengembalikan hasil yang mengandung kata kunci pada judul atau isi pesan |
| FR-007 | Menghapus/Mengganti Nama Percakapan | User dapat menghapus percakapan atau mengganti judulnya. | Should | User | Percakapan milik user tersebut | Percakapan terhapus permanen atau judul berubah | Aksi delete menghapus dari daftar dan tidak bisa di-undo; rename langsung ter-reflect di sidebar |
| FR-008 | Generate Dokumen dari Prompt | User meminta AI membuat dokumen baru dari instruksi bahasa natural. | Must | User | Sesi chat aktif | Dokumen terstruktur baru muncul di Document Workspace, tersimpan dengan status "Draft" | Dokumen baru punya judul, isi terstruktur (heading/paragraf/list sesuai konteks), dan version 1 tercatat |
| FR-009 | Generate Dokumen dari Template | User memilih template sebagai basis sebelum/selama prompting. | Should | User | Minimal 1 template tersedia & user memiliki akses | Draft dokumen mengikuti struktur template yang dipilih | Struktur heading dokumen hasil generate sesuai urutan section pada template |
| FR-010 | Regenerasi Dokumen Penuh | User meminta AI membuat ulang seluruh dokumen dari awal dengan instruksi baru. | Should | User | Dokumen sudah ada | Dokumen digantikan versi baru penuh, versi lama tetap tersimpan di history | Version history bertambah; isi dokumen versi terbaru sepenuhnya baru sesuai instruksi |
| FR-011 | Regenerasi Dokumen Sebagian (Selective Regeneration) | User meminta AI merevisi bagian tertentu saja (section/paragraf/tabel) tanpa mengubah bagian lain. | Must | User | Dokumen sudah ada, bagian yang dirujuk teridentifikasi | Hanya bagian yang dirujuk berubah; bagian lain tetap identik | Diff antar versi menunjukkan perubahan hanya pada section yang diminta |
| FR-012 | Format Heading Terstruktur | Document editor mendukung heading berjenjang (H1-H3) yang konsisten dengan struktur dokumen formal. | Must | User | Document Workspace terbuka | Heading dapat dibuat/diubah levelnya, tampil konsisten di editor dan hasil ekspor | Heading level 1-3 ter-render berbeda secara visual dan ikut terbawa saat ekspor DOCX/PDF |
| FR-013 | Format Paragraf | Editor mendukung paragraf teks biasa dengan formatting dasar (bold, italic, underline). | Must | User | Document Workspace terbuka | Teks paragraf dapat diformat dan tersimpan sesuai formatting | Formatting tetap konsisten setelah reload halaman dan setelah ekspor |
| FR-014 | Bullet List | Editor mendukung bullet list (unordered list) dengan nested level. | Must | User | Document Workspace terbuka | User dapat membuat/mengubah list jadi bullet, termasuk sub-bullet | Bullet list ter-render dengan indentasi benar di editor dan ekspor |
| FR-015 | Numbered List | Editor mendukung numbered list (ordered list) dengan penomoran otomatis dan nested level. | Must | User | Document Workspace terbuka | Penomoran otomatis menyesuaikan saat item ditambah/dihapus/diurutkan ulang | Urutan angka tetap konsisten setelah insert/delete item di tengah list |
| FR-016 | Tabel | Editor mendukung pembuatan dan pengeditan tabel (tambah/hapus baris & kolom, isi sel). | Must | User | Document Workspace terbuka | Tabel dapat dibuat, diedit, dan dihapus dalam dokumen | Tabel tampil dengan border dan struktur sel yang benar di editor dan saat diekspor ke DOCX/PDF |
| FR-017 | Undo/Redo | Editor mendukung undo dan redo untuk setiap aksi edit manual maupun hasil AI. | Must | User | Document Workspace terbuka, minimal 1 aksi edit dilakukan | Aksi edit dapat dibatalkan/diulang tanpa korupsi struktur dokumen | Undo mengembalikan state tepat sebelum aksi terakhir; redo mengembalikan state setelahnya; tersedia minimal multi-step undo |
| FR-018 | Autosave | Perubahan pada dokumen (manual maupun hasil AI) tersimpan otomatis tanpa aksi "Save" eksplisit dari user. | Must | User, System | Document Workspace terbuka | Dokumen tersimpan ke backend secara periodik/on-change dengan indikator status simpan | Indikator "Saved"/"Saving..." muncul di UI; menutup tab tanpa klik save tidak menghilangkan perubahan terakhir |
| FR-019 | Version History | Setiap perubahan signifikan (generate, regenerasi, edit manual tersimpan) membuat versi baru yang dapat dilihat dan dibandingkan. | Must | User | Dokumen memiliki minimal 2 versi | User dapat melihat daftar versi dengan timestamp dan actor (user/AI), serta membandingkan isi antar versi | Daftar versi terurut kronologis; memilih 2 versi menampilkan diff isi |
| FR-020 | Restore Versi Sebelumnya | User dapat mengembalikan dokumen ke versi tertentu dari version history. | Should | User | Dokumen memiliki minimal 2 versi | Dokumen aktif berubah menjadi identik dengan versi yang dipilih, tercatat sebagai versi baru (bukan menimpa history) | Restore menambah entri version history baru, versi-versi sebelumnya tetap dapat diakses |
| FR-021 | Upload Dokumen | User dapat mengunggah dokumen (DOCX/PDF/TXT) untuk dijadikan referensi dalam chat atau sumber knowledge base. | Must | User | User login, memiliki permission upload | File tersimpan di object storage, status pemrosesan ditampilkan | Upload berhasil menampilkan nama file dan status "Processing"/"Ready"; format tidak didukung ditolak dengan pesan jelas |
| FR-022 | Validasi & Klasifikasi Upload | Setiap dokumen yang diunggah wajib diberi klasifikasi sensitivitas (Public/Internal/Confidential/Restricted) sebelum dapat digunakan sebagai sumber RAG. | Must | User, Admin | Dokumen sudah diunggah | Dokumen tanpa klasifikasi tidak masuk index RAG | Dokumen berstatus "unclassified" tidak muncul sebagai hasil retrieval RAG kepada siapa pun |
| FR-023 | Document Understanding (Ringkasan) | AI dapat merangkum/menjawab pertanyaan atas dokumen yang diunggah dalam sesi chat yang sama. | Must | User | Dokumen sudah diunggah dan berstatus "Ready" | AI menjawab berdasarkan isi dokumen yang diunggah, bukan halusinasi | Jawaban AI mengutip/merujuk bagian dokumen sumber yang relevan |
| FR-024 | Ekstraksi Terstruktur dari Dokumen | AI dapat mengekstrak informasi terstruktur (misal poin aksi, tabel data) dari dokumen yang diunggah atas permintaan user. | Should | User | Dokumen sudah diunggah dan diproses | Hasil ekstraksi ditampilkan terstruktur (list/tabel) di chat atau disalin ke document editor | Hasil ekstraksi dapat disisipkan langsung ke Document Workspace sebagai konten terstruktur |
| FR-025 | Pengelolaan Knowledge Base | Admin dapat menambah, mengedit, menghapus, dan mengatur klasifikasi entri knowledge base (SOP, kebijakan, dokumen referensi). | Must | Admin | Admin login | Entri KB tersimpan dengan metadata klasifikasi dan permission akses | Entri KB baru langsung tersedia untuk retrieval sesuai permission yang diset |
| FR-026 | Permission Knowledge Base per Entri | Admin dapat mengatur siapa (user/role) yang berhak mengakses tiap entri KB. | Must | Admin | Entri KB sudah ada | Permission tersimpan dan ditegakkan saat retrieval | User tanpa permission tidak pernah menerima entri tersebut dalam hasil retrieval RAG, termasuk tidak di log context yang terekspos ke user |
| FR-027 | Retrieval RAG Permission-Aware | Saat AI menjawab/generate dokumen, sistem melakukan retrieval dari vector DB dan memfilter hasil berdasarkan permission user di level backend sebelum masuk context model. | Must | User, System | User mengajukan prompt yang relevan dengan KB | Hanya dokumen yang user berhak akses yang masuk context inference | Uji coba dengan 2 user berbeda permission terhadap entri KB sama menghasilkan context retrieval yang berbeda sesuai permission masing-masing (merujuk `master_spec.md` §9 poin 4) |
| FR-028 | Sitasi Sumber RAG | Saat AI menggunakan hasil retrieval KB dalam jawaban/dokumen, sistem menampilkan sumber yang dirujuk. | Should | User | Retrieval RAG menghasilkan minimal 1 dokumen relevan | Jawaban/dokumen menyertakan referensi ke sumber KB yang dipakai | User dapat klik sumber untuk melihat entri KB asal (jika user punya akses) |
| FR-029 | Pengelolaan Template Dokumen | Admin dapat membuat, mengedit, menghapus, dan mengaktifkan/nonaktifkan template dokumen (struktur section baku per jenis dokumen). | Must | Admin | Admin login | Template tersimpan dan tersedia dipilih user saat generate dokumen | Template nonaktif tidak muncul di pilihan user, tapi dokumen yang sudah dibuat dari template tersebut tetap valid |
| FR-030 | Kategori & Pencarian Template | User dapat menelusuri template berdasarkan kategori (misal SOP, Surat, Laporan, Proposal) saat memulai dokumen baru. | Should | User | Minimal 1 template per kategori tersedia | Daftar template terfilter sesuai kategori yang dipilih | Pencarian/kategori filter menampilkan hanya template yang relevan dan yang user punya akses |
| FR-031 | Ekspor Dokumen ke DOCX/PDF | User dapat mengekspor dokumen dari Document Workspace ke format DOCX dan PDF, mempertahankan formatting (heading, list, tabel). | Must | User | Dokumen sudah ada di Document Workspace | File hasil ekspor terunduh dengan formatting identik dengan tampilan editor | Dibuka di Microsoft Word/Adobe Reader, heading/list/tabel tampil sesuai struktur di editor |
| FR-032 | Pengelolaan Akun User (CRUD) | Admin dapat membuat, mengedit, menonaktifkan (bukan hard-delete demi audit), dan menghapus akun Authorized User. | Must | Admin | Admin login | Perubahan akun langsung berlaku pada akses login user terkait | Akun yang dinonaktifkan tidak bisa login tapi riwayatnya tetap tersimpan untuk audit |
| FR-033 | Reset Kredensial User | Admin dapat memicu reset password untuk akun user (self-service reset oleh user tidak tersedia karena no public registration/self-service di MVP). | Must | Admin | Akun user sudah ada | User menerima mekanisme reset kredensial yang aman | Password lama tidak lagi valid setelah reset berhasil dikonfirmasi |
| FR-034 | Pengelolaan Role & Permission (RBAC) | Admin dapat menetapkan role (Administrator/Authorized User) ke akun dan mengatur permission granular terkait akses KB/template/fitur sesuai kapabilitas RBAC MVP. | Must | Admin | Admin login | Perubahan role/permission langsung berlaku pada sesi/akses berikutnya user terkait | User dengan role diubah kehilangan/mendapat akses fitur sesuai role baru pada request API berikutnya |
| FR-035 | Admin Dashboard — Usage Analytics | Admin dapat melihat metrik penggunaan: jumlah percakapan, dokumen dibuat, user aktif, dalam rentang waktu tertentu. | Should | Admin | Admin login, data histori tersedia | Dashboard menampilkan grafik/angka metrik sesuai filter waktu | Metrik yang ditampilkan konsisten dengan data mentah di database untuk periode yang sama |
| FR-036 | Admin Dashboard — Audit Log | Admin dapat melihat log audit atas aksi sensitif (akses dokumen Confidential/Restricted, perubahan permission, perubahan role, approval dokumen). | Must | Admin | Minimal 1 aksi sensitif terjadi | Log menampilkan actor, aksi, target, timestamp, dan tidak dapat diedit/dihapus Admin itu sendiri | Setiap akses sensitif Admin tercatat tanpa kemungkinan dihapus dari UI (merujuk `master_spec.md` §9 poin 6) |
| FR-037 | Pengajuan Dokumen untuk Approval | User dapat mengajukan dokumen hasil generate/edit untuk disetujui, mengubah status dari "Draft" menuju lifecycle Document Control. | Must | User, Admin | Dokumen berstatus "Draft" dan sudah direview user | Status dokumen berubah sesuai state lifecycle Document Control (lihat §29), approval dijalankan oleh Admin pada MVP | Dokumen berstatus "Draft" tidak dapat diekspor sebagai "dokumen resmi final" tanpa melalui tahap approval yang tercatat |

Total: **37 Functional Requirements** (FR-001 s/d FR-037), memenuhi minimum 25 FR yang disyaratkan dan mencakup seluruh kapabilitas inti pada `master_spec.md` §2 (Conversational UI, document generation/understanding/editor/management, knowledge base + RAG, user & RBAC management, document export, AI monitoring/audit).

## 10. Non-Functional Requirements

| Requirement ID | Requirement Name | Description | Priority | Actor | Preconditions | Expected Result | Acceptance Criteria |
|---|---|---|---|---|---|---|---|
| NFR-001 | Latensi Respons Chat | Respons awal AI (first token) untuk percakapan non-dokumen muncul dalam waktu wajar pada beban normal. | Must | System | Inference service tersedia dan sehat | User melihat indikasi respons mulai (streaming/typing indicator) tanpa delay berlebihan | First token muncul ≤ 3 detik pada beban normal (1 concurrent request per GPU inference unit) di lingkungan uji ditentukan `ML_Deployment_Strategy.md` |
| NFR-002 | Latensi Generate Dokumen | Generate dokumen penuh (bukan chat singkat) selesai dalam waktu wajar mengingat ukuran dokumen umum. | Must | System | Prompt generate dokumen diterima | Dokumen terstruktur selesai dibuat dan tampil di Document Workspace | Dokumen 1-5 halaman selesai digenerate ≤ 60 detik pada kondisi beban normal |
| NFR-003 | Skalabilitas Pengguna Bersamaan | Sistem mendukung jumlah pengguna bersamaan sesuai skala enterprise internal (bukan skala publik massal). | Must | System | Deployment MVP sudah berjalan | Sistem tetap responsif tanpa degradasi signifikan pada jumlah user bersamaan target | Sistem menangani minimal 50 concurrent active sessions pada MVP tanpa error rate di atas 1% |
| NFR-004 | Ketersediaan Sistem (Availability) | Aplikasi tersedia untuk digunakan pada jam kerja dengan downtime terjadwal minimal. | Must | System | Infrastruktur produksi berjalan | Uptime terukur dan dilaporkan di monitoring | Target uptime minimal 99% pada jam kerja (di luar maintenance terjadwal yang diinformasikan) |
| NFR-005 | Keamanan Otorisasi Backend | Seluruh endpoint API menegakkan RBAC di backend, tidak bergantung pada UI atau system prompt sebagai kontrol akses. | Must | System | — | Request API tanpa permission memadai ditolak di level API | Request dengan token valid tapi tanpa permission relevan menghasilkan HTTP 403, diverifikasi via automated test, bukan hanya UI yang menyembunyikan tombol (merujuk `master_spec.md` §9 poin 8) |
| NFR-006 | Perlindungan Data & Privasi | Data percakapan dan dokumen upload tidak otomatis dipakai sebagai training data tanpa approval eksplisit. | Must | System, Admin | Dataset training akan disusun | Tidak ada proses otomatis yang mengirim data produksi ke pipeline training tanpa approval tercatat | Audit menunjukkan setiap dataset training memiliki record approval eksplisit (merujuk `master_spec.md` §9 poin 1-3) |
| NFR-007 | Klasifikasi & Kontrol Data Sensitif | Dokumen sensitif wajib terklasifikasi dan tidak boleh dikirim ke API eksternal pihak ketiga tanpa persetujuan kebijakan formal. | Must | System, Admin | Integrasi API eksternal (jika ada) dikonfigurasi | Default konfigurasi adalah deny pengiriman data ke API eksternal | Percobaan mengirim dokumen berklasifikasi Confidential/Restricted ke integrasi eksternal gagal tanpa approval policy tercatat |
| NFR-008 | Audit Trail Tidak Dapat Dimanipulasi | Log audit atas akses sensitif tidak dapat diedit atau dihapus melalui antarmuka aplikasi, termasuk oleh Admin. | Must | System | Minimal 1 entri audit log tercatat | Entri log bersifat append-only dari sisi aplikasi | Tidak ada endpoint API yang mengizinkan update/delete pada tabel audit log dari role apa pun |
| NFR-009 | Usabilitas Antarmuka | UI mengikuti pola interaksi yang familiar (ChatGPT/Claude/Gemini-like) sehingga pengguna baru dapat menggunakan fitur generate dokumen tanpa pelatihan panjang. | Should | User | — | User baru dapat menyelesaikan Journey A (§8.1) tanpa bantuan dalam percobaan pertama | Minimal 80% pengguna uji (internal UAT) berhasil menyelesaikan generate dokumen pertama tanpa bantuan dalam sesi onboarding |
| NFR-010 | Dukungan Bahasa Indonesia Formal | Output AI (chat maupun dokumen) secara default menggunakan Bahasa Indonesia formal yang konsisten dengan gaya dokumen bisnis, sesuai tujuan fine-tuning pada `master_spec.md` §3. | Must | System | Model fine-tuned tersedia (bukan base model generik) | Dokumen hasil generate menggunakan ejaan dan struktur formal yang konsisten | Sample evaluasi internal menunjukkan kepatuhan struktur format baku (heading/numbering) sesuai rubric evaluasi pada `ML_Deployment_Strategy.md` |
| NFR-011 | Observability & Monitoring | Sistem mencatat metrik kesehatan (latensi, error rate, utilisasi inference) yang dapat dipantau Admin secara real-time atau near real-time. | Should | Admin, System | Monitoring stack terpasang | Dashboard monitoring menampilkan metrik terbaru dengan lag minimal | Metrik yang ditampilkan ter-update dalam interval ≤ 5 menit |
| NFR-012 | Portabilitas Komponen Inference | Backend aplikasi tidak hardcode terhadap satu model/provider inference tertentu, memungkinkan penggantian model (upgrade fine-tuning) tanpa mengubah kontrak API aplikasi. | Must | System | — | Inference service diakses backend melalui kontrak API internal yang stabil | Penggantian model inference service tidak memerlukan perubahan pada endpoint chat/document API yang dipakai frontend |
| NFR-013 | Pemulihan Bencana & Backup Data | Data dokumen, percakapan, dan audit log dicadangkan secara berkala dan dapat direstore. | Must | Admin, System | Backup job terjadwal aktif | Backup berjalan sesuai jadwal dan dapat diverifikasi integritasnya | Restore test dari backup terbaru berhasil mengembalikan data tanpa korupsi dalam uji berkala |

Total: **13 Non-Functional Requirements** (NFR-001 s/d NFR-013), memenuhi minimum 10 NFR yang disyaratkan.

## 11. UI/UX Requirements

Referensi pola interaksi dari ChatGPT/Claude/Gemini, diadaptasi dengan penekanan pada document generation (bukan menyalin fitur yang tidak relevan dengan scope produk ini, misal tidak ada image generation permanen dan tidak ada voice interaction di MVP — `master_spec.md` §2).

- **Sidebar**: Panel kiri persisten berisi tombol "New Chat" di atas, daftar Conversation History di bawahnya (judul percakapan + timestamp relatif), kolom pencarian riwayat, dan akses ke User Settings di bagian bawah sidebar. Sidebar dapat di-collapse untuk memperluas area kerja dokumen.
- **New Chat**: Tombol aksi utama yang selalu terlihat (sticky di atas sidebar). Klik membuka sesi chat kosong dengan chat input siap pakai dan opsi "mulai dari template" ditawarkan sebagai shortcut, tanpa memaksa user memilih template.
- **Conversation History**: Daftar percakapan diurutkan terbaru-ke-lama, tiap item menampilkan judul (auto-generated dari prompt pertama, dapat di-rename), indikator jika percakapan memiliki dokumen terkait, dan menu kontekstual (rename/delete) via klik kanan atau ikon titik tiga.
- **Document Workspace**: Panel kerja dokumen yang muncul bersebelahan dengan chat panel (split view) ketika sesi chat menghasilkan/berisi dokumen — bukan menggantikan chat, melainkan mendampingi, sehingga user tetap dapat memberi instruksi revisi via chat sambil melihat dokumen update secara live. Menampilkan judul dokumen, status lifecycle (Draft/Review/dsb — §29), indikator autosave, dan akses ke version history serta tombol export.
- **Chat Input**: Area input di bagian bawah chat panel, mendukung multi-line, tombol kirim, tombol attach file, indikator "AI is typing/generating" saat respons berlangsung, dan tombol "Stop generating" saat streaming aktif.
- **File Attachment**: Ikon paperclip di chat input membuka dialog upload; file yang diunggah muncul sebagai chip/preview di atas chat input sebelum dikirim, dengan status upload (uploading/processing/ready) dan opsi hapus sebelum kirim. Setelah terkirim, file tetap terlihat sebagai referensi yang dapat diklik ulang dalam histori chat.
- **Document Editor**: Editor WYSIWYG terstruktur (bukan plain markdown mentah bagi user) di dalam Document Workspace, dengan toolbar minimal (heading level, bold/italic, bullet/numbered list, insert table, undo/redo) dan dukungan highlight bagian yang baru diubah AI setelah instruksi revisi (lihat §15).
- **User Settings**: Halaman/panel berisi profil dasar user (nama, email — diisi Admin, read-only bagi Authorized User), preferensi tampilan (tema terang/gelap), dan untuk Authorized User tidak ada opsi registrasi/ubah role (sesuai closed deployment).
- **Admin Dashboard**: Area terpisah (bukan bagian chat UI biasa) yang hanya dapat diakses role Administrator, berisi navigasi ke: User Management, Role & Permission Management, Knowledge Base Management, Template Management, Audit Log, Usage Analytics, dan System Monitoring — masing-masing sebagai halaman terstruktur dengan tabel data, filter, dan aksi CRUD sesuai §22-24.

## 12. Conversational AI

- Mendukung percakapan multi-turn dengan konteks percakapan dipertahankan dalam satu sesi (lihat FR-003, FR-004).
- Model dapat membedakan permintaan percakapan umum vs permintaan terkait dokumen (generate/revisi) dan merespons dengan format yang sesuai (teks biasa vs dokumen terstruktur ke Document Workspace).
- Streaming response wajib untuk pengalaman real-time (FR-004); dapat dihentikan user di tengah jalan.
- Model inference berjalan sebagai service terpisah dari backend aplikasi (lihat §28 System Architecture), diakses melalui kontrak API internal yang stabil (NFR-012) sehingga model dapat diganti saat hasil fine-tuning baru tersedia tanpa mengubah backend aplikasi.
- Tidak ada image generation (permanen out of scope) dan tidak ada voice interaction (out of scope MVP) sesuai `master_spec.md` §2 — tombol/menu terkait fitur tersebut tidak ditampilkan di UI agar tidak menimbulkan ekspektasi salah pengguna.

## 13. Chat History

- Setiap percakapan tersimpan permanen di database (PostgreSQL) terasosiasi dengan user pemilik (FR-005).
- Mendukung pencarian berbasis judul dan isi pesan (FR-006).
- User dapat mengganti nama dan menghapus percakapan miliknya sendiri (FR-007); penghapusan bersifat permanen bagi user namun dapat dipertahankan sebagai soft-delete di backend untuk kebutuhan audit Admin, tanpa mengekspos kembali ke user.
- Riwayat percakapan yang berkaitan dengan dokumen menyimpan referensi ke dokumen terkait, sehingga membuka percakapan lama otomatis membuka dokumen versi yang relevan di Document Workspace.

## 14. Document Generation

- Dipicu dari instruksi natural language di chat (FR-008), opsional menggunakan template sebagai basis struktur (FR-009).
- Hasil generate langsung terstruktur (heading/paragraf/list/tabel) sesuai konteks permintaan, bukan blok teks tanpa format.
- Setiap generate baru membuat entri dokumen dengan status lifecycle awal **"Draft"** (lihat §29) dan version 1 di version history.
- Retrieval RAG (jika relevan terhadap permintaan) dijalankan sebelum generation untuk menyuntikkan knowledge internal yang relevan dan permission-aware (lihat §19).
- Mendukung regenerasi penuh (FR-010) maupun sebagian/selective regeneration (FR-011) — lihat detail behavior di §15.

## 15. Document Editor

Document Editor adalah komponen inti Document Workspace, wajib mendukung elemen berikut dengan behavior sebagai berikut:

- **Heading**: Mendukung minimal 3 level heading (H1-H3) untuk struktur section/sub-section dokumen formal. Mengubah level heading suatu baris tidak mengubah konten teksnya, hanya styling dan struktur hierarki (relevan untuk daftar isi otomatis di Phase 2). (FR-012)
- **Paragraph**: Blok teks biasa dengan formatting inline dasar (bold, italic, underline). Enter membuat paragraf baru; Shift+Enter membuat line break dalam paragraf yang sama. (FR-013)
- **Bullet List**: Unordered list dengan dukungan nested (sub-bullet melalui indentasi/tab). Toggle bullet dapat diterapkan pada paragraf yang sudah ada tanpa menghapus isinya. (FR-014)
- **Numbered List**: Ordered list dengan penomoran otomatis yang menyesuaikan saat item ditambah, dihapus, atau diurutkan ulang; mendukung nested numbering (1, 1.1, 1.1.1 gaya dokumen formal). (FR-015)
- **Table**: Tabel dengan kontrol tambah/hapus baris dan kolom, merge sel (Should, bukan Must di MVP), dan isi sel dapat berisi teks terformat dasar. (FR-016)
- **Undo/Redo**: Setiap aksi edit (manual maupun hasil AI yang diterapkan ke editor) tercatat dalam stack undo/redo per sesi editing, mendukung minimal 20 langkah undo berurutan. (FR-017)
- **Autosave**: Perubahan tersimpan ke backend secara debounced (misal setiap beberapa detik setelah user berhenti mengetik) tanpa aksi simpan manual; indikator status "Saving..."/"Saved" selalu terlihat. Kegagalan autosave (misal koneksi putus) ditampilkan jelas ke user dengan opsi retry, tidak silently gagal. (FR-018)
- **Version History**: Setiap milestone perubahan signifikan (generate awal, regenerasi penuh, regenerasi sebagian, restore) membuat entri versi baru dengan actor (user/AI), timestamp, dan ringkasan perubahan. User dapat membuka diff antar dua versi mana pun. (FR-019)
- **Selective Regeneration**: Saat user memberi instruksi revisi yang merujuk bagian spesifik ("ubah bagian X", "perpendek paragraf Y"), sistem mengidentifikasi section/blok yang relevan (berdasarkan heading/struktur dokumen) dan hanya mengirim bagian tersebut (plus context secukupnya) ke model untuk diregenerasi, lalu menggantikan hanya blok tersebut di dokumen — bagian lain dijamin tidak berubah karakter-per-karakter. Hasil perubahan di-highlight sementara di editor agar user mudah melihat apa yang berubah. (FR-011)
- **Export DOCX/PDF**: Tombol export selalu tersedia di Document Workspace; ekspor mempertahankan seluruh struktur (heading, list, tabel, formatting inline) sehingga file yang dihasilkan identik secara visual dengan tampilan editor saat diklik export, tidak ada reformatting ulang di sisi converter yang mengubah hierarki section. (FR-031)

## 16. Document Upload

- Format yang didukung pada MVP: DOCX, PDF, TXT (FR-021). Format lain ditolak dengan pesan error jelas.
- File diunggah ke object storage, diproses secara asinkron (ekstraksi teks, chunking untuk indexing RAG bila relevan) dengan status visible ke user: Uploading → Processing → Ready / Failed.
- Setiap dokumen upload wajib diklasifikasikan sensitivitasnya (Public/Internal/Confidential/Restricted) sebelum dapat dipakai sebagai sumber RAG atau dibagikan ke user lain (FR-022). Klasifikasi default untuk upload pribadi tanpa niat berbagi adalah "Internal" dan hanya terlihat bagi pengunggah, kecuali diubah eksplisit oleh pengunggah/Admin.
- Upload dokumen tidak otomatis menjadi training data model (merujuk `master_spec.md` §9 poin 2).

## 17. Document Understanding

- Dalam sesi chat yang menyertakan dokumen terunggah, AI dapat meringkas, menjawab pertanyaan, atau membandingkan isi dokumen tersebut (FR-023), dengan jawaban yang merujuk ke bagian dokumen sumber untuk mengurangi halusinasi.
- Mendukung ekstraksi informasi terstruktur (misal daftar action item, tabel data dari dokumen naratif) yang hasilnya dapat langsung disisipkan ke Document Workspace sebagai konten terstruktur baru, bukan hanya teks di chat (FR-024).
- Document Understanding bekerja dalam lingkup sesi chat (dokumen yang diunggah user pada sesi tersebut); ini berbeda dari Knowledge Base (§18) yang bersifat persisten dan lintas sesi/pengguna sesuai permission.

## 18. Knowledge Base

- Knowledge Base adalah kumpulan dokumen referensi persisten (SOP, kebijakan, template referensi) yang dikelola Admin, bukan file upload sesi chat individual (FR-025).
- Setiap entri KB memiliki metadata: klasifikasi sensitivitas, kategori/departemen (dasar untuk Phase 2 Department-Based Access Control), dan permission akses per user/role (FR-026).
- Admin dapat menambah entri KB baik dari upload dokumen baru maupun "mempromosikan" dokumen upload sesi chat (dengan klasifikasi dan permission eksplisit) menjadi entri KB permanen.
- KB adalah sumber dari proses indexing vector DB yang digunakan RAG (§19); KB tidak langsung dipakai untuk fine-tuning model (lihat pembedaan fine-tuning vs RAG, `master_spec.md` §3).

## 19. RAG Integration

- RAG digunakan untuk menyuntikkan knowledge dinamis (KB, dokumen internal yang sering berubah) saat inference, melengkapi fine-tuning yang hanya mengubah *behavior* model (struktur, gaya, instruction-following) — pembedaan ini final sesuai `master_spec.md` §3 dan tidak boleh dikontradiksi.
- **Permission-aware retrieval (wajib, merujuk `master_spec.md` §9 poin 4)**: setiap kali sistem melakukan retrieval dari vector DB untuk menjawab/generate dokumen, backend WAJIB memfilter kandidat hasil retrieval berdasarkan ACL dokumen/entri KB terhadap identitas dan permission user yang sedang membuat request, **sebelum** kandidat tersebut masuk ke context yang dikirim ke model inference. Filtering ini dilakukan di query vector DB (misal metadata filter berbasis permission) dan/atau post-filter di backend — tidak pernah hanya diatasi lewat instruksi di system prompt, karena system prompt dapat di-bypass melalui prompt injection dan bukan kontrol akses yang dapat diandalkan (konsisten dengan NFR-005 dan `master_spec.md` §9 poin 8). (FR-027)
- Jika suatu entri KB relevan secara semantik namun user tidak memiliki permission, entri tersebut tidak pernah muncul di context model maupun di hasil yang terlihat user — bukan hanya disembunyikan di UI, karena filtering terjadi sebelum data meninggalkan backend retrieval layer.
- Hasil retrieval yang dipakai AI dalam jawaban/dokumen ditampilkan sebagai sitasi sumber kepada user, hanya untuk entri yang user memang berhak akses (FR-028).
- Vector DB pada MVP adalah **pgvector** (default, minim moving parts, co-located dengan PostgreSQL) sesuai `master_spec.md` §4; evaluasi migrasi ke Qdrant untuk Phase 2 bila kebutuhan scale/fitur retrieval lanjutan muncul, didetailkan di `ML_Deployment_Strategy.md` §A dan tidak diulang di sini.

## 20. Template Management

- Admin mengelola template dokumen (struktur section baku per jenis dokumen: SOP, Surat, Laporan, Proposal, dll.) melalui CRUD penuh (FR-029).
- Template memiliki status aktif/nonaktif; template nonaktif tidak muncul sebagai pilihan baru bagi user namun dokumen yang sudah dibuat dari template tersebut sebelumnya tetap valid dan tidak terpengaruh.
- User menelusuri template berdasarkan kategori saat memulai dokumen baru, sebagai shortcut opsional — bukan langkah wajib dalam Journey A (FR-030).
- Template pada MVP bersifat global (belum dibatasi per departemen); pembatasan template per departemen adalah bagian dari Phase 2 Department-Based Access Control.

## 21. Document Export

- User dapat mengekspor dokumen aktif dari Document Workspace ke **DOCX** dan **PDF** kapan saja selama proses penulisan, tidak harus menunggu approval (FR-031).
- Hasil ekspor mempertahankan seluruh struktur editor: heading levels, paragraf dengan formatting inline, bullet/numbered list (termasuk nested), dan tabel.
- Ekspor mencantumkan metadata dasar (judul dokumen, versi yang diekspor, tanggal ekspor) agar dapat dibedakan dari dokumen final yang sudah melalui approval Document Control.
- Ekspor dokumen yang belum berstatus "Approved" (lihat §29) secara visual/metadata ditandai sebagai draft, untuk mencegah dokumen draft disalahartikan sebagai dokumen resmi final.

## 22. User Management

- Seluruh akun dibuat oleh Admin; tidak ada self-registration publik (FR-032), konsisten dengan `master_spec.md` §1 "closed deployment, no public registration, admin-managed accounts only".
- Admin dapat membuat, mengedit, menonaktifkan, dan menghapus akun Authorized User. Penonaktifan (bukan penghapusan) adalah aksi default yang direkomendasikan agar riwayat tetap tertelusur untuk audit (FR-032).
- Reset kredensial dipicu oleh Admin, bukan self-service user, sesuai model closed deployment pada MVP (FR-033). Mekanisme self-service reset password dapat dievaluasi di Phase 2 jika kebutuhan operasional berubah.

## 23. Role and Permission Management

- RBAC pada MVP: dua role aplikasi utama, **Administrator** dan **Authorized User** (FR-034), sesuai `master_spec.md` §4 dan §5.
- Admin mengatur permission granular tambahan di atas role dasar untuk mengontrol akses ke entri KB dan template tertentu per user (bagian dari FR-026, FR-034) — ini adalah pondasi menuju Department-Based Access Control dan Resource-Level Permission di Phase 2, namun pada MVP belum granular per-department secara otomatis.
- Perubahan role/permission berlaku pada request API berikutnya dari user terkait (tidak perlu menunggu re-login, namun sesi yang sedang berjalan mengambil permission terbaru pada validasi token/permission check berikutnya).

## 24. Admin Dashboard

Admin Dashboard adalah area terpisah dari chat UI, mencakup modul:
- **User Management** (§22) dan **Role & Permission Management** (§23).
- **Knowledge Base Management** (§18) termasuk klasifikasi dan permission per entri.
- **Template Management** (§20).
- **Audit Log** (FR-036): daftar append-only aksi sensitif (akses dokumen Confidential/Restricted, perubahan permission/role, approval dokumen), dengan filter berdasarkan actor, jenis aksi, dan rentang waktu.
- **Usage Analytics** (FR-035): metrik agregat (jumlah percakapan, dokumen dibuat, user aktif) dengan filter rentang waktu, untuk kebutuhan evaluasi adopsi internal, bukan analytics marketing.
- **System Monitoring** (NFR-011): ringkasan kesehatan sistem (latensi, error rate, utilisasi inference service) — detail penuh observability stack dirujuk ke `ML_Deployment_Strategy.md`.
- Pada MVP, Admin juga menjalankan fungsi **Document Control** (approval dokumen draft → status berikutnya) karena role Document Control terpisah baru ada di Phase 2 (`master_spec.md` §4).

## 25. Security and Privacy

Mengikuti baseline keamanan `master_spec.md` §9 secara penuh, diterapkan ke aplikasi sebagai berikut:
1. Percakapan user **tidak otomatis** dipakai untuk training model (NFR-006).
2. Dokumen upload **tidak otomatis** menjadi training data (NFR-006).
3. Dataset training memerlukan approval eksplisit yang tercatat (mekanisme approval detail ada di `business_process.md`, bukan diulang di sini).
4. RAG retrieval **wajib permission-aware** — filtering ACL dokumen dilakukan di backend sebelum context masuk ke model, bukan hanya di system prompt (§19, FR-027, NFR-005).
5. Dokumen sensitif **wajib** memiliki klasifikasi (Public/Internal/Confidential/Restricted) sebelum dapat dipakai sebagai sumber RAG atau dibagikan (FR-022).
6. Seluruh akses sensitif Admin **tercatat di audit log** append-only, tidak dapat dihapus/diedit lewat UI aplikasi (FR-036, NFR-008).
7. Data internal **tidak boleh** dikirim ke API eksternal pihak ketiga tanpa persetujuan kebijakan perusahaan; default adalah **deny**, override membutuhkan approval formal tercatat (NFR-007).
8. **Otorisasi ditegakkan di backend (API layer)**, bukan hanya UI atau system prompt; system prompt tidak pernah menjadi pengganti backend authorization (NFR-005).

Tambahan untuk aplikasi (derivasi dari baseline di atas):
- Semua endpoint API memvalidasi JWT/session token dan permission pada setiap request, termasuk endpoint yang dipanggil dari dalam alur chat/document editor (bukan hanya endpoint "halaman utama").
- Koneksi FE-BE dan BE-inference service wajib terenkripsi (TLS) baik di lingkungan dev-staging-production yang menyentuh data riil.
- Object storage dokumen menerapkan access control per dokumen, sejalan dengan klasifikasi sensitivitas; dokumen Confidential/Restricted tidak dapat diakses langsung via URL publik tanpa autentikasi/otorisasi berlaku.

## 26. Database Requirements

- **PostgreSQL** sebagai database relasional utama (`master_spec.md` §4), menyimpan: akun user & role/permission, percakapan & pesan, dokumen (metadata + konten terstruktur + version history), template, metadata knowledge base, audit log, usage analytics agregat.
- **pgvector** (extension PostgreSQL) sebagai vector store MVP untuk embedding knowledge base, co-located dengan data relasional untuk meminimalkan moving parts operasional pada tahap awal (`master_spec.md` §4; evaluasi migrasi ke Qdrant Phase 2 di `ML_Deployment_Strategy.md`).
- Konten dokumen disimpan dalam representasi terstruktur (misal JSON/ProseMirror-style document tree atau setara) yang dapat direkonstruksi persis oleh editor dan diterjemahkan secara deterministik ke DOCX/PDF saat ekspor — tidak disimpan sebagai blob HTML/markdown mentah tanpa struktur yang dapat divalidasi.
- Version history disimpan sebagai snapshot/delta per versi dokumen, terasosiasi dengan actor (user ID atau "AI") dan timestamp, cukup untuk merekonstruksi diff antar versi mana pun (FR-019).
- Audit log disimpan sebagai tabel append-only (tidak ada operasi UPDATE/DELETE terbuka ke aplikasi) terpisah secara logis dari data operasional agar tidak terpengaruh retensi data operasional.
- Skema database dirancang agar kolom permission/akses pada tabel dokumen & KB dapat diperluas untuk Department-Based Access Control dan Resource-Level Permission Phase 2 tanpa migrasi destruktif (misal menyediakan kolom `department_id`/`resource_scope` nullable sejak MVP, diisi bertahap).

## 27. API Requirements

- Backend aplikasi dibangun dengan **Python FastAPI** (`master_spec.md` §4), menyediakan REST/HTTP API (dan/atau WebSocket/SSE untuk streaming chat) yang dikonsumsi frontend Next.js.
- API utama yang wajib ada pada MVP (nama endpoint ilustratif, detail kontrak di dokumen desain teknis terpisah bila dibuat):
  - Autentikasi & sesi (login, refresh token, logout) — tidak ada endpoint self-registration publik.
  - Conversation & message (create/list/get/delete conversation; send message; streaming response).
  - Document (create dari generate, get, update manual edit, selective regenerate, list version history, restore version, export).
  - Upload (upload file, get processing status, trigger document understanding query).
  - Knowledge Base (CRUD entri, set permission, trigger re-index).
  - Template (CRUD, list by category).
  - Admin (user CRUD, role/permission assignment, audit log query, usage analytics query).
- Seluruh endpoint di atas menegakkan RBAC (NFR-005) dan dicatat ke audit log bila menyentuh data sensitif (akses dokumen Confidential/Restricted, perubahan permission).
- Backend berkomunikasi dengan inference service melalui kontrak API internal terpisah (bukan memanggil library model langsung), memenuhi NFR-012 (portabilitas model) dan prinsip T2 pemisahan backend-inference pada `master_spec.md` §4.

## 28. System Architecture (Ringkas)

Detail arsitektur ML lengkap ada di `ML_Deployment_Strategy.md`; bagian ini hanya meringkas komponen aplikasi yang relevan bagi PRD ini:

```
[Next.js Frontend] --HTTPS--> [FastAPI Backend]
                                   |-- PostgreSQL (data relasional: user, chat, dokumen, template, audit log)
                                   |-- pgvector (vector store KB, co-located di PostgreSQL, MVP)
                                   |-- Object Storage (file upload, dokumen asli, lampiran)
                                   |-- Inference Service (terpisah, self-hosted model fine-tuned; diakses via API internal)
```

- **Next.js Frontend**: conversational UI, Document Workspace/Editor, Admin Dashboard; SSR untuk performa awal (`master_spec.md` §4).
- **FastAPI Backend**: satu-satunya pihak yang menegakkan otorisasi, menjalankan orkestrasi RAG permission-aware (query pgvector dengan filter permission, lalu menyusun context), dan menjalankan logika document generation/editing/version control/export.
- **PostgreSQL**: database relasional utama; juga menghosting pgvector pada MVP.
- **Vector DB (pgvector MVP)**: indexing embedding knowledge base untuk retrieval RAG.
- **Inference Service**: service terpisah yang menyajikan model fine-tuned (self-hosted); backend tidak pernah memanggil model langsung sebagai library in-process, selalu lewat API internal, agar model dapat diganti/di-upgrade independen (lihat §12, NFR-012).
- **Object Storage**: menyimpan file upload asli dan aset ekspor (DOCX/PDF) sebelum/ setelah diunduh user.

Arsitektur ini sengaja dipisah per komponen (bukan monolith) agar scaling inference (GPU-bound) dapat dilakukan independen dari scaling backend aplikasi (umumnya CPU/IO-bound), sesuai realita hardware pada `master_spec.md` §7.

## 29. Integration Requirements

- **Integrasi ke Document Control Lifecycle** (merujuk `PRD_Document_Control_SaaS.md`, Dokumen #2): setiap dokumen yang dihasilkan AI (baik dari generate baru FR-008 maupun hasil edit lanjutan) **masuk ke lifecycle Document Control dengan status awal "Draft"**. Status ini bukan status final/resmi — dokumen pada status "Draft" dapat diekspor (§21) namun ditandai sebagai draft, dan tidak dianggap dokumen resmi perusahaan sampai melalui tahapan approval berikutnya pada lifecycle Document Control.
- Transisi status dari "Draft" ke status lifecycle berikutnya (misal "In Review", "Approved", dst. — nomenklatur dan state machine lengkap didefinisikan di `PRD_Document_Control_SaaS.md` dan tidak didefinisikan ulang di sini untuk mencegah duplikasi/kontradiksi) dipicu melalui aksi "Ajukan untuk Approval" oleh Authorized User (FR-037).
- Pada MVP, fungsi approval dalam lifecycle Document Control **dijalankan oleh Administrator** (bukan role Document Control terpisah), sesuai keputusan `master_spec.md` §4 "Document Control role". Di Phase 2, fungsi ini dapat dipindahkan ke role Document Control terdedikasi tanpa mengubah model status lifecycle dasar.
- Setiap transisi status dokumen (termasuk "Draft" → status berikutnya) tercatat sebagai entri audit log (FR-036) dan sebagai entri version history (FR-019), menjamin AI output dapat ditelusuri penuh dari generate awal sampai status approval terakhir.
- Human review adalah **wajib** untuk dokumen bisnis penting sebelum melewati status "Draft"; AI output tidak pernah menjadi dokumen resmi secara otomatis (`master_spec.md` §4 "Human review").

## 30. Acceptance Criteria

Produk MVP dinyatakan siap untuk User Acceptance Testing (UAT) internal apabila seluruh kondisi berikut terpenuhi:
1. Seluruh FR berpriority **Must** (lihat §9) telah diimplementasikan dan lulus pengujian fungsional sesuai kolom Acceptance Criteria masing-masing.
2. Seluruh NFR berpriority **Must** (lihat §10) terverifikasi melalui pengujian non-fungsional (load test dasar, security test otorisasi backend, uji restore backup).
3. Journey A (generate dokumen baru) dan Journey B (revisi dokumen berdasarkan instruksi) pada §8 dapat dijalankan end-to-end oleh pengguna uji tanpa intervensi developer.
4. RAG permission-aware (FR-027) terverifikasi dengan skenario uji minimal dua user berbeda permission terhadap entri KB yang sama, menghasilkan context retrieval yang berbeda secara terbukti (bukan asumsi).
5. Audit log mencatat 100% aksi sensitif yang didefinisikan pada §25 selama periode UAT, tervalidasi dengan sampling manual oleh Admin.
6. Ekspor DOCX dan PDF dari dokumen berisi heading, paragraf, bullet list, numbered list, dan tabel menghasilkan file yang dapat dibuka tanpa corrupt di Microsoft Word dan Adobe Reader, dengan struktur visual identik dengan editor.
7. Tidak ada jalur di UI maupun API yang memungkinkan bypass RBAC/permission (diverifikasi melalui pengujian otorisasi terhadap seluruh endpoint pada §27).
8. Integrasi status lifecycle Document Control (§29) terverifikasi: dokumen baru selalu lahir dengan status "Draft" dan transisi status tercatat lengkap di audit log dan version history.

## 31. MVP Scope

**Termasuk dalam MVP** (selaras penuh dengan `master_spec.md` §2 dan §6):
- Conversational UI lengkap (chat, streaming, riwayat, pencarian riwayat) — §12, §13.
- Document generation, document editor penuh (heading/paragraf/bullet/numbered/tabel/undo-redo/autosave/version history/selective regeneration), export DOCX/PDF — §14, §15, §21.
- Document upload dan document understanding dalam lingkup sesi chat — §16, §17.
- Knowledge Base dan RAG permission-aware di atas pgvector — §18, §19.
- Template Management dasar (CRUD, kategori) — §20.
- User Management dan RBAC dua-role (Administrator, Authorized User) — §22, §23.
- Admin Dashboard: User Management, Role & Permission, KB Management, Template Management, Audit Log, Usage Analytics dasar, System Monitoring ringkas — §24.
- Fungsi approval Document Control dijalankan oleh Administrator (belum role terpisah) — §29.
- Keamanan baseline penuh sesuai `master_spec.md` §9 — §25.

**Secara eksplisit TIDAK termasuk MVP** (mengikuti `master_spec.md` §2 dan §6):
- Image generation (permanen out of scope, bukan hanya MVP).
- Voice interaction (out of scope MVP, dapat di-reassess Phase 2+).
- Department-Based Access Control granular dan Resource-Level Permission granular.
- Role Document Control terpisah dari Administrator.
- Multi-tenant SaaS komersial ke pelanggan eksternal.
- Foundation model pretraining dari nol (strategi tetap fine-tuning open-weight model).
- Self-service password reset oleh Authorized User (reset hanya dipicu Admin pada MVP).

## 32. Product Roadmap

| Fase | Fokus | Item Kunci |
|---|---|---|
| **MVP** | Aplikasi inti dapat dipakai internal terbatas untuk generate & revisi dokumen dengan RBAC dasar dan RAG permission-aware. | Seluruh item §31 "Termasuk dalam MVP"; model inference tahap awal adalah hasil fine-tuning kandidat pertama (lihat `ML_Deployment_Strategy.md`) yang lolos evaluasi internal minimum. |
| **Phase 2** | Memperdalam kontrol akses dan memisahkan fungsi governance dokumen. | Department-Based Access Control; Resource-Level Permission; role Document Control terpisah dari Administrator; evaluasi migrasi vector DB pgvector → Qdrant bila kebutuhan scale/fitur retrieval lanjutan terbukti; self-service password reset; peningkatan Usage Analytics dan System Monitoring. |
| **Future (belum committed, kandidat evaluasi)** | Perluasan kapabilitas lanjutan bila ada kebutuhan bisnis dan traksi yang memadai. | Reassessment voice interaction (bukan committed, hanya dapat dievaluasi ulang sesuai `master_spec.md` §2); perluasan integrasi API eksternal dengan guardrail data governance penuh; perluasan multi-model inference (routing antar model fine-tuned untuk use case berbeda); tidak termasuk image generation karena itu permanen out of scope sesuai keputusan final `master_spec.md` §2. |

---

**Catatan Penutup:** Dokumen ini tidak mendefinisikan ulang state machine lifecycle Document Control (didefinisikan di `PRD_Document_Control_SaaS.md`), tidak mendefinisikan ulang detail benchmarking/model/VRAM/GPU pricing (didefinisikan di `ML_Deployment_Strategy.md`), dan tidak membuat ID requirement baru untuk keputusan yang sudah final di `master_spec.md` — seluruh keputusan tersebut dirujuk, bukan diduplikasi atau dikontradiksi.
