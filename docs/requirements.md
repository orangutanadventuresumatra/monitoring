# Requirements — Sistem Monitoring & Manajemen Jaringan Multi-Backend

> Produk **self-hosted komersial** untuk memantau dan mengatur jaringan banyak
> cabang dari satu pusat, dengan backend enforcement yang dapat dipilih
> (Palo Alto / Fortinet / MikroTik / Linux mandiri). Fokus utama: **realtime**
> dan **minimal bandwidth**.

- **Versi dokumen:** 1.0
- **Status:** Draft untuk persetujuan
- **Bahasa pemrograman:** Go 1.25+

---

## 1. Ringkasan & Tujuan

Membangun platform monitoring + manajemen jaringan yang:
- Dikelola **terpusat** dari HQ untuk N cabang (acuan awal: 32 cabang, ~20
  device/cabang).
- **Tidak terkunci satu vendor** — pelanggan memakai perangkat yang sudah ada
  (Palo Alto/Fortinet/MikroTik) atau Linux server mandiri sebagai gateway.
- Memberi visibilitas **realtime** atas trafik, perangkat, dan kesehatan host,
  dengan konsumsi **bandwidth WAN seminimal mungkin**.
- Dapat **dijual** sebagai produk (lisensi komersial, distribusi binary
  terproteksi).

### Sasaran kualitas utama
1. **Realtime** — perubahan terlihat di dashboard dalam hitungan detik.
2. **Minimal bandwidth** — telemetri antar-cabang ke HQ sangat ringan.

---

## 2. Glosarium

| Istilah | Arti |
|---------|------|
| Central | Komponen pusat di HQ (dashboard, API, DB, alerting, lisensi). |
| Agent / Collector | Komponen di cabang yang mengumpulkan metrik & menjalankan perintah. |
| Enforcement Backend | Perangkat/mekanisme yang menegakkan kebijakan (PA/Forti/MT/Linux). |
| Driver | Adapter yang menerjemahkan perintah seragam ke API/konfig backend tertentu. |
| Active probe | Pengecekan host dengan sistem sendiri yang mengirim request berkala. |
| Passive monitoring | Mengamati trafik user nyata (log/flow/DNS). |
| Delta | Pengiriman hanya data yang berubah, bukan snapshot penuh. |

---

## 3. Functional Requirements

Format acceptance criteria: **WHEN/IF \<kondisi\>, THE SYSTEM SHALL \<perilaku\>**
(disingkat: "Sistem HARUS ...").

### FR-1 — Monitoring Trafik Bandwidth & Status Perangkat
**User story:** Sebagai admin jaringan, saya ingin melihat penggunaan bandwidth
dan status online/offline perangkat secara realtime, agar bisa cepat tahu
kondisi tiap cabang.

Acceptance criteria:
1. Sistem HARUS menampilkan throughput (download/upload) per interface dan per
   device secara realtime (interval dapat dikonfigurasi, default ≤ 5 detik).
2. Sistem HARUS menampilkan daftar perangkat beserta status **online/offline**.
3. WHEN sebuah perangkat berubah status (online↔offline), THE SYSTEM SHALL
   memperbarui dashboard dalam ≤ 2 detik setelah terdeteksi oleh agent.
4. Sistem HARUS menyimpan data historis untuk grafik tren (retensi dapat
   dikonfigurasi).
5. Sistem HARUS menampilkan "top talkers" (perangkat dengan pemakaian terbesar)
   per cabang.

### FR-2 — Manajemen Bandwidth
**User story:** Sebagai admin, saya ingin mengatur prioritas dan batas kecepatan
per host/device, serta memisahkan jaringan tamu vs karyawan, agar bandwidth
terdistribusi adil dan aman.

Acceptance criteria:
1. Sistem HARUS mengizinkan penetapan **prioritas host** (mis. tinggi/normal/
   rendah) per IP/subnet/device.
2. Sistem HARUS mengizinkan penetapan **batas kecepatan download dan upload**
   per IP/device.
3. WHEN admin menyimpan kebijakan bandwidth, THE SYSTEM SHALL menerapkannya ke
   enforcement backend cabang terkait dan mengonfirmasi keberhasilan/kegagalan.
4. Sistem HARUS menyediakan **captive portal / halaman login WiFi** yang
   memisahkan jaringan **karyawan** dan **tamu**.
5. WHEN seorang tamu login melalui captive portal, THE SYSTEM SHALL menempatkan
   perangkatnya pada segmen tamu (VLAN/kebijakan tamu) dengan batas bandwidth
   tamu yang berlaku.
6. IF backend tidak mendukung suatu kemampuan kontrol, THE SYSTEM SHALL
   menampilkan keterangan "tidak didukung pada backend ini" (tidak crash).

### FR-3 — Monitoring Host Berbasis URL (Active + Passive)
**User story:** Sebagai admin, saya ingin menambahkan URL host untuk dipantau,
melihat alur (route) yang dilalui, dan mendapat catatan log saat respons bukan
200, agar cepat tahu host bermasalah.

Acceptance criteria:
1. Sistem HARUS mengizinkan penambahan **URL host** untuk dipantau melalui
   dashboard.
2. Sistem HARUS melakukan **active probe** berkala (interval dapat
   dikonfigurasi) dan mencatat: status code, latency, TTFB, ketersediaan.
3. WHEN respons host **bukan 200** (mis. 4xx/5xx) ATAU timeout/error,
   THE SYSTEM SHALL membuat **entri log insiden** berisi waktu, URL, status
   code, latency, pesan error, dan jejak route saat itu.
4. Sistem HARUS menampilkan **visualisasi alur/route** (hop demi hop, berbasis
   traceroute) ke host, menandai hop yang lambat/putus.
5. WHEN insiden non-200 melewati ambang yang dikonfigurasi, THE SYSTEM SHALL
   mengirim alert melalui channel yang dipilih.
6. (Passive, opsional) Sistem HARUS dapat mencatat **aktivitas akses user** ke
   domain tertentu secara realtime dari sumber pasif (log firewall/flow/DNS),
   dengan ketentuan: untuk HTTPS tanpa SSL-decryption, hanya domain/SNI,
   status koneksi, dan latency yang tersedia (bukan status code aplikasi).

### FR-4 — Multi-Backend (Pluggable Enforcement)
**User story:** Sebagai pemilik produk, saya ingin satu sistem mendukung
beberapa jenis perangkat, agar bisa dijual ke pelanggan dengan infrastruktur
berbeda.

Acceptance criteria:
1. Sistem HARUS menyediakan **antarmuka driver seragam** untuk monitoring dan
   enforcement.
2. Sistem HARUS mendukung pemilihan backend **per cabang**: Palo Alto, Fortinet,
   MikroTik, atau Linux mandiri.
3. WHEN backend baru ditambahkan, THE SYSTEM SHALL hanya memerlukan penambahan
   driver baru tanpa mengubah dashboard, skema data, atau logika inti.
4. IF backend = Linux mandiri, THE SYSTEM SHALL berperan sebagai gateway inline
   pada topologi `Internet → Palo Alto → Linux → Device`.

### FR-5 — Realtime & Distribusi Live
1. Sistem HARUS mengirim pembaruan ke dashboard secara **push** (bukan polling)
   sehingga tampil live tanpa refresh manual.
2. WHEN agent mendeteksi perubahan, THE SYSTEM SHALL meneruskannya ke browser
   yang terhubung dalam ≤ 2 detik (kondisi WAN normal).

### FR-6 — Manajemen Terpusat & Multi-Cabang
1. Sistem HARUS mengelola dan menampilkan banyak cabang dari satu dashboard
   ("semua cabang dalam satu layar").
2. Sistem HARUS mendukung **enrollment** cabang menggunakan token sehingga agent
   mendaftar otomatis dan menerima konfigurasi dari central.
3. WHEN tunnel WAN cabang terputus, THE SYSTEM SHALL membuffer data di agent dan
   mengirim ulang saat koneksi pulih (tanpa kehilangan data dalam batas buffer).

### FR-7 — Alerting & Logging
1. Sistem HARUS mendukung alert berbasis threshold (mis. host down, latency
   tinggi, bandwidth penuh, respons non-200).
2. Sistem HARUS mengirim notifikasi multi-channel (mis. Email, Telegram, Slack).
3. Sistem HARUS menyimpan log insiden dan dapat ditelusuri/diekspor.

### FR-8 — Autentikasi, RBAC & Audit
1. Sistem HARUS mewajibkan autentikasi untuk mengakses dashboard.
2. Sistem HARUS mendukung peran: **admin**, **operator**, **viewer**.
3. WHEN seorang user mengubah kebijakan, THE SYSTEM SHALL mencatat **audit log**
   (siapa, apa, kapan, target).

### FR-9 — Lisensi Komersial
1. Sistem HARUS memvalidasi **file lisensi bertanda tangan (Ed25519)** secara
   offline saat start dan secara berkala.
2. File lisensi HARUS dapat membatasi: jumlah cabang/device, masa berlaku
   (expired), dan fitur yang diaktifkan (tier).
3. WHEN lisensi kedaluwarsa, THE SYSTEM SHALL memberi masa tenggang (grace
   period) dengan peringatan sebelum membatasi fungsi.
4. (Opsional) Sistem HARUS mendukung **online activation/heartbeat** yang dapat
   dinonaktifkan untuk pelanggan air-gapped.

### FR-10 — Deployment
1. Sistem HARUS didistribusikan sebagai **binary Go terkompilasi** (bukan source)
   dan/atau **image Docker** multi-arch (amd64/arm64).
2. Sistem HARUS dapat dijalankan via **docker-compose** (central) dan container
   tunggal (agent/collector cabang).
3. Frontend HARUS di-embed ke dalam binary central (go:embed) agar deployment
   ringkas.

---

## 4. Non-Functional Requirements

### NFR-1 Performa & Realtime
- Latensi end-to-end perubahan → dashboard ≤ 2 detik (WAN normal).
- Interval telemetri default ≤ 5 detik, dapat dikonfigurasi (1–60 detik).

### NFR-2 Efisiensi Bandwidth (prioritas utama)
- Telemetri agent→central HARUS memakai **transport persisten** (WebSocket),
  **encoding biner** (Protobuf), **kompresi** (zstd), dan **delta**.
- Data NetFlow HARUS **diagregasi di edge**; flow mentah TIDAK dikirim ke HQ.
- Target: konsumsi telemetri ~≤ 1 kbps per cabang pada kondisi normal.

### NFR-3 Keamanan
- Komunikasi terenkripsi (TLS) untuk semua jalur ke central.
- Kredensial API perangkat disimpan terenkripsi (secret management).
- Password di-hash (bcrypt/argon2). Token sesi via JWT.
- Distribusi binary di-obfuscate (garble).

### NFR-4 Keandalan
- Agent tahan putus WAN (buffer + retry).
- Untuk backend Linux inline: dukungan opsional **High Availability** (failover)
  karena menjadi titik kegagalan jalur trafik.

### NFR-5 Maintainability
- Arsitektur modular dengan interface (driver, store, transport) agar mudah
  diperluas/ditukar.
- Dependency minimal & berlisensi permisif (lihat STACK.md).

### NFR-6 Kompatibilitas
- Central & agent berjalan di Linux; image multi-arch (amd64/arm64).
- Captive portal bergantung pada dukungan VLAN & redirect pada Access Point.

### NFR-7 Lisensi & Kepatuhan
- Semua dependency yang di-*link* ke binary HARUS berlisensi permisif
  (MIT/BSD/Apache/ISC).
- Komponen GPL (mis. OpenNDS/FreeRADIUS) HANYA dipakai sebagai proses terpisah,
  tidak di-link ke binary produk.

---

## 5. Batasan & Asumsi

- Antar-cabang dan HQ terhubung via **tunnel Palo Alto** (jaringan internal
  routable).
- Tiap cabang memiliki server existing; untuk peran **gateway Linux inline**
  diperlukan mesin/VM **dedicated**.
- Ketersediaan fitur API perangkat bergantung pada **versi & lisensi** perangkat
  (PAN-OS/FortiOS/RouterOS) → diverifikasi saat onboarding.
- Status code untuk akses HTTPS pasif hanya tersedia bila ada SSL-decryption di
  firewall.

---

## 6. Di Luar Lingkup (Out of Scope) — untuk versi awal

- Menggantikan fungsi keamanan perimeter Palo Alto (IPS/AV/threat prevention).
- SSL-decryption sebagai fitur wajib (hanya opsional bila pelanggan
  mengaktifkannya di firewall).
- Orkestrasi Kubernetes (cukup docker-compose untuk awal).
- Modul billing/pembayaran otomatis (penerbitan lisensi manual dulu).

---

## 7. Kriteria Penerimaan Rilis MVP

- FR-1 (monitoring trafik & device) dan FR-3 (active probe + log non-200 +
  route) berjalan untuk **satu backend** (rekomendasi: MikroTik atau Linux).
- Realtime via WebSocket dan telemetri hemat bandwidth (NFR-2) terbukti.
- Enrollment cabang (FR-6.2) dan dashboard multi-cabang dasar berfungsi.
