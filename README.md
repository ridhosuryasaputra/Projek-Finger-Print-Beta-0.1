# Projek-Finger-Print-Beta-0.1

Aplikasi web untuk menampilkan hasil presensi fingerprint siswa dari database **JIBAS**.

> **Projek ini milik grup ekskul Networking SMK Muhammadiyah 2 Metro.**
> Dikembangkan sebagai projek belajar dan untuk kebutuhan sekolah.

---

## 1. Tentang projek

Sekolah memakai JIBAS dan mesin fingerprint, tetapi hasil presensinya belum mudah dilihat oleh guru,
bagian kesiswaan, kepala sekolah, dan orang tua. Aplikasi ini menampilkan data presensi itu dalam
halaman web yang sederhana dan bisa dibuka dari HP.

Aplikasi ini **hanya membaca data dari JIBAS (read-only)**. Database JIBAS tidak pernah diubah.
Yang dikelola aplikasi ini hanya akun pengguna dan penugasan kelas, disimpan di SQLite lokal.

## 2. Rancangan aplikasi

**File terbaru:** `presensi-jibas V.05` (V0.5). Versi sebelumnya: V.04, V.03, V.02, dan `presensi jibas V.01.zip`.

**Teknologi:** Laravel 12 (PHP 8.2), Nginx + PHP-FPM, Debian 12 (VM / Proxmox LXC), SQLite untuk akun lokal,
PhpSpreadsheet untuk ekspor Excel.

**Sumber data dari JIBAS**

| Data | Database JIBAS |
|---|---|
| Scan fingerprint (NIS, jam) | `jbsumum.cardscandata` (bawaan) atau `jbssat.frpresence` |
| Siswa, kelas, dan tingkat | `jbsakad` |
| Akun dan data guru | `jbsuser.login`, `jbssdm.pegawai` |

Sumber scan dipilih lewat `PRESENSI_SUMBER_SCAN` di `.env` (`cardscandata` atau `frpresence`).

**Data yang disimpan aplikasi sendiri (SQLite lokal)**

| Data | Keterangan |
|---|---|
| Akun admin, kesiswaan, kepala sekolah, guru, orang tua | Tabel `pengguna` |
| Hubungan orang tua ke siswa | Tabel `ortu_siswa` |
| Penugasan guru ke kelas | Tabel `guru_kelas` |

Tabel lokal tidak memakai foreign key ke JIBAS, jadi perubahan data di JIBAS tidak membuat aplikasi error.

**Peran pengguna**

| Peran | Masuk dengan | Yang bisa dilihat |
|---|---|---|
| Publik / orang tua | Tanpa login, halaman `/cari` | Jam masuk dan pulang hari ini untuk satu NIS |
| Guru (wali kelas) | NIP + password JIBAS, atau akun lokal dari admin | Hanya kelas yang ditugaskan admin (harian dan bulanan), plus ekspor kelasnya |
| Kesiswaan | Akun dari sekolah | Ringkasan seluruh sekolah, semua kelas, cari siswa, riwayat siswa, ekspor |
| Kepala Sekolah | Akun dari sekolah | Sama dengan kesiswaan, untuk memantau |
| Admin | Akun dari sekolah | Semua halaman kesiswaan + kelola akun + buat akun orang tua per kelas |
| Orang tua (akun) | Akun dari sekolah | Hanya presensi anaknya sendiri |

**Aturan presensi**
- Masuk = scan pertama hari itu, pulang = scan terakhir.
- Terlambat jika scan pertama melewati jam masuk (`PRESENSI_JAM_MASUK`, toleransi `PRESENSI_TOLERANSI`).
- Hari sekolah = tanggal yang punya minimal satu scan siswa.
- Di halaman `/cari` dan ekspor harian, siswa tanpa scan dianggap **Alpa** setelah batas jam pulang
  (bawaan 14:00, kunci `jam_pulang`). Hari Minggu ditampilkan sebagai libur.

**Keamanan**
- Koneksi ke JIBAS memakai user MySQL khusus yang hanya boleh `SELECT`.
- Port database JIBAS tidak dibuka ke internet; yang dibuka hanya web (80/443) dengan HTTPS.
- Orang tua hanya bisa membuka siswa yang terhubung ke akunnya, dan guru hanya kelas yang ditugaskan (dicek di server).
- Password akun lokal disimpan dengan bcrypt.
- Pembatasan percobaan: login 10 kali per menit, halaman `/cari` 30 kali per menit, ekspor 20 kali per menit.
- Halaman `/cari` hanya menerima NIS yang persis sama dan hanya menampilkan siswa aktif.

**Isi folder**

    app/        kode aplikasi (controller, service, middleware, perintah artisan)
    config/     pengaturan presensi (pemetaan tabel JIBAS, jam masuk, sumber scan)
    public/     logo, foto gedung, latar, dan font tampilan
    resources/  tampilan halaman
    routes/     daftar alamat halaman
    database/   migrasi akun lokal dan data contoh untuk uji coba
    deploy/     skrip pemasangan dan pembaruan server

**Pemasangan singkat:** salin folder ke server Debian 12, lalu jalankan `bash deploy/install-lxc.sh`.
Untuk memperbarui versi lama: `bash deploy/update.sh` (migrasi database dijalankan otomatis,
pengaturan `.env` tidak ditimpa).

Pemasangan manual: `composer install`, salin `.env.example` menjadi `.env`, isi `JIBAS_DB_*`,
jalankan `php artisan key:generate` dan `php artisan migrate`, lalu buat admin pertama dengan
`php artisan presensi:admin`.

## 3. Yang diperbarui dari V0 ke V0.1

**V0** hanya punya peran **guru**.

**V0.1 menambahkan:**
- Peran baru: **kesiswaan**, **admin**, dan **orang tua**.
- Akun lokal (admin, kesiswaan, orang tua) di database SQLite, terpisah dari JIBAS.
- Halaman login satu pintu untuk semua peran (akun lokal dulu, lalu akun guru JIBAS).
- Ringkasan presensi seluruh sekolah per kelas, lengkap dengan persentase kehadiran.
- Pencarian siswa dan riwayat presensi bulanan per siswa.
- Filter status (hadir, terlambat, belum scan) pada rekap harian kelas.
- Admin dapat mengelola akun: tambah, ubah, nonaktifkan, hapus, dan hubungkan orang tua dengan siswa.
- **Pembuatan akun orang tua massal per kelas**, dengan password acak yang bisa dicetak.
- Ganti password sendiri untuk akun lokal.
- Perintah `php artisan presensi:admin` untuk membuat admin pertama.
- Skrip `deploy/update.sh` untuk memperbarui server tanpa menimpa pengaturan.
- Skrip `deploy/install-lxc.sh` diperbarui: menyiapkan SQLite, migrasi, dan membuat admin.
- Tampilan baru dengan menu per peran.

## 4. Yang diperbarui dari V0.1 ke V0.2

**V0.2 menambahkan:**
- Peran baru: **Kepala Sekolah**, dengan akses setara kesiswaan untuk memantau (ringkasan sekolah,
  filter kelas, cari siswa). Kepala sekolah tidak bisa membuka menu admin.
- **Akun guru (wali kelas) lokal**: admin membuatnya dari menu Pengguna, lalu mencentang kelas yang
  diwalikan. Username guru lokal tidak boleh sama dengan NIP pegawai JIBAS, supaya tidak bentrok
  dengan login guru JIBAS.
- **Penugasan kelas guru oleh admin**: disimpan di tabel lokal `guru_kelas`, tidak menyentuh JIBAS.
- Pencarian siswa lebih fleksibel: setiap kata dicocokkan ke NIS atau nama (urutan kata bebas),
  dan NIS yang persis sama ditampilkan paling atas.
- Perintah `php artisan presensi:admin` kini menerima `--peran=kepsek`.
- Form pengguna menampilkan kolom kelas (untuk guru) atau NIS (untuk orang tua) sesuai peran yang dipilih.

**Perubahan perilaku (perlu diperhatikan saat memperbarui dari V0.1):**
- Guru yang login dengan akun JIBAS kini hanya melihat kelas yang **ditugaskan admin**, tidak lagi
  otomatis dari kolom wali kelas di JIBAS. Guru yang belum ditugaskan melihat daftar kosong.
  Setelah menjalankan `bash deploy/update.sh`, tugaskan kelas ke setiap guru.

## 5. Yang diperbarui dari V0.2 ke V0.5

**V0.3 sampai V0.5 menambahkan:**
- **Halaman publik `/cari` untuk orang tua**: cukup memasukkan NIS (tanpa login) untuk melihat jam masuk,
  jam pulang, dan status hari ini (hadir, sudah masuk, belum scan, alpa, atau libur). Kelas ditampilkan
  bersama tingkatnya, misalnya "X Â· TJKT 1".
- **Ekspor laporan ke Excel (.xlsx) dan CSV**:
  - Per kelas, harian atau bulanan. Guru hanya bisa mengunduh kelas yang diwalikan.
  - Seluruh sekolah, harian atau bulanan, untuk kesiswaan, kepala sekolah, dan admin.
  - Tombol "Unduh Excel" dan "Unduh CSV" ada di halaman kelas dan dasbor kesiswaan.
- **Dua sumber data scan**: `cardscandata` (modul kartu, bawaan) atau `frpresence` (modul presensi JIBAS),
  dipilih lewat `.env`. Opsi `PRESENSI_FR_HANYA_AKTIF` membatasi ke baris aktif.
- **Pengaturan baru di `.env`**: `PRESENSI_SUMBER_SCAN`, `PRESENSI_FR_HANYA_AKTIF`, `JIBAS_PASSWORD_HASH`
  (md5, sha1, atau plain), `PRESENSI_TIMEZONE` (bawaan Asia/Jakarta), `JIBAS_DB_SAT`.
- **Status Alpa** setelah batas jam pulang untuk siswa yang tidak punya scan sama sekali.
- **Tampilan baru**: halaman login dirombak dengan logo dan foto gedung sekolah, font lokal
  (Plus Jakarta Sans), dan tampilan Ringkasan Sekolah yang lebih rapi di HP.
- **Pembatasan percobaan** (throttle) pada login, pencarian publik, dan ekspor.

**Batasan yang diketahui di V0.5:**
- Halaman admin untuk menugaskan kelas ke guru JIBAS (yang login dengan NIP) belum ada; sementara
  penugasannya lewat perintah di server. Guru lokal sudah bisa diatur langsung dari form pengguna.
- Guru lokal belum punya menu ganti password sendiri; password direset admin lewat menu Ubah.

## Status

Versi **beta (V0.5)**. Aplikasi sudah terpasang di server Debian 12, dan login, ekspor Excel, serta
tampilan sudah berjalan di server. Pembacaan data JIBAS asli (cara penyimpanan password, kolom tabel
siswa dan kelas, tahun ajaran) masih perlu diverifikasi, dan data scan fingerprint asli baru bisa
diuji setelah siswa dan guru mulai absen.
