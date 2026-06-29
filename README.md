# Deploy — Satu Data Kota Singkawang

Konfigurasi deployment untuk portal **Satu Data Kota Singkawang** (Next.js) dan backend
**CKAN**, di belakang reverse proxy.

- **Portal (publik):** `satudata.singkawangkota.go.id` → aplikasi Next.js
  ([repo aplikasi: `satudata-skw`](https://github.com/dhestaadeanggajuanda/satudata-skw))
- **Backend:** `data.singkawangkota.go.id` → CKAN (admin + REST API)

> **PENTING — placeholder:** semua berkas memakai token **`BACKEND_IP`** sebagai pengganti
> alamat IP LAN server backend. **Ganti `BACKEND_IP`** dengan IP backend Anda yang sebenarnya
> sebelum dipakai (mis. via cari-ganti, atau set lewat `.env`).

## Arsitektur

```
Internet
  │
  ▼
Reverse proxy / WAF (edge)                ← aaPanel + AAWAF (vhost: *.conf)
  ├─ satudata...  →  https://BACKEND_IP          (docker-nginx → portal Next.js)
  └─ data...      →  http://BACKEND_IP:5000       (CKAN langsung)
        │
        ▼
Docker stack (server BACKEND_IP)          ← docker-compose.yml + nginx.conf
  ├─ nginx  (TLS, security headers, CSP, rate-limit)
  └─ portal (Next.js standalone, :3000)
```

- **Build-time** portal mengambil data CKAN via IP internal (`DMS_INTERNAL`) — cepat & tak
  bergantung DNS/SSL publik. **URL yang sampai ke browser** di-rebase ke host publik
  (`CKAN_PUBLIC_URL` / `DMS_PUBLIC`), jadi tak ada IP internal yang bocor ke klien.

## Berkas

| Berkas | Fungsi |
|---|---|
| `docker-compose.yml` | Stack portal: build dari repo `satudata-skw` + nginx reverse proxy |
| `nginx.conf` | Reverse proxy docker (TLS, security headers, CSP ketat, rate-limit) |
| `satudata.singkawangkota.go.id.conf` | Contoh vhost edge (aaPanel) untuk portal |
| `data.singkawangkota.go.id.conf` | Contoh vhost edge (aaPanel) untuk CKAN |
| `.env.example` | Template variabel lingkungan (salin ke `.env`) |

## Cara pakai

1. **Siapkan env:**
   ```bash
   cp .env.example .env
   # edit .env: ganti BACKEND_IP, sesuaikan DMS_PUBLIC & path sertifikat SSL
   ```
   | Variabel | Keterangan |
   |---|---|
   | `DMS_INTERNAL` | Host FETCH CKAN (IP LAN) — I/O server-side build & runtime proxy |
   | `DMS_PUBLIC` | Host PUBLIK CKAN — URL unduh/gambar di browser + allowlist proxy |
   | `SSL_CERT_PATH` / `SSL_KEY_PATH` | Path sertifikat & kunci SSL di host |

2. **Ganti `BACKEND_IP`** dengan IP backend nyata di `nginx.conf` dan vhost `*.conf`
   (atau hanya di `.env` bila memakai docker-compose saja).

3. **Jalankan stack portal:**
   ```bash
   docker compose build --no-cache portal
   docker compose up -d
   ```

4. **Edge (aaPanel):** gunakan `satudata.singkawangkota.go.id.conf` &
   `data.singkawangkota.go.id.conf` sebagai acuan konfigurasi vhost reverse proxy + header
   keamanan. Aktifkan **Force HTTPS** dan batasi TLS ke **1.2/1.3**.

## Catatan keamanan

- **`.env` tidak di-commit** (lihat `.gitignore`) — rahasia hanya di server. Pakai
  `.env.example` sebagai template.
- Dokumen hasil *security assessment* sengaja **tidak** disertakan di repo publik ini.
