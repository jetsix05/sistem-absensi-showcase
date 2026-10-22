# API_SPEC.md - Spesifikasi REST API Mobile APK (Flutter)

Dokumen ini adalah acuan resmi integrasi REST API antara backend Laravel 11 dan aplikasi mobile Android APK (Flutter) untuk **Sistem Absensi Multi-Cabang, Approval & Payroll**.

---

## 1. Standar Format Request & Response

- **Base URL:** `http://<IP_SERVER_ATAU_DOMAIN>/api` (contoh: `http://192.9.200.237/api`)
- **Default Headers:**
  ```http
  Accept: application/json
  Authorization: Bearer <token_sanctum>
  ```
- **Struktur Standard Response Sukses:**
  ```json
  {
    "success": true,
    "message": "Pesan deskripsi jika ada",
    "data": { ... }
  }
  ```
- **Struktur Standard Response Gagal / Error:**
  ```json
  {
    "success": false,
    "message": "Pesan kesalahan",
    "code": "ERROR_CODE_OPTIONAL",
    "errors": { ... }
  }
  ```

---

## 2. Modul Autentikasi (Auth)

### A. Login Karyawan
- **Endpoint:** `POST /api/auth/login`
- **Akses:** Publik (Tanpa Token)
- **Request Body (JSON / Form-Data):**
  | Parameter | Tipe | Wajib? | Keterangan |
  |---|---|---|---|
  | `username` | string | Ya | User ID / **Kode Karyawan** (contoh: `EMP-001`) atau Email |
  | `password` | string | Ya | Password login (default awal: `123456`) |
  | `device_id`| string | Opsional | Android Device ID unik untuk validasi Selective Device Binding |
- **Response Sukses (200 OK):**
  ```json
  {
    "success": true,
    "message": "Login berhasil.",
    "data": {
      "token": "1|abcdef123456...",
      "token_type": "Bearer",
      "employee": {
        "id": 1,
        "employee_code": "EMP-001",
        "full_name": "Danny",
        "phone_wa": "08123456789",
        "bpjs_number": "00012345678",
        "npwp_number": "01.234.567.8-123.000",
        "photo_selfie_master": "http://192.9.200.237/storage/...",
        "position": {
          "id": 3,
          "name": "Headstore Cabang",
          "level": 3
        },
        "primary_branch": {
          "id": 1,
          "name": "Toko Pusat",
          "latitude": -6.20000000,
          "longitude": 106.81666600,
          "radius_meters": 50
        },
        "can_multi_branch": false,
        "assigned_branches": [],
        "can_outside_duty": false,
        "is_device_locked": false,
        "has_pending_medical_letter": false
      }
    }
  }
  ```

### B. Profil Karyawan Login
- **Endpoint:** `GET /api/auth/me`
- **Headers:** `Authorization: Bearer <token>`
- **Response (200 OK):** Mengembalikan data user dan detail karyawan saat ini.

### C. Logout
- **Endpoint:** `POST /api/auth/logout`
- **Headers:** `Authorization: Bearer <token>`
- **Response (200 OK):** Token dicabut dan tidak dapat digunakan lagi.

---

## 3. Modul Dashboard Mobile

### Unified Dashboard Summary
- **Endpoint:** `GET /api/dashboard`
- **Headers:** `Authorization: Bearer <token>`
- **Keterangan:** Endpoint terpadu untuk memuat seluruh kebutuhan halaman utama APK dalam 1x pemanggilan (Jadwal hari ini, status presensi, rekap denda, kuota izin, dan status insentif).
- **Response (200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "server_time": "2026-10-22 10:30:00",
      "server_time_formatted": "10:30 WIB",
      "can_view_salary_slip": false,
      "employee": {
        "id": 1,
        "code": "EMP-001",
        "name": "Danny",
        "position": "Headstore Cabang",
        "branch": "Toko Pusat Utama YHP",
        "photo": "http://192.9.200.237/storage/photos/master_selfie.jpg",
        "has_pending_medical_letter": false
      },
      "today": {
        "date": "2026-10-22",
        "date_formatted": "Kamis, 22 Oktober 2026",
        "schedule_type": "REGULAR",
        "is_outside_duty": false,
        "notes": null,
        "shift": {
          "name": "Shift Pagi",
          "check_in": "09:00:00",
          "break_out": "13:00:00",
          "break_in": "17:00:00",
          "out_store": "21:00:00"
        },
        "attendance": {
          "daily_status": "PRESENT",
          "check_in_at": "08:55",
          "break_out_at": null,
          "break_in_at": null,
          "out_store_at": null,
          "no_break_status": "NONE"
        }
      },
      "month_summary": {
        "month_name": "Oktober 2026",
        "total_late_minutes": 15,
        "total_fines": 2000,
        "total_fines_formatted": "Rp 2.000",
        "alpha_count": 0,
        "late_count": 1,
        "leave_count": 0,
        "opname_count": 0,
        "salary_cut_percentage": 0,
        "incentive_cut_percentage": 0,
        "forfeit_diligence_allowance": false,
        "is_alpha_penalty": false,
        "incentive_status": "Aman (Berhak Menerima Insentif)",
        "remaining_leave_quota": 3
      }
    }
  }
  ```

---

## 4. Modul Presensi 4 Sesi Harian

### A. Status Absensi Hari Ini
- **Endpoint:** `GET /api/attendance/today`
- **Headers:** `Authorization: Bearer <token>`
- **Keterangan:** Cek jam buka/tutup tombol, sesi yang sudah diambil, dan status absensi hari ini.

### B. Sesi 1: Check-In Masuk Toko
- **Endpoint:** `POST /api/attendance/check-in`
- **Headers:** `Authorization: Bearer <token>`, `Content-Type: multipart/form-data`
- **Request Body:**
  | Parameter | Tipe | Wajib? | Keterangan |
  |---|---|---|---|
  | `branch_id` | integer | Ya | ID Cabang tempat absen |
  | `latitude` | numeric | Ya | Titik koordinat GPS Lat karyawan |
  | `longitude` | numeric | Ya | Titik koordinat GPS Lng karyawan |
  | `photo` | file / base64 | Ya | Foto selfie wajah dari kamera langsung |
  | `device_id` | string | Opsional | Device ID perangkat |
  | `attendance_type` | enum | Opsional | `STORE`, `MULTI_BRANCH`, atau `OUTSIDE_DUTY` |
  | `is_mock_location`| boolean | Ya | Flag anti-cheat: `true` jika mendeteksi Fake GPS |
- **Pengecekan Keamanan (Server-side):**
  1. Jika `is_mock_location == true` $\to$ Ditolak error 403 `FAKE_GPS_DETECTED`.
  2. Jika `has_pending_medical_letter == true` $\to$ Ditolak error 400 (Wajib upload surat dokter).
  3. Jika `is_device_locked == true` dan `device_id` berbeda $\to$ Ditolak error 403.
  4. Jika jarak GPS ke toko $> \text{radius\_meters}$ toko $\to$ Ditolak error 400.
  5. Foto selfie dikompres $< 150\text{KB}$ dan dibakar (*burn-in*) watermark GPS & Timestamp.

### C. Sesi 2: Break-Out (Mulai Istirahat)
- **Endpoint:** `POST /api/attendance/break-out`
- **Request Body:** `latitude`, `longitude`, `photo`, `is_mock_location`.

### D. Sesi 3: Break-In (Selesai Istirahat)
- **Endpoint:** `POST /api/attendance/break-in`
- **Request Body:** `latitude`, `longitude`, `photo`, `is_mock_location`.

### E. Sesi 4: Out-Store (Check-Out Pulang)
- **Endpoint:** `POST /api/attendance/out-store`
- **Request Body:** `latitude`, `longitude`, `photo`, `is_mock_location`.

### F. Pengajuan Tidak Istirahat (No-Break)
- **Endpoint:** `POST /api/attendance/request-no-break`
- **Keterangan:** Diajukan karyawan setelah Check-In jika memutuskan tidak mengambil waktu istirahat agar bebas denda lupa absen Break.

### G. Riwayat Presensi Karyawan
- **Endpoint:** `GET /api/attendance/history?month=10&year=2026`
- **Response:** Daftar riwayat absensi bulanan beserta foto ber-watermark, jam per sesi, status keterlambatan, dan denda.

---

## 5. Modul Pengajuan Izin & Cuti

### A. Kuota Izin & Aturan Libur
- **Endpoint:** `GET /api/leave/rules-and-quota`
- **Response:** Kuota maksimal (3x/bulan), jumlah izin yang sudah diambil, kuota tersisa, dan daftar hari libur toko.

### B. Daftar Hari Libur (OFF) Karyawan untuk Hari Pengganti
- **Endpoint:** `GET /api/leave/off-days`
- **Response:** Daftar tanggal OFF mingguan karyawan untuk 3 bulan ke depan yang bisa dipilih sebagai 2 hari pengganti weekend.

### C. Kirim Pengajuan Izin
- **Endpoint:** `POST /api/leave/submit`
- **Headers:** `Content-Type: multipart/form-data`
- **Request Body:**
  | Parameter | Tipe | Wajib? | Keterangan |
  |---|---|---|---|
  | `leave_category` | enum | Ya | `SICK`, `PERSONAL`, `EARLY_LEAVE_SICK`, `EARLY_LEAVE_PERSONAL` |
  | `start_date` | date | Ya | Format: `YYYY-MM-DD` |
  | `end_date` | date | Ya | Format: `YYYY-MM-DD` |
  | `reason` | string | Ya | Alasan izin |
  | `attachment_doctor_letter` | file | Kondisional | Wajib jika `SICK` biasa |
  | `attachment_opname_photo` | file | Kondisional | Wajib jika `SICK` dan $\ge 3$ hari |
  | `replacement_dates[]` | array date | Kondisional | Wajib diisi 2x tanggal pengganti jika mengenai Sabtu/Minggu/Libur |

### D. Unggah Susulan Surat Dokter (Buka Blokir Check-In)
- **Endpoint:** `POST /api/leave/upload-pending-doctor-letter`
- **Request Body:** `leave_request_id` (integer), `doctor_letter` (file foto/PDF).
- **Efek:** Otomatis membuka blokir Check-In masuk karyawan.

### E. Daftar Pengajuan Izin Saya
- **Endpoint:** `GET /api/leave/my-requests`
- **Response:** Riwayat izin dan status tahapan persetujuan berjenjang (Headstore, HRD, Owner).

---

## 6. Modul Approval Khusus Headstore / Atasan

### A. Permintaan Tidak Break (No-Break) Cabang
- **Endpoint:** `GET /api/headstore/no-break-requests`
- **Response:** Daftar karyawan di cabang yang mengajukan No-Break hari ini.

### B. Respon Permintaan Tidak Break
- **Endpoint:** `POST /api/headstore/respond-no-break`
- **Request Body:**
  - `attendance_id`: integer
  - `action`: `APPROVE` atau `REJECT`

### C. Daftar Izin Bawahan Menunggu Approval
- **Endpoint:** `GET /api/headstore/pending-leaves`
- **Response:** Daftar pengajuan izin bawahan yang memerlukan tanda tangan user.

### D. Respon Approval Izin
- **Endpoint:** `POST /api/headstore/respond-leave`
- **Request Body:**
  - `approval_id`: integer
  - `action`: `APPROVE` atau `REJECT`
  - `notes`: string (opsional jika approve, wajib jika reject)

### E. Daftar Pengajuan Penyesuaian Jadwal Bawahan
- **Endpoint:** `GET /api/schedule-change/headstore/requests`
- **Response:** Daftar pengajuan tukar hari libur, tukar jam kerja, dan tugas luar dari bawahan pada cabang yang sama.

### F. Respon Approval Penyesuaian Jadwal
- **Approve:** `POST /api/schedule-change/headstore/approve/{id}`
  - Efek: Status menjadi `APPROVED`, memperbarui tabel `employee_schedules`, dan mengirim WhatsApp konfirmasi ke karyawan.
- **Reject:** `POST /api/schedule-change/headstore/reject/{id}`
  - Request Body: `{"reason": "Alasan penolakan"}`
  - Efek: Status menjadi `REJECTED` dan mengirim WhatsApp pemberitahuan penolakan ke karyawan.

---

## 7. Modul Slip Gaji (Payroll)

### A. Aturan Akses & Pengamanan Digital
1. **Window Time Slip Gaji (Tanggal 1 - 7):**
   - Endpoint slip gaji hanya dapat diakses pada **tanggal 1 s/d 7** setiap bulannya.
   - Di luar tanggal 1-7, sistem mengembalikan HTTP `403 Forbidden` dengan pesan: *"Slip gaji hanya dapat diakses pada tanggal 1-7 setiap bulannya."*. Tombol di APK otomatis disembunyikan jika `can_view_salary_slip: false`.
2. **Izin Jabatan:**
   - Karyawan hanya bisa melihat slip gaji jika jabatannya memiliki toggle `can_view_salary_slip = true` pada Master Jabatan.
3. **Anti-Screenshot & Layar Hitam (Anti-Fraud):**
   - Halaman Slip Gaji pada APK Android diproteksi menggunakan flag `FLAG_SECURE` (`flutter_windowmanager`) sehingga tidak dapat di-screenshot atau direkam layar.

### B. Daftar Slip Gaji Saya
- **Endpoint:** `GET /api/payroll/my-slips`
- **Response:** Daftar slip gaji bulanan yang sudah diterbitkan (`is_published = true`).

### C. Detail Rincian Slip Gaji
- **Endpoint:** `GET /api/payroll/my-slips/{id}`
- **Response:**
  - Rincian gaji pokok (`basic_salary`), uang makan fisik (`total_meal_allowance`), tunjangan jabatan (`position_allowance`).
  - Uang kerajinan (`diligence_allowance`) & insentif (`incentive_allowance`) dengan status hangus jika terdapat penalti Alpha $\ge 1$ atau terkena aturan potongan kerajinan.
  - Rincian potongan rupiah denda telat (`total_late_fines`) dan lupa absen (`total_forgot_fines`).
  - Rincian persentase potongan berjenjang:
    - `salary_cut_percentage` & `salary_cut_amount`
    - `incentive_cut_percentage` & `incentive_cut_amount`
    - `total_cut_percentage_amount`
  - Total gaji bersih (*Take Home Pay* / `net_salary`).

---

## 8. Modul Penyesuaian Jadwal & Tugas Luar (Schedule Changes)

### A. Mengajukan Tukar Hari Libur (Exchange OFF Day)
- **Endpoint:** `POST /api/schedule-change/exchange-off`
- **Request Body (JSON):**
  | Parameter | Tipe | Wajib? | Keterangan |
  |---|---|---|---|
  | `original_off_date` | date | Ya | Tanggal libur (OFF) asal yang ingin ditukar |
  | `target_off_date` | date | Ya | Tanggal kerja target yang diinginkan menjadi hari libur baru |
  | `reason` | string | Ya | Alasan penukaran hari libur |

### B. Mengajukan Penyesuaian Jam Kerja / Shift
- **Endpoint:** `POST /api/schedule-change/change-shift`
- **Request Body (JSON):**
  | Parameter | Tipe | Wajib? | Keterangan |
  |---|---|---|---|
  | `shift_target_date` | date | Ya | Tanggal kerja yang ingin diubah jam kerjanya |
  | `requested_shift_id` | integer | Opsional | ID Master Shift baru yang dipilih |
  | `custom_check_in_time` | string (H:i) | Opsional | Jam check-in kustom |
  | `custom_break_out_time` | string (H:i) | Opsional | Jam break-out kustom |
  | `custom_break_in_time` | string (H:i) | Opsional | Jam break-in kustom |
  | `custom_out_store_time` | string (H:i) | Opsional | Jam out-store kustom |
  | `reason` | string | Ya | Alasan perubahan jam kerja |

### C. Mengajukan Izin Absen Luar Toko (Tugas Luar Terjadwal)
- **Endpoint:** `POST /api/schedule-change/outside-duty`
- **Request Body (JSON):**
  | Parameter | Tipe | Wajib? | Keterangan |
  |---|---|---|---|
  | `duty_date` | date | Ya | Tanggal pelaksanaan tugas luar |
  | `reason` | string | Ya | Deskripsi penugasan luar / dinas |

### D. Riwayat Pengajuan Jadwal Karyawan
- **Endpoint:** `GET /api/schedule-change/my-requests`
- **Response:** Daftar seluruh permohonan penyesuaian jadwal milik karyawan yang login beserta statusnya (`PENDING`, `APPROVED`, `REJECTED`).

### E. Daftar Pengajuan Bawahan (Headstore / SPV Review)
- **Endpoint:** `GET /api/schedule-change/headstore/requests`
- **Response:** Daftar pengajuan penyesuaian jadwal dari seluruh rekan bawahan pada cabang yang sama.

### F. Tindakan Persetujuan & Penolakan
- **Approve:** `POST /api/schedule-change/headstore/approve/{id}`
  - Efek: Status menjadi `APPROVED`, otomatis mengupdate data di tabel `employee_schedules`, dan mengirimkan pesan WhatsApp konfirmasi ke nomor WA karyawan.
- **Reject:** `POST /api/schedule-change/headstore/reject/{id}`
  - Request Body: `{"reason": "Alasan penolakan"}`
  - Efek: Status menjadi `REJECTED` dan mengirimkan pesan notifikasi WhatsApp penolakan beserta alasannya ke pemohon.
