# TEK1314 — Keamanan Siber (Kelompok 10 / Kelas A)

Proyek PBL **Simulated SOC Engagement & Purple Team Loop** untuk mengamankan infrastruktur backend dan web dashboard monitoring sistem IoT **PiRoTech (Plastic Pyrolysis Technology)** pada lingkungan lab virtual terisolasi.

> **Batasan Ruang Lingkup:** Seluruh pengujian, simulasi serangan, analisis log, dan rekomendasi dilakukan sepenuhnya pada **replika lokal** menggunakan **data dummy/sintetis** di subnet lab `192.168.10.0/24`, tanpa terhubung maupun menyentuh sistem produksi milik Mitra.

---

## Anggota Kelompok 10

| Peran | Nama | NIM |
| :--- | :--- | :--- |
| **Lead** | Najmalia Nur Azizah | J0404241018 |
| **Red Team (Attacker)** | Wira Setyo Adi | J0404241049 |
| **Blue Team (Defender/Network)** | Muhammad Akmal Muzhafar | J0404241076 |
| **Blue Team (Defender/Network)** | Muhammad Bahrain Sekai Suryawi | J0404241171 |

---

## Arsitektur & IP Plan Lab

* **Segmen Jaringan:** `192.168.10.0/24` (flat, satu broadcast domain)
* **Mode Network (VirtualBox):** Host-only Adapter / Internal Network (*Promiscuous Mode: Allow VMs* pada Monitoring Node)

| Hostname | IP Address | OS & Layanan | Peran |
| :--- | :--- | :--- | :--- |
| `ATK-KEL10` | `192.168.10.100` | Kali Linux (`nmap`, `curl`, `Hydra`, Burp Suite) | Attacker Node (Red Team) |
| `serverpjblsiber` | `192.168.10.5` | Ubuntu Server 26.04 — PiRoTech Web Next.js (`:3000`), Firebase RTDB Emulator (`:9000`), Firebase Auth Emulator (`:9099`), OpenSSH (`:22`) | Target Node (Blue Team) |
| `SOC-KEL10` | `192.168.10.200` | CyberOps Security Onion (Suricata/Sguil, Kibana, Wireshark) | Monitoring Node (Blue Team) |

---

## Ringkasan Akhir Proyek (Fase 1–3)

1. **Fase 1 (Desain & Replika):** Membangun topologi lab 3-node di VirtualBox dan mereplikasi layanan backend PiRoTech (Next.js + Firebase RTDB & Auth Emulator) secara lokal.
2. **Fase 2 (Hardening Baseline & Simulasi Purple Team Loop):**
   * **Hardening Awal:** Menutup login root & password SSH (`PasswordAuthentication no`, `PermitRootLogin no`, `MaxAuthTries 3`), memasang `fail2ban`, mengaktifkan UFW *default-deny* (hanya membuka port `22, 3000, 9000, 9099`), memperbarui paket OS, dan mengamankan *environment variables*.
   * **Skenario 1 (`/api/users`):** Pada pengujian awal, endpoint Next.js dapat diakses tanpa autentikasi (ekspose akun admin `HTTP 200 OK` dan pembuatan akun admin ilegal `redteam_pwn` `HTTP 201 Created`), lalu dimitigasi pada Tahap 4 dan diverifikasi ulang.
   * **Skenario 2 (Firebase RTDB Emulator):** Pada pengujian awal, aturan `.write: true` memungkinkan injeksi data sensor palsu (`/sensor_data/redteam_spoof.json`, `HTTP 200 OK`), lalu diperketat pada Tahap 4 dan diverifikasi ulang.
   * **Skenario 3 (Reconnaissance & SSH Brute-Force):** Pemindaian `nmap` menunjukkan 65.531 port berstatus `filtered` oleh UFW, dan percobaan brute-force `Hydra` langsung digagalkan oleh kebijakan SSH *key-only* (`method reply 4`).
3. **Fase 3 (Finalisasi):** Seluruh temuan, analisis forensik/akar masalah, langkah mitigasi, dan hasil verifikasi disintesis dalam Incident Report (NIST SP 800-61r2), video demo, dan slide presentasi final.

---

## Video Demo Purple Team Loop

* **Tautan Video Demo (5–8 menit):** [Tonton Video Demo Kelompok 10](https://drive.google.com/file/d/12-3V3ac1E3rLSWu9NI-32P77FvQ0RtjB/view?usp=drivesdk)

---

## Indeks Artefak Proyek (Fase 1–3)

| Fase | Dokumen / Artefak | Tautan |
| :--- | :--- | :--- |
| **Fase 1** | IP Plan Lab | [`docs/design/ip_plan.md`](docs/design/ip_plan.md) |
| **Fase 1** | Diagram Topologi Jaringan | [`docs/design/topology.png`](docs/design/topology.png) |
| **Fase 2** | Hardening Checklist Blue Team | [`docs/blueteam/hardening_checklist.md`](docs/blueteam/hardening_checklist.md) |
| **Fase 2** | Bukti Hardening & Monitoring Blue Team | [`docs/blueteam/evidence/`](docs/blueteam/evidence/) |
| **Fase 2** | Attack Plan Red Team | [`docs/redteam/attack_plan.md`](docs/redteam/attack_plan.md) |
| **Fase 2** | Bukti Eksekusi Red Team | [`docs/redteam/evidence/`](docs/redteam/evidence/) |
| **Fase 2** | Slide Presentasi Minggu ke-9 | [`docs/presentation-minggu9/`](docs/presentation-minggu9/) |
| **Fase 2** | Purple Team Loop Log (Tahap 1–5) | [`docs/purple-team-log.md`](docs/purple-team-log.md) |
| **Fase 3** | **Incident Report Final (NIST SP 800-61r2)** | [`docs/final/incident_report.pdf`](docs/final/incident_report.pdf) |
| **Fase 3** | **Slide Presentasi Final** | [`docs/presentation-final/slides.pdf`](docs/presentation-final/slides.pdf) |
