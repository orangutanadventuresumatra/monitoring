# Rekomendasi Stack & Library Go

> Status per Juni 2026. Semua dipilih dengan prioritas: **powerful, terawat,
> dan berlisensi permisif** (aman untuk produk komersial tertutup).
> Aturan emas: setiap library baru → **verifikasi lisensi & advisory keamanan**
> sebelum dipakai.

---

## 1. Inti Aplikasi (Central & Agent)

| Kebutuhan | Library / Modul | Versi mayor | Lisensi | Catatan |
|-----------|-----------------|-------------|---------|---------|
| HTTP router | `github.com/go-chi/chi/v5` | v5 | MIT ✅ | Kompatibel penuh `net/http`. ⚠ pakai versi terbaru — ada advisory keamanan (GHSA-3fxj-6jh8-hvhx, Mei 2026) yang sudah dipatch |
| Driver PostgreSQL | `github.com/jackc/pgx/v5` | **v5.10.0** | MIT ✅ | ⚠ pakai ≥ versi yang sudah patch CVE-2026-33816 (memory-safety, April 2026) |
| Query type-safe | `sqlc` (codegen) | v1.x | MIT ✅ | Generate Go dari SQL — lebih aman & cepat dari ORM |
| Migrasi DB | `pressly/goose` atau `golang-migrate/migrate` | v3 / v4 | MIT ✅ | Versioning skema TimescaleDB |
| Penjadwalan tugas | `github.com/go-co-op/gocron/v2` | **v2** | MIT ✅ | Untuk siklus polling/cek berkala; fluent & powerful |
| Konfigurasi | `github.com/knadh/koanf/v2` | v2 | MIT ✅ | Multi-sumber (file/env), modular |
| Logging | `log/slog` (**stdlib**) | - | BSD ✅ | Structured logging bawaan Go; tambah `lmittmann/tint` untuk warna saat dev |
| Validasi input | `github.com/go-playground/validator/v10` | v10 | MIT ✅ | Validasi struct/request |
| WebSocket (realtime dashboard) | `github.com/coder/websocket` | v1.x | ISC ✅ | Penerus `nhooyr.io/websocket`, idiomatik |
| Retry HTTP client | `github.com/hashicorp/go-retryablehttp` | v0.7 | MPL-2.0 🟡 | MPL = copyleft file-level; aman dipakai sbg dependency, tak wajib buka kode sendiri |

---

## 2. Monitoring (Fitur 1 & 3)

| Kebutuhan | Library / Modul | Lisensi | Catatan |
|-----------|-----------------|---------|---------|
| SNMP polling | `github.com/gosnmp/gosnmp` | BSD ✅ | Standar de-facto SNMP di Go (v1/v2c/v3) |
| NetFlow/IPFIX/sFlow collector | `github.com/netsampler/goflow2` | BSD-3 ✅ | Performa tinggi, aktif dikembangkan; untuk bandwidth per-IP |
| ICMP ping & **traceroute** | `github.com/prometheus-community/pro-bing` | MIT ✅ | RTT, jitter, loss + traceroute (Fitur 3: alur/route) |
| HTTP host check (Fitur 3) | `net/http` (**stdlib**) | BSD ✅ | Cukup stdlib: status code, TTFB, latency, deteksi non-200 |
| Metrik internal app | `github.com/prometheus/client_golang` | Apache-2.0 ✅ | Ekspos metrik sistem sendiri (opsional) |

---

## 3. Enforcement Backend / Driver (Fitur 2)

### 3a. Driver perangkat vendor (via API)
| Backend | Library / Modul | Lisensi | Catatan |
|---------|-----------------|---------|---------|
| **MikroTik** | `github.com/go-routeros/routeros` | ⚠ verifikasi (umumnya permisif) | Library kanonik RouterOS API: Simple Queue, Queue Tree, Hotspot |
| **Palo Alto** | `github.com/PaloAltoNetworks/pango` | ISC ✅ | SDK Go resmi-komunitas PAN-OS; QoS policy, Auth Portal |
| **Fortinet** | REST API langsung (`net/http`) atau SDK komunitas | - | FortiOS REST API; SDK Go kurang matang → pakai HTTP client sendiri |

### 3b. Backend Linux mandiri (gateway inline — pure Go via netlink)
| Kebutuhan | Library / Modul | Lisensi | Catatan |
|-----------|-----------------|---------|---------|
| Traffic shaping (`tc`) | `github.com/florianl/go-tc` | MIT ✅ | Atur qdisc/class/filter HTB langsung via netlink (pure Go) — limit & prioritas bandwidth |
| Firewall/NAT (`nftables`) | `github.com/google/nftables` | Apache-2.0 ✅ | Kelola ruleset nftables (NAT, isolasi VLAN) via netlink |
| Operasi link/route | `github.com/vishvananda/netlink` | Apache-2.0 ✅ | Interface, IP, routing |
| RADIUS (otorisasi captive portal) | `layeh.com/radius` | ⚠ verifikasi (MPL?) | Client/server RADIUS; integrasi FreeRADIUS |
| Captive portal engine | **OpenNDS/CoovaChilli** (eksternal) | GPL ⚠ | Dijalankan sbg proses terpisah (bukan di-link ke binary) → tetap aman untuk produk tertutup |
| DHCP leases | `github.com/insomniacslk/dhcp` atau baca file leases | BSD ✅ | Deteksi device & assign gateway |

> ⚠ **Penting soal GPL:** OpenNDS/CoovaChilli/FreeRADIUS berlisensi GPL, **tapi**
> kita memakainya sebagai **proses/binary terpisah** (bukan meng-import code-nya
> ke dalam binary Go kita). Berinteraksi via API/CLI/config = **tidak menulari**
> lisensi produkmu. Yang dilarang adalah me-*link* kode GPL ke dalam binary
> proprietary-mu.

---

## 4. Keamanan, Auth & Lisensi Komersial

| Kebutuhan | Library / Modul | Lisensi | Catatan |
|-----------|-----------------|---------|---------|
| JWT (sesi dashboard) | `github.com/golang-jwt/jwt/v5` | MIT ✅ | Auth token |
| Hash password | `golang.org/x/crypto/bcrypt` atau `argon2` | BSD ✅ | Simpan password aman |
| Tanda tangan lisensi | `crypto/ed25519` (**stdlib**) | BSD ✅ | License file offline bertanda tangan |
| Secret management | `crypto/*` (stdlib) + integrasi Vault opsional | BSD ✅ | Simpan kredensial API firewall |
| Embed frontend | `embed` (**stdlib**) | BSD ✅ | Bungkus dashboard ke 1 binary |

---

## 5. Notifikasi / Alerting

| Kebutuhan | Library / Modul | Lisensi | Catatan |
|-----------|-----------------|---------|---------|
| Multi-channel notif | `github.com/containrrr/shoutrrr` | ⚠ verifikasi | 1 library → Telegram, Slack, Email, Discord, dll. Sangat praktis |
| Email langsung (alternatif) | `net/smtp` (**stdlib**) atau `wneessen/go-mail` | BSD/MIT ✅ | Kalau cukup email saja |

---

## 6. Build, Proteksi & Deploy

| Kebutuhan | Tool | Lisensi | Catatan |
|-----------|------|---------|---------|
| Obfuscation binary | `mvdan.cc/garble` | BSD ✅ | Acak simbol/string → reverse-engineering lebih sulit |
| Container | Docker multi-stage build | - | Image multi-arch (amd64/arm64) |
| Orkestrasi | docker-compose (awal) | - | Cukup untuk MVP |

---

## 7. Frontend (di luar Go)

| Kebutuhan | Pilihan | Lisensi | Catatan |
|-----------|---------|---------|---------|
| Framework UI | **React** (atau Vue/Svelte) | MIT ✅ | Build statis → di-embed ke binary |
| Grafik | **Apache ECharts** atau Recharts | Apache/MIT ✅ | Grafik bandwidth/latency real-time & historis |
| Realtime | WebSocket (pasangan `coder/websocket`) | - | Update live dashboard |

---

## Ringkasan Stack Final

```
Bahasa            : Go 1.25+
Router/API        : chi v5
Database          : TimescaleDB (PostgreSQL) + pgx v5 + sqlc + goose
Scheduler         : gocron v2
Monitoring        : gosnmp, goflow2, pro-bing, net/http
Driver perangkat  : go-routeros (MikroTik), pango (Palo Alto),
                    REST (Fortinet)
Linux backend     : go-tc, google/nftables, vishvananda/netlink,
                    layeh radius (+ OpenNDS/FreeRADIUS sbg proses terpisah)
Auth/Lisensi      : golang-jwt v5, crypto/ed25519, bcrypt/argon2
Alerting          : shoutrrr (multi-channel)
Realtime          : coder/websocket
Frontend          : React + ECharts → go:embed
Build/Proteksi    : garble + Docker multi-arch
```

### Catatan lisensi yang WAJIB diverifikasi sebelum dipakai
- `layeh.com/radius` — cek lisensi (kemungkinan MPL).
- `containrrr/shoutrrr` — cek lisensi.
- `go-routeros/routeros` — cek lisensi (umumnya permisif).
- Komponen GPL (OpenNDS/CoovaChilli/FreeRADIUS) **hanya** sebagai proses
  terpisah, **tidak** di-link ke binary produk.
