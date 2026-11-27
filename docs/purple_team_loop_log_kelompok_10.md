# Purple Team Loop Log — Kelompok 10 (PiRoTech)

**Mata Kuliah:** TEK1314 — Keamanan Siber (D4 Teknologi Rekayasa Komputer, SV IPB)  
**Objek Pengujian:** Replika Backend & Web Dashboard Monitoring IoT PiRoTech (*Plastic Pyrolysis Technology*)  
**Lingkup:** Jaringan lab virtual VirtualBox (`192.168.10.0/24`) menggunakan data dummy/sintetis (tidak terhubung ke sistem produksi Mitra).

| Peran | Anggota | Hostname / IP Lab |
| :--- | :--- | :--- |
| **Lead** | Najmalia Nur Azizah (J0404241018) | Koordinasi, verifikasi konsistensi, & pelaporan |
| **Red Team (Attacker)** | Wira Setyo Adi (J0404241049) | `ATK-KEL10` (`192.168.10.100`, Kali Linux) |
| **Blue Team (Defender/Network)** | Muhammad Akmal Muzhafar (J0404241076) & Muhammad Bahrain Sekai Suryawi (J0404241171) | `serverpjblsiber` (`192.168.10.5`, Ubuntu Server 26.04) & `SOC-KEL10` (`192.168.10.200`, CyberOps Security Onion) |

---

## Ringkasan Siklus Purple Team Loop (Tahap 1–5)

| Skenario | Tahap 1: Serangan Awal (Siklus 1) | Tahap 2–3: Deteksi & Analisis | Tahap 4: Mitigasi Blue Team | Tahap 5: Verifikasi Ulang (Siklus 2) | Status Akhir |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Reconnaissance** | `nmap -sV -sC -p- -T4 192.168.10.5` menemukan 4 port terbuka (`22, 3000, 9000, 9099`); 65.531 port `filtered` oleh UFW. | Lonjakan paket probe TCP (`SYN`/`RST`) dan *fingerprinting* HTTP port `3000` terekam di Wireshark `SOC-KEL10` (`eth1`, filter `ip.addr == 192.168.10.100`). | Pertahankan UFW *default-deny* dan *whitelist* 4 port layanan lab. | Pemindaian ulang selesai dalam `360,15 detik`; tetap hanya 4 port terbuka, trafik probe penuh terekam di Wireshark. | **Terkendali & Terpantau** |
| **Skenario 1 (`/api/users`)** | `GET` bocorkan akun admin `pirotechsvipb@gmail.com` (`200 OK`); `POST` buat akun admin ilegal `redteam_pwn` (`201 Created`). | Permintaan HTTP ke `:3000` tampak seperti trafik web normal tanpa alert NIDS bawaan (*blind spot*), namun terekam pada *packet capture* Wireshark. Akar masalah: Firebase Admin SDK dipanggil tanpa verifikasi token. | Hapus akun ilegal `redteam_pwn`; tambah verifikasi Firebase ID Token & pengecekan peran `admin` pada seluruh metode API (`/api/users`). | `GET`, `POST`, `PATCH`, dan `DELETE` seluruhnya ditolak **`HTTP/1.1 401 Unauthorized`** (`{"error":"Unauthorized"}`), terkonfirmasi di terminal Kali dan Wireshark. | **Mitigasi Berhasil** |
| **Skenario 2 (Firebase RTDB `:9000`)** | `PUT /sensor_data/redteam_spoof.json` sukses menyuntikkan suhu palsu `999.9` (`200 OK`) akibat `.write: true`. | Trafik HTTP `PUT` ke `:9000` terekam di Wireshark, tidak otomatis memicu alert NIDS. Akar masalah: aturan `firebase.rules.json` membuka akses tulis anonim. | Hapus data palsu; perketat `firebase.rules.json` pada namespace aplikasi (`demo-pirotech-default-rtdb`) agar mewajibkan autentikasi. | **C1 (DB Aplikasi `?ns=demo-pirotech-default-rtdb`):** ditolak **`HTTP/1.1 401 Unauthorized`** (`Permission denied`), terekam di Wireshark.<br>**C2 (Tanpa `?ns=`):** `200 OK` hanya masuk ke namespace dummy `fake-server` (keterbatasan arsitektur emulator, bukan DB PiRoTech). | **Mitigasi Berhasil (DB Aplikasi)** |
| **Skenario 3 (Brute-Force SSH & Web Login)** | `Hydra` dengan `wordlist_kecil.txt` ke user `malmal` (`:22`) langsung ditolak (`method reply 4`) berkat hardening awal SSH *key-only*. | Pertukaran banner `SSHv2` (`libssh_0.11.3` ↔ `OpenSSH_10.2p1`) terekam di Wireshark; pada hardening awal `fail2ban` mode standar belum mem-ban karena negosiasi langsung gagal di awal. | Pertahankan `PermitRootLogin no`, `PasswordAuthentication no`, `MaxAuthTries 3`; aktifkan **`mode = aggressive`** pada `/etc/fail2ban/jail.local` (`maxretry = 4`, `bantime = 1h`). | `Hydra` SSH tetap ditolak (`method reply 4`) dan **`fail2ban` otomatis memblokir `192.168.10.100`** (`Total failed: 4`, `Currently banned: 1`). Uji tambahan brute-force login web `:9099` (5 password) seluruhnya gagal `INVALID_PASSWORD` (`400 Bad Request`). | **Mitigasi Berhasil & Terblokir** |

---

## Log Detail Pelaksanaan Per Skenario

### 0. Tahap Reconnaissance (`nmap`)
* **Siklus 1 (`25 Nov 2026 17:54:16 GMT` / `26 Nov 00:54:16 WIB`):**
  * Verifikasi konektivitas dari `ATK-KEL10` (`eth1: 192.168.10.100`) ke `192.168.10.5` menggunakan `ip a && ping -c 4 192.168.10.5` (`0% packet loss`, rata-rata `0,360 ms`).
  * Perintah: `nmap -sV -sC -p- -T4 192.168.10.5 -oN 01_nmap_scan.txt`
  * Hasil: `65531 filtered tcp ports (no-response)` (membuktikan UFW *default-deny* aktif), 4 port `open`: `22/tcp` (`OpenSSH 10.2p1 Ubuntu 2ubuntu3.8`), `3000/tcp` (`X-Powered-By: Next.js`), `9000/tcp` (Firebase RTDB Emulator), dan `9099/tcp` (Firebase Auth Emulator).
* **Deteksi Security Onion (`SOC-KEL10`):**
  * Pemantauan dilakukan melalui **Wireshark** pada antarmuka `eth1` (*Promiscuous Mode*) dengan filter `ip.addr == 192.168.10.100`.
  * Terekam lonjakan ribuan paket probe TCP berurutan (1.713–2.627 paket pada sesi awal; ratusan ribu paket selama `360,15 detik` pada pemindaian ulang Siklus 2 mulai `26 Nov 2026 19:21:55 UTC`) beserta *fingerprinting* HTTP ke port `3000` (`400 Bad Request`, `200 OK`, `405 Method Not Allowed`).

---

### 1. Skenario 1 — Web/API: Akses Tanpa Autentikasi pada `/api/users`

#### Tahap 1 — Serangan Awal (Siklus 1)
* **Waktu:** 25 November 2026 pukul `18:04:00 GMT` (`GET`) dan `18:04:47 GMT` (`POST`) / 26 November 2026 pukul `01:04 WIB`.
* **Perintah Red Team:**
  ```bash
  curl -s -i http://192.168.10.5:3000/api/users | tee 02a_api_users_get.txt
  curl -s -i -X POST http://192.168.10.5:3000/api/users \
    -H "Content-Type: application/json" \
    -d '{"email":"redteam@attacker.local","username":"redteam_pwn","password":"Password123!","role":"admin"}' \
    | tee 02b_api_users_post.txt
  ```
* **Hasil Aktual:**
  * `GET` dibalas **`HTTP/1.1 200 OK`** dan membocorkan akun admin: `uid: 9yVBB7ALPhZuTDwivLLqZKFnJHYI`, `email: pirotechsvipb@gmail.com`, `username: admin`, `role: admin`.
  * `POST` dibalas **`HTTP/1.1 201 Created`** (`{"message":"Akun berhasil dibuat!","uid":"gTONCBGUy4sRfmNoCokvZD9JXMQI"}`), berhasil membuat akun admin ilegal `redteam_pwn`.

#### Tahap 2 & 3 — Deteksi & Analisis Akar Masalah
* **Deteksi:** Lalu lintas HTTP ke port `3000` tidak memicu alert NIDS bawaan karena secara sintaksis tampak seperti permintaan aplikasi normal, namun pasangan paket HTTP terekam di Wireshark `SOC-KEL10`.
* **Akar Masalah:** Handler Next.js API route `/api/users` langsung memanggil **Firebase Admin SDK** (yang melewati aturan keamanan Firebase) tanpa memverifikasi header `Authorization` (Firebase ID Token) maupun memeriksa hak akses `admin`.

#### Tahap 4 — Mitigasi Blue Team (Hardening Kedua)
* **Containment:** Menghapus akun admin ilegal `redteam_pwn` (`redteam@attacker.local`, UID `gTONCBGUy4sRfmNoCokvZD9JXMQI`) dari Firebase Auth Emulator.
* **Eradication:** Memperbarui handler endpoint `/api/users` (mencakup metode `GET`, `POST`, `PATCH`, dan `DELETE`) untuk mewajibkan verifikasi Firebase ID Token dan pengecekan peran `admin` sebelum memanggil Firebase Admin SDK, serta mengembalikan `HTTP 401 Unauthorized` jika token tidak ada atau tidak valid.
* **Recovery:** Memulai ulang layanan Next.js pada port `3000` dan memastikan akses admin sah tetap berjalan normal.

#### Tahap 5 — Verifikasi Ulang (Siklus 2)
* **Waktu:** 26 November 2026 pukul `19:29:19 UTC` (`27 Nov 02:29:19 WIB`).
* **Perintah Uji Ulang Red Team (`~/redteam-siklus2`):**
  * `GET /api/users` (`02a_api_users_get.txt`) → **`HTTP/1.1 401 Unauthorized`** (`{"error":"Unauthorized"}`)
  * `POST /api/users` (`02b_api_users_post.txt`) → **`HTTP/1.1 401 Unauthorized`** (`{"error":"Unauthorized"}`)
  * `PATCH /api/users/dummy-uid` (`02c_api_users_patch.txt`) → **`HTTP/1.1 401 Unauthorized`** (`{"error":"Unauthorized"}`)
  * `DELETE /api/users/dummy-uid` (`02d_api_users_delete.txt`) → **`HTTP/1.1 401 Unauthorized`** (`{"error":"Unauthorized"}`)
* **Bukti Monitoring:** Wireshark pada `SOC-KEL10` merekam keempat permintaan dari `192.168.10.100` ke `192.168.10.5:3000` beserta balasan `HTTP/1.1 401 Unauthorized (application/json)`.

---

### 2. Skenario 2 — Web/API/IoT: Penulisan Data Sensor Tanpa Autentikasi (Firebase RTDB)

#### Tahap 1 — Serangan Awal (Siklus 1)
* **Waktu:** 26 November 2026 pukul `01:05 WIB` (payload timestamp `2026-11-26T01:05:00Z`).
* **Perintah Red Team:**
  ```bash
  curl -s -i "http://192.168.10.5:9000/.json?shallow=true" | tee 03a_rtdb_read.txt
  curl -s -i -X PUT "http://192.168.10.5:9000/sensor_data/redteam_spoof.json" \
    -H "Content-Type: application/json" \
    -d '{"suhu":999.9,"tekanan":99.9,"status":"BAHAYA_SPOOFED_BY_REDTEAM","timestamp":"2026-11-26T01:05:00Z"}' \
    | tee 03b_rtdb_spoof_write.txt
  ```
* **Hasil Aktual:** Kedua perintah dibalas **`HTTP/1.1 200 OK`**, membuktikan data sensor palsu berhasil disuntikkan ke RTDB Emulator tanpa autentikasi.

#### Tahap 2 & 3 — Deteksi & Analisis Akar Masalah
* **Deteksi:** Permintaan `PUT` ke port `9000` terekam di Wireshark `SOC-KEL10` sebagai trafik HTTP biasa tanpa alert NIDS bawaan.
* **Akar Masalah:** Berkas `firebase.rules.json` menetapkan `.write: true` tanpa autentikasi pada node `sensor_data`, `monitoring`, dan `contact/messages` (serta `.read: true` pada `control`) untuk meniru penulisan langsung dari ESP32 tanpa *device-auth*.

#### Tahap 4 — Mitigasi Blue Team (Hardening Kedua)
* **Containment:** Menghapus data telemetri palsu pada `/sensor_data/redteam_spoof.json`.
* **Eradication:** Memperketat `firebase.rules.json` pada namespace aplikasi (`demo-pirotech-default-rtdb`) dengan menghapus izin tulis bebas (`.write: true`) dan mewajibkan autentikasi pengguna/perangkat agar permintaan anonim ditolak.
* **Recovery:** Memuat ulang aturan pada Firebase RTDB Emulator (`:9000`) dan memverifikasi integritas data dashboard.

#### Tahap 5 — Verifikasi Ulang (Siklus 2)
* **Waktu:** 26 November 2026 pukul `19:29:49 UTC` (`27 Nov 02:29:49 WIB`).
* **Pengujian C1 — Namespace Asli Aplikasi (`?ns=demo-pirotech-default-rtdb`):**
  ```bash
  curl -s -i -X PUT "http://192.168.10.5:9000/sensor_data/redteam_spoof_s2.json?ns=demo-pirotech-default-rtdb" \
    -H "Content-Type: application/json" \
    -d '{"suhu":999.9,"tekanan":99.9,"status":"SPOOFED_SIKLUS_2","timestamp":"2026-11-26T10:15:00Z"}' \
    | tee 03a_rtdb_spoof_with_ns.txt
  ```
  * **Hasil:** Ditolak **`HTTP/1.1 401 Unauthorized`** dengan body `{"error": "Permission denied"}` dan header `X-Firebase-Project-Id: demo-pirotech-default-rtdb`. Paket `PUT` dan balasan `401 Unauthorized` terekam di Wireshark.
* **Pengujian C2 — Pembanding Artefak Emulator Tanpa `?ns=`:**
  ```bash
  curl -s -i -X PUT "http://192.168.10.5:9000/sensor_data/redteam_spoof_no_ns.json" \
    -H "Content-Type: application/json" \
    -d '{"suhu":999.9,"status":"SPOOFED_NO_NS"}' \
    | tee 03b_rtdb_spoof_no_ns.txt
  ```
  * **Hasil & Analisis Teknis:** Mengembalikan `HTTP/1.1 200 OK`, tetapi header respons menunjukkan **`X-Firebase-Project-Id: fake-server`**. Hal ini merupakan keterbatasan arsitektural bawaan Firebase RTDB Emulator (saat diakses tanpa parameter `?ns=`, emulator mengarahkan permintaan ke namespace dummy `fake-server` yang tidak memuat `firebase.rules.json` proyek). Data ini tidak masuk ke database aplikasi PiRoTech maupun dashboard, dan tidak terjadi pada layanan Firebase Cloud produksi karena setiap URL produksi terikat langsung pada nama proyeknya.

---

### 3. Skenario 3 — Exploitation: Brute-Force SSH & Login Web

#### Tahap 1 — Serangan Awal (Pasca Hardening Baseline)
* **Waktu:** 26 November 2026 pukul `02:15:59` (`04a_hydra_ssh.txt`).
* **Perintah Red Team:** `hydra -l malmal -P wordlist_kecil.txt -t 4 -V ssh://192.168.10.5` (10 kata sandi).
* **Hasil Aktual:** Serangan langsung terhenti pada percobaan pertama dengan galat `[ERROR] target ssh://192.168.10.5:22/ does not support password authentication (method reply 4)`.

#### Tahap 2 & 3 — Deteksi & Analisis
* **Deteksi:** Wireshark pada `SOC-KEL10` merekam pertukaran protokol **`SSHv2`** antara `192.168.10.100` (`SSH-2.0-libssh_0.11.3`) dan `192.168.10.5:22` (`SSH-2.0-OpenSSH_10.2p1 Ubuntu-2ubuntu3.8`).
* **Temuan Tuning Deteksi:** Karena `PasswordAuthentication no` langsung memutus sesi pada tahap negosiasi metode autentikasi, filter bawaan `fail2ban` mode normal tidak langsung menghitung koneksi putus dini tersebut sebagai kegagalan yang memicu ban IP.

#### Tahap 4 — Mitigasi & Penguatan Blue Team
* Mempertahankan hardening awal SSH (`PermitRootLogin no`, `PasswordAuthentication no`, `MaxAuthTries 3`) dan UFW *default-deny*.
* Mengonfigurasi ulang `/etc/fail2ban/jail.local` pada jail `sshd` dengan **`mode = aggressive`**, `backend = systemd`, `journalmatch = _SYSTEMD_UNIT=ssh.service`, `maxretry = 4`, `findtime = 10m`, `bantime = 1h`, dan `ignoreip = 127.0.0.1/8 192.168.10.1`, dilanjutkan `sudo systemctl restart fail2ban`.

#### Tahap 5 — Verifikasi Ulang (Siklus 2)
* **Uji SSH (`Hydra`, 6 password, `04a_hydra_ssh.txt`):** Tetap ditolak dengan `does not support password authentication (method reply 4)`, terekam sebagai paket `SSHv2` di Wireshark.
* **Bukti Pemblokiran `fail2ban` (`~/evidence/s3_fail2ban_status.txt`):**
  ```text
  Status for the jail: sshd
  |- Filter
  |  |- Currently failed: 1
  |  |- Total failed:     4
  |  `- Journal matches:  _SYSTEMD_UNIT=ssh.service
  `- Actions
     |- Currently banned: 1
     |- Total banned:     1
     `- Banned IP list:   192.168.10.100
  ```
* **Uji Tambahan — Brute-Force Login Web Auth Emulator (`:9099`, `04b_web_bruteforce.txt`):**
  * Red Team menjalankan 5 percobaan `POST /identitytoolkit.googleapis.com/v1/accounts:signInWithPassword` terhadap akun `pirotechsvipb@gmail.com` (`123456`, `admin`, `pirotech123`, `Password123!`, `malmal`).
  * Seluruh 5 percobaan gagal dengan `code: 400` (`INVALID_PASSWORD`), dan 5 pasang transaksi `POST` → `HTTP/1.1 400 Bad Request` terekam utuh di Wireshark `SOC-KEL10`.