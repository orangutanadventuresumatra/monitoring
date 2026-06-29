# Monitoring — Sistem Monitoring & Manajemen Jaringan Multi-Backend

Produk **self-hosted** untuk memantau dan mengatur jaringan banyak cabang dari
satu pusat, dengan **backend enforcement yang dapat dipilih**: Palo Alto,
Fortinet, MikroTik, atau Linux server mandiri.

Fokus utama: **realtime** dan **minimal bandwidth**.

## Fitur Inti

1. **Monitoring trafik bandwidth & status perangkat** (online/offline), realtime.
2. **Manajemen bandwidth** — prioritas host, batas kecepatan download/upload,
   captive portal WiFi untuk memisahkan jaringan karyawan & tamu.
3. **Monitoring host berbasis URL** — tambah URL host, lihat alur (route) yang
   dilalui, dan catat log saat respons bukan 200.

## Arsitektur Singkat

```
[HQ] Central (dashboard + API + DB + alerting)
        ▲  WebSocket (Protobuf+zstd, delta) via tunnel
        │
[Cabang] Agent (collector + buffer + driver backend)
        │
   Enforcement Backend: Palo Alto / Fortinet / MikroTik / Linux
```

- **Transport:** WebSocket (realtime). Protobuf+zstd untuk jalur WAN (hemat
  bandwidth), JSON ke browser, REST untuk operasi biasa.
- **Bahasa:** Go 1.25+ · **Database:** TimescaleDB · **Frontend:** React + ECharts
  (di-embed ke binary) · **Deploy:** Docker multi-arch.

## Dokumentasi

| Dokumen | Isi |
|---------|-----|
| [docs/requirements.md](docs/requirements.md) | Functional & non-functional requirements |
| [docs/design.md](docs/design.md) | Desain teknis & arsitektur |
| [docs/planning.md](docs/planning.md) | Roadmap bertahap & rencana eksekusi |
| [docs/rancangan_frontend.md](docs/rancangan_frontend.md) | Rancangan UI/UX & teknis frontend (dashboard) |
| [docs/RANCANGAN.md](docs/RANCANGAN.md) | Draf arsitektur eksplorasi (multi-backend) |
| [docs/STACK.md](docs/STACK.md) | Rekomendasi library Go (terbaru, permisif) |

## Status

Tahap **spesifikasi & perencanaan**. Implementasi mulai dari Fase 0 (fondasi)
lalu Fase 1 (MVP monitoring untuk satu backend — rekomendasi: MikroTik).
