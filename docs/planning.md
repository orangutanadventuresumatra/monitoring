# Planning — Sistem Monitoring & Manajemen Jaringan Multi-Backend

> Rencana eksekusi bertahap untuk mewujudkan `requirements.md` & `design.md`.
> Filosofi: **MVP dulu, satu backend dulu, validasi, lalu perluas.**

- **Versi:** 1.0
- **Acuan:** requirements.md, design.md, STACK.md

---

## 1. Prinsip Eksekusi

1. **Mulai sempit** — satu backend, fitur inti, buktikan realtime + minimal
   bandwidth bekerja.
2. **Interface lebih dulu** — definisikan kontrak (driver/transport/store)
   sebelum implementasi, agar perluasan murah.
3. **Vertical slice** — tiap fase menghasilkan sesuatu yang bisa dilihat &
   dipakai end-to-end (collector → central → dashboard).
4. **Validasi** — uji ke jaringan nyata / calon pelanggan sebelum menambah scope.

---

## 2. Rekomendasi Backend Pertama

| Kandidat | Alasan dipilih duluan | Catatan |
|----------|------------------------|---------|
| **MikroTik** ⭐ | Termurah, API lengkap (`go-routeros`), **Hotspot** captive portal bawaan, Simple Queue mudah untuk limit per-IP | Tercepat membuktikan SEMUA fitur (1–3) jalan |
| Linux mandiri | Paling fleksibel & "milik sendiri"; cocok jadi produk | Paling banyak kerja (inline, HA, tc/nftables/RADIUS) |

**Keputusan disarankan:** mulai **MikroTik** untuk MVP (cepat lihat hasil &
captive portal), lalu **Linux** sebagai backend kedua (nilai jual mandiri),
baru Palo Alto/Fortinet.

---

## 3. Roadmap Bertahap

### Fase 0 — Fondasi & Kerangka (enabler)
**Tujuan:** kerangka project, interface, pipa data dasar.
- [ ] Struktur monorepo: `central/`, `agent/`, `proto/`, `frontend/`, `deploy/`.
- [ ] Definisi interface: `EnforcementBackend`, `TrafficSource`,
      `TelemetryTransport`, `Store`.
- [ ] Skema Protobuf untuk `DeltaBatch` & `Command`.
- [ ] Store: in-memory (dev) + skema TimescaleDB (goose) + impl `pgx`.
- [ ] Setup Docker (multi-stage, multi-arch) + docker-compose central.
- [ ] CI: build, `go vet`, `golangci-lint`, unit test.

**Exit criteria:** `central` & `agent` bisa di-build; agent dummy mengirim
DeltaBatch ke central via WebSocket; tersimpan di store.

### Fase 1 — MVP Monitoring (FR-1, FR-3 active, FR-5, FR-6) — 1 backend
**Tujuan:** monitoring realtime trafik & host yang benar-benar berguna.
- [ ] Agent: ICMP/TCP/HTTP check (pro-bing, net/http) + agregasi.
- [ ] Agent: buffer disk tahan-putus-WAN + reconnect/backoff + resume `seq`.
- [ ] Transport: WebSocket + Protobuf + zstd + **delta** (agent→central).
- [ ] Central: WS Hub, ingest, broadcast delta ke browser (JSON).
- [ ] Driver MikroTik (`TrafficSource`): interface stats, device list
      (online/offline), per-client stats.
- [ ] Fitur 3 active probe: tambah URL, cek berkala (gocron), status code/TTFB/
      latency, **log saat ≠200** + traceroute (route_json).
- [ ] Enrollment cabang via token.
- [ ] Dashboard React + ECharts: status cabang, grafik bandwidth live, daftar
      device, status URL host + log insiden.
- [ ] Auth dasar (login + JWT) & RBAC minimal.

**Exit criteria (= Kriteria MVP di requirements §7):** monitoring trafik &
device + active probe + log non-200 + route, realtime via WebSocket, telemetri
hemat bandwidth terverifikasi, multi-cabang dasar.

### Fase 2 — Manajemen Bandwidth (FR-2.1–2.3) — backend MikroTik
- [ ] Policy Engine + model kebijakan netral.
- [ ] Driver MikroTik (`EnforcementBackend`): rate limit per IP (Simple Queue),
      prioritas host.
- [ ] Alur apply kebijakan + konfirmasi sukses/gagal + **audit log**.
- [ ] UI pengaturan kebijakan + status penerapan.

### Fase 3 — Captive Portal (FR-2.4–2.5)
- [ ] Landing page login (di-host central/agent).
- [ ] Integrasi MikroTik Hotspot / RADIUS; pemisahan VLAN tamu vs karyawan.
- [ ] Batas bandwidth & kuota tamu; manajemen sesi (`captive_sessions`).

### Fase 4 — Multi-Vendor & Passive (FR-4 penuh, FR-3.6)
- [ ] Driver **Linux mandiri** (go-tc, nftables, netlink, RADIUS + OpenNDS
      proses terpisah) — topologi inline `Internet→PaloAlto→Linux→Device`.
- [ ] Driver **Palo Alto** (pango) & **Fortinet** (REST).
- [ ] Driver **contract tests** (satu suite untuk semua backend).
- [ ] Passive monitoring akses user (log firewall/flow/DNS) + visualisasi route
      lanjutan.
- [ ] High Availability untuk backend Linux inline (failover).

### Fase 5 — Produk Komersial (FR-9, FR-10, hardening)
- [ ] License Manager (Ed25519): batasi cabang/device/fitur/expired + grace
      period; online activation opsional.
- [ ] Obfuscation (garble) + pipeline rilis image multi-arch + offline tarball.
- [ ] Multi-tenancy/isolasi, tiering fitur.
- [ ] Hardening keamanan (secret management, TLS wajib), dokumentasi instalasi.
- [ ] Alerting multi-channel lengkap (shoutrrr) + manajemen alert.

---

## 4. Milestone & Demo

| Milestone | Selesai saat | Demo |
|-----------|--------------|------|
| M0 Fondasi | Fase 0 | Data dummy mengalir agent→central→DB |
| M1 MVP | Fase 1 | Dashboard live: cabang up/down, bandwidth, URL host + log non-200 |
| M2 Kontrol | Fase 2–3 | Atur limit/prioritas + captive portal tamu dari dashboard |
| M3 Multi-vendor | Fase 4 | Backend Linux + PA/Forti via interface yang sama |
| M4 Produk | Fase 5 | Instalasi berlisensi, siap dijual |

---

## 5. Dependensi & Urutan

```
Fase 0 (interface, transport, store, docker)
   └─▶ Fase 1 (monitoring MVP, 1 backend)        ← buktikan realtime + hemat BW
          └─▶ Fase 2 (kontrol bandwidth)
                 └─▶ Fase 3 (captive portal)
                        └─▶ Fase 4 (multi-vendor + passive + HA)
                               └─▶ Fase 5 (lisensi, packaging jual)
```
Kunci: **interface di Fase 0** membuat Fase 4 (multi-vendor) tidak membongkar
apa pun.

---

## 6. Risiko & Mitigasi

| Risiko | Dampak | Mitigasi |
|--------|--------|----------|
| Scope membengkak (4 backend sekaligus) | Proyek tak selesai | Satu backend dulu (MikroTik); sisanya di balik interface |
| Captive portal bergantung AP | Fitur tamu gagal | Verifikasi dukungan VLAN+redirect AP di onboarding |
| Backend Linux = titik kegagalan inline | Internet cabang putus | HA (keepalived/VRRP), hardware andal, monitoring |
| Status code HTTPS pasif tak terlihat | Ekspektasi keliru | Andalkan active probe; passive hanya domain/SNI tanpa decryption |
| Lisensi perangkat membatasi fitur API | Fitur tak jalan di pelanggan | Cek versi/lisensi PAN-OS/FortiOS/RouterOS saat onboarding |
| Bandwidth telemetri membengkak | Melanggar NFR-2 | Delta + protobuf + zstd + agregasi NetFlow di edge; uji bandwidth |
| Lisensi dependency tak kompatibel jual | Masalah hukum | Hanya library permisif; GPL sebagai proses terpisah; audit lisensi |
| Kepercayaan kontrol firewall (PA/Forti) | Sulit diterima pelanggan | Posisikan kontrol sebagai fitur premium lanjut; approval + audit + rollback |

---

## 7. Definition of Done (per fitur)

- Kode + unit/integration test hijau; `go vet` & lint bersih.
- Memenuhi acceptance criteria terkait di requirements.md.
- Terlihat end-to-end di dashboard (bila relevan).
- Audit log & error handling sesuai design.md.
- Lisensi dependency baru terverifikasi permisif.

---

## 8. Langkah Berikutnya (actionable)

1. **Konfirmasi backend pertama** (rekomendasi: MikroTik).
2. **Mulai Fase 0** — scaffold monorepo + interface + skema protobuf + skema DB
   + setup Docker.
3. Lanjut **Fase 1 (MVP)** sampai milestone M1 untuk validasi.

> Catatan: file fondasi awal sudah ada di `central/` (model & store in-memory)
> dari sesi eksplorasi; akan dirapikan/diselaraskan saat Fase 0 dimulai resmi.
