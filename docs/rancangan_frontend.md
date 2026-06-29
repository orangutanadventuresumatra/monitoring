# Rancangan Frontend — Dashboard Monitoring & Manajemen Jaringan

> Desain UI/UX dan teknis frontend untuk sistem NMS multi-backend.
> Selaras dengan `requirements.md` & `design.md`.
> Prinsip: **realtime, ringan, dan jauh lebih mudah dipakai daripada Zabbix.**

- **Versi:** 1.0
- **Acuan:** requirements.md, design.md, STACK.md

---

## 1. Tujuan & Prinsip Desain

1. **Realtime-first** — data tampil live via WebSocket tanpa refresh manual.
2. **Sederhana & intuitif** — "semua cabang dalam satu layar", drill-down jelas.
   Ini pembeda jualan utama (Zabbix terkenal rumit).
3. **Ringan** — di-build statis & **di-embed ke binary Go** (`go:embed`), tanpa
   server frontend terpisah.
4. **Status berbasis warna** — hijau/kuning/merah agar kondisi cepat terbaca.
5. **Lokalisasi** — Bahasa Indonesia (default) + English.
6. **Responsif** — desktop NOC utama, tetap layak di tablet/ponsel.

---

## 2. Tech Stack Frontend

| Kebutuhan | Pilihan | Alasan |
|-----------|---------|--------|
| Framework UI | **React 18+** (TypeScript) | Ekosistem besar, mudah cari developer |
| Build tool | **Vite** | Build cepat → output statis untuk go:embed |
| Routing | **React Router** | Navigasi SPA |
| State server & cache | **TanStack Query** | Fetch REST + cache + invalidation rapi |
| State realtime | **Zustand** (store ringan) | Tampung delta WebSocket → komponen |
| Grafik | **Apache ECharts** (`echarts-for-react`) | Grafik time-series performa tinggi |
| Styling | **Tailwind CSS** + komponen headless (mis. shadcn/ui/Radix) | Cepat, konsisten, dark mode mudah |
| Form & validasi | **React Hook Form** + **Zod** | Form kebijakan & pengaturan |
| Tabel besar | **TanStack Table** + virtualisasi | Daftar device/host banyak baris |
| i18n | **react-i18next** | ID/EN |
| Ikon | **lucide-react** | Konsisten & ringan |

Semua berlisensi permisif (MIT/Apache). Output `vite build` → folder statis →
di-embed ke binary central.

---

## 3. Arsitektur Data Frontend

```
   REST (chi)  ──▶ TanStack Query  ──▶ data "tidak-realtime"
   (login, daftar cabang/host,        (histori, kebijakan, pengaturan)
    kebijakan, histori)

   WebSocket   ──▶ WS Client ──▶ Zustand store ──▶ komponen live
   (delta JSON)    (reconnect)    (merge delta)     (kartu/grafik update)
```

### Penanganan WebSocket (live)
- Buka 1 koneksi WS ke central setelah login (token di header/subprotocol).
- Terima **delta JSON** → merge ke store (hanya field berubah).
- **Reconnect** otomatis (exponential backoff) + indikator "Live / Reconnecting".
- **Throttle render** (mis. batch update tiap 250–500 ms) agar UI tetap mulus
  walau banyak cabang.

---

## 4. Information Architecture (Struktur Navigasi)

```
Login
└── App Shell (sidebar + topbar)
    ├── Overview            ← semua cabang dalam satu layar (peta/grid status)
    ├── Cabang
    │   ├── Daftar cabang
    │   └── Detail cabang   ← device, bandwidth live, top talkers
    ├── Bandwidth
    │   ├── Kebijakan (prioritas & batas down/up)
    │   └── Captive Portal (sesi tamu, konfigurasi login page)
    ├── Host Monitoring (URL)
    │   ├── Daftar host      ← status, status code, latency
    │   ├── Detail host      ← grafik + visualisasi route (traceroute)
    │   └── Log Insiden      ← catatan respons ≠200
    ├── Alerts               ← daftar & acknowledge
    └── Pengaturan
        ├── Pengguna & Role (RBAC)
        ├── Backend per cabang
        ├── Notifikasi (Email/Telegram/Slack)
        └── Lisensi
```

Topbar: indikator koneksi **Live**, pemilih bahasa, dark mode, profil/logout.

---

## 5. Desain Halaman Kunci

### 5.1 Overview — "Semua Cabang dalam Satu Layar"
- **Grid kartu cabang**: nama, status warna, jumlah device online/total,
  bandwidth agregat (sparkline), jumlah alert aktif.
- Ringkasan global di atas: total cabang up/down, total device online, alert
  kritis, total bandwidth.
- (Opsional) **peta** lokasi cabang dengan titik berwarna status.
- Klik kartu → Detail cabang.

### 5.2 Detail Cabang
- Header: status, backend yang dipakai (PA/Forti/MT/Linux), last seen.
- **Grafik bandwidth live** (ECharts area, RX/TX) per interface.
- **Tabel device**: IP/MAC/hostname, status online/offline, RX/TX live.
- **Top talkers**: device pemakai bandwidth terbesar.

### 5.3 Bandwidth — Kebijakan
- Tabel kebijakan: target (IP/subnet/device), tipe (prioritas/limit), nilai
  down/up, status penerapan.
- Form tambah/edit (React Hook Form + Zod): pilih target, set prioritas atau
  batas kecepatan → tombol **Terapkan** → tampilkan status sukses/gagal dari
  driver backend.
- Badge "tidak didukung pada backend ini" bila backend cabang tak mendukung.

### 5.4 Bandwidth — Captive Portal
- Daftar **sesi tamu aktif**: MAC, user, VLAN, login, kuota/sisa, masa berlaku.
- Konfigurasi **landing page**: branding (logo, warna, teks), metode login
  (voucher/akun), batas bandwidth & kuota tamu.
- Tombol cabut sesi.

### 5.5 Host Monitoring (URL)
- **Daftar host**: nama, URL, status (🟢/🟡/🔴), status code terakhir, latency,
  TTFB, success rate.
- Tombol **Tambah URL** (scope: dari central / dari agent cabang, interval).
- **Detail host**:
  - Grafik latency & TTFB historis (ECharts).
  - **Visualisasi route (traceroute)** — daftar/diagram hop dengan latency,
    tandai hop lambat/loss.
  - Status code terkini & indikator kesehatan.
- **Log Insiden**: tabel kejadian respons ≠200 (waktu, status code, latency,
  error, link ke route saat itu) — dapat difilter & diekspor.

### 5.6 Alerts
- Daftar alert: severity (warna), sumber, pesan, waktu, status acknowledge.
- Aksi acknowledge/resolve; filter per cabang/severity.

### 5.7 Pengaturan
- **Pengguna & Role**: kelola user, peran admin/operator/viewer.
- **Backend per cabang**: jenis backend + kredensial (disimpan terenkripsi di
  server; UI hanya kirim, tak menampilkan rahasia).
- **Notifikasi**: konfigurasi channel.
- **Lisensi**: info masa berlaku, batas cabang/device, fitur aktif, unggah file
  lisensi.

---

## 6. Komponen Reusable

- `StatusBadge` (warna + label dari HealthScore/threshold)
- `BranchCard`, `MetricCard`
- `LiveLineChart` / `Sparkline` (wrapper ECharts dengan update delta)
- `DeviceTable`, `IncidentTable` (TanStack Table + virtualisasi)
- `RouteTimeline` (visualisasi hop traceroute)
- `PolicyForm`, `CaptivePortalForm`
- `ConnectionIndicator` (status WebSocket Live/Reconnecting)
- `RoleGuard` (sembunyikan/aktifkan aksi sesuai RBAC)

---

## 7. Visual & UX

- **Skema warna status:** hijau (sehat) / kuning (waspada) / merah (buruk/down)
  / abu-abu (stale/tak terjangkau).
- **HealthScore** opsional ditampilkan sebagai angka 0–100 untuk ramah
  manajemen, dengan drill-down ke metrik mentah (RTT/jitter/loss).
- **Dark mode** default-friendly untuk layar NOC.
- **Empty & error states** yang jelas (mis. "Cabang belum mengirim data").
- **Skeleton loading** saat fetch awal.

---

## 8. Performa Frontend

- **Virtualisasi** tabel device/host besar.
- **Throttle/batch** penerapan delta WebSocket (250–500 ms).
- **Lazy-load** halaman berat (code splitting via Vite).
- **Downsampling** data historis di server sebelum dikirim ke grafik.
- Grafik live menyimpan jendela bergulir (mis. 5–15 menit) di memori.

---

## 9. Struktur Folder (frontend/)

```
frontend/
├── index.html
├── vite.config.ts
├── src/
│   ├── main.tsx
│   ├── app/            # routing, app shell, providers
│   ├── pages/          # overview, branches, bandwidth, hosts, alerts, settings
│   ├── components/     # komponen reusable
│   ├── features/       # logika per domain (branches, policies, hosts...)
│   ├── lib/
│   │   ├── api.ts      # klien REST (TanStack Query)
│   │   ├── ws.ts       # klien WebSocket + reconnect + merge delta
│   │   └── store.ts    # Zustand store realtime
│   ├── i18n/           # id.json, en.json
│   └── styles/
└── dist/               # hasil build → di-embed ke binary central (go:embed)
```

---

## 10. Integrasi dengan Backend

- Saat build rilis: `vite build` → `frontend/dist/` → di-`go:embed` ke binary
  central; central men-serve aset statis + endpoint `/api/*` (REST) dan
  `/ws` (WebSocket).
- Kontrak data mengikuti model di `design.md` (TargetStatus, BranchSummary,
  url_checks, incident_logs, dll.).

---

## 11. Lokalisasi (i18n)

- Default **Bahasa Indonesia**, opsi **English**.
- Semua teks via key i18n; tanggal/waktu mengikuti zona waktu pengguna.

---

## 12. Aksesibilitas

- Kontras warna memadai (status tidak hanya bergantung warna — sertakan ikon/
  label).
- Navigasi keyboard pada tabel & form.
- Label ARIA pada komponen interaktif.
