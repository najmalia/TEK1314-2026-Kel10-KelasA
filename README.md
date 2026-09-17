# TEK1314-2025-Kel10-KelasA

## Deskripsi Skenario Proyek

Proyek ini mensimulasikan skenario **serangan terhadap aplikasi web (Web Application Attack)** dalam lingkungan lab virtual menggunakan **VirtualBox** dengan jaringan **Host-only/Internal Network** pada subnet `192.168.10.0/24`.

**Entitas utama:**
1. **Attacker Node** (`192.168.10.100`) — Kali Linux / CyberOps, digunakan Red Team untuk reconnaissance (scanning) dan eksploitasi celah keamanan pada aplikasi web target.
2. **Target Node** (`192.168.10.5`) — VM Ubuntu yang menjalankan **DVWA (Damn Vulnerable Web Application)** pada port 80/443, sengaja dikonfigurasi dengan celah keamanan umum (SQL Injection, XSS, CSRF, Command Injection) sebagai objek pembelajaran.
3. **Monitoring Node** (`192.168.10.200`) — Security Onion (Sguil, Kibana, Wireshark). NIC pada VM ini diset ke **Promiscuous Mode** sehingga dapat menangkap seluruh trafik yang lewat di jaringan Host-only/Internal yang sama, meski tidak menjadi jalur komunikasi langsung.
4. **Reserved IP Range** (`192.168.10.210` – `250`) — dicadangkan untuk kebutuhan node tambahan di tahap implementasi berikutnya.

Detail lengkap topologi dan alokasi IP dapat dilihat pada folder `docs/design/`:
- [`docs/design/topology.png`](docs/design/topology.png) — diagram topologi jaringan.
- [`docs/design/ip_plan.md`](docs/design/ip_plan.md) — tabel IP plan.
