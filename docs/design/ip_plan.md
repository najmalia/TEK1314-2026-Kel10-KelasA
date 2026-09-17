# IP Plan - Kelompok 10

**Segmen Jaringan:** 192.168.10.0/24 (flat, satu broadcast domain)
**Mode Network (VirtualBox):** Host-only Adapter/Internal Network

| Hostname          | IP Address          | OS Direncanakan        | Peran            |
|-------------------|----------------------|--------------------------|------------------|
| ATK-KEL10         | 192.168.10.100       | Kali Linux / CyberOps    | Attacker Node    |
| SRV-WEB-KEL10     | 192.168.10.5         | Ubuntu + DVWA            | Target Node (Korban) |
| SOC-KEL10         | 192.168.10.200       | Security Onion           | Monitoring Node  |
| (Reserved)        | 192.168.10.210–250   | -                        | Cadangan node tambahan bila diperlukan |

## Catatan
- Semua node berada di satu subnet /24  — mengikuti format contoh di Panduan Minggu 4.
- Karena flat /24, **tidak ada routing antar-subnet**; port mirroring cukup dilakukan lewat Promiscuous Mode pada adapter monitoring di VirtualBox (Allow VMs), tanpa perlu VM router terpisah.
- Adapter kedua di tiap VM ditambahkan sebagai NAT terpisah jika butuh akses internet untuk update/install tools — traffic NAT ini tidak boleh tercampur dengan traffic lab yang dipantau.
- Range reserved (.210–.250) belum dialokasikan ke node spesifik; akan diupdate jika Red/Blue Team menambah target atau tools tambahan.

