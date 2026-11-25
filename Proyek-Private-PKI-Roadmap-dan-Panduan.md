# Proyek Administrasi Server: Private PKI + Full HTTPS Internal

> **Ringkasan satu kalimat:** Kelompok membangun *Certificate Authority* (CA) sendiri di Ubuntu Server 24.04, lalu semua layanan internal (web, admin panel, mail, webmail, DNS) memakai sertifikat dari CA itu secara otomatis (ACME), dilengkapi mTLS, DNSSEC, DoT, dan pencabutan sertifikat.

| Item | Isi |
|---|---|
| Anggota | 3 orang (Anggota A, B, C — isi nama di bagian 4) |
| Server | 1 VPS Ubuntu Server 24.04, RAM 8 GB, storage 40 GB, 2 core (sudah ada dan bisa diakses) |
| Tanggal mulai | Senin, 16 November 2026 |
| Tenggat | **Minggu, 29 November 2026** (asumsi "dua minggu lagi" dari 15 November; sesuaikan kalau tenggat sebenarnya beda) |
| Domain lokal | `kelompok.internal` (ganti dengan nama kelompok, misalnya `grup3.internal`) |
| DNS | Bind9 (wajib) |

> **Pembaruan v1.2 (dokumen ini):** (1) Web server dan database memakai **XAMPP** (Apache + PHP + MariaDB di `/opt/lampp`), bukan Apache/Nginx dari `apt`. (2) **MariaDB** dipakai sebagai database **Roundcube**, hanya mendengarkan `localhost` dan diamankan. (3) Ditambahkan **diagram arsitektur dan flowchart** (bagian 3.5), **langkah instal → jalankan → integrasikan** per layanan (Fase 2–7), serta **peta integrasi, urutan menjalankan, dan skrip cek** (Fase 9). (4) Halaman uji mTLS: `/whoami.shtml`. (5) Aplikasi baru kini juga mencakup XAMPP dan MariaDB.

---

## Daftar Isi

1. Tujuan dan ruang lingkup
2. Pemetaan ke kriteria penilaian dan syarat tugas
3. Arsitektur, daftar aplikasi, domain, dan port
4. Pembagian tugas 3 orang
5. Roadmap dan prioritas (Must / Should / Could)
6. Rencana pengerjaan harian (16–29 November)
7. Aturan kerja bersama di satu VPS
8. Panduan implementasi step-by-step (Fase 0–9)
9. Skenario demo dan naskah presentasi
10. Kerangka laporan
11. Matriks pengujian
12. Troubleshooting
13. Risiko dan mitigasi
14. Hal yang dikonfirmasi ke dosen/aslab
15. Checklist akhir sebelum pengumpulan
16. Referensi

---

## 1. Tujuan dan Ruang Lingkup

### 1.1 Masalah yang diselesaikan
Layanan di jaringan internal tidak bisa memakai Let's Encrypt karena domainnya tidak publik. Akibatnya muncul sertifikat *self-signed* dan peringatan "Not Secure". Solusi di dunia nyata adalah **Private PKI**: organisasi punya CA sendiri dan perangkat klien dipercayakan ke CA tersebut.

### 1.2 Tujuan proyek
1. Membangun hierarki CA: **Root CA** (kunci disimpan offline) dan **Intermediate CA** (online, menerbitkan sertifikat).
2. Menjalankan server **ACME** sendiri (step-ca) sehingga sertifikat terbit dan diperpanjang otomatis.
3. Mengamankan semua layanan internal dengan HTTPS/TLS dari CA sendiri.
4. Menerapkan **mTLS**: halaman admin hanya bisa dibuka dengan sertifikat klien.
5. Menerapkan **pencabutan sertifikat** (revocation) dan membuktikannya.
6. Memperkuat DNS dengan **DNSSEC**, **DoT**, dan record **CAA**.
7. Mendokumentasikan, menguji, dan mendemokan semuanya.

### 1.3 Di luar ruang lingkup
Sertifikat untuk domain publik, HSM/hardware token, federasi CA, dan kluster step-ca (HA).

### 1.4 Output akhir
- Server berjalan dengan semua layanan di bagian 3.
- Laporan proyek (PDF).
- Slide presentasi dan demo langsung.
- Folder konfigurasi di Git (bukti dan cadangan).

---

## 2. Pemetaan ke Kriteria Penilaian dan Syarat Tugas

### 2.1 Syarat tugas
| Syarat | Bagaimana dipenuhi |
|---|---|
| Minimal 2 aplikasi server di Ubuntu Server | Total sekitar 11–12 aplikasi (lihat bagian 3.2) |
| Bind9 wajib | Bind9 dengan zona lokal, DNSSEC, DoT, CAA, resolver |
| Boleh memakai aplikasi lama, minimal 2 baru | Baru: **step-ca, step CLI, certbot, Fail2ban, chrony, Roundcube, XAMPP/MariaDB** (jika hanya layanan jaringan yang dihitung: step-ca, Roundcube, dan MariaDB; Unbound opsional menjadi yang ke-4) |

### 2.2 Kriteria penilaian
| Kriteria | Strategi |
|---|---|
| Kompleksitas proyek | PKI dua tingkat, ACME otomatis, mTLS, revocation, DNSSEC, DoT, integrasi mail |
| Jumlah aplikasi server | Tabel bagian 3.2; kumpulkan bukti instalasi (screenshot `dpkg -l`/`systemctl status`) |
| Presentasi/demo | Skenario 7 langkah yang visual (gembok hijau, akses ditolak lalu diterima, sertifikat dicabut) |
| Laporan | Kerangka bagian 10, diagram, screenshot per fase, tabel pengujian |

---

## 3. Arsitektur, Aplikasi, Domain, dan Port

### 3.1 Arsitektur

```
 Klien (laptop / HP penguji)
   - Root CA dipasang        - DNS = <IP_VPS> (atau file hosts)
   - sertifikat klien (.p12) - browser dan klien email
          │   DNS 53/853 · HTTPS/mTLS 443 · SMTP 587 · IMAPS 993
          ▼
┌───────────────────── VPS Ubuntu 24.04 (8 GB RAM, 40 GB, 2 core) ──────────────────────┐
│  UFW (firewall) · Fail2ban · chrony (sinkron waktu)                                    │
│                                                                                        │
│  ┌── DNS ────────────────────┐    ┌── PKI ──────────────────────────────────────┐     │
│  │ Bind9  :53 / :853 (DoT)   │    │ step-ca :9000 (Root + Intermediate + ACME)  │     │
│  │ DNSSEC, CAA               │    │ certbot (klien ACME + timer perpanjangan)   │     │
│  │ Unbound :5353 (opsional)  │    └───────────────────────┬─────────────────────┘     │
│  └───────────────────────────┘                            │ sertifikat + deploy hook  │
│                                                           ▼                           │
│  ┌── XAMPP (/opt/lampp) ─────────────────────────────┐   ┌── Mail ──────────────┐     │
│  │ Apache :80 / :443                                 │   │ Postfix :25 / :587   │     │
│  │   www · admin (mTLS) · webmail                    │   │ Dovecot :993         │     │
│  │ PHP → Roundcube ──────────────────────────────────┼──►│ (IMAPS, SMTP+SASL)   │     │
│  │ MariaDB 127.0.0.1:3306  (database: roundcube)     │   └──────────────────────┘     │
│  └───────────────────────────────────────────────────┘                                │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

**Hierarki kepercayaan**

```
Root CA (kunci offline)
   └── Intermediate CA (step-ca, online)
          ├── sertifikat server: www, admin, webmail, mail, ns1 (via ACME)
          └── sertifikat klien: budi, ani (untuk mTLS)
```

### 3.2 Daftar aplikasi

**Aplikasi lama (dari praktikum)**

| No | Aplikasi | Paket | Peran |
|---|---|---|---|
| 1 | Ubuntu Server 24.04 | (OS) | Sistem dasar |
| 2 | Bind9 | `bind9 bind9-utils bind9-dnsutils` | DNS, DNSSEC, DoT, CAA |
| 3 | Web Server Apache (di dalam paket XAMPP) | XAMPP `.run` | HTTPS, mTLS, host PHP untuk Roundcube |
| 4 | Postfix | `postfix` | SMTP dengan TLS |
| 5 | Dovecot | `dovecot-imapd dovecot-pop3d` | IMAP/POP3 dengan TLS |
| 6 | Webmin (opsional) | repo resmi | Panel admin, HTTPS bisa pakai sertifikat CA sendiri |

**Aplikasi baru**

| No | Aplikasi | Instal | Peran |
|---|---|---|---|
| 7 | step-ca | `.deb` Smallstep | CA + server ACME |
| 8 | step CLI | `.deb` Smallstep | Membuat sertifikat klien, mencabut, inspeksi |
| 9 | certbot | `apt` | Klien ACME, perpanjangan otomatis |
| 10 | Roundcube | `apt` | Webmail HTTPS |
| 11 | Fail2ban | `apt` | Proteksi brute force |
| 12 | chrony | `apt` | Sinkronisasi waktu |
| 13 | **XAMPP** (Apache + PHP + MariaDB + phpMyAdmin) | installer `.run` di `/opt/lampp` | Paket web server, PHP, dan database |
| 14 | **MariaDB** (bawaan XAMPP) | ikut XAMPP | Database Roundcube, hanya `localhost` |

**Opsional (menambah layanan server):** Unbound (resolver tervalidasi DNSSEC, Fase 8B).

**Pendukung:** UFW, OpenSSL, curl, `net-tools`, `knot-dnsutils` (kdig), `swaks` (tes email), phpMyAdmin (bawaan XAMPP, dibatasi ke localhost).

**Klasifikasi jenis aplikasi (penting untuk hitungan "aplikasi server"):**

| Jenis | Aplikasi |
|---|---|
| Layanan server (daemon jaringan) | Bind9, Apache (XAMPP), Postfix, Dovecot, **step-ca**, **MariaDB**, **Unbound (opsional)** |
| Aplikasi web di atas Apache | **Roundcube** |
| Daemon pendukung sistem | **Fail2ban**, **chrony**, UFW |
| Klien/alat bantu | **certbot**, **step CLI**, OpenSSL, curl, swaks, dig/delv/kdig |
| Paket distribusi | **XAMPP** (membungkus Apache, PHP, MariaDB, phpMyAdmin) |

Dengan hitungan paling ketat (hanya layanan jaringan yang baru), yang lolos adalah **step-ca**, **Roundcube**, dan **MariaDB**, ditambah **Unbound** bila dikerjakan. Karena itu pertanyaan ke dosen di bagian 14 (nomor 1 dan 7) penting dijawab lebih awal.

### 3.3 Rencana nama domain dan alamat

Ganti `<IP_VPS>` dengan IP VPS kalian dan `kelompok.internal` dengan domain pilihan.

| FQDN | Alamat | Layanan |
|---|---|---|
| `ns1.kelompok.internal` | `<IP_VPS>` | Bind9 |
| `ca.kelompok.internal` | `<IP_VPS>` | step-ca (port 9000) |
| `www.kelompok.internal` | `<IP_VPS>` | Situs HTTPS |
| `admin.kelompok.internal` | `<IP_VPS>` | Halaman admin (mTLS) |
| `mail.kelompok.internal` | `<IP_VPS>` | Postfix + Dovecot |
| `webmail.kelompok.internal` | `<IP_VPS>` | Roundcube |

Catatan: jangan pakai `.local` (bentrok dengan mDNS). `.internal` dicadangkan untuk jaringan privat.

### 3.4 Port

| Port | Layanan | Catatan keamanan di VPS publik |
|---|---|---|
| 22/tcp | SSH | Batasi ke IP anggota bila memungkinkan |
| 53/tcp+udp | Bind9 | Rekursi hanya untuk IP tertentu (bagian Fase 1) |
| 80/tcp | Apache XAMPP (ACME HTTP-01 + redirect) | Wajib terbuka untuk validasi |
| 443/tcp | Apache HTTPS | |
| 853/tcp | DNS over TLS | |
| 9000/tcp | step-ca (ACME/API) | Cukup diakses dari VPS sendiri dan klien pengujian |
| 3306/tcp | MariaDB (XAMPP) | **Hanya localhost** (`bind-address = 127.0.0.1`). Jangan dibuka di UFW |
| 21/tcp | ProFTPD bawaan XAMPP | **Dimatikan**, tidak dipakai |
| 25, 587, 993/tcp | Mail | Banyak penyedia VPS memblokir port 25 keluar; uji email secara lokal |

### 3.5 Diagram Arsitektur dan Diagram Alur Sistem

> Diagram ditulis dengan **Mermaid** (tampil otomatis di GitHub, GitLab, VS Code dengan ekstensi Markdown Mermaid, Obsidian, dan Notion). Jika penampil kalian tidak mendukung Mermaid, salin kode ke `mermaid.live` untuk diekspor menjadi PNG/SVG untuk laporan. Versi teks arsitektur ada di bagian 3.1.

#### 3.5.1 Diagram arsitektur (komponen dan koneksi)

```mermaid
flowchart LR
    subgraph KL["Klien: laptop / HP penguji"]
        BR["Browser<br/>Root CA + sertifikat klien .p12"]
        MC["Klien email"]
        RS["Resolver klien<br/>DNS = IP VPS"]
    end

    subgraph VPS["VPS Ubuntu 24.04: 8 GB RAM, 40 GB, 2 core"]
        subgraph DNSG["DNS"]
            B9["Bind9<br/>53 dan 853 DoT<br/>DNSSEC + CAA"]
            UB["Unbound 5353<br/>opsional"]
        end
        subgraph PKIG["PKI"]
            SC["step-ca 9000<br/>Root + Intermediate + ACME"]
            CB["certbot<br/>timer perpanjangan"]
        end
        subgraph XAMPPG["XAMPP di /opt/lampp"]
            AP["Apache 80 dan 443<br/>www, admin mTLS, webmail"]
            RC["PHP + Roundcube"]
            DB[("MariaDB 127.0.0.1:3306<br/>database roundcube")]
        end
        subgraph MAILG["Mail"]
            PF["Postfix 25 dan 587"]
            DV["Dovecot 993"]
        end
        SEC["UFW + Fail2ban + chrony"]
    end

    RS -->|DNS 53 atau DoT 853| B9
    BR -->|HTTPS dan mTLS 443| AP
    MC -->|SMTP 587| PF
    MC -->|IMAPS 993| DV
    AP --> RC
    RC --> DB
    RC -->|IMAPS 993| DV
    RC -->|SMTP 587| PF
    PF -->|SASL via socket| DV
    CB -->|ACME| SC
    SC -->|validasi HTTP-01 port 80| AP
    CB -->|sertifikat + deploy hook| AP
    CB -.->|sertifikat mail| PF
    CB -.->|sertifikat mail| DV
    CB -.->|sertifikat ns1| B9
    UB -->|stub zone| B9
    SEC -.->|melindungi| AP
```

**Cara membaca:** garis tegas adalah aliran data/permintaan; garis putus-putus adalah distribusi sertifikat atau perlindungan. Semua komponen berada di satu VPS.

#### 3.5.2 Flowchart alur implementasi proyek (urutan pengerjaan)

```mermaid
flowchart TD
    S(["Mulai 16 Nov"]) --> F0["Fase 0: VPS, chrony, UFW, repo Git"]
    F0 --> F1["Fase 1: Bind9, zona, resolver lokal"]
    F0 --> F2["Fase 2: step-ca, Root offline, ACME"]
    F0 --> F6a["Fase 6A: Postfix + Dovecot tanpa TLS"]
    F1 --> F3X["Fase 3: XAMPP + MariaDB diamankan"]
    F2 --> F3C["Fase 3: trust Root, certbot, HTTPS"]
    F3X --> F3C
    F3C --> M1{"M1: gembok hijau?"}
    M1 -- Tidak --> F3C
    M1 -- Ya --> F4["Fase 4: mTLS + revocation"]
    M1 -- Ya --> F5["Fase 5: DNSSEC, DoT, CAA"]
    M1 -- Ya --> F6["Fase 6: TLS Postfix + Dovecot"]
    F6a --> F6
    F6 --> F7["Fase 7: Roundcube + MariaDB"]
    F4 --> M2{"M2: inti selesai?"}
    F5 --> M2
    F7 --> M2
    M2 -- Belum --> FX["Perbaiki, geser jadwal, potong fitur"]
    FX --> M2
    M2 -- Ya --> F8["Fase 8: Fail2ban, hardening, Unbound opsional"]
    F8 --> F9["Fase 9: integrasi, uji T01 sampai T24, laporan"]
    F9 --> M4{"Siap demo?"}
    M4 -- Belum --> F9
    M4 -- Ya --> E(["Kumpulkan 29 Nov"])
```

#### 3.5.3 Flowchart alur penerbitan sertifikat otomatis (ACME)

```mermaid
sequenceDiagram
    autonumber
    participant CB as certbot
    participant SC as step-ca
    participant DNS as Bind9
    participant AP as Apache XAMPP
    CB->>SC: Buat order sertifikat untuk www.kelompok.internal
    SC-->>CB: Challenge HTTP-01 beserta token
    CB->>AP: Tulis token ke /opt/lampp/acme/.well-known/acme-challenge/
    SC->>DNS: Resolve www.kelompok.internal
    DNS-->>SC: Alamat IP VPS
    SC->>AP: GET /.well-known/acme-challenge/token di port 80
    AP-->>SC: Isi token
    CB->>SC: Kirim CSR (finalize)
    SC-->>CB: Sertifikat ditandatangani Intermediate CA
    CB->>AP: Simpan di /etc/letsencrypt/live lalu jalankan deploy hook
    AP->>AP: lampp reloadapache memuat sertifikat baru
```

#### 3.5.4 Flowchart alur akses admin (mTLS) dan pencabutan sertifikat

```mermaid
flowchart TD
    A(["Klien membuka admin.kelompok.internal"]) --> B{"Klien mengirim sertifikat?"}
    B -- Tidak --> X1["Ditolak: SSLVerifyClient require"]
    B -- Ya --> C{"Rantai valid sampai Root CA kita?"}
    C -- Tidak --> X2["Ditolak: sertifikat tidak tepercaya"]
    C -- Ya --> D{"Sertifikat dicabut? CRL atau RewriteMap"}
    D -- Ya --> X3["403 Forbidden"]
    D -- Tidak --> OK["200 OK: whoami.shtml menampilkan identitas klien"]

    R1["Admin menjalankan step ca revoke"] --> R2["step-ca menandai sertifikat dicabut"]
    R2 --> R3{"Metode pemblokiran di Apache"}
    R3 -- "Plan A" --> R4["Ambil CRL ke crl.pem lalu reload Apache"]
    R3 -- "Plan B" --> R5["Tambah serial ke revoked_serials.txt"]
    R4 --> D
    R5 --> D
```

#### 3.5.5 Flowchart alur email dan webmail (Roundcube + MariaDB)

```mermaid
sequenceDiagram
    participant U as Alice di browser
    participant RC as Roundcube (PHP XAMPP)
    participant DB as MariaDB
    participant PF as Postfix
    participant DV as Dovecot
    participant B as Bob
    U->>RC: Login via HTTPS webmail.kelompok.internal
    RC->>DB: Simpan sesi dan pengaturan pengguna
    RC->>DV: IMAPS 993 login, TLS dari CA sendiri
    DV-->>RC: Sesi IMAP berhasil
    U->>RC: Kirim email ke bob
    RC->>PF: SMTP 587 STARTTLS + SASL
    PF->>DV: Cek login lewat socket auth
    DV-->>PF: Valid
    PF->>PF: Simpan ke Maildir milik bob
    B->>RC: Buka inbox
    RC->>DV: IMAP ambil pesan
    DV-->>RC: Pesan dari Alice
```

#### 3.5.6 Flowchart alur DNS (DNSSEC dan DoT)

```mermaid
flowchart LR
    Q["Klien: dig atau delv www.kelompok.internal"] --> BN["Bind9 menjawab A + RRSIG"]
    BN --> V{"Validasi DNSSEC dengan trust anchor?"}
    V -- Ya --> AD["delv: fully validated"]
    V -- Tidak --> RAW["Jawaban diterima tanpa validasi"]
    Q2["Klien: kdig dengan +tls"] --> DOT["Bind9 port 853 memakai sertifikat ns1"]
    DOT --> CK{"Sertifikat dipercaya Root CA?"}
    CK -- Ya --> OK2["Query terenkripsi berhasil"]
    CK -- Tidak --> ER["Koneksi gagal"]
```

#### 3.5.7 Urutan menyalakan layanan saat boot

```mermaid
flowchart LR
    A["Boot VPS"] --> B["chrony"] --> C["named (Bind9)"] --> D["step-ca"] --> E["xampp: Apache + MariaDB"] --> F["postfix + dovecot"] --> G["fail2ban"] --> H["certbot.timer"]
```

---

## 4. Pembagian Tugas 3 Orang

Penugasan memperhitungkan ketergantungan: **DNS (B) dan CA (A) harus selesai lebih awal** karena dipakai semua orang. C mulai dari mail dasar dan dokumentasi, lalu masuk ke TLS setelah CA siap.

| Peran | Nama | Fokus utama |
|---|---|---|
| **A — PKI & Web Lead** | `[isi nama]` | step-ca, Root/Intermediate, ACME, certbot, Apache HTTPS, mTLS, revocation |
| **B — DNS & Infrastruktur Lead** | `[isi nama]` | Baseline VPS, chrony, UFW, Bind9, DNSSEC, DoT, CAA, Fail2ban |
| **C — Mail & Dokumentasi Lead** | `[isi nama]` | Postfix, Dovecot, Roundcube, laporan, diagram, skrip demo, slide, matriks pengujian |

### 4.1 Rincian per orang

**Anggota A — PKI & Web**
- [ ] Instal step CLI + step-ca, inisialisasi CA, systemd service
- [ ] Pindahkan kunci Root CA ke penyimpanan offline
- [ ] Tambah provisioner ACME, atur durasi sertifikat
- [ ] Konfigurasi certbot dan perpanjangan otomatis
- [ ] Apache XAMPP: file `httpd-kelompok.conf`, virtual host `www`, `admin`, `webmail`, redirect HTTP→HTTPS
- [ ] mTLS di `admin`, penerbitan sertifikat klien (`.crt`, `.key`, `.p12`)
- [ ] Revocation (CRL/daftar serial) dan buktinya
- [ ] Peta integrasi dan skrip `cek-semua.sh` (Fase 9) bersama B
- [ ] Tulis bagian laporan: PKI, ACME, mTLS, revocation

**Anggota B — DNS & Infrastruktur**
- [ ] Verifikasi VPS, update, chrony, UFW
- [ ] Bind9: zona `kelompok.internal`, rekursi terbatas, forwarder
- [ ] Atur resolver VPS ke Bind9 lokal
- [ ] DNSSEC (`dnssec-policy`) + trust anchor + verifikasi `delv`
- [ ] DoT (port 853) + verifikasi `kdig`
- [ ] Record CAA, MX, TXT
- [ ] XAMPP: instal, service systemd, `lampp security`, MariaDB hanya localhost (Fase 3.2–3.5)
- [ ] Fail2ban (sshd, postfix, dovecot, apache)
- [ ] Tulis bagian laporan: DNS, DNSSEC, DoT, keamanan jaringan

**Anggota C — Mail & Dokumentasi**
- [ ] Kerangka laporan, template screenshot, penamaan file bukti
- [ ] Postfix + Dovecot dasar (Maildir, user `alice`, `bob`)
- [ ] TLS pada Postfix/Dovecot memakai sertifikat dari step-ca
- [ ] Submission 587 + SASL via Dovecot
- [ ] Roundcube di XAMPP (HTTPS) dengan database MariaDB
- [ ] Diagram (arsitektur, alur ACME, alur mTLS), slide, naskah demo
- [ ] Matriks pengujian (bagian 11) diisi dan dilengkapi bukti
- [ ] Tulis bagian laporan: mail, webmail, pengujian, kesimpulan

### 4.2 Tugas bersama
- Hari 1: verifikasi VPS bersama (Fase 0).
- Rapat singkat harian 10–15 menit (chat/voice): kemarin selesai apa, hari ini apa, hambatan apa.
- Latihan demo dua kali (27 dan 28 November).
- Peer review: tiap orang membaca ulang bagian laporan orang lain.

### 4.3 Siapa membantu siapa (cadangan)
A ↔ B saling cadangan untuk DNS/CA. B ↔ C saling cadangan untuk mail/DNS (MX, SPF). C membantu A saat konfigurasi Apache dan Roundcube.

---

## 5. Roadmap dan Prioritas

### 5.1 Fase

| Fase | Nama | Pemilik | Target selesai |
|---|---|---|---|
| 0 | Verifikasi VPS & baseline | Semua | Sen 16 Nov |
| 1 | Bind9 dasar | B | Sel 17 Nov |
| 2 | step-ca: Root + Intermediate + ACME | A | Sel 17 Nov |
| 3 | XAMPP + MariaDB, trust, certbot, Apache HTTPS | B (3.2–3.5), A (sisanya) | Rab 18 Nov |
| 4 | mTLS + revocation | A | Jum 20 Nov |
| 5 | DNSSEC, DoT, CAA | B | Sab 21 Nov |
| 6 | Mail TLS (Postfix + Dovecot) | C | Sab 21 Nov |
| 7 | Roundcube | C | Sen 23 Nov |
| 8 | Fail2ban + hardening | B | Sen 23 Nov |
| 9 | Integrasi, pengujian, laporan, demo | Semua | Sab 28 Nov |

### 5.2 Prioritas (aturan pemotongan bila terlambat)

| Level | Isi |
|---|---|
| **MUST** | Bind9, step-ca, certbot/ACME, XAMPP (Apache HTTPS), mTLS, Postfix + Dovecot TLS, Fail2ban, chrony, laporan, demo |
| **SHOULD** | Revocation, DNSSEC, DoT, Roundcube, CAA |
| **COULD** | OCSP, Unbound, DNS-01 via `nsupdate`, FTPS, Webmin dengan sertifikat CA, SPF/DKIM |

**Urutan pemotongan jika tertinggal:** COULD semua → Roundcube → DoT → DNSSEC. Jangan memotong MUST dan revocation (inti demo). Keputusan pemotongan dibuat paling lambat **Senin 23 November**.

### 5.3 Milestone

| Milestone | Tanggal | Kriteria lulus |
|---|---|---|
| M1: "Gembok hijau" | Rab 18 Nov | `https://www.kelompok.internal` valid di laptop penguji |
| M2: "Inti selesai" | Min 22 Nov | M1 + mTLS + revocation + DNSSEC + DoT + mail TLS |
| M3: "Fitur beku" | Rab 25 Nov | Semua layanan jalan, tidak ada fitur baru |
| M4: "Siap demo" | Sab 28 Nov | Dua latihan demo berhasil, laporan final |
| M5: Pengumpulan | Min 29 Nov | Laporan + slide + bukti terkirim |

---

## 6. Rencana Pengerjaan Harian

### Minggu 1 — Membangun inti (16–22 November)

| Hari | A (PKI & Web) | B (DNS & Infra) | C (Mail & Dok) |
|---|---|---|---|
| **Sen 16** | Fase 0 bersama. Instal step CLI + step-ca | Fase 0 bersama. Update, chrony, UFW. Mulai Bind9 | Fase 0 bersama. Kerangka laporan, template screenshot |
| **Sel 17** | `step ca init`, systemd, Root offline, provisioner ACME | Zona `kelompok.internal`, resolver lokal, tes `dig`. Lalu instal XAMPP, service systemd, `lampp security` (Fase 3.2–3.5) | Instal Postfix + Dovecot (tanpa TLS), user `alice`/`bob` |
| **Rab 18** | Trust Root di VPS, vhost, certbot, Apache HTTPS **(M1)** | Bantu A: DNS untuk ACME; snapshot VPS | Diagram arsitektur, tes kirim email lokal |
| **Kam 19** *(ringan)* | Sertifikat klien + konfigurasi mTLS | Persiapan DNSSEC (baca, backup) | Tulis Bab II (teori) |
| **Jum 20** | Revocation (CRL/serial) | **DNSSEC** + trust anchor + `delv` | Sertifikat `mail` via certbot, TLS Postfix/Dovecot |
| **Sab 21** | Perpanjangan otomatis (timer), tes renew | **DoT** + CAA + `kdig` | Submission 587, tes `swaks` dan `openssl s_client` |
| **Min 22** | **Review M2**, perbaikan, snapshot | Review M2 | Review M2, perbarui laporan |

### Minggu 2 — Pelengkap, pengujian, laporan (23–29 November)

| Hari | A | B | C |
|---|---|---|---|
| **Sen 23** | Keputusan potong/tidak. Polesan revocation | Fail2ban + hardening | Roundcube + database MariaDB |
| **Sel 24** | Uji integrasi bagian web/mTLS | Uji integrasi DNS | Isi matriks pengujian, Bab IV |
| **Rab 25** | **Fitur beku (M3)**, snapshot final | Snapshot, cadangan konfigurasi ke Git | Bab V, slide |
| **Kam 26** *(ringan)* | Perbaikan bug | Perbaikan bug | Peer review laporan |
| **Jum 27** | **Latihan demo 1** (rekam waktu) | Latihan demo 1 | Latihan demo 1, perbaiki naskah |
| **Sab 28** | **Latihan demo 2 (M4)**, siapkan perangkat uji | Siapkan cadangan (hosts file, video demo) | Finalisasi laporan PDF, slide |
| **Min 29** | Pengumpulan + cek akhir (bagian 15) | Pengumpulan | Pengumpulan |

**Cadangan waktu:** Kamis dan Minggu 22 November sengaja ringan sebagai penyangga. Kalau satu fase molor sehari, geser ke slot tersebut.

---

## 7. Aturan Kerja Bersama di Satu VPS

Tiga orang di satu server sangat rawan saling menimpa konfigurasi.

1. **Satu pemilik per layanan.** Hanya pemilik (bagian 4) yang mengubah konfigurasi layanannya. Orang lain mengusulkan lewat chat.
2. **Akun sendiri.** Buat user sudo masing-masing agar log jelas (`sudo adduser nama && sudo usermod -aG sudo nama`). Kalau pakai SSH key, masukkan kunci publik tiap orang.
3. **Repo konfigurasi di Git.** Setelah tiap perubahan penting, salin file konfigurasi ke repo (`~/config-repo/`) dan commit dengan pesan jelas. Contoh: `git commit -m "B: aktifkan DNSSEC zona kelompok.internal"`. Repo ini juga lampiran laporan. **Jangan commit kunci privat dan password.**
4. **Snapshot VPS** di panel penyedia pada: akhir Fase 3, sebelum DNSSEC, M2, M3. Perubahan DNSSEC adalah yang paling mudah merusak semuanya.
5. **Backup file sebelum diedit:** `sudo cp file file.bak.$(date +%F-%H%M)`.
6. **Catat perintah.** Simpan riwayat penting di `LOG.md` (siapa, kapan, apa). Dasar bagian implementasi laporan.
7. **Password manager/dokumen rahasia bersama** (bukan chat grup) untuk password CA. Password CA hilang berarti CA harus dibuat ulang.
8. **Screenshot sambil bekerja**, bukan di akhir. Penamaan: `F3-A-apache-https-gembok.png`.

---

## 8. Panduan Implementasi Step-by-Step

> Perintah memakai `kelompok.internal` dan `<IP_VPS>`. Ganti sesuai kondisi. Semua perintah dijalankan di VPS kecuali disebut klien. Versi paket dan beberapa opsi bisa berbeda; kalau ada opsi yang ditolak, cek `perintah --help` atau dokumentasi di bagian 16.

### Fase 0 — Verifikasi VPS dan baseline (Semua, Senin 16 Nov)

**Tujuan:** memastikan semua orang bisa masuk dan lingkungan siap.

```bash
# Info sistem
lsb_release -a            # harus Ubuntu 24.04
nproc; free -h; df -h /   # 2 core, ~8 GB, ~40 GB
ip -br a                  # catat IP, itu <IP_VPS>
curl -4 ifconfig.me       # IP publik (jika VPS publik)

# Update
sudo apt update && sudo apt -y upgrade
sudo apt -y install curl wget git jq unzip ca-certificates dnsutils knot-dnsutils swaks net-tools

# Waktu (sertifikat sangat sensitif terhadap jam)
sudo apt -y install chrony
sudo systemctl enable --now chrony
chronyc tracking          # pastikan "Leap status: Normal"
timedatectl               # zona waktu; bisa: sudo timedatectl set-timezone Asia/Jakarta
```

**Firewall (UFW).** Pastikan SSH diizinkan lebih dulu agar tidak terkunci.

```bash
sudo ufw allow 22/tcp
sudo ufw allow 53
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 853/tcp
sudo ufw allow 9000/tcp
sudo ufw allow 25/tcp
sudo ufw allow 587/tcp
sudo ufw allow 993/tcp
sudo ufw enable
sudo ufw status numbered
```

Opsional (lebih aman di VPS publik): batasi 22 dan 9000 hanya dari IP anggota, misalnya `sudo ufw allow from <IP_ANGGOTA> to any port 22 proto tcp`. Hati-hati agar tidak mengunci diri.

**Repo konfigurasi dan user:**

```bash
mkdir -p ~/config-repo && cd ~/config-repo && git init
echo "# LOG pengerjaan" > LOG.md && git add . && git commit -m "init"
```

**Verifikasi:** semua anggota bisa SSH, `chronyc tracking` normal, `ufw status` aktif. Screenshot: `F0-*`.

---

### Fase 1 — Bind9 dasar (B, Selasa 17 Nov)

**Tujuan:** DNS lokal untuk `kelompok.internal` berjalan dan dipakai oleh VPS sendiri.

**1.1 Instal**
```bash
sudo apt -y install bind9 bind9-utils bind9-dnsutils
sudo systemctl enable --now named
```

**1.2 Opsi global** — edit `/etc/bind/named.conf.options`:
```
options {
    directory "/var/cache/bind";

    listen-on { any; };
    listen-on-v6 { any; };

    // Zona kita bisa dijawab siapa saja (otoritatif),
    // tetapi rekursi dibatasi agar VPS tidak jadi open resolver.
    allow-query { any; };
    recursion yes;
    allow-recursion { localhost; <IP_VPS>; <IP_ANGGOTA_1>; <IP_ANGGOTA_2>; <IP_ANGGOTA_3>; };
    allow-query-cache { localhost; <IP_VPS>; <IP_ANGGOTA_1>; <IP_ANGGOTA_2>; <IP_ANGGOTA_3>; };

    forwarders { 1.1.1.1; 8.8.8.8; };
    dnssec-validation auto;
};
```
Tambahkan IP perangkat penguji (laptop/HP demo) ke daftar `allow-recursion` bila akan memakai VPS sebagai DNS.

**1.3 Deklarasi zona** — edit `/etc/bind/named.conf.local`:
```
zone "kelompok.internal" {
    type primary;
    file "/var/lib/bind/db.kelompok.internal";
};
```

**1.4 File zona** — buat `/var/lib/bind/db.kelompok.internal`:
```
$TTL 3600
@   IN  SOA ns1.kelompok.internal. admin.kelompok.internal. (
            2026111701 ; serial (naikkan tiap perubahan: YYYYMMDDnn)
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
```

**1.5 Validasi dan jalankan**
```bash
sudo named-checkconf
sudo named-checkzone kelompok.internal /var/lib/bind/db.kelompok.internal
sudo systemctl restart named
sudo systemctl status named --no-pager
```

**1.6 Pakai Bind9 sebagai resolver VPS.** VPS harus bisa me-resolve `ca.kelompok.internal` karena certbot dan step-ca saling memanggil lewat nama.
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/kelompok.conf <<'EOF'
[Resolve]
DNS=127.0.0.1
Domains=kelompok.internal
EOF
sudo systemctl restart systemd-resolved
resolvectl status | head -20
```

**Verifikasi**
```bash
dig @127.0.0.1 www.kelompok.internal +short     # → <IP_VPS>
dig www.kelompok.internal +short                # lewat resolver sistem
dig @127.0.0.1 kelompok.internal MX +short
dig @127.0.0.1 google.com +short                # rekursi/forwarder jalan
```
Dari **klien**: atur DNS laptop ke `<IP_VPS>` (atau tambahkan baris di file hosts sebagai cadangan, lihat Fase 3.5) lalu `nslookup www.kelompok.internal`.

Commit konfigurasi ke repo (`named.conf.options`, `named.conf.local`, zona).

---

### Fase 2 — step-ca: Root, Intermediate, ACME (A, Selasa 17 Nov)

**Tujuan:** CA berjalan sebagai service dan Root key keluar dari server.

**2.1 Instal step CLI dan step-ca.** Ambil paket `.deb` dari situs resmi Smallstep (arsitektur `amd64`; cek `dpkg --print-architecture`).
```bash
cd /tmp
wget https://dl.smallstep.com/cli/docs-cli-install/latest/step-cli_amd64.deb
wget https://dl.smallstep.com/certificates/docs-ca-install/latest/step-ca_amd64.deb
sudo dpkg -i step-cli_amd64.deb step-ca_amd64.deb
step version && step-ca version
```
Jika URL berubah, buka dokumentasi instalasi Smallstep (bagian 16) dan ikuti tautan terbaru.

**2.2 Buat user dan direktori khusus**
```bash
sudo useradd --system --home /etc/step-ca --shell /usr/sbin/nologin step
sudo mkdir -p /etc/step-ca
```

**2.3 Password CA.** Simpan di password manager bersama, jangan di chat.
```bash
sudo bash -c 'umask 077; read -s -p "Password CA: " P; echo "$P" > /etc/step-ca/password.txt; echo'
```

**2.4 Inisialisasi CA** (membuat Root, Intermediate, dan konfigurasi):
```bash
sudo -E env STEPPATH=/etc/step-ca step ca init \
  --deployment-type=standalone \
  --name="Kelompok Internal CA" \
  --dns="ca.kelompok.internal" \
  --dns="localhost" \
  --address=":9000" \
  --provisioner="admin@kelompok.internal" \
  --password-file=/etc/step-ca/password.txt \
  --provisioner-password-file=/etc/step-ca/password.txt
sudo chown -R step:step /etc/step-ca
sudo chmod 600 /etc/step-ca/password.txt
ls -R /etc/step-ca
```
Hasil penting: `certs/root_ca.crt`, `certs/intermediate_ca.crt`, `secrets/root_ca_key`, `secrets/intermediate_ca_key`, `config/ca.json`.

**2.5 Service systemd** — `/etc/systemd/system/step-ca.service`:
```
[Unit]
Description=step-ca (Private CA kelompok.internal)
After=network-online.target
Wants=network-online.target

[Service]
User=step
Group=step
Environment=STEPPATH=/etc/step-ca
ExecStart=/usr/bin/step-ca /etc/step-ca/config/ca.json --password-file=/etc/step-ca/password.txt
Restart=on-failure
RestartSec=5
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now step-ca
sudo systemctl status step-ca --no-pager
sudo journalctl -u step-ca -n 30 --no-pager
```

**2.6 Pindahkan kunci Root CA ke offline.** Ini yang dijelaskan di laporan sebagai praktik terbaik. CA online hanya butuh Intermediate.
```bash
# Salin ke laptop anggota A (jalankan dari laptop):
scp user@<IP_VPS>:/etc/step-ca/secrets/root_ca_key ./root_ca_key.BACKUP
# Cadangkan juga ke flash disk/penyimpanan terenkripsi, lalu hapus dari server:
sudo shred -u /etc/step-ca/secrets/root_ca_key
```
Jika `scp` ditolak karena izin, salin dulu ke home dengan `sudo cp`, ubah pemiliknya, lalu hapus salinannya setelah selesai. Pastikan salinan offline benar-benar ada **sebelum** `shred`.

**2.7 Tambah provisioner ACME**
```bash
sudo -u step env STEPPATH=/etc/step-ca step ca provisioner add acme --type ACME
sudo systemctl restart step-ca
```
Atur durasi sertifikat (opsional; cek `step ca provisioner update --help` untuk nama opsi pada versi kalian), misalnya default 7 hari:
```bash
sudo -u step env STEPPATH=/etc/step-ca step ca provisioner update acme --x509-default-dur=168h
sudo systemctl restart step-ca
```
Untuk demo "perpanjangan otomatis" kalian bisa memakai `--force-renewal` (Fase 3), tidak perlu menunggu masa berlaku habis.

**2.8 Catat fingerprint Root** (dipakai klien):
```bash
step certificate fingerprint /etc/step-ca/certs/root_ca.crt
step certificate inspect /etc/step-ca/certs/root_ca.crt --short
```

**Verifikasi**
```bash
curl --cacert /etc/step-ca/certs/root_ca.crt https://ca.kelompok.internal:9000/health
# → {"status":"ok"}
curl --cacert /etc/step-ca/certs/root_ca.crt https://ca.kelompok.internal:9000/acme/acme/directory | jq .
```
Screenshot: `systemctl status step-ca`, output `health`, struktur `/etc/step-ca`, tampilan Root (`inspect`).

---

### Fase 3 — XAMPP + MariaDB, trust, certbot, Apache HTTPS (B untuk 3.2–3.5 pada 17 Nov, A untuk sisanya pada 18 Nov)

**Tujuan:** XAMPP terpasang, berjalan otomatis saat boot, aman, lalu `https://www.kelompok.internal` valid dengan gembok hijau **(Milestone M1)**.

**Konsep singkat:** XAMPP memasang Apache, PHP, MariaDB, dan phpMyAdmin dalam satu folder `/opt/lampp`. Karena bukan paket `apt`, ia **tidak** dikelola systemd dan **tidak** ikut `apt upgrade`; karena itu kita membuat service systemd sendiri (3.3) dan memakai path XAMPP di semua konfigurasi.

| Hal | Lokasi di XAMPP |
|---|---|
| Folder utama | `/opt/lampp` |
| Konfigurasi Apache | `/opt/lampp/etc/httpd.conf`, tambahan di `/opt/lampp/etc/extra/` |
| Dokumen web | `/opt/lampp/htdocs/` |
| Log Apache | `/opt/lampp/logs/access_log`, `/opt/lampp/logs/error_log` |
| Konfigurasi MariaDB | `/opt/lampp/etc/my.cnf`, data di `/opt/lampp/var/mysql/` |
| Perintah kelola | `/opt/lampp/lampp start\|stop\|restart\|status`, `startapache`, `startmysql`, `reloadapache`, dll. |
| Klien/dump database | `/opt/lampp/bin/mysql`, `/opt/lampp/bin/mysqldump` |
| PHP | `/opt/lampp/bin/php`, konfigurasi `/opt/lampp/etc/php.ini` |

**3.1 Percayakan Root CA di VPS** (supaya certbot dan python memercayai step-ca):
```bash
sudo cp /etc/step-ca/certs/root_ca.crt /usr/local/share/ca-certificates/kelompok-root-ca.crt
sudo update-ca-certificates
curl https://ca.kelompok.internal:9000/health      # sekarang tanpa --cacert
```
Jika certbot nanti tetap gagal memverifikasi sertifikat CA, set `export REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt` (sementara) atau lihat bagian 12.

**3.2 Instal XAMPP (pemilik: B, Selasa 17 Nov)**

Siapkan sistem dan pastikan tidak ada web server lain yang memakai port 80/443:
```bash
sudo apt -y install net-tools            # dibutuhkan XAMPP untuk cek port
sudo apt -y purge apache2 nginx 2>/dev/null || true   # bila sebelumnya sempat terpasang
sudo ss -tlnp | grep -E ':(80|443|3306) ' || echo "port bebas"
```
Unduh installer Linux 64-bit dari halaman Linux di `apachefriends.org` (pilih PHP 8.2.x atau yang lebih baru; sesuaikan nama file). Contoh:
```bash
cd /tmp
wget -O xampp-installer.run \
  "https://sourceforge.net/projects/xampp/files/XAMPP%20Linux/8.2.12/xampp-linux-x64-8.2.12-0-installer.run/download"
chmod +x xampp-installer.run
sudo ./xampp-installer.run                 # ikuti wizard; atau tambahkan: --mode unattended
ls /opt/lampp && /opt/lampp/bin/httpd -v && /opt/lampp/bin/php -v | head -1
```
Jika unduhan gagal, ambil tautan terbaru dari situs resmi lalu salin ke VPS dengan `scp`.

**3.3 Jalankan pertama kali, lalu buat service systemd**

```bash
sudo /opt/lampp/lampp start                # menyalakan Apache, MariaDB, dan ProFTPD
sudo /opt/lampp/lampp status
curl -I http://localhost                   # → HTTP/1.1 302 atau 200 (dashboard XAMPP)
sudo /opt/lampp/lampp stopftp              # ProFTPD tidak dipakai
```

Agar XAMPP otomatis jalan saat boot (lolos uji reboot T20), buat `/etc/systemd/system/xampp.service`. Hanya Apache dan MariaDB yang dinyalakan:
```ini
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
```
```bash
sudo /opt/lampp/lampp stop                 # hentikan instance manual, serahkan ke systemd
sudo systemctl daemon-reload
sudo systemctl enable --now xampp
systemctl status xampp --no-pager          # "active (exited)" itu normal untuk Type=oneshot
sudo /opt/lampp/lampp status
```
Mulai sekarang kelola dengan `sudo systemctl restart xampp` (semua) atau `sudo /opt/lampp/lampp restartapache` / `restartmysql` (sebagian).

**3.4 Amankan XAMPP dan MariaDB (wajib, sebelum layanan dibuka ke luar)**

```bash
sudo /opt/lampp/lampp security
```
Wizard ini menanyakan password untuk akun `root` MariaDB, phpMyAdmin, dan FTP, serta proteksi halaman XAMPP. Isi semuanya, **simpan di password manager bersama**, jangan di chat.

Batasi MariaDB hanya untuk localhost dan batasi phpMyAdmin:
```bash
sudo cp /opt/lampp/etc/my.cnf /opt/lampp/etc/my.cnf.bak
sudo sed -i '/^\[mysqld\]/a bind-address = 127.0.0.1' /opt/lampp/etc/my.cnf
sudo /opt/lampp/lampp restartmysql
sudo ss -tlnp | grep 3306                  # harus 127.0.0.1:3306, bukan 0.0.0.0

# phpMyAdmin hanya dari localhost
grep -n -i 'phpmyadmin' /opt/lampp/etc/extra/httpd-xampp.conf
#   Pada blok <Directory "/opt/lampp/phpmyadmin"> ubah baris Require menjadi:  Require local
```
Cek: login ke MariaDB tanpa password harus **gagal**:
```bash
/opt/lampp/bin/mysql -u root -e "SELECT 1"     # harus: Access denied
sudo /opt/lampp/bin/mysql -u root -p -e "SHOW DATABASES;"
```
Jangan buka port 3306 dan 21 di UFW (tidak ada di daftar port).

**3.5 Konfigurasi Apache dasar proyek (B)**

Cek modul yang dibutuhkan sudah aktif:
```bash
/opt/lampp/bin/httpd -M | grep -E "ssl|rewrite|headers|include|socache_shmcb"
```
Jika ada yang hilang, buka `/opt/lampp/etc/httpd.conf` dan hapus tanda `#` pada baris `LoadModule` modul tersebut.

Siapkan folder dan file konfigurasi sendiri, lalu muat dari `httpd.conf`:
```bash
sudo mkdir -p /opt/lampp/etc/ssl /opt/lampp/acme /opt/lampp/htdocs/www /opt/lampp/htdocs/admin
echo "<h1>www.kelompok.internal — HTTPS dari CA sendiri</h1>" | sudo tee /opt/lampp/htdocs/www/index.html
echo "<h1>Panel Admin (mTLS)</h1>" | sudo tee /opt/lampp/htdocs/admin/index.html

sudo cp /opt/lampp/etc/httpd.conf /opt/lampp/etc/httpd.conf.bak
# nonaktifkan vhost SSL bawaan XAMPP (self-signed) agar tidak bentrok
sudo sed -i 's|^Include etc/extra/httpd-ssl.conf|#Include etc/extra/httpd-ssl.conf|' /opt/lampp/etc/httpd.conf
echo 'Include etc/extra/httpd-kelompok.conf' | sudo tee -a /opt/lampp/etc/httpd.conf
grep -n "httpd-ssl\|httpd-kelompok" /opt/lampp/etc/httpd.conf
```

**3.6 Virtual host port 80 (ACME + redirect), tahap awal (A, Rabu 18 Nov)**

Buat `/opt/lampp/etc/extra/httpd-kelompok.conf` dengan isi awal berikut. Vhost HTTPS **belum** ditambahkan karena sertifikatnya belum ada:
```bash
sudo tee /opt/lampp/etc/extra/httpd-kelompok.conf <<'EOF'
# ===== Konfigurasi proyek Private PKI (dimuat dari httpd.conf) =====
Listen 443
SSLSessionCache "shmcb:/opt/lampp/logs/ssl_scache(512000)"
SSLSessionCacheTimeout 300
ServerTokens Prod
ServerSignature Off

<VirtualHost *:80>
    ServerName www.kelompok.internal
    ServerAlias ns1.kelompok.internal ca.kelompok.internal admin.kelompok.internal mail.kelompok.internal webmail.kelompok.internal
    DocumentRoot "/opt/lampp/htdocs/www"

    Alias /.well-known/acme-challenge/ "/opt/lampp/acme/.well-known/acme-challenge/"
    <Directory "/opt/lampp/acme/.well-known/acme-challenge/">
        Require all granted
    </Directory>
    <Directory "/opt/lampp/htdocs/www">
        Require all granted
    </Directory>

    RewriteEngine On
    RewriteCond %{REQUEST_URI} !^/\.well-known/acme-challenge/
    RewriteRule ^/?(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]
</VirtualHost>
EOF
sudo /opt/lampp/bin/httpd -t                      # → Syntax OK
sudo /opt/lampp/lampp reloadapache
curl -I http://www.kelompok.internal              # → 301 ke https
```

**3.7 Minta sertifikat via ACME dari step-ca (A)**

Satu sertifikat untuk tiga nama web (`www`, `admin`, `webmail`):
```bash
sudo apt -y install certbot
sudo certbot certonly --webroot -w /opt/lampp/acme \
  --server https://ca.kelompok.internal:9000/acme/acme/directory \
  -d www.kelompok.internal -d admin.kelompok.internal -d webmail.kelompok.internal \
  --register-unsafely-without-email --agree-tos --no-eff-email \
  --deploy-hook "/opt/lampp/lampp reloadapache"
```
Hasil: `/etc/letsencrypt/live/www.kelompok.internal/{fullchain.pem,privkey.pem}`. (`fullchain.pem` berisi server + intermediate.) Sertifikat `mail` dan `ns1` diminta di Fase 5 dan 6, dengan `-w /opt/lampp/acme` yang sama.

**3.8 Tambah virtual host HTTPS `www`**
```bash
sudo tee -a /opt/lampp/etc/extra/httpd-kelompok.conf <<'EOF'

<VirtualHost *:443>
    ServerName www.kelompok.internal
    DocumentRoot "/opt/lampp/htdocs/www"

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/www.kelompok.internal/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/www.kelompok.internal/privkey.pem
    SSLProtocol -all +TLSv1.2 +TLSv1.3

    Header always set Strict-Transport-Security "max-age=31536000"

    <Directory "/opt/lampp/htdocs/www">
        Require all granted
    </Directory>
</VirtualHost>
EOF
sudo /opt/lampp/bin/httpd -t && sudo /opt/lampp/lampp reloadapache
```

**3.9 Siapkan klien (laptop penguji):**
1. Ambil `root_ca.crt` dari VPS (`scp`).
2. Instal Root:
   - **Windows:** klik dua kali `root_ca.crt` → *Install Certificate* → *Local Machine* → **Trusted Root Certification Authorities**.
   - **Linux:** salin ke `/usr/local/share/ca-certificates/` lalu `sudo update-ca-certificates` (Firefox: *Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import*).
   - **Android:** *Settings → Security → Encryption & credentials → Install a certificate → CA certificate*.
   - **macOS:** buka di Keychain Access, ubah *Trust* menjadi *Always Trust*.
3. DNS klien → `<IP_VPS>`, atau (cadangan demo) tambahkan di file hosts:
   ```
   <IP_VPS>  ns1.kelompok.internal ca.kelompok.internal www.kelompok.internal admin.kelompok.internal mail.kelompok.internal webmail.kelompok.internal
   ```
   (Windows: `C:\Windows\System32\drivers\etc\hosts` sebagai Administrator; Linux/macOS: `/etc/hosts`.)

**3.10 Perpanjangan otomatis**
```bash
systemctl list-timers | grep certbot       # timer bawaan paket certbot
sudo certbot renew --force-renewal         # uji; ini juga yang dipakai di demo
sudo certbot certificates
```
Tampilkan `journalctl -u certbot` atau output `certbot renew` saat demo sebagai bukti otomatisasi.

**Integrasi yang terjadi di fase ini:** Bind9 menjawab nama → step-ca memvalidasi lewat port 80 Apache XAMPP → certbot menyimpan sertifikat → *deploy hook* memanggil `lampp reloadapache` → Apache memuat sertifikat baru.

**Verifikasi M1**
```bash
curl -v https://www.kelompok.internal 2>&1 | grep -E "subject|issuer|SSL certificate verify"
openssl s_client -connect www.kelompok.internal:443 -servername www.kelompok.internal </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```
Di browser klien: gembok tanpa peringatan, rantai sertifikat Root → Intermediate → Server. Ambil snapshot VPS dan screenshot sebelum/sesudah Root dipasang di klien.

---

### Fase 4 — mTLS dan Revocation (A, 19–20 Nov)

**Tujuan:** `admin.kelompok.internal` hanya bisa dibuka dengan sertifikat klien, dan sertifikat yang dicabut ditolak.

**4.1 Rantai CA untuk verifikasi klien**
```bash
sudo bash -c 'cat /etc/step-ca/certs/intermediate_ca.crt /etc/step-ca/certs/root_ca.crt > /opt/lampp/etc/ssl/ca-chain.crt'
sudo cp /etc/step-ca/certs/root_ca.crt /opt/lampp/etc/ssl/root_ca.crt     # dipakai juga oleh Roundcube (Fase 7)
# Daftar serial yang dicabut (Plan B, lihat 4.5); baris diawali # adalah komentar
echo "# serial_hex 1" | sudo tee /opt/lampp/etc/ssl/revoked_serials.txt
```

**4.2 Tambah virtual host admin (mTLS)**
```bash
sudo tee -a /opt/lampp/etc/extra/httpd-kelompok.conf <<'EOF'

<VirtualHost *:443>
    ServerName admin.kelompok.internal
    DocumentRoot "/opt/lampp/htdocs/admin"

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/www.kelompok.internal/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/www.kelompok.internal/privkey.pem
    SSLProtocol -all +TLSv1.2 +TLSv1.3

    # mTLS: klien wajib membawa sertifikat dari CA kita
    SSLCACertificateFile /opt/lampp/etc/ssl/ca-chain.crt
    SSLVerifyClient require
    SSLVerifyDepth 2

    # Plan A (aktifkan setelah file CRL ada, lihat 4.5):
    # SSLCARevocationFile  /opt/lampp/etc/ssl/crl.pem
    # SSLCARevocationCheck leaf

    # Plan B: blokir serial yang dicabut
    RewriteEngine On
    RewriteMap revoked "txt:/opt/lampp/etc/ssl/revoked_serials.txt"
    RewriteCond "${revoked:%{SSL:SSL_CLIENT_M_SERIAL}|0}" "=1"
    RewriteRule ".*" "-" [F]

    <Directory "/opt/lampp/htdocs/admin">
        Require all granted
        Options +Includes
        AddOutputFilter INCLUDES .shtml
        SSLOptions +StdEnvVars
    </Directory>
</VirtualHost>
EOF
```
Halaman uji identitas klien (memakai SSI/`mod_include`):
```bash
sudo tee /opt/lampp/htdocs/admin/whoami.shtml <<'EOF'
<pre>Terautentikasi sebagai: <!--#echo var="SSL_CLIENT_S_DN" -->
Serial: <!--#echo var="SSL_CLIENT_M_SERIAL" -->
Verifikasi: <!--#echo var="SSL_CLIENT_VERIFY" --></pre>
EOF
sudo /opt/lampp/bin/httpd -t && sudo /opt/lampp/lampp reloadapache
```

**4.3 Terbitkan sertifikat klien** (untuk `budi` dan `ani`):
```bash
mkdir -p ~/clients && cd ~/clients
step ca certificate "budi" budi.crt budi.key \
  --ca-url https://ca.kelompok.internal:9000 \
  --root /etc/step-ca/certs/root_ca.crt \
  --provisioner admin@kelompok.internal \
  --not-after 720h
step certificate p12 budi.p12 budi.crt budi.key \
  --ca /etc/step-ca/certs/intermediate_ca.crt
```
(Password provisioner diminta; `.p12` diminta password ekspor untuk di-import ke browser.) Ulangi untuk `ani`. File `.p12` dikirim ke klien lewat jalur aman.

**4.4 Uji mTLS**
```bash
# Tanpa sertifikat klien → ditolak (handshake gagal / 400)
curl -i https://admin.kelompok.internal/whoami.shtml
# Dengan sertifikat klien → diterima
curl -i --cert ~/clients/budi.crt --key ~/clients/budi.key https://admin.kelompok.internal/whoami.shtml
```
Di browser: import `budi.p12` (Windows: klik dua kali; Firefox: *Certificates → Your Certificates → Import*; Chrome memakai store sistem). Buka `https://admin.kelompok.internal/whoami.shtml`, pilih sertifikat saat diminta.

**4.5 Revocation**

*Plan A — CRL dari step-ca (butuh versi step-ca yang mendukung CRL; cek dokumentasi/rilis).* Di `/etc/step-ca/config/ca.json` tambahkan blok di level atas:
```json
"crl": { "enabled": true, "generateOnRevoke": true }
```
Restart `step-ca`, cabut sertifikat, ambil CRL, lalu aktifkan di Apache:
```bash
sudo systemctl restart step-ca
step ca revoke --cert ~/clients/budi.crt --key ~/clients/budi.key \
  --ca-url https://ca.kelompok.internal:9000 --root /etc/step-ca/certs/root_ca.crt
curl -s "https://ca.kelompok.internal:9000/1.0/crl?pem" | sudo tee /opt/lampp/etc/ssl/crl.pem >/dev/null
# di vhost admin (httpd-kelompok.conf), hapus tanda # pada SSLCARevocationFile dan SSLCARevocationCheck leaf
sudo /opt/lampp/bin/httpd -t && sudo /opt/lampp/lampp reloadapache
curl -i --cert ~/clients/budi.crt --key ~/clients/budi.key https://admin.kelompok.internal/whoami.shtml   # → ditolak
```
`SSLCARevocationCheck leaf` hanya memeriksa CRL milik penerbit sertifikat klien (Intermediate), sehingga tidak butuh CRL untuk Root. Apache perlu di-reload setiap CRL diperbarui; untuk otomatis, buat cron/systemd timer yang menjalankan `curl ... > crl.pem && /opt/lampp/lampp reloadapache` tiap beberapa menit (opsional).

*Plan B — daftar serial yang dicabut (pasti jalan, mudah didemokan).* Setelah `step ca revoke`, tambahkan serialnya ke file map; Apache membaca ulang file map otomatis saat berubah (tanpa reload):
```bash
SERIAL=$(openssl x509 -in ~/clients/budi.crt -noout -serial | cut -d= -f2)
echo "$SERIAL 1" | sudo tee -a /opt/lampp/etc/ssl/revoked_serials.txt
curl -i --cert ~/clients/budi.crt --key ~/clients/budi.key https://admin.kelompok.internal/whoami.shtml   # → 403
```
Otomatisasi (opsional, bagian COULD): skrip `/usr/local/bin/revoke-client.sh <file.crt> <file.key>` yang menjalankan `step ca revoke` lalu menambah serial ke file map. Di laporan, jelaskan bahwa `step ca revoke` mengaktifkan pencabutan di sisi CA (CA menolak memperbarui sertifikat itu), sedangkan pemblokiran di sisi layanan memakai CRL atau daftar serial.

**Integrasi yang terjadi di fase ini:** sertifikat klien diterbitkan step-ca → Apache memverifikasi rantai dengan `ca-chain.crt` → pencabutan di step-ca disambungkan ke Apache lewat CRL atau daftar serial (lihat flowchart 3.5.4).

**Verifikasi:** tabel hasil uji (tanpa sertifikat, sertifikat valid, sertifikat dicabut) dengan screenshot. Tambah catatan di `LOG.md`.

---

### Fase 5 — DNSSEC, DoT, CAA (B, 20–21 Nov)

> **Sebelum mulai:** snapshot VPS dan cadangkan `/etc/bind` dan `/var/lib/bind`. Kesalahan DNSSEC bisa membuat semua resolusi gagal.

**5.1 DNSSEC dengan `dnssec-policy`**

Ubah deklarasi zona di `/etc/bind/named.conf.local`:
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
ls /var/cache/bind/K*kelompok.internal*      # kunci dibuat otomatis
```
Tunggu beberapa detik sampai zona ditandatangani, lalu cek:
```bash
dig @127.0.0.1 www.kelompok.internal A +dnssec        # ada RRSIG
dig @127.0.0.1 kelompok.internal DNSKEY +short
```

**5.2 Trust anchor.** `.internal` tidak ditandatangani oleh root DNS, jadi resolver harus diberi tahu kunci zona kita secara manual. Ambil DS:
```bash
sudo dnssec-dsfromkey -2 /var/cache/bind/K*kelompok.internal*.key
# contoh: kelompok.internal. IN DS 12345 13 2 ABCDEF...
```
Tambahkan di `/etc/bind/named.conf.options` (di luar blok `options`):
```
trust-anchors {
    kelompok.internal. static-ds 12345 13 2 "ABCDEF....";
};
```
(Ganti angka dan hash sesuai output DS kalian.) Lalu:
```bash
sudo named-checkconf && sudo systemctl restart named
```

**5.3 Verifikasi dengan `delv`** (validasi mandiri, tidak bergantung pada flag `ad`):
```bash
cat > /tmp/trust.key <<'EOF'
trust-anchors {
    kelompok.internal. static-ds 12345 13 2 "ABCDEF....";
};
EOF
delv @127.0.0.1 -a /tmp/trust.key www.kelompok.internal A
# harus ada baris: "; fully validated"
```
Tes negatif untuk demo: ubah sebentar satu record di zona **tanpa** menaikkan serial/menandatangani ulang tidak disarankan. Cukup tunjukkan RRSIG dan `fully validated`.

Setelah mengedit zona di kemudian hari: naikkan serial, lalu `sudo rndc reload kelompok.internal`.

**5.4 Record CAA.** Tambahkan di file zona (naikkan serial):
```
@   IN  CAA 0 issue "ca.kelompok.internal"
@   IN  CAA 0 iodef "mailto:admin@kelompok.internal"
```
Di laporan jelaskan: CAA menyatakan hanya CA tersebut yang berwenang menerbitkan sertifikat untuk domain ini. Tidak semua server ACME otomatis menegakkannya; di proyek ini ia berfungsi sebagai kebijakan yang terdokumentasi dan bisa diperiksa dengan `dig CAA`.
```bash
sudo rndc reload kelompok.internal
dig @127.0.0.1 kelompok.internal CAA +short
```

**5.5 DNS over TLS**

Minta sertifikat untuk `ns1` dengan *deploy hook* yang menyalin ke lokasi yang bisa dibaca user `bind`:
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
Tambahkan di `/etc/bind/named.conf.options` (blok `tls` di luar `options`, dan baris `listen-on` di dalam):
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
Jika AppArmor memblokir pembacaan `/etc/bind/certs`, lihat `journalctl -k | grep -i apparmor`, atau letakkan sertifikat di `/etc/bind/` yang sudah diizinkan profil bawaan.

Screenshot: output `delv` "fully validated", `kdig` dengan `+tls` sukses, `dig CAA`.

---

### Fase 6 — Mail TLS: Postfix + Dovecot (C, 17–21 Nov)

**6.1 Instal dasar (Selasa 17 Nov, belum pakai TLS)**
```bash
sudo apt -y install postfix dovecot-imapd dovecot-pop3d
# Saat ditanya: General type = "Internet Site", System mail name = kelompok.internal
sudo adduser alice
sudo adduser bob
```

**6.2 Postfix dasar**
```bash
sudo postconf -e "myhostname = mail.kelompok.internal"
sudo postconf -e "mydomain = kelompok.internal"
sudo postconf -e "myorigin = \$mydomain"
sudo postconf -e "mydestination = \$myhostname, localhost.\$mydomain, localhost, \$mydomain"
sudo postconf -e "home_mailbox = Maildir/"
sudo systemctl restart postfix
```

**6.3 Dovecot dasar**

`/etc/dovecot/conf.d/10-mail.conf`:
```
mail_location = maildir:~/Maildir
```
`/etc/dovecot/conf.d/10-auth.conf`:
```
disable_plaintext_auth = yes
auth_mechanisms = plain login
```
`/etc/dovecot/conf.d/10-master.conf` (bagian `service auth`):
```
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}
```

**6.4 Sertifikat `mail` via ACME (Jumat 20 Nov)**
```bash
sudo certbot certonly --webroot -w /opt/lampp/acme \
  --server https://ca.kelompok.internal:9000/acme/acme/directory \
  -d mail.kelompok.internal --register-unsafely-without-email --agree-tos --no-eff-email \
  --deploy-hook "systemctl reload postfix dovecot"
```

**6.5 Aktifkan TLS di Dovecot** — `/etc/dovecot/conf.d/10-ssl.conf`:
```
ssl = required
ssl_cert = </etc/letsencrypt/live/mail.kelompok.internal/fullchain.pem
ssl_key  = </etc/letsencrypt/live/mail.kelompok.internal/privkey.pem
ssl_min_protocol = TLSv1.2
```

**6.6 Aktifkan TLS dan SASL di Postfix**
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

# Port 587 (submission) wajib TLS + login
sudo postconf -M submission/inet="submission inet n - y - - smtpd"
sudo postconf -P "submission/inet/syslog_name=postfix/submission"
sudo postconf -P "submission/inet/smtpd_tls_security_level=encrypt"
sudo postconf -P "submission/inet/smtpd_sasl_auth_enable=yes"

sudo systemctl restart postfix dovecot
```
Tambahkan tag ketat SMTP lain hanya jika waktu cukup. `smtpd_recipient_restrictions` di atas menjaga server tidak menjadi *open relay*; jangan dihapus.

**6.7 Verifikasi**
```bash
# TLS valid (rantai dipercaya)
openssl s_client -starttls smtp -connect mail.kelompok.internal:587 -CAfile /etc/step-ca/certs/root_ca.crt </dev/null | grep -E "Verify return code|subject|issuer"
openssl s_client -connect mail.kelompok.internal:993 -CAfile /etc/step-ca/certs/root_ca.crt </dev/null | grep "Verify return code"
# → Verify return code: 0 (ok)

# Kirim email lewat 587 dengan login
swaks --to bob@kelompok.internal --from alice@kelompok.internal \
      --server mail.kelompok.internal --port 587 -tls \
      --auth LOGIN --auth-user alice --auth-password '<password alice>'
sudo ls /home/bob/Maildir/new/
sudo tail -n 20 /var/log/mail.log      # atau: sudo journalctl -u postfix -n 30
```
Email yang masuk dengan header `Received: ... (using TLSv1.3 ...)` adalah bukti untuk laporan. Port 25 keluar ke internet sering diblokir; cukup tunjukkan pengiriman internal antar-user.

---

### Fase 7 — Roundcube + database MariaDB (C, Senin 23 Nov; dibantu B untuk database)

**Tujuan:** webmail HTTPS yang terhubung ke Dovecot (baca email), Postfix (kirim email), dan MariaDB XAMPP (data Roundcube).

**7.1 Cek prasyarat PHP dan MariaDB**
```bash
/opt/lampp/bin/php -m | grep -i -E "mbstring|intl|xml|dom|curl|openssl|zip|fileinfo|ctype|json|iconv|session|pdo_mysql|mysqli"
sudo /opt/lampp/lampp status                     # Apache dan MySQL harus running
```
Jika ada ekstensi yang belum aktif (terutama `intl`, `mbstring`), buka `/opt/lampp/etc/php.ini`, hapus tanda `;` pada baris `extension=...` yang sesuai, lalu `sudo /opt/lampp/lampp restartapache`.

**7.2 Buat database dan user khusus Roundcube**
```bash
sudo /opt/lampp/bin/mysql -u root -p <<'SQL'
CREATE DATABASE roundcube CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'roundcube'@'localhost' IDENTIFIED BY 'GANTI_PASSWORD_KUAT';
GRANT ALL PRIVILEGES ON roundcube.* TO 'roundcube'@'localhost';
FLUSH PRIVILEGES;
SQL
```
(Gunakan password kuat berbeda dari password `root`, simpan di password manager. Jangan memakai akun `root` untuk aplikasi.)

**7.3 Unduh dan pasang Roundcube**
```bash
cd /tmp
RC_VER=1.6.11            # cek versi stabil terbaru di roundcube.net/download dan syarat PHP-nya
wget https://github.com/roundcube/roundcubemail/releases/download/${RC_VER}/roundcubemail-${RC_VER}-complete.tar.gz
sudo tar xzf roundcubemail-${RC_VER}-complete.tar.gz -C /opt/lampp/htdocs/
sudo mv /opt/lampp/htdocs/roundcubemail-${RC_VER} /opt/lampp/htdocs/roundcube
```

**7.4 Impor skema ke database**
```bash
sudo /opt/lampp/bin/mysql -u roundcube -p roundcube < /opt/lampp/htdocs/roundcube/SQL/mysql.initial.sql
sudo /opt/lampp/bin/mysql -u roundcube -p roundcube -e "SHOW TABLES;"     # harus muncul tabel users, session, dll.
```

**7.5 Konfigurasi Roundcube yang terhubung ke MariaDB, Dovecot, dan Postfix**
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
// PHP XAMPP memakai penyimpanan sertifikat sendiri: arahkan ke Root CA kita
\$config['imap_conn_options'] = ['ssl' => ['verify_peer' => true, 'verify_peer_name' => true, 'cafile' => '/opt/lampp/etc/ssl/root_ca.crt']];
\$config['smtp_conn_options'] = ['ssl' => ['verify_peer' => true, 'verify_peer_name' => true, 'cafile' => '/opt/lampp/etc/ssl/root_ca.crt']];
EOF
sudo chown root:daemon /opt/lampp/htdocs/roundcube/config/config.inc.php
sudo chmod 640 /opt/lampp/htdocs/roundcube/config/config.inc.php
sudo chown -R daemon:daemon /opt/lampp/htdocs/roundcube/temp /opt/lampp/htdocs/roundcube/logs
sudo rm -rf /opt/lampp/htdocs/roundcube/installer
```
Ganti `GANTI_PASSWORD_KUAT` dengan password database dari 7.2. (`daemon` adalah user yang menjalankan Apache XAMPP.)

**7.6 Virtual host webmail**
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

**7.7 Jalankan dan uji**
1. Buka `https://webmail.kelompok.internal`, login `alice` (user Linux dari Fase 6), kirim email ke `bob`.
2. Login `bob` dan pastikan email diterima.
3. Cek database terisi: `sudo /opt/lampp/bin/mysql -u roundcube -p roundcube -e "SELECT user_id, username FROM users;"`.
4. Gembok hijau di browser. Jika gagal, lihat `/opt/lampp/htdocs/roundcube/logs/errors.log`, `/opt/lampp/logs/error_log`, dan bagian 12.

**Integrasi yang terjadi di fase ini:** Apache → PHP → Roundcube → MariaDB (data), Dovecot 993 (baca email), Postfix 587 (kirim email), semuanya dengan TLS yang diverifikasi memakai Root CA kita (lihat flowchart 3.5.5).

---

### Fase 8 — Fail2ban dan hardening (B, Senin 23 Nov)

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
Jail `apache-auth` membaca `/opt/lampp/logs/error_log` (karena itu `backend = auto` dan `logpath` ditulis eksplisit). Jika jail `postfix` atau `dovecot` error karena sumber log, pasang `rsyslog` (`sudo apt -y install rsyslog`) dan set `backend = auto` serta `logpath = /var/log/mail.log` pada jail tersebut.

**Daftarkan `ignoreip` untuk semua IP anggota sebelum menguji** agar tidak memblokir diri sendiri saat demo. Untuk demo brute force, gunakan perangkat satu orang yang tidak ada di `ignoreip`, atau lepas ban lewat `sudo fail2ban-client set sshd unbanip <IP>`.

**Hardening tambahan (cepat, nilai plus di laporan):**
- SSH: matikan login password jika semua sudah punya key (`PasswordAuthentication no` di `/etc/ssh/sshd_config`) — lakukan hanya setelah memastikan key masing-masing berfungsi.
- Apache XAMPP: `ServerTokens Prod` dan `ServerSignature Off` sudah ada di `httpd-kelompok.conf` (Fase 3.6). Halaman `/dashboard` dan `/phpmyadmin` bawaan XAMPP tidak boleh bisa diakses publik (Fase 3.4).
- MariaDB: hanya `127.0.0.1`, akun anonim dan database `test` dihapus oleh `lampp security`, aplikasi memakai user khusus (bukan `root`).
- Batasi izin: `sudo chmod 600 /etc/step-ca/password.txt`, kunci privat tidak pernah masuk Git.
- `unattended-upgrades` aktif (opsional).

---

### Fase 8B — Unbound (opsional, B, jika waktu cukup)

**Tujuan:** menambah satu layanan server yang jelas (resolver DNS dengan validasi DNSSEC) di luar Bind9. Unbound dijalankan di `127.0.0.1:5353` agar tidak bentrok dengan Bind9 di port 53. Kerjakan hanya setelah Fase 5 selesai; jika ada kendala (misalnya bentrok dengan `systemd-resolved`), lewati karena ini bagian COULD.

```bash
sudo apt -y install unbound
sudo tee /etc/unbound/unbound.conf.d/kelompok.conf <<'EOF'
server:
    interface: 127.0.0.1
    port: 5353
    access-control: 127.0.0.0/8 allow
    do-not-query-localhost: no
    trust-anchor: "kelompok.internal. DS 12345 13 2 ABCDEF...."

stub-zone:
    name: "kelompok.internal"
    stub-addr: 127.0.0.1@53
EOF
sudo unbound-checkconf
sudo systemctl enable --now unbound && sudo systemctl restart unbound
dig @127.0.0.1 -p 5353 www.kelompok.internal +dnssec
```
Isi angka dan hash `DS` sama dengan hasil `dnssec-dsfromkey` di Fase 5.2. **Verifikasi:** respons memuat flag `ad` (authenticated data). Screenshot untuk laporan.

---

### Fase 9 — Integrasi, menjalankan, pengujian, laporan, demo (Semua, 24–28 Nov)

**Tujuan:** memastikan semua layanan saling terhubung dengan benar, bisa dinyalakan/dimatikan dengan urutan yang jelas, lulus semua pengujian, dan siap didemokan.

**9.1 Peta integrasi: siapa bergantung pada siapa**

| Layanan | Bergantung pada | Titik integrasi (file/port) | Cara cek |
|---|---|---|---|
| chrony | — | Jam sistem | `chronyc tracking` |
| Bind9 | chrony, file zona | 53 dan 853, `/var/lib/bind` | `dig @127.0.0.1 www.kelompok.internal +short` |
| Resolver VPS | Bind9 | `/etc/systemd/resolved.conf.d/kelompok.conf` | `resolvectl status` |
| step-ca | Bind9 (nama `ca`), chrony | 9000, `/etc/step-ca` | `curl https://ca.kelompok.internal:9000/health` |
| certbot | step-ca, Apache port 80, Bind9 | `--server .../acme/acme/directory`, webroot `/opt/lampp/acme` | `sudo certbot certificates` |
| Apache (XAMPP) | Sertifikat certbot, DNS | `httpd-kelompok.conf`, `/etc/letsencrypt/live/` | `curl -I https://www.kelompok.internal` |
| MariaDB (XAMPP) | XAMPP, disk | `127.0.0.1:3306`, `/opt/lampp/var/mysql` | `sudo /opt/lampp/lampp status` |
| mTLS admin | Apache, `ca-chain.crt`, sertifikat klien | `/opt/lampp/etc/ssl/` | `curl --cert ... https://admin.../whoami.shtml` |
| DoT (Bind9) | Sertifikat `ns1` dan *deploy hook* | `/etc/bind/certs/` | `kdig +tls ...` |
| Postfix | Sertifikat `mail`, Dovecot (SASL), user Linux | `postconf`, `/var/spool/postfix/private/auth` | `swaks ... --port 587 -tls` |
| Dovecot | Sertifikat `mail`, user Linux | 993, `10-ssl.conf` | `openssl s_client -connect mail...:993` |
| Roundcube | Apache + PHP, MariaDB, Dovecot, Postfix, Root CA | `config.inc.php` | Login webmail dan `SHOW TABLES` |
| Fail2ban | Log SSH, mail, `/opt/lampp/logs/error_log` | `/etc/fail2ban/jail.local` | `sudo fail2ban-client status` |
| Unbound (opsional) | Bind9, trust anchor | 5353 | `dig @127.0.0.1 -p 5353 ... +dnssec` |
| UFW | Semua layanan | Daftar port bagian 3.4 | `sudo ufw status numbered` |

**9.2 Menjalankan, menghentikan, dan memeriksa layanan**

```bash
# Aktifkan semua layanan agar otomatis jalan saat boot (sekali saja)
sudo systemctl enable chrony named step-ca xampp postfix dovecot fail2ban

# Menyalakan (urutan aman: waktu → DNS → CA → web/DB → mail → proteksi)
sudo systemctl start chrony named step-ca xampp postfix dovecot fail2ban

# Menghentikan (urutan terbalik)
sudo systemctl stop fail2ban dovecot postfix xampp step-ca named

# Restart satu layanan
sudo systemctl restart named          # DNS
sudo systemctl restart step-ca        # CA
sudo /opt/lampp/lampp restartapache   # Apache saja (setelah ubah httpd-kelompok.conf)
sudo /opt/lampp/lampp restartmysql    # MariaDB saja
sudo systemctl restart postfix dovecot

# Memeriksa
systemctl is-active chrony named step-ca xampp postfix dovecot fail2ban
sudo /opt/lampp/lampp status
sudo ss -tlnp
```

Aturan praktis: ubah konfigurasi → uji sintaks (`named-checkconf`, `/opt/lampp/bin/httpd -t`, `postfix check`, `dovecot -n`) → reload/restart → verifikasi.

**9.3 Skrip pemeriksaan cepat `cek-semua.sh`** (juga dipakai saat latihan demo):
```bash
sudo tee /usr/local/bin/cek-semua.sh <<'EOF'
#!/bin/bash
echo "== Status layanan =="
for s in chrony named step-ca xampp postfix dovecot fail2ban; do
  printf "%-10s %s\n" "$s" "$(systemctl is-active $s)"
done
echo; echo "== XAMPP =="; /opt/lampp/lampp status 2>&1 | head -n 8
echo; echo "== Port mendengarkan =="
ss -tlnp | grep -E ':(22|53|80|443|853|9000|25|587|993|3306)\b' | awk '{print $4}' | sort -u
echo; echo "== Uji cepat =="
printf "DNS                    : "; dig +short www.kelompok.internal | head -1
printf "CA health              : "; curl -s https://ca.kelompok.internal:9000/health; echo
printf "HTTPS www (kode)       : "; curl -s -o /dev/null -w "%{http_code}\n" https://www.kelompok.internal
printf "mTLS tanpa sertifikat  : "; curl -s -o /dev/null -w "%{http_code} (000 atau 400 = ditolak)\n" https://admin.kelompok.internal/whoami.shtml
printf "IMAPS                  : "; echo | openssl s_client -connect mail.kelompok.internal:993 -CAfile /etc/step-ca/certs/root_ca.crt 2>/dev/null | grep "Verify return code"
echo; sudo certbot certificates 2>/dev/null | grep -E "Domains|Expiry"
EOF
sudo chmod +x /usr/local/bin/cek-semua.sh
sudo cek-semua.sh
```

**9.4 Uji integrasi end-to-end (urutkan dari hulu ke hilir)**
1. **DNS:** `dig www.kelompok.internal +short` memberi `<IP_VPS>`.
2. **CA:** `/health` menjawab `ok`.
3. **ACME:** `sudo certbot renew --force-renewal` sukses dan Apache ter-reload.
4. **HTTPS:** `https://www.kelompok.internal` bergembok hijau di klien.
5. **mTLS:** tanpa sertifikat ditolak, dengan sertifikat `whoami.shtml` menampilkan identitas.
6. **Revocation:** cabut sertifikat `budi`, akses kembali ditolak.
7. **DNSSEC dan DoT:** `delv` *fully validated*, `kdig +tls` berhasil.
8. **Mail:** `swaks` mengirim alice → bob lewat 587 ber-TLS dan email masuk Maildir.
9. **Webmail dan database:** login Roundcube, kirim/terima, lalu `SELECT username FROM users;` menampilkan pengguna.
10. **Proteksi:** gagal login SSH berulang menyebabkan IP uji diblokir Fail2ban.

**9.5 Uji reboot dan simulasi kegagalan**
- **Reboot:** `sudo reboot`, tunggu, lalu `sudo cek-semua.sh`. Semua layanan, termasuk `xampp`, harus aktif sendiri; `sudo ss -tlnp` memperlihatkan 3306 hanya di `127.0.0.1`.
- **step-ca mati:** `sudo systemctl stop step-ca`. Situs tetap berjalan (sertifikat masih berlaku), tetapi `certbot renew --force-renewal` gagal. Nyalakan lagi dan ulangi.
- **MariaDB mati:** `sudo /opt/lampp/lampp stopmysql`. Roundcube gagal login, layanan lain tidak terpengaruh. Nyalakan dengan `startmysql`.
- **Bind9 mati:** penerbitan sertifikat dan resolusi nama gagal. Nyalakan dengan `systemctl start named`.

**9.6 Backup (database dan konfigurasi)**
```bash
mkdir -p ~/backup && cd ~/backup
sudo /opt/lampp/bin/mysqldump -u roundcube -p roundcube > roundcube-$(date +%F).sql
sudo tar czf konfigurasi-$(date +%F).tgz \
  /etc/bind /opt/lampp/etc /etc/postfix /etc/dovecot /etc/fail2ban/jail.local \
  /etc/step-ca/config /etc/step-ca/certs \
  /etc/systemd/system/xampp.service /etc/systemd/system/step-ca.service
```
Jangan menyertakan `/etc/step-ca/secrets` dan `password.txt` pada arsip yang dibagikan. File `.sql` berisi data pengguna: jangan masuk ke repo Git publik.

**9.7 Pengujian, laporan, dan demo**
1. Isi **matriks pengujian** (bagian 11) lengkap dengan bukti.
2. Uji ulang dari **perangkat bersih** (laptop/HP yang belum pernah dipakai): instal Root, atur DNS, jalankan skenario demo.
3. **Cadangan demo:** rekam video demo penuh dan siapkan file hosts agar tidak tergantung DNS.
4. Susun laporan dan slide dari screenshot dan diagram di bagian 3.5 (ekspor Mermaid ke PNG/SVG).

---

## 9. Skenario Demo dan Naskah Presentasi

### 9.1 Skenario demo (sekitar 10 menit)

| # | Langkah | Yang ditunjukkan | Pembicara |
|---|---|---|---|
| 1 | Buka `https://www.kelompok.internal` di laptop **tanpa** Root | Peringatan "Not Secure" | A |
| 2 | Pasang Root CA, muat ulang | Gembok hijau, lihat rantai Root → Intermediate → Server | A |
| 3 | Buka `https://admin.kelompok.internal` tanpa sertifikat klien | Ditolak | A |
| 4 | Pasang `budi.p12`, buka lagi, lalu `/whoami.shtml` | Masuk, tampil identitas klien | A |
| 5 | Jalankan `sudo certbot renew --force-renewal` | Perpanjangan otomatis dari step-ca | A |
| 6 | Cabut sertifikat `budi`, coba akses lagi | 403 / ditolak | A |
| 7 | `delv` (fully validated) dan `kdig +tls` | DNSSEC + DoT berfungsi | B |
| 8 | Brute force SSH singkat, tunjukkan Fail2ban memblokir | Proteksi aktif | B |
| 9 | Kirim email alice → bob dengan TLS, baca di Roundcube | Mail TLS + webmail HTTPS | C |

Untuk 10 menit, bagian 8 bisa diganti tangkapan layar jika waktu mepet.

### 9.2 Struktur presentasi (±10–12 slide)
1. Judul dan anggota
2. Masalah: kenapa layanan internal butuh CA sendiri
3. Konsep PKI: Root, Intermediate, sertifikat, ACME, mTLS (satu diagram)
4. Arsitektur dan daftar aplikasi (lama vs baru)
5. Alur ACME otomatis (diagram)
6. Peran Bind9: zona, DNSSEC, DoT, CAA
7. Keamanan berlapis: mTLS, revocation, Fail2ban, UFW
8. Hasil pengujian (ringkas dari matriks)
9. **Demo langsung**
10. Kendala dan solusi
11. Kesimpulan dan pengembangan lanjut (OCSP, HSM, CA kluster)
12. Tanya-jawab

### 9.3 Pertanyaan yang mungkin ditanyakan (siapkan jawabannya)
- Kenapa Root CA kunci-nya offline? → Jika Intermediate bocor, cukup dicabut; Root tetap aman dan tidak perlu didistribusikan ulang ke semua klien.
- Apa bedanya ACME dan membuat sertifikat manual? → Otomatis, berumur pendek, mengurangi risiko sertifikat kedaluwarsa.
- Apa bedanya TLS biasa dan mTLS? → Pada mTLS server juga memverifikasi identitas klien.
- Bagaimana DNSSEC mencegah pemalsuan? → Setiap record ditandatangani; resolver memverifikasi rantai tanda tangan sampai trust anchor.
- Kalau step-ca mati? → Layanan berjalan sampai sertifikat kedaluwarsa; perpanjangan tertunda.
- Kenapa tidak Let's Encrypt? → Domain `.internal` bukan domain publik sehingga tidak bisa divalidasi.
- Bagaimana pencabutan dipantau klien? → CRL/OCSP, atau (di proyek ini) daftar serial di sisi server.

---

## 10. Kerangka Laporan

**Halaman depan, kata pengantar, daftar isi/gambar/tabel**

**Bab I — Pendahuluan**: latar belakang, rumusan masalah, tujuan, ruang lingkup, manfaat, sistematika.

**Bab II — Landasan teori**: PKI dan sertifikat X.509; Root dan Intermediate CA; TLS dan mTLS; ACME; DNS, DNSSEC, DoT, CAA; SMTP/IMAP dengan TLS; Fail2ban, UFW.

**Bab III — Perancangan**: kebutuhan perangkat keras/lunak (spek VPS), topologi/arsitektur, tabel domain dan port, alur ACME, alur mTLS, rancangan keamanan, pembagian tugas dan jadwal.

**Bab IV — Implementasi** (satu subbab per fase 0–8, dengan perintah, konfigurasi, dan screenshot): baseline, Bind9, step-ca, XAMPP dan MariaDB, certbot dan Apache, mTLS dan revocation, DNSSEC/DoT/CAA, Postfix/Dovecot TLS, Roundcube dengan MariaDB, Fail2ban, peta integrasi.

**Bab V — Pengujian dan pembahasan**: matriks pengujian (bagian 11), analisis hasil, kendala dan solusi, keterbatasan (misalnya Root dan Intermediate pada satu mesin, CRL vs OCSP).

**Bab VI — Penutup**: kesimpulan, saran pengembangan.

**Daftar pustaka. Lampiran:** file konfigurasi, tautan repo Git, log, kontribusi tiap anggota.

**Tips nilai laporan:** setiap perintah diikuti hasilnya, setiap konfigurasi diikuti penjelasan baris pentingnya, setiap gambar diberi nomor dan keterangan.

---

## 11. Matriks Pengujian

| ID | Pengujian | Perintah/langkah | Hasil yang diharapkan | Status | Bukti |
|---|---|---|---|---|---|
| T01 | Resolusi DNS lokal | `dig www.kelompok.internal +short` | `<IP_VPS>` | ☐ | |
| T02 | Rekursi dibatasi | `dig @<IP_VPS> google.com` dari IP tak terdaftar | REFUSED | ☐ | |
| T03 | CA sehat | `curl https://ca.kelompok.internal:9000/health` | `{"status":"ok"}` | ☐ | |
| T04 | Rantai sertifikat | `openssl s_client -connect www.kelompok.internal:443` | `Verify return code: 0 (ok)` | ☐ | |
| T05 | HTTPS valid di browser | Buka situs di klien | Gembok tanpa peringatan | ☐ | |
| T06 | Redirect HTTP→HTTPS | `curl -I http://www.kelompok.internal` | 301 ke https | ☐ | |
| T07 | ACME issuance | `certbot certificates` | Sertifikat terbit dari step-ca | ☐ | |
| T08 | Perpanjangan otomatis | `certbot renew --force-renewal` | Sukses, layanan reload | ☐ | |
| T09 | mTLS tanpa sertifikat | `curl https://admin.../whoami.shtml` | Ditolak (400) | ☐ | |
| T10 | mTLS dengan sertifikat | `curl --cert ... ` | 200 + identitas | ☐ | |
| T11 | Revocation | Cabut lalu akses | 403 | ☐ | |
| T12 | DNSSEC | `delv -a trust.key www...` | `fully validated` | ☐ | |
| T13 | DoT | `kdig +tls ...` | Jawaban via TLS | ☐ | |
| T14 | CAA | `dig CAA kelompok.internal` | Record CAA tampil | ☐ | |
| T15 | SMTP TLS | `openssl s_client -starttls smtp ...:587` | Verify ok | ☐ | |
| T16 | IMAPS | `openssl s_client ...:993` | Verify ok | ☐ | |
| T17 | Kirim email | `swaks ...` | Masuk ke Maildir bob | ☐ | |
| T18 | Webmail | Login Roundcube, kirim/terima | Berhasil via HTTPS | ☐ | |
| T19 | Fail2ban | Gagal login SSH berulang dari IP uji | IP diblokir | ☐ | |
| T20 | Reboot | `sudo reboot` | Semua service aktif otomatis | ☐ | |
| T21 | Waktu | `chronyc tracking` | Sinkron | ☐ | |
| T22 | Perangkat bersih | Skenario demo dari awal | Berhasil | ☐ | |
| T23 | MariaDB hanya localhost | `sudo ss -tlnp` lalu cari `3306`; login root tanpa password | Alamat `127.0.0.1:3306`; login tanpa password ditolak | ☐ | |
| T24 | Roundcube terhubung ke database | `mysql -u roundcube -p roundcube -e "SELECT username FROM users;"` | Pengguna yang pernah login tercatat | ☐ | |

---

## 12. Troubleshooting

| Gejala | Kemungkinan penyebab | Solusi |
|---|---|---|
| certbot: gagal menghubungi `ca.kelompok.internal` | VPS tidak me-resolve nama, atau Root belum dipercaya | `dig ca.kelompok.internal`, ulangi Fase 1.6 dan 3.1 |
| certbot: `SSL: CERTIFICATE_VERIFY_FAILED` | Root belum masuk store atau python memakai bundle lain | `update-ca-certificates`; coba `REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt certbot ...` |
| ACME challenge gagal (HTTP-01) | Port 80 tertutup, vhost salah, nama tidak mengarah ke VPS | Cek `ufw status`, `curl http://www.kelompok.internal/.well-known/acme-challenge/x`, isi `/opt/lampp/acme`, dan `/opt/lampp/logs/error_log` |
| Sertifikat "belum berlaku"/"kedaluwarsa" salah | Jam tidak sinkron | `chronyc tracking`, `timedatectl` |
| step-ca tidak mau start | Password salah, izin file, JSON rusak | `journalctl -u step-ca -n 50`; cek `ca.json` dengan `jq . /etc/step-ca/config/ca.json` |
| Browser tetap warning | Root belum dipasang di store yang benar (Firefox punya store sendiri) | Instal Root di Firefox terpisah atau aktifkan enterprise roots |
| Browser tidak menawarkan sertifikat klien | `.p12` belum di-import, rantai tidak cocok | Import ulang, pastikan memakai `--ca intermediate` saat membuat `.p12` |
| Apache: CRL tidak terbaca / semua klien ditolak setelah CRL dipasang | File `crl.pem` kosong/rusak atau bukan CRL milik Intermediate | Cek `openssl crl -in /opt/lampp/etc/ssl/crl.pem -noout -issuer`; pakai `SSLCARevocationCheck leaf`; jika masih gagal, gunakan Plan B (daftar serial) |
| Setelah DNSSEC, resolusi gagal | Trust anchor salah/kedaluwarsa, zona belum selesai sign | Cek `rndc dnssec -status`, hapus `trust-anchors` sementara; bila parah, kembalikan snapshot |
| Bind: DoT tidak listen di 853 | Izin sertifikat, AppArmor, sintaks `tls` | `named-checkconf`, `journalctl -u named`, `ss -tlnp \| grep 853` |
| `kdig` gagal verifikasi | Nama host berbeda dengan SAN | Gunakan `+tls-hostname=ns1.kelompok.internal` dan `+tls-ca=root_ca.crt` |
| Postfix menolak login SASL | Socket Dovecot salah atau izin | Cek `/var/spool/postfix/private/auth`, restart Dovecot dulu lalu Postfix |
| Email tidak terkirim keluar | Port 25 diblokir penyedia VPS | Uji internal saja (alice→bob) |
| XAMPP tidak mau start / "port 80 in use" | Apache atau Nginx lain, atau proses lama, memakai port | `sudo ss -tlnp` lalu cari `:80`; `sudo systemctl disable --now apache2 nginx`; `sudo /opt/lampp/lampp restartapache` |
| `httpd -t` error `SSLCertificateFile does not exist` | Vhost HTTPS ditambahkan sebelum sertifikat terbit | Hapus sementara blok `*:443` dari `httpd-kelompok.conf`, terbitkan sertifikat (Fase 3.7), lalu tambahkan lagi |
| Halaman dashboard XAMPP muncul, vhost proyek tidak aktif | `Include etc/extra/httpd-kelompok.conf` belum ada di `httpd.conf`, atau `Listen 443` ganda | `grep -n httpd-kelompok /opt/lampp/etc/httpd.conf`, `grep -n Listen /opt/lampp/etc/httpd.conf`, `/opt/lampp/bin/httpd -t` |
| `lampp startmysql` gagal | Port 3306 dipakai, data/izin bermasalah, `my.cnf` salah | Lihat berkas `*.err` di `/opt/lampp/var/mysql/`; kembalikan `my.cnf.bak`; cek `sudo ss -tlnp` |
| Roundcube: "Database connection failed" | User/password di `db_dsnw` salah, MariaDB mati, skema belum diimpor | `sudo /opt/lampp/bin/mysql -u roundcube -p roundcube -e "SHOW TABLES;"`, periksa `config.inc.php` |
| Roundcube: "Connection to storage server failed" | PHP XAMPP tidak memercayai Root CA, nama host salah, Dovecot mati | Pastikan `cafile` ke `/opt/lampp/etc/ssl/root_ca.crt`, `systemctl status dovecot`, `dig mail.kelompok.internal` |
| Roundcube blank/500 | Ekstensi PHP kurang atau izin `temp`/`logs` salah | Lihat `/opt/lampp/htdocs/roundcube/logs/errors.log` dan `/opt/lampp/logs/error_log`; aktifkan ekstensi di `php.ini`; `chown -R daemon:daemon temp logs` |
| Setelah reboot XAMPP tidak jalan | Service `xampp` belum di-enable atau MariaDB gagal | `systemctl status xampp`, `sudo systemctl enable xampp`, `journalctl -u xampp -n 50` |
| *Deploy hook* certbot gagal | Path hook salah | Pastikan `--deploy-hook "/opt/lampp/lampp reloadapache"` dan cek `/etc/letsencrypt/renewal/*.conf` |
| Fail2ban tidak membaca log | Backend/logpath tidak cocok | Gunakan `backend = systemd`, atau pasang rsyslog |
| Diri sendiri terblokir Fail2ban | IP belum di `ignoreip` | Akses lewat konsol VPS, `fail2ban-client set <jail> unbanip <IP>` |
| Terkunci dari SSH karena UFW | Lupa mengizinkan 22 | Pakai konsol/VNC dari panel VPS, `ufw allow 22/tcp` |

---

## 13. Risiko dan Mitigasi

| Risiko | Dampak | Mitigasi |
|---|---|---|
| DNSSEC salah konfigurasi | Semua resolusi gagal | Snapshot sebelum mulai, `delv` dan `named-checkzone`, bisa diputus dengan menghapus `dnssec-policy` |
| Password CA hilang | CA harus dibuat ulang | Simpan di password manager bersama; cadangan terenkripsi |
| Kunci Root bocor | Seluruh PKI tidak tepercaya | Kunci offline, jangan masuk Git/chat |
| Tiga orang menimpa konfigurasi | Layanan rusak | Satu pemilik per layanan, Git, backup `.bak` |
| Klien demo tidak mempercayai Root | Demo gagal | Siapkan perangkat uji, file hosts, video cadangan |
| VPS publik diserang | Layanan terganggu | UFW, Fail2ban, rekursi DNS dibatasi, port 9000 dibatasi bila mungkin |
| Fitur tidak selesai tepat waktu | Nilai turun | Aturan pemotongan bagian 5.2, M2 pada 22 November |
| Aplikasi baru dianggap kurang | Nilai jumlah aplikasi | Daftar bagian 3.2 + konfirmasi bagian 14 |
| Port 25 diblokir | Mail ke luar gagal | Uji internal; jelaskan di laporan |
| Dosen menganggap Root dan Intermediate di satu mesin | Poin kecil | Jelaskan: Root key dipindah offline sesudah init (Fase 2.6) |
| Bawaan XAMPP longgar (root DB tanpa password, dashboard terbuka) | Akses tidak sah ke database/panel | `lampp security`, `bind-address = 127.0.0.1`, batasi phpMyAdmin, jangan buka 3306 dan 21 di UFW |
| XAMPP tidak ikut `apt upgrade` dan tidak dikelola systemd bawaan | Patch tertinggal, layanan tidak naik saat boot | Service `xampp` buatan sendiri, catat versi, jadwalkan pengecekan pembaruan manual, jelaskan di laporan |
| Path non-standar XAMPP membuat tutorial umum tidak cocok | Salah konfigurasi, waktu terbuang | Gunakan tabel path di Fase 3, satu file `httpd-kelompok.conf`, selalu uji dengan `httpd -t` |
| XAMPP tidak diterima dosen | Harus berpindah web server | Tanyakan sejak awal (bagian 14 no. 7). Rencana cadangan: Apache dari `apt` (konfigurasi sama, path berbeda) |

---

## 14. Hal yang Dikonfirmasi ke Dosen/Aslab

Kirim sebelum atau di awal pekan pertama:
1. Apakah **step-ca, certbot, Fail2ban, chrony, Roundcube** dihitung sebagai aplikasi server baru? Apakah "aplikasi server" hanya layanan jaringan (daemon), atau klien seperti certbot dan step CLI juga dihitung? Apakah Unbound (opsional) perlu ditambahkan agar aman?
2. Apakah fitur lanjutan Bind9 (DNSSEC, DoT, CAA) dihitung sebagai kompleksitas tambahan?
3. Boleh memakai domain `.internal` dan tidak memakai domain publik?
4. Apakah demo dilakukan langsung di VPS dan boleh memakai perangkat sendiri (laptop/HP) sebagai klien?
5. Format dan batas panjang laporan, format pengumpulan, dan tanggal/jam presentasi sebenarnya.
6. Apakah ada pembatasan di VPS (port tertentu diblokir, batasan snapshot/akses root)?
7. Apakah **XAMPP** (Apache + PHP + MariaDB di `/opt/lampp`) boleh dipakai di Ubuntu Server, dan apakah MariaDB dihitung sebagai aplikasi server baru?

---

## 15. Checklist Akhir Sebelum Pengumpulan

**Sistem**
- [ ] Semua service `active (running)` setelah reboot (`systemctl --failed` kosong)
- [ ] Snapshot final dan backup konfigurasi dibuat
- [ ] Kunci Root tidak ada di server, salinan offline aman
- [ ] MariaDB hanya `127.0.0.1`, password `root`/phpMyAdmin sudah diset, Roundcube memakai user database sendiri
- [ ] `cek-semua.sh` menampilkan semua layanan `active` setelah reboot
- [ ] Tidak ada password/kunci privat di repo Git

**Fungsional**
- [ ] Matriks pengujian T01–T24 terisi dan berbukti
- [ ] Skenario demo berhasil dari perangkat bersih, dua kali
- [ ] File hosts dan video demo siap sebagai cadangan

**Dokumen**
- [ ] Laporan PDF lengkap (Bab I–VI, lampiran, daftar pustaka)
- [ ] Semua screenshot bernomor dan berketerangan
- [ ] Kontribusi tiap anggota tercantum
- [ ] Slide final, nama anggota benar, waktu presentasi dilatih

**Administrasi**
- [ ] Format penamaan file sesuai aturan dosen
- [ ] Pengumpulan sebelum tenggat (rencana: **Min 29 Nov**, usahakan sehari sebelumnya)

---

## 16. Referensi

- Smallstep (step-ca, step CLI, ACME, revocation): `smallstep.com/docs`
- BIND 9 Administrator Reference Manual (dnssec-policy, tls, trust-anchors): `bind9.readthedocs.io`
- Apache HTTP Server (mod_ssl, mod_rewrite, mod_include): `httpd.apache.org/docs/2.4`
- XAMPP untuk Linux (instalasi, `lampp security`): `apachefriends.org/faq_linux.html`
- MariaDB Knowledge Base: `mariadb.com/kb`
- Mermaid (sintaks diagram): `mermaid.js.org`
- Certbot (`--server`, `--webroot`, deploy hook): `eff-certbot.readthedocs.io`
- Postfix TLS dan SASL: `postfix.org/TLS_README.html`, `postfix.org/SASL_README.html`
- Dovecot SSL: `doc.dovecot.org`
- Roundcube (instalasi di Debian/Ubuntu): `roundcube.net`
- Fail2ban: `github.com/fail2ban/fail2ban/wiki`
- RFC 8555 (ACME), RFC 5280 (X.509), RFC 4033–4035 (DNSSEC), RFC 7858 (DoT), RFC 8659 (CAA)

> **Catatan:** perintah dan opsi bisa berubah antar versi. Selalu cocokkan dengan dokumentasi resmi versi yang terpasang di VPS, dan catat versi yang dipakai (`step version`, `named -v`, `/opt/lampp/bin/httpd -v`, `postconf mail_version`, `dovecot --version`) di laporan.
