# Panduan Tugas Anggota B dan C (Versi Mudah Dipahami)

Turunan dari `Proyek-Private-PKI-Roadmap-dan-Panduan.md` (v1.2). Isinya sama, tetapi dipecah per tugas, dengan penjelasan "ini untuk apa", "hasil yang benar", dan "kalau error".

| | Anggota B | Anggota C |
|---|---|---|
| Peran | DNS & Infrastruktur | Mail & Dokumentasi |
| Tugas teknis | Bind9, XAMPP + MariaDB, Fail2ban, DNSSEC, DoT, CAA | Postfix, Dovecot, Roundcube + database, diagram |
| Tugas non-teknis | Bantu cek-semua dan backup | Laporan, slide, naskah demo, matriks pengujian |

> **Catatan waktu:** per 25 November 2026, tenggat 29 November tinggal 4 hari. Kalau sebagian tugas sudah selesai, lewati bagian itu. Kalau belum, ikuti **jadwal darurat di bagian 5**. Urutan prioritasnya: Must dulu, Should kalau sempat, Could dilewati.

---

## 0. Cara Memakai Dokumen Ini

1. Kerjakan **berurutan dari atas ke bawah**. Jangan lompat, karena tiap langkah bergantung pada langkah sebelumnya.
2. Salin perintah **satu blok demi satu blok**, jangan semuanya sekaligus. Cek hasilnya sebelum lanjut.
3. Ganti tulisan berikut dengan milik kalian:
   - `<IP_VPS>` → alamat IP VPS kalian
   - `<IP_ANGGOTA_1>`, `<IP_ANGGOTA_2>`, `<IP_ANGGOTA_3>` → IP internet tiap anggota (cek di `ifconfig.me`)
   - `kelompok.internal` → nama domain pilihan kelompok (kalau diganti, ganti **di semua tempat**)
4. Setiap tugas punya tiga hal:
   - **Untuk apa:** alasan langkah itu ada.
   - **Hasil yang benar:** cara memastikan berhasil.
   - **Kalau error:** pertolongan pertama.
5. Sebelum mengubah file penting, buat salinan: `sudo cp file file.bak`. Setelah tugas selesai, ambil **screenshot** untuk laporan.
6. **Satu pemilik per layanan.** B hanya mengubah DNS, XAMPP, Fail2ban. C hanya mengubah Postfix, Dovecot, Roundcube. Kalau perlu mengubah milik orang lain, minta lewat chat.
7. Password apa pun (XAMPP, database, SSH) **disimpan di password manager bersama**, bukan di chat grup.

### Konsep dalam 1 menit (supaya paham yang dikerjakan)

| Istilah | Penjelasan sederhana |
|---|---|
| Domain `kelompok.internal` | Alamat "rumah" layanan kita, hanya berlaku di jaringan kelompok |
| Bind9 | Buku telepon: mengubah nama (`www.kelompok.internal`) menjadi alamat IP |
| step-ca (milik A) | "Kantor catatan sipil" yang menerbitkan KTP digital (sertifikat) |
| Sertifikat | KTP digital layanan, bukti bahwa layanan itu asli dan komunikasinya terenkripsi |
| XAMPP | Satu paket berisi web server (Apache), database (MariaDB), dan PHP |
| Postfix | Petugas pos yang **mengirim** email |
| Dovecot | Kotak surat yang **menyimpan dan membukakan** email |
| Roundcube | Tampilan email di browser (seperti Gmail versi sendiri) |
| Fail2ban | Satpam yang memblokir orang yang berkali-kali salah password |

### Serah terima antar anggota (siapa butuh apa dari siapa)

| Dari | Ke | Apa yang diserahkan | Dibutuhkan untuk |
|---|---|---|---|
| B | Semua | Bind9 jalan, nama `ca`, `www`, `mail`, dst sudah bisa di-resolve | Penerbitan sertifikat dan semua layanan |
| B | A | XAMPP jalan dan aman (Fase 3.2–3.5) | A memasang vhost HTTPS dan mTLS |
| B | C | Password `root` MariaDB (lewat password manager) | C membuat database Roundcube |
| A | C | Sertifikat `mail.kelompok.internal` (A atau C yang menjalankan certbot) | TLS Postfix dan Dovecot |
| A | B | step-ca dan certbot sudah bisa menerbitkan sertifikat | B meminta sertifikat `ns1` untuk DoT |
| C | Semua | Draf laporan dan template screenshot | Semua anggota menulis bagiannya |

---

# BAGIAN B: DNS & Infrastruktur

## Ringkasan tugas B

| No | Tugas | Untuk apa | Prioritas | Perkiraan waktu |
|---|---|---|---|---|
| B1 | Baseline: waktu (chrony) dan firewall (UFW) | Sertifikat butuh jam yang benar, firewall menutup port yang tak perlu | Must | 20 menit |
| B2 | Bind9: zona `kelompok.internal` | Semua layanan memakai nama, bukan IP | Must | 60 menit |
| B3 | XAMPP + MariaDB (instal, service, keamanan) | Web server dan database untuk A dan C | Must | 60–90 menit |
| B4 | Fail2ban dan hardening | Melindungi SSH dan layanan dari tebak password | Must | 30 menit |
| B5 | DNSSEC | Tanda tangan digital pada DNS agar tidak bisa dipalsukan | Should | 60 menit |
| B6 | CAA dan DoT | Batasi CA yang sah, enkripsi query DNS | Should | 45 menit |
| B7 | Unbound (opsional) | Layanan server baru tambahan | Could | 30 menit |
| B8 | Bantu integrasi: `cek-semua.sh` dan backup | Memastikan semua jalan setelah reboot | Must | 30 menit |

---

## B1. Baseline: waktu dan firewall

**Untuk apa:** Sertifikat punya tanggal berlaku. Kalau jam server salah, sertifikat dianggap "belum berlaku" atau "kedaluwarsa". Firewall membuat hanya port yang dibutuhkan yang terbuka.

**Langkah 1: update dan pasang alat bantu**
```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install curl wget git jq unzip ca-certificates dnsutils knot-dnsutils swaks net-tools chrony
```
Artinya: memperbarui sistem lalu memasang alat uji (`dig`, `kdig`, `swaks`, `netstat`) dan chrony.

**Langkah 2: nyalakan sinkron waktu**
```bash
sudo systemctl enable --now chrony
chronyc tracking
timedatectl
```
**Hasil yang benar:** `chronyc tracking` menampilkan `Leap status : Normal`.

**Langkah 3: atur firewall (urutan ini penting)**
```bash
sudo ufw allow 22/tcp     # SSH, WAJIB duluan supaya tidak terkunci
sudo ufw allow 53         # DNS
sudo ufw allow 80/tcp     # web / validasi sertifikat
sudo ufw allow 443/tcp    # HTTPS
sudo ufw allow 853/tcp    # DNS over TLS
sudo ufw allow 9000/tcp   # step-ca
sudo ufw allow 25/tcp     # email
sudo ufw allow 587/tcp    # email kirim
sudo ufw allow 993/tcp    # email baca (IMAPS)
sudo ufw enable
sudo ufw status numbered
```
Jangan buka port **3306** (database) dan **21** (FTP bawaan XAMPP).

**Kalau error:** terkunci dari SSH → masuk lewat konsol/VNC di panel VPS, jalankan `sudo ufw allow 22/tcp`.

**Bukti untuk laporan:** screenshot `chronyc tracking` dan `ufw status`.

---

## B2. Bind9: DNS lokal

**Untuk apa:** Semua layanan dipanggil lewat nama. step-ca juga harus bisa me-resolve `www.kelompok.internal` untuk memvalidasi sertifikat. Jadi **Bind9 harus selesai paling awal**, karena orang lain menunggu ini.

**Langkah 1: instal**
```bash
sudo apt -y install bind9 bind9-utils bind9-dnsutils
sudo systemctl enable --now named
```

**Langkah 2: atur opsi global** (ganti seluruh isi file)
```bash
sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bak
sudo tee /etc/bind/named.conf.options <<'EOF'
options {
    directory "/var/cache/bind";

    listen-on { any; };
    listen-on-v6 { any; };

    // Siapa saja boleh menanyakan zona kita,
    // tetapi hanya IP tertentu yang boleh minta Bind9 mencari domain internet.
    allow-query { any; };
    recursion yes;
    allow-recursion   { localhost; <IP_VPS>; <IP_ANGGOTA_1>; <IP_ANGGOTA_2>; <IP_ANGGOTA_3>; };
    allow-query-cache { localhost; <IP_VPS>; <IP_ANGGOTA_1>; <IP_ANGGOTA_2>; <IP_ANGGOTA_3>; };

    forwarders { 1.1.1.1; 8.8.8.8; };
    dnssec-validation auto;
};
EOF
```
Artinya: bagian `allow-recursion` mencegah VPS menjadi "DNS terbuka" yang bisa disalahgunakan orang luar. Tambahkan juga IP laptop/HP yang dipakai untuk demo.

**Langkah 3: daftarkan zona**
```bash
sudo tee -a /etc/bind/named.conf.local <<'EOF'

zone "kelompok.internal" {
    type primary;
    file "/var/lib/bind/db.kelompok.internal";
};
EOF
```

**Langkah 4: buat file zona** (isi daftar nama dan alamatnya)
```bash
sudo tee /var/lib/bind/db.kelompok.internal <<'EOF'
$TTL 3600
@   IN  SOA ns1.kelompok.internal. admin.kelompok.internal. (
            2026111701 ; serial (naikkan setiap kali file ini diubah)
            3600       ; refresh
            900        ; retry
            604800     ; expire
            300 )      ; negative TTL

@        IN  NS   ns1.kelompok.internal.
@        IN  MX   10 mail.kelompok.internal.

ns1      IN  A    <IP_VPS>
ca       IN  A    <IP_VPS>
www      IN  A    <IP_VPS>
admin    IN  A    <IP_VPS>
mail     IN  A    <IP_VPS>
webmail  IN  A    <IP_VPS>
EOF
```
Penting: **setiap mengubah file ini, naikkan angka serial** (misalnya `2026111701` → `2026111702`).

**Langkah 5: cek sintaks, lalu jalankan**
```bash
sudo named-checkconf
sudo named-checkzone kelompok.internal /var/lib/bind/db.kelompok.internal
sudo systemctl restart named
sudo systemctl status named --no-pager
```
**Hasil yang benar:** `named-checkzone` mengakhiri dengan `OK`, dan status `active (running)`.

**Langkah 6: jadikan Bind9 ini resolver VPS sendiri**
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/kelompok.conf <<'EOF'
[Resolve]
DNS=127.0.0.1
Domains=kelompok.internal
EOF
sudo systemctl restart systemd-resolved
```

**Langkah 7: uji**
```bash
dig @127.0.0.1 www.kelompok.internal +short     # harus muncul <IP_VPS>
dig www.kelompok.internal +short                # lewat resolver sistem
dig @127.0.0.1 kelompok.internal MX +short      # harus: 10 mail.kelompok.internal.
dig @127.0.0.1 google.com +short                # internet masih bisa di-resolve
```
**Kalau error:**
- `named-checkzone` gagal → biasanya titik di akhir nama (`mail.kelompok.internal.`) atau tanda kurung terlewat.
- `dig` tidak ada jawaban → `sudo systemctl status named` dan `sudo journalctl -u named -n 30`.
- Setelah selesai, kabari A dan C di chat: **"DNS sudah jalan"**.

**Bukti untuk laporan:** screenshot hasil keempat `dig`, dan isi file zona.

---

## B3. XAMPP + MariaDB

**Untuk apa:** XAMPP menyediakan Apache (web server untuk HTTPS dan mTLS milik A) dan MariaDB (database untuk Roundcube milik C). XAMPP bukan paket `apt`, jadi harus kita buatkan service sendiri agar otomatis jalan setelah reboot.

**Langkah 1: bersihkan dan siapkan**
```bash
sudo apt -y purge apache2 nginx 2>/dev/null || true
sudo ss -tlnp | grep -E ':(80|443|3306) ' || echo "port bebas"
```
**Hasil yang benar:** tertulis `port bebas`.

**Langkah 2: unduh dan instal XAMPP**
```bash
cd /tmp
wget -O xampp-installer.run \
  "https://sourceforge.net/projects/xampp/files/XAMPP%20Linux/8.2.12/xampp-linux-x64-8.2.12-0-installer.run/download"
chmod +x xampp-installer.run
sudo ./xampp-installer.run
```
Ikuti wizard (Next sampai selesai). Opsi tanpa wizard: tambahkan `--mode unattended`. Kalau unduhan gagal, ambil tautan versi terbaru dari `apachefriends.org` (halaman Linux) dan sesuaikan nama file.

**Langkah 3: coba jalankan**
```bash
sudo /opt/lampp/lampp start
sudo /opt/lampp/lampp status
curl -I http://localhost
sudo /opt/lampp/lampp stopftp      # FTP bawaan tidak dipakai
```
**Hasil yang benar:** status menunjukkan Apache dan MySQL `running`, `curl` memberi kode `200` atau `302`.

**Langkah 4: buat service supaya otomatis jalan saat boot**
```bash
sudo tee /etc/systemd/system/xampp.service <<'EOF'
[Unit]
Description=XAMPP (Apache + MariaDB)
After=network-online.target named.service
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/opt/lampp/lampp startapache
ExecStart=/opt/lampp/lampp startmysql
ExecStop=/opt/lampp/lampp stopapache
ExecStop=/opt/lampp/lampp stopmysql

[Install]
WantedBy=multi-user.target
EOF
sudo /opt/lampp/lampp stop
sudo systemctl daemon-reload
sudo systemctl enable --now xampp
systemctl status xampp --no-pager
sudo /opt/lampp/lampp status
```
**Hasil yang benar:** status `active (exited)` itu **normal** untuk layanan jenis ini. Yang penting `lampp status` menunjukkan Apache dan MySQL running.

**Langkah 5: amankan (wajib)**
```bash
sudo /opt/lampp/lampp security
```
Wizard akan menanyakan password untuk MariaDB `root`, phpMyAdmin, dan lainnya. Isi semua dengan password kuat dan **simpan di password manager bersama**.

Lalu batasi MariaDB hanya untuk localhost:
```bash
sudo cp /opt/lampp/etc/my.cnf /opt/lampp/etc/my.cnf.bak
sudo sed -i '/^\[mysqld\]/a bind-address = 127.0.0.1' /opt/lampp/etc/my.cnf
sudo /opt/lampp/lampp restartmysql
sudo ss -tlnp | grep 3306
```
**Hasil yang benar:** alamat yang tampil `127.0.0.1:3306` (bukan `0.0.0.0:3306`).

Batasi phpMyAdmin: buka `/opt/lampp/etc/extra/httpd-xampp.conf`, cari blok `<Directory "/opt/lampp/phpmyadmin">`, lalu ubah baris `Require ...` menjadi `Require local`. Setelah itu `sudo /opt/lampp/lampp restartapache`.

Cek bahwa login tanpa password gagal:
```bash
/opt/lampp/bin/mysql -u root -e "SELECT 1"      # harus: Access denied
```

**Langkah 6: siapkan konfigurasi dasar Apache untuk proyek**
```bash
/opt/lampp/bin/httpd -M | grep -E "ssl|rewrite|headers|include|socache_shmcb"
```
Semua lima modul harus muncul. Kalau ada yang hilang, buka `/opt/lampp/etc/httpd.conf` dan hapus tanda `#` di depan baris `LoadModule` modul itu.

```bash
sudo mkdir -p /opt/lampp/etc/ssl /opt/lampp/acme /opt/lampp/htdocs/www /opt/lampp/htdocs/admin
echo "<h1>www.kelompok.internal - HTTPS dari CA sendiri</h1>" | sudo tee /opt/lampp/htdocs/www/index.html
echo "<h1>Panel Admin (mTLS)</h1>" | sudo tee /opt/lampp/htdocs/admin/index.html

sudo cp /opt/lampp/etc/httpd.conf /opt/lampp/etc/httpd.conf.bak
sudo sed -i 's|^Include etc/extra/httpd-ssl.conf|#Include etc/extra/httpd-ssl.conf|' /opt/lampp/etc/httpd.conf
echo 'Include etc/extra/httpd-kelompok.conf' | sudo tee -a /opt/lampp/etc/httpd.conf
grep -n "httpd-ssl\|httpd-kelompok" /opt/lampp/etc/httpd.conf
```
Artinya: menyiapkan folder, lalu memberi tahu Apache untuk memuat file konfigurasi proyek `httpd-kelompok.conf`. **Isi file itu dibuat oleh A** (Fase 3.6), jadi sampai A membuatnya, Apache mungkin menolak start. Kalau begitu, buat file kosong dulu: `sudo touch /opt/lampp/etc/extra/httpd-kelompok.conf`.

**Serah terima:** kabari A dan C: **"XAMPP sudah jalan dan aman"**, dan berikan password MariaDB `root` ke C lewat password manager.

**Kalau error:**
- XAMPP gagal start karena port 80 dipakai → `sudo ss -tlnp` lalu cari `:80`, matikan program lain.
- MariaDB gagal start → lihat file `*.err` di `/opt/lampp/var/mysql/`, atau kembalikan `my.cnf.bak`.
- Setelah reboot XAMPP tidak jalan → `systemctl status xampp`, `sudo systemctl enable xampp`.

**Bukti untuk laporan:** screenshot `lampp status`, `systemctl status xampp`, `ss -tlnp` (3306 di 127.0.0.1), dan login root tanpa password yang ditolak.

---

## B4. Fail2ban dan hardening

**Untuk apa:** Memblokir otomatis IP yang berkali-kali gagal login (SSH, email, web). Ini juga bahan demo.

```bash
sudo apt -y install fail2ban
sudo tee /etc/fail2ban/jail.local <<'EOF'
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
backend  = systemd
ignoreip = 127.0.0.1/8 <IP_ANGGOTA_1> <IP_ANGGOTA_2> <IP_ANGGOTA_3>

[sshd]
enabled = true

[postfix]
enabled = true

[dovecot]
enabled = true

[apache-auth]
enabled = true
backend = auto
logpath = /opt/lampp/logs/error_log
EOF
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```
**Hasil yang benar:** daftar jail muncul dan `sshd` aktif.

**Penting:** masukkan IP semua anggota ke `ignoreip` **sebelum** menguji, supaya kalian tidak memblokir diri sendiri.

**Uji (opsional, untuk demo):** dari perangkat yang IP-nya **tidak** ada di `ignoreip`, coba SSH dengan password salah 5 kali, lalu cek `sudo fail2ban-client status sshd`. Untuk membuka blokir: `sudo fail2ban-client set sshd unbanip <IP>`.

**Hardening cepat:**
- Matikan login password SSH **hanya setelah** semua anggota berhasil login pakai SSH key.
- Pastikan password database dan XAMPP sudah diset (B3).

**Kalau error:** jail `postfix` atau `dovecot` error karena tidak menemukan log → `sudo apt -y install rsyslog`, lalu pada jail tersebut isi `backend = auto` dan `logpath = /var/log/mail.log`.

**Bukti untuk laporan:** screenshot `fail2ban-client status` dan IP yang terblokir.

---

## B5. DNSSEC (Should)

**Untuk apa:** Setiap jawaban DNS diberi tanda tangan digital, sehingga resolver bisa memastikan jawabannya asli. Kerjakan **setelah** B2, B3, dan ACME milik A berjalan.

> **Wajib sebelum mulai:** buat **snapshot VPS** di panel penyedia dan cadangkan konfigurasi:
> ```bash
> sudo cp -r /etc/bind ~/bind-backup && sudo cp -r /var/lib/bind ~/bind-lib-backup
> ```
> Salah konfigurasi DNSSEC bisa membuat semua resolusi nama gagal.

**Langkah 1: aktifkan penandatanganan.** Ubah blok zona di `/etc/bind/named.conf.local` menjadi:
```
zone "kelompok.internal" {
    type primary;
    file "/var/lib/bind/db.kelompok.internal";
    dnssec-policy default;
    inline-signing yes;
};
```
```bash
sudo named-checkconf
sudo rndc reconfig
sudo rndc dnssec -status kelompok.internal
ls /var/cache/bind/K*kelompok.internal*
```
**Hasil yang benar:** file kunci `K...kelompok.internal...` muncul.

**Langkah 2: cek tanda tangan sudah ada**
```bash
dig @127.0.0.1 www.kelompok.internal A +dnssec
```
**Hasil yang benar:** ada baris `RRSIG`.

**Langkah 3: ambil "sidik jari" kunci (DS) untuk trust anchor**
```bash
sudo dnssec-dsfromkey -2 /var/cache/bind/K*kelompok.internal*.key
```
Contoh keluaran: `kelompok.internal. IN DS 12345 13 2 ABCDEF...`. Karena `.internal` tidak ada di DNS publik, kita harus memberi tahu resolver secara manual. Tambahkan di `/etc/bind/named.conf.options` (di luar blok `options`):
```
trust-anchors {
    kelompok.internal. static-ds 12345 13 2 "ABCDEF....";
};
```
Ganti angka dan hash dengan hasil `dnssec-dsfromkey` kalian. Lalu:
```bash
sudo named-checkconf && sudo systemctl restart named
```

**Langkah 4: verifikasi dengan `delv`**
```bash
cat > /tmp/trust.key <<'EOF'
trust-anchors {
    kelompok.internal. static-ds 12345 13 2 "ABCDEF....";
};
EOF
delv @127.0.0.1 -a /tmp/trust.key www.kelompok.internal A
```
**Hasil yang benar:** ada tulisan `; fully validated`.

**Kalau error:**
- Semua `dig` tiba-tiba gagal → hapus sementara blok `trust-anchors` dan `dnssec-policy`, restart `named`; kalau parah, kembalikan snapshot.
- Setiap mengubah file zona sesudahnya: naikkan serial lalu `sudo rndc reload kelompok.internal`.

**Bukti untuk laporan:** screenshot `delv` (fully validated) dan `rndc dnssec -status`.

---

## B6. CAA dan DoT (Should)

### CAA: pernyataan "hanya CA ini yang sah"
Tambahkan dua baris di file zona (naikkan serial):
```
@   IN  CAA 0 issue "ca.kelompok.internal"
@   IN  CAA 0 iodef "mailto:admin@kelompok.internal"
```
```bash
sudo rndc reload kelompok.internal
dig @127.0.0.1 kelompok.internal CAA +short
```
**Hasil yang benar:** kedua record CAA tampil. Catatan untuk laporan: CAA berfungsi sebagai kebijakan terdokumentasi, tidak semua klien ACME menegakkannya otomatis.

### DoT: DNS yang dienkripsi (port 853)
**Syarat:** step-ca dan certbot milik A sudah berfungsi. Minta sertifikat khusus `ns1`:

```bash
sudo mkdir -p /etc/bind/certs && sudo chown root:bind /etc/bind/certs && sudo chmod 750 /etc/bind/certs

sudo tee /usr/local/bin/deploy-ns1.sh <<'EOF'
#!/bin/bash
install -o root -g bind -m 640 /etc/letsencrypt/live/ns1.kelompok.internal/fullchain.pem /etc/bind/certs/ns1.crt
install -o root -g bind -m 640 /etc/letsencrypt/live/ns1.kelompok.internal/privkey.pem   /etc/bind/certs/ns1.key
systemctl reload named
EOF
sudo chmod +x /usr/local/bin/deploy-ns1.sh

sudo certbot certonly --webroot -w /opt/lampp/acme \
  --server https://ca.kelompok.internal:9000/acme/acme/directory \
  -d ns1.kelompok.internal --register-unsafely-without-email --agree-tos --no-eff-email \
  --deploy-hook /usr/local/bin/deploy-ns1.sh
sudo ls -l /etc/bind/certs
```
Artinya: skrip `deploy-ns1.sh` menyalin sertifikat ke folder yang bisa dibaca Bind9, dan dijalankan otomatis setiap sertifikat diperpanjang.

Aktifkan DoT di `/etc/bind/named.conf.options`: tambahkan blok `tls` di luar `options`, dan satu baris `listen-on` di dalam `options`:
```
tls local-tls {
    cert-file "/etc/bind/certs/ns1.crt";
    key-file  "/etc/bind/certs/ns1.key";
};

options {
    ...
    listen-on port 853 tls local-tls { any; };
    ...
};
```
```bash
sudo named-checkconf && sudo systemctl restart named
sudo ss -tlnp | grep 853
kdig -d @ns1.kelompok.internal +tls +tls-ca=/etc/step-ca/certs/root_ca.crt \
     +tls-hostname=ns1.kelompok.internal www.kelompok.internal
```
**Hasil yang benar:** port 853 mendengarkan, dan `kdig` mengembalikan jawaban lewat TLS.

**Kalau error:** AppArmor memblokir folder sertifikat → `sudo journalctl -k | grep -i apparmor`. Nama host tidak cocok → pastikan memakai `+tls-hostname=ns1.kelompok.internal`.

**Bukti untuk laporan:** screenshot `dig CAA`, `ss -tlnp` (853), dan keluaran `kdig +tls`.

---

## B7. Unbound (Could, opsional)

Hanya kalau semua tugas Must dan Should sudah selesai. Langkahnya ada di **Fase 8B** pada file roadmap utama. Kalau ada kendala (misalnya bentrok dengan `systemd-resolved`), lewati.

---

## B8. Bantu integrasi: `cek-semua.sh` dan backup

**Untuk apa:** Satu perintah untuk memastikan semua layanan hidup, dan cadangan kalau ada yang rusak.

Skrip pemeriksaan ada di **Fase 9.3** pada file roadmap utama. Setelah dipasang:
```bash
sudo cek-semua.sh
```
Pastikan semua layanan `active`. Lalu **uji reboot**:
```bash
sudo reboot
```
Setelah VPS hidup lagi, login dan jalankan `sudo cek-semua.sh`. Semua layanan (termasuk `xampp`) harus aktif sendiri.

Backup konfigurasi:
```bash
mkdir -p ~/backup && cd ~/backup
sudo tar czf konfigurasi-$(date +%F).tgz /etc/bind /opt/lampp/etc /etc/fail2ban/jail.local /etc/systemd/system/xampp.service
```
Jangan menyertakan folder rahasia (`/etc/step-ca/secrets`) pada arsip yang dibagikan.

---

## Checklist akhir B

- [ ] B1: chrony normal, UFW aktif, port 3306 dan 21 tidak dibuka
- [ ] B2: `dig www.kelompok.internal +short` memberi IP VPS, rekursi dibatasi
- [ ] B3: XAMPP otomatis jalan setelah reboot, MariaDB hanya `127.0.0.1`, password sudah diset
- [ ] B4: Fail2ban aktif, IP anggota di `ignoreip`
- [ ] B5: `delv` menampilkan `fully validated` (jika dikerjakan)
- [ ] B6: CAA tampil, `kdig +tls` berhasil (jika dikerjakan)
- [ ] B8: `cek-semua.sh` hijau setelah reboot, backup dibuat
- [ ] Semua screenshot sudah disimpan dengan nama jelas (contoh: `B2-dig-www.png`)
- [ ] Bagian laporan B ditulis: Bab IV (Bind9, XAMPP dan MariaDB, Fail2ban, DNSSEC, DoT, CAA)

---

# BAGIAN C: Mail & Dokumentasi

## Ringkasan tugas C

| No | Tugas | Untuk apa | Prioritas | Perkiraan waktu |
|---|---|---|---|---|
| C1 | Kerangka laporan dan folder bukti | Dokumentasi rapi sejak awal, bukan menumpuk di akhir | Must | 30 menit |
| C2 | Postfix + Dovecot dasar | Email antar-user bisa jalan | Must | 45 menit |
| C3 | TLS untuk Postfix + Dovecot | Email terenkripsi dengan sertifikat dari CA sendiri | Must | 45 menit |
| C4 | Uji kirim-terima email | Bukti email dan TLS bekerja | Must | 20 menit |
| C5 | Roundcube + database MariaDB | Email lewat browser | Should | 75 menit |
| C6 | Diagram, slide, naskah demo | Bahan presentasi | Must | 90 menit |
| C7 | Matriks pengujian dan laporan | Dasar penilaian laporan | Must | berjalan |

---

## C1. Kerangka laporan dan folder bukti

**Untuk apa:** Laporan adalah salah satu kriteria nilai. Kalau screenshot dikumpulkan sambil bekerja, laporan hampir selesai dengan sendirinya.

1. Buat folder bersama (Google Drive atau Git) dengan struktur:
   ```
   proyek-pki/
   ├── laporan/        (draf Bab I–VI)
   ├── screenshot/     (nama: F2-A-stepca-health.png, B2-dig-www.png, C3-openssl-993.png)
   ├── diagram/
   ├── slide/
   └── LOG.md          (siapa, kapan, apa yang dikerjakan)
   ```
2. Buat kerangka laporan (lihat bagian 10 file roadmap utama): Bab I Pendahuluan, II Landasan Teori, III Perancangan, IV Implementasi, V Pengujian, VI Penutup, Lampiran.
3. Bagikan aturan nama screenshot ke A dan B: `Fase-Peran-Isi.png`.
4. Tulis dulu **Bab I dan II** (tidak bergantung pada server): latar belakang, tujuan, teori PKI, ACME, mTLS, DNSSEC, DoT.

**Hasil yang benar:** folder dan kerangka sudah ada, semua anggota tahu cara menamai bukti.

---

## C2. Postfix + Dovecot dasar (belum pakai TLS)

**Untuk apa:** Memastikan email dasar berjalan dulu. Kalau langsung dengan TLS lalu error, sulit mencari penyebabnya. Prasyarat: DNS (B2) sudah jalan karena ada record `mail` dan `MX`.

**Langkah 1: instal dan buat user**
```bash
sudo apt -y install postfix dovecot-imapd dovecot-pop3d
```
Saat ditanya Postfix: pilih **Internet Site**, lalu *System mail name* isi `kelompok.internal`.
```bash
sudo adduser alice
sudo adduser bob
```
Artinya: `alice` dan `bob` adalah akun Linux sekaligus akun email. Catat passwordnya di password manager.

**Langkah 2: atur Postfix dasar**
```bash
sudo postconf -e "myhostname = mail.kelompok.internal"
sudo postconf -e "mydomain = kelompok.internal"
sudo postconf -e "myorigin = \$mydomain"
sudo postconf -e "mydestination = \$myhostname, localhost.\$mydomain, localhost, \$mydomain"
sudo postconf -e "home_mailbox = Maildir/"
sudo systemctl restart postfix
```
Artinya: email disimpan per pengguna di folder `Maildir` di home masing-masing.

**Langkah 3: atur Dovecot dasar**

Edit tiga file ini (buka dengan `sudo nano nama-file`):

1. `/etc/dovecot/conf.d/10-mail.conf` → pastikan baris ini ada:
   ```
   mail_location = maildir:~/Maildir
   ```
2. `/etc/dovecot/conf.d/10-auth.conf` → pastikan:
   ```
   disable_plaintext_auth = yes
   auth_mechanisms = plain login
   ```
3. `/etc/dovecot/conf.d/10-master.conf` → ubah blok `service auth` menjadi:
   ```
   service auth {
     unix_listener /var/spool/postfix/private/auth {
       mode = 0660
       user = postfix
       group = postfix
     }
   }
   ```
   Artinya: Postfix meminjam sistem login Dovecot (SASL) untuk memeriksa password pengirim.

```bash
sudo systemctl restart dovecot
sudo systemctl status postfix dovecot --no-pager
```
**Hasil yang benar:** keduanya `active (running)`.

**Kalau error:** `sudo postfix check` dan `sudo dovecot -n` menunjukkan kesalahan sintaks konfigurasi.

---

## C3. TLS untuk Postfix + Dovecot

**Untuk apa:** Email tidak dikirim polos, tetapi dienkripsi memakai sertifikat dari CA kelompok. Prasyarat: **step-ca dan certbot milik A berfungsi** dan Apache XAMPP aktif di port 80 (dipakai untuk validasi).

**Langkah 1: minta sertifikat `mail`**
```bash
sudo certbot certonly --webroot -w /opt/lampp/acme \
  --server https://ca.kelompok.internal:9000/acme/acme/directory \
  -d mail.kelompok.internal --register-unsafely-without-email --agree-tos --no-eff-email \
  --deploy-hook "systemctl reload postfix dovecot"
```
**Hasil yang benar:** muncul `Successfully received certificate` dan folder `/etc/letsencrypt/live/mail.kelompok.internal/` terisi. Kalau gagal, minta A mengecek step-ca dan certbot, lalu kabari B kalau masalahnya DNS.

**Langkah 2: aktifkan TLS di Dovecot.** Edit `/etc/dovecot/conf.d/10-ssl.conf`:
```
ssl = required
ssl_cert = </etc/letsencrypt/live/mail.kelompok.internal/fullchain.pem
ssl_key  = </etc/letsencrypt/live/mail.kelompok.internal/privkey.pem
ssl_min_protocol = TLSv1.2
```
(Perhatikan tanda `<` sebelum path. Itu bagian dari sintaks Dovecot.)

**Langkah 3: aktifkan TLS dan login di Postfix**
```bash
sudo postconf -e "smtpd_tls_cert_file = /etc/letsencrypt/live/mail.kelompok.internal/fullchain.pem"
sudo postconf -e "smtpd_tls_key_file = /etc/letsencrypt/live/mail.kelompok.internal/privkey.pem"
sudo postconf -e "smtpd_tls_security_level = may"
sudo postconf -e "smtpd_tls_auth_only = yes"
sudo postconf -e "smtp_tls_security_level = may"
sudo postconf -e "smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt"
sudo postconf -e "smtpd_sasl_type = dovecot"
sudo postconf -e "smtpd_sasl_path = private/auth"
sudo postconf -e "smtpd_sasl_auth_enable = yes"
sudo postconf -e "smtpd_recipient_restrictions = permit_sasl_authenticated, permit_mynetworks, reject_unauth_destination"
```
Baris terakhir **jangan dihapus**: itu mencegah server menjadi "open relay" yang bisa dipakai pengirim spam.

**Langkah 4: buka port 587 (submission) yang wajib TLS dan login**
```bash
sudo postconf -M submission/inet="submission inet n - y - - smtpd"
sudo postconf -P "submission/inet/syslog_name=postfix/submission"
sudo postconf -P "submission/inet/smtpd_tls_security_level=encrypt"
sudo postconf -P "submission/inet/smtpd_sasl_auth_enable=yes"
sudo systemctl restart postfix dovecot
```
**Kalau error:**
- Postfix tidak mau start → `sudo postfix check` dan `sudo journalctl -u postfix -n 30`.
- Dovecot tidak mau start → `sudo dovecot -n` dan `sudo journalctl -u dovecot -n 30`.
- Login ditolak → pastikan file socket `/var/spool/postfix/private/auth` ada (restart Dovecot dulu, baru Postfix).

---

## C4. Uji kirim-terima email

**Uji 1: sertifikat dipercaya**
```bash
openssl s_client -starttls smtp -connect mail.kelompok.internal:587 -CAfile /etc/step-ca/certs/root_ca.crt </dev/null | grep -E "Verify return code|subject|issuer"
openssl s_client -connect mail.kelompok.internal:993 -CAfile /etc/step-ca/certs/root_ca.crt </dev/null | grep "Verify return code"
```
**Hasil yang benar:** `Verify return code: 0 (ok)` pada keduanya.

**Uji 2: kirim email dari alice ke bob**
```bash
swaks --to bob@kelompok.internal --from alice@kelompok.internal \
      --server mail.kelompok.internal --port 587 -tls \
      --auth LOGIN --auth-user alice --auth-password '<password alice>'
sudo ls /home/bob/Maildir/new/
```
**Hasil yang benar:** swaks berakhir tanpa error dan ada file baru di `Maildir/new/` milik bob. Buka salah satu file (`sudo cat`) dan cari baris `Received:` yang menyebut `TLSv1.3`. Itu bukti email lewat TLS.

**Catatan:** pengiriman ke luar internet (port 25) sering diblokir penyedia VPS. Cukup tunjukkan pengiriman internal alice → bob, dan jelaskan di laporan.

**Bukti untuk laporan:** screenshot kedua `openssl` (Verify ok), output `swaks`, dan isi `Maildir/new/`.

---

## C5. Roundcube + database MariaDB

**Untuk apa:** Pengguna bisa membaca dan mengirim email lewat browser, dan datanya disimpan di MariaDB (milik XAMPP dari B). Prasyarat: XAMPP jalan (B3), Dovecot dan Postfix ber-TLS (C3), vhost `webmail` sudah diizinkan oleh A (sertifikat `webmail` sudah terbit di Fase 3.7).

**Langkah 1: cek PHP dan MariaDB**
```bash
/opt/lampp/bin/php -m | grep -i -E "mbstring|intl|xml|dom|curl|openssl|zip|fileinfo|ctype|json|iconv|session|pdo_mysql|mysqli"
sudo /opt/lampp/lampp status
```
Kalau ada ekstensi yang hilang (terutama `intl` dan `mbstring`), buka `/opt/lampp/etc/php.ini`, hapus tanda `;` pada baris `extension=...` yang sesuai, lalu `sudo /opt/lampp/lampp restartapache`.

**Langkah 2: buat database dan user khusus** (minta password `root` MariaDB dari B)
```bash
sudo /opt/lampp/bin/mysql -u root -p <<'SQL'
CREATE DATABASE roundcube CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'roundcube'@'localhost' IDENTIFIED BY 'GANTI_PASSWORD_KUAT';
GRANT ALL PRIVILEGES ON roundcube.* TO 'roundcube'@'localhost';
FLUSH PRIVILEGES;
SQL
```
Artinya: aplikasi tidak memakai akun `root` (terlalu berkuasa), melainkan akun `roundcube` yang hanya boleh mengurus satu database.

**Langkah 3: unduh Roundcube**
```bash
cd /tmp
RC_VER=1.6.11     # cek versi stabil terbaru di roundcube.net/download
wget https://github.com/roundcube/roundcubemail/releases/download/${RC_VER}/roundcubemail-${RC_VER}-complete.tar.gz
sudo tar xzf roundcubemail-${RC_VER}-complete.tar.gz -C /opt/lampp/htdocs/
sudo mv /opt/lampp/htdocs/roundcubemail-${RC_VER} /opt/lampp/htdocs/roundcube
```

**Langkah 4: isi tabel database**
```bash
sudo /opt/lampp/bin/mysql -u roundcube -p roundcube < /opt/lampp/htdocs/roundcube/SQL/mysql.initial.sql
sudo /opt/lampp/bin/mysql -u roundcube -p roundcube -e "SHOW TABLES;"
```
**Hasil yang benar:** muncul daftar tabel (`users`, `session`, dan lainnya).

**Langkah 5: konfigurasi koneksi** (ganti `GANTI_PASSWORD_KUAT` dengan password database dari langkah 2)
```bash
DESKEY=$(openssl rand -base64 24 | cut -c1-24)
sudo tee /opt/lampp/htdocs/roundcube/config/config.inc.php <<EOF
<?php
\$config['db_dsnw'] = 'mysql://roundcube:GANTI_PASSWORD_KUAT@localhost/roundcube';
\$config['imap_host'] = 'ssl://mail.kelompok.internal:993';
\$config['smtp_host'] = 'tls://mail.kelompok.internal:587';
\$config['smtp_user'] = '%u';
\$config['smtp_pass'] = '%p';
\$config['product_name'] = 'Webmail Kelompok';
\$config['support_url'] = '';
\$config['des_key'] = '$DESKEY';
\$config['plugins'] = [];
\$config['imap_conn_options'] = ['ssl' => ['verify_peer' => true, 'verify_peer_name' => true, 'cafile' => '/opt/lampp/etc/ssl/root_ca.crt']];
\$config['smtp_conn_options'] = ['ssl' => ['verify_peer' => true, 'verify_peer_name' => true, 'cafile' => '/opt/lampp/etc/ssl/root_ca.crt']];
EOF
sudo chown root:daemon /opt/lampp/htdocs/roundcube/config/config.inc.php
sudo chmod 640 /opt/lampp/htdocs/roundcube/config/config.inc.php
sudo chown -R daemon:daemon /opt/lampp/htdocs/roundcube/temp /opt/lampp/htdocs/roundcube/logs
sudo rm -rf /opt/lampp/htdocs/roundcube/installer
```
Artinya: PHP XAMPP memakai daftar CA sendiri, jadi kita tunjukkan lokasi Root CA kelompok (`root_ca.crt`) agar koneksi ke Dovecot dan Postfix diverifikasi dengan benar. Kalau file `/opt/lampp/etc/ssl/root_ca.crt` belum ada, minta A (Fase 4.1) atau salin sendiri: `sudo cp /etc/step-ca/certs/root_ca.crt /opt/lampp/etc/ssl/root_ca.crt`.

**Langkah 6: tambahkan virtual host `webmail`** (kalau belum ada dari A)
```bash
sudo tee -a /opt/lampp/etc/extra/httpd-kelompok.conf <<'EOF'

<VirtualHost *:443>
    ServerName webmail.kelompok.internal
    DocumentRoot "/opt/lampp/htdocs/roundcube"

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/www.kelompok.internal/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/www.kelompok.internal/privkey.pem
    SSLProtocol -all +TLSv1.2 +TLSv1.3

    <Directory "/opt/lampp/htdocs/roundcube">
        Options +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    <LocationMatch "^/(config|temp|logs|SQL|bin|installer)/">
        Require all denied
    </LocationMatch>
</VirtualHost>
EOF
sudo /opt/lampp/bin/httpd -t && sudo /opt/lampp/lampp reloadapache
```
Konfirmasi dulu ke A sebelum mengedit `httpd-kelompok.conf`, karena file ini milik A dan dipakai bersama.

**Langkah 7: uji**
1. Buka `https://webmail.kelompok.internal` di browser (Root CA sudah dipasang di laptop).
2. Login sebagai `alice` dengan password akun Linux-nya, kirim email ke `bob`.
3. Login sebagai `bob` dan pastikan email diterima.
4. Cek database terisi:
   ```bash
   sudo /opt/lampp/bin/mysql -u roundcube -p roundcube -e "SELECT user_id, username FROM users;"
   ```

**Hasil yang benar:** gembok hijau, email terkirim dan diterima, dan tabel `users` memuat `alice` dan `bob`.

**Kalau error:**
| Pesan | Penyebab umum | Perbaikan |
|---|---|---|
| Database connection failed | Password/user salah, MariaDB mati | Cek `db_dsnw`, uji `SHOW TABLES` pakai user roundcube |
| Connection to storage server failed | Root CA tidak dikenali PHP, Dovecot mati | Cek `cafile`, `systemctl status dovecot`, `dig mail.kelompok.internal` |
| Halaman kosong / error 500 | Ekstensi PHP kurang, izin `temp` dan `logs` | Lihat `/opt/lampp/htdocs/roundcube/logs/errors.log` dan `/opt/lampp/logs/error_log` |

**Bukti untuk laporan:** screenshot login webmail, email terkirim/diterima, dan `SHOW TABLES`/`SELECT` dari database.

---

## C6. Diagram, slide, dan naskah demo

**Untuk apa:** Presentasi dan demo adalah kriteria nilai tersendiri.

1. **Diagram:** semua diagram sudah tersedia dalam format Mermaid di bagian 3.5 file roadmap utama (arsitektur, alur implementasi, ACME, mTLS, email, DNS). Salin kodenya ke `mermaid.live`, lalu ekspor sebagai PNG atau SVG untuk laporan dan slide.
2. **Slide** (10–12 lembar), urutan:
   1. Judul dan anggota
   2. Masalah: kenapa layanan internal butuh CA sendiri
   3. Konsep PKI: Root, Intermediate, sertifikat, ACME, mTLS
   4. Arsitektur dan daftar aplikasi (lama vs baru)
   5. Alur penerbitan sertifikat (ACME)
   6. Peran Bind9: DNSSEC, DoT, CAA
   7. Keamanan berlapis: mTLS, pencabutan, Fail2ban, UFW
   8. Hasil pengujian
   9. Demo langsung
   10. Kendala dan solusi
   11. Kesimpulan dan pengembangan
   12. Tanya-jawab
3. **Naskah demo** (±10 menit), 9 langkah dari bagian 9 file roadmap utama. Tulis siapa yang bicara dan perintah/klik yang dilakukan di tiap langkah.
4. **Cadangan demo:** rekam video demo penuh dan siapkan file `hosts` di laptop penguji, supaya demo tetap jalan kalau DNS bermasalah.

---

## C7. Matriks pengujian dan laporan

1. Buka tabel **matriks pengujian** di bagian 11 file roadmap utama (T01–T24). Salin ke spreadsheet atau dokumen kelompok.
2. Setiap kali ada fitur selesai, minta anggota terkait menjalankan tes yang bersangkutan, lalu isi kolom **Status** dan **Bukti** (nama file screenshot).
3. Lengkapi laporan bab demi bab:
   | Bab | Isi | Kapan |
   |---|---|---|
   | I, II | Pendahuluan, teori | Mulai sekarang |
   | III | Perancangan: diagram, tabel domain dan port, jadwal | Setelah diagram diekspor |
   | IV | Implementasi: per fase dengan perintah, hasil, screenshot | Berjalan, menyusul fitur selesai |
   | V | Pengujian: matriks T01–T24 dan analisis | Setelah fitur beku |
   | VI | Kesimpulan dan saran | Terakhir |
4. Ketik di bagian Lampiran: daftar file konfigurasi, tautan repo, dan kontribusi tiap anggota.
5. **Peer review:** minta A dan B membaca bagian mereka di laporan sebelum difinalkan.

---

## Checklist akhir C

- [ ] C1: folder bukti dan kerangka laporan dibuat, aturan penamaan screenshot sudah dibagikan
- [ ] C2: user `alice` dan `bob` dibuat, Postfix dan Dovecot `active (running)`
- [ ] C3: sertifikat `mail` terbit, `Verify return code: 0 (ok)` untuk port 587 dan 993
- [ ] C4: email alice → bob terkirim lewat TLS dan masuk `Maildir`
- [ ] C5: Roundcube bisa login/kirim/terima, tabel database terisi
- [ ] C6: slide, naskah demo, dan video cadangan selesai
- [ ] C7: matriks pengujian terisi lengkap dengan bukti, laporan Bab I–VI selesai dan sudah di-review

---

# 5. Jadwal Darurat (25–29 November)

Pakai bagian ini kalau pekerjaan masih tertinggal. Prinsipnya: **selesaikan Must, kerjakan Should kalau sempat, buang Could.**

| Hari | Anggota B | Anggota C |
|---|---|---|
| **Rabu 25 Nov** | B2 Bind9 (kalau belum), B1 baseline, B3 XAMPP + service + keamanan. Kabari A dan C begitu DNS dan XAMPP jalan | C1 kerangka laporan, C2 Postfix + Dovecot dasar, tulis Bab I dan II |
| **Kamis 26 Nov** | B4 Fail2ban. Lanjut B5 DNSSEC kalau ACME milik A sudah jalan | C3 TLS mail (butuh sertifikat dari A), C4 uji email, mulai Bab III |
| **Jumat 27 Nov** | B6 CAA/DoT (kalau sempat), B8 `cek-semua.sh`, uji reboot, backup | C5 Roundcube + database, C6 slide, susun Bab IV |
| **Sabtu 28 Nov** | Latihan demo 1 dan 2 bersama. Siapkan perangkat uji dan file `hosts`. Backup akhir | Finalisasi laporan PDF, slide, matriks uji, rekam video demo |
| **Minggu 29 Nov** | Cek akhir dan pengumpulan | Cek akhir dan pengumpulan |

**Aturan potong (dibuat paling lambat Kamis malam):**
1. Lewati **Unbound** (B7).
2. Kalau DNSSEC atau DoT belum jalan sampai Jumat siang, **lewati** dan jelaskan sebagai pengembangan lanjutan di laporan.
3. Kalau Roundcube bermasalah, tetap tunjukkan email lewat `swaks` dan Dovecot (C4). Ini sudah cukup memenuhi bagian Mail.
4. **Jangan** memotong: Bind9, XAMPP, Postfix + Dovecot dengan TLS, Fail2ban, laporan, dan demo.

**Sinkron harian (10 menit, jam yang sama tiap hari):** apa yang selesai, apa yang jadi hambatan, apa yang dibutuhkan dari orang lain.

---

# 6. Pertolongan Pertama (Error yang Paling Sering Muncul)

## Untuk B

| Gejala | Coba ini |
|---|---|
| `named-checkzone` gagal | Cek titik di akhir nama (`mail.kelompok.internal.`) dan tanda kurung pada SOA |
| `dig` tidak menjawab | `sudo systemctl status named`, `sudo journalctl -u named -n 30`, pastikan `ufw` mengizinkan port 53 |
| Setelah DNSSEC semua `dig` gagal | Hapus sementara `trust-anchors` dan `dnssec-policy`, restart `named`, atau kembalikan snapshot |
| XAMPP gagal start (port 80 dipakai) | `sudo ss -tlnp` cari `:80`, matikan layanan lain |
| MariaDB tidak mau start | Lihat berkas `*.err` di `/opt/lampp/var/mysql/`, kembalikan `my.cnf.bak` |
| Setelah reboot XAMPP mati | `sudo systemctl enable xampp`, `journalctl -u xampp -n 50` |
| Terblokir Fail2ban | Masuk lewat konsol VPS, `sudo fail2ban-client set sshd unbanip <IP>` |
| Terkunci dari SSH karena UFW | Konsol/VNC di panel VPS, `sudo ufw allow 22/tcp` |

## Untuk C

| Gejala | Coba ini |
|---|---|
| certbot gagal untuk `mail` | Pastikan DNS jalan (`dig mail.kelompok.internal`), Apache XAMPP aktif di port 80, dan minta A mengecek step-ca |
| Postfix tidak mau start | `sudo postfix check`, `sudo journalctl -u postfix -n 30` |
| Dovecot tidak mau start | `sudo dovecot -n`, `sudo journalctl -u dovecot -n 30` |
| Login SMTP ditolak | Pastikan `/var/spool/postfix/private/auth` ada, restart Dovecot lalu Postfix |
| `Verify return code` bukan 0 | Pastikan memakai `-CAfile /etc/step-ca/certs/root_ca.crt` dan nama host `mail.kelompok.internal` |
| Roundcube: database connection failed | Cek password di `config.inc.php`, jalankan `SHOW TABLES` memakai user `roundcube` |
| Roundcube: storage server failed | Cek `cafile`, `systemctl status dovecot`, `dig mail.kelompok.internal` |
| Port 25 ke luar tidak jalan | Wajar, diblokir penyedia VPS. Uji internal saja (alice → bob) |

**Kapan harus minta bantuan:** kalau satu masalah sudah memakan lebih dari 30 menit, tulis di chat grup: *layanan apa, perintah yang dijalankan, dan pesan error lengkap*. Lampirkan hasil `journalctl -u <layanan> -n 30`.
