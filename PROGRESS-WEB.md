# Progress — Web/Infra (Fase 3): XAMPP, MariaDB, certbot, HTTPS

Branch: `WEB` · Fase 3 (B3) + bagian HTTPS/vhost peran A

Checklist di bawah ditandai berdasarkan bukti nyata dari sesi pengerjaan.
Legenda: `[x]` selesai & terverifikasi · `[~]` sebagian / sedang berjalan · `[ ]` belum.

## Fase 3 — XAMPP + MariaDB (B3)

- [~] **Instal XAMPP di VPS (`/opt/lampp`)**
  - [x] Port 80/443/3306/21 bebas (tidak ada apache2/nginx bentrok)
  - [x] Installer `xampp-linux-x64-8.2.12-0-installer.run` berhasil di-`scp` ke VPS `/tmp`
  - [ ] Jalankan installer (`--mode text`) → `/opt/lampp` ter-install — belum ada bukti
  - [ ] Verifikasi `httpd -v` dan `php -v`
- [ ] **Jalankan XAMPP pertama kali**: `lampp start`, `status`, `stopftp` (matikan ProFTPD)
- [ ] **Service systemd `xampp.service`** supaya otomatis saat boot (`active (exited)` normal)
- [ ] **Amankan**: `lampp security`, MariaDB bind `127.0.0.1`, phpMyAdmin `Require local`
  - [ ] password MariaDB root → password manager bersama (diserahkan ke C utk Roundcube)
- [ ] **Konfigurasi dasar Apache**: buat folder, `Include etc/extra/httpd-kelompok.conf`

## Fase 3/4 — HTTPS & Virtual Host (peran A)

- [ ] **Certbot + perpanjangan otomatis** (deploy hook `lampp reloadapache`)
- [ ] **`httpd-kelompok.conf`**: virtual host `www`, `admin`, `webmail`
- [ ] **Redirect HTTP → HTTPS**
- [ ] Target M1: gembok hijau di browser

---

## Catatan status (dari sesi pengerjaan terakhir)

- XAMPP **belum terpasang**: `/opt/lampp` tidak ada, `xampp.service` belum ada.
- Port 80/443/3306/21 **bebas** (tidak ada apache2/nginx) — aman untuk install.
- Installer sudah **berhasil ditransfer** ke VPS (`/tmp`) via `scp` (sempat lambat ~30 KB/s
  karena Tailscale lewat relay DERP Singapura; akhirnya selesai).
- Langkah berikutnya: jalankan installer di VPS, nyalakan pertama kali, buat `xampp.service`.
- File `httpd-kelompok.conf` perlu dibuat (boleh kosong dulu: `touch`) sebelum di-`Include`,
  agar Apache tidak menolak start.

> ⚠️ Jangan commit kunci privat / password (MariaDB root, dll.). Lihat `.gitignore`.
> Perbarui centang di atas saat langkah terverifikasi, dan catat di `LOG.md`.
