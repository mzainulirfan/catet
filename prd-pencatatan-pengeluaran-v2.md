# PRD: Aplikasi Pencatatan Pengeluaran Rutin — Ayah & Umma
**Versi:** 1.0  
**Tanggal:** Juli 2026  
**Status:** Final — Siap diimplementasi  

---

## Daftar Isi
1. [Gambaran Umum](#1-gambaran-umum)
2. [Target Pengguna](#2-target-pengguna)
3. [Tech Stack & Arsitektur](#3-tech-stack--arsitektur)
4. [Struktur Data & Model](#4-struktur-data--model)
5. [Arsitektur Penyimpanan & Sinkronisasi](#5-arsitektur-penyimpanan--sinkronisasi)
6. [Halaman & Fitur Detail](#6-halaman--fitur-detail)
7. [Komponen UI Reusable](#7-komponen-ui-reusable)
8. [Visual Style Guide](#8-visual-style-guide)
9. [Validasi & Error Handling](#9-validasi--error-handling)
10. [Penanganan Edge Case](#10-penanganan-edge-case)
11. [Kriteria Penerimaan (Acceptance Criteria)](#11-kriteria-penerimaan-acceptance-criteria)
12. [Di Luar Cakupan v1](#12-di-luar-cakupan-v1)

---

## 1. Gambaran Umum

### Latar Belakang
Ayah dan Umma ingin mencatat pengeluaran rumah tangga sehari-hari secara konsisten — mulai dari jajan ke warung, beli bahan pokok, bensin, sampai tagihan bulanan. Tantangan utamanya: keduanya mencatat dari HP masing-masing, sehingga perlu cara untuk menggabungkan catatan mereka berdua secara berkala.

### Tujuan Produk
Membangun web app satu file HTML yang:
- Bisa dibuka di browser HP maupun laptop (mobile-first)
- Proses catat transaksi selesai dalam < 10 detik (minimal tap)
- Data tersimpan lokal di device (localStorage), tidak butuh internet setelah dibuka
- Ada mekanisme Export & Import JSON untuk menggabungkan data dua device secara manual

### Satu File, Semua di Dalam
Seluruh aplikasi — HTML, CSS, dan JavaScript — dikemas dalam **satu file `.html`** tunggal. Tidak ada file terpisah, tidak ada server backend, tidak ada build step. Cukup buka file di browser, langsung bisa dipakai.

---

## 2. Target Pengguna

| Pengguna | Perangkat | Pola Penggunaan |
|---|---|---|
| Ayah | HP Android (Chrome) atau laptop | Input transaksi setiap habis belanja, cek laporan mingguan |
| Umma | HP Android (Chrome) atau laptop | Input transaksi harian, cek dashboard bulan ini |

### Kebutuhan Kritis Pengguna
- **Kecepatan input** adalah prioritas #1. Setiap langkah tambahan = kemungkinan besar malas mencatat.
- Tidak perlu login/password — identitas cukup pilih "Ayah" atau "Umma" saat input.
- Format Indonesia: Rupiah (Rp), tanggal dalam Bahasa Indonesia (Senin, 1 Juli 2026).

---

## 3. Tech Stack & Arsitektur

### Stack yang Digunakan

| Layer | Teknologi | Catatan |
|---|---|---|
| Markup | HTML5 | Satu file, tidak ada template engine |
| Styling | CSS custom properties (`:root` vars) + Tailwind CDN | Style utama di `<style>` tag, Tailwind hanya sebagai utilitas tambahan jika perlu |
| Logic | JavaScript ES5/ES6 (vanilla, tanpa framework) | Tidak ada React, Vue, atau jQuery |
| Penyimpanan | `localStorage` browser | Key-value, data dalam format JSON string |
| CDN | Tailwind via `https://cdn.tailwindcss.com` | Dimuat dari CDN, butuh internet saat pertama buka |

### Arsitektur Rendering
Aplikasi menggunakan pola **Single Page App sederhana tanpa framework**:
- Ada satu `<div id="app">` sebagai root
- Setiap navigasi hanya mengganti konten di dalam div tersebut via `innerHTML`
- Tidak ada routing URL — navigasi antar halaman tidak mengubah URL browser
- State disimpan dalam satu objek JavaScript (`state = { page, modal, filter, ... }`)
- Setiap perubahan state → panggil fungsi `render()` → tulis ulang HTML konten halaman

```
Struktur file:
index.html
├── <head> → meta, title, <style> CSS, CDN script
└── <body>
    ├── #app-shell (wrapper max-width 480px, center)
    │   ├── #page-content (konten halaman aktif)
    │   ├── .bottom-nav (navigasi bawah, fixed)
    │   └── .fab (tombol + floating, fixed)
    ├── #modal-root (modal sheet, fixed overlay)
    ├── #toast-root (notifikasi toast, fixed)
    └── <script> (semua logic JS di sini)
```

---

## 4. Struktur Data & Model

Semua data disimpan di `localStorage` dalam format JSON. Ada dua key utama:

| localStorage Key | Isi | Tipe |
|---|---|---|
| `expapp_categories_v1` | Array kategori & sub-kategori | `Category[]` |
| `expapp_transactions_v1` | Array semua transaksi | `Transaction[]` |
| `expapp_lastuser_v1` | Nama pengguna terakhir ("Ayah"/"Umma") | `string` |

### 4.1 Model: Category

```js
{
  id: string,          // contoh: "cat-makanan" atau "id-lxk2a4-r8f3z1" (untuk yang baru ditambah)
  name: string,        // contoh: "Makanan & Minuman"
  color: string,       // kunci warna: "orange" | "blue" | "teal" | "red" | "purple" | "gray" | "green" | "pink" | "yellow" | "indigo"
  subcategories: [
    {
      id: string,      // contoh: "sub-jajan" atau "id-xxx-yyy"
      name: string     // contoh: "Jajan/snack"
    }
  ]
}
```

### 4.2 Model: Transaction

```js
{
  id: string,            // WAJIB UNIK — format: "id-" + timestamp36 + "-" + random6char
                         // contoh: "id-lxk2a4-r8f3z1"
  date: string,          // format: "YYYY-MM-DD", contoh: "2026-07-01"
  amount: number,        // integer, dalam Rupiah. Tidak boleh 0 atau negatif.
  categoryId: string,    // harus cocok dengan salah satu Category.id
  subcategoryId: string, // harus cocok dengan salah satu subcategory.id di dalam category tersebut
  place: string,         // opsional, bisa "" (kosong). contoh: "Alfamart Buah Batu"
  note: string,          // opsional, bisa "" (kosong). catatan bebas.
  inputBy: string,       // "Ayah" atau "Umma"
  createdAt: string      // ISO 8601 timestamp, contoh: "2026-07-01T08:30:00.000Z"
                         // digunakan untuk sorting jika tanggal sama
}
```

### 4.3 Kategori Default (Data Awal)

Jika localStorage kosong (pertama kali dibuka), muat 6 kategori default ini:

| id | name | color | Subcategories |
|---|---|---|---|
| `cat-makanan` | Makanan & Minuman | orange | Jajan/snack, Makan di luar/resto, Bahan pokok, Sayur & lauk harian, Kopi/minuman |
| `cat-transportasi` | Transportasi | blue | Bensin, Parkir/tol, Ojek online, Transportasi umum |
| `cat-rumahtangga` | Rumah Tangga | teal | Listrik/air/gas, Sabun & kebutuhan rumah, Pulsa/internet |
| `cat-kesehatan` | Kesehatan | red | Obat, Dokter/klinik |
| `cat-hiburan` | Hiburan & Langganan | purple | Nonton/bioskop, Streaming/langganan, Hobi |
| `cat-lain` | Lain-lain/Tak terduga | gray | Lainnya |

### 4.4 Palet Warna Kategori

```js
const PALETTE = {
  orange: '#f97316',
  blue:   '#3b82f6',
  teal:   '#14b8a6',
  red:    '#ef4444',
  purple: '#a855f7',
  gray:   '#6b7280',
  green:  '#22c55e',
  pink:   '#ec4899',
  yellow: '#eab308',
  indigo: '#6366f1'
}
```

---

## 5. Arsitektur Penyimpanan & Sinkronisasi

### Alur Penyimpanan

```
Pengguna input transaksi
        ↓
Validasi di JS
        ↓
Tambahkan ke array transactions
        ↓
JSON.stringify → localStorage.setItem('expapp_transactions_v1', ...)
        ↓
Re-render halaman
```

### Alur Export
1. Ambil data dari localStorage: `categories` + `transactions`
2. Buat objek JSON:
   ```json
   {
     "app": "catatan-pengeluaran-ayah-umma",
     "version": 1,
     "exportedAt": "2026-07-01T10:00:00.000Z",
     "categories": [...],
     "transactions": [...]
   }
   ```
3. Buat `Blob` dari JSON string, buat URL temporary (`URL.createObjectURL`)
4. Trigger download via `<a>` tag dengan `download="catatan-pengeluaran-2026-07-01.json"`
5. Tampilkan toast: *"Data berhasil di-export."*

### Alur Import & Merge
1. Pengguna klik tombol Import → trigger `<input type="file" accept="application/json">`
2. Baca file dengan `FileReader.readAsText()`
3. `JSON.parse()` isi file — jika gagal parse, tampilkan toast error
4. Validasi struktur: cek apakah ada field `transactions` (array) — jika tidak, toast error
5. **Merge Transaksi** (aturan: skip duplikat berdasarkan `id`):
   ```
   Untuk setiap transaksi di file import:
     Jika id TIDAK ada di localStorage → tambahkan (ditambah)
     Jika id SUDAH ada di localStorage → lewati (skip)
   ```
6. **Merge Kategori** (aturan: gabungkan sub-kategori baru):
   ```
   Untuk setiap kategori di file import:
     Jika id TIDAK ada di localStorage → tambahkan kategori beserta seluruh sub-kategorinya
     Jika id SUDAH ada → cek setiap sub-kategorinya:
       Jika sub-id TIDAK ada → tambahkan sub-kategori tersebut
       Jika sub-id SUDAH ada → lewati
   ```
7. Simpan hasil merge ke localStorage
8. Re-render seluruh halaman
9. Tampilkan toast hasil: *"Impor selesai: X transaksi baru, Y dilewati (sudah ada)."*

---

## 6. Halaman & Fitur Detail

Aplikasi memiliki **5 halaman utama** yang dikelola via bottom navigation:

```
[ Dashboard ] [ Riwayat ]  [   +   ]  [ Laporan ] [ Lainnya ]
                           (FAB)
```

> **FAB (Floating Action Button)** — tombol bulat di tengah nav bar bawah, selalu terlihat di semua halaman, dipakai untuk membuka form tambah transaksi.

---

### 6.1 Halaman Dashboard

**Tujuan:** Snapshot pengeluaran bulan ini secara sekilas.

**Konten (dari atas ke bawah):**

#### A. Header
- Teks: "Catatan Pengeluaran" (judul besar)
- Teks kecil: nama bulan + tahun (contoh: "Juli 2026")

#### B. Card Total Bulan Ini
- Background: warna primer (indigo/ungu)
- Teks kecil: "Total pengeluaran bulan ini"
- Angka besar: format Rupiah (contoh: **Rp 1.250.000**)
- Teks kecil di bawah: perbandingan dengan bulan lalu (contoh: "↑ 12% dari bulan lalu" atau "↓ 5% dari bulan lalu")
- Jika bulan lalu tidak ada data: jangan tampilkan baris perbandingan

**Logika perhitungan delta:**
```
delta_pct = round(((total_bulan_ini - total_bulan_lalu) / total_bulan_lalu) * 100)
Jika delta_pct >= 0 → "↑ X% dari bulan lalu" (tidak diberi warna, karena di card berwarna)
Jika delta_pct < 0 → "↓ X% dari bulan lalu"
```

#### C. Card Breakdown per Kategori
- Judul: "Per kategori"
- Untuk setiap kategori yang punya transaksi bulan ini, tampilkan:
  - Titik warna kategori + nama kategori (kiri)
  - Jumlah Rupiah (kanan)
  - Bar progress horizontal (tinggi 8px, warna sesuai kategori, lebar = % dari total)
- Urutkan dari terbesar ke terkecil
- Jika tidak ada transaksi: tampilkan teks kosong "Belum ada pengeluaran bulan ini."

#### D. Card Transaksi Terbaru
- Judul: "Transaksi terbaru" + tombol kecil "Lihat semua" (navigasi ke Riwayat)
- Tampilkan maksimal **5 transaksi terakhir** bulan ini, diurutkan dari terbaru
- Setiap baris transaksi: lihat spesifikasi **Komponen TxnRow** di bagian 7
- Jika diklik → buka modal Edit Transaksi
- Jika tidak ada transaksi: tampilkan teks "Belum ada transaksi. Tap tombol + untuk mulai mencatat."

---

### 6.2 Halaman Riwayat

**Tujuan:** Daftar lengkap transaksi dengan kemampuan filter.

#### A. Header
- Teks: "Riwayat"
- Teks kecil: "X transaksi · Rp Y" (X = jumlah setelah filter, Y = total setelah filter)

#### B. Filter Bar (horizontal scroll jika overflow)
Tiga dropdown `<select>`:

| Dropdown | Opsi | Default |
|---|---|---|
| Periode | Bulan ini, Bulan lalu, Semua | Bulan ini |
| Kategori | Semua kategori, + daftar kategori besar | Semua kategori |
| Orang | Semua orang, Ayah, Umma | Semua orang |

**Logika filter:**
- Filter diterapkan secara kumulatif (AND)
- "Bulan ini" = tanggal dimulai dengan `YYYY-MM` saat ini
- "Bulan lalu" = tanggal dimulai dengan `YYYY-MM` bulan sebelumnya

#### C. Daftar Transaksi
- Dikelompokkan per tanggal (group header = nama hari + tanggal lengkap, contoh: "Selasa, 1 Juli 2026")
- Di dalam setiap grup: card putih berisi baris-baris transaksi (komponen TxnRow)
- Urutan: terbaru di atas (sort by date DESC, lalu createdAt DESC jika tanggal sama)
- Jika tidak ada hasil: "Tidak ada transaksi untuk filter ini."
- Jika diklik baris → buka modal Edit Transaksi

---

### 6.3 Modal Tambah / Edit Transaksi

**Tujuan:** Form cepat input transaksi. Muncul sebagai bottom sheet (slide dari bawah).

**Cara membuka:**
- Tombol FAB (+ di tengah nav) → mode Tambah
- Klik baris transaksi di mana saja → mode Edit (data prefill)

**Tampilan overlay:** background gelap semi-transparan di belakang sheet. Klik overlay → tutup modal (data tidak tersimpan).

#### Elemen Form (urutan dari atas ke bawah):

**1. Header Modal**
- Teks: "Tambah Transaksi" atau "Edit Transaksi"
- Tombol X (close) di kanan atas

**2. Pilih Kategori**
- Label: "Kategori"
- Tampilkan semua kategori sebagai **chip/pill button** dalam baris yang bisa wrap
- Setiap chip: titik warna + nama kategori
- Chip yang dipilih: background berubah jadi warna kategori tersebut, teks putih
- Hanya satu yang bisa dipilih (single select)
- Jika belum ada yang dipilih dan form di-submit → tampilkan error di bawah chip list

**3. Pilih Sub-kategori**
- Hanya muncul SETELAH kategori dipilih
- Label: "Sub-kategori"
- Tampilkan sub-kategori dari kategori yang dipilih sebagai chip
- **Urutan chip:** sub-kategori yang paling sering dipakai muncul paling kiri/atas
  - Hitung frekuensi berdasarkan `subcategoryId` dari semua transaksi yang ada
  - Sort descending by frekuensi
- Chip yang dipilih: background indigo, teks putih
- Jika kategori baru dipilih, reset sub-kategori yang sebelumnya dipilih

**4. Jumlah**
- Label: "Jumlah"
- Input field dengan prefix "Rp" di dalam field
- `inputmode="numeric"` agar keyboard angka muncul di HP
- Format otomatis dengan pemisah ribuan saat mengetik: "50000" → "50.000"
- Simpan sebagai integer murni (tanpa titik)
- Placeholder: "0"
- Jika kosong atau 0 saat submit → error

**5. Tanggal**
- Label: "Tanggal"
- Input `type="date"`
- Default: tanggal hari ini (`YYYY-MM-DD`)
- Bisa diubah ke tanggal lain (untuk catat transaksi yang lupa dicatat)

**6. Tempat/Toko**
- Label: "Tempat/toko" + "(opsional)"
- Input text biasa
- Placeholder: "mis. Alfamart, Warung Bu Siti"
- Boleh kosong

**7. Catatan**
- Label: "Catatan" + "(opsional)"
- Input text biasa
- Placeholder: "Catatan tambahan"
- Boleh kosong

**8. Diinput Oleh**
- Label: "Diinput oleh"
- Dua tombol/chip: "Ayah" dan "Umma" berdampingan (50% - 50%)
- Default: nama pengguna terakhir yang disimpan di localStorage (`expapp_lastuser_v1`)
- Saat simpan: update `expapp_lastuser_v1` dengan pilihan saat ini

**9. Tombol Aksi**
- Mode **Tambah**: satu tombol "Simpan" (full width, warna primer)
- Mode **Edit**: tombol "Hapus" (warna merah, kiri) + tombol "Simpan" (warna primer, kanan, flex-grow)

#### Alur Submit (mode Tambah)
1. Validasi semua field wajib (kategori, sub-kategori, jumlah, tanggal)
2. Jika ada error → tampilkan pesan di bawah field yang bermasalah, jangan tutup modal
3. Jika valid → buat objek transaksi baru dengan `id = uid()`, `createdAt = new Date().toISOString()`
4. Push ke array `state.transactions`
5. `saveTransactions(state.transactions)`
6. Update `lastUser` di localStorage
7. Tutup modal
8. Re-render halaman
9. Tampilkan toast: *"Transaksi disimpan."*

#### Alur Submit (mode Edit)
1. Sama dengan Tambah, tapi update objek yang sudah ada berdasarkan `id`
2. Toast: *"Transaksi diperbarui."*

#### Alur Hapus (mode Edit)
1. Tampilkan `confirm()` browser: "Hapus transaksi ini?"
2. Jika OK → hapus dari array, simpan, tutup modal, re-render
3. Toast: *"Transaksi dihapus."*

---

### 6.4 Halaman Laporan

**Tujuan:** Analisis pengeluaran per bulan, bisa navigasi antar bulan.

#### A. Header
- Teks: "Laporan"

#### B. Navigasi Bulan
- Tombol `<` (prev) + nama bulan + tahun (tengah) + tombol `>` (next)
- Default: bulan saat ini
- Klik `<` / `>` → ubah bulan yang ditampilkan, re-render bagian laporan

#### C. Card Ringkasan
- Total pengeluaran bulan dipilih (angka besar)
- Teks perbandingan dengan bulan sebelumnya (berwarna):
  - Naik → teks merah dengan ↑
  - Turun → teks hijau dengan ↓
  - Tidak ada data bulan sebelumnya → teks abu "Belum ada data bulan sebelumnya"
- Baris kontribusi: "Ayah: Rp X · Umma: Rp Y"

#### D. Card Rincian per Kategori
- Judul: "Rincian per kategori"
- Setiap kategori ditampilkan sebagai **collapsible/accordion** (`<details>` HTML):
  - **Header (summary):** ikon chevron ↓ + titik warna + nama kategori + persentase + jumlah Rupiah + bar progress
  - **Body (ketika expand):** daftar sub-kategori beserta totalnya, indent ke kanan
- Urutan: kategori dengan total terbesar di atas
- Hanya tampilkan kategori yang punya transaksi di bulan tersebut

---

### 6.5 Halaman Lainnya

**Tujuan:** Menu akses ke fitur tambahan.

**Konten:**

#### Bagian 1: Pengaturan
- Menu item: "Kelola Kategori" → navigasi ke halaman Pengaturan Kategori

#### Bagian 2: Data
- Menu item: "Export Data (.json)" → jalankan fungsi export
- Menu item: "Import Data (.json)" → buka file picker, lalu jalankan merge

#### Bagian 3: Tentang
- Teks penjelasan singkat tentang aplikasi dan anjuran rutin export

#### Bagian 4: Danger Zone
- Tombol "Hapus Semua Data" (warna merah muda/destructive)
- Perlu **dua konfirmasi** `confirm()` berturut-turut sebelum eksekusi (failsafe)

---

### 6.6 Halaman Pengaturan Kategori

**Tujuan:** CRUD kategori besar dan sub-kategori.

**Navigasi:** dari Lainnya → Kelola Kategori. Header punya tombol `<` untuk kembali ke Lainnya.

**Tampilan per kategori besar:**

Setiap kategori ditampilkan dalam card terpisah, berisi:
1. **Header card:** titik warna + nama kategori (kiri) + tombol ✏️ (rename) + tombol 🗑️ (hapus) (kanan)
2. **Daftar sub-kategori:** setiap sub-kategori ditampilkan dalam satu baris dengan tombol X untuk hapus
3. **Tombol tambah:** "+ Tambah sub-kategori" (full width, style outline)

**Di bawah semua card:**
- Tombol "+ Tambah Kategori Besar" (secondary button, full width)

**Logika:**
- **Rename kategori:** `prompt()` browser minta nama baru → update name → simpan → re-render
- **Hapus kategori:**
  - Cek apakah ada transaksi yang pakai kategori ini
  - Jika ada → alert "Kategori ini masih punya transaksi. Hapus atau pindahkan transaksinya dulu."
  - Jika tidak ada → `confirm()` → hapus → simpan → re-render
- **Tambah sub-kategori:** `prompt()` minta nama → push ke subcategories → simpan → re-render
- **Hapus sub-kategori:**
  - Cek apakah ada transaksi yang pakai sub-kategori ini
  - Jika ada → alert tidak bisa hapus
  - Jika tidak ada → `confirm()` → hapus → simpan → re-render
- **Tambah kategori besar:** `prompt()` nama → assign warna otomatis (rotasi dari PALETTE) → push → simpan → re-render

---

## 7. Komponen UI Reusable

### 7.1 TxnRow (Baris Transaksi)
Dipakai di Dashboard, Riwayat.

```
[ ikon-warna (36px, bg warna kategori, huruf pertama) ] [ info ] [ jumlah ]

info:
  - Baris 1: nama sub-kategori + " · " + nama tempat (jika ada)
  - Baris 2: badge Ayah/Umma + " " + tanggal singkat (contoh: "1 Jul")
```

- Seluruh baris = satu `<button>` yang bisa diklik
- Tap → buka modal Edit

**Badge warna:**
- Ayah: background biru muda, teks biru tua
- Umma: background pink muda, teks pink tua

### 7.2 Toast Notification
- Muncul di atas layar (posisi fixed top, center)
- Warna normal: background hitam/gelap, teks putih
- Warna error: background merah, teks putih
- Animasi: fade in + slide down saat muncul, fade out setelah 2,6 detik
- Bisa muncul bertumpuk jika ada beberapa toast sekaligus

### 7.3 Bottom Sheet Modal
- Background overlay: hitam 45% opacity
- Sheet: background putih, border radius atas kiri + kanan 18px
- Animasi: slide up dari bawah saat buka
- Max height: 92vh, bisa scroll jika konten panjang
- Safe area padding: `env(safe-area-inset-bottom)` untuk iPhone notch/island

### 7.4 FAB (Floating Action Button)
- Posisi: fixed, `bottom: 74px` (di atas nav bar), `left: 50%`, `transform: translateX(-50%)`
- Ukuran: 54x54px, border-radius 50%
- Background: indigo, ikon + putih
- Border: 4px solid background warna halaman (buat "floating" effect)
- Box shadow: `0 6px 16px rgba(79,70,229,.35)`

### 7.5 Chip/Pill Selector
- Border: 1px solid border color
- Border radius: 999px (pil)
- Padding: 9px 14px
- Warna aktif: sesuai konteks (warna kategori untuk pilih kategori, warna primer untuk sub-kategori & pilih user)
- Transisi warna: 0.15s ease

---

## 8. Visual Style Guide

### 8.1 CSS Variables (Design Tokens)

Definisikan di `:root {}`:

```css
--background: #fafafa;
--foreground: #18181b;
--muted: #71717a;
--muted-bg: #f4f4f5;
--border: #e4e4e7;
--card: #ffffff;
--primary: #4f46e5;       /* indigo-600 */
--primary-hover: #4338ca;  /* indigo-700 */
--primary-foreground: #ffffff;
--destructive: #dc2626;
--destructive-bg: #fee2e2;
--radius: 10px;
```

### 8.2 Tipografi

| Elemen | Font Size | Font Weight |
|---|---|---|
| Page title | 20px | 700 |
| Card title / label | 13-14px | 600 |
| Body / list item | 14px | 400-500 |
| Sub-teks / meta | 12-13px | 400 |
| Jumlah besar (total) | 24-28px | 700 |
| Jumlah transaksi | 14px | 600 |

Font: system font stack — `ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, Helvetica, Arial, sans-serif`

### 8.3 Komponen Style

**Card:**
- Background: `var(--card)` (#fff)
- Border: `1px solid var(--border)`
- Border-radius: `var(--radius)` (10px)
- Padding: 16px

**Button primary:** bg indigo, text white, radius 8px, padding 10px 16px  
**Button secondary:** bg `--muted-bg`, border, text foreground  
**Button outline:** bg transparent, border, text foreground  
**Button ghost:** bg transparent, text muted, no border  
**Button destructive:** bg `--destructive-bg`, text `--destructive`

**Input / Select:**
- Border: `1px solid var(--border)`
- Border-radius: 8px
- Padding: 10px 12px
- Font size: 14px
- Focus: outline `2px solid var(--primary)`

### 8.4 Spacing

Gunakan kelipatan 4px: 4, 8, 12, 16, 20, 24, 32px

### 8.5 App Shell

```css
.app-shell {
  max-width: 480px;
  margin: 0 auto;
  min-height: 100vh;
  background: var(--background);
  padding-bottom: 96px; /* ruang untuk bottom nav */
}
```

### 8.6 Animasi

- Page load: `fadeIn` 0.18s ease (opacity 0 → 1)
- Modal open: `slideUp` 0.22s ease (translateY(24px)+opacity 0.4 → translateY(0)+opacity 1)
- Transisi button/chip hover: 0.15s
- `prefers-reduced-motion`: jika aktif, semua durasi animasi = 0.001ms

---

## 9. Validasi & Error Handling

### 9.1 Validasi Form Transaksi

| Field | Aturan | Pesan Error |
|---|---|---|
| Kategori | Harus dipilih (tidak boleh null) | "Pilih kategori dulu" |
| Sub-kategori | Harus dipilih SETELAH kategori dipilih | "Pilih sub-kategori" |
| Jumlah | Harus angka positif > 0 | "Masukkan jumlah yang valid" |
| Tanggal | Tidak boleh kosong | "Pilih tanggal" |

- Error message muncul sebagai teks merah kecil di bawah field yang bermasalah
- Semua error ditampilkan sekaligus (bukan satu per satu)
- Modal tidak tertutup saat ada error
- Setelah pengguna memperbaiki field → error hilang saat submit ulang (bukan real-time)

### 9.2 Validasi Import

| Kondisi | Tindakan |
|---|---|
| File bukan JSON (parse error) | Toast error: "File tidak valid / bukan format JSON." |
| JSON valid tapi tidak punya field `transactions` | Toast error: "Struktur file tidak dikenali." |
| File kosong / 0 transaksi | Toast sukses dengan "0 transaksi baru, 0 dilewati" |

### 9.3 Validasi Hapus Kategori/Sub-kategori

| Kondisi | Tindakan |
|---|---|
| Kategori masih punya transaksi | Alert: "Kategori ini masih punya transaksi. Hapus atau pindahkan transaksinya dulu sebelum menghapus kategori." |
| Sub-kategori masih punya transaksi | Alert: "Sub-kategori ini masih punya transaksi. Hapus atau pindahkan transaksinya dulu." |

### 9.4 localStorage Tidak Tersedia

Jika localStorage tidak bisa diakses (misalnya mode incognito dengan setting ketat):
- Wrap semua operasi localStorage dalam `try/catch`
- Jika error → aplikasi tetap jalan dengan data in-memory saja
- Beri tanda (tidak perlu UI khusus untuk v1)

---

## 10. Penanganan Edge Case

| Situasi | Perilaku yang Diharapkan |
|---|---|
| Buka aplikasi pertama kali (localStorage kosong) | Load DEFAULT_CATEGORIES, transaksi = array kosong |
| localStorage rusak / tidak valid JSON | `try/catch` di fungsi load → fallback ke default/kosong |
| Kategori dihapus tapi ada transaksi lama yang pakai kategori itu | Transaksi tetap ada, saat ditampilkan: kategori = "Tidak diketahui", warna = abu |
| Import file dari format aplikasi lain (struktur berbeda) | Cek field `transactions` ada dan array → tolak jika tidak ada |
| Import file yang sama dua kali | Semua transaksi akan di-skip (sudah ada), tidak ada duplikat. Toast: "X transaksi baru, Y dilewati" |
| Dua transaksi di tanggal yang sama, urutkan how? | Sort by `createdAt` descending |
| Jumlah input pakai huruf/simbol | Strip semua non-angka sebelum parse: `input.replace(/[^0-9]/g, '')` |
| Input jumlah = 0 setelah di-strip | Validasi error: "Masukkan jumlah yang valid" |
| Navigasi bulan di laporan ke bulan depan | Boleh, tapi total = 0 dan pesan kosong ditampilkan |
| Kategori tidak punya sub-kategori (baru dibuat, belum isi sub-kat) | Di form tambah transaksi, section sub-kategori tetap muncul tapi kosong dengan teks petunjuk |
| Teks nama kategori/sub-kategori sangat panjang | Gunakan `white-space: nowrap; overflow: hidden; text-overflow: ellipsis` pada elemen list |
| Banyak transaksi (100+) di riwayat | Tidak ada pagination di v1 — semua ditampilkan. Render sederhana sudah cukup cepat untuk ratusan item. |

---

## 11. Kriteria Penerimaan (Acceptance Criteria)

### AC-01: Tambah Transaksi
- [ ] FAB (+) di semua halaman membuka modal
- [ ] Bisa pilih kategori (chip), lalu sub-kategori muncul
- [ ] Sub-kategori sering dipakai muncul lebih dulu
- [ ] Input jumlah: keyboard angka muncul di HP, format rupiah otomatis
- [ ] Tanggal default = hari ini
- [ ] Bisa pilih Ayah/Umma
- [ ] Validasi wajib bekerja (kategori, sub, jumlah, tanggal)
- [ ] Setelah simpan: modal tutup, toast muncul, transaksi ada di dashboard & riwayat

### AC-02: Edit & Hapus Transaksi
- [ ] Klik baris transaksi di mana saja → modal edit terbuka dengan data prefill
- [ ] Bisa ubah semua field
- [ ] Simpan → perubahan tersimpan, toast "Transaksi diperbarui"
- [ ] Hapus → konfirmasi muncul, setelah OK transaksi hilang, toast "Transaksi dihapus"

### AC-03: Dashboard
- [ ] Total bulan ini benar
- [ ] Perbandingan % dengan bulan lalu benar (atau tidak muncul jika tidak ada data lalu)
- [ ] Bar per kategori, panjangnya proporsional
- [ ] 5 transaksi terbaru tampil, klik → modal edit

### AC-04: Riwayat
- [ ] Filter periode, kategori, dan orang bekerja secara kumulatif
- [ ] Jumlah transaksi & total di header ikut berubah sesuai filter
- [ ] Transaksi dikelompokkan per tanggal

### AC-05: Laporan
- [ ] Navigasi prev/next bulan bekerja
- [ ] Delta dibandingkan bulan sebelumnya tampil dan berwarna (merah/hijau)
- [ ] Accordion sub-kategori bisa expand/collapse
- [ ] Kontribusi Ayah & Umma tampil terpisah

### AC-06: Kelola Kategori
- [ ] Bisa rename kategori besar
- [ ] Tidak bisa hapus kategori/sub-kategori yang masih punya transaksi (ada alert)
- [ ] Bisa hapus jika tidak ada transaksi (ada confirm)
- [ ] Bisa tambah sub-kategori baru via prompt
- [ ] Bisa tambah kategori besar baru via prompt (warna otomatis)

### AC-07: Export
- [ ] Klik Export → file JSON terunduh otomatis
- [ ] Nama file: `catatan-pengeluaran-YYYY-MM-DD.json`
- [ ] Isi JSON valid dan berisi `categories` + `transactions`

### AC-08: Import & Merge
- [ ] Klik Import → file picker terbuka (filter .json)
- [ ] Setelah pilih file valid → data tergabung tanpa duplikat
- [ ] Toast menampilkan berapa transaksi yang ditambah vs dilewati
- [ ] File tidak valid → toast error yang jelas
- [ ] Import file yang sama 2x → tidak ada duplikat

### AC-09: Hapus Semua Data
- [ ] Ada 2 konfirmasi berturut-turut sebelum eksekusi
- [ ] Setelah hapus: transaksi = kosong, kategori kembali ke default

### AC-10: UI & UX
- [ ] Tampilan rapi di layar HP (lebar 360-414px) dan laptop
- [ ] Semua tombol mudah ditap (min 44x44px touch target)
- [ ] Format Rupiah konsisten di seluruh aplikasi
- [ ] Toast muncul setelah setiap aksi penting
- [ ] Animasi halus (modal slide up, toast fade)
- [ ] Tidak ada error di browser console saat penggunaan normal

---

## 12. Di Luar Cakupan v1

Fitur-fitur berikut **tidak dibangun** di versi ini. Bisa jadi roadmap v2:

| Fitur | Alasan Ditunda |
|---|---|
| Sinkronisasi real-time antar device | Butuh backend server |
| Budget/limit per kategori + notifikasi | Menambah kompleksitas UI |
| Multi-currency | Tidak relevan untuk kebutuhan saat ini |
| Login/autentikasi formal | Overkill untuk 2 pengguna |
| Pagination riwayat | Belum diperlukan di awal |
| Export ke format Excel/CSV | Bisa export JSON lalu convert manual |
| Dark mode | Belum diminta |
| PWA / install ke home screen | Bisa ditambah dengan satu baris manifest |
| Recurring transaction (transaksi berulang otomatis) | Kompleksitas tinggi |

---

*Dokumen ini adalah spesifikasi lengkap v1. Developer junior harus membaca seluruh dokumen sebelum mulai coding. Jika ada ambiguitas, gunakan contoh yang diberikan sebagai acuan.*
