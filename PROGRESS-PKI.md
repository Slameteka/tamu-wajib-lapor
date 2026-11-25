# Progress — Peran A (PKI): step-ca, ACME, mTLS, Revocation

Branch: `PKI` · Peran A · Fase 2 & Fase 4 (PRD §4)

Checklist di bawah ditandai berdasarkan bukti nyata dari sesi pengerjaan.
Legenda: `[x]` selesai & terverifikasi · `[~]` sebagian / sedang berjalan · `[ ]` belum.

## Fase 2 — Certificate Authority (step-ca) ✅ hampir selesai

- [x] **Instal step CLI + step-ca, inisialisasi CA, systemd service**
  - [x] 2.1 `step-cli 0.31.0` + `step-ca 0.30.2` terpasang (via `scp`, unduhan VPS diblokir)
  - [x] 2.2 User `step` dan direktori `/etc/step-ca`
  - [x] 2.3 Password CA tersimpan di `password.txt` (izin `600`)
  - [x] 2.4 `step ca init`: Root + Intermediate terbentuk, `ca.json` ada
  - [x] 2.5 Service systemd `step-ca` jalan — log `Serving HTTPS on :9000`, `/health` = `ok`
- [~] **Pindahkan kunci Root CA ke penyimpanan offline** (lalu `shred` dari server)
  - [x] 2.6 Root key dibackup ke laptop (2 salinan, hash cocok `0ea6619e...ea79`)
  - [ ] 2.6 `shred` kunci Root di VPS — **belum terkonfirmasi**
    (cek: `sudo ls -l /etc/step-ca/secrets` tinggal `intermediate_ca_key`,
    dan `~/root_ca_key.tmp` sudah tidak ada)
- [x] **Tambah provisioner ACME, atur durasi sertifikat**
  - [x] 2.7 Provisioner ACME aktif
  - [x] 2.8 Fingerprint Root dicatat: `b73c4097...b093c0`

## Fase 4 — mTLS & Revocation

- [ ] **mTLS di `admin`**: penerbitan sertifikat klien (`.crt`, `.key`, `.p12`)
- [ ] **Revocation** (CRL / daftar serial) beserta buktinya

## Tugas bersama (A + B)

- [ ] **Peta integrasi** dan skrip `scripts/cek-semua.sh` (Fase 9) — dikerjakan bersama B
- [ ] **Bagian laporan**: PKI, ACME, mTLS, revocation

---

## Catatan status (sesi 2026-11-25)

- Fase 2 (step-ca) praktis selesai: CA aktif di `:9000`, `/health = ok`, ACME aktif.
- **Satu-satunya sisa Fase 2:** konfirmasi `shred` Root key di VPS. Backup offline sudah ada
  (2 salinan di laptop, hash cocok), jadi aman untuk di-`shred` setelah dicek.
  Verifikasi: `/etc/step-ca/secrets` hanya menyisakan `intermediate_ca_key`.
- Fingerprint Root CA: `b73c4097...b093c0` (untuk distribusi/trust store).
- Lanjutan berikutnya: Fase 4 — mTLS pada vhost `admin` + penerbitan cert klien, lalu revocation/CRL.

> ⚠️ Jangan commit kunci privat / password (`password.txt`, `*_ca_key`, `.p12`). Lihat `.gitignore`.
> Perbarui centang di atas saat langkah terverifikasi, dan catat di `LOG.md`.
