# PROGRESS_CHANGELOG.md - Catatan Progres Pengembangan

Dokumen ini mencatat setiap progres, penambahan fitur, perbaikan bug, dan perubahan struktur file dalam proyek **Sistem Absensi Multi-Cabang, Approval & Payroll**.

---

## [2026-10-22] - Penyusunan Dokumentasi Portofolio README & Aset Visual
### Ditambahkan / Dibuat:
1. **Desain Ulang `README.md` Utama:**
   - Mentransformasi `README.md` standar Laravel menjadi etalase portofolio profesional berkelas industri.
   - Penambahan badge teknologi modern (Laravel 11, FilamentPHP v3, Flutter, MariaDB, Docker WAHA, Security FLAG_SECURE).
   - Penjelasan komprehensif 8 pilar fitur unggulan enterprise (Anti-Fraud GPS, 4-Split Shift & No-Break, Kalkulasi Denda & Potongan Berjenjang, Weekend 2x Replacement, Penyesuaian Jadwal Mandiri, WhatsApp Gateway Dua Arah, Pengamanan Slip Gaji Window 1-7, dan Auto-Purge Storage).
   - Diagram struktur arsitektur direktori proyek dan panduan instalasi lengkap untuk backend & mobile APK.
2. **Aset Visual & Mockup Antarmuka (`docs/assets/`):**
   - `docs/assets/dashboard_admin.jpg`: Mockup Dashboard Web Admin Filament v3 Dark Mode dengan analitik kehadiran dan rekapitulasi denda.
   - `docs/assets/mobile_dashboard.jpg`: Mockup antarmuka Mobile APK Flutter dengan indikator geofencing, tombol 4 sesi presensi, dan ringkasan bulan berjalan.
   - `docs/assets/mobile_camera_watermark.jpg`: Mockup kamera selfie presensi dengan panduan wajah dan banner watermark resmi server (koordinat GPS desimal 8 digit + timestamp WIB).

### File yang Dibuat / Diubah:
- `README.md`
- `docs/assets/dashboard_admin.jpg`
- `docs/assets/mobile_dashboard.jpg`
- `docs/assets/mobile_camera_watermark.jpg`
- `docs/PROGRESS_CHANGELOG.md`

---

## [2026-10-22] - Penambahan Nomor BPJS & NPWP pada Master Karyawan
### Ditambahkan / Dibuat:
1. **Atribut Opsional BPJS & NPWP Karyawan (`employees`):**
   - Penambahan kolom `bpjs_number` (Varchar 50, Nullable) dan `npwp_number` (Varchar 50, Nullable) via migrasi database `2026_10_22_115136_add_bpjs_and_npwp_to_employees_table.php`.
   - Update model `Employee.php` ($fillable).
   - Penambahan field input opsional pada Filament `EmployeeResource` di Tab **1. Identitas & Profil**.
   - Penambahan kolom toggleable `No. BPJS` dan `No. NPWP` pada tabel daftar karyawan di Filament.
   - Penambahan field `bpjs_number` dan `npwp_number` pada response payload login/profil API (`AuthController.php`).

### File yang Dibuat / Diubah:
- `database/migrations/2026_10_22_115136_add_bpjs_and_npwp_to_employees_table.php`
- `app/Models/Employee.php`
- `app/Filament/Resources/EmployeeResource.php`
- `app/Http/Controllers/Api/AuthController.php`
- `docs/DATABASE_SCHEMA.md`
- `docs/API_SPEC.md`
- `docs/PROGRESS_CHANGELOG.md`

---

## [2026-10-22] - Penerapan Single Settings untuk Kebijakan Denda & Kuota Izin
### Diperbarui / Disesuaikan:
1. **Transformasi Menu Kebijakan Menjadi Single Settings (1 Data Saja):**
   - Menghapus redundansi baris data ganda pada **Denda Keterlambatan** (`LateFineRule`), menetapkan tarif aktif tunggal: Rp 2.000 (Jam Pertama) dan Rp 200/jam (Jam Berikutnya).
   - Mengubah 3 menu kebijakan menjadi mode **Single Settings Form**:
     - `LateFineRuleResource` (Denda Keterlambatan)
     - `ForgotClockFineRuleResource` (Denda Lupa Absen)
     - `LeaveRuleResource` (Aturan Kuota Izin)
   - Menonaktifkan pembuatan baris baru (`canCreate => false`) dan penghapusan (`canDelete => false`).
   - Navigasi sidebar kini langsung mengarahkan (*auto-redirect*) ke form edit pengaturan data tunggal tersebut tanpa harus melalui tabel perantara.
   - Pindahnya `LeaveRuleResource` ke grup navigasi `'Pengaturan & Kebijakan'` agar seragam dengan kebijakan lainnya.
2. **Penyelarasan Query Service:**
   - Memastikan `AttendanceService::calculateLateFine` dan `evaluateForgotFines` mengambil `LateFineRule::first()` dan `ForgotClockFineRule::first()` dengan validasi status aktif.

### File yang Diubah:
- `app/Filament/Resources/LateFineRuleResource.php` & Pages
- `app/Filament/Resources/ForgotClockFineRuleResource.php` & Pages
- `app/Filament/Resources/LeaveRuleResource.php` & Pages
- `app/Services/AttendanceService.php`
- `docs/PROGRESS_CHANGELOG.md`

---

## [2026-10-22] - Master Persentase Potongan Berjenjang, Pengamanan Slip Gaji & Modul Penyesuaian Jadwal
### Ditambahkan / Dibuat:
1. **Master Persentase Potongan Dinamis (`penalty_percentage_rules`):**
   - Mendukung penambahan level berjenjang tanpa batas oleh Admin di Filament (`PenaltyPercentageRuleResource`).
   - 4 Kategori: `LEAVE` (Izin Biasa / Sakit), `OPNAME` (Rawat Inap), `LATE` (Keterlambatan), dan `ALPHA` (Mangkir).
   - Setiap level mengatur: `min_occurrences`, persentase potong insentif (`incentive_cut_percentage`), persentase potong gaji pokok (`salary_cut_percentage`), dan pembatalan uang kerajinan (`forfeit_diligence_allowance`).
   - Mengambil persentase level tertinggi yang dicapai per kategori pada saat perhitungan payroll bulanan (`PayrollService.php`).
   - Denda keterlambatan nominal rupiah dan potongan persentase gaji/insentif berjalan secara simultan.

2. **Pengamanan Digital & Window Time Slip Gaji (Tanggal 1 - 7):**
   - Menu dan endpoint Slip Gaji (`/api/payroll/my-slips`) hanya aktif pada tanggal 1 s/d 7 setiap bulannya. Di luar rentang ini, tombol disembunyikan di dashboard dan API mengembalikan HTTP 403 Forbidden.
   - Penambahan toggle `can_view_salary_slip` pada Master Jabatan (`positions`) untuk mengontrol izin lihat slip gaji.
   - Pengamanan anti-screenshot dan perekaman layar (*black screen*) menggunakan `FLAG_SECURE` (`flutter_windowmanager`) pada layar slip gaji APK Android.

3. **Modul Pengajuan & Penyesuaian Jadwal Mandiri (`Schedule Change Requests`):**
   - 3 Jenis pengajuan mandiri karyawan via APK:
     - `EXCHANGE_OFF_DAY`: Tukar hari libur reguler ke tanggal kerja lain.
     - `CHANGE_SHIFT`: Permohonan penyesuaian jam kerja / shift khusus.
     - `OUTSIDE_DUTY`: Pengajuan izin absen di luar radius toko (tugas luar).
   - Menu review dan approval di Web Admin Filament (`ScheduleChangeRequestResource`) dan di APK Android (`HeadstoreApprovalScreen` Tab 3).
   - Persetujuan otomatis memperbarui data jadwal kerja di `employee_schedules`.

4. **Notifikasi WhatsApp Dua Arah Otomatis (WAHA Gateway):**
   - Notifikasi dikirimkan ke Headstore/SPV/HRD saat karyawan mengajukan penyesuaian jadwal, izin, atau Tidak Break.
   - Notifikasi konfirmasi dikirimkan kembali ke WhatsApp karyawan begitu pengajuan disetujui atau ditolak beserta catatannya.

5. **Penyempurnaan Widget Statistik Dashboard Admin:**
   - `AttendanceStatsOverviewWidget`: Menampilkan statistik harian (hadir, telat, izin, alpha, tidak break) dengan link drill-down interaktif, total denda rupiah berjalan, dan akumulasi estimasi potongan persentase bulan berjalan.

### File yang Dibuat / Diubah:
- `database/migrations/2026_10_22_101425_create_penalty_percentage_rules_and_schedule_changes_tables.php`
- `app/Models/PenaltyPercentageRule.php`
- `app/Models/ScheduleChangeRequest.php`
- `app/Models/Position.php`
- `app/Models/EmployeeSchedule.php`
- `app/Models/Payroll.php`
- `app/Models/Employee.php`
- `app/Services/PayrollService.php`
- `app/Services/ScheduleChangeService.php`
- `app/Services/AttendanceService.php`
- `app/Services/LeaveService.php`
- `app/Services/WahaService.php`
- `app/Filament/Resources/PenaltyPercentageRuleResource.php`
- `app/Filament/Resources/ScheduleChangeRequestResource.php`
- `app/Filament/Resources/PositionResource.php`
- `app/Filament/Resources/LeaveRuleResource.php`
- `app/Filament/Widgets/AttendanceStatsOverviewWidget.php`
- `app/Http/Controllers/Api/DashboardApiController.php`
- `app/Http/Controllers/Api/PayrollApiController.php`
- `app/Http/Controllers/Api/LeaveApiController.php`
- `app/Http/Controllers/Api/ScheduleChangeApiController.php`
- `routes/api.php`
- `mobile_apk/pubspec.yaml`
- `mobile_apk/lib/config/api_constants.dart`
- `mobile_apk/lib/screens/dashboard_screen.dart`
- `mobile_apk/lib/screens/payroll_slip_screen.dart`
- `mobile_apk/lib/screens/schedule_change_screen.dart`
- `mobile_apk/lib/screens/headstore_approval_screen.dart`
- `docs/DATABASE_SCHEMA.md`
- `docs/API_SPEC.md`
- `docs/PROGRESS_CHANGELOG.md`

---

## [2026-10-21] - Inisialisasi Proyek, Environment & Database Master Schema
### Ditambahkan / Dibuat:
- **Inisialisasi Project Laravel 11:**
  - Setup core Laravel 11 dengan PHP 8.3 & Composer 2.10 di `/www/wwwroot/absensi.test`.
  - Konfigurasi environment `.env` untuk koneksi MySQL/MariaDB server aaPanel.
- **Instalasi FilamentPHP v3:**
  - Instalasi package `filament/filament:"^3.2"` dan panel Admin bawaan Filament v3.
- **Database Migrations - Master Modules:**
  - `branches`: Master cabang/toko (koordinat lat/lng, radius geofencing meter).
  - `positions`: Master jabatan & struktur hirarki organisasi (level, parent_id).
  - `shifts`: Master shift kerja 4 sesi (Check-In, Break-Out, Break-In, Out-Store, serta window time batas buka/tutup).
  - `users` & `employees`: Master akun dan data karyawan lengkap dengan komponen gaji (basic_salary, daily_meal_allowance, diligence_allowance, incentive_allowance, position_allowance), selective device locking, multi-branch, dan outside duty flags.
  - `employee_branches`: Pivot table relasi karyawan ke cabang tambahan (multi-branch).
  - `late_fine_rules`: Master aturan denda telat jam ke-1 dan jam berikutnya.
  - `forgot_clock_fine_rules`: Master aturan denda lupa absen 4 sesi.
  - `leave_rules`: Master kuota dan regulasi pengajuan izin (maks 3x/bulan, min days opname, weekend multiplier).
  - `holidays`: Master hari libur nasional & toko (pemicu wajib ganti 2x hari di weekday).
  - `approval_workflows`: Master alur custom multi-level approval berjenjang (Headstore, HRD, Owner).

- **Eksekusi Migrasi Database & Seeder Awal:**
  - Berhasil menghubungkan aplikasi ke MariaDB aaPanel (`sql_absensi_test`).
  - Menjalankan 12 migration tabel Master data.
  - Menjalankan `MasterDataSeeder` untuk inisialisasi akun Super Administrator (`admin@absensi.test`), 4 level jabatan hirarki, shift kerja 4 sesi default (Shift Pagi & Siang), cabang pusat contoh, master aturan denda telat/lupa, dan kuota izin.

- **Perbaikan Web Server & Aset Livewire:**
  - Mempublikasikan aset fisik JavaScript Livewire v3 ke `public/livewire/livewire.js` agar tidak diblokir/404 oleh rule static file Nginx aaPanel.
  - Menyesuaikan `APP_URL=http://192.9.200.237` di `.env` untuk mendukung form submit AJAX Livewire/Filament tanpa bentrok domain.

### File yang Dibuat / Diubah:
- `docs/PROGRESS_CHANGELOG.md`
- `composer.json` & `composer.lock`
- `.env` & `.env.example`
- `app/Providers/Filament/AdminPanelProvider.php`
- `app/Models/*` (10 model data master)
- `database/migrations/*_create_master_tables.php` (9 file migrasi master)
- `database/seeders/MasterDataSeeder.php` & `database/seeders/DatabaseSeeder.php`
- `config/livewire.php` & `public/livewire/*`

---

## [2026-10-21] - Pembuatan Menu CRUD Master Data (Filament Resources)
### Ditambahkan / Dibuat:
- **Filament Resources CRUD Master Data:**
  - `BranchResource`: CRUD data cabang/toko, titik koordinat latitude & longitude GPS, serta batas radius presensi (geofencing) meter.
  - `PositionResource`: CRUD jabatan organisasi beserta tingkat level hirarki (Level 1: Owner s/d Level 4: Staff) dan penentuan atasan langsung.
  - `ShiftResource`: CRUD pengaturan jam kerja 4 sesi presensi harian (Check-In, Break-Out, Break-In, Out-Store) serta pembatasan window time buka/tutup tombol absen APK.
  - `HolidayResource`: CRUD tanggal merah nasional & libur operasional toko dengan toggle aturan wajib ganti 2x hari di weekday.
  - `EmployeeResource`: CRUD data master karyawan berbasis tab lengkap:
    - Tab Identitas: Profil, NIK, No WhatsApp (WAHA Gateway), dokumen PKWT, upload foto selfie master & foto KTP.
    - Tab Penempatan: Cabang utama, jabatan, shift default, jatah libur mingguan (OFF), dan cabang tambahan (multi-cabang).
    - Tab Keamanan: Toggle izin multi-cabang, tugas luar/dinas, selective device lock (IMEI/Device ID), dan status penundaan surat dokter.
    - Tab Payroll: Komponen gaji pokok, uang makan harian (hadir fisik), tunjangan jabatan, serta uang kerajinan & insentif (dengan peringatan hangus jika Alpha >= 1x).
  - `LateFineRuleResource`: CRUD kebijakan tarif denda keterlambatan (jam ke-1 dan jam berikutnya).
  - `ForgotClockFineRuleResource`: CRUD nominal denda lupa absensi pada masing-masing 4 sesi kerja harian.
  - `LeaveRuleResource`: CRUD kuota maksimal izin per bulan, minimal hari sakit wajib opname, dan pengali ganti hari libur 2x.
  - `ApprovalWorkflowResource`: CRUD alur persetujuan dinamis berjenjang (Requester, Urutan Langkah, Approver, dan opsi filter cabang yang sama).
- **Storage Symlink:**
  - Menjalankan `php artisan storage:link` untuk menghubungkan `storage/app/public` ke `public/storage` untuk kebutuhan foto KTP dan foto selfie master karyawan.

### File yang Dibuat / Diubah:
- `docs/PROGRESS_CHANGELOG.md`
- `app/Filament/Resources/BranchResource.php` & Pages
- `app/Filament/Resources/PositionResource.php` & Pages
- `app/Filament/Resources/ShiftResource.php` & Pages
- `app/Filament/Resources/HolidayResource.php` & Pages
- `app/Filament/Resources/EmployeeResource.php` & Pages
- `app/Filament/Resources/LateFineRuleResource.php` & Pages
- `app/Filament/Resources/ForgotClockFineRuleResource.php` & Pages
- `app/Filament/Resources/LeaveRuleResource.php` & Pages
- `app/Filament/Resources/ApprovalWorkflowResource.php` & Pages

---

## [2026-10-21] - Custom Master Tingkat Level (Grade) Dinamis
### Ditambahkan / Dibuat:
- **Tabel & Model `job_levels`:**
  - Skema tabel `job_levels` (`name`, `grade_code`, `rank_order`, `description`).
  - Menghubungkan tabel `positions` dengan `job_level_id` (foreign key ke `job_levels`).
- **Filament `JobLevelResource` (Master Organisasi -> Tingkat Level / Grade):**
  - Menu CRUD mandiri untuk mengatur tingkatan level/grade tanpa batas (misal: Level 1 s/d Level 10+).
  - Tampilan badge ranking hirarki `# 1` (Owner), `# 2` (Manager), `# 3` (Supervisor), `# 4` (Staff), dst.
- **Pembaruan `PositionResource`:**
  - Mengubah dropdown statis level menjadi relasi dinamis ke `JobLevel` dengan fitur modal pembuatan level langsung (*inline create option*).
  - Tampilan tabel jabatan kini menampilkan badge nama Tingkat Level secara dinamis dari database.

### File yang Dibuat / Diubah:
- `docs/DATABASE_SCHEMA.md`
- `docs/PROGRESS_CHANGELOG.md`
- `database/migrations/2026_10_21_000010_create_job_levels_table.php`
- `app/Models/JobLevel.php`
- `app/Models/Position.php`
- `app/Filament/Resources/JobLevelResource.php` & Pages
- `app/Filament/Resources/PositionResource.php`

---

## [2026-10-21] - Integrasi Visual Interactive Map Picker pada Master Cabang
### Ditambahkan / Dibuat:
- **Visual Map Picker (Leaflet + OpenStreetMap):**
  - Komponen peta interaktif di `BranchResource` tanpa perlu API Key berbayar.
  - Pin marker yang bisa digeser (*draggable*) atau diklik pada peta untuk otomatis mengisi koordinat Latitude & Longitude dengan presisi 8 angka di belakang koma (`.toFixed(8)`).
  - Visualisasi radius geofencing presensi (lingkaran biru transparan) yang dinamis mengikuti inputan `radius_meters`.
  - Fitur pencarian lokasi / alamat instan (OpenStreetMap Nominatim).
  - Fitur tombol **Gunakan GPS Saya Saat Ini** untuk mendeteksi posisi perangkat secara otomatis.
- **Presisi Koordinat Geofencing:**
  - Memastikan penyimpanan database MySQL (`decimal(10, 8)` dan `decimal(11, 8)`) terjaga dengan presisi hingga 8 digit desimal (~1 milimeter), sangat akurat untuk kebutuhan validasi presensi karyawan di toko.

### File yang Dibuat / Diubah:
- `docs/PROGRESS_CHANGELOG.md`
- `resources/views/filament/forms/components/map-picker.blade.php`
- `app/Filament/Resources/BranchResource.php`

---

## [2026-10-21] - Otomatisasi Akun Karyawan (NIK & Password) & Integrasi WAHA Gateway
### Ditambahkan / Dibuat:
- **Penyederhanaan Form Karyawan (All-in-One Creation):**
  - Menghapus kewajiban memilih akun User secara manual.
  - Menambahkan kolom NIK KTP sebagai User ID login APK dan input password awal (default: `123456`).
  - `CreateEmployee` otomatis membuat record `users` di backend dan menghubungkannya langsung ke data karyawan (`user_id`).
  - `EditEmployee` mendukung penggantian/reset password dan sinkronisasi nama/email secara otomatis.
- **Integrasi WhatsApp Gateway (`WahaService`):**
  - Service `app/Services/WahaService.php` untuk pengiriman notifikasi WhatsApp otomatis melalui WAHA Docker lokal (`http://localhost:3000`).
  - Pemformatan nomor WhatsApp Indonesia otomatis (`628xxx@c.us`).
  - Mengirimkan pesan selamat bergabung beserta User ID (NIK) dan password awal saat karyawan baru didaftarkan.
  - Penanganan error yang aman (*graceful error handling*): Jika WAHA belum terhubung atau offline, proses penyimpanan karyawan tetap berjalan sukses dan pesan dicatat di log.
  - Tombol aksi **Kirim WA** di tabel karyawan dan halaman ubah karyawan untuk mengirim ulang kredensial kapan saja.

### File yang Dibuat / Diubah:
- `docs/PROGRESS_CHANGELOG.md`
- `config/services.php`
- `app/Services/WahaService.php`
- `app/Filament/Resources/EmployeeResource.php`
- `app/Filament/Resources/EmployeeResource/Pages/CreateEmployee.php`
- `app/Filament/Resources/EmployeeResource/Pages/EditEmployee.php`

---

## [2026-10-21] - User ID Login Berbasis Kode Karyawan & Toggle Notifikasi WhatsApp (WAHA)
### Ditambahkan / Diubah:
- **User ID Login Menggunakan Kode Karyawan Toko (`employee_code`):**
  - Mengubah acuan login akun APK dari NIK KTP menjadi **Kode Karyawan** (contoh: `EMP-001`).
  - Menambahkan kolom `username` (unique, nullable) pada tabel `users` via migration `2026_10_21_154854_add_username_to_users_table.php`.
  - NIK KTP tetap tersimpan rapi sebagai identitas kependudukan legal di form & database.
  - Saat pendaftaran karyawan baru, sistem otomatis membuatkan akun `users` dengan `username = employee_code` dan `email = strtolower(employee_code) . '@absensi.test'`.
- **Toggle Kirim Kredensial via WhatsApp:**
  - Menambahkan toggle **Kirim Kredensial Akun via WhatsApp** pada form input karyawan baru (default: AKTIF).
  - Admin memiliki kendali penuh: Jika toggle dimatikan, akun tetap tersimpan tanpa memicu pengiriman pesan WhatsApp.
  - Jika toggle diaktifkan, kredensial (Kode Karyawan & Password) langsung dikirimkan ke nomor WhatsApp karyawan melalui WAHA Gateway.
- **Konfigurasi WAHA Session `notif_1`:**
  - Menyesuaikan sesi WAHA ke nama sesi aktif **`notif_1`** di `.env`.
  - Template pesan WhatsApp disesuaikan menampilkan `User ID (Kode Karyawan): EMP-001`.
  - Aksi tabel dan halaman edit diperbarui agar menampilkan dan mengirimkan Kode Karyawan ke nomor HP karyawan.

### File yang Dibuat / Diubah:
- `docs/DATABASE_SCHEMA.md`
- `docs/PROGRESS_CHANGELOG.md`
- `database/migrations/2026_10_21_154854_add_username_to_users_table.php`
- `app/Models/User.php`
- `app/Filament/Resources/EmployeeResource.php`
- `app/Filament/Resources/EmployeeResource/Pages/CreateEmployee.php`
- `app/Filament/Resources/EmployeeResource/Pages/EditEmployee.php`
- `app/Services/WahaService.php`
- `.env`

---

## [2026-10-21] - Migrasi & Model Eloquent Transaksi Harian (Absensi, Izin, Pengganti Weekend, Jadwal & Payroll)
### Ditambahkan / Dibuat:
- **6 File Migrasi Tabel Transaksi Harian:**
  - `leave_requests`: Pengajuan izin sakit/keperluan pribadi/pulang cepat, dukungan upload surat dokter/opname, flag pending surat dokter di Hari H, dan deteksi kewajiban ganti hari weekend.
  - `leave_replacements`: Tracking pemilihan mandiri 2 tanggal pengganti di weekday (hari OFF) untuk izin yang jatuh pada hari weekend atau libur.
  - `leave_approvals`: Log tahapan persetujuan izin berjenjang (Headstore, HRD, Owner) dengan catatan dan riwayat waktu aksi.
  - `employee_schedules`: Jadwal kerja harian karyawan (`REGULAR`, `WEEKLY_OFF`, `LEAVE`, `REPLACEMENT_WORK`) yang terhubung langsung ke request izin terkait.
  - `attendances`: Log absensi presensi 4 sesi harian (Check-In, Break-Out, Break-In, Out-Store) lengkap dengan koordinat GPS desimal 8 digit, path foto, request & approval No-Break Headstore, kalkulasi menit/jam keterlambatan, denda telat & denda lupa, status harian, serta penanda kelayakan uang makan harian.
  - `payrolls`: Rekapitulasi slip gaji bulanan karyawan, penghitungan kehadiran fisik (uang makan), denda keterlambatan/lupa, serta penerapan otomatis **Penalti Alpha 1x** (hangus uang kerajinan & insentif).
- **6 Model Eloquent Transaksi:**
  - `App\Models\LeaveRequest`
  - `App\Models\LeaveReplacement`
  - `App\Models\LeaveApproval`
  - `App\Models\EmployeeSchedule`
  - `App\Models\Attendance`
  - `App\Models\Payroll`
- **Pembaruan Relasi Model Eksisting:**
  - `Employee`: Relasi `schedules()`, `attendances()`, `leaveRequests()`, `leaveReplacements()`, `payrolls()`.
  - `Branch`: Relasi `attendances()`.
  - `Shift`: Relasi `schedules()` dan `attendances()`.
  - `Position`: Relasi `leaveApprovals()`.
- **Eksekusi Sukses Migrasi Database:**
  - Menjalankan `php artisan migrate` di server MariaDB aaPanel, seluruh 6 tabel transaksi harian berhasil terbuat dan terindeks dengan rapi.

### File yang Dibuat / Diubah:
- `docs/PROGRESS_CHANGELOG.md`
- `database/migrations/2026_10_21_160001_create_leave_requests_table.php`
- `database/migrations/2026_10_21_160002_create_leave_replacements_table.php`
- `database/migrations/2026_10_21_160003_create_leave_approvals_table.php`
- `database/migrations/2026_10_21_160004_create_employee_schedules_table.php`
- `database/migrations/2026_10_21_160005_create_attendances_table.php`
- `database/migrations/2026_10_21_160006_create_payrolls_table.php`
- `app/Models/LeaveRequest.php`
- `app/Models/LeaveReplacement.php`
- `app/Models/LeaveApproval.php`
- `app/Models/EmployeeSchedule.php`
- `app/Models/Attendance.php`
- `app/Models/Payroll.php`
- `app/Models/Employee.php`
- `app/Models/Branch.php`
- `app/Models/Shift.php`
- `app/Models/Position.php`

---

## [2026-10-21] - Layanan Logika Bisnis & Perhitungan Otomatis (Geofence, Watermark, Attendance, Leave & Payroll)
### Ditambahkan / Dibuat:
- **Instalasi Intervention Image v4:**
  - Menambahkan package `intervention/image:^4.4` dengan driver PHP GD.
- **Service Layanan Logika Bisnis Terpadu:**
  - `GeofenceService`: Menghitung jarak GPS radius toko (Haversine formula), validasi hak akses cabang (cabang reguler vs multi-cabang vs tugas luar), selective device binding (IMEI/Device ID anti-titip absen), serta proteksi blokir Check-In susulan surat dokter.
  - `ImageWatermarkService`: Kompresi foto selfie presensi (target < 150KB) dan burn-in watermark 4 baris data (waktu server WIB, koordinat GPS lat/lng, cabang/lokasi tugas, identitas kode & nama karyawan) pada banner semi-transparan.
  - `AttendanceService`: Menangani 4 sesi harian (Check-In, Break-Out, Break-In, Out-Store), perhitungan denda keterlambatan dinamis jam ke-1 dan jam berikutnya, pengajuan & persetujuan No-Break oleh Headstore, serta evaluasi denda lupa absen.
  - `LeaveService`: Validasi kuota izin bulanan (maks 3x/bulan), validasi syarat surat dokter & opname, penegakan aturan izin weekend/hari libur wajib memilih 2 hari pengganti di weekday, alur approval berjenjang dinamis, serta integrasi notifikasi pesan WhatsApp ke approver dan pemohon via WAHA Gateway.
  - `PayrollService`: Perhitungan slip gaji bulanan, penghitungan uang makan harian berbasis kehadiran fisik, pemotongan denda telat & lupa absen, serta penegakan otomatis **Penalti Alpha 1x** (hangus uang kerajinan & insentif jika ada 1x saja mangkir/alpha dalam sebulan).
- **Automasi Scheduler / Console Commands:**
  - `attendance:purge-old-photos`: Command artisan untuk membersihkan file fisik foto absensi yang berusia lebih dari 60 hari demi menghemat storage server aaPanel.
  - `attendance:evaluate-daily`: Command artisan harian untuk mendeteksi karyawan mangkir (ALPHA) dan mengevaluasi denda lupa absen pada akhir hari.
  - Terjadwal otomatis di `routes/console.php` (evaluasi harian pukul 23:55 malam & auto-purge pukul 02:00 pagi).

### File yang Dibuat / Diubah:
- `docs/PROGRESS_CHANGELOG.md`
- `composer.json` & `composer.lock`
- `app/Services/GeofenceService.php`
- `app/Services/ImageWatermarkService.php`
- `app/Services/AttendanceService.php`
- `app/Services/LeaveService.php`
- `app/Services/PayrollService.php`
- `app/Console/Commands/PurgeOldAttendancePhotos.php`
- `app/Console/Commands/EvaluateDailyAttendance.php`
- `routes/console.php`

---

## [2026-10-21] - Filament Resources Transaksi & Dashboard Monitoring Admin
### Ditambahkan / Dibuat:
- **Dashboard Monitoring Widget (`AttendanceStatsOverviewWidget`):**
  - Kartu statistik real-time: **Total Karyawan**, **Total Masuk** (Hadir Tepat & Telat), **Total Telat**, **Total Izin** (Izin Penuh & Pulang Cepat), **Total OFF / Alpha**, dan **Total Denda Hari Ini**.
  - Dilengkapi fitur interaktif *drill-down URL*: mengklik kartu statistik langsung mengarahkan Admin ke tabel log presensi terfilter.
- **Resource Presensi Harian (`AttendanceResource`):**
  - Menampilkan tabel presensi harian dengan avatar circular foto Check-In & Out-Store, badge status, denda telat, denda lupa absen, dan hak uang makan.
  - Infolist detail komprehensif: menampilkan foto selfie 4 sesi ukuran besar ber-watermark, koordinat GPS dengan tombol tautan langsung ke Google Maps, kalkulasi denda, dan status No-Break.
  - Aksi tabel untuk Approve / Reject permintaan Tidak Break (No-Break) oleh Headstore secara instan.
- **Resource Pengajuan & Approval Izin (`LeaveRequestResource`):**
  - Monitoring seluruh pengajuan izin, sakit, dan izin pulang cepat di Hari H.
  - Infolist menampilkan dokumen foto surat dokter, foto bukti rawat inap / opname, daftar 2 hari pengganti di weekday, dan riwayat langkah tahapan persetujuan berjenjang.
  - Aksi tabel untuk **Approve** (dengan input catatan opsional) dan **Reject** (dengan input alasan wajib) yang memicu pergerakan workflow dan notifikasi WhatsApp ke pihak terkait.
- **Resource Manajemen Jadwal Kerja (`EmployeeScheduleResource`):**
  - Pengelolaan jadwal harian karyawan (`REGULAR`, `WEEKLY_OFF`, `LEAVE`, `REPLACEMENT_WORK`).
  - Header Action **Generate Jadwal Bulanan Massal**: Fitur satu-klik untuk meng-generate jadwal kerja sebulan penuh untuk semua karyawan berdasarkan shift default dan jatah libur mingguan (OFF) tanpa menimpa izin yang sudah disetujui.
- **Resource Penggajian & Slip Gaji (`PayrollResource`):**
  - Rekapitulasi slip gaji bulanan karyawan dengan kalkulasi uang makan berbasis hadir fisik, denda telat & lupa, serta visualisasi peringatan **Penalti Alpha 1x** (hangus uang insentif & kerajinan).
  - Header Action **Hitung Gaji Bulanan (Generate)**: Mengkalkulasi slip gaji seluruh karyawan untuk bulan dan tahun yang dipilih secara otomatis.
  - Row Action **Publish & Kirim WA**: Menerbitkan slip gaji ke aplikasi APK karyawan dan mengirimkan rincian Take Home Pay via WhatsApp (WAHA Gateway).

### File yang Dibuat / Diubah:
- `docs/PROGRESS_CHANGELOG.md`
- `app/Providers/Filament/AdminPanelProvider.php`
- `app/Filament/Widgets/AttendanceStatsOverviewWidget.php`
- `app/Filament/Resources/AttendanceResource.php` & Pages
- `app/Filament/Resources/LeaveRequestResource.php` & Pages
- `app/Filament/Resources/EmployeeScheduleResource.php` & Pages
- `app/Filament/Resources/PayrollResource.php` & Pages

---

## [2026-10-21] - Penyusunan REST API Mobile APK (Flutter) & Dokumentasi API_SPEC.md
### Ditambahkan / Dibuat:
- **Instalasi Laravel Sanctum:**
  - Menjalankan `php artisan install:api` untuk setup routing API dan tabel `personal_access_tokens`.
  - Menambahkan trait `HasApiTokens` pada model `User`.
- **22 Endpoint REST API Mobile APK:**
  - **Auth (`/api/auth`):** Login via User ID (Kode Karyawan) & Password, validasi Selective Device Binding, profil `me`, dan logout (pencabutan token).
  - **Unified Dashboard (`/api/dashboard`):** Menampilkan profil, jadwal hari ini, status presensi 4 sesi, ringkasan denda bulan berjalan, status penalti Alpha, dan sisa kuota izin.
  - **Presensi 4 Sesi (`/api/attendance`):** Endpoint status hari ini, Check-In, Break-Out, Break-In, Out-Store, pengajuan No-Break, dan riwayat presensi. Dilengkapi deteksi Anti-Mock GPS (`is_mock_location`), geofencing radius meter, selective device lock, serta kompresi foto selfie & watermark GPS.
  - **Pengajuan Izin (`/api/leave`):** Endpoint cek sisa kuota izin, daftar hari libur (OFF) mingguan untuk dipilih mandiri sebagai hari pengganti weekend, submit izin (sakit, pribadi, pulang cepat hari H), susulan upload surat dokter untuk membuka blokir check-in, dan daftar riwayat pengajuan izin.
  - **Approval Khusus Headstore / SPV (`/api/headstore`):** Endpoint daftar & aksi persetujuan Tidak Break (No-Break) serta approval pengajuan izin bawahan.
  - **Slip Gaji (`/api/payroll`):** Endpoint daftar slip gaji bulanan yang telah dipublish dan rincian lengkap komponen pendapatan, potongan denda telat/lupa, serta Take Home Pay.
- **Dokumentasi Lengkap API Specification (`docs/API_SPEC.md`):**
  - Menyusun panduan integrasi mobile lengkap mencakup Headers, parameter Request Body, format Response JSON sukses & gagal, hingga kode error.

### File yang Dibuat / Diubah:
- `docs/API_SPEC.md`
- `docs/PROGRESS_CHANGELOG.md`
- `composer.json` & `composer.lock`
- `bootstrap/app.php`
- `routes/api.php`
- `app/Models/User.php`
- `app/Http/Controllers/Api/AuthController.php`
- `app/Http/Controllers/Api/AttendanceApiController.php`
- `app/Http/Controllers/Api/LeaveApiController.php`
- `app/Http/Controllers/Api/HeadstoreApprovalApiController.php`
- `app/Http/Controllers/Api/PayrollApiController.php`
- `app/Http/Controllers/Api/DashboardApiController.php`
- `database/migrations/2026_10_21_201459_create_personal_access_tokens_table.php`

---

## [2026-10-21] - Pengujian Menyeluruh REST API & Inisialisasi Project Flutter Mobile APK (/mobile_apk)
### Ditambahkan / Dibuat:
- **Pengujian Endpoint REST API via cURL:**
  - Login `POST /api/auth/login` berhasil menghasilkan Bearer Token Sanctum dan data Karyawan.
  - Profil `GET /api/auth/me` dan Dashboard Terpadu `GET /api/dashboard` berhasil mengembalikan data lengkap.
  - Pengecekan Anti-Fake GPS: Berhasil mendeteksi `is_mock_location == true` dan menolak request dengan status 403 `FAKE_GPS_DETECTED`.
  - Endpoint presensi hari ini, kuota izin, daftar hari OFF untuk hari pengganti, serta slip gaji bulanan berhasil diuji dengan status 200 OK.
- **Inisialisasi Project Flutter Mobile APK di `/mobile_apk`:**
  - `pubspec.yaml`: Mengonfigurasi dependensi `http`, `shared_preferences`, `geolocator`, `image_picker`, `device_info_plus`, dan `intl`.
  - `AndroidManifest.xml`: Izin hardware kamera fisik, lokasi presisi GPS tinggi (`ACCESS_FINE_LOCATION`), dan internet.
  - `lib/config/api_constants.dart`: Mapping seluruh URL endpoint API backend.
  - `lib/services/*`: Layanan `ApiService` (token injection), `LocationService` (deteksi anti-mock GPS), dan `AuthService`.
  - `lib/screens/login_screen.dart`: Form login User ID (Kode Karyawan) dan Device ID binding otomatis.
  - `lib/screens/dashboard_screen.dart`: Antarmuka mobile terpadu menampilkan shift, 4 tombol presensi harian, quick menu, dan ringkasan bulanan (peringatan penalti Alpha & denda).
  - `lib/screens/attendance_camera_screen.dart`: Kamera selfie langsung (tanpa galeri) dengan indikator geofencing dan proteksi anti-fake GPS.
  - `lib/screens/leave_request_screen.dart`: Pengajuan izin sakit/pribadi/pulang cepat dengan pemilihan mandiri 2 hari pengganti di weekday jika izin bertepatan dengan weekend.
  - `lib/screens/payroll_slip_screen.dart`: Tampilan riwayat dan rincian slip gaji bulanan (Take Home Pay).
  - `lib/screens/headstore_approval_screen.dart`: Modul approval permohonan Tidak Break dan pengajuan izin staf bagi Headstore / SPV.
  - `lib/main.dart`: Entry point aplikasi dengan persistensi sesi login dan tema dark modern.
  - `mobile_apk/README.md`: Panduan build dan compile APK Flutter.

### File yang Dibuat / Diubah:
- `docs/PROGRESS_CHANGELOG.md`
- `mobile_apk/pubspec.yaml`
- `mobile_apk/README.md`
- `mobile_apk/android/app/src/main/AndroidManifest.xml`
- `mobile_apk/lib/main.dart`
- `mobile_apk/lib/config/api_constants.dart`
- `mobile_apk/lib/services/api_service.dart`
- `mobile_apk/lib/services/location_service.dart`
- `mobile_apk/lib/screens/login_screen.dart`
- `mobile_apk/lib/screens/dashboard_screen.dart`
- `mobile_apk/lib/screens/attendance_camera_screen.dart`
- `mobile_apk/lib/screens/leave_request_screen.dart`
- `mobile_apk/lib/screens/payroll_slip_screen.dart`
- `mobile_apk/lib/screens/headstore_approval_screen.dart`---

## [2026-10-21] - Konfigurasi Cloudflare Tunnel (Akses Luar Jaringan via HTTPS)
### Ditambahkan / Dibuat:
- **Instalasi Binary Cloudflare Tunnel (`cloudflared`):**
  - Download dan konfigurasi `cloudflared` x86_64 pada `/home/f/.local/bin/cloudflared`.
- **Script Manajemen Tunnel (`scripts/tunnel.sh`):**
  - Perintah mudah untuk mengontrol tunnel di background:
    - `./scripts/tunnel.sh start`: Mengaktifkan Quick Tunnel publik (URL `trycloudflare.com`).
    - `./scripts/tunnel.sh status`: Memeriksa status tunnel dan menampilkan URL aktif.
    - `./scripts/tunnel.sh stop`: Menghentikan proses tunnel.
    - `./scripts/tunnel.sh start-token <TOKEN>`: Menjalankan Named Tunnel permanen dengan token Cloudflare Zero Trust.
- **Verifikasi Konektivitas API Publik:**
  - Pengujian cURL dari jaringan internet luar menuju endpoint `https://specifically-picked-effort-devel.trycloudflare.com/api/auth/login` berhasil 100% dengan respons JSON sukses dan enkripsi SSL/TLS HTTPS.
- **Pembaruan Konfigurasi Mobile APK:**
  - Mengarahkan `ApiConstants.baseUrl` di `mobile_apk/lib/config/api_constants.dart` ke URL Cloudflare Tunnel publik aktif.

### File yang Dibuat / Diubah:
- `docs/PROGRESS_CHANGELOG.md`
- `scripts/tunnel.sh`
- `mobile_apk/lib/config/api_constants.dart`
