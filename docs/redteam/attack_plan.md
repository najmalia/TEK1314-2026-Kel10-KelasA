# Attack Plan — Fase 2 (Target Node: PiRoTech Web Monitoring, 192.168.10.5)

## Skenario 1 — Web/API: Akses tanpa autentikasi pada `/api/users`

- **Kategori:** Web/API/Protokol IoT
- **Tools:** `curl`, Burp Suite/OWASP ZAP (opsional untuk eksplorasi lanjutan)
- **Target & celah:** Endpoint `GET/POST http://192.168.10.5:3000/api/users` memanggil Firebase Admin SDK tanpa memverifikasi token sesi. Siapa pun di jaringan lab dapat membaca seluruh daftar user (email, role, username) atau membuat akun baru dengan `role: admin`.
- **Hasil yang diharapkan:** Dari Kali, `curl http://192.168.10.5:3000/api/users` mengembalikan daftar user termasuk akun admin tanpa login — membuktikan broken access control (mirip OWASP API1:2023 Broken Object Level Authorization / Missing Function-Level Access Control).
- **Relevansi tech stack:** Celah ini ada pada kode aplikasi replika yang identik dengan rencana arsitektur API asli (Next.js API routes + Firebase Admin SDK), sehingga temuan berlaku langsung untuk sistem produksi.

## Skenario 2 — Web/API: Penulisan data sensor tanpa autentikasi (RTDB write terbuka)

- **Kategori:** Web/API/Protokol IoT
- **Tools:** `curl` / `mosquitto_pub`-style HTTP injection ke REST API emulator, MQTT Explorer (jika protokol MQTT ikut direplikasi)
- **Target & celah:** `firebase.rules.json` node `sensor_data`, `monitoring`, dan `contact/messages` memiliki `.write: true` tanpa syarat autentikasi. Node `control` memiliki `.read: true` terbuka publik.
- **Hasil yang diharapkan:** `curl -X PUT -d '{"palsu":true}' http://192.168.10.5:9000/sensor_data/test.json` berhasil menyuntik data sensor palsu tanpa login, membuktikan spoofing data telemetri yang bisa menyesatkan operator dashboard.
- **Relevansi tech stack:** Node ini meniru pola tulis langsung dari firmware ESP32 ke Realtime Database tanpa device-auth — risiko nyata jika direplikasi di sistem produksi tanpa least-privilege rule per perangkat.

## Skenario 3 — Reconnaissance & Exploitation: Brute-force SSH

- **Kategori:** Exploitation/Akses (didahului Reconnaissance)
- **Tools:** `nmap` (scanning awal, sudah dijalankan), `Hydra`
- **Target & celah:** Port 22/tcp terbuka ke seluruh subnet lab. Sebelum hardening, autentikasi password diaktifkan tanpa rate-limit.
- **Hasil yang diharapkan:** `hydra -l malmal -P wordlist.txt ssh://192.168.10.5` — pada baseline (sebelum hardening) berhasil menemukan kredensial; setelah hardening (key-only auth + fail2ban), percobaan diblokir dan dicatat sebagai alert.
- **Catatan:** Skenario ini dipakai sebagai pembanding before/after untuk kontrol hardening A1/A2 di `hardening_checklist.md`, sekaligus skenario Purple Team Loop pertama yang sudah melalui Tahap 1–5 secara lengkap.
