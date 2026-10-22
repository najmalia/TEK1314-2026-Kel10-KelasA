# IP Plan - Kelompok 10

- **Segmen Jaringan:** `192.168.10.0/24` (flat, satu broadcast domain)
- **Mode Network (VirtualBox):** Host-only Adapter / Internal Network (Promiscuous Mode: Allow VMs)

| Hostname | IP Address | OS & Layanan | Peran |
| :--- | :--- | :--- | :--- |
| `ATK-KEL10` | `192.168.10.100` | Kali Linux / CyberOps | Attacker Node (Red Team) |
| `SRV-WEB-KEL10` | `192.168.10.5` | Ubuntu Server 26.04 (PiRoTech Web Next.js + Firebase Emulator) | Target Node (Blue Team) |
| `SOC-KEL10` | `192.168.10.200` | CyberOps Security Onion | Monitoring Node (Blue Team) |
