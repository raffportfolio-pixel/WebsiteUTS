# SIPAK Cloud — Sistem Peminjaman Alat Kampus Berbasis Cloud

**SIPAK Cloud** adalah aplikasi web manajemen inventaris dan sirkulasi peminjaman alat laboratorium kampus yang dibangun dengan **PHP 8.2+**, **Laravel 12**, **Laravel Blade**, **Eloquent ORM**, dan **SQLite** yang dirancang khusus untuk dijalankan baik secara lokal maupun di-deploy dari **GitHub** ke layanan **Platform as a Service (PaaS)** pada **Public Cloud** menggunakan **Persistent Storage Volume**.

---

## Ringkasan Jawaban Akademik Tugas Kuliah

1. **Teknologi Database, Bahasa, dan Framework**:
   Database yang digunakan adalah **SQLite**, dengan bahasa pemrograman **PHP (8.2+)** dan framework **Laravel 12**.
2. **Model Layanan dan Deployment Cloud**:
   Model layanan cloud yang direncanakan adalah **PaaS (Platform as a Service)** dengan model deployment **Public Cloud** (seperti Railway atau Fly.io) pada **satu instance aplikasi (single-instance)** di tahap awal yang terhubung ke **Persistent Volume**.
3. **Arsitektur Tiga Lapisan Logis (Three-Tier Logical Architecture)**:
   Arsitektur aplikasi terdiri dari tiga lapisan logis yang terpisah secara rapi namun dapat berjalan secara efisien di dalam **satu server aplikasi** dengan database **SQLite lokal**:
   - **Presentation Layer (`resources/views/`, `public/css/sipak.css`, `public/js/sipak.js`)**: Antarmuka pengguna berbasis **Laravel Blade**, CSS modern bergaya bersih, dan JavaScript fungsional.
   - **Application Layer (`app/Http/Controllers/`, `app/Services/LoanWorkflowService.php`, `app/Policies/LoanPolicy.php`, `app/Http/Middleware/EnsureUserRole.php`)**: Menangani alur bisnis, otorisasi peran, validasi server-side, perhitungan keterlambatan, pencegahan peminjaman ganda, serta transaksi atomik.
   - **Data Layer (`app/Models/`, `database/migrations/`, `database/seeders/`, SQLite)**: Mengelola pemetaan objek-relasional melalui **Eloquent ORM** dan penyimpanan fisik pada file database **SQLite** di penyimpanan persisten.
4. **Alasan Pemilihan Teknologi**:
   - **SQLite** dipilih karena ringan, tanpa overhead proses server database terpisah (serverless file-based RDBMS), mendukung penuh transaksi ACID, *foreign key constraints*, mode *Write-Ahead Logging* (`WAL`), serta sangat efisien untuk skala aplikasi layanan kampus.
   - **Laravel** memudahkan pengembangan terstruktur melalui pola MVC, keamanan bawaan (CSRF, password hashing, autentikasi, policy), serta pengelolaan skema database melalui *migration* dan *seeder*.
   - **PaaS** memudahkan proses build dan deployment otomatis tanpa harus mengelola sistem operasi atau konfigurasi mesin virtual dari nol, sedangkan **pembagian tiga lapisan logis** memudahkan pemeliharaan kode dan pengujian.
5. **Peran GitHub dan Alur Eksekusi di Server Cloud**:
   **GitHub** berfungsi menyimpan kode sumber, riwayat versi, skema migrasi, dan pemicu (*webhook*) alur CI/CD deployment ke PaaS. Kode aplikasi PHP/Laravel dan file operasional **SQLite** berjalan di server cloud, di mana file database SQLite diletakkan di luar direktori *ephemeral container* yaitu pada **Persistent Storage Volume** (misalnya `/data/database.sqlite`).

---

## Fitur Utama & Aturan Bisnis yang Diimplementasikan

### Peran Mahasiswa
- **Registrasi Publik & Login**: Registrasi publik dikunci hanya menghasilkan akun dengan peran `mahasiswa`.
- **Dashboard Mahasiswa**: Ringkasan pengajuan menunggu, peminjaman aktif, peringatan keterlambatan, dan pintasan unit siap pinjam.
- **Katalog & Detail Alat**: Kartu bergambar untuk setiap unit alat dengan kode inventaris unik, pencarian, filter kategori/kondisi/status, serta keranjang pemilihan multi-unit.
- **Formulir Pengajuan Peminjaman Multi-Unit**: Mendukung pemilihan beberapa unit alat sekaligus, validasi urutan tanggal, pencegahan duplikasi unit dalam satu pengajuan, **Ringkasan Unit Terpilih** secara langsung sebelum dikirim, serta perlindungan *double-submit* dengan `idempotency_key`.
- **Daftar, Detail, & Riwayat Peminjaman**: Mahasiswa hanya dapat mengakses transaksi miliknya sendiri (`LoanPolicy`) dan dapat membatalkan pengajuan selama masih berstatus `menunggu`.

### Peran Admin / Petugas Laboratorium
- **Dashboard Admin (4 Indikator Konsisten)**:
  1. **Jumlah Unit Alat (`total_equipment`)**: Seluruh unit fisik alat yang terdaftar di tabel `equipment`.
  2. **Pengajuan Menunggu (`pending_loans`)**: Transaksi berstatus `menunggu` yang belum mereservasi alat dan membutuhkan keputusan admin.
  3. **Peminjaman Aktif (`active_loans`)**: Transaksi yang sedang mengikat unit alat, yaitu gabungan status `disetujui` (telah direservasi, siap diambil) dan `dipinjam` (sedang dibawa peminjam).
  4. **Peminjaman Terlambat (`overdue_loans`)**: Transaksi berstatus `dipinjam` dengan tanggal `due_date < hari ini` yang masih memiliki minimal satu unit alat belum dikembalikan.
- **Pengelolaan Kategori & Unit Alat**: CRUD kategori dan unit alat dengan kode inventaris unik (misal: 3 kamera Sony Alpha a6400 identik memiliki kode `CAM-SNY-001`, `CAM-SNY-002`, `CAM-SNY-003`), unggah foto melalui Laravel Storage (`public` disk), penyelesaian perawatan cepat, serta perlindungan riwayat transaksi (unit yang sudah memiliki riwayat transaksi tidak dihapus permanen melainkan dinonaktifkan).
- **Persetujuan, Penolakan, & Penyerahan Alat**:
  - Pengajuan `menunggu` belum mereservasi alat (`active_lock = null`).
  - Persetujuan admin (`disetujui`) memeriksa ulang ketersediaan unit di dalam transaksi database (`DB::transaction()` dengan mode `IMMEDIATE` di SQLite) dan mengunci unit (`active_lock = 1`, `operational_status = 'direservasi'`) serta dilindungi oleh indeks unik tingkat database `UNIQUE(equipment_id, active_lock)`.
  - Penolakan (`ditolak`) wajib menyertakan alasan penolakan.
  - Penyerahan (`dipinjam`) hanya berlaku pada transaksi `disetujui`, memeriksa ulang ketersediaan, serta mencatat kondisi fisik awal setiap unit saat diserahkan.
- **Pengembalian Sebagian & Seluruh Unit**:
  - Tabel `returns` berelasi 1-ke-1 dengan `loan_items` (`loan_item_id` unik), sehingga pengembalian bertahap (sebagian unit) maupun seluruh unit tercatat dengan akurat.
  - Transaksi baru berubah menjadi `dikembalikan` setelah seluruh unit pada transaksi tersebut memiliki catatan pengembalian.
  - Unit yang dikembalikan dalam kondisi `rusak_ringan` atau `rusak_berat` secara otomatis masuk ke status operasional `perawatan`.
- **Laporan Peminjaman & Ekspor CSV**: Filter berdasarkan rentang tanggal dan status, metrik ringkasan periode, tampilan cetak (`window.print()`), dan unduh CSV kompatibel Excel.

---

## Kebutuhan Lingkungan Kerja

- **PHP**: `>= 8.2` (terverifikasi pada PHP `8.2.12`)
- **Ekstensi PHP Wajib**: `pdo`, `pdo_sqlite`, `sqlite3`, `mbstring`, `openssl`, `fileinfo`, `ctype`, `json`
- **Composer**: `>= 2.x` (terverifikasi pada Composer `2.8.5`)
- **Database**: SQLite 3

---

## Langkah Instalasi & Menjalankan Aplikasi Secara Lokal

1. **Masuk ke direktori project dan instal dependency PHP**:
   ```bash
   composer install
   ```
2. **Salin konfigurasi environment dan buat `APP_KEY`**:
   ```bash
   cp .env.example .env
   php artisan key:generate
   ```
3. **Siapkan file database SQLite, jalankan migrasi beserta seeder data contoh, dan buat symlink storage**:
   ```bash
   # Di Windows PowerShell:
   if (!(Test-Path "database/database.sqlite")) { New-Item -ItemType File -Path "database/database.sqlite" -Force }
   php artisan migrate:fresh --seed
   php artisan storage:link
   ```
   *(Di Linux/macOS: `touch database/database.sqlite && php artisan migrate:fresh --seed && php artisan storage:link`)*
4. **Jalankan server pengembangan lokal**:
   ```bash
   php artisan serve
   ```
5. Buka browser ke alamat: **`http://127.0.0.1:8000`**

---

## Akun Demo Lokal & Cara Menyiapkan Akun Admin Produksi

### Kredensial Akun Demo (Hasil Seeder)

| Peran | Nama Pengguna | Email Login | Password | NIM / NIP |
| :--- | :--- | :--- | :--- | :--- |
| **Admin Lab** | Budi Santoso, S.Kom. | `admin@kampus.ac.id` | `password123` | `198705142014041001` |
| **Admin Lab 2** | Rina Kartika, A.Md. | `petugas@kampus.ac.id` | `password123` | `199208212019032004` |
| **Mahasiswa 1** | Rizky Pratama | `mahasiswa@kampus.ac.id` | `password123` | `230511014` |
| **Mahasiswa 2** | Nadia Putri Maharani | `nadia@kampus.ac.id` | `password123` | `230512042` |
| **Mahasiswa 3** | Dimas Arya Wijaya | `dimas@kampus.ac.id` | `password123` | `220513088` |

### Mekanisme Terkontrol Pembuatan Akun Admin di Produksi
Registrasi publik (`/register`) hanya membuat akun dengan peran `mahasiswa`. Untuk membuat akun Admin baru secara aman di server produksi tanpa membuka celah eskalasi hak akses publik, jalankan perintah Artisan khusus melalui terminal/SSH server:

```bash
php artisan sipak:create-admin \
  --name="Nama Petugas Lab" \
  --email="petugas.baru@kampus.ac.id" \
  --nip="199002152018031002" \
  --department="UPT Laboratorium Terpadu" \
  --phone="081122334455" \
  --password="PasswordKuatMinimal8Karakter"
```

---

## Hasil Pemeriksaan Fungsi & Pengujian Otomatis

Seluruh aturan bisnis, otorisasi peran, pembatasan akses transaksi antar-mahasiswa, pencegahan persetujuan ganda pada unit yang sama, alur pengembalian sebagian/penuh, perubahan status perawatan otomatis, dan pelaporan telah diuji menggunakan PHPUnit (`tests/Feature/SipakLoanSystemTest.php`).

Menjalankan pengujian otomatis:
```bash
php artisan test
```

**Hasil eksekusi terverifikasi**:
- `Tests\Unit\ExampleTest` — **PASS**
- `Tests\Feature\ExampleTest` — **PASS**
- `Tests\Feature\SipakLoanSystemTest`:
  1. `public registration creates student role and enforces role restrictions` — **PASS**
  2. `admin can manage categories and equipment with history protection` — **PASS**
  3. `student can request multiple units and idempotency prevents duplicate submission` — **PASS**
  4. `student cannot view or cancel another students loan` — **PASS**
  5. `prevents approving two active loans for the same equipment unit` — **PASS**
  6. `full lifecycle handover partial return full return and damaged unit enters maintenance` — **PASS**
  7. `overdue calculation and report generation` — **PASS**
  8. `rejection requires reason and invalid state transitions are blocked` — **PASS**
  9. `all application pages render and admin dashboard indicators are consistent` — **PASS**
- **Total**: **11 passed (106 assertions)**

---

## Panduan Deployment GitHub ke Cloud PaaS & Persistent Storage SQLite

### 1. Persiapan Repository GitHub
File `.gitignore` telah dikonfigurasi untuk mengecualikan:
- File rahasia environment (`.env`, `.env.*` kecuali `.env.example`)
- Dependency hasil instalasi (`/vendor`, `/node_modules`)
- File database operasional SQLite beserta file jurnal/WAL (`*.sqlite`, `*.sqlite-shm`, `*.sqlite-wal`, `*.sqlite-journal`)

### 2. Arsitektur Penyimpanan Persisten di PaaS (Railway / Fly.io)
Karena container PaaS bersifat *ephemeral* (sistem file container dibuat ulang setiap kali *redeploy*), file database SQLite dan foto unggahan **wajib** diletakkan pada **Persistent Storage Volume** yang di-*mount* ke path absolut (contoh: `/data`):
- Path Database SQLite: `DB_DATABASE=/data/database.sqlite`
- Direktori `/data` wajib memiliki izin tulis (*writable*) oleh proses PHP karena SQLite dalam mode `WAL` membuat file pendamping `/data/database.sqlite-wal` dan `/data/database.sqlite-shm` di direktori yang sama.
- Gunakan **1 instance aplikasi (single-instance)** tanpa *overlapping zero-downtime multi-writer* agar hanya satu instance yang menulis ke file SQLite pada satu waktu.
- Skrip startup produksi [`scripts/cloud-start.sh`](scripts/cloud-start.sh) telah disiapkan untuk:
  1. Memastikan direktori `/data`, `/data/backups`, dan `/data/public_uploads/equipment` tersedia dan *writable*.
  2. Membuat file `/data/database.sqlite` hanya jika belum ada (sehingga data tetap bertahan saat redeploy/restart).
  3. Menautkan direktori unggahan gambar ke penyimpanan persisten dan menjalankan `php artisan storage:link --force`.
  4. Menjalankan migrasi secara aman dengan `php artisan migrate --force` (tanpa menghapus data yang sudah ada).
  5. Menjalankan server web dengan **Document Root** diarahkan ke folder `public/`.

### 3. Konfigurasi Environment Variable Produksi di PaaS
Set variabel berikut pada dashboard penyedia PaaS:
```env
APP_NAME="SIPAK Cloud - Sistem Peminjaman Alat Kampus"
APP_ENV=production
APP_KEY=base64:ISI_DENGAN_APP_KEY_TETAP_ANDA
APP_DEBUG=false
APP_URL=https://nama-domain-anda.up.railway.app
APP_TIMEZONE=Asia/Jakarta

DB_CONNECTION=sqlite
DB_DATABASE=/data/database.sqlite
DB_FOREIGN_KEYS=true
DB_BUSY_TIMEOUT=5000
DB_JOURNAL_MODE=WAL
DB_SYNCHRONOUS=NORMAL
DB_TRANSACTION_MODE=IMMEDIATE

FILESYSTEM_DISK=public
SESSION_DRIVER=database
CACHE_STORE=database
```
> **Penting**: Pertahankan nilai `APP_KEY` yang sama antar-deployment agar sesi pengguna dan data terenkripsi tetap valid.

### 4. Kebutuhan & Estimasi Biaya Layanan PaaS (Berdasarkan Dokumentasi Resmi Terverifikasi)
- **Railway (docs.railway.com)**:
  - Mendukung deployment langsung dari repository GitHub menggunakan `nixpacks.toml` / `Procfile` yang telah disediakan di repository ini.
  - Mendukung fitur **Railway Volumes** yang di-*mount* ke path seperti `/data` dengan performa hingga 3.000 Read/Write IOPS.
  - **Informasi Biaya Resmi**: Paket **Hobby Plan** berbiaya **$5 / bulan** (sudah termasuk kredit penggunaan $5/bulan dan mendukung volume hingga 5 GB per layanan), sedangkan tarif penyimpanan Persistent Volume adalah **$0.15 per GB per bulan** (dihitung per menit sesuai pemakaian aktual). Tersedia juga *Trial/Free tier* dengan batas volume 0.5 GB.
- **Fly.io (fly.io/docs/laravel)**:
  - Mendukung **Fly Volumes** (NVMe lokal persisten) yang dikonfigurasi pada blok `[mounts]` (`destination = "/data"`) dan `DB_DATABASE = "/data/database.sqlite"`.
  - **Informasi Biaya Resmi**: Penyimpanan Fly Volume dikenakan tarif **$0.15 per GB per bulan** di luar biaya mesin compute VM.

### 5. Panduan Backup & Pemulihan (Restore) SQLite yang Konsisten
Jangan menyalin file `.sqlite` secara mentah dengan `cp` saat transaksi sedang berjalan di mode WAL tanpa menyertakan state WAL. Gunakan perintah `.backup` SQLite atau skrip [`scripts/backup-sqlite.sh`](scripts/backup-sqlite.sh) yang menyalin database secara atomik dan konsisten:

- **Membuat Backup Konsisten**:
  ```bash
  bash scripts/backup-sqlite.sh
  # atau secara langsung dengan sqlite3 CLI:
  sqlite3 /data/database.sqlite ".backup '/data/backups/sipak_backup_$(date +%Y%m%d_%H%M%S).sqlite'"
  ```
- **Langkah Pemulihan (Restore) dari File Backup**:
  1. Aktifkan mode pemeliharaan agar tidak ada transaksi tulis baru:
     ```bash
     php artisan down
     ```
  2. Hapus file `-wal` dan `-shm` lama lalu pulihkan file backup:
     ```bash
     rm -f /data/database.sqlite-wal /data/database.sqlite-shm
     sqlite3 /data/database.sqlite ".restore '/data/backups/nama_file_backup.sqlite'"
     ```
  3. Nonaktifkan kembali mode pemeliharaan:
     ```bash
     php artisan up
     ```
