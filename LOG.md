# LOG Pengerjaan — Private PKI + Full HTTPS Internal

Catatan progress sesuai NFR-05: setiap perubahan konfigurasi penting dicatat di sini.
Format: tanggal — inisial (A/B/C) — fase — apa yang dilakukan — hasil/verifikasi.

| Tanggal | Oleh | Fase | Perubahan | Hasil / verifikasi |
|---|---|---|---|---|
| 2026-11-25 | — | setup | Inisialisasi repo Git, `.gitignore`, struktur folder progress | repo lokal siap, commit pertama |

<!--
Contoh entri berikutnya:
| 2026-11-26 | B | F1 | Konfigurasi zona kelompok.internal + resolver lokal | dig www.kelompok.internal +short -> IP VPS |
| 2026-11-26 | A | F2 | step ca init, Root dipindah offline, provisioner ACME | /health = ok, /acme/.../directory dapat diakses |
-->
