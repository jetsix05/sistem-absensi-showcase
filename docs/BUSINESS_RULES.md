# BUSINESS_RULES.md - Master Blueprint Alur Bisnis & Aturan Sistem
**Nama Proyek:** Sistem Absensi Multi-Cabang, Approval Berjenjang & Payroll (Web Admin Laravel 11 + FilamentPHP v3 & Mobile Android APK Flutter)

Dokumen ini adalah panduan komprehensif bagi developer, project lead, dan sistem AI untuk memahami seluruh alur logika bisnis, parameter operasional, formula perhitungan, serta mekanisme proteksi anti-kecurangan sistem ini.

---

## 1. Arsitektur Teknologi & Lingkungan
- **Backend & Web Admin:** Laravel 11, PHP 8.3, FilamentPHP v3 (Master CRUD, Dashboard Statistik, Review Pengajuan, dan Rekapitulasi Gaji).
- **Database:** MySQL 8.0 / MariaDB 10.11 (aaPanel Environment).
- **Mobile Client:** Flutter APK (Android) dengan arsitektur REST API Sanctum, Geolocation Service, Camera Controller, dan Proteksi Layar `FLAG_SECURE`.
- **WhatsApp Gateway:** WAHA (WhatsApp HTTP API) berbasis Docker pada port lokal `http://localhost:3000` dengan session aktif untuk notifikasi dua arah.
- **Image Processing:** `Intervention Image` untuk optimasi foto selfie presensi (<150KB) dan burn-in watermark koordinat GPS + Server Timestamp.
- **Storage Management:** Scheduler otomatis menghapus file foto fisik absensi yang berusia > 60 hari untuk efisiensi penyimpanan server.

---

## 2. Manajemen Akun Karyawan & Autentikasi
1. **Format User ID (Username Login APK):**
   - Karyawan login menggunakan **Kode Karyawan Toko** (contoh: `EMP-001`), **BUKAN NIK KTP**.
2. **Password Awal Terpusat:**
   - Password awal diset seragam (default: `123456`) dan diinputkan oleh Admin di Tab 1 Master Karyawan.
3. **Sinkronisasi Otomatis `users` & `employees`:**
   - Saat Admin menyimpan profil karyawan di Filament Admin, sistem secara otomatis membuatkan akun login di tabel `users` dan mengaitkannya ke tabel `employees`.
4. **Notifikasi Kredensial via WhatsApp:**
   - Tersedia toggle opsi kirim pesan WhatsApp otomatis ke nomor karyawan via WAHA Gateway berisikan User ID dan Password awal begitu akun berhasil didaftarkan.

---

## 3. Cabang, Geofencing & Mekanisme Anti-Curang (Anti-Fraud)
1. **Multi-Cabang & Geofencing Presensi:**
   - Setiap cabang memiliki koordinat GPS (`latitude`, `longitude`) dan batas radius meter presensi (`radius_meters`).
   - Karyawan reguler hanya diizinkan absen di 1 cabang penempatan utamanya (`primary_branch_id`).
   - Karyawan multi-cabang (diatur melalui pivot table `employee_branches`) dapat melakukan absensi di cabang mana pun yang terdaftar padanya.
2. **Proteksi Anti-Curang pada APK:**
   - **Kamera Langsung (No Gallery):** APK memicu kamera internal secara langsung, tanpa akses pemilihan foto dari galeri HP.
   - **Anti-Fake GPS (Mock Location Detection):** APK mendeteksi flag `is_mock_location`. Jika aplikasi Fake GPS aktif, sistem server langsung menolak dengan status HTTP 403 `FAKE_GPS_DETECTED`.
   - **Selective Device Binding (Kunci HP):** Tersedia opsi `is_device_locked` dan `registered_device_id` per karyawan di Master Data. Jika diaktifkan, akun tersebut hanya bisa login dan presensi dari 1 perangkat HP yang telah terdaftar.
   - **Burn-In Watermark:** Setiap foto selfie yang diunggah dikompresi menjadi <150KB dan dicap watermark berisikan Nama Karyawan, Kode Karyawan, Cabang/Tugas Luar, Koordinat GPS, dan Timestamp server.

---

## 4. Jam Kerja (4 Sesi Presensi) & Fitur Approval Tidak Break
1. **4 Sesi Presensi Harian:**
   - **Sesi 1 - Check-In:** Jam masuk kerja (tombol mulai dibuka pukul 06:00 pagi).
   - **Sesi 2 - Break-Out:** Jam mulai istirahat siang/sore.
   - **Sesi 3 - Break-In:** Jam selesai istirahat dan kembali bertugas.
   - **Sesi 4 - Out-Store:** Jam pulang kerja (batas maksimal tombol adalah pukul 24:00 malam).
2. **Aturan Tidak Break (No-Break Approval):**
   - Jika operasional toko sangat sibuk dan karyawan tidak mengambil istirahat (tidak melakukan Break-Out & Break-In), karyawan **wajib mengajukan permohonan "Tidak Break" ke Headstore via APK**.
   - Pengajuan mengirimkan pesan notifikasi WhatsApp ke nomor Headstore.
   - Jika disetujui oleh Headstore, karyawan **bebas dari denda lupa absen Break-Out dan Break-In** untuk hari tersebut (tidak dihitung lembur/overtime, murni pembebasan denda).
   - Begitu direspon (disetujui/ditolak), sistem mengirimkan WhatsApp konfirmasi kembali ke karyawan pemohon.

---

## 5. Perhitungan Denda Otomatis & Persentase Potongan Berjenjang

### A. Denda Nominal Rupiah (Berjalan Berdasarkan Kejadian)
1. **Denda Terlambat Check-In:**
   - Jam ke-1 keterlambatan dikenakan tarif jam pertama (default: Rp 2.000).
   - Jam ke-2 dan kelipatan jam berikutnya dikenakan tarif per jam tambahan (default: Rp 200/jam).
   - Menit keterlambatan dibulatkan ke atas per jam (contoh: telat 15 menit = 1 jam; telat 65 menit = 2 jam).
2. **Denda Lupa Absen 4 Sesi:**
   - Denda berlaku independen per sesi jika karyawan lupa melakukan absensi: Check-In, Break-Out, Break-In, dan Out-Store (kecuali sesi break yang disetujui No-Break).

### B. Master Persentase Potongan Dinamis Tanpa Batas Level (`penalty_percentage_rules`)
Admin bebas menambah level tier potongan sebanyak mungkin sesuai kebijakan internal:
- **4 Kategori:**
  1. `LEAVE`: Akumulasi izin biasa atau sakit mandiri dalam bulan berjalan.
  2. `OPNAME`: Akumulasi hari rawat inap rumah sakit dalam bulan berjalan.
  3. `LATE`: Akumulasi frekuensi keterlambatan dalam bulan berjalan.
  4. `ALPHA`: Akumulasi frekuensi mangkir/alpha dalam bulan berjalan.
- **Komponen yang Dipotong:**
  - `incentive_cut_percentage`: Pemotongan persentase uang insentif (0 - 100%).
  - `salary_cut_percentage`: Pemotongan persentase gaji pokok (0 - 100%).
  - `forfeit_diligence_allowance`: Jika bernilai `true`, uang kerajinan otomatis hangus (Rp 0).
- **Aturan Evaluasi:** Mengambil **persentase dari tingkat level tertinggi yang dicapai** per kategori.
- **Simultan:** Denda keterlambatan nominal rupiah dan potongan persentase gaji/insentif diterapkan secara simultan pada slip gaji.

---

## 6. Aturan Pengajuan Izin, Hari Libur & Ganti Hari Mandiri
1. **Batas Kuota Bulanan:** Izin dibatasi maksimal 3 kali dalam satu bulan kalender (dapat diatur dinamis di Master Kebijakan Izin).
2. **Persyaratan Lampiran Bukti Sakit:**
   - Izin Sakit Biasa: Wajib melampirkan foto Surat Dokter.
   - Izin Sakit Rawat Inap (Opname): Wajib melampirkan foto Surat Opname / Keterangan Rawat Inap.
3. **Izin Weekend / Hari Libur Toko = Wajib Ganti 2x di Weekday (Pilihan Mandiri):**
   - Jika tanggal izin jatuh pada hari Sabtu, Minggu, atau tanggal merah terdaftar di Master Hari Libur:
     - Karyawan **wajib langsung memilih 2 tanggal pengganti di hari kerja (Weekday)** yang diambil dari jadwal libur (OFF) mingguan karyawan.
     - APK menampilkan 4 opsi tanggal libur mingguan berikutnya.
     - Pada jadwal kerja dan dashboard, tanggal tersebut berstatus `MASUK (Ganti Izin Tanggal X)`.
4. **Kasus Izin di Hari H / Pulang Cepat (LEAVE_EARLY setelah Check-In):**
   - Jika karyawan sudah Check-In lalu mendadak sakit/izin pulang cepat:
     a. **Uang Makan:** Tidak dihitung (Rp 0) untuk hari tersebut.
     b. **Kuota Izin:** Memotong 1 kuota izin bulanan.
     c. **Pembebasan Denda Sisa Sesi:** Bebas dari denda lupa Break-Out, Break-In, dan Out-Store.
     d. **Weekend Replacement:** Tetap wajib memilih 2 hari pengganti jika kejadian pada weekend.
     e. **Susulan Surat Dokter (Blokir Check-In Masuk):** Jika surat dokter belum ada saat pulang (`is_medical_letter_pending = 1`), sistem **memblokir tombol Check-In pada hari kerja berikutnya** sampai karyawan mengunggah foto Surat Dokter yang sah.

---

## 7. Penyesuaian Jadwal Mandiri & Tugas Luar (`schedule_change_requests`)
1. **Penyesuaian Jadwal Mandiri Karyawan:**
   - Murni untuk perubahan jadwal pribadi karyawan sendiri. Penukaran shift antar rekan kerja dilakukan oleh Admin/HRD di Web Admin.
2. **3 Jenis Pengajuan:**
   - `EXCHANGE_OFF_DAY` (Tukar Hari Libur Mandiri): Menukar hari libur mingguan (OFF) ke hari kerja lain.
   - `CHANGE_SHIFT` (Penyesuaian Jam Kerja): Menyesuaikan jam masuk atau memilih shift baru.
   - `OUTSIDE_DUTY` (Izin Absen Luar Toko): Pengajuan tugas luar terjadwal yang membebaskan radius geofencing toko pada tanggal tersebut.
3. **Persetujuan Otomatis:** Setelah disetujui Headstore/SPV, sistem langsung memperbarui record `employee_schedules`.

---

## 8. Notifikasi WhatsApp Dua Arah Otomatis (WAHA Gateway)
1. **Saat Pengajuan Diajukan oleh Karyawan:**
   - Sistem mengirim pesan WhatsApp otomatis ke atasan/approver (Headstore/SPV/HRD) berisi nama, kode karyawan, kategori pengajuan, tanggal, dan alasan.
2. **Saat Pengajuan Direspon oleh Atasan:**
   - Sistem mengirim pesan WhatsApp konfirmasi ke nomor WhatsApp karyawan pemohon berisi status (`APPROVED` atau `REJECTED`) beserta catatan alasannya.

---

## 9. Komponen Payroll, Proteksi Slip Gaji & Penalti Alpha
1. **Komponen Slip Gaji Bulanan:**
   - Gaji Pokok (`basic_salary`).
   - Uang Makan (`daily_meal_allowance * total_present_days`).
   - Tunjangan Jabatan (`position_allowance`).
   - Uang Kerajinan (`diligence_allowance`).
   - Uang Insentif (`incentive_allowance`).
   - Potongan Denda Telat & Lupa Absen (Nominal Rupiah).
   - Potongan Persentase Gaji Pokok & Insentif (Berdasarkan evaluasi Master Potongan Berjenjang).
2. **Penalti Alpha 1x:**
   - Jika karyawan tercatat **Alpha (Mangkir) 1 kali saja dalam 1 bulan**, maka **Uang Kerajinan dan Uang Insentif otomatis HANGUS (Rp 0)** pada slip gaji bulan tersebut.
3. **Window Time Akses Slip Gaji (Tanggal 1 - 7):**
   - Menu dan rincian Slip Gaji **hanya dapat diakses pada tanggal 1 s/d 7 setiap bulannya**.
   - Pada tanggal 8 sampai akhir bulan, tombol Slip Gaji di APK **otomatis disembunyikan (hilang)**, dan API mengembalikan status 403 `SLIP_WINDOW_EXPIRED`.
4. **Proteksi Anti-Screenshot (`FLAG_SECURE`):**
   - Halaman Slip Gaji di APK dilindungi flag `FLAG_SECURE` (`flutter_windowmanager`) agar tidak dapat di-screenshot atau direkam layar (tampilan menjadi layar hitam di perekam layar).
5. **Kontrol Hak Akses Jabatan:**
   - Ditentukan oleh toggle `can_view_salary_slip` pada Master Jabatan di Web Admin.

---

## 10. Dashboard & Monitoring
1. **Dashboard APK:**
   - Foto master selfie karyawan di header.
   - Jam server real-time WIB.
   - Kartu presensi hari ini (jam masuk/istirahat/pulang, status geofencing).
   - Ringkasan denda rupiah berjalan, estimasi potongan persentase bulan ini, dan sisa kuota izin.
2. **Dashboard Web Admin (Filament v3):**
   - Kartu statistik harian: Total Karyawan, Hadir, Terlambat, Izin, Alpha, Permintaan Tidak Break.
   - Drill-down detail presensi dan foto selfie ber-watermark.
   - Rekapitulasi estimasi potongan persentase dan total denda rupiah berjalan.
