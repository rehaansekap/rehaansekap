# Panduan Deployment MERN Stack di DigitalOcean

Panduan lengkap untuk mendeploy aplikasi MongoDB Express React Node (MERN) ke DigitalOcean Droplet dari GitHub.

---

## Daftar Isi
1. [Prasyarat](#prasyarat)
2. [Langkah 1: Membuat Droplet di DigitalOcean](#langkah-1-membuat-droplet-di-digitalocean)
3. [Langkah 2: Setup Awal Server](#langkah-2-setup-awal-server)
4. [Langkah 3: Install Dependencies](#langkah-3-install-dependencies)
5. [Langkah 4: Setup Database MongoDB](#langkah-4-setup-database-mongodb)
6. [Langkah 5: Clone Repository dari GitHub](#langkah-5-clone-repository-dari-github)
7. [Langkah 6: Konfigurasi Environment Variables](#langkah-6-konfigurasi-environment-variables)
8. [Langkah 7: Setup Backend Express](#langkah-7-setup-backend-express)
9. [Langkah 8: Setup Frontend React](#langkah-8-setup-frontend-react)
10. [Langkah 9: Jalankan Aplikasi Tanpa Domain](#langkah-9-jalankan-aplikasi-tanpa-domain)
11. [Langkah 10: Setup Reverse Proxy dengan Nginx](#langkah-10-setup-reverse-proxy-dengan-nginx)
12. [(Opsional) Langkah 11: Setup Domain dan SSL](#opsional-langkah-11-setup-domain-dan-ssl)

---

## Prasyarat

Sebelum memulai, pastikan Anda memiliki:
- Akun GitHub dengan repository MERN yang sudah siap
- Akun DigitalOcean (gratis $200 credit untuk 60 hari pertama)
- SSH client di komputer lokal (bawaan pada Linux/Mac, PuTTY atau Git Bash untuk Windows)
- Git terinstall di komputer lokal
- Basic knowledge tentang Linux command line

---

## Langkah 1: Membuat Droplet di DigitalOcean

### 1.1 Login ke DigitalOcean
1. Buka [digitalocean.com](https://www.digitalocean.com)
2. Login atau buat akun baru
3. Pilih menu "Droplets" di sidebar kiri

### 1.2 Create New Droplet
1. Klik tombol **"Create"** → **"Droplets"**
2. Pilih Region terdekat dengan target pengguna (misal: Singapore, Jakarta, atau Asia lainnya)

### 1.3 Pilih OS dan Ukuran
- **Choose an image**: Pilih **Ubuntu 24.04 x64** (LTS terbaru dan stabil)
- **Droplet type**: Pilih **Regular Intel Processor** (standar)
- **Droplet size**: Pilih **$6/bulan** (2 GB RAM, 1 CPU) minimal untuk MERN stack
  - Jika traffic besar, naik ke **$12/bulan** (4 GB RAM, 2 CPU)

### 1.4 Konfigurasi VPC dan Authentication
- **VPC Network**: Biarkan default
- **Authentication**: Pilih **SSH keys** untuk keamanan lebih baik
  - Jika belum punya SSH key, click **"New SSH Key"**
  - Buka terminal di komputer lokal jalankan:
    ```bash
    ssh-keygen -t rsa -b 4096 -f ~/.ssh/digitalocean -C "your-email@example.com"
    ```
  - Copy isi file `~/.ssh/digitalocean.pub` ke DigitalOcean
  - Beri nama key (misal: "DliLearn Server")
  - Klik **"Add SSH Key"**

### 1.5 Finalisasi
- **Hostname**: Beri nama (misal: "DliLearn-prod")
- **Monitoring**: Pilih **"Enable Monitoring"** untuk monitoring sistem
- Klik **"Create Droplet"**

**Tunggu 1-2 menit hingga droplet ready**

### 1.6 Catat IP Address
Setelah droplet selesai dibuat, catat **IP Address** yang ditampilkan (misal: `123.45.67.89`). Ini yang akan digunakan untuk SSH.

---

## Langkah 2: Setup Awal Server

### 2.1 Connect via SSH
Dari terminal komputer lokal, jalankan:
```bash
ssh -i ~/.ssh/digitalocean root@<IP_ADDRESS>
```

Contoh:
```bash
ssh -i ~/.ssh/digitalocean root@123.45.67.89
```

Ketik "yes" jika diminta konfirmasi.

### 2.2 Update System Packages
```bash
apt update && apt upgrade -y
```

### 2.3 Setup Firewall (UFW)
```bash
ufw enable
ufw allow 22/tcp    # SSH
ufw allow 80/tcp    # HTTP
ufw allow 443/tcp   # HTTPS
ufw status
```

### 2.4 Buat Non-root User (Opsional tapi Recommended)
```bash
adduser deploy
usermod -aG sudo deploy
su - deploy
```

Selanjutnya gunakan user `deploy` untuk operasi (ganti `root` menjadi `deploy`).

---

## Langkah 3: Install Dependencies

### 3.1 Update apt repos
```bash
sudo apt update
```

### 3.2 Install Node.js dan npm
Gunakan NodeSource repository untuk versi terbaru:
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

Verifikasi instalasi:
```bash
node --version
npm --version
```

### 3.3 Install Git
```bash
sudo apt install -y git
```

### 3.4 Install Nginx (Web Server/Reverse Proxy)
```bash
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### 3.5 Install MongoDB Server
```bash
sudo apt-get install -y mongodb-org
```

Jika tidak tersedia di repository default, gunakan MongoDB official repository:
```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt-get update
sudo apt-get install -y mongodb-org
```

Mulai MongoDB:
```bash
sudo systemctl start mongod
sudo systemctl enable mongod
```

Verifikasi MongoDB running:
```bash
sudo systemctl status mongod
```

### 3.6 Install PM2 (Process Manager untuk Node.js)
```bash
sudo npm install -g pm2
```

---

## Langkah 4: Setup Database MongoDB

### 4.1 Akses MongoDB Shell
```bash
mongosh
```

### 4.2 Buat Database
```javascript
// Di dalam mongosh
use dli_learn
db.createCollection("users")
show databases
```

### 4.3 Exit MongoDB
```javascript
exit
```

**Catatan**: Jika menggunakan MongoDB Atlas (cloud), skip langkah ini dan gunakan connection string dari Atlas nanti di environment variables.

---

## Langkah 5: Clone Repository dari GitHub

### 5.1 Setup SSH Key untuk GitHub (Recommended)

Buat SSH key untuk GitHub:
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/github -C "your-email@example.com"
cat ~/.ssh/github.pub
```

Salin output dan tambahkan ke GitHub:
1. Login GitHub → Settings → SSH and GPG keys
2. Click "New SSH key"
3. Paste dan save

### 5.2 Clone Repository
```bash
cd ~
git clone git@github.com:YOUR_USERNAME/DliLearn.git
cd DliLearn
```

Ganti `YOUR_USERNAME` dengan username GitHub Anda.

### 5.3 Verifikasi Struktur
```bash
ls -la
```

Pastikan ada folder `app/`, `resources/`, `server/`, atau struktur MERN yang sesuai.

---

## Langkah 6: Konfigurasi Environment Variables

### 6.1 Backend - Buat `.env` di folder backend

```bash
# Masuk ke folder backend
cd ~/DliLearn/server  # atau sesuai struktur project Anda

# Buat file .env
nano .env
```

### 6.2 Isi Environment Variables Backend

Sesuaikan dengan konfigurasi Anda:

```env
# Backend Configuration
PORT=5000
NODE_ENV=production

# Database MongoDB
MONGODB_URI=mongodb://localhost:27017/dli_learn
# Atau jika menggunakan MongoDB Atlas:
# MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/dli_learn?retryWrites=true&w=majority

# JWT Secret (generate dengan: openssl rand -base64 32)
JWT_SECRET=your-very-secret-key-here-change-this

# Frontend URL untuk CORS
FRONTEND_URL=http://YOUR_IP_ADDRESS:3000
# Nanti ganti dengan domain jika sudah ada

# Email Configuration (jika perlu)
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USER=your-email@gmail.com
MAIL_PASSWORD=your-app-password

# Lainnya sesuai kebutuhan aplikasi Anda
```

Tekan `Ctrl + X`, kemudian `Y`, `Enter` untuk save.

### 6.3 Frontend - Buat `.env` atau `.env.production` di folder frontend

```bash
cd ~/DliLearn/client  # atau sesuai struktur project Anda
nano .env.production
```

```env
# Frontend Configuration
VITE_API_URL=http://YOUR_IP_ADDRESS:5000
# Atau jika build production: http://localhost:5000 (akan di-proxy oleh Nginx)
```

---

## Langkah 7: Setup Backend Express

### 7.1 Install Dependencies Backend
```bash
cd ~/DliLearn/server
npm install
```

### 7.2 Build/Compile Backend (jika ada)
Jika ada build step:
```bash
npm run build
# atau
npm run compile
```

### 7.3 Test Backend
```bash
npm start
```

Jika berjalan tanpa error, tekan `Ctrl + C` untuk stop.

### 7.4 Setup PM2 untuk Backend
```bash
pm2 start npm --name "dli-backend" -- start
pm2 save
pm2 startup
```

Verifikasi:
```bash
pm2 list
```

---

## Langkah 8: Setup Frontend React

### 8.1 Install Dependencies Frontend
```bash
cd ~/DliLearn/client
npm install
```

### 8.2 Build Frontend untuk Production
```bash
npm run build
```

Ini akan membuat folder `dist/` dengan file-file optimized untuk production.

### 8.3 Verifikasi Build
```bash
ls -la dist/
```

Pastikan folder `dist` berisi `index.html` dan assets lainnya.

---

## Langkah 9: Jalankan Aplikasi Tanpa Domain

### 9.1 Konfigurasi Nginx sebagai Reverse Proxy

Edit konfigurasi Nginx default:
```bash
sudo nano /etc/nginx/sites-available/default
```

Ganti seluruh isi dengan:

```nginx
# Upstream backend Node.js
upstream backend {
    server 127.0.0.1:5000;
}

# Server block untuk API
server {
    listen 80;
    listen [::]:80;
    
    server_name _;
    
    # API endpoints
    location /api/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Frontend React
    location / {
        root /home/deploy/DliLearn/client/dist;  # Sesuaikan path
        try_files $uri $uri/ /index.html;
        
        # Cache busting untuk production
        add_header Cache-Control "public, max-age=3600";
    }
    
    # Static assets
    location /assets/ {
        root /home/deploy/DliLearn/client/dist;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

**Penting**: Ganti `/home/deploy/DliLearn/client/dist` dengan path yang sesuai di server Anda.

### 9.2 Test Konfigurasi Nginx
```bash
sudo nginx -t
```

Output harus: `nginx: configuration file test is successful`

### 9.3 Reload Nginx
```bash
sudo systemctl reload nginx
```

### 9.4 Verifikasi Backend PM2 Running
```bash
pm2 list
```

Pastikan `dli-backend` status **online**.

### 9.5 Test Aplikasi via IP Address

Buka browser dan akses:
```
http://YOUR_IP_ADDRESS
```

Contoh: `http://123.45.67.89`

**Jika berhasil**, Anda akan melihat:
- Frontend React aplikasi
- API endpoint berjalan di `http://YOUR_IP_ADDRESS/api/`

---

## Langkah 10: Setup Reverse Proxy dengan Nginx

*Langkah ini sudah dilakukan di Langkah 9 untuk konfigurasi dasar.*

### 10.1 Setup SSL dengan Let's Encrypt (Opsional saat ini)

Untuk production dengan domain, Anda perlu SSL. Skip untuk sekarang jika belum punya domain.

### 10.2 Enable GZIP Compression

Edit file nginx config:
```bash
sudo nano /etc/nginx/nginx.conf
```

Pastikan di dalam `http {}` block ada:
```nginx
gzip on;
gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss;
gzip_min_length 1000;
```

Save dan reload:
```bash
sudo systemctl reload nginx
```

### 10.3 Setup Monitoring & Logging

Cek error log:
```bash
sudo tail -50 /var/log/nginx/error.log
```

Cek access log:
```bash
sudo tail -50 /var/log/nginx/access.log
```

---

## Langkah 11: Update Backend URL di Frontend Production Build

Jika API endpoint berbeda saat production, rebuild frontend:

### 11.1 Update Environment Variable

```bash
cd ~/DliLearn/client
nano .env.production
```

Pastikan `VITE_API_URL` sesuai:
```env
VITE_API_URL=http://localhost:5000
# atau IP yang sama karena sama-sama di server
```

### 11.2 Rebuild Frontend
```bash
npm run build
```

### 11.3 Restart Nginx
```bash
sudo systemctl restart nginx
```

---

## Troubleshooting

### Backend tidak berjalan
```bash
pm2 logs dli-backend
```

### Frontend tidak muncul
Cek apakah folder dist ada:
```bash
ls -la ~/DliLearn/client/dist/
```

Rebuild jika kosong.

### Nginx error
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### MongoDB connection error
```bash
sudo systemctl status mongod
# Atau untuk MongoDB Atlas, verifikasi connection string di .env
```

### Port sudah terpakai
```bash
sudo lsof -i :5000  # Cek port 5000
sudo lsof -i :80   # Cek port 80
```

---

## Checklist Deployment

- [ ] Droplet DigitalOcean dibuat
- [ ] SSH connection berhasil
- [ ] System updated
- [ ] Node.js & npm terinstall
- [ ] MongoDB terinstall & running
- [ ] Git setup selesai
- [ ] Repository di-clone
- [ ] Environment variables dikonfigurasi
- [ ] Backend dependencies installed
- [ ] Frontend build berhasil
- [ ] PM2 backend running
- [ ] Nginx configured
- [ ] Aplikasi accessible via IP address

---

---

# (OPSIONAL) LANGKAH 11: SETUP DOMAIN DAN SSL

*Ikuti langkah ini hanya jika Anda sudah membeli domain dan ingin menambahkannya.*

---

## Persiapan Domain

### 11.1 Beli Domain
- Domain provider populer: Namecheap, GoDaddy, CloudFlare, domain.id, niaga.hosting, etc.
- Harga berkisar dari Rp50,000 - Rp200,000 per tahun

### 11.2 Setup DNS Pointing ke DigitalOcean

Di control panel domain provider Anda:

1. Cari menu **DNS Records** atau **Name Servers**
2. Tambah A record baru:
   - **Name/Host**: `@` atau nama domain
   - **Type**: A
   - **Value/IP**: Masukkan IP Droplet Anda (misal: `123.45.67.89`)
   - **TTL**: 3600 (atau default)
   - Save

3. (Opsional) Tambah CNAME untuk www:
   - **Name**: `www`
   - **Type**: CNAME
   - **Value**: `your-domain.com`
   - Save

**Tunggu 5-30 menit untuk DNS propagation** (bisa dicek di [whatsmydns.net](https://www.whatsmydns.net/))

---

## Setup SSL dengan Let's Encrypt & Certbot

### 11.3 Install Certbot
```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

### 11.4 Generate SSL Certificate
```bash
sudo certbot certonly --nginx -d your-domain.com -d www.your-domain.com
```

Ganti `your-domain.com` dengan domain Anda.

Masukkan email untuk renewal notifications dan agree terms.

### 11.5 Verifikasi Certificate
```bash
sudo certbot certificates
```

Catat path certificate (/etc/letsencrypt/live/your-domain.com/)

---

## Update Nginx Configuration dengan SSL

### 11.6 Edit Nginx Config dengan SSL
```bash
sudo nano /etc/nginx/sites-available/default
```

Ganti dengan konfigurasi SSL:

```nginx
# Upstream backend
upstream backend {
    server 127.0.0.1:5000;
}

# HTTP redirect ke HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com www.your-domain.com;
    
    return 301 https://$server_name$request_uri;
}

# HTTPS dengan SSL
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    
    server_name your-domain.com www.your-domain.com;
    
    # SSL Certificates
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    # SSL Security
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # API endpoints
    location /api/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Frontend React
    location / {
        root /home/deploy/DliLearn/client/dist;
        try_files $uri $uri/ /index.html;
        
        add_header Cache-Control "public, max-age=3600";
    }
    
    # Static assets
    location /assets/ {
        root /home/deploy/DliLearn/client/dist;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}

# Redirect www ke non-www (opsional)
server {
    listen 443 ssl;
    server_name www.your-domain.com;
    
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    return 301 https://your-domain.com$request_uri;
}
```

**Penting**: Ganti semua `your-domain.com` dengan domain Anda.

### 11.7 Test dan Reload Nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## Update Environment Variables untuk Domain

### 11.8 Update Backend .env
```bash
nano ~/DliLearn/server/.env
```

Update FRONTEND_URL:
```env
FRONTEND_URL=https://your-domain.com
```

Restart backend:
```bash
pm2 restart dli-backend
```

### 11.9 Update Frontend .env.production (jika perlu)
```bash
cd ~/DliLearn/client
nano .env.production
```

```env
VITE_API_URL=https://your-domain.com/api
```

Rebuild frontend:
```bash
npm run build
sudo systemctl reload nginx
```

---

## Setup Auto Renewal SSL Certificate

### 11.10 Test Auto Renewal
```bash
sudo certbot renew --dry-run
```

Jika sukses, setup cron job (biasanya sudah otomatis):
```bash
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

---

## Custom Domain Testing

### 11.11 Test HTTPS Connection
```bash
curl -I https://your-domain.com
```

Harus return status `200` dan menunjukkan SSL certificate info.

### 11.12 Test dari Browser
```
https://your-domain.com
```

Pastikan:
- ✅ Green lock icon di browser
- ✅ Frontend React muncul
- ✅ API berjalan normal
- ✅ Tidak ada mixed content warning

---

## Monitoring Ongoing

### 11.13 Monitor SSL Certificate Expiry
```bash
sudo certbot certificates
```

Certificate Let's Encrypt valid 90 hari, auto-renewal 30 hari sebelum expired.

### 11.14 Monitor Server Health
```bash
# Cek CPU dan Memory
free -h
df -h

# Cek logs
pm2 logs
sudo tail /var/log/nginx/error.log
```

### 11.15 Setup Automated Backups (Opsional)
Di DigitalOcean dashboard:
1. Pilih Droplet
2. Klik "Backups"
3. Enable backups otomatis

---

## Checklist Domain & SSL

- [ ] Domain dibeli dan DNS dikonfigurasi
- [ ] DNS record pointing ke IP Droplet
- [ ] DNS propagation selesai (~30 menit)
- [ ] SSL certificate generated dengan Certbot
- [ ] Nginx configuration updated untuk SSL
- [ ] HTTPS redirect configured
- [ ] Certificate auto-renewal enabled
- [ ] Backend environment updated
- [ ] Frontend rebuilt
- [ ] Domain accessible via HTTPS
- [ ] SSL certificate valid di browser

---

## Kesimpulan

Aplikasi MERN Anda sekarang:
- ✅ Berjalan di DigitalOcean Droplet
- ✅ Accessible via IP address
- ✅ (Opsional) Secured dengan domain dan SSL

### Tips Maintenance:
1. **Update dependencies**: `npm audit fix`
2. **Monitor logs**: `pm2 logs` dan `sudo tail /var/log/nginx/error.log`
3. **Backup database**: Regular export MongoDB
4. **SSL renewal**: Automatic dengan Certbot
5. **Scaling**: Upgrade droplet jika needed

### Kontak Support:
- DigitalOcean: https://www.digitalocean.com/support
- Let's Encrypt: https://letsencrypt.org/support/

---

**Happy Deploying! 🚀**
