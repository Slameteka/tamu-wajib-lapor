# Progress — Peran A (PKI): step-ca, ACME, mTLS, Revocation

Branch: `PKI` · Peran A · Fase 2 & Fase 4 (PRD §4)

Checklist di bawah ditandai berdasarkan bukti nyata dari sesi pengerjaan.
Legenda: `[x]` selesai & terverifikasi · `[~]` sebagian / sedang berjalan · `[ ]` belum.

## Fase 2 — Certificate Authority (step-ca)

- [~] **Instal step CLI + step-ca, inisialisasi CA, systemd service**
  - [x] Paket `step-cli` + `step-ca` terpasang di VPS (via `.deb`, di-`scp` dari laptop)
  - [ ] `step ca init` (Root + Intermediate + config) — belum ada bukti output
  - [ ] Service `systemd` (`step-ca.service`) — belum ada bukti `active (running)`
  - _Catatan: Root CA sudah dirujuk di `/etc/step-ca/certs/root_ca.crt`._
- [ ] **Pindahkan kunci Root CA ke penyimpanan offline** (lalu `shred` dari server)
- [ ] **Tambah provisioner ACME, atur durasi sertifikat**
  - [ ] verifikasi `/health` = ok
  - [ ] verifikasi endpoint `/acme/.../directory` dapat diakses
  - [ ] catat fingerprint Root CA

## Fase 4 — mTLS & Revocation

- [ ] **mTLS di `admin`**: penerbitan sertifikat klien (`.crt`, `.key`, `.p12`)
- [ ] **Revocation** (CRL / daftar serial) beserta buktinya

## Tugas bersama (A + B)

- [ ] **Peta integrasi** dan skrip `scripts/cek-semua.sh` (Fase 9) — dikerjakan bersama B
- [ ] **Bagian laporan**: PKI, ACME, mTLS, revocation

---

## Catatan status (dari sesi pengerjaan terakhir)

- step-ca & step CLI **sudah terpasang** di VPS; `.deb` berhasil ditransfer via `scp`.
- Root CA sudah ada (`/etc/step-ca/certs/root_ca.crt` dirujuk saat pasang ke trust store).
- Langkah `init CA`, `systemd service`, `Root offline`, dan `provisioner ACME`
  **belum ada bukti penyelesaian** pada sesi ini, sehingga ditandai belum / sebagian.
- Prasyarat: nama `ca.kelompok.internal` harus dapat di-resolve (tergantung Fase 1 / Bind9 peran B);
  sementara bisa ditambahkan di `/etc/hosts` VPS: `127.0.0.1 ca.kelompok.internal`.

> ⚠️ Jangan commit kunci privat / password. Lihat `.gitignore`.
> Perbarui centang di atas saat langkah terverifikasi, dan catat di `LOG.md`.
