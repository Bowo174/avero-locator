# 📘 Product Requirement Document (PRD)

## AVERO Locator – Sistem Pencarian Lokasi Produk Internal

## 1. Background

AVERO KOMPUTER adalah toko layanan IT dan ritel yang menjual berbagai produk komputer, aksesoris, hingga perlengkapan CCTV. Dengan banyaknya variasi barang yang ada di toko, karyawan baru sering kali mengalami kesulitan untuk menghafal nama produk, bentuk fisik, dan lokasi persisnya di dalam toko. Saat ini, belum ada sistem katalog yang mendokumentasikan tata letak barang secara terpusat.

Website **AVERO Locator** ini akan berfungsi sebagai "Katalog Perpustakaan" digital khusus untuk internal toko—sebuah platform praktis yang:

* Menampilkan informasi lokasi barang secara spesifik (Etalase/Rak dan Baris).
* Menyediakan visual/gambar produk untuk menghindari kesalahan pengambilan barang.
* Mempercepat proses adaptasi karyawan baru tanpa harus terus bertanya kepada karyawan senior.
* Menggunakan antarmuka yang sangat sederhana agar pencarian bisa dilakukan dalam hitungan detik saat melayani pelanggan.

---

## 2. Product Goals

1. **Meningkatkan efisiensi waktu** karyawan dalam mencari produk di toko.
2. **Membangun kemandirian karyawan baru** melalui sistem pencarian yang intuitif dan mandiri.
3. **Menyediakan fleksibilitas manajemen letak** agar penambahan atau perubahan letak etalase/rak di masa depan dapat dilakukan secara dinamis (CRUD) tanpa mengubah *source code*.
4. **Menciptakan antarmuka yang ringan dan cepat (Zero-reload)** menggunakan teknologi AJAX dan DataTables.

---

## 3. Target Users

* **Staff / Karyawan Baru** — Pengguna utama yang akan menggunakan fitur pencarian untuk menemukan letak dan wujud barang di toko.
* **Admin / Pemilik Toko** — Pengguna yang mengelola *database* produk dan mengatur master data lokasi (Etalase & Rak).

---

## 4. Design & Visual Style

### Visual Direction

* **Design language:** Sederhana, fungsional, dan bersih (Minimalist).
* **Mood:** Praktis, cepat, *user-friendly*.
* **Primary accent color:** Warna netral (Putih, Abu-abu terang) dengan aksen biru atau warna identitas AVERO KOMPUTER.
* **Typography:**
* Headings: *Inter* atau *Roboto* (tegas, modern).
* Body text: *Open Sans* (tingkat keterbacaan tinggi pada layar kecil maupun besar).



### Design Principles

* **Kecepatan utama:** Hindari elemen visual yang tidak perlu agar *loading* data secepat mungkin.
* **Fokus pada Interaksi Klik:** Tabel dibuat seminimal mungkin (hanya nama produk). Interaksi detail disembunyikan dalam klik untuk menjaga UI tetap rapi.
* **Interaksi tanpa berpindah halaman:** Menggunakan *Modal Pop-up* untuk melihat gambar dan detail agar pencarian tidak terputus.

---

## 5. Website Structure

### 1. Home / Halaman Pencarian (Staff)

* Kolom pencarian (*Search bar*) utama yang besar dan jelas di bagian atas.
* Tabel Data (Yajra DataTables) hanya menampilkan: **Nama Produk** (dan mungkin kategori singkat).
* **Interaksi Utama:** Saat baris/nama produk diklik, sistem akan memunculkan *Modal Pop-up* (Detail Card).
* **Isi Modal Detail:** Menampilkan **Gambar Produk** di bagian atas, diikuti dengan teks lokasi yang informatif di bawahnya (Contoh: *"PLC Splitter BOX (etalase E baris ke 1)"*).

### 2. Login Admin

* Halaman autentikasi sederhana khusus untuk mengelola data.

### 3. Dashboard Admin

* Ringkasan total produk dan total lokasi yang terdaftar.

### 4. Manajemen Lokasi (Master Data)

* Daftar lokasi fisik di toko (Etalase, Rak Biasa, Rak Laptop).
* Konfigurasi batas maksimal baris untuk setiap lokasi.

### 5. Manajemen Produk

* Daftar seluruh barang.
* Formulir input nama produk, lokasi, letak baris, dan upload gambar.

---

## 6. Key Features

| Feature | Description |
| --- | --- |
| **Instant AJAX Search** | Pencarian produk langsung memfilter tabel tanpa *reload* halaman menggunakan Yajra DataTables. |
| **Dynamic Location Management** | CRUD untuk membuat Etalase atau Rak baru secara fleksibel beserta jumlah barisnya. |
| **Detail Pop-up (Modal)** | Klik pada produk membuka pop-up berisi foto produk dan kalimat instruksi lokasi (Contoh: etalase E baris ke 1). |
| **Relational Product Mapping** | Sistem *dropdown* dinamis saat Admin menambah produk (pilih lokasi -> pilih baris sesuai kapasitas rak). |
| **Simple Admin Panel** | Antarmuka *backend* khusus Admin untuk mengelola katalog tanpa fitur *stock management*. |
| **Responsive Design** | Nyaman diakses baik dari PC kasir maupun *smartphone* karyawan saat berjalan di area toko. |

---

## 7. Technical Specifications (Laravel Environment)

| Category | Specification |
| --- | --- |
| **Language** | PHP 8.3 |
| **Framework** | Laravel (Versi 12) |
| **Frontend** | **Laravel Blade + Bootstrap 5** |
| **UI Interactivity** | **Vanilla JavaScript / jQuery** untuk menangani *On-click event* baris DataTable ke Modal Bootstrap. |
| **AJAX CRUD** | **jQuery AJAX** untuk proses manipulasi data *seamless* di sisi Admin. |
| **DataTable Integration** | **Yajra Laravel DataTables** untuk *server-side processing* pencarian produk. |
| **Database** | MySQL (Berjalan optimal di *environment* Laragon). |
| **Admin Panel** | Custom-built menggunakan Blade templates (tanpa Filament/Nova agar lebih ringan dan sesuai kebutuhan). |
| **Authentication** | Laravel session/auth bawaan (hanya untuk Admin). |
| **Storage** | Local storage (symlink `/storage/app/public/products` untuk gambar barang). |
| **Security** | CSRF protection, pencegahan SQL Injection (via ORM). |

---

## 8. Admin Panel Features

---

### 8.1 Access & Authentication

* Admin login menggunakan email dan password.
* Hanya Admin yang dapat mengakses rute `/admin/*`. Halaman depan (`/`) terbuka bebas untuk jaringan toko.

---

### 8.2 Dashboard

* Tampilan cepat jumlah Total Produk dan Total Lokasi/Rak.
* Akses cepat (Shortcut) untuk "Tambah Produk Baru".

---

### 8.3 Content Management Modules

#### a. Location Management (Master Letak)

* CRUD untuk data lokasi fisik di toko.
* *Default Data* awal:
* Etalase A (2 Baris)
* Etalase B (4 Baris)
* Etalase C (4 Baris)
* Etalase D (3 Baris)
* Etalase E (4 Baris)
* Rak A (4 Baris)
* Rak B (4 Baris)
* **Rak Laptop (3 Baris)**


* Validasi proteksi: Lokasi tidak dapat dihapus jika masih ada produk yang menempatinya.

#### b. Product Management (Katalog Barang)

* CRUD untuk inventaris barang.
* Fields:
* Nama Produk
* Lokasi (*Dropdown* dari tabel Location)
* Baris Letak (*Dropdown* dinamis bergantung pada batas baris lokasi yang dipilih)
* Upload Gambar


* Fitur edit untuk memindahkan letak barang jika ada penataan ulang toko.

---

### 8.4 Admin Panel UI & UX

* Navigasi Sidebar sederhana: Dashboard, Manajemen Lokasi, Manajemen Produk.
* Tampilan tabel menggunakan DataTables.
* *Image Preview* sebelum file gambar di-upload.

---

### 8.5 Route Structure

```text
/ (Halaman Staff - Tampilan Pencarian Produk)
/login (Autentikasi Admin)

/admin
├── /dashboard
├── /locations
│   ├── create / edit / delete
└── /products
    ├── create / edit / delete

```

---

## 9. Project Timeline (Estimation)

| Phase | Duration | Deliverables |
| --- | --- | --- |
| Database & Schema | 3 Hari | Desain relasi lokasi-produk & setup lokal |
| Backend Logic (Admin) | 1 Minggu | CRUD Master Lokasi, CRUD Produk, Upload File |
| Frontend (Staff View) | 4 Hari | Integrasi Yajra DataTables, onClick event Modal |
| Testing & Deployment | 2 Hari | Input data asli toko & uji coba pencarian |

**Total estimated time:** ~2.5 Minggu

---

## 10. Success Metrics

* Karyawan baru dapat mengetahui letak dan bentuk fisik barang dengan **1 kali pencarian dan 1 kali klik**.
* Waktu *loading* pencarian data di tabel sangat cepat (kurang dari 1 detik).
* Penambahan rak baru (seperti rak aksesoris di masa depan) dapat langsung ditambahkan oleh admin melalui aplikasi.

---

## 11. Additional Notes

* **Penyimpanan Gambar:** Ukuran gambar harus dibatasi agar server lokal tidak cepat penuh dan *loading* Modal tetap ringan.
* **Format Teks Modal:** Teks lokasi di dalam modal diformat secara otomatis oleh sistem (menggabungkan nama produk, nama lokasi, dan nomor baris) menjadi *string* yang mudah dibaca, contoh: `"Nama Produk (Nama Lokasi baris ke Nomor Baris)"`.