# Checklist Hardening — Target Node (PiRoTech Web Monitoring)

**Status:** Final (Pasca Hardening Awal & Hardening Kedua / Purple Team Loop Tahap 4–5)  
**Target Node:** `serverpjblsiber` (`192.168.10.5` — Ubuntu Server 26.04)

---

## A. Access & Authentication

| Kontrol | Status | Bukti |
| :--- | :--- | :--- |
| Nonaktifkan login root langsung (`PermitRootLogin no`) | **Selesai** | [`evidence/01_sshd_before_after.png`](evidence/01_sshd_before_after.png) (`permitrootlogin prohibit-password` → `no`, `maxauthtries 6` → `3`) |
| SSH key-based auth, password auth dimatikan (`PasswordAuthentication no`) | **Selesai** | [`evidence/01_sshd_before_after.png`](evidence/01_sshd_before_after.png), [`evidence/02_ssh_login_password_rejected.png`](evidence/02_ssh_login_password_rejected.png) (percobaan login password & Hydra dari Kali ditolak) |
| Batas percobaan login (`MaxAuthTries 3`) + `fail2ban` pada `sshd` | **Selesai** | [`evidence/03_fail2ban_status_sshd.png`](evidence/03_fail2ban_status_sshd.png), [`evidence/20_fail2ban_aggressive_ban_192_168_10_100.png`](evidence/20_fail2ban_aggressive_ban_192_168_10_100.png) (`mode = aggressive`, IP `192.168.10.100` diblokir) |
| MFA pada dashboard monitoring | **Di Luar Scope Lab** | Tidak didukung pada Firebase Auth Emulator; didokumentasikan sebagai rekomendasi wajib untuk sistem produksi PiRoTech (lihat Incident Report Bab 7.2 & 7.3) |

---

## B. Service & Attack Surface

| Kontrol | Status | Bukti |
| :--- | :--- | :--- |
| Firewall (UFW) default-deny, whitelist port perlu saja (`22, 3000, 9000, 9099` dari subnet lab) | **Selesai** | [`evidence/04_ufw_before_after.png`](evidence/04_ufw_before_after.png), [`evidence/12_ufw_after_status.png`](evidence/12_ufw_after_status.png) (65.531 port berstatus `filtered` saat dipindai Nmap) |
| Update & patch OS ke versi aman | **Selesai** | [`evidence/05_apt_upgradable_before.png`](evidence/05_apt_upgradable_before.png) (64 paket) vs [`evidence/06_apt_upgradable_after.png`](evidence/06_apt_upgradable_after.png) (seluruh paket keamanan utama diperbarui, tersisa 2 paket non-kritis) |
| Nonaktifkan service/port tidak dipakai | **Selesai** | [`evidence/07_ss_tulpn_after.png`](evidence/07_ss_tulpn_after.png) — port publik hanya `22`, `3000`, `9000`, `9099`, serta `4400` dan `4500` khusus localhost (`127.0.0.1`) |
| Hapus kredensial/password default | **Selesai** | [`evidence/08_seed_login_verify.png`](evidence/08_seed_login_verify.png) — password seed emulator diganti dari nilai awal (`tools/seedAdmin.mjs` bawaan tidak dipakai); 5 percobaan brute-force password umum gagal `INVALID_PASSWORD` ([`../redteam/evidence/07_siklus2_web_bruteforce_400.png`](../redteam/evidence/07_siklus2_web_bruteforce_400.png)) |

---

## C. Data & Application

| Kontrol | Status | Bukti |
| :--- | :--- | :--- |
| Secret tidak hardcoded (env var, bukan di kode) | **Selesai** | `.env.local` dan `~/.seed.env` diatur `chmod 600` serta masuk `.gitignore` — [`evidence/09_git_grep_secrets.png`](evidence/09_git_grep_secrets.png), [`evidence/10_env_permissions_chmod600.png`](evidence/10_env_permissions_chmod600.png) |
| Validasi input & autentikasi pada endpoint API (`/api/users`) | **Selesai** | Mitigasi Tahap 4 menambahkan verifikasi Firebase ID Token dan pengecekan peran `admin` pada seluruh metode (`GET`, `POST`, `PATCH`, `DELETE`). Uji ulang Siklus 2 tanpa token seluruhnya ditolak `HTTP/1.1 401 Unauthorized` — [`evidence/16_seconion_wireshark_s1_api_401.png`](evidence/16_seconion_wireshark_s1_api_401.png), [`../redteam/evidence/05c_siklus2_s1_after_get_post_401.png`](../redteam/evidence/05c_siklus2_s1_after_get_post_401.png), [`../redteam/evidence/05d_siklus2_api_users_401_headers.png`](../redteam/evidence/05d_siklus2_api_users_401_headers.png) |
| Least-privilege pada akses data (`firebase.rules.json` RTDB) | **Selesai** | Aturan `.write: true` anonim ditutup dan diganti berbasis klaim autentikasi (`auth != null && auth.token.iot === true` untuk `latest`/`readings`, `role === 'admin'`/`operator` untuk `batches`/`commands`/`config`) — [`evidence/11_firebase_rules_verify.png`](evidence/11_firebase_rules_verify.png). Injeksi tanpa token ke namespace aplikasi (`?ns=demo-pirotech-default-rtdb`) ditolak `HTTP/1.1 401 Permission denied` ([`evidence/17_seconion_wireshark_s2_rtdb_401.png`](evidence/17_seconion_wireshark_s2_rtdb_401.png), [`../redteam/evidence/06_siklus2_rtdb_with_ns_401_denied.png`](../redteam/evidence/06_siklus2_rtdb_with_ns_401_denied.png)). *(Catatan: akses tanpa `?ns=` ke `fake-server` adalah perilaku bawaan emulator, bukan database aplikasi)* |
| Logging aktivitas mencurigakan | **Selesai** | UFW logging aktif ([`evidence/14_ufw_log_blocks.png`](evidence/14_ufw_log_blocks.png)), log `sshd` aktif ([`evidence/13_hydra_test_auth_log.png`](evidence/13_hydra_test_auth_log.png)), `fail2ban` aktif ([`evidence/20_fail2ban_aggressive_ban_192_168_10_100.png`](evidence/20_fail2ban_aggressive_ban_192_168_10_100.png)), serta tangkapan paket Wireshark pada Security Onion ([`evidence/15_seconion_wireshark_nmap.png`](evidence/15_seconion_wireshark_nmap.png) s.d. [`evidence/19_seconion_wireshark_s3_web_400.png`](evidence/19_seconion_wireshark_s3_web_400.png)) |
