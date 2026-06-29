# Rancangan Sistem Monitoring & Manajemen Jaringan Multi-Backend

> Sistem terpusat untuk memantau dan mengatur jaringan banyak cabang, dengan
> **backend enforcement yang dapat dipilih**: Palo Alto, Fortinet, MikroTik,
> atau konfigurasi mandiri pakai Linux server.

---

## 1. Tujuan & Ruang Lingkup

Membangun produk monitoring + manajemen jaringan yang:
- Dikelola **terpusat** dari HQ untuk banyak cabang.
- **Tidak terkunci ke satu vendor** — pelanggan bisa pakai firewall/router yang
  sudah mereka miliki (Palo Alto/Fortinet/MikroTik) atau Linux server mandiri.
- Berfokus pada 3 kelompok fitur (lihat bagian 5–7).

### Fitur yang harus ada
1. **Monitoring trafik bandwidth + status perangkat online/offline.**
2. **Manajemen bandwidth**: prioritas host, batas kecepatan download/upload,
   login WiFi (captive portal) untuk memisahkan jaringan karyawan & tamu.
3. **Monitoring host berbasis URL**: tambah URL host, lihat alur (path/route)
   yang dilalui, dan **catat log saat respons bukan 200** pada host tertentu.

---

## 2. Prinsip Desain: Backend Enforcement yang Pluggable

Inti dari fleksibilitas multi-vendor adalah **lapisan abstraksi (driver)**.
Logika produk berbicara ke satu *interface* seragam; tiap perangkat punya
*driver* sendiri yang menerjemahkan perintah ke API/konfigurasi spesifiknya.

```
        Logika Produk & Dashboard (seragam)
                      │
                      ▼
        ┌─────────────────────────────────┐
        │   Enforcement Backend (interface) │
        └──────────────┬──────────────────┘
        ┌──────┬───────┼────────┬──────────┐
        ▼      ▼       ▼        ▼          ▼
   PaloAlto Fortinet MikroTik  Linux   (vendor lain
   Driver   Driver   Driver    Driver   di masa depan)
   (PAN-OS  (FortiOS (RouterOS (tc/nft/
    API)     API)     API)     RADIUS)
```

Keuntungan: menambah dukungan perangkat baru = menambah 1 driver, **tanpa**
mengubah dashboard, database, atau logika inti.

---

## 3. Arsitektur Umum

```
 ┌──────────────────────── HQ / DATA CENTER ────────────────────────┐
 │  CENTRAL                                                          │
 │  • Dashboard (web)        • API & Auth/RBAC                        │
 │  • Time-Series DB         • Policy Engine (kebijakan bandwidth)    │
 │  • Alerting & Log Store   • License Manager                        │
 └───────────────────────────────▲──────────────────────────────────┘
                                  │ tunnel (Palo Alto) / VPN
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                          ▼
 ┌──────────────┐         ┌──────────────┐          ┌──────────────┐
 │  CABANG 1     │         │  CABANG 2     │   ...    │  CABANG N     │
 │  AGENT +      │         │  AGENT +      │          │  AGENT +      │
 │  Enforcement  │         │  Enforcement  │          │  Enforcement  │
 │  Backend:     │         │  Backend:     │          │  Backend:     │
 │  (PA/Forti/   │         │  (PA/Forti/   │          │  (PA/Forti/   │
 │   MT/Linux)   │         │   MT/Linux)   │          │   MT/Linux)   │
 └──────────────┘         └──────────────┘          └──────────────┘
```

- **Central (HQ):** otak + dashboard + penyimpanan + penerbitan kebijakan.
- **Agent (per cabang):** mengumpulkan metrik & menjalankan perintah kebijakan
  melalui driver backend yang dipilih untuk cabang itu.
- **Enforcement Backend:** perangkat/mekanisme yang benar-benar menegakkan
  kebijakan (inline di jalur trafik).

### Catatan topologi untuk backend "Linux mandiri"
```
   Internet ──▶ Palo Alto ──▶ Linux Server (gateway) ──▶ Device
                (perimeter)    (tc/nftables/RADIUS/        (PC, HP)
                               captive portal)
```
Linux server menjadi *default gateway* device sehingga bisa membentuk
(shaping) & memantau seluruh trafik. Untuk backend PA/Forti/MikroTik, perangkat
itu sendiri yang sudah inline — agent cukup bicara via API.

---

## 4. Lapisan Abstraksi (Kontrak Driver)

Dua kontrak utama: **EnforcementBackend** (kontrol) dan **TrafficSource**
(monitoring). Satu driver perangkat biasanya mengimplementasikan keduanya.

```
EnforcementBackend:
  HealthCheck()
  SetRateLimit(target, downloadKbps, uploadKbps)   // batas kecepatan
  SetPriority(target, prioritas)                    // prioritas host (QoS)
  RemovePolicy(policyID)
  ConfigureCaptivePortal(config)                    // login WiFi tamu/karyawan
  ListAuthenticatedClients()

TrafficSource (monitoring):
  GetInterfaceStats()        // throughput per interface
  GetPerClientStats()        // bandwidth per IP/device
  GetConnectedDevices()      // daftar device online/offline
```

`target` = IP / subnet / MAC / user-group, tergantung kemampuan backend.

---

## 5. Fitur 1 — Monitoring Trafik Bandwidth & Status Perangkat

### Yang dipantau
- Throughput per interface & per device (download/upload, real-time + historis).
- Status **online/offline** tiap perangkat.
- Top talkers (device yang paling banyak memakai bandwidth).

### Cara per backend
| Backend   | Sumber data trafik              | Deteksi device online/offline       |
|-----------|----------------------------------|-------------------------------------|
| Palo Alto | NetFlow/IPFIX + PAN-OS API       | ARP/sesi via API, ping sweep        |
| Fortinet  | NetFlow + FortiOS API            | DHCP/ARP via API                    |
| MikroTik  | RouterOS API (`/interface`,      | DHCP leases + ARP via API           |
|           | `/ip/accounting`) atau SNMP      |                                     |
| Linux     | `tc -s`, counter `nftables`,     | tabel ARP/`conntrack`, DHCP leases, |
|           | `/proc/net/dev`, conntrack       | ping sweep                          |

### Alur data
```
Agent kumpulkan stats (interval mis. 10–30 dtk)
   → kirim ke Central via tunnel (buffer bila WAN putus)
   → simpan ke Time-Series DB
   → tampilkan grafik real-time + historis di dashboard
```

---

## 6. Fitur 2 — Manajemen Bandwidth & Captive Portal

### 2a. Prioritas Host (QoS)
Tetapkan kelas prioritas (mis. tinggi/normal/rendah) per host/aplikasi.

| Backend   | Mekanisme                                  |
|-----------|--------------------------------------------|
| Palo Alto | QoS Policy + Profile (via API)             |
| Fortinet  | Traffic Shaping Policy (via API)           |
| MikroTik  | Queue Tree / priority (RouterOS API)       |
| Linux     | `tc` HTB + `fq_codel`, klasifikasi `nftables` |

### 2b. Batas Kecepatan Download/Upload per IP/Device
| Backend   | Mekanisme                                  |
|-----------|--------------------------------------------|
| Palo Alto | QoS class dengan guaranteed/max bandwidth  |
| Fortinet  | Per-IP Traffic Shaper                      |
| MikroTik  | Simple Queue per IP (sangat mudah)         |
| Linux     | `tc` HTB class per IP (ceil = batas atas)  |

### 2c. Login WiFi / Captive Portal (pisah karyawan vs tamu)
Alur umum:
```
Device tamu konek WiFi → diarahkan ke LANDING PAGE LOGIN (di-host Central/agent)
   → user login (voucher/akun/sosial)
   → otorisasi via RADIUS
   → device masuk VLAN tamu + kena batas bandwidth tamu
Karyawan → VLAN kantor (akses penuh / kebijakan berbeda)
```

| Backend   | Mekanisme captive portal                   |
|-----------|--------------------------------------------|
| Palo Alto | Authentication Portal + User-ID            |
| Fortinet  | Captive Portal + user groups               |
| MikroTik  | **Hotspot** (captive portal bawaan, kuat)  |
| Linux     | **OpenNDS/CoovaChilli + FreeRADIUS** + VLAN |

> Captive portal sangat bergantung pada **Access Point**: AP harus mendukung
> VLAN dan redirect ke captive portal (atau gunakan WiFi controller).

---

## 7. Fitur 3 — Monitoring Host Berbasis URL

### Kemampuan
1. **Tambah URL host** lewat dashboard (mis. `https://layanan.cabang7.local`).
2. **Pengecekan berkala**: status code, TTFB, total response time, ketersediaan.
3. **Lihat alur (path/route)** yang dilalui ke host — visualisasi hop demi hop
   (berbasis traceroute) untuk tahu di mana letak lambat/putus.
4. **Logging saat respons bukan 200**: setiap kali host mengembalikan status
   selain `200 OK` (mis. 4xx/5xx) atau timeout → buat **entri log insiden**
   berisi: waktu, URL, status code, response time, pesan error, dan jejak route
   saat itu. Bisa memicu alert.

### Data yang dicatat per pengecekan
```
timestamp, url, status_code, latency_ms, ttfb_ms, up(bool),
error(opsional), route_hops[] (saat insiden / berkala)
```

### Logika "log saat respons selain 200"
```
lakukan HTTP check
   jika status_code == 200 → simpan metrik biasa
   jika status_code != 200 ATAU timeout/error:
       → tulis ENTRI LOG INSIDEN (detail lengkap + route)
       → naikkan counter kegagalan
       → jika melewati ambang → kirim ALERT (email/Telegram/Slack)
```

### Visualisasi alur (route)
```
Client/Agent ─▶ hop1 (gateway) ─▶ hop2 (Palo Alto) ─▶ hop3 (ISP) ─▶ ... ─▶ Host
   tiap hop: latency & status → tandai hop yang lambat/loss
```
Pengecekan dijalankan dari **Central** (untuk host eksternal) dan/atau dari
**Agent cabang** (untuk host internal cabang), agar terlihat dari sudut pandang
yang relevan.

---

## 8. Matriks Dukungan per Backend

| Kemampuan                     | Palo Alto | Fortinet | MikroTik | Linux mandiri |
|-------------------------------|:---------:|:--------:|:--------:|:-------------:|
| Monitoring trafik             |    ✅     |    ✅    |    ✅    |      ✅       |
| Device online/offline         |    ✅     |    ✅    |    ✅    |      ✅       |
| Prioritas host (QoS)          |    ✅     |    ✅    |    ✅    |      ✅       |
| Batas download/upload per IP  |    ✅     |    ✅    |    ✅    |      ✅       |
| Captive portal WiFi           |    ✅     |    ✅    |  ✅(Hotspot) |   ✅(OpenNDS) |
| Perlu perangkat inline tambahan | ❌      |    ❌    |    ❌    | ✅ (Linux box) |
| Integrasi via                 |  API      |   API    |   API    | exec lokal    |

> Catatan: ketersediaan fitur API tertentu bergantung **versi & lisensi**
> perangkat (mis. PAN-OS/FortiOS tertentu). Wajib diverifikasi saat onboarding.

---

## 9. Data Model (ringkas)

```
branches(id, name, location, backend_type, status, last_seen)
devices(id, branch_id, ip, mac, hostname, online, last_seen)
interfaces(id, branch_id, name)
metrics_traffic(time, branch_id, device_id/iface_id, rx_bps, tx_bps)   [hypertable]
policies(id, branch_id, target, type[priority|ratelimit], down_kbps, up_kbps, priority)
captive_sessions(id, branch_id, mac, user, vlan, login_at, expires_at, quota)
url_hosts(id, name, url, owner_branch_id, interval, expected_code=200)
url_checks(time, url_host_id, status_code, latency_ms, ttfb_ms, up)     [hypertable]
incident_logs(id, time, url_host_id, status_code, latency_ms, error, route_json)
alerts(id, time, source, severity, message, acknowledged)
audit_logs(id, time, user, action, target, detail)
users(id, name, role)   // RBAC: admin/operator/viewer
licenses(...)           // untuk produk komersial
```

---

## 10. Deployment per Skenario

| Skenario pelanggan        | Yang dipasang                                   |
|---------------------------|-------------------------------------------------|
| Punya Palo Alto/Fortinet  | Central (HQ) + Agent ringan; kontrol via API     |
| Punya MikroTik            | Central + Agent; kontrol via RouterOS API        |
| Tanpa firewall pengatur   | Central + **Linux gateway inline** per cabang    |
| Campuran                  | Tiap cabang pilih backend-nya masing-masing      |

Semua dikemas sebagai **container Docker** (image multi-arch). Backend dipilih
per cabang lewat konfigurasi/onboarding di dashboard.

---

## 11. Stack Teknologi

```
Backend & Agent : Go (chi router, pgx)         — binary terkompilasi (proteksi)
Database         : TimescaleDB (PostgreSQL)
Driver perangkat : PAN-OS API, FortiOS API, RouterOS API, (Linux: tc/nftables/
                   FreeRADIUS/OpenNDS)
Monitoring lib   : gosnmp, goflow2 (NetFlow), pro-bing (ping/traceroute)
Realtime         : WebSocket (coder/websocket)
Auth & Lisensi   : golang-jwt, crypto/ed25519
Frontend         : React + ECharts (di-embed ke binary via go:embed)
Deploy           : Docker / docker-compose (multi-arch)
```
Semua dependency berlisensi permisif (MIT/BSD/Apache) — aman untuk produk
komersial tertutup.

---

## 12. Roadmap Bertahap

| Fase | Fokus                                                                 |
|------|-----------------------------------------------------------------------|
| 1 (MVP) | Central + Agent; Fitur 1 (monitoring trafik & device) + Fitur 3 dasar (URL check, log non-200) untuk **1 backend dulu** (rekomendasi: MikroTik atau Linux) |
| 2 | Fitur 2: prioritas host + batas kecepatan (driver pertama)              |
| 3 | Captive portal WiFi (tamu vs karyawan)                                 |
| 4 | Tambah driver backend lain (multi-vendor penuh) + visualisasi route    |
| 5 | Fitur komersial: lisensi, multi-tenant, RBAC, packaging jual           |
```
