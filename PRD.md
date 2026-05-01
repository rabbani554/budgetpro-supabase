# PRD — BUDGETpro
**Product Requirements Document (Berbasis Kode Aktual)**
Versi: 1.0 | Tanggal: 2026-05-01 | Penulis: RBN

---

## 1. Ringkasan Produk

**BUDGETpro** adalah aplikasi web mobile-first untuk manajemen anggaran perjalanan dinas / event, dilengkapi fitur LPJ (Laporan Pertanggungjawaban) berbasis foto struk. Aplikasi ini berjalan sebagai **single-file HTML** dengan React di browser, backend **Supabase** (PostgreSQL + Storage + Realtime).

**Tagline:** *"semoga membantu kamu mempertanggungjawabkan budget yang sudah dibuat"*

**Target pengguna:** Tim internal organisasi / HR / Finance untuk mencatat, memantau, dan melaporkan realisasi anggaran perjalanan dinas atau event.

---

## 2. Tech Stack (As-Built)

| Layer | Teknologi | Versi |
|---|---|---|
| UI Framework | React (UMD CDN) | 18 |
| Styling | Tailwind CSS (CDN) | Latest |
| JSX Transpiler | Babel Standalone (CDN) | Latest |
| Backend / DB | Supabase (PostgreSQL) | v2 |
| Storage | Supabase Storage | v2 |
| Realtime | Supabase Realtime Channels | v2 |
| Icons | FontAwesome 6 (CDN) | 6.4.2 |
| PDF Export | html2pdf.js (CDN) | 0.10.1 |
| Excel I/O | SheetJS / XLSX (CDN) | 0.18.5 |
| Session | localStorage | Browser API |

**Arsitektur:** Single-page app (SPA) dalam satu file `index.html`. Tidak ada build step, tidak ada bundler, tidak ada server-side rendering.

---

## 3. Skema Database Supabase

### Tabel: `users`
| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | int/serial | Primary key (auto) |
| `username` | text | Unik, dipakai untuk login |
| `password` | text | Plain text (no hashing — noted as tech debt) |
| `role` | text | `'admin'` atau `'user'` |

### Tabel: `projects`
| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | text | ID unik (Date.now().toString()) |
| `name` | text | Nama event / trip |
| `target` | text | Tujuan / keterangan |
| `startDate` | text | Format `YYYY-MM-DD` |
| `endDate` | text | Format `YYYY-MM-DD` |
| `owner` | text | Username pembuat (FK ke users.username) |
| `items` | jsonb | Array item budget (lihat sub-skema di bawah) |

### Sub-skema: `items` (JSONB array di dalam projects)
```json
[
  {
    "id": "string (Date.now)",
    "name": "string",
    "qty": "number",
    "unit": "string",
    "budget": "number (per unit)",
    "notes": "string (opsional)",
    "actuals": [
      {
        "id": "string (Date.now)",
        "amount": "number",
        "description": "string",
        "receiptUrl": "string (URL Supabase Storage | null)"
      }
    ],
    "status": "pending | under-budget | on-budget | over-budget"
  }
]
```

### Storage Bucket: `receipts`
- Path: `{project_id}/{item_id}_{timestamp}.jpg`
- Akses: public URL
- Tipe konten: `image/jpeg`

---

## 4. Logika Bisnis Inti

### 4.1 Kalkulasi Status Item
```
plannedTotal = item.qty × item.budget
actualTotal  = SUM(actuals[].amount)

status:
  → 'pending'       jika actuals kosong
  → 'over-budget'   jika actualTotal > plannedTotal
  → 'under-budget'  jika actualTotal < plannedTotal
  → 'on-budget'     jika actualTotal === plannedTotal
```

### 4.2 Kalkulasi Ringkasan Project
```
totalPlan   = SUM(items[].qty × items[].budget)
totalActual = SUM(items[].actuals[].amount)
variance    = totalPlan - totalActual
  → variance >= 0 : sisa (aman)
  → variance < 0  : over budget (minus)
```

### 4.3 Aturan Akses (Role-Based)
```
admin → bisa melihat SEMUA project (read-only untuk project milik user lain)
user  → hanya bisa melihat & edit project milik sendiri (owner === username)

isReadonly = (role === 'admin') && (project.owner !== currentUser.username)
```

### 4.4 Sinkronisasi Realtime
- Supabase Realtime Channels berlangganan perubahan pada tabel `users` dan `projects`
- Setiap perubahan dari user lain otomatis memperbarui state React
- State `activeProject` di-sync ulang saat `projects` berubah

---

## 5. Fitur Aplikasi (Feature Inventory)

### F-01: Autentikasi
- **Login** dengan username + password
- **Register** dengan pilihan role (user / admin HR-Finance)
- Session disimpan di `localStorage` (key: `budget_current_user`)
- Default admin `admin/admin123` dibuat otomatis jika DB kosong
- Logout menghapus session dan reset ke dashboard

### F-02: Dashboard
- Menampilkan daftar project sesuai role
- Setiap card project menampilkan: nama, tanggal, jumlah item, rencana, terpakai, sisa/over
- Badge status: hijau (semua item done) / oranye (ada yang pending)
- Admin melihat badge nama pemilik project
- Tombol FAB "Buat Trip Baru" (hanya tampil, bisa diklik siapa saja yang login)

### F-03: Buat Project Baru
- Input: nama event, tanggal mulai, tanggal selesai, target (opsional)
- Dua mode pembuatan:
  - **Manual kosong**: langsung ke halaman project detail
  - **Import Excel**: mapping kolom `Item`, `Qty`, `Unit`, `Budget Satuan`, `Keterangan`
- Download template Excel tersedia

### F-04: Detail Project
- Edit inline: nama, tanggal, target (disabled jika readonly)
- Summary card: Rencana, Aktual, Variance (berwarna hijau/merah)
- Daftar semua item budget dengan indikator warna status di sisi kiri
- Preview thumbnail struk foto (max 3 + counter lebih)
- Tombol "Detail Laporan" dan "Export Excel"
- Tombol FAB "Tambah Biaya"
- Tombol hapus project (merah, disabled jika readonly)

### F-05: Manajemen Item Budget (Rencana)
- **Tambah** item: nama, qty, satuan, budget per unit, catatan
- **Edit** item yang sudah ada
- **Hapus** item dengan konfirmasi
- Preview real-time total plan = qty × budget
- Tidak bisa edit/hapus jika readonly

### F-06: Input Aktual / LPJ
- Per item budget, bisa input multiple transaksi
- Setiap transaksi: nominal (formatted rupiah saat ketik), keterangan, foto struk
- Upload foto: via kamera (capture) atau galeri
- Foto diunggah ke Supabase Storage dan disimpan URL-nya
- Preview foto sebelum simpan
- Hapus transaksi dari daftar (di dalam modal)
- Ringkasan live: budget, total terpakai, sisa/over
- Simpan sekaligus seluruh daftar transaksi

### F-07: Sidebar Detail Item
- Tampilkan info lengkap item: nama, qty, unit, budget, status badge
- Ringkasan: budget rencana, terpakai aktual, variance
- Riwayat semua transaksi dengan foto struk (klik buka di tab baru)
- Tombol "Tambah Transaksi LPJ Baru" (disabled jika readonly)

### F-08: Laporan Lengkap (Full Report)
- Halaman laporan formal: nama event, tanggal, PJ, target
- Summary total rencana, total aktual, variance
- Tabel rincian per item: alokasi rencana, riwayat transaksi, total terpakai, sisa/over
- Bagian tanda tangan (muncul saat export PDF): "Dibuat Oleh" + "Mengetahui (Finance/HRD)"
- Lampiran foto struk (grid 2 kolom, dengan label item + keterangan + nominal)

### F-09: Export
- **Export PDF** (html2pdf.js): menyertakan laporan + tanda tangan + lampiran foto
- **Export CSV/Excel** (link download): header project, tabel item, detail transaksi

### F-10: Konfirmasi Hapus Universal
- Modal konfirmasi untuk: hapus project, hapus item, hapus transaksi
- Teks konfirmasi disesuaikan per tipe

---

## 6. Komponen UI (Component Inventory)

| Komponen | Deskripsi |
|---|---|
| `AuthScreen` | Layar login + register (tab toggle) |
| `Dashboard` | Daftar project + header user |
| `NewProjectModal` | Bottom sheet buat project + import Excel |
| `ProjectDetail` | Halaman detail project + daftar item |
| `ItemPlanModal` | Bottom sheet tambah/edit item budget |
| `ActualInputModal` | Bottom sheet input transaksi LPJ + upload foto |
| `ItemDetailSidebar` | Slide-in panel riwayat transaksi per item |
| `FullReportSidebar` | Full-screen laporan formal + export PDF |
| `ConfirmModal` | Modal konfirmasi hapus universal |

**Animasi:**
- `animate-fade-in`: `opacity 0→1` (0.2s)
- `animate-slide-up`: `translateY(100%)→0` (0.3s, spring)
- `animate-slide-left`: `translateX(100%)→0` (0.35s, spring)

---

## 7. State Management

Semua state dikelola di komponen `App` dengan `useState` React:

```
projects        []        — daftar semua project dari Supabase
users           []        — daftar semua user dari Supabase
currentUser     null      — session user aktif { username, role }
isInitializing  true      — flag loading awal
currentView     'dashboard' | 'project_detail'
activeProject   null      — project yang sedang dibuka
showNewProjectModal  bool
showItemModal        bool
showActualModal      bool
showDetailSidebar    bool
showFullReportSidebar bool
selectedItem    null      — item yang sedang dipilih
deleteConfig    { isOpen, type, payload, title, message }
```

---

## 8. Navigasi & Alur Pengguna

```
[Login/Register]
      ↓
[Dashboard] ──────────────────────────────────────────────────────────┐
  │                                                                    │
  ├── Tap project → [Project Detail]                                  │
  │     │                                                              │
  │     ├── Tap item → [Item Detail Sidebar]                          │
  │     │     └── Tap "Tambah Transaksi" → [Actual Input Modal]       │
  │     │                                                              │
  │     ├── Tap "Input Aktual" (per item) → [Actual Input Modal]      │
  │     ├── Tap "Edit" (per item) → [Item Plan Modal]                 │
  │     ├── Tap "+" (FAB) → [Item Plan Modal - tambah baru]           │
  │     ├── Tap "Detail Laporan" → [Full Report Sidebar]              │
  │     │     └── Tap "Cetak PDF" → download PDF                      │
  │     └── Tap "Export Excel" → download CSV                         │
  │                                                                    │
  └── Tap "Buat Trip Baru" → [New Project Modal]                      │
        ├── Manual → [Project Detail] (kosong)                        │
        └── Import Excel → [Project Detail] (dengan items)            │
                                                                       │
[Logout] ─────────────────────────────────────────────────────────────┘
```

---

## 9. Tech Debt & Catatan Keamanan

| # | Item | Risiko | Prioritas |
|---|---|---|---|
| TD-01 | Password disimpan plain text di DB | Tinggi (data breach) | P1 |
| TD-02 | Supabase credentials di-hardcode di HTML publik | Sedang (anon key, bukan service key) | P2 |
| TD-03 | ID pakai `Date.now()` — bisa collision di multi-user | Rendah | P3 |
| TD-04 | Tidak ada validasi tipe file saat upload (hanya `accept`) | Rendah | P3 |
| TD-05 | Tidak ada pagination di dashboard | Sedang (performa di >50 project) | P2 |
| TD-06 | Tidak ada RLS (Row Level Security) di Supabase | Tinggi (user bisa akses DB langsung) | P1 |
| TD-07 | Tidak ada error boundary di React | Rendah | P3 |

---

## 10. Roadmap Fitur Potensial

| ID | Fitur | Deskripsi | Estimasi |
|---|---|---|---|
| R-01 | Password hashing | Ganti plain text dengan bcrypt / Supabase Auth | Medium |
| R-02 | Filter & search dashboard | Filter project by date, owner, status | Small |
| R-03 | Kategori item | Tambah field kategori (Transportasi, Akomodasi, dll) | Small |
| R-04 | Budget approval flow | Admin approve/reject sebelum project bisa di-LPJ | Large |
| R-05 | Notifikasi over-budget | Alert realtime saat item melebihi anggaran | Medium |
| R-06 | Multi-foto per transaksi | Saat ini hanya 1 foto per transaksi | Small |
| R-07 | Dark mode | Toggle tema gelap | Small |
| R-08 | Summary chart | Pie/bar chart rencana vs aktual per item | Medium |
| R-09 | Duplicate project | Copy template project yang sudah ada | Small |
| R-10 | Supabase Auth | Migrasi auth ke Supabase Auth resmi | Large |
| R-11 | PWA / install to homescreen | Manifest + service worker | Medium |
| R-12 | Komentar / catatan approval | Admin bisa tambah komentar di project | Medium |

---

## 11. Konvensi Kode

- Seluruh aplikasi dalam satu file `index.html` — komponen didefinisikan sebagai function di dalam fungsi `App()`
- Format mata uang menggunakan `Intl.NumberFormat('id-ID', { currency: 'IDR' })`
- Tanggal disimpan sebagai string `YYYY-MM-DD`, ditampilkan apa adanya
- Icon menggunakan FontAwesome via wrapper komponen `FAIcon`
- Warna status: hijau (aman/under), merah (over), biru (on-budget), abu (pending)
- Semua operasi DB adalah async/await langsung ke Supabase client
- Update optimistic: state lokal diperbarui langsung, lalu Supabase dipanggil paralel

---

*Dokumen ini dibuat otomatis dari analisis kode `index.html` pada 2026-05-01. Update dokumen ini setiap kali ada perubahan fitur signifikan.*
