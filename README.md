# Private PKI + Full HTTPS Internal

Repositori **progress dan konfigurasi** proyek Administrasi Server: membangun *Certificate Authority* (CA) sendiri di Ubuntu Server 24.04, lalu mengamankan semua layanan internal (web, admin panel, mail, webmail, DNS) dengan sertifikat dari CA itu secara otomatis (ACME), dilengkapi mTLS, DNSSEC, DoT, CAA, dan pencabutan sertifikat.

> ⚠️ **Keamanan:** repo ini **tidak boleh** memuat kunci privat atau password. Lihat `.gitignore`. Rujukan PRD: FR-24, FR-70, NFR-01, Risiko R2.

## Dokumen utama

| Dokumen | Isi |
|---|---|
| [`PRD-Private-PKI-Full-HTTPS-Internal.md`](./PRD-Private-PKI-Full-HTTPS-Internal.md) | Product Requirements: tujuan, kebutuhan fungsional (FR), non-fungsional (NFR), metrik, risiko |
| [`Proyek-Private-PKI-Roadmap-dan-Panduan.md`](./Proyek-Private-PKI-Roadmap-dan-Panduan.md) | Roadmap, panduan implementasi Fase 0–9, matriks uji T01–T24, troubleshooting |
| [`LOG.md`](./LOG.md) | Catatan progress harian (siapa, kapan, apa) — NFR-05 |

## Struktur repo

```
.
├── PRD-*.md, Roadmap-*.md   # dokumen proyek
├── README.md, LOG.md
├── .gitignore               # pelindung rahasia (env, *.key, password, dll.)
├── configs/                 # salinan file konfigurasi (TANPA kunci/password)
│   ├── bind9/  step-ca/  xampp-apache/  postfix-dovecot/
│   ├── roundcube/  fail2ban/  systemd/
├── bukti/                   # screenshot bukti per fase (F0–F9)
├── logs/                    # log yang boleh dibagikan (log mentah besar di-ignore)
├── scripts/                 # skrip bantu (cek-semua.sh, deploy hook, dll.)
├── pengujian/               # matriks uji terisi, hasil
└── laporan/                 # laporan + slide
```

## Tim & peran (dari PRD §4)

| Peran | Fokus |
|---|---|
| A — PKI & Web | step-ca, ACME, certbot, Apache HTTPS, mTLS, revocation |
| B — DNS & Infra | Bind9, DNSSEC, DoT, CAA, XAMPP/MariaDB, Fail2ban, UFW |
| C — Mail & Dok | Postfix, Dovecot, Roundcube, laporan, slide, pengujian |

## Progress fase (ceklis cepat)

- [ ] Fase 0 — Baseline VPS, chrony, UFW
- [ ] Fase 1 — Bind9 dasar
- [ ] Fase 2 — step-ca (Root + Intermediate + ACME)
- [ ] Fase 3 — XAMPP + MariaDB, certbot, HTTPS (M1 gembok hijau)
- [ ] Fase 4 — mTLS + revocation
- [ ] Fase 5 — DNSSEC, DoT, CAA
- [ ] Fase 6 — Mail TLS (Postfix + Dovecot)
- [ ] Fase 7 — Roundcube + MariaDB
- [ ] Fase 8 — Fail2ban + hardening
- [ ] Fase 9 — Integrasi, pengujian T01–T24, laporan, demo

## Aturan commit konfigurasi

1. Salin file konfigurasi ke `configs/<layanan>/` **setelah** membersihkan rahasia.
2. Pesan commit jelas, sebutkan inisial: `git commit -m "B: aktifkan DNSSEC zona kelompok.internal"`.
3. **Jangan pernah** `git add` file `*.key`, `*.p12`, `password.txt`, `*.sql`. `.gitignore` sudah memblokirnya, tetapi tetap periksa dengan `git status` sebelum commit.
4. Catat setiap perubahan penting di `LOG.md`.
