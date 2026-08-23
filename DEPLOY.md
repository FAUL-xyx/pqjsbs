# Deploy ke VPS — Panduan Step-by-Step

Panduan ini mengasumsikan VPS **Ubuntu 22.04** (DigitalOcean, Vultr, Contabo,
Hetzner, dsb — semua mirip). Waktu total sekitar 30–45 menit kalau domain
sudah siap.

Yang akan kamu dapat di akhir: `https://domainmu.com` menampilkan kartu
digitalmu untuk siapa saja, dan `https://domainmu.com/admin.html` untuk
mengeditnya — semua data tersimpan di server, bukan di browser pengunjung.

---

## 0. Yang perlu disiapkan dulu

- VPS aktif (IP publik-nya, contoh: `203.0.113.10`)
- Domain yang sudah kamu beli (contoh: `kartuku.com`), dengan akses ke
  pengaturan DNS-nya
- Akses SSH ke VPS (biasanya `ssh root@203.0.113.10`)

---

## 1. Arahkan domain ke VPS

Di dashboard penyedia domainmu (Namecheap, Niagahoster, Cloudflare, dll),
buat **A record**:

| Type | Name | Value             |
|------|------|-------------------|
| A    | @    | `203.0.113.10`    |
| A    | www  | `203.0.113.10`    |

Propagasi DNS biasanya 5–30 menit (kadang sampai beberapa jam). Cek dengan:

```bash
ping kartuku.com
```

Kalau sudah menunjuk ke IP VPS-mu, lanjut.

---

## 2. Masuk ke VPS & update sistem

```bash
ssh root@203.0.113.10
apt update && apt upgrade -y
```

Buat user baru (opsional tapi disarankan, jangan selalu pakai root):

```bash
adduser deploy
usermod -aG sudo deploy
su - deploy
```

---

## 3. Install Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v   # pastikan v20.x muncul
npm -v
```

---

## 4. Upload project ke VPS

Paling gampang pakai `git`. Kalau projectmu belum ada di GitHub, upload dulu
(lihat panduan sebelumnya soal push ke GitHub), lalu:

```bash
sudo apt install -y git
git clone https://github.com/username/nama-repo.git
cd nama-repo
```

Atau, kalau mau upload langsung dari komputermu tanpa git, dari **komputer
lokal** (bukan di VPS):

```bash
scp -r ./project deploy@203.0.113.10:/home/deploy/nama-repo
```

---

## 5. Install dependency & konfigurasi server

```bash
cd nama-repo/server
npm install
cp .env.example .env
```

Buka `.env` dan isi `JWT_SECRET` dengan string acak yang panjang:

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

Salin hasilnya, lalu edit:

```bash
nano .env
```

```
PORT=3000
JWT_SECRET=<tempel-hasil-random-di-sini>
```

Simpan (`Ctrl+O`, Enter, `Ctrl+X`).

**Test jalan dulu:**

```bash
node server.js
```

Harus muncul: `Digital Business Card server running on port 3000`.
Cek dari VPS: `curl localhost:3000` (harus keluar HTML). Tekan `Ctrl+C`
untuk stop, lalu lanjut ke langkah PM2 supaya server jalan terus-menerus.

---

## 6. Jalankan server 24/7 dengan PM2

```bash
sudo npm install -g pm2
pm2 start server.js --name digital-card
pm2 save
pm2 startup
```

Perintah terakhir (`pm2 startup`) akan menampilkan satu baris perintah
`sudo env PATH=...` — **copy-paste dan jalankan** baris itu supaya PM2
otomatis start lagi kalau VPS reboot.

Cek status:

```bash
pm2 status
pm2 logs digital-card
```

---

## 7. Pasang Nginx sebagai reverse proxy

Nginx akan menerima trafik di port 80/443 (web normal) dan meneruskannya
ke Node.js yang jalan di port 3000.

```bash
sudo apt install -y nginx
sudo nano /etc/nginx/sites-available/digital-card
```

Isi dengan:

```nginx
server {
    listen 80;
    server_name kartuku.com www.kartuku.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Aktifkan:

```bash
sudo ln -s /etc/nginx/sites-available/digital-card /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Sekarang `http://kartuku.com` (belum HTTPS) harusnya sudah menampilkan
kartumu.

---

## 8. Pasang SSL gratis (HTTPS) dengan Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d kartuku.com -d www.kartuku.com
```

Ikuti prompt-nya (masukkan email, setuju ToS). Certbot otomatis mengubah
konfigurasi Nginx untuk redirect HTTP → HTTPS dan pasang sertifikat.
Sertifikat auto-renew sendiri; test renewal-nya:

```bash
sudo certbot renew --dry-run
```

Sekarang `https://kartuku.com` sudah aktif dengan gembok hijau.

---

## 9. Amankan firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

Ini memastikan hanya port 22 (SSH), 80, dan 443 yang terbuka — port 3000
Node tidak perlu dibuka ke publik karena hanya diakses lewat Nginx secara
lokal.

---

## 10. Setup PIN pertama kali

Buka `https://kartuku.com/admin.html` di browser. Karena ini pertama kali,
kamu akan diminta membuat PIN — PIN ini akan di-hash dan disimpan di
`server/data/pin.hash` di VPS, bukan di browser. Siapa pun yang tahu PIN
ini bisa login dari device manapun.

Lanjut edit profil, link, tema seperti biasa — semua tersimpan langsung
ke server dan langsung terlihat oleh siapa pun yang membuka
`https://kartuku.com`.

---

## 11. Update project di kemudian hari

Kalau kamu edit kode (bukan lewat dashboard, tapi ubah file HTML/CSS/JS)
dan mau deploy ulang:

```bash
cd ~/nama-repo
git pull
cd server
npm install     # kalau ada dependency baru
pm2 restart digital-card
```

---

## 12. Backup data

Data pentingmu ada di dua file di VPS:

```
server/data/store.json   → semua isi profil, link, tema, analytics
server/data/pin.hash      → PIN admin (ter-hash)
```

Backup berkala, misalnya:

```bash
scp deploy@203.0.113.10:~/nama-repo/server/data/store.json ./backup-$(date +%F).json
```

---

## Catatan keamanan

- Ganti `JWT_SECRET` di `.env` dengan nilai unik milikmu sendiri — jangan
  pakai contoh dari dokumen ini.
- Endpoint login dibatasi 10 percobaan / 15 menit per IP (lihat
  `server/server.js`), tapi PIN pendek tetap lebih lemah dari password
  panjang — pakai PIN 6 digit dan jangan yang mudah ditebak (tanggal
  lahir, `123456`, dst).
- `server/data/store.json` dan `pin.hash` **jangan** pernah di-commit ke
  git publik (sudah ada di `.gitignore`) — itu berisi data live kamu.
- Untuk trafik tinggi atau banyak orang mengedit bersamaan, `store.json`
  berbasis file biasa mulai jadi batasan — di titik itu pertimbangkan
  migrasi ke SQLite/Postgres (struktur `readData()`/`writeData()` di
  `server.js` sengaja dibuat terpusat supaya gampang diganti nanti).
