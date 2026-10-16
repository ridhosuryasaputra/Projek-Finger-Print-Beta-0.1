# Projek-Finger-Print-Beta-0.1

Aplikasi web untuk menampilkan hasil presensi fingerprint siswa dari database **JIBAS**.

> **Projek ini milik grup ekskul Networking SMK Muhammadiyah 2 Metro.**
> Dikembangkan sebagai projek belajar dan untuk kebutuhan sekolah.

---

## 1. Tentang projek

Sekolah memakai JIBAS dan mesin fingerprint, tetapi hasil presensinya belum mudah dilihat oleh guru,
bagian kesiswaan, dan orang tua. Aplikasi ini menampilkan data presensi itu dalam halaman web yang
sederhana dan bisa dibuka dari HP.

Aplikasi ini **hanya menampilkan data (read-only)**. Database JIBAS tidak pernah diubah.

## 2. Rancangan aplikasi (file `presensi-jibas.zip`)

**Teknologi:** Laravel (PHP), Nginx + PHP-FPM, Debian 12 (VM / Proxmox LXC), SQLite untuk akun lokal.

**Sumber data dari JIBAS**

| Data | Database JIBAS |
|---|---|
| Scan fingerprint (NIS, jam) | `jbsumum.cardscandata` |
| Siswa dan kelas | `jbsakad` |
| Akun dan data guru | `jbsuser.login`, `jbssdm.pegawai` |

**Peran pengguna**

| Peran | Masuk dengan | Yang bisa dilihat |
|---|---|---|
| Guru | NIP + password JIBAS | Kelas yang diwalikan (harian dan bulanan) |
| Kesiswaan | Akun dari sekolah | Ringkasan seluruh sekolah, semua kelas, cari siswa, riwayat siswa |
| Admin | Akun dari sekolah | Semua halaman kesiswaan + kelola akun + buat akun orang tua per kelas |
| Orang tua | Akun dari sekolah | Hanya presensi anaknya sendiri |

**Aturan presensi**
- Masuk = scan pertama hari itu, pulang = scan terakhir.
- Terlambat jika scan pertama melewati jam masuk (bisa diatur).
- Hari sekolah = tanggal yang punya minimal satu scan siswa.

**Keamanan**
- Koneksi ke JIBAS memakai user MySQL khusus yang hanya boleh `SELECT`.
- Port database JIBAS tidak dibuka ke internet; yang dibuka hanya web (80/443) dengan HTTPS.
- Orang tua hanya bisa membuka siswa yang terhubung ke akunnya (dicek di server).
- Password akun lokal disimpan dengan bcrypt.

**Isi folder**

    app/        kode aplikasi (controller, service, middleware)
    config/     pengaturan presensi (pemetaan tabel JIBAS, jam masuk)
    resources/  tampilan halaman
    routes/     daftar alamat halaman
    database/   migrasi akun lokal dan data contoh untuk uji coba
    deploy/     skrip pemasangan dan pembaruan server

**Pemasangan singkat:** salin folder ke server Debian 12, lalu jalankan `bash deploy/install-lxc.sh`.
Untuk memperbarui versi lama: `bash deploy/update.sh`.

## 3. Yang diperbarui dari V ke V0.1

**V0.1** hanya punya peran **guru**.

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

## Status

Versi **beta**. Peran guru sudah diuji di VM. Peran kesiswaan, admin, dan orang tua masih dalam tahap
uji coba, dan pembacaan data JIBAS asli (password, kolom tabel siswa) masih perlu diverifikasi.
Data scan fingerprint asli belum ada karena siswa dan guru belum mulai absen.
