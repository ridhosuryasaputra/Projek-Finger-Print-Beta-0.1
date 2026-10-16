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

**File terbaru:** `presensi-jibas V.02.zip` (V0.2). Versi sebelumnya: `presensi jibas V.01.zip`.

**Teknologi:** Laravel (PHP), Nginx + PHP-FPM, Debian 12 (VM / Proxmox LXC), SQLite untuk akun lokal.

**Sumber data dari JIBAS**

| Data | Database JIBAS |
|---|---|
| Scan fingerprint (NIS, jam) | `jbsumum.cardscandata` |
| Siswa dan kelas | `jbsakad` |
| Akun dan data guru | `jbsuser.login`, `jbssdm.pegawai` |

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
| Guru (wali kelas) | NIP + password JIBAS, atau akun lokal dari admin | Hanya kelas yang ditugaskan admin (harian dan bulanan) |
| Kesiswaan | Akun dari sekolah | Ringkasan seluruh sekolah, semua kelas, cari siswa, riwayat siswa |
| Kepala Sekolah | Akun dari sekolah | Sama dengan kesiswaan, untuk memantau |
| Admin | Akun dari sekolah | Semua halaman kesiswaan + kelola akun + tugaskan kelas guru + buat akun orang tua per kelas |
| Orang tua | Akun dari sekolah | Hanya presensi anaknya sendiri |

**Aturan presensi**
- Masuk = scan pertama hari itu, pulang = scan terakhir.
- Terlambat jika scan pertama melewati jam masuk (bisa diatur).
- Hari sekolah = tanggal yang punya minimal satu scan siswa.

**Keamanan**
- Koneksi ke JIBAS memakai user MySQL khusus yang hanya boleh `SELECT`.
- Port database JIBAS tidak dibuka ke internet; yang dibuka hanya web (80/443) dengan HTTPS.
- Orang tua hanya bisa membuka siswa yang terhubung ke akunnya, dan guru hanya kelas yang ditugaskan (dicek di server).
- Password akun lokal disimpan dengan bcrypt.

**Isi folder**

    app/        kode aplikasi (controller, service, middleware)
    config/     pengaturan presensi (pemetaan tabel JIBAS, jam masuk)
    resources/  tampilan halaman
    routes/     daftar alamat halaman
    database/   migrasi akun lokal dan data contoh untuk uji coba
    deploy/     skrip pemasangan dan pembaruan server

**Pemasangan singkat:** salin folder ke server Debian 12, lalu jalankan `bash deploy/install-lxc.sh`.
Untuk memperbarui versi lama: `bash deploy/update.sh` (migrasi database dijalankan otomatis,
pengaturan `.env` tidak ditimpa).

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

**Batasan yang diketahui di V0.2:**
- Halaman admin untuk menugaskan kelas ke guru JIBAS (yang login dengan NIP) belum ada; sementara
  penugasannya lewat perintah di server. Guru lokal sudah bisa diatur langsung dari form pengguna.
- Guru lokal belum punya menu ganti password sendiri; password direset admin lewat menu Ubah.

## Status

Versi **beta (V0.2)**. Peran guru sudah diuji di VM. Peran kesiswaan, kepala sekolah, admin, orang tua,
dan guru lokal masih dalam tahap uji coba, dan pembacaan data JIBAS asli (password, kolom tabel
siswa dan kelas, tahun ajaran) masih perlu diverifikasi. Data scan fingerprint asli belum ada karena
siswa dan guru belum mulai absen.
