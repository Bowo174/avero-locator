# AVERO Locator – Implementation Plan (CDN-Only, Single `web.php`, No Prefix, No Blade Components)

## 1. Scope Recap & Approach

AVERO Locator adalah katalog internal untuk membantu staff menemukan barang secara cepat berdasarkan nama, foto, lokasi fisik, dan nomor baris. Fitur inti meliputi pencarian publik dengan server-side DataTables dan modal detail produk, serta panel admin untuk ringkasan, CRUD lokasi, dan CRUD produk. Aplikasi tidak mengelola stok, harga, transaksi, atau mutasi inventaris.

Implementasi menggunakan Laravel 12 (kompatibel dengan arahan Laravel 11/12), PHP 8.3, MySQL, Blade konvensional, Bootstrap 5, jQuery 3.7.1, dan DataTables. Bootstrap, jQuery, dan aset DataTables dimuat langsung dari CDN; tidak ada Vite, Node.js, NPM, atau Tailwind. Paket backend Yajra Laravel DataTables dipasang melalui Composer untuk membuat respons server-side.

Seluruh definisi rute berada di `routes/web.php`. Admin dibedakan dari halaman publik menggunakan middleware `auth` dan nama rute `admin.*`; tidak ada route group dengan URL prefix. View menggunakan `@extends`, `@section`, `@yield`, dan `@include`, tanpa Blade Components. Interaksi admin menggunakan jQuery AJAX dan JavaScript biasa, dengan fallback validasi/error yang jelas.

## 2. High-Level Architecture

```text
Staff browser                         Admin browser
     |                                      |
     | GET /, AJAX DataTables               | Login session, CRUD AJAX
     +------------------+-------------------+
                        |
                 Laravel 12 / PHP 8.3
       +----------------+--------------------+
       |                 |                    |
 PublicSearch       Auth middleware     Admin controllers
 controller          (session)          Location/Product
       |                                      |
       +------------------+-------------------+
                          |
                    Eloquent ORM
                          |
                  MySQL: users, locations,
                       products
                          |
        JSON rows/counts/detail + validation JSON
                          |
        DataTables renders result rows in browser
                          |
          click product -> Bootstrap 5 modal
           image + location name + row number
```

Pada halaman pencarian, DataTables mengirim parameter paging, sorting, dan kata kunci ke endpoint AJAX Laravel. Controller menerapkan pencarian ke query Eloquent, menghitung total dan hasil terfilter, lalu mengembalikan format JSON yang dimengerti DataTables. Data produk terkait lokasi diambil melalui relasi eager loading agar tidak menimbulkan query berulang.

Ketika staff memilih nama produk, browser mengambil detail yang sudah disertakan pada data baris atau meminta endpoint detail publik, lalu mengisi dan menampilkan Bootstrap Modal tanpa meninggalkan tabel. Endpoint detail hanya mengembalikan field katalog yang dibutuhkan. Operasi admin melewati middleware `auth`; controller memvalidasi input, menulis ke MySQL, menyimpan/menghapus file gambar pada disk `public`, dan mengembalikan JSON untuk memperbarui tabel tanpa reload.

## 3. Environment & Tooling

| Area | Pilihan / Konfigurasi |
| --- | --- |
| Sistem lokal | Laragon, Apache atau Nginx, PHP 8.3, MySQL 8.x |
| Framework | Laravel 12; Laravel 11 dapat digunakan dengan pola yang sama |
| Paket server-side tabel | `yajra/laravel-datatables-oracle` versi yang kompatibel dengan versi Laravel yang dipilih, dipasang melalui Composer |
| Browser | Chrome/Edge versi terkini; layout responsif untuk layar kasir dan ponsel |
| URL lokal | `http://avero-locator.test` (virtual host Laragon) atau `http://localhost:8000` |
| Bootstrap CSS | `https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css` |
| jQuery | `https://code.jquery.com/jquery-3.7.1.min.js` |
| DataTables Bootstrap 5 CSS | `https://cdn.datatables.net/2.1.8/css/dataTables.bootstrap5.min.css` |
| DataTables core JS | `https://cdn.datatables.net/2.1.8/js/dataTables.min.js` |
| DataTables Bootstrap 5 JS | `https://cdn.datatables.net/2.1.8/js/dataTables.bootstrap5.min.js` |
| Bootstrap bundle JS | `https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js` |

Contoh urutan stylesheet dan script pada layout Blade (urutan JavaScript penting):

```html
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
<link href="https://cdn.datatables.net/2.1.8/css/dataTables.bootstrap5.min.css" rel="stylesheet">

<script src="https://code.jquery.com/jquery-3.7.1.min.js"></script>
<script src="https://cdn.datatables.net/2.1.8/js/dataTables.min.js"></script>
<script src="https://cdn.datatables.net/2.1.8/js/dataTables.bootstrap5.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
```

Tambahkan `integrity` dan `crossorigin` berdasarkan nilai SRI resmi CDN yang dipilih saat implementasi/deployment, atau gunakan URL CDN versi yang dipatok dan dikelola oleh proyek. Jangan mencampur versi DataTables core dan adapter Bootstrap. Yajra PHP package bukan file JavaScript CDN; ia menangani protokol server-side di Laravel.

## 4. Database Design

### `users` (Admin)

| Kolom | Tipe yang disarankan | Aturan / Keterangan |
| --- | --- | --- |
| `id` | BIGINT UNSIGNED | Primary key |
| `name` | VARCHAR(255) | Nama admin |
| `email` | VARCHAR(255) | Unique, dipakai untuk login |
| `password` | VARCHAR(255) | Hash password Laravel, tidak pernah disimpan sebagai teks biasa |
| `email_verified_at` | TIMESTAMP nullable | Opsional sesuai kebijakan autentikasi |
| `remember_token` | VARCHAR(100) nullable | Kolom standar autentikasi session Laravel |
| `created_at`, `updated_at` | TIMESTAMP nullable | Timestamp Laravel |

Gunakan migration `users` bawaan Laravel bila tersedia. Karena seluruh akun pada lingkup awal adalah admin, tidak diperlukan kolom role tambahan; bila kelak ada beberapa level akses, role/permission harus dirancang sebagai kebutuhan terpisah.

### `locations` (Master letak)

| Kolom | Tipe yang disarankan | Aturan / Keterangan |
| --- | --- | --- |
| `id` | BIGINT UNSIGNED | Primary key |
| `name` | VARCHAR(100) | Wajib; contoh `Etalase E`, `Rak Laptop` |
| `max_rows` | UNSIGNED SMALLINT | Wajib, minimum 1; jumlah baris maksimum pada lokasi |
| `created_at`, `updated_at` | TIMESTAMP nullable | Timestamp Laravel |

Nama lokasi sebaiknya unique setelah normalisasi spasi/case pada validasi aplikasi. Batas kapasitas disimpan sebagai bilangan positif.

### `products` (Katalog barang)

| Kolom | Tipe yang disarankan | Aturan / Keterangan |
| --- | --- | --- |
| `id` | BIGINT UNSIGNED | Primary key |
| `name` | VARCHAR(255) | Wajib; nama yang dicari staff |
| `location_id` | BIGINT UNSIGNED | Foreign key ke `locations.id`, wajib dan indexed |
| `row_number` | UNSIGNED SMALLINT | Wajib; harus berada pada rentang 1 sampai `locations.max_rows` |
| `image_path` | VARCHAR(255) nullable | Path relatif di disk `public`, misalnya `products/abc123.jpg` |
| `created_at`, `updated_at` | TIMESTAMP nullable | Timestamp Laravel |

### Relasi dan integritas

- Satu `Location` memiliki banyak `Product`; satu `Product` wajib berada pada tepat satu `Location` (`locations.id` → `products.location_id`).
- Gunakan foreign key dengan `ON DELETE RESTRICT`/perilaku `restrictOnDelete()`. Lokasi yang masih dipakai tidak dapat dihapus, sesuai PRD.
- `row_number` tidak dapat dijamin berada di bawah `max_rows` hanya dengan foreign key biasa. Validasi harus dilakukan pada create/update produk setelah lokasi dipilih; query dan penyimpanan dilakukan dalam transaksi bila ada kemungkinan perubahan kapasitas bersamaan.
- Ketika kapasitas lokasi diturunkan, tolak perubahan jika masih terdapat produk dengan `row_number > max_rows` baru. Admin harus memindahkan produk terlebih dahulu.
- Tidak ada tabel stok atau kolom jumlah persediaan karena aplikasi hanya memetakan letak fisik.

## 5. Routing & Controllers (Single `web.php`, No Prefix)

Berikut rancangan isi `routes/web.php`. Semua controller berada di `App\Http\Controllers`; nama method dapat disesuaikan dengan implementasi, tetapi seluruh URL tetap didefinisikan di file ini.

```php
<?php

use App\Http\Controllers\Admin\DashboardController;
use App\Http\Controllers\Admin\LocationController;
use App\Http\Controllers\Admin\ProductController;
use App\Http\Controllers\Auth\LoginController;
use App\Http\Controllers\PublicSearchController;
use Illuminate\Support\Facades\Route;

Route::get('/', [PublicSearchController::class, 'index'])->name('home');
Route::get('/products/search', [PublicSearchController::class, 'search'])
    ->name('products.search');
Route::get('/products/{product}', [PublicSearchController::class, 'show'])
    ->whereNumber('product')
    ->name('products.show');

Route::middleware('guest')->group(function () {
    Route::get('/login', [LoginController::class, 'create'])->name('login');
    Route::post('/login', [LoginController::class, 'store'])->name('login.store');
});

Route::post('/logout', [LoginController::class, 'destroy'])
    ->middleware('auth')
    ->name('logout');

Route::middleware('auth')->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])
        ->name('admin.dashboard');

    Route::get('/locations', [LocationController::class, 'index'])
        ->name('admin.locations.index');
    Route::get('/locations/data', [LocationController::class, 'data'])
        ->name('admin.locations.data');
    Route::post('/locations', [LocationController::class, 'store'])
        ->name('admin.locations.store');
    Route::get('/locations/{location}', [LocationController::class, 'show'])
        ->whereNumber('location')
        ->name('admin.locations.show');
    Route::put('/locations/{location}', [LocationController::class, 'update'])
        ->whereNumber('location')
        ->name('admin.locations.update');
    Route::delete('/locations/{location}', [LocationController::class, 'destroy'])
        ->whereNumber('location')
        ->name('admin.locations.destroy');

    Route::get('/admin-products', [ProductController::class, 'index'])
        ->name('admin.products.index');
    Route::get('/admin-products/data', [ProductController::class, 'data'])
        ->name('admin.products.data');
    Route::post('/admin-products', [ProductController::class, 'store'])
        ->name('admin.products.store');
    Route::get('/admin-products/{product}', [ProductController::class, 'show'])
        ->whereNumber('product')
        ->name('admin.products.show');
    Route::put('/admin-products/{product}', [ProductController::class, 'update'])
        ->whereNumber('product')
        ->name('admin.products.update');
    Route::delete('/admin-products/{product}', [ProductController::class, 'destroy'])
        ->whereNumber('product')
        ->name('admin.products.destroy');

    Route::get('/locations/{location}/rows', [LocationController::class, 'rows'])
        ->whereNumber('location')
        ->name('admin.locations.rows');
});
```

URL admin pada contoh sengaja menggunakan path datar tanpa `/admin` prefix, sementara pemisahan area dilakukan oleh middleware `auth` dan nama `admin.*`. Path `/locations/...` dipakai modul lokasi; modul produk menggunakan `/admin-products/...` agar tidak bertabrakan dengan resource lokasi. Ini adalah pilihan URL teknis; bila path produk perlu diganti, pastikan seluruhnya tetap tanpa prefix dan tidak menimbulkan konflik.

Controller `PublicSearchController` melayani halaman staff, JSON DataTables, dan detail publik. `LoginController` memvalidasi email/password, menjalankan `Auth::attempt`, meregenerasi session saat berhasil, dan melakukan logout dengan invalidasi session serta regenerasi token. Controller admin memvalidasi data dan mengembalikan JSON konsisten (`message`, `data`, `errors`) dengan status HTTP yang sesuai. Login pertama dapat dibuat melalui seeder/command terproteksi; jangan menyediakan registrasi publik.

## 6. Frontend Implementation (Plain Blade + Partials, No Components)

Struktur view yang disarankan:

```text
resources/views/
  layouts/app.blade.php
  layouts/admin.blade.php
  partials/flash.blade.php
  partials/admin-nav.blade.php
  public/search.blade.php
  auth/login.blade.php
  admin/dashboard.blade.php
  admin/locations/index.blade.php
  admin/products/index.blade.php
```

Layout memasang CDN, meta CSRF, slot konten dengan `@yield`, dan slot script. View halaman memakai `@extends('layouts.app')`, `@section('content')`, kemudian mengikutsertakan navigasi/flash message memakai `@include('partials...')`. Hindari `<x-...>` dan seluruh pemanggilan Blade Components. Form konvensional tetap menggunakan `@csrf`; untuk AJAX token dikirim melalui header `X-CSRF-TOKEN` yang diambil dari meta tag.

Contoh pola event modal di halaman staff:

```javascript
const table = $('#products-table').DataTable({
    processing: true,
    serverSide: true,
    ajax: { url: searchUrl, type: 'GET' },
    columns: [
        { data: 'name', name: 'products.name' }
    ],
    pageLength: 10,
    order: [[0, 'asc']]
});

$('#products-table tbody').on('click', 'tr', function () {
    const product = table.row(this).data();
    if (!product) return;

    $('#product-modal-title').text(product.name);
    $('#product-modal-image').attr('src', product.image_url || fallbackImageUrl);
    $('#product-modal-location').text(product.location_text);
    bootstrap.Modal.getOrCreateInstance('#product-modal').show();
});
```

`name`, `image_url`, dan `location_text` harus berasal dari JSON server yang telah dibentuk dengan aman. Gunakan `.text()` untuk konten teks, bukan menyisipkan nama produk ke HTML. Render URL gambar melalui helper URL Laravel. Modal memiliki teks alternatif gambar dan fallback jika gambar kosong/gagal dimuat. Bila DataTables menggambar ulang baris, event delegation pada `tbody` menjaga klik tetap berfungsi.

Untuk tabel admin, gunakan DataTables server-side bila volume data memerlukannya. Form create/edit dapat memakai modal atau area form pada halaman yang sama. Submit dilakukan dengan jQuery AJAX, tombol diberi status loading, error validasi Laravel ditampilkan dekat field, lalu tabel di-reload tanpa reset halaman bila operasi berhasil. Konfirmasi hapus harus menyebut nama record; setelah sukses, tutup modal dan perbarui tabel.

## 7. Admin Panel & User UX Details

### Admin CRUD lokasi

1. Admin membuka daftar lokasi yang menampilkan nama, kapasitas baris, jumlah produk terkait, serta aksi edit/hapus.
2. Form create menerima `name` dan `max_rows`; validasi server mensyaratkan nama terisi, panjang maksimum, dan kapasitas integer minimal 1.
3. Edit kapasitas memeriksa produk yang menempati baris di atas kapasitas baru. Jika ada, kembalikan error yang menjelaskan bahwa produk harus dipindahkan terlebih dahulu.
4. Hapus lokasi yang memiliki produk ditolak server dengan HTTP 409 atau respons validasi yang dapat ditampilkan inline. Jangan hanya mengandalkan tombol UI.

### Admin CRUD produk

- Field: nama produk, lokasi, nomor baris, gambar opsional saat edit dan dianjurkan saat create.
- Saat lokasi dipilih, JavaScript memanggil `admin.locations.rows` dan membangun pilihan nomor 1 sampai `max_rows`. Nonaktifkan dropdown baris selama request, reset nilainya ketika lokasi berubah, dan tampilkan state kosong/loading/error.
- Endpoint rows hanya mengembalikan ID, nama lokasi, dan kapasitas; validasi server tetap memastikan lokasi tersedia dan `row_number` berada pada rentang yang benar. Nilai dari browser tidak pernah dipercaya sebagai batas kapasitas.
- Jika produk dipindahkan ke lokasi lain, muat ulang pilihan baris dan validasi ulang. Saat edit, pilihan lokasi/baris yang sedang tersimpan harus terpilih setelah data dimuat.
- Tabel produk menampilkan nama, lokasi, baris, thumbnail kecil, dan aksi edit/hapus. Pencarian publik tetap hanya berfokus pada nama produk agar sesuai rancangan katalog.

### Upload gambar

Terima JPEG, PNG, atau WebP dengan batas ukuran awal 2 MB (konfigurabel). Validasi menggunakan MIME/content inspection Laravel, bukan ekstensi saja. Simpan dengan nama acak pada disk `public`, folder `products`; simpan hanya path relatif di `image_path`. Preview lokal menggunakan `URL.createObjectURL()` dan dicabut setelah tidak diperlukan. Saat mengganti gambar, simpan file baru terlebih dahulu, commit perubahan record, lalu hapus file lama; ketika menghapus produk, hapus file terkait setelah penghapusan record berhasil. Jika penyimpanan DB gagal, bersihkan file baru agar tidak menjadi orphan.

### Pengalaman pencarian staff

Halaman `/` langsung menampilkan input pencarian dengan placeholder yang jelas dan fokus visual. DataTables menggunakan server-side processing, debounce pencarian bawaan, indikator loading, pesan hasil kosong, dan pagination sederhana. Tampilkan hanya nama produk yang dapat diklik; pada ponsel target sentuh dibuat cukup besar. Modal menyajikan foto dengan ukuran responsif, teks format `Nama Produk (Nama Lokasi baris ke N)`, dan pesan fallback ketika foto tidak tersedia. Tutup modal tidak mengosongkan query/tabel sehingga staff dapat lanjut mencari tanpa kehilangan konteks.

## 8. Security & Storage

- Semua form memakai CSRF Laravel. AJAX mengirim token dari `<meta name="csrf-token">` melalui header `X-CSRF-TOKEN`; jangan menonaktifkan middleware CSRF untuk endpoint CRUD.
- Semua endpoint admin, termasuk endpoint AJAX/data dan endpoint pilihan baris, wajib berada di dalam middleware `auth`. Gunakan session bawaan Laravel, hash melalui `Hash::make`, `Auth::attempt`, session regeneration saat login, dan invalidasi session saat logout.
- Tidak ada registrasi publik. Buat akun admin melalui seeder lokal yang password-nya berasal dari environment, atau melalui command administrasi yang meminta input aman. Jangan commit kredensial default.
- Gunakan Form Request atau validasi controller untuk setiap input. Gunakan Eloquent/query builder dengan parameter binding; jangan membentuk SQL dari string pencarian pengguna.
- Validasi `location_id`, kapasitas, `row_number`, dan file pada server. Batasi tipe/ukuran gambar, gunakan nama file acak, dan jangan menerima path file dari request.
- Escape output Blade dengan `{{ }}`; pada JavaScript gunakan `.text()` dan bukan concatenation HTML untuk data pengguna. Untuk gambar gunakan URL hasil helper Laravel.
- Jalankan `php artisan storage:link`. File pada `storage/app/public/products` dapat diakses di `public/storage/products`; `image_path` berisi path relatif seperti `products/nama-acak.webp` dan URL dibuat dengan `Storage::disk('public')->url($path)` atau `asset('storage/'.$path)`.
- Pastikan permission direktori `storage` dan `bootstrap/cache` benar pada server. Lakukan backup folder gambar bersama backup database karena keduanya membentuk katalog produk.
- Halaman pencarian memang terbuka untuk jaringan toko menurut PRD; batasi akses jaringan di konfigurasi server/firewall jika aplikasi tidak boleh diakses dari internet. Detail publik jangan mengembalikan data admin atau metadata internal yang tidak diperlukan.

## 9. Content Management Flow

1. Pemilik/admin membuat akun admin melalui proses provisioning terkontrol, lalu masuk dari `/login`.
2. Admin membuka manajemen lokasi dan menambahkan lokasi fisik, misalnya `Etalase E` dengan `max_rows = 4`. Sistem memvalidasi nama dan kapasitas lalu menyimpan record.
3. Admin membuka manajemen produk, mengisi nama produk, memilih `Etalase E`, lalu dropdown baris memuat pilihan 1 sampai 4 dari server.
4. Admin memilih baris, memilih gambar, memeriksa preview, lalu menyimpan. Server memvalidasi file dan relasi lokasi/baris, menyimpan gambar ke disk publik, serta menyimpan path relatif di database.
5. Admin dapat mengedit produk untuk memperbaiki nama, gambar, lokasi, atau baris. Jika memindahkan produk, pilihan baris dimuat ulang berdasarkan lokasi tujuan.
6. Staff membuka `/`, mengetik nama atau sebagian nama produk. DataTables mengirim pencarian ke server dan menampilkan hasil yang cocok.
7. Staff mengklik nama produk. Modal menampilkan gambar serta lokasi dalam format `Nama Produk (Etalase E baris ke 1)` tanpa memuat ulang halaman.
8. Jika tata letak toko berubah, admin memperbarui lokasi/produk. Sistem mencegah penghapusan lokasi yang masih dipakai dan mencegah kapasitas lokasi diturunkan di bawah baris produk yang ada.

## 10. Deployment Checklist

### Persiapan proyek

Pastikan PHP 8.3, Composer, ekstensi PHP Laravel (termasuk `pdo_mysql`, `fileinfo`, `mbstring`, `openssl`, dan `gd` atau `imagick` bila diperlukan), serta MySQL aktif. Untuk proyek Laravel baru:

```bash
composer create-project laravel/laravel avero-locator "^12.0"
cd avero-locator
composer require yajra/laravel-datatables-oracle
```

Jika repository Laravel sudah tersedia, lewati `create-project` dan pasang paket Yajra yang kompatibel dengan versi framework. Pilih versi paket yang sesuai dengan tabel kompatibilitas resmi Yajra untuk versi Laravel yang digunakan.

### Environment dan schema

```bash
cp .env.example .env
php artisan key:generate
```

Atur `.env` untuk MySQL, misalnya `DB_CONNECTION=mysql`, `DB_HOST=127.0.0.1`, `DB_PORT=3306`, `DB_DATABASE=avero_locator`, `DB_USERNAME`, dan `DB_PASSWORD`. Buat database kosong di MySQL/Laragon, lalu jalankan:

```bash
php artisan make:model Location -m
php artisan make:model Product -m
php artisan make:controller PublicSearchController
php artisan make:controller Auth/LoginController
php artisan make:controller Admin/DashboardController
php artisan make:controller Admin/LocationController
php artisan make:controller Admin/ProductController
php artisan migrate
```

Buat migration/seeder secara eksplisit sebelum `migrate`: tabel locations, products, foreign key restrict, index pencarian/relasi, dan akun admin awal. Jangan menjalankan seeder berisi password tetap pada production.

### Storage dan menjalankan aplikasi

```bash
php artisan storage:link
php artisan optimize:clear
php artisan serve
```

Buka URL lokal yang ditentukan Laragon atau `http://127.0.0.1:8000`. Jalankan seeder data lokasi awal dari PRD (Etalase A-E, Rak A-B, Rak Laptop) hanya jika data tersebut memang sesuai layout toko saat ini. Buat admin awal menggunakan seeder yang membaca kredensial environment atau perintah provisioning aman.

### Sebelum digunakan staff

- Verifikasi seluruh halaman publik dan admin dapat diakses sesuai session/middleware.
- Pastikan setiap rute berada di `routes/web.php`, route admin tidak menggunakan URL prefix, serta route name memakai `admin.*`.
- Pastikan halaman memakai Blade layouts/partials biasa tanpa komponen, dan tidak ada build step Vite/Node/NPM/Tailwind.
- Pastikan CDN Bootstrap, jQuery, dan DataTables dapat diakses dari jaringan toko; bila internet lokal tidak tersedia, rencanakan CDN mirror internal yang tetap memenuhi kebijakan aset proyek.
- Pastikan `storage:link` bekerja dan gambar dapat dibuka dari URL publik, batas ukuran/type berjalan, serta penghapusan/penggantian gambar tidak meninggalkan file.
- Pastikan staff dapat mencari, membuka modal, melihat fallback gambar, dan mempertahankan hasil pencarian setelah modal ditutup.
- Pastikan validasi baris, perlindungan penghapusan lokasi terpakai, dan penolakan penurunan kapasitas bekerja di server.
- Pada deployment production, set `APP_ENV=production`, `APP_DEBUG=false`, kredensial database yang kuat, HTTPS bila tersedia, backup database dan gambar, lalu jalankan `php artisan config:cache` dan `php artisan route:cache` setelah konfigurasi final.

### Catatan konsistensi URL

Instruksi mewajibkan nama rute admin menggunakan `admin.*` dan melarang URL prefix. Karena itu, path pada contoh dibuat datar dan proteksinya berasal dari middleware. Bila aplikasi nantinya memerlukan path `/admin/...`, hal tersebut merupakan perubahan kontrak yang harus disetujui sebelum mengubah routing.
