## Deskripsi Skenario Proyek

Proyek ini mensimulasikan skenario **Simulated SOC Engagement** untuk mengamankan infrastruktur backend dan web dashboard monitoring dari sistem IoT **PiRoTech (Plastic Pyrolysis Technology)** dalam lingkungan lab virtual menggunakan **VirtualBox** dengan jaringan **Host-only/Internal Network** pada subnet `192.168.10.0/24`.

Seluruh pengujian dilakukan pada **replika tech stack lokal** menggunakan data sintetis/dummy tanpa terhubung ke sistem produksi Mitra.

**Entitas utama:**

1. **Attacker Node (`192.168.10.100`)** — Kali Linux / CyberOps, digunakan oleh **Red Team** untuk melakukan *reconnaissance* (scanning port) dan eksploitasi celah keamanan terhadap layanan web, API, serta akses remote pada Target Node.
2. **Target Node (`192.168.10.5`)** — VM Ubuntu Server 26.04 yang digunakan oleh **Blue Team** untuk menjalankan replika sistem backend PiRoTech:
   - **Web Dashboard Monitoring (Next.js)** pada port `3000`
   - **Firebase Realtime Database Emulator** pada port `9000`
   - **Firebase Auth Emulator** pada port `9099`
   - **OpenSSH Server** pada port `22`
3. **Monitoring Node (`192.168.10.200`)** — CyberOps Security Onion (Sguil, Kibana, Wireshark). NIC pada VM ini diatur ke **Promiscuous Mode (Allow VMs)** sehingga dapat memantau seluruh lalu lintas jaringan di subnet `192.168.10.0/24` untuk keperluan deteksi insiden dan analisis *baseline traffic*.

---

## Struktur Dokumentasi Repository

Detail artefak desain, progres hardening, dan rencana skenario pengujian tersimpan pada folder `docs/`:

- **Desain Topologi & IP Plan:** [`docs/design/topology.png`](docs/design/topology.png) & [`docs/design/ip_plan.md`](docs/design/ip_plan.md)
- **Materi Checkpoint Minggu ke-9:** [`docs/presentation-minggu9/slides.pdf`](docs/presentation-minggu9/slides.pdf) & [`docs/presentation-minggu9/tech_stack_declaration.md`](docs/presentation-minggu9/tech_stack_declaration.md)
- **Progres & Bukti Hardening (Blue Team):** [`docs/blueteam/hardening_checklist.md`](docs/blueteam/hardening_checklist.md) & [`docs/blueteam/evidence/`](docs/blueteam/evidence/)
- **Rencana Skenario Serangan (Red Team):** [`docs/redteam/attack_plan.md`](docs/redteam/attack_plan.md)
