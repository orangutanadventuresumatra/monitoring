# Design — Sistem Monitoring & Manajemen Jaringan Multi-Backend

> Dokumen desain teknis yang mengimplementasikan `requirements.md`.
> Prinsip utama: **modular (di balik interface)**, **realtime via push**, dan
> **minimal bandwidth**.

- **Versi:** 1.0
- **Acuan:** requirements.md, STACK.md, RANCANGAN.md

---

## 1. Arsitektur Tingkat Tinggi

```
 ┌──────────────────────── HQ / DATA CENTER ────────────────────────┐
 │  CENTRAL (Go binary, frontend di-embed)                          │
 │  ┌────────────┐ ┌──────────────┐ ┌──────────────┐ ┌───────────┐ │
 │  │ REST API   │ │ WS Hub        │ │ Policy Engine │ │ Alerting  │ │
 │  │ (chi)      │ │ (realtime)    │ │ + Driver Mgr  │ │ + Logs    │ │
 │  └─────┬──────┘ └──────┬───────┘ └──────┬───────┘ └─────┬─────┘ │
 │        └───────────────┴───── Store (TimescaleDB via pgx) ──────┘ │
 │  License Manager · Auth/RBAC · Audit                              │
 └───────────────────────────────▲──────────────────────────────────┘
                 WebSocket (Protobuf+zstd, delta) │ via tunnel Palo Alto
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                          ▼
 ┌──────────────┐         ┌──────────────┐          ┌──────────────┐
 │ AGENT Cabang1 │         │ AGENT Cabang2 │   ...    │ AGENT CabangN │
 │ Collector +   │         │ Collector +   │          │ Collector +   │
 │ Buffer +      │         │ Buffer +      │          │ Buffer +      │
 │ Driver(backend)│        │ Driver(backend)│         │ Driver(backend)│
 └──────┬───────┘         └──────┬───────┘          └──────┬───────┘
        ▼                        ▼                         ▼
  Enforcement Backend     Enforcement Backend       Enforcement Backend
  (PA/Forti/MT/Linux)     (PA/Forti/MT/Linux)       (PA/Forti/MT/Linux)
```

Browser dashboard terhubung ke Central via **WebSocket (JSON, delta)** untuk
update live, dan **REST** untuk operasi request/response.

---

## 2. Komponen

### 2.1 Central (HQ)
- **REST API (chi):** auth/login, CRUD kebijakan & URL host, kueri histori,
  enrollment, manajemen lisensi.
- **WebSocket Hub:** menerima telemetri dari agent; menyiarkan delta ke browser.
- **Policy Engine + Driver Manager:** menyimpan kebijakan & mengirim perintah ke
  agent/driver backend.
- **Store:** TimescaleDB (hypertable untuk metrik time-series).
- **Alerting & Log Store:** evaluasi threshold, kirim notifikasi (shoutrrr),
  simpan log insiden.
- **License Manager:** verifikasi Ed25519, enforcement batasan.
- **Auth/RBAC + Audit.**

### 2.2 Agent (per cabang)
- **Collector:** ICMP/TCP/HTTP check (pro-bing, net/http), SNMP (gosnmp),
  NetFlow collector lokal (goflow2) **dengan agregasi**.
- **Buffer:** antrian tahan-putus-WAN (disk-backed), retry dengan backoff.
- **Driver backend:** implementasi `EnforcementBackend` + `TrafficSource`.
- **Transport client:** WebSocket + Protobuf + zstd; kirim delta.

### 2.3 Enforcement Backend (driver)
- `PaloAltoDriver` (pango / PAN-OS API)
- `FortinetDriver` (FortiOS REST)
- `MikroTikDriver` (go-routeros)
- `LinuxDriver` (go-tc, google/nftables, vishvananda/netlink, radius + OpenNDS
  sebagai proses terpisah)

### 2.4 Frontend
- React + ECharts, di-build statis, **di-embed** ke binary central via
  `go:embed`. Terhubung via WebSocket (live) + REST (operasi).

---

## 3. Lapisan Abstraksi (Interface Inti)

```go
// Kontrol (enforcement)
type EnforcementBackend interface {
    Name() string
    HealthCheck(ctx context.Context) error
    SetRateLimit(ctx context.Context, t Target, downKbps, upKbps int) error
    SetPriority(ctx context.Context, t Target, priority Priority) error
    RemovePolicy(ctx context.Context, policyID string) error
    ConfigureCaptivePortal(ctx context.Context, cfg CaptivePortalConfig) error
    ListAuthenticatedClients(ctx context.Context) ([]Client, error)
}

// Monitoring (sumber data)
type TrafficSource interface {
    InterfaceStats(ctx context.Context) ([]InterfaceStat, error)
    PerClientStats(ctx context.Context) ([]ClientStat, error)
    ConnectedDevices(ctx context.Context) ([]Device, error)
}

// Transport telemetri (disembunyikan agar bisa diganti)
type TelemetryTransport interface {
    Send(ctx context.Context, batch *DeltaBatch) error
    Recv(ctx context.Context) (<-chan Command, error)
}

// Penyimpanan (sudah ada: in-memory untuk dev, TimescaleDB untuk produksi)
type Store interface {
    Ingest(results []CheckResult)
    Status() []TargetStatus
    History(branchID, target string, limit int) []CheckResult
    Branches() []BranchSummary
}
```

Menambah perangkat baru = tambah implementasi `EnforcementBackend`/
`TrafficSource`. Mengganti transport = implementasi `TelemetryTransport` baru
(default: WebSocket).

---

## 4. Desain Transport (Realtime + Minimal Bandwidth)

### 4.1 Keputusan
- **WebSocket** untuk semua jalur realtime (bukan gRPC) — sederhana, lolos
  proxy/tunnel, native ke browser. Lihat alasan di STACK.md/diskusi.

### 4.2 Dua jalur
| Jalur | Transport | Encoding | Alasan |
|-------|-----------|----------|--------|
| Agent → Central (WAN) | WebSocket | **Protobuf + zstd** | payload padat, hemat WAN |
| Central → Browser (LAN) | WebSocket | **JSON** (+permessage-deflate) | browser-native |
| Operasi biasa | REST (chi) | JSON | tak butuh realtime |

### 4.3 Teknik hemat bandwidth
1. **Delta encoding** — kirim hanya metrik yang berubah > ambang.
2. **Protobuf** untuk jalur WAN.
3. **zstd** kompresi.
4. **Batching** beberapa metrik per pesan.
5. **Agregasi NetFlow di edge** — hanya ringkasan (top-talkers, total per
   device) dikirim; flow mentah tetap di cabang.
6. **Heartbeat tipis** saat idle.
7. **Interval adaptif** — metrik kritis lebih cepat.

### 4.4 Skema pesan (ringkas)
```
DeltaBatch {
  branch_id, seq, ts
  repeated DeviceDelta { device_id, rx_bps?, tx_bps?, online? }
  repeated IfaceDelta  { iface_id, rx_bps?, tx_bps? }
  repeated UrlCheck    { url_id, status_code, latency_ms, ttfb_ms, up }
  repeated Incident    { url_id, ts, status_code, error, route_json }
  heartbeat?
}
```
Field opsional (`?`) hanya hadir saat berubah → ukuran minimal.

### 4.5 Ketahanan WAN
- Agent menulis batch ke **buffer disk** sebelum kirim; hapus setelah ACK.
- Reconnect dengan exponential backoff; resume dari `seq` terakhir.

---

## 5. Desain Per Fitur

### 5.1 Monitoring Trafik & Device (FR-1)
```
Agent: kumpulkan stats (interval) → hitung delta → kirim DeltaBatch
Central: ingest → simpan hypertable → broadcast delta ke browser
Dashboard: grafik live (ECharts) + daftar device online/offline
```
Deteksi online/offline: ARP/conntrack/DHCP leases + ping sweep (Linux);
ARP/sesi via API (PA/Forti/MT).

### 5.2 Manajemen Bandwidth (FR-2)
Policy Engine menyimpan kebijakan netral; driver menerjemahkan:
| Aksi | MikroTik | Palo Alto | Fortinet | Linux |
|------|----------|-----------|----------|-------|
| Rate limit per IP | Simple Queue | QoS class | Per-IP shaper | tc HTB class |
| Prioritas host | Queue priority | QoS policy | Shaping policy | tc HTB+fq_codel |
| Captive portal | Hotspot | Auth Portal | Captive Portal | OpenNDS+RADIUS |

Alur penerapan:
```
Admin simpan kebijakan (REST) → Central validasi → kirim Command ke agent
→ driver apply ke backend → driver balas sukses/gagal → audit log + status UI
```

### 5.3 Captive Portal (FR-2.4/2.5)
```
Tamu konek WiFi → AP redirect ke landing page (di-host agent/central)
→ user login (voucher/akun) → otorisasi via RADIUS
→ device masuk VLAN tamu + batas bandwidth tamu
Karyawan → VLAN kantor (kebijakan berbeda)
```
Bergantung pada AP yang mendukung VLAN + redirect captive portal.

### 5.4 Monitoring Host Berbasis URL (FR-3) — Hybrid
**Active probe (utama):**
```
gocron jadwalkan tiap N detik:
  HTTP GET url → ukur status_code, ttfb, latency
  jika 200 → simpan metrik
  jika ≠200/timeout → buat Incident (+ jalankan traceroute → route_json) → alert
```
Probe dijalankan dari **central** (host eksternal) dan/atau **agent** (host
internal cabang) sesuai konfigurasi.

**Passive (opsional):**
```
Sumber: log firewall (PA/Forti/MT) / conntrack+DNS (Linux)
→ deteksi akses domain oleh user (realtime, event-driven)
→ untuk HTTPS tanpa decryption: domain/SNI + status koneksi + latency saja
→ push ke dashboard
```

**Visualisasi route:**
```
agent/central ─▶ hop1 ─▶ hop2 (Palo Alto) ─▶ hop3 (ISP) ─▶ ... ─▶ host
tiap hop: latency + status; tandai hop lambat/loss
```

---

## 6. Data Model (ringkas)

```
users(id, name, email, password_hash, role)               -- RBAC
branches(id, name, location, backend_type, enroll_token, status, last_seen)
devices(id, branch_id, ip, mac, hostname, online, last_seen)
interfaces(id, branch_id, name)
metrics_traffic(time, branch_id, device_id/iface_id, rx_bps, tx_bps)  -- hypertable
policies(id, branch_id, target, type[priority|ratelimit], down_kbps, up_kbps, priority, status)
captive_sessions(id, branch_id, mac, user, vlan, login_at, expires_at, quota)
url_hosts(id, name, url, scope[central|agent], owner_branch_id, interval, expected_code)
url_checks(time, url_host_id, status_code, latency_ms, ttfb_ms, up)   -- hypertable
incident_logs(id, time, url_host_id, status_code, latency_ms, error, route_json)
access_events(time, branch_id, src_ip/user, domain, allowed, latency_ms)  -- passive
alerts(id, time, source, severity, message, acknowledged)
audit_logs(id, time, user_id, action, target, detail)
licenses(id, customer, max_branches, max_devices, features, expires_at, signature)
```
Retensi data time-series diatur via kebijakan retensi TimescaleDB.

---

## 7. Keamanan & Lisensi

- **TLS** untuk semua koneksi ke central (WebSocket & REST).
- **Auth:** login → JWT (golang-jwt v5); password bcrypt/argon2.
- **RBAC:** admin/operator/viewer; aksi kontrol butuh peran memadai.
- **Audit log** untuk setiap perubahan kebijakan/lisensi.
- **Secret management:** kredensial API perangkat dienkripsi at-rest.
- **Lisensi:** file `.lic` ditandatangani **Ed25519**; public key tertanam di
  binary; verifikasi offline saat start + berkala; batasi cabang/device/fitur/
  expired; grace period sebelum membatasi; online activation opsional
  (nonaktif untuk air-gapped).
- **Distribusi:** binary di-obfuscate (garble); akses private registry =
  bagian kontrol lisensi.

---

## 8. Deployment

### 8.1 Central (docker-compose)
```
services:
  database:  TimescaleDB (volume persisten)
  central:   image nms-central (mount license.lic, expose 8080)
  proxy:     Caddy/Traefik (auto-HTTPS) [opsional]
```
### 8.2 Agent (container per cabang)
```
docker run -d \
  -e CENTRAL_URL=wss://central-hq:8080 \
  -e BRANCH_ID=cabang-07 \
  -e ENROLL_TOKEN=xxxx \
  -e BACKEND=mikrotik|paloalto|fortinet|linux \
  registry-mu/nms-collector:<ver>
```
### 8.3 Enrollment
```
Agent start → kirim token ke central → central verifikasi → balas config
(target, interval, kebijakan) → agent mulai bekerja
```
### 8.4 Distribusi & arsitektur
- Image **multi-arch** (amd64/arm64); offline tarball (`docker save/load`)
  untuk pelanggan air-gapped.

---

## 9. Technology Stack (ringkas; detail di STACK.md)

```
Go 1.25+ · chi v5 (REST) · coder/websocket (realtime)
TimescaleDB + pgx v5 (≥5.10) + sqlc + goose
gocron v2 (penjadwalan) · slog (logging)
Monitoring: gosnmp · goflow2 · pro-bing · net/http
Encoding/kompresi WAN: google.golang.org/protobuf · klauspost/compress (zstd)
Driver: go-routeros (MikroTik) · PaloAltoNetworks/pango · REST (Fortinet)
        Linux: florianl/go-tc · google/nftables · vishvananda/netlink · layeh radius
Auth/Lisensi: golang-jwt v5 · crypto/ed25519 · bcrypt/argon2
Alerting: containrrr/shoutrrr
Frontend: React + ECharts (go:embed)
Build: garble + Docker multi-arch
```
Catatan lisensi: verifikasi `layeh.com/radius`, `containrrr/shoutrrr`,
`go-routeros/routeros` sebelum dipakai; komponen GPL hanya sebagai proses
terpisah (lihat STACK.md).

---

## 10. Penanganan Error & Edge Case

- **WAN putus:** buffer + retry; UI tandai cabang "stale/last seen".
- **Backend tak mendukung fitur:** kembalikan error tipe `ErrUnsupported`; UI
  tampilkan "tidak didukung".
- **Apply kebijakan gagal:** rollback ke state sebelumnya bila memungkinkan;
  catat audit + alert.
- **Lisensi invalid/expired:** grace period → mode terbatas (read-only).
- **HTTPS pasif tanpa decryption:** sajikan domain/SNI + status koneksi saja.

---

## 11. Strategi Pengujian

- **Unit:** logika delta, policy engine, license verifier, driver (mock backend).
- **Integration:** agent↔central (WebSocket), store (TimescaleDB test container).
- **Driver contract tests:** satu rangkaian tes terhadap interface, dijalankan
  untuk tiap driver.
- **Bandwidth test:** ukur byte terkirim per cabang untuk verifikasi NFR-2.
- **Resilience test:** simulasi putus WAN → pastikan tidak ada kehilangan data.
