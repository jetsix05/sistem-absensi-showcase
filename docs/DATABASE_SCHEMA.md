# DATABASE_SCHEMA.md - Struktur Tabel Sistem Absensi & Payroll

Dokumen ini adalah acuan tunggal (Single Source of Truth) untuk skema database MySQL/MariaDB. Setiap perubahan struktur tabel wajib dicatat di sini.

---

## A. KELOMPOK TABEL MASTER (9 MODUL UTAMA)

### 0. `users` (Akun Login Sistem & APK)
- `id` (BigInt, PK)
- `name` (Varchar) - Nama pengguna / nama karyawan
- `username` (Varchar, Unique, Nullable) - **User ID login aplikasi APK (menggunakan Kode Karyawan, contoh: `EMP-001`)**
- `email` (Varchar, Unique) - Email autentikasi sistem (`kode_karyawan@absensi.test` atau email pribadi)
- `password` (Varchar, Hashed) - Password login
- `timestamps`

### 1. `branches` (Master Cabang / Toko)
- `id` (BigInt, PK)
- `code` (Varchar, Unique) - Kode Cabang
- `name` (Varchar) - Nama Cabang / Toko
- `address` (Text) - Alamat lengkap
- `latitude` (Decimal 10,8) - Titik koordinat Latitude
- `longitude` (Decimal 11,8) - Titik koordinat Longitude
- `radius_meters` (Integer) - Batas radius absen dalam meter (contoh: 50)[cite: 1]
- `is_active` (Boolean, default: true)
- `timestamps`

### 2. `job_levels` (Master Tingkat Level / Grade)
- `id` (BigInt, PK)
- `name` (Varchar) - Nama Tingkat Level (Contoh: Level 1 - Owner, Level 2 - Managerial, Level 5 - Team Leader)
- `grade_code` (Varchar, Nullable, Unique) - Kode Grade (Contoh: LV-01, GR-01)
- `rank_order` (Integer, default: 1) - Urutan bobot hirarki (1 tertinggi / Owner, semakin besar angkanya semakin bawah levelnya)
- `description` (Text, Nullable) - Deskripsi kriteria/wewenang level
- `timestamps`

### 3. `positions` (Master Jabatan & Hirarki)[cite: 2]
- `id` (BigInt, PK)
- `name` (Varchar) - Nama Jabatan (Owner, HRD, Supervisor/Headstore, Staff, Teknisi, dll)[cite: 2]
- `job_level_id` (BigInt, Nullable, FK -> `job_levels.id`) - Tingkat level / grade organisasi
- `parent_id` (BigInt, Nullable, FK -> `positions.id`) - Atasan langsung secara struktur organisasi
- `level` (Integer) - Nilai level numerik bawaan (1: Owner, 2: HRD/Manager, 3: SPV/Headstore, 4: Staff)[cite: 2]
- `timestamps`

### 3. `shifts` (Master Shift Kerja - 4 Sesi)[cite: 1, 2]
- `id` (BigInt, PK)
- `name` (Varchar) - Contoh: "Shift Pagi", "Shift Siang"[cite: 1]
- `open_check_in_time` (Time) - Jam mulai buka tombol check-in (contoh: 06:00:00)[cite: 1]
- `check_in_time` (Time) - Jam masuk seharusnya (contoh Pagi: 09:00:00, Siang: 12:00:00)[cite: 1]
- `break_out_time` (Time) - Jam mulai istirahat (contoh Pagi: 13:00:00, Siang: 17:00:00)[cite: 1]
- `break_in_time` (Time) - Jam selesai istirahat (contoh Pagi: 17:00:00, Siang: 18:00:00)[cite: 1]
- `out_store_time` (Time) - Jam pulang (contoh: 21:00:00)[cite: 1]
- `close_out_store_time` (Time) - Batas akhir tombol check-out (contoh: 23:59:59)[cite: 1]
- `timestamps`

### 4. `employees` (Master Karyawan & Komponen Gaji)[cite: 2]
- `id` (BigInt, PK)
- `user_id` (BigInt, FK -> `users.id`) - Akun login
- `employee_code` (Varchar, Unique) - ID Karyawan[cite: 2]
- `nik` (Varchar, Unique) - NIK KTP[cite: 2]
- `full_name` (Varchar) - Nama Lengkap[cite: 2]
- `phone_wa` (Varchar) - Nomor WhatsApp aktif untuk notifikasi WAHA[cite: 2]
- `address` (Text) - Alamat[cite: 2]
- `bpjs_number` (Varchar, Nullable) - Nomor BPJS Ketenagakerjaan / Kesehatan (Opsional)
- `npwp_number` (Varchar, Nullable) - Nomor NPWP Karyawan (Opsional)
- `position_id` (BigInt, FK -> `positions.id`) - Jabatan[cite: 2]
- `primary_branch_id` (BigInt, FK -> `branches.id`) - Cabang utama
- `default_shift_id` (BigInt, FK -> `shifts.id`) - Shift default
- `weekly_off_day` (TinyInt) - Hari libur mingguan default (1=Senin ... 7=Minggu)[cite: 1]
- `photo_selfie_master` (Varchar) - Path foto wajah master[cite: 2]
- `photo_ktp` (Varchar) - Path foto KTP[cite: 2]
- `pkwt_number` (Varchar, Nullable) - Nomor/Status PKWT[cite: 2]
- `pkwt_end_date` (Date, Nullable) - Tanggal berakhir kontrak PKWT
- **Keamanan & Lokasi:**
  - `can_multi_branch` (Boolean, default: false) - Bisa absen di beberapa cabang terdaftar[cite: 1, 2]
  - `can_outside_duty` (Boolean, default: false) - Bisa absen tugas luar[cite: 1]
  - `is_device_locked` (Boolean, default: false) - Kunci IMEI/Device ID (Opsional untuk karyawan nakal)
  - `registered_device_id` (Varchar, Nullable) - Android Device ID terdaftar
  - `has_pending_medical_letter` (Boolean, default: false) - Penanda wajib upload surat dokter sebelum bisa Check-In berikutnya
- **Komponen Gaji (Payroll):**[cite: 1, 2]
  - `basic_salary` (Decimal 15,2, default: 0) - Gaji Pokok[cite: 2]
  - `daily_meal_allowance` (Decimal 15,2, default: 0) - Uang Makan Harian (dikali jumlah hadir)[cite: 2]
  - `diligence_allowance` (Decimal 15,2, default: 0) - Uang Kerajinan (Hangus jika Alpha >= 1x sebulan)[cite: 1, 2]
  - `incentive_allowance` (Decimal 15,2, default: 0) - Uang Insentif (Hangus jika Alpha >= 1x sebulan)[cite: 1]
  - `position_allowance` (Decimal 15,2, default: 0) - Tunjangan Jabatan[cite: 2]
- `timestamps`

### 5. `employee_branches` (Pivot Tabel Penempatan Multi-Cabang)[cite: 1, 2]
- `id` (BigInt, PK)
- `employee_id` (BigInt, FK -> `employees.id`)
- `branch_id` (BigInt, FK -> `branches.id`)

### 6. `late_fine_rules` (Master Denda Terlambat)[cite: 1, 2]
- `id` (BigInt, PK)
- `first_hour_fine` (Decimal 15,2) - Tarif denda 1 jam pertama (Contoh: 2000)[cite: 1]
- `next_hour_fine` (Decimal 15,2) - Tarif denda per jam berikutnya (Contoh: 200)[cite: 1]
- `is_active` (Boolean, default: true)
- `timestamps`

### 7. `forgot_clock_fine_rules` (Master Denda Lupa Absen)[cite: 2]
- `id` (BigInt, PK)
- `forgot_check_in_fine` (Decimal 15,2) - Denda lupa Check-In
- `forgot_break_out_fine` (Decimal 15,2) - Denda lupa Break-Out
- `forgot_break_in_fine` (Decimal 15,2) - Denda lupa Break-In
- `forgot_out_store_fine` (Decimal 15,2) - Denda lupa Out-Store
- `timestamps`

### 8. `leave_rules` (Master Pengajuan Izin) & `holidays` (Master Hari Libur)[cite: 2]
**Tabel `leave_rules`:**
- `id` (BigInt, PK)
- `max_leave_per_month` (Integer, default: 3) - Maksimal izin sebulan[cite: 1]
- `min_days_notice` (Integer, default: 0) - Batas pengajuan H+ / H-
- `min_sick_days_for_opname` (Integer, default: 3) - Batas hari sakit wajib lampiran opname[cite: 2]
- `weekend_replacement_multiplier` (Integer, default: 2) - Jumlah hari ganti jika izin di weekend/libur[cite: 1]

**Tabel `holidays`:**
- `id` (BigInt, PK)
- `holiday_date` (Date, Unique) - Tanggal merah / libur khusus
- `description` (Varchar) - Keterangan hari libur
- `is_double_replacement` (Boolean, default: true) - Memicu wajib ganti 2x di weekday

### 9. `approval_workflows` (Master Custom Approval)[cite: 2]
- `id` (BigInt, PK)
- `requester_position_id` (BigInt, FK -> `positions.id`) - Jabatan yang mengajukan
- `step_order` (Integer) - Urutan approval (1, 2, dst)
- `approver_position_id` (BigInt, FK -> `positions.id`) - Jabatan yang wajib menyetujui (Contoh: HRD, Headstore, atau Owner)[cite: 2]
- `is_same_branch_only` (Boolean, default: true) - True untuk Headstore cabang bersangkutan, False untuk HRD/Owner pusat
- `timestamps`

---

## B. KELOMPOK TABEL TRANSAKSI HARIAN (ABSENSI, IZIN & PAYROLL)

### 10. `employee_schedules` (Jadwal Kerja Harian & Tracking Hari Pengganti)
- `id` (BigInt, PK)
- `employee_id` (BigInt, FK -> `employees.id`)
- `schedule_date` (Date)
- `shift_id` (BigInt, Nullable, FK -> `shifts.id`)
- `schedule_type` (Enum: `REGULAR`, `WEEKLY_OFF`, `LEAVE`, `REPLACEMENT_WORK`)
- `replaced_leave_id` (BigInt, Nullable, FK -> `leave_requests.id`) - Diisi jika hari ini adalah jadwal masuk untuk mengganti izin weekend sebelumnya
- `notes` (Varchar, Nullable) - Contoh: "Ganti Izin Minggu, 25 Okt 2026"
- `timestamps`

### 11. `attendances` (Log Absensi 4 Sesi Harian)
- `id` (BigInt, PK)
- `employee_id` (BigInt, FK -> `employees.id`)
- `branch_id` (BigInt, FK -> `branches.id`)
- `shift_id` (BigInt, FK -> `shifts.id`)
- `attendance_date` (Date)
- `attendance_type` (Enum: `STORE`, `MULTI_BRANCH`, `OUTSIDE_DUTY`)
- **Sesi 1 (Check-In):**
  - `check_in_at` (DateTime, Nullable)
  - `check_in_photo` (Varchar, Nullable) - Dihapus otomatis oleh Cron Job jika > 2 bulan[cite: 2]
  - `check_in_lat` / `check_in_lng` (Decimal)
- **Sesi 2 (Break-Out):**
  - `break_out_at` (DateTime, Nullable)
  - `break_out_photo` (Varchar, Nullable)
  - `break_out_lat` / `break_out_lng` (Decimal)
- **Sesi 3 (Break-In):**
  - `break_in_at` (DateTime, Nullable)
  - `break_in_photo` (Varchar, Nullable)
  - `break_in_lat` / `break_in_lng` (Decimal)
- **Sesi 4 (Out-Store):**
  - `out_store_at` (DateTime, Nullable)
  - `out_store_photo` (Varchar, Nullable)
  - `out_store_lat` / `out_store_lng` (Decimal)
- **Fitur Approval Tidak Break (No-Break):**[cite: 1]
  - `is_no_break_requested` (Boolean, default: false)
  - `no_break_status` (Enum: `NONE`, `PENDING`, `APPROVED`, `REJECTED`)
  - `no_break_approved_by` (BigInt, Nullable, FK -> `employees.id`) - Headstore[cite: 1]
- **Perhitungan Keterlambatan, Denda & Status Harian:**
  - `late_minutes` (Integer, default: 0)
  - `late_hours_rounded` (Integer, default: 0) - Pembulatan jam telat
  - `late_fine_amount` (Decimal 15,2, default: 0) - Nominal denda telat hari ini
  - `forgot_fine_amount` (Decimal 15,2, default: 0) - Nominal denda lupa absen hari ini
  - `daily_status` (Enum: `PRESENT`, `LATE`, `LEAVE_FULL`, `LEAVE_EARLY`, `OFF`, `ALPHA`)
  - `is_meal_counted` (Boolean, default: true) - Set `false` jika `LEAVE_EARLY` (Pulang Cepat), `LEAVE_FULL`, `OFF`, atau `ALPHA`
- `timestamps`

### 12. `leave_requests` (Pengajuan Izin & Pulang Cepat)
- `id` (BigInt, PK)
- `employee_id` (BigInt, FK -> `employees.id`)
- `leave_category` (Enum: `SICK`, `PERSONAL`, `EARLY_LEAVE_SICK`, `EARLY_LEAVE_PERSONAL`)
- `start_date` (Date)
- `end_date` (Date)
- `total_days` (Integer)
- `reason` (Text)
- `attachment_doctor_letter` (Varchar, Nullable) - Wajib untuk `SICK` atau susulan sebelum Check-In berikutnya[cite: 1]
- `attachment_opname_photo` (Varchar, Nullable) - Wajib jika `SICK` dan `total_days >= 3`[cite: 2]
- `is_medical_letter_pending` (Boolean, default: false) - True jika izin sakit di Hari H dan belum upload surat dokter
- `requires_weekend_replacement` (Boolean, default: false) - True jika terkena hari Sabtu/Minggu/Libur[cite: 1]
- `overall_status` (Enum: `PENDING`, `IN_PROGRESS`, `APPROVED`, `REJECTED`)
- `timestamps`

### 13. `leave_replacements` (pilihan Mandiri 2 Hari Pengganti Izin Weekend)
- `id` (BigInt, PK)
- `leave_request_id` (BigInt, FK -> `leave_requests.id`)
- `employee_id` (BigInt, FK -> `employees.id`)
- `original_leave_date` (Date) - Tanggal weekend yang ditinggalkan
- `replacement_date` (Date) - Tanggal weekday (hari Off) yang dipilih karyawan untuk masuk[cite: 1]
- `fulfillment_status` (Enum: `SCHEDULED`, `ATTENDED`, `MISSED_ALPHA`)
- `timestamps`

### 14. `leave_approvals` (Log Persetujuan Berjenjang)
- `id` (BigInt, PK)
- `leave_request_id` (BigInt, FK -> `leave_requests.id`)
- `step_order` (Integer)
- `approver_employee_id` (BigInt, Nullable, FK -> `employees.id`)
- `required_position_id` (BigInt, FK -> `positions.id`)
- `status` (Enum: `WAITING`, `APPROVED`, `REJECTED`)
- `notes` (Text, Nullable)
- `acted_at` (DateTime, Nullable)
- `timestamps`

### 15. `payrolls` (Rekapitulasi Slip Gaji Bulanan)[cite: 1, 2]
- `id` (BigInt, PK)
- `employee_id` (BigInt, FK -> `employees.id`)
- `period_month` (Integer) - Bulan (1-12)
- `period_year` (Integer) - Tahun (contoh: 2026)
- `total_present_days` (Integer) - Jumlah hari hadir fisik (pengali Uang Makan)
- `total_late_count` (Integer)
- `total_leave_count` (Integer)
- `total_alpha_count` (Integer) - Jika >= 1, maka insentif & kerajinan = 0[cite: 1]
- `basic_salary` (Decimal 15,2)[cite: 2]
- `total_meal_allowance` (Decimal 15,2) - `daily_meal_allowance * total_present_days`
- `diligence_allowance` (Decimal 15,2) - Rp 0 jika `total_alpha_count >= 1` atau terkena aturan potongan kerajinan
- `incentive_allowance` (Decimal 15,2) - Rp 0 jika `total_alpha_count >= 1`
- `position_allowance` (Decimal 15,2)[cite: 2]
- `total_late_fines` (Decimal 15,2) - Potongan denda telat nominal rupiah[cite: 1, 2]
- `total_forgot_fines` (Decimal 15,2) - Potongan denda lupa absen[cite: 2]
- `salary_cut_percentage` (Decimal 5,2, default: 0) - Persentase potongan gaji pokok dari akumulasi level penalti tertinggi
- `salary_cut_amount` (Decimal 15,2, default: 0) - Nominal potongan gaji pokok
- `incentive_cut_percentage` (Decimal 5,2, default: 0) - Persentase potongan uang insentif dari akumulasi level penalti tertinggi
- `incentive_cut_amount` (Decimal 15,2, default: 0) - Nominal potongan uang insentif
- `total_cut_percentage_amount` (Decimal 15,2, default: 0) - Total gabungan potongan persentase gaji & insentif
- `net_salary` (Decimal 15,2) - Total Gaji Bersih (Take Home Pay)
- `is_published` (Boolean, default: false) - Jika true, muncul di menu Slip Gaji APK karyawan (hanya tanggal 1-7)[cite: 1]
- `timestamps`

---

## B. TABEL ATURAN POTONGAN BERJENJANG & PENYESUAIAN JADWAL (UPDATE 2026-10-22)

### 16. `penalty_percentage_rules` (Master Persentase Potongan Berjenjang Dinamis)
- `id` (BigInt, PK)
- `category` (Enum: `LEAVE`, `OPNAME`, `LATE`, `ALPHA`) - Kategori pelanggaran / ketidakhadiran
- `tier_level` (Integer, default: 1) - Tingkat level (Level 1, Level 2, Level 3, dst - bebas ditambah oleh Admin)
- `min_occurrences` (Integer) - Ambang batas minimal kejadian (misal: 1x, 2x, 5x)
- `incentive_cut_percentage` (Decimal 5,2, default: 0) - Persentase pemotongan insentif (0 - 100%)
- `salary_cut_percentage` (Decimal 5,2, default: 0) - Persentase pemotongan gaji pokok (0 - 100%)
- `forfeit_diligence_allowance` (Boolean, default: false) - Jika true, uang kerajinan otomatis hangus (Rp 0)
- `description` (Varchar, Nullable) - Keterangan aturan
- `is_active` (Boolean, default: true)
- `timestamps`

### 17. `schedule_change_requests` (Pengajuan Tukar Hari Libur, Jam Kerja & Tugas Luar Mandiri)
- `id` (BigInt, PK)
- `employee_id` (BigInt, FK -> `employees.id`) - Karyawan pemohon
- `type` (Enum: `EXCHANGE_OFF_DAY`, `CHANGE_SHIFT`, `OUTSIDE_DUTY`) - Jenis pengajuan
- `status` (Enum: `PENDING`, `APPROVED`, `REJECTED`, default: `PENDING`)
- `original_off_date` (Date, Nullable) - Tanggal libur asal (untuk `EXCHANGE_OFF_DAY`)
- `target_off_date` (Date, Nullable) - Tanggal hari kerja yang ditukar menjadi libur (untuk `EXCHANGE_OFF_DAY`)
- `shift_target_date` (Date, Nullable) - Tanggal kerja yang disesuaikan jamnya (untuk `CHANGE_SHIFT`)
- `requested_shift_id` (BigInt, Nullable, FK -> `shifts.id`) - Shift baru yang dipilih
- `custom_check_in_time` (Time, Nullable) - Custom jam check-in
- `custom_break_out_time` (Time, Nullable) - Custom jam break-out
- `custom_break_in_time` (Time, Nullable) - Custom jam break-in
- `custom_out_store_time` (Time, Nullable) - Custom jam out-store
- `duty_date` (Date, Nullable) - Tanggal izin tugas luar toko (untuk `OUTSIDE_DUTY`)
- `reason` (Text) - Alasan pengajuan
- `rejection_reason` (Text, Nullable) - Alasan penolakan jika ditolak
- `approved_by_id` (BigInt, Nullable, FK -> `users.id`) - Penyetuju (Headstore/SPV/HRD)
- `approved_at` (DateTime, Nullable)
- `timestamps`

### Perubahan Kolom pada Tabel yang Ada:
1. **`positions`**:
   - `can_view_salary_slip` (Boolean, default: true) - Izin apakah karyawan dengan jabatan ini diperbolehkan melihat menu & rincian Slip Gaji di aplikasi APK.
2. **`employee_schedules`**:
   - `custom_check_in_time` (Time, Nullable)
   - `custom_break_out_time` (Time, Nullable)
   - `custom_break_in_time` (Time, Nullable)
   - `custom_out_store_time` (Time, Nullable)
   - `is_outside_duty` (Boolean, default: false) - True jika karyawan diizinkan absen di luar geofencing toko pada tanggal tersebut.