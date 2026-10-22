# Checklist Hardening — Target Node (PiRoTech Web Monitoring)

## A. Access & Authentication

| Kontrol | Status | Bukti |
|---|---|---|
| Nonaktifkan login root langsung (`PermitRootLogin no`) | Selesai | `evidence/sshd-config.png`, `evidence/sshd-T-after.txt` |
| SSH key-based auth, password auth dimatikan (`PasswordAuthentication no`) | Selesai | `evidence/ssh-login-password-rejected.png` (percobaan dari Kali ditolak) |
| Batas percobaan login (`MaxAuthTries 3`) + fail2ban pada sshd | Selesai | `evidence/fail2ban-status.png` |
| MFA pada dashboard monitoring | Belum | Di luar scope emulator Firebase Auth; direncanakan sebagai mitigasi Fase 2 jika relevan |

## B. Service & Attack Surface

| Kontrol | Status | Bukti |
|---|---|---|
| Firewall (UFW) default-deny, whitelist port perlu saja (22, 3000, 9000, 9099 dari subnet lab) | Selesai | `evidence/ufw-status-before.txt`, `evidence/ufw-status-after.txt` |
| Update & patch OS ke versi aman | Selesai/Sebagian | `evidence/apt-upgradable-before.txt` (64 update) vs `evidence/apt-upgradable-after.txt` |
| Nonaktifkan service/port tidak dipakai | Selesai | `evidence/ss-tulpn-after.txt` — hanya 22/3000/9000/9099/4400(lo) terbuka |
| Hapus kredensial/password default | Selesai | Password seed emulator diganti dari nilai awal; `tools/seedAdmin.mjs` bawaan repo tidak dipakai |

## C. Data & Application

| Kontrol | Status | Bukti |
|---|---|---|
| Secret tidak hardcoded (env var, bukan di kode) | Selesai | `.env.local` dan `~/.seed.env` di-`chmod 600`, tidak ter-commit (`.gitignore`) — `evidence/git-grep-secrets.txt` |
| Validasi input pada endpoint API | Belum | `/api/users` (GET & POST) saat ini **tanpa autentikasi** — temuan sengaja dibiarkan sebagai target Skenario Fase 2 (lihat `attack_plan.md`), akan dimitigasi pada Purple Team Loop Minggu 10–11 |
| Least-privilege pada akses data (RTDB rules) | Sebagian | `firebase.rules.json` menerapkan role-based access (admin/operator) pada sebagian besar node, tetapi node `sensor_data`, `monitoring`, `contact/messages` masih `.write: true` tanpa autentikasi — disengaja sebagai celah simulasi ESP32, dicatat sebagai temuan |
| Logging aktivitas mencurigakan | Selesai | UFW logging aktif (`evidence/ufw-log-blocks.txt`), log sshd failed login (`evidence/sshd-failed-log.txt`) |
