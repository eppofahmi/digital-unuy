# PRD: Digital UNUY — Website & Digital Service Portal

**Status:** Approved · **Version:** 1.0 · **Owner:** Arief Faqihudin · **Issue:** TBD

## 1. Overview

| Item | Detail |
|------|--------|
| **What** | Website resmi dan Digital Service Portal Direktorat Transformasi Digital UNU Yogyakarta: katalog layanan, pengajuan tanpa akun, pelacakan tiket, dan CMS admin. |
| **Who** | Publik/pemohon (tanpa akun), Admin/Petugas, dan Superadmin Direktorat Transformasi Digital. |
| **Why** | Permohonan layanan masuk lewat kanal pribadi tanpa antrean bersama, SLA, atau jejak audit. |
| **Success Metric** | Median waktu penyelesaian permohonan, dari submit hingga status `Selesai`. Angka target belum ditetapkan (Q1). |

## 2. Problem Statement

Unit, prodi, dan civitas membutuhkan layanan digital (akun, sistem informasi, infrastruktur, integrasi data). Saat ini permintaan datang melalui WhatsApp, email pribadi, atau bertemu langsung. Akibatnya:

- **Tidak ada antrean bersama.** Petugas tidak tahu beban kerja aktual; permintaan bisa hilang atau dikerjakan ganda.
- **Tidak ada status yang dapat dilihat pemohon.** Pemohon berulang kali menanyakan progres melalui kanal pribadi.
- **Tidak ada SLA atau data waktu penyelesaian.** Direktorat tidak dapat mengukur kecepatan layanan atau memprioritaskan antrean.
- **Tidak ada jejak audit.** Perubahan status, catatan, dan penugasan tidak tercatat pada satu sistem.
- **Katalog layanan tidak diketahui.** Unit tidak tahu layanan apa yang tersedia, syaratnya, atau siapa pemiliknya.

Biaya bila dibiarkan: waktu petugas habis untuk koordinasi manual, pemohon menunggu tanpa kepastian, dan pimpinan tidak memiliki data untuk mengevaluasi layanan.

## 3. Goals & Non-Goals

**Goals**

- Satu katalog layanan digital kampus dengan detail sasaran, syarat, dokumen, prosedur, dan estimasi waktu.
- Pengajuan layanan tanpa akun, dengan nomor tiket unik dan token pelacakan pribadi.
- Antrean permohonan terpadu dengan status, penugasan, catatan publik/internal, dan lampiran privat.
- Website publik yang dikelola CMS: layanan, ekosistem digital, dokumentasi berversi, publikasi, halaman, tim, dan SEO.
- RBAC granular dengan permission per modul; superadmin tidak dapat dikunci oleh admin.
- Jejak audit untuk perubahan sensitif dan notifikasi operasional.

**Non-Goals (Phase 1)**

- SSO kampus, SIAKAD/Feeder, atau integrasi data institusional.
- Object storage (S3/R2/GCS); upload tetap di filesystem lokal.
- Pemindaian malware/antivirus pada unggahan.
- Multi-instance dengan Redis (belum diimplementasikan; lihat Q7).
- Aplikasi mobile native, tanda tangan elektronik, dan pembayaran.

## 4. User Personas & Roles

| Persona | Description | Core Needs | Role rules |
|---------|-------------|------------|------------|
| **Publik / Pemohon** | Civitas atau unit yang mencari layanan dan mengajukan permohonan. Tidak memiliki akun. | Menemukan layanan, membaca syarat, mengajukan formulir, menyimpan tiket + token, melacak progres, mengirim kelengkapan. | Hanya dapat membaca konten publik dan mengakses permohonannya sendiri melalui nomor tiket + token acak. Tidak dapat melihat permohonan orang lain. |
| **Admin/Petugas** | Staf Direktorat yang memproses permohonan dan mengelola konten. | Antrean permohonan, verifikasi, penugasan, catatan, dokumen, layanan, publikasi, notifikasi. | Permission-driven: `requests.read`, `requests.manage`, `requests.assign`, `services.manage`, `content.manage`, `documents.manage`, `notifications.read`. Role `admin` dapat diubah permission-nya oleh Superadmin. |
| **Superadmin** | Pemilik sistem dan kebijakan. | Semua kemampuan Admin, ditambah pengguna, role, pengaturan, dan audit. | Semua permission. Role superadmin bersifat tetap dan tidak dapat diubah. Minimal satu Superadmin aktif harus selalu ada; tidak dapat menonaktifkan/menurunkan akun sendiri. |

**Aturan role tambahan**

- Akun staf dibuat admin dan selalu `active: false`; aktivasi mengirim tautan sekali pakai 48 jam.
- Login hanya untuk `admin`/`superadmin`; akun pemohon tidak dibuat pada Phase 1.
- Permission dicek di server pada setiap endpoint; sidebar hanya menyembunyikan menu.
- Perubahan permission role `admin` langsung menginvalidasi cache permission (TTL 60 detik).

## 5. User Stories & Acceptance Criteria

- [ ] **US-01 — Katalog layanan.** As a **Publik**, I want to browse and search services so that I can find the right one.
  - [ ] AC1: Hanya layanan `active` yang tampil; filter kategori, pencarian, dan pagination.
  - [ ] AC2: Detail menampilkan sasaran, persyaratan, dokumen, prosedur, estimasi waktu, dokumen terkait, dan status menerima permohonan.
  - [ ] AC3: Empty state dan loading state tersedia; pencarian tidak peka huruf besar/kecil dan regex di-escape.
- [ ] **US-02 — Pengajuan tanpa akun.** As a **Pemohon**, I want to submit a request without creating an account so that I can get help quickly.
  - [ ] AC1: Formulir dibangun dari versi form aktif layanan; field `showIf` hanya tampil bila kondisi terpenuhi.
  - [ ] AC2: Validasi server: required, tipe, panjang, email, pattern, opsi, ukuran/MIME file.
  - [ ] AC3: Submit memerlukan header `x-idempotency-key` (10–120 karakter); key yang sama mengembalikan tiket yang sama.
  - [ ] AC4: Nomor tiket format `UNU-DTD-YYMM-XXXXXX` dan unik.
  - [ ] AC5: Respons berisi `ticketNumber`, `requestId`, `trackingUrl`, dan `emailStatus`.
  - [ ] AC6: Rate limit 40 submit/jam per IP dan 12/jam per email.
- [ ] **US-03 — Pelacakan publik.** As a **Pemohon**, I want to track my request so that I know its progress.
  - [ ] AC1: Akses memakai nomor tiket + token acak; token hanya disimpan sebagai hash.
  - [ ] AC2: Respons memuat status, substatus, timeline, catatan publik, lampiran, dan ringkasan penyelesaian; catatan internal tidak pernah dikirim.
  - [ ] AC3: Token salah atau tiket tidak ada → 404 tanpa membocorkan keberadaan.
  - [ ] AC4: Rate limit 30 permintaan/10 menit.
- [ ] **US-04 — Kelengkapan pemohon.** As a **Pemohon**, I want to respond and upload missing documents so that my request can continue.
  - [ ] AC1: Pemohon dapat mengirim pesan, tanggapan, dan lampiran melalui token pelacakan.
  - [ ] AC2: Lampiran divalidasi MIME, ekstensi, magic bytes, dan ukuran; disimpan privat.
  - [ ] AC3: Lampiran hanya dapat diakses pemilik permohonan atau staf berizin.
- [ ] **US-05 — Login staf.** As an **Admin**, I want to log in securely so that I can process requests.
  - [ ] AC1: Login email/kata sandi dengan bcrypt cost 12; rate limit 8 percobaan/10 menit per email+IP.
  - [ ] AC2: Akun nonaktif → 403; kredensial salah → 401; akun pemohon tidak dapat login.
  - [ ] AC3: Sesi 7 hari, cookie `HttpOnly`, `SameSite=Lax`, `Secure` di produksi, token ber-hash.
  - [ ] AC4: Login sukses memperbarui `lastLoginAt` dan menulis audit `auth.login`; kegagalan menulis `auth.login_failed`.
- [ ] **US-06 — Aktivasi akun.** As a **Superadmin**, I want to invite staff so that they can set a password.
  - [ ] AC1: Akun baru selalu nonaktif dan belum punya password.
  - [ ] AC2: Aktivasi membuat token 48 jam, sekali pakai, disimpan sebagai hash.
  - [ ] AC3: Aktivasi mengirim email tautan; bila pengiriman gagal, akun tetap nonaktif dan API mengembalikan 502.
  - [ ] AC4: Setelah password dibuat, token dihapus dan sesi langsung dibuat.
  - [ ] AC5: Token kedaluwarsa/terpakai → 410.
- [ ] **US-07 — Dashboard admin.** As an **Admin**, I want a summary so that I can prioritise work.
  - [ ] AC1: Menampilkan total, diajukan, diproses, selesai, ditolak, menunggu pemohon, tingkat penyelesaian, dan rata-rata jam penyelesaian.
  - [ ] AC2: Periode 7/30/90 hari/semua; data dihitung dari permohonan nyata.
  - [ ] AC3: Antrean tindakan dan permohonan terbaru ditampilkan.
- [ ] **US-08 — Siklus permohonan.** As an **Admin**, I want to verify, process, request info, complete, or reject so that requests move correctly.
  - [ ] AC1: Transisi legal: `Diajukan → Diverifikasi`, `Diverifikasi → Diproses`, `Diproses → Selesai`, dan `Diajukan/Diverifikasi/Diproses → Ditolak`.
  - [ ] AC2: `request_info` menetapkan substatus `Menunggu Kelengkapan Pemohon` dan mewajibkan catatan.
  - [ ] AC3: `complete` mewajibkan ringkasan ≥10 karakter dan hanya dari `Diproses`.
  - [ ] AC4: `reject` mewajibkan alasan ≥10 karakter.
  - [ ] AC5: Status terminal tidak dapat diubah → 409.
  - [ ] AC6: Update bersyarat mencegah balapan; bila status berubah, → 409.
  - [ ] AC7: Setiap transisi menulis riwayat status, audit, dan email notifikasi (bila email tersedia).
- [ ] **US-09 — Penugasan dan catatan.** As an **Admin**, I want to assign officers and add notes so that work is coordinated.
  - [ ] AC1: Hanya user aktif ber-role admin/superadmin yang dapat ditugaskan.
  - [ ] AC2: Catatan publik terlihat pemohon; catatan internal tidak.
  - [ ] AC3: `requests.assign` diperlukan untuk menugaskan.
- [ ] **US-10 — Manajemen layanan + form builder.** As an **Admin**, I want to manage services and forms so that request types stay current.
  - [ ] AC1: Layanan memiliki kategori, ringkasan, deskripsi, ikon, status aktif, menerima permohonan, unggulan, urutan, sasaran, syarat, dokumen, prosedur, estimasi.
  - [ ] AC2: Formulir minimal 1 field, maksimal 60 field; key unik dan aman; tipe field tervalidasi.
  - [ ] AC3: Perubahan field membuat versi form baru; permohonan lama menyimpan snapshot.
  - [ ] AC4: Layanan tidak dapat diaktifkan sebelum form memiliki minimal satu field.
  - [ ] AC5: Slug unik; duplikat → 409.
- [ ] **US-11 — Dokumentasi berversi.** As an **Admin**, I want to upload and version documents so that SOP/panduan remain current.
  - [ ] AC1: Kategori SOP, Panduan, Kebijakan, Formulir, Dokumen Lainnya.
  - [ ] AC2: Upload divalidasi MIME/ekstensi/magic bytes/max 10 MB; file privat.
  - [ ] AC3: Versi baru memiliki label unik per dokumen dan menjadi versi aktif.
  - [ ] AC4: Publish/unpublish/arsip mengatur visibilitas publik; unduhan hanya untuk yang published.
- [ ] **US-12 — Ekosistem digital.** As an **Admin**, I want to manage platforms and categories so that the public catalog stays accurate.
  - [ ] AC1: Platform memiliki kategori, status (Sudah Rilis/Dalam Pengembangan/Pilot/Beta), audiens, modul, metrik, galeri, problem/solution/benefit/features.
  - [ ] AC2: Kategori dan platform dapat diurutkan, diaktifkan, dan dipublikasikan.
  - [ ] AC3: Slug unik; filter publik berdasarkan kategori, audiens, status, dan pencarian.
- [ ] **US-13 — Publikasi dan halaman.** As an **Admin**, I want to publish articles and custom pages so that information is available.
  - [ ] AC1: Artikel memiliki status draft/published, kategori, penulis, tanggal, thumbnail, dan konten markdown tersanitasi.
  - [ ] AC2: Halaman kustom memiliki slug, konten, excerpt, dan SEO.
  - [ ] AC3: Slug unik; duplikat → 409.
- [ ] **US-14 — Tim kami.** As an **Admin**, I want to manage team groups and members so that the public team page is current.
  - [ ] AC1: Anggota memiliki nama, jabatan, unit, bio, kelompok, urutan, publikasi, dan foto 3:4.
  - [ ] AC2: Foto diproses dengan Sharp dan hanya dapat diakses publik setelah anggota dipublikasikan.
  - [ ] AC3: Kelompok final: Pimpinan, Product, Engineering, Design (dapat diurutkan).
- [ ] **US-15 — Pengguna dan role.** As a **Superadmin**, I want to manage staff and permissions so that access is controlled.
  - [ ] AC1: Buat pengguna nonaktif; email duplikat → 409; role valid.
  - [ ] AC2: Mengaktifkan akun tanpa password mengirim undangan aktivasi; kegagalan email membatalkan aktivasi.
  - [ ] AC3: Tidak dapat menonaktifkan/menurunkan diri sendiri; minimal satu Superadmin aktif.
  - [ ] AC4: Permission role `admin` dapat diubah; role `superadmin` tetap.
- [ ] **US-16 — Pengaturan situs.** As a **Superadmin**, I want to edit identity, contact, hero, metrics, and SEO so that the site reflects the directorate.
  - [ ] AC1: Email dan URL tervalidasi; sosial maksimal 8; metrik maksimal 8 dan hanya `verified` yang tampil publik.
  - [ ] AC2: Perubahan pengaturan diaudit.
- [ ] **US-17 — Audit.** As a **Superadmin**, I want an activity log so that changes are accountable.
  - [ ] AC1: Mencatat login, user, role, layanan, permohonan, dokumen, pengaturan, dan konten.
  - [ ] AC2: Metadata menyembunyikan password/token/secret/authorization/cookie.
  - [ ] AC3: Filter dan pagination tersedia.
- [ ] **US-18 — Notifikasi.** As an **Admin**, I want operational notifications so that I know what needs action.
  - [ ] AC1: Ringkasan `actionCount` untuk permohonan yang perlu tindakan.
  - [ ] AC2: Daftar notifikasi dapat difilter per tampilan.
- [ ] **US-19 — SEO publik.** As the **Direktorat**, I want search engines to index public content only so that private areas stay out.
  - [ ] AC1: Hanya produksi + HTTPS + non-localhost yang boleh diindeks; staging memakai `NUXT_PUBLIC_NOINDEX`.
  - [ ] AC2: `/admin`, `/masuk`, `/aktivasi`, `/lacak`, `/pencarian`, form pengajuan, dan `/api` selalu noindex.
  - [ ] AC3: Sitemap dan robots dihasilkan dari data; konten `noindex` keluar dari sitemap.
- [ ] **US-20 — Keamanan permukaan publik.** As the **Direktorat**, I want the portal to resist abuse so that data and availability are protected.
  - [ ] AC1: Semua mutasi memerlukan header `x-unu-csrf: 1` dan Origin yang cocok.
  - [ ] AC2: Rate limit login, aktivasi, pengajuan, pelacakan, komentar, pencarian.
  - [ ] AC3: HTML/Markdown disanitasi dengan allowlist; upload privat di luar direktori publik.
  - [ ] AC4: Security header: nosniff, SAMEORIGIN, referrer policy, permissions policy, COOP.

## 6. User Flows

**Flow A — Ajukan layanan (publik)**

1. `/layanan` → cari/filter → `/layanan/{slug}`.
2. Klik "Ajukan" → `/layanan/{slug}/ajukan`.
3. Isi data pemohon (nama, email, telepon) dan field formulir; field `showIf` mengikuti jawaban induk.
4. Klien mengirim multipart dengan `payload`, `__requester`, file, dan header `x-idempotency-key`.
5. Server memvalidasi layanan aktif + menerima permohonan + form siap, lalu memvalidasi field dan file.
6. Server membuat tiket, token pelacakan, riwayat status awal, audit, dan email konfirmasi.
7. UI menampilkan nomor tiket + tautan pelacakan; pemohon menyimpannya.

**Failure paths:** layanan nonaktif → 404; tidak menerima permohonan → 409; form kosong → 409; validasi field → 422 dengan `errors`; file > batas → 422/413; key idempotensi tidak valid → 400; duplikat dengan identitas berbeda → 409.

**Flow B — Lacak dan lengkapi**

1. `/lacak` → masukkan nomor tiket + token.
2. Server mencocokkan hash token dan mengembalikan detail publik.
3. Pemohon mengirim pesan/kelengkapan/lampiran.
4. Status/substatus dan timeline diperbarui; petugas dapat melihat lampiran.

**Flow C — Proses permohonan (admin)**

1. `/admin/permohonan` → filter → detail.
2. `verify` (Diajukan → Diverifikasi) atau `request_info` (substatus menunggu kelengkapan).
3. `start` (Diverifikasi → Diproses), `complete` (Diproses → Selesai, wajib ringkasan), `reject` (alasan wajib).
4. Setiap aksi: conditional update, riwayat status, audit, email.
5. Konflik status → 409 dan UI meminta muat ulang.

**State machine — permohonan**

| From | Action | To | Actor |
|------|--------|----|-------|
| Diajukan | verify | Diverifikasi | requests.manage |
| Diajukan | request_info | Diajukan + Menunggu Kelengkapan | requests.manage |
| Diajukan | reject | Ditolak | requests.manage |
| Diverifikasi | start | Diproses | requests.manage |
| Diverifikasi | request_info | Diverifikasi + Menunggu Kelengkapan | requests.manage |
| Diverifikasi | reject | Ditolak | requests.manage |
| Diproses | complete | Selesai | requests.manage |
| Diproses | request_info | Diproses + Menunggu Kelengkapan | requests.manage |
| Diproses | reject | Ditolak | requests.manage |
| Selesai/Ditolak | any | — | 409 |

**Flow D — Aktivasi staf**

1. Superadmin membuat akun (nonaktif) atau mengaktifkan akun tanpa password.
2. Server membuat token aktivasi 48 jam dan mengirim email.
3. Staf membuka `/aktivasi?token=...`, membuat password yang memenuhi kebijakan.
4. Server mengklaim token secara atomik, menyimpan hash password, menandai email terverifikasi, menghapus token lain, dan membuat sesi.
5. Token terpakai/kedaluwarsa → 410.

## 7. Tech Stack

| Layer | Technology | Version | Confirmed |
|-------|------------|---------|-----------|
| Runtime | Node.js | 20+ (README) | Yes |
| Framework | Nuxt | ^3.21.11 | Yes — `package.json` |
| UI | Vue | ^3.5.44 | Yes |
| Language | TypeScript | strict | Yes |
| Styling | Tailwind CSS + @nuxtjs/tailwindcss | ^3.4.19 / ^6.14.0 | Yes |
| Database | MongoDB + Mongoose | ^9.11.4 | Yes |
| Auth | bcryptjs (cost 12), session token SHA-256 | ^3.0.3 | Yes |
| Email | nodemailer | ^10.3.1 | Yes |
| Markdown | marked + sanitize-html | ^18.1.3 / ^2.18.1 | Yes |
| Images | sharp | ^0.35.5 | Yes |
| Storage | Local private filesystem (`NUXT_UPLOAD_DIR`) | — | Yes |
| Testing | Playwright-core scripts (`test:e2e`, `test:visual`, `test:responsive`) | ^1.64.1 | Yes |
| Deployment | Nuxt Nitro Node output, port 3000 | — | Phase 1 |

**Target arsitektur (D-STACK, 7 Nov 2026):** backend Go + Gin dan frontend Vue/Nuxt sebagai dua repo terpisah. Tabel di atas tetap mendokumentasikan Phase 1 as-built (monolit Nitro) dan tidak berubah retroaktif; rincian target, porting surface, dan urutan migrasi ada di `IMPLEMENTATION_PLAN.md` §4.1 dan §13. Akibat yang harus diselesaikan sebelum endpoint pertama pindah: transport auth (BFF + internal token vs cookie lintas-origin), penanganan upload (`NUXT_UPLOAD_DIR` vs object storage), dan rate limit bersama.

## 8. Core Features (MVP / V1 / V2)

| # | Feature | For Whom / Solves What | Priority | BE | FE | Notes |
|---|---------|------------------------|----------|----|----|-------|
| FR-01 | Public home + CMS-driven content | Publik — profil dan katalog | P1 | Yes | Yes | Hero, layanan unggulan, ekosistem, dokumen, publikasi, metrik terverifikasi |
| FR-02 | Service catalog + detail | Publik — menemukan layanan | P0 | Yes | Yes | Search, category, pagination, detail |
| FR-03 | Request submission (guest) | Pemohon — mengajukan tanpa akun | P0 | Yes | Yes | Dynamic form, files, idempotency, ticket |
| FR-04 | Public tracking + follow-up | Pemohon — melihat progres | P0 | Yes | Yes | Ticket + token, comments, attachments |
| FR-05 | Auth + activation | Staf — akses admin | P0 | Yes | Yes | bcrypt, sessions, 48h activation |
| FR-06 | Admin dashboard | Admin — prioritas kerja | P1 | Yes | Yes | Counts, completion rate, average hours |
| FR-07 | Request lifecycle | Admin — memproses permohonan | P0 | Yes | Yes | Verify/start/info/complete/reject, notes, assignment |
| FR-08 | Service + form builder | Admin — mengelola layanan | P0 | Yes | Yes | Versioned forms, snapshot |
| FR-09 | Documents + versions | Admin/Publik — SOP & panduan | P1 | Yes | Yes | Private files, publish/archive |
| FR-10 | Digital ecosystem CMS | Admin/Publik — katalog produk | P1 | Yes | Yes | Categories, platforms, modules, metrics |
| FR-11 | Publications + pages | Admin/Publik — informasi | P1 | Yes | Yes | Markdown sanitized, SEO |
| FR-12 | Team page CMS | Admin/Publik — profil tim | P2 | Yes | Yes | Groups, photos, publish |
| FR-13 | Users + roles | Superadmin — akses | P0 | Yes | Yes | Invitations, permission matrix |
| FR-14 | Site settings + SEO | Superadmin — identitas & indexing | P1 | Yes | Yes | Contact, hero, metrics, SEO |
| FR-15 | Audit log | Superadmin — akuntabilitas | P1 | Yes | Yes | Sanitized metadata |
| FR-16 | Notifications | Admin — tindakan | P2 | Yes | Yes | Action count + list |
| FR-17 | Global search | Publik — menemukan konten | P1 | Yes | Yes | Services/platforms/documents/articles/pages |
| FR-18 | SEO routes | Publik — indexing | P1 | Yes | N/A — server | Sitemap, robots, headers |
| FR-19 | Security middleware | Semua — perlindungan | P0 | Yes | N/A — server | CSRF, rate limit, headers, upload validation |
| FR-20 | Responsive public + admin shell | Semua — penggunaan seluler | P1 | N/A — UI-only | Yes | 320–1920 px verified |

## 9. API Overview

**Approved convention (D4):** response envelope `{ "data": {}, "message": "", "error": "" }` under `/api/v1/`. Phase 1 serves raw JSON at `/api/*`; the table records the target path with the Phase 1 path. Full request/response shapes are in `docs/API_CONTRACTS.md`.

| Endpoint (target) | Method | Auth | Description |
|---|---|---|---|
| `/api/v1/home` | GET | Public | Home payload: settings, services, platforms, documents, articles, counts |
| `/api/v1/site` | GET | Public | Public site settings |
| `/api/v1/search` | GET | Public | Cross-content search (q, type, page) |
| `/api/v1/services` | GET | Public | Active service list (q, category, page) |
| `/api/v1/services/{slug}` | GET | Public | Service detail + form fields + related docs |
| `/api/v1/services/{slug}/requests` | POST | Public | Submit request (multipart, idempotency header) |
| `/api/v1/platforms` | GET | Public | Published platforms + categories + audiences |
| `/api/v1/platforms/{slug}` | GET | Public | Platform detail |
| `/api/v1/documents` | GET | Public | Published documents (q, category, page) |
| `/api/v1/documents/{slug}` | GET | Public | Document detail |
| `/api/v1/documents/{slug}/download` | GET | Public | Download current published version |
| `/api/v1/articles` | GET | Public | Published articles |
| `/api/v1/articles/{slug}` | GET | Public | Article detail |
| `/api/v1/pages/{slug}` | GET | Public | Custom page |
| `/api/v1/team` | GET | Public | Published team groups/members |
| `/api/v1/team/members/{id}/photo` | GET | Public | Published member photo |
| `/api/v1/track` | POST | Ticket + token | Request detail (public view) |
| `/api/v1/track/respond` | POST | Ticket + token | Requester response |
| `/api/v1/track/comments` | POST | Ticket + token | Requester comment |
| `/api/v1/track/attachments` | POST | Ticket + token | Requester attachment upload |
| `/api/v1/requests/attachments/{id}` | GET | Ticket + token / staff | Private attachment |
| `/api/v1/auth/login` | POST | Public | Staff login |
| `/api/v1/auth/logout` | POST | Session | Logout |
| `/api/v1/auth/session` | GET | Public | Current session or null |
| `/api/v1/auth/activate` | POST | Public | Set password with activation token |
| `/api/v1/admin/dashboard` | GET | requests.read | Admin dashboard counts |
| `/api/v1/admin/requests` | GET | requests.read | Request list + filters |
| `/api/v1/admin/requests/options` | GET | requests.read | Filter options (services, officers, statuses) |
| `/api/v1/admin/requests/{id}` | GET | requests.read | Request detail (internal view) |
| `/api/v1/admin/requests/{id}` | PATCH | requests.manage | Assign/comment/verify/start/request_info/complete/reject |
| `/api/v1/admin/requests/{id}/attachments` | POST | requests.manage | Staff attachment upload |
| `/api/v1/admin/services` | GET, POST | services.manage | List/create services |
| `/api/v1/admin/services/{id}` | GET, PATCH | services.manage | Detail/update service + form version |
| `/api/v1/admin/documents` | GET, POST | documents.manage | List/create documents |
| `/api/v1/admin/documents/options` | GET | documents.manage | Document options |
| `/api/v1/admin/documents/{id}` | GET, PATCH | documents.manage | Detail/update |
| `/api/v1/admin/documents/{id}/versions` | POST | documents.manage | Upload new version |
| `/api/v1/admin/documents/{id}/download` | GET | documents.manage | Download any version |
| `/api/v1/admin/platforms` | GET, POST | content.manage | List/create platforms |
| `/api/v1/admin/platforms/{id}` | PATCH, DELETE | content.manage | Update/delete |
| `/api/v1/admin/platform-categories` | GET, POST | content.manage | List/create categories |
| `/api/v1/admin/platform-categories/{id}` | PATCH, DELETE | content.manage | Update/delete |
| `/api/v1/admin/articles` | GET, POST | content.manage | List/create articles |
| `/api/v1/admin/articles/{id}` | GET, PATCH | content.manage | Detail/update |
| `/api/v1/admin/pages` | GET, POST | content.manage | List/create pages |
| `/api/v1/admin/pages/{id}` | PATCH | content.manage | Update page |
| `/api/v1/admin/team/members` | GET, POST | content.manage | List/create members |
| `/api/v1/admin/team/members/{id}` | GET, PATCH, DELETE | content.manage | Detail/update/delete |
| `/api/v1/admin/team/members/{id}/photo` | GET | content.manage | Member photo |
| `/api/v1/admin/team/groups` | GET, POST | content.manage | List/create groups |
| `/api/v1/admin/team/groups/{id}` | PATCH, DELETE | content.manage | Update/delete |
| `/api/v1/admin/users` | GET, POST | users.manage | List/create users |
| `/api/v1/admin/users/{id}` | PATCH | users.manage | Update status/role/name/email |
| `/api/v1/admin/users/{id}/resend` | POST | users.manage | Resend activation |
| `/api/v1/admin/roles` | GET | roles.manage | List roles/permissions |
| `/api/v1/admin/roles/{slug}` | PATCH | roles.manage | Update admin permissions |
| `/api/v1/admin/settings` | GET, PUT | settings.manage | Site settings |
| `/api/v1/admin/notifications` | GET | notifications.read | Action count + notification list |
| `/api/v1/admin/audit` | GET | audit.read | Audit log |
| `/sitemap.xml` | GET | Public | Sitemap |
| `/robots.txt` | GET | Public | Robots policy |

**Error model:** `400` malformed request, `401` unauthenticated, `403` missing permission/CSRF, `404` not found/not visible, `409` conflict/state machine/duplicate, `410` expired activation, `413` upload too large, `422` validation, `429` rate limited, `502` email delivery failure, `503` service unavailable.

## 10. Data Model

MongoDB collections (Mongoose). Full field-level catalog is in `docs/DATA_CATALOG.md`.

| Collection | Purpose | Key fields / indexes |
|------------|---------|----------------------|
| `roles` | Permission role | slug unique, permissions[], isSystem |
| `permissions` | Permission metadata | key unique, group |
| `users` | Staff accounts | email unique, role enum, active index, passwordHash select:false |
| `sessions` | Login sessions | tokenHash unique, user index, expiresAt TTL |
| `activationtokens` | Activation invites | tokenHash unique, user index, expiresAt TTL, usedAt |
| `services` | Service catalog | slug unique, active index, featured/order index, category enum |
| `serviceformversions` | Versioned forms | (service, version) unique |
| `servicerequests` | Requests | ticketNumber unique, status/service/user/email indexes, idempotencyKey sparse unique |
| `requeststatushistories` | Timeline | request index |
| `requestcomments` | Public/internal notes | request index, visibility |
| `requestattachments` | Private files | request index, storageKey unique |
| `documents` | Documentation | slug unique, category/published/archived indexes |
| `documentversions` | File versions | (document, version) unique, storageKey unique |
| `teamgroups` | Team groups | slug unique, order |
| `teammembers` | Team members | group index, order, published index |
| `platformcategories` | Ecosystem categories | slug unique, order, active |
| `digitalplatforms` | Ecosystem products | slug unique, categoryId/published/featured/status indexes |
| `articles` | Publications | slug unique, status index, publishedAt |
| `pages` | Custom pages | slug unique |
| `sitesettings` | Global settings | key unique (`global`) |
| `notificationlogs` | Email delivery log | template, recipient, status |
| `auditlogs` | Activity log | actor/action/entityType indexes, createdAt |

## 11. Non-Functional Requirements

| Requirement | Target | Implementation |
|-------------|--------|----------------|
| Scale | ≤1.000 staf, ≤5.000 permohonan/tahun | Single Nuxt/Nitro process; MongoDB |
| Public page performance | Target TBD (Q1); SSR + compressed assets | Nitro `compressPublicAssets`, SSR, skeleton states |
| Request submission latency | Target TBD | Multipart validation + local file writes; no queue |
| Availability | Target TBD | Single instance; health/monitoring not implemented |
| Security | OWASP Top 10 / ASVS L1 | RBAC, CSRF marker+Origin, rate limit, bcrypt cost 12, hashed tokens, sanitize-html, private uploads, security headers |
| Upload safety | MIME + extension + magic bytes + max 10 MB/file, 12 MB total | `server/utils/uploads.ts`, Sharp for team photos |
| Privacy | Minimal requester data; no raw tracking token stored | Token hash; attachment authorization |
| Data retention | TBD (Q8) | No retention job for requests/audit/logs |
| Backup | Daily + tested restore (SOP) | Not implemented (Q5) |
| SEO | Public indexable; private noindex | `server/utils/indexing.ts`, sitemap, robots |
| Accessibility | Basic keyboard/focus/contrast; formal audit pending | Focus-visible rings, skip link, labels |
| Responsive | 320–1920 px | Verified 140 combinations |
| Observability | Structured logs + uptime (SOP) | Console logs only; no health endpoint |
| Email | Campus SMTP in production | Nodemailer; dev outbox simulation |

## 12. Third-Party Integrations & Env Vars

| Service | Purpose | Env Vars |
|---------|---------|----------|
| MongoDB | Primary datastore | `NUXT_MONGODB_URI` |
| Session secret | Deployment secret (sessions use random hashed tokens) | `NUXT_SESSION_SECRET` |
| Public site URL | Canonical, sitemap, email links | `NUXT_PUBLIC_SITE_URL` |
| Private uploads | Local filesystem path | `NUXT_UPLOAD_DIR` |
| SMTP | Activation and request notifications | `NUXT_SMTP_HOST`, `NUXT_SMTP_PORT`, `NUXT_SMTP_SECURE`, `NUXT_SMTP_USER`, `NUXT_SMTP_PASS`, `NUXT_EMAIL_FROM` |
| SEO staging switch | Force noindex on staging | `NUXT_PUBLIC_NOINDEX` |
| Seed passwords | Development seed overrides | `SEED_ADMIN_PASSWORD`, `SEED_PETUGAS_PASSWORD` |

## 13. Milestones & Timeline

| Phase | Features | Target | Deliverable |
|-------|----------|--------|-------------|
| **Phase 1 (delivered)** | FR-01…FR-20 as-built | Current `main` | Public site, request portal, admin CMS, tests |
| Phase 2 (planned) | SSO/institutional identity, object storage, malware scanning | TBD | Production hardening per SOP |
| Phase 3 (planned) | Redis/rate-limit store, retention jobs, health/monitoring, formal accessibility audit | TBD | Multi-instance and Gerbang 2 readiness |

**Phase 1 DoD:** `npm run build` succeeds; E2E 105/105, visual 44/44, responsive 140/140; seed documented; known limitations listed in README.

## 14. Out of Scope

- SSO kampus, SIAKAD, Feeder, atau integrasi data institusional (D6).
- Object storage S3/R2/GCS; upload tetap lokal (D6).
- Pemindaian malware/antivirus (D6).
- Multi-instance dengan Redis — belum diimplementasikan; dicatat sebagai Q7, bukan non-goal yang disetujui.
- Aplikasi mobile native, tanda tangan elektronik, pembayaran.
- Audit aksesibilitas formal (axe/WCAG penuh) dan pengujian Safari/Firefox.
- Antivirus, WORM audit storage, dan object storage lifecycle.
- Verifikasi otomatis lisensi email institusi/kepegawaian.

## 15. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Local uploads on one node | Files lost if node dies; no horizontal scaling | Tinggi (fakta) | Backup uploads; object storage in Phase 2 |
| In-memory rate limit | Limits reset on restart and are per process | Tinggi | Single instance; Redis in Phase 3 |
| SMTP not configured | No activation/notification email in production | Sedang | Configure campus SMTP before go-live |
| Email failure blocks activation | Admin cannot activate user | Sedang | 502 with clear message; fix SMTP and retry |
| Seed credentials in repo/docs | Unauthorized access if seed used in prod | Sedang | Never run `seed:reset` in prod; change passwords |
| No malware scanning | Malicious upload stored privately | Sedang | MIME/magic-byte validation; add scanner Phase 2 |
| Audit append-only at app level | Tampering possible with DB access | Rendah | Restrict DB access; immutable store later |
| Manual identity verification | Wrong person gets staff account | Sedang | Admin approval + campus email policy |
| Request status race | Two officers update simultaneously | Sedang | Conditional `findOneAndUpdate` + 409 |
| Form version drift | Old requests break after form edit | Rendah | Snapshot per request |
| Session secret unused for session signing | Misleading config | Rendah | Document; random tokens are hashed server-side |
| Personal GitHub remote | SOP non-compliance | Tinggi | Migrate to GitLab UNU (future) |

## 16. Decisions Record

**Decisions (confirmed in interview)**

- **D1 — Scope:** PRD mendokumentasikan **Phase 1 as-built**.
- **D2 — Root problem:** permohonan layanan terfragmentasi tanpa antrean, SLA, atau audit.
- **D3 — Success metric:** median waktu penyelesaian permohonan.
- **D4 — API convention:** target `/api/v1` + envelope `{data,message,error}`; Phase 1 tetap `/api/*` raw.
- **D5 — Scale:** ≤1.000 staf, ≤5.000 permohonan/tahun.
- **D6 — Non-goals:** SSO/integrasi data institusional, object storage, pemindaian malware.
- **D7 — Owner/approver:** Arief Faqihudin.
- **D8 — Stack (7 Nov 2026):** semua proyek memakai **Go untuk backend dan Vue/Nuxt untuk frontend** (D-STACK). Untuk produk ini artinya dua repo (`fe/`, `be/`) dan migrasi bertahap dari monolit Nitro (plan §13). D8 menutup pertanyaan "apakah backend Go dikerjakan"; yang tersisa adalah model auth dan urutan migrasi.

**Assumptions (correct if wrong)**

- **A1:** Pengajuan publik tetap tanpa akun pada Phase 1; tidak ada akun pemohon.
- **A2:** Deployment Phase 1 adalah satu instance Node dengan MongoDB dan filesystem lokal.
- **A3:** Domain produksi dan SMTP kampus ditetapkan operator (Q2/Q4).
- **A4:** Role non-superadmin hanya `admin`, dengan permission yang dapat diubah.
- **A5:** Metrik capaian hanya tampil bila `verified: true`.
- **A6:** Email development disimulasikan ke `.data/mail-outbox`, bukan dikirim.
- **A7:** Multi-instance/Redis belum diputuskan (Q7).

**Open questions**

- **Q1:** Target waktu penyelesaian dan target performa halaman publik.
- **Q2:** Domain produksi, staging, dan pemilik DNS/TLS.
- **Q3:** Nomor issue dan tanggal persetujuan.
- **Q4:** Penyedia SMTP kampus dan alamat pengirim resmi.
- **Q5:** Pemilik backup/restore dan jadwal uji pemulihan.
- **Q6:** Rencana SSO/integrasi institusional dan object storage.
- **Q7:** Kebutuhan multi-instance/Redis dan waktu implementasinya.
- **Q8:** Kebijakan retensi permohonan, audit, dan lampiran.
- **Q9:** Target audit aksesibilitas formal.
- **Q-API:** Rencana migrasi `/api/*` ke `/api/v1` + envelope.

## 17. Document Info & Sign-off

| Field | Value |
|-------|-------|
| Status | Approved |
| Version | 1.0 |
| Author / Owner | Arief Faqihudin |
| Issue key(s) | TBD (Q3) |
| Approver + date | Arief Faqihudin — 7 November 2026 |
| PRD file location | `docs/PRD.md` |
| Source of truth | Dokumen ini untuk kebutuhan produk; implementasi mengikuti kode Phase 1 |
| Related docs | `README.md`, `docs/SEO.md`, `docs/DESIGN_BRIEF.md`, `docs/IMPLEMENTATION_PLAN.md` |

## 18. Edge Cases Register

| Feature / Entity | Edge Case | Expected Behavior | Status |
|------------------|-----------|-------------------|--------|
| Service request | Service inactive or missing | 404 "tidak ditemukan atau nonaktif" | Decided |
| Service request | `acceptsRequests=false` | 409 | Decided |
| Service request | Form has no fields | 409 | Decided |
| Service request | Missing/invalid idempotency key | 400 | Decided |
| Service request | Same key, same requester | Return existing ticket, `duplicate: true` | Decided |
| Service request | Same key, different requester | 409 duplicate | Decided |
| Service request | Invalid field value | 422 with `errors` map | Decided |
| Service request | File type/extension/magic mismatch | 422 | Decided |
| Service request | File > field max or 10 MB | 422 | Decided |
| Service request | Total multipart > 12 MB | 413 | Decided |
| Service request | Ticket collision | Retry up to 5, then 503 | Decided |
| Tracking | Wrong token | 404 (no existence leak) | Decided |
| Tracking | Ticket missing | 404 | Decided |
| Tracking | Internal comment | Never returned publicly | Decided |
| Tracking | Attachment owned by another requester | 404/403 | Decided |
| Request lifecycle | Invalid transition | 409 | Decided |
| Request lifecycle | Concurrent status change | 409 conditional update | Decided |
| Request lifecycle | Complete without summary | 422 | Decided |
| Request lifecycle | Reject without reason | 422 | Decided |
| Request lifecycle | Terminal status modified | 409 | Decided |
| Request lifecycle | request_info twice | 409 | Decided |
| Assignment | Assign inactive/unknown staff | 422 | Decided |
| Assignment | No `requests.assign` | 403 | Decided |
| Comments | Empty/short note | 422 | Decided |
| Comments | Internal vs public | Visibility enforced server-side | Decided |
| Auth | Wrong credentials | 401 | Decided |
| Auth | Inactive account | 403 | Decided |
| Auth | Requester/non-staff role | 403 | Decided |
| Auth | Session expired | 401 | Decided |
| Auth | CSRF marker missing | 403 on mutation | Decided |
| Activation | Invalid/used/expired token | 410 | Decided |
| Activation | Weak password | 422 | Decided |
| Activation | Email send fails | 502, account stays inactive | Decided |
| User admin | Deactivate self | 409 | Decided |
| User admin | Demote self | 409 | Decided |
| User admin | Last active superadmin | 409 | Decided |
| User admin | Duplicate email | 409 | Decided |
| Role | Edit superadmin role | 409 | Decided |
| Role | Unknown permission | 422 | Decided |
| Service admin | Duplicate slug | 409 | Decided |
| Service admin | Activate without form fields | 422 | Decided |
| Service admin | Duplicate form field key | 422 | Decided |
| Service admin | Invalid regex/pattern | 422 | Decided |
| Service admin | Conditional field points to missing field | 422 | Decided |
| Service admin | >60 fields | 422 | Decided |
| Document | Duplicate slug | 409 | Decided |
| Document | Duplicate version label | 409 | Decided |
| Document | File upload fails after DB write | Cleanup file/DB, error returned | Decided |
| Document | Download unpublished | 404/403 | Decided |
| Platform | Duplicate slug | 409 | Decided |
| Platform | Unknown category | 422 | Decided |
| Platform | Delete with dependent data | No FK in Mongo; platform removed, references by id may dangle | Open → Q8 |
| Article/Page | Duplicate slug | 409 | Decided |
| Team | Photo upload fails | Member not created; file cleaned up | Decided |
| Team | Unpublished member photo | 404 publicly | Decided |
| Settings | Invalid email/URL | 422 | Decided |
| Settings | Metrics not verified | Hidden publicly | Decided |
| Settings | Unknown social URL | Filtered out | Decided |
| Audit | Sensitive metadata keys | Removed before storage | Decided |
| Search | <2 characters | Empty result set | Decided |
| Search | Invalid type filter | 422 | Decided |
| SEO | Non-production/localhost | noindex + robots disallow | Decided |
| SEO | Sitemap in production with localhost | 503 | Decided |
| SEO | Private routes | Always noindex | Decided |
| Uploads | Path traversal in storage key | 400 | Decided |
| Uploads | Local filesystem full | 500; no queue/retry | Open → Q7 |
| Seed | `seed:reset` in production | Must never run | Open → runbook guard |
