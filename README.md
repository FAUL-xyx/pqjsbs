# Digital Business Card — Premium Interactive (Server-backed)

Kartu nama digital satu halaman dengan efek 3D ringan, gaya *dark luxury*
(charcoal + champagne gold), plus dashboard editor. Sekarang datanya
tersimpan di **server** (bukan browser masing-masing pengunjung), jadi bisa
benar-benar dipakai orang lain lewat internet.

Mau langsung deploy ke VPS supaya bisa diakses publik? Baca **`DEPLOY.md`**.
Dokumen ini untuk menjalankan/mencoba di komputermu sendiri dulu.

## Menjalankan di komputer sendiri (local dev)

Butuh [Node.js](https://nodejs.org) versi 18 ke atas.

```bash
cd server
npm install
cp .env.example .env
```

Edit `.env`, isi `JWT_SECRET` dengan string acak (jangan pakai contoh
bawaan):

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

Jalankan servernya:

```bash
npm start
```

Buka:
- `http://localhost:3000` — kartu publik
- `http://localhost:3000/admin.html` — dashboard editor (akan minta buat
  PIN saat pertama kali dibuka)

## Cara mengedit / menambah profil

1. Buka `/admin.html`, buat/masukkan PIN.
2. Sidebar untuk pindah bagian:
   - **Profile** — nama, username, jabatan, bio, foto, logo, email, WhatsApp, status.
   - **Links** — tambah/edit/hapus tautan sosial & custom, aktifkan/nonaktifkan,
     seret ikon `⠿` untuk ubah urutan.
   - **Gaming** — profil game (Mobile Legends, Free Fire, Roblox, dst).
   - **Appearance** — 6 preset tema atau kustomisasi warna/radius/shadow/3D/animasi.
   - **Analytics** — views, klik per link, share.
   - **Settings** — ganti PIN (perlu PIN lama), export/import JSON, reset data.
3. Panel **Live Preview** kanan (atau tombol "Preview" di mobile) update
   otomatis setiap perubahan — tanpa reload.
4. Tombol **Share** di kartu publik: Copy Link, Download QR, share
   WhatsApp/Telegram, atau Web Share API bawaan browser.

## Struktur project

```
/project
├── DEPLOY.md              → panduan deploy ke VPS step-by-step
├── README.md
│
├── server/
│   ├── server.js           → Express: API + serve frontend
│   ├── package.json
│   ├── .env.example
│   ├── .gitignore
│   └── data/               → dibuat otomatis saat pertama kali run
│       ├── store.json       → semua data profil/link/tema/analytics
│       └── pin.hash          → PIN admin (ter-hash, bukan plaintext)
│
└── public/                 → semua file yang dilayani ke browser
    ├── index.html            → kartu publik
    ├── admin.html             → dashboard editor
    ├── css/
    │   ├── style.css          → design tokens & style kartu publik
    │   └── admin.css           → style dashboard
    └── js/
        ├── storage.js          → API client (fetch/save ke server, auth token)
        ├── cards.js             → render kartu (dipakai publik & preview)
        ├── app.js                → logika halaman publik (tilt, share, tracking)
        ├── editor.js              → logika dashboard (form, drag&drop, tema)
        ├── animation.js            → efek tilt 3D + animasi intro (GSAP)
        └── share.js                 → share sheet (copy link, QR, WA/Telegram)
```

## Bagaimana data & keamanan bekerja sekarang

- **Data** disimpan di `server/data/store.json` (file JSON di server) —
  dibaca/ditulis lewat REST API sederhana di `server.js`. Semua pengunjung
  melihat data yang sama, dan admin yang login dari device manapun bisa
  mengedit data yang sama itu.
- **Login admin**: PIN di-hash (bcrypt) dan disimpan di
  `server/data/pin.hash` — bukan plaintext, bukan pula di browser. Saat
  login berhasil, server memberi token (JWT) berlaku 12 jam yang disimpan
  di `localStorage` browser kamu sendiri hanya untuk mengingat sesi;
  setiap permintaan simpan data selalu dicek ulang oleh server.
- **Rate limiting** otomatis pada endpoint login (10 percobaan/15 menit
  per IP) untuk memperlambat tebak-PIN.
- Kalau butuh skala lebih besar (trafik tinggi, banyak admin sekaligus),
  ganti `readData()`/`writeData()` di `server.js` dengan database
  sungguhan (SQLite/Postgres) — bagian lain kode tidak perlu diubah
  karena semua akses data lewat dua fungsi itu.

## Fitur yang sudah berfungsi penuh

- Kartu utama dengan efek 3D tilt (pointer & sentuhan), parallax antar
  layer, moving highlight — ringan di mobile.
- Kartu link dengan hover/press animation.
- Animasi intro stagger saat halaman dibuka (menghormati
  `prefers-reduced-motion`, juga ada saklar manual di Appearance).
- Dashboard penuh: profil, link (CRUD + toggle + drag&drop), gaming,
  tema (6 preset + kustomisasi), analytics, settings (PIN, export/import,
  reset).
- Live preview real-time tanpa reload (lewat `postMessage`, bukan lagi
  bergantung pada localStorage).
- Share sheet: copy link, QR code (generate+download), WhatsApp/Telegram,
  Web Share API native.
- Sepenuhnya responsive (mobile/tablet/desktop).

## Deploy supaya bisa diakses publik

Lihat **`DEPLOY.md`** untuk panduan lengkap step-by-step: provision VPS,
install Node, jalankan dengan PM2, reverse proxy Nginx, domain, SSL
gratis (Let's Encrypt), firewall, sampai backup data.
