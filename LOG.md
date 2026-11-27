# LOG Pengerjaan — Private PKI + Full HTTPS Internal

Catatan progress sesuai NFR-05: setiap perubahan konfigurasi penting dicatat di sini.
Lingkungan: VPS Ubuntu Server 24.04, user `a2-8`, IP kampus `10.33.109.85`, IP Tailscale `100.105.214.44`.
Domain: `kelompok.internal`. CA: step-ca (`https://ca.kelompok.internal:9000`, health `ok`).

---

## Ringkasan status per fase

| Fase | Nama | Pemilik | Status |
|---|---|---|---|
| 0 | Baseline VPS, chrony, UFW | Semua | ✅ selesai |
| 1 | Bind9 dasar | B | ✅ selesai |
| 2 | step-ca (Root + Intermediate + ACME) | A | ✅ selesai (CA health `ok`) |
| 3 | XAMPP + MariaDB, certbot, Apache HTTPS | A/B | ✅ selesai (sertifikat www/admin/webmail terbit) |
| 4 | mTLS + revocation | A | ✅ vhost admin mTLS aktif (Plan B serial siap) |
| 5 | DNSSEC, DoT, CAA | B | ✅ DNSSEC aktif (zona ter-sign, DS tersedia) |
| 6 | Mail TLS (Postfix + Dovecot) | C | ✅ active (TLS mail/ns1 terbit) |
| 7 | Roundcube + MariaDB | C | ✅ webmail HTTPS jalan, DB terhubung |
| 8 | Fail2ban + hardening | B | ✅ Fail2ban 4 jail; ⬜ uji T19; ⬜ hardening |
| 8B | Unbound (resolver validasi DNSSEC) | B | ✅ selesai (flag `ad` terbukti) |

---

## Catatan harian

| Tanggal | Oleh | Fase | Perubahan | Hasil / verifikasi |
|---|---|---|---|---|
| 2026-11-25 | — | setup | Inisialisasi repo Git, `.gitignore`, struktur folder progress | repo lokal siap, commit pertama (`30319f7`), push ke `github.com/Slameteka/tamu-wajib-lapor` via SSH-over-443 |
| 2026-11-27 | C | F7 | Finalisasi Roundcube 1.6.17 di `/opt/lampp/htdocs/roundcube` | DB `roundcube` + user khusus, 17 tabel terisi, `config.inc.php` (640), installer dihapus |
| 2026-11-27 | A/C | F3→F7 | Tambah vhost `webmail.kelompok.internal` (`*:443`) di `httpd-kelompok.conf` | `httpd -t` OK, reload; `curl` → `HTTP 200`, `Set-Cookie: roundcube_sessid` |
| 2026-11-27 | A/C | F7 | Verifikasi TLS webmail | `openssl s_client` → `Verify return code: 0 (ok)`; SAN cert memuat www/admin/webmail; issuer Intermediate CA |
| 2026-11-27 | C | F7 | Resolusi klien (laptop) via Tailscale | `/etc/hosts` → `100.105.214.44 ...kelompok.internal`; `getent` OK; `curl -kI` → `HTTP 200` |
| 2026-11-27 | B | F8 | Install + konfigurasi Fail2ban (`/etc/fail2ban/jail.local`) | 4 jail aktif: `sshd, postfix, dovecot, apache-auth` |
| 2026-11-27 | B | F8 | Perbaiki `ignoreip` (ganti placeholder) | `ignoreip = 127.0.0.1/8 ::1 100.123.28.43 10.33.109.85`; reload OK |
| 2026-11-27 | B | F8 | Verifikasi sumber log jail | postfix/dovecot → journald; apache-auth → `/opt/lampp/logs/error_log` |
| 2026-11-27 | B | F8B | Install Unbound, config `/etc/unbound/unbound.conf.d/kelompok.conf` | `unbound-checkconf: no errors`; listen `127.0.0.1:5353` |
| 2026-11-27 | B | F8B | Uji validasi DNSSEC via Unbound | `dig @127.0.0.1 -p 5353 ... +dnssec` → **`flags: ... ad`** + RRSIG (fully validated) |

---

## Detail konfigurasi (bukti laporan)

### Fase 7 — Roundcube + MariaDB (C)
- **Vhost webmail** ditambahkan ke `/opt/lampp/etc/extra/httpd-kelompok.conf`:
  - `ServerName webmail.kelompok.internal`, DocumentRoot `/opt/lampp/htdocs/roundcube`
  - Sertifikat reuse dari cert `www` (SAN mencakup `webmail`): `/etc/letsencrypt/live/www.kelompok.internal/{fullchain,privkey}.pem`
  - `SSLProtocol -all +TLSv1.2 +TLSv1.3`, HSTS aktif
  - Folder sensitif diblokir: `<LocationMatch "^/(config|temp|logs|SQL|bin|installer)/"> Require all denied`
- **Sertifikat** (dari `certbot certificates`): cert `www.kelompok.internal` mencakup domain `admin`, `webmail`, `www`; expiry 2026-12-04.
- **Verifikasi TLS:** `openssl s_client -connect webmail.kelompok.internal:443` → `Verify return code: 0 (ok)`, TLSv1.3, issuer = Intermediate CA.
- **Akses klien:** laptop pop-os via Tailscale, `/etc/hosts` memetakan nama `*.kelompok.internal` → `100.105.214.44`.
- **Sisa uji (untuk bukti T18/T24):** login `alice` → kirim ke `bob` → `bob` terima; `SELECT user_id, username FROM users;` setelah login pertama.

### Fase 8 — Fail2ban (B)
- File: `/etc/fail2ban/jail.local`
  ```
  [DEFAULT]
  bantime  = 1h
  findtime = 10m
  maxretry = 5
  backend  = systemd
  ignoreip = 127.0.0.1/8 ::1 100.123.28.43 10.33.109.85

  [sshd]       enabled = true
  [postfix]    enabled = true
  [dovecot]    enabled = true
  [apache-auth] enabled = true
               backend = auto
               logpath = /opt/lampp/logs/error_log
  ```
- `fail2ban-client status` → `Number of jail: 4` → `apache-auth, dovecot, postfix, sshd`.
- Sumber log: `mail.log` tidak ada (pakai journald, normal di Ubuntu 24.04); apache error_log ada.
- **Sisa uji (T19):** brute force SSH dari IP di luar `ignoreip` → muncul di `Banned IP list`.

### Fase 8B — Unbound (B, opsional/COULD)
- File: `/etc/unbound/unbound.conf.d/kelompok.conf`
  ```
  server:
      interface: 127.0.0.1
      port: 5353
      access-control: 127.0.0.0/8 allow
      do-not-query-localhost: no
      trust-anchor: "kelompok.internal. DS 10567 13 2 188C2CA626DCE6982ADE5D781E7E31802C47605693FEC80565DD40762F37CC03"
  stub-zone:
      name: "kelompok.internal"
      stub-addr: 127.0.0.1@53
  ```
- DS asli diperoleh dari: `dig @127.0.0.1 kelompok.internal DNSKEY | dnssec-dsfromkey -f - kelompok.internal`.
- `unbound-checkconf: no errors`; `systemctl status unbound` → active (running); listen `127.0.0.1:5353`.
- **Bukti validasi:** `dig @127.0.0.1 -p 5353 www.kelompok.internal +dnssec` → `flags: qr rd ra ad` + RRSIG → DNSSEC fully validated.

---

## Matriks pengujian (status terkini)

| ID | Uji | Status | Catatan |
|---|---|---|---|
| T03 | CA health | ✅ | `{"status":"ok"}` |
| T04 | Rantai sertifikat | ✅ | `Verify return code: 0 (ok)` (webmail) |
| T05 | HTTPS valid browser | ⬜ | perlu Root CA di laptop + screenshot gembok |
| T07 | ACME issuance | ✅ | cert www/admin/webmail, mail, ns1 terbit |
| T12 | DNSSEC | ✅ | Unbound `ad` + RRSIG |
| T18 | Webmail | ⬜ | perlu login alice→bob di browser |
| T19 | Fail2ban ban | ⬜ | perlu brute force dari IP non-ignore |
| T24 | Roundcube ke DB | ⬜ | perlu `SELECT ... FROM users;` setelah login |

---

## Sisa pekerjaan (TODO)

- [ ] F7/T18: uji login `alice` → kirim ke `bob` → `bob` terima di browser (screenshot)
- [ ] F7/T24: `SELECT user_id, username FROM users;` setelah login pertama
- [ ] F8/T19: brute force SSH dari IP non-ignore → `Banned IP list` (screenshot)
- [ ] F8 hardening: `ServerTokens Prod`/`ServerSignature Off`, `/dashboard` & `/phpmyadmin` tidak publik, `chmod 600 /etc/step-ca/password.txt`
- [ ] T05: pasang Root CA di laptop penguji, screenshot gembok hijau
