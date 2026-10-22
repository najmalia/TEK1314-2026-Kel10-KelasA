# Deklarasi Tech Stack — PiRoTech Web Monitoring

| Komponen | Teknologi pada Proyek IoT Asli | Direplikasi sebagai (Target Node) |
|---|---|---|
| Sistem Operasi backend | Serverless Cloud Environment (Vercel & Google Cloud Managed Linux) | Ubuntu Server 26.04 (VirtualBox VM, `serverpjblsiber`, 192.168.10.5) |
| Database/penyimpanan data | Firebase Realtime Database (Google Cloud, project produksi) | Firebase Realtime Database **Emulator** (firebase-tools 15.34.1, lokal di Target Node, tanpa koneksi ke project cloud asli) |
| Protokol komunikasi sensor | HTTP REST (ESP32 → Firebase REST API) | HTTP REST ke emulator Database (port 9000), skema node sama persis dengan `firebase.rules.json` produksi |
| Dashboard/monitoring | Next.js (App Router) + Firebase Auth (di-hosting di Vercel) | Next.js 16.3.0 + Firebase Auth **Emulator** (port 9099), di-build dan dijalankan langsung di Target Node via PM2 (`npm run build` lalu `next start -H 0.0.0.0`, port 3000) |

**Catatan replikasi:**
- Seluruh kredensial dan data pada Target Node bersifat dummy/sintetis: akun admin dibuat lewat skrip seed (`tools/seedEmulator.mjs`) dengan email dan password lab, bukan kredensial Mitra.
- Aturan akses (`firebase.rules.json`) yang dipakai emulator adalah salinan identik dari aturan proyek asli, sehingga perilaku otorisasi (role admin/operator, custom claims) tereplikasi secara akurat.
- Firmware ESP32 asli (`docs/pirotech_fixed.ino`) tidak dijalankan secara fisik di lab ini; simulasi traffic sensor dilakukan lewat request HTTP manual/skrip ke endpoint yang sama.
- Tidak ada koneksi keluar dari Target Node ke project Firebase cloud asli selama simulasi — emulator berjalan 100% lokal (`demo-pirotech`, project ID dummy).
