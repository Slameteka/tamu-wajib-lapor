# LOG Pengerjaan — Private PKI + Full HTTPS Internal

Catatan progress sesuai NFR-05: setiap perubahan konfigurasi penting dicatat di sini.
Format: tanggal — inisial (A/B/C) — fase — apa yang dilakukan — hasil/verifikasi.

| Tanggal | Oleh | Fase | Perubahan | Hasil / verifikasi |
|---|---|---|---|---|
| 2026-11-25 | — | setup | Inisialisasi repo Git, `.gitignore`, struktur folder progress | repo lokal siap, commit pertama |
| 2026-11-25 | A | F2 | Instal step-cli + step-ca di VPS (via .deb/scp), Root CA tersedia; `PROGRESS-PKI.md` ditambahkan | paket terpasang; init CA/systemd/ACME/offline belum terverifikasi |
| 2026-11-25 | A | F2 | Fase 2 selesai: `step ca init` (Root+Intermediate), systemd `step-ca` jalan, provisioner ACME aktif, Root dibackup offline | `/health`=ok, `Serving HTTPS on :9000`, fingerprint Root `b73c4097...b093c0`; sisa: konfirmasi `shred` Root key di VPS |
| 2026-11-25 | A | F2 | Konfirmasi `shred` Root key di VPS — Fase 2 TUNTAS penuh (2.1–2.8) | `/etc/step-ca/secrets` tinggal `intermediate_ca_key`; Root key hanya ada offline di laptop |

<!--
Contoh entri berikutnya:
| 2026-11-26 | B | F1 | Konfigurasi zona kelompok.internal + resolver lokal | dig www.kelompok.internal +short -> IP VPS |
| 2026-11-26 | A | F2 | step ca init, Root dipindah offline, provisioner ACME | /health = ok, /acme/.../directory dapat diakses |
-->
