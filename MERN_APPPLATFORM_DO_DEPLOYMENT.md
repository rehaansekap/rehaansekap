## Daftar Isi

1. [Prasyarat](#prasyarat)
2. [Langkah 1: Persiapan Repository di GitHub](#langkah-1-persiapan-repository-di-github)
3. [Langkah 2: Setup Environment Variables](#langkah-2-setup-environment-variables)
4. [Langkah 3: Konfigurasi App Platform](#langkah-3-konfigurasi-app-platform)
5. [Langkah 4: Setup Database MongoDB](#langkah-4-setup-database-mongodb)
6. [Langkah 5: Deploy Aplikasi](#langkah-5-deploy-aplikasi)
7. [Langkah 6: Testing Aplikasi](#langkah-6-testing-aplikasi)
8. [Langkah 7: Setup Custom Domain (Opsional)](#langkah-7-setup-custom-domain-opsional)
9. [Troubleshooting & Monitoring](#troubleshooting--monitoring)

---

## Langkah 1: Persiapan Repository di GitHub

App Platform membutuhkan struktur repository yang tepat untuk auto-detect build commands.

### 1.1 Cek Struktur Repository

Repository MERN Kamu harus memiliki struktur seperti ini:

```
DliLearn/
├── package.json           # Root level (opsional)
├── server/                # Backend Express
│   ├── package.json
│   ├── index.js
│   ├── .env
│   └── node_modules/
├── client/                # Frontend React
│   ├── package.json
│   ├── vite.config.ts     # Untuk Vite
│   ├── src/
│   └── dist/
├── .gitignore
├── README.md
└── app.yaml              # (KAMI AKAN BUAT INI)
```

### 1.2 Setup .gitignore

Pastikan `.node_modules`, `.env`, dan `dist` ada di `.gitignore`:

```bash
cd ~/DliLearn
cat .gitignore
```

Kalau belum ada, buat/update `.gitignore`:

```
node_modules/
.env
.env.local
.env.*.local
dist/
build/
*.log
.DS_Store
.idea/
.vscode/
```

Push ke GitHub:

```bash
git add .gitignore
git commit -m "Update .gitignore for deployment"
git push origin main
```

### 1.3 Buat app.yaml di Root Repository

File `app.yaml` adalah configuration file untuk App Platform. Buat file ini di root directory project:

```bash
cd ~/DliLearn
nano app.yaml
```

Isi dengan konfigurasi berikut:

```yaml
name: dli-learn
services:
    # Backend Express API
    - name: backend
      github:
          repo: YOUR_USERNAME/DliLearn
          branch: main
          deploy_on_push: true
      build_command: cd server && npm install && npm run build
      run_command: cd server && npm start
      envs:
          # Backend environment variables
          - key: NODE_ENV
            value: production
          - key: PORT
            value: '5000'
          # MongoDB - akan diisi nanti
          - key: MONGODB_URI
            scope: RUN_AND_BUILD_TIME
            value: ${db.DATABASE_URL}
          # JWT Secret - GANTI DENGAN NILAI RANDOM
          - key: JWT_SECRET
            scope: RUN_AND_BUILD_TIME
            value: your-secret-key-generate-with-openssl-rand
          # CORS
          - key: FRONTEND_URL
            scope: RUN_AND_BUILD_TIME
            value: ${frontend.PUBLIC_URL}
          # Tambah environment variables lainnya sesuai kebutuhan
      http_port: 5000
      resource_type: service
      # Opsional: ukuran resource
      # instance_count: 2
      # instance_size_slug: basic-m

    # Frontend React
    - name: frontend
      github:
          repo: YOUR_USERNAME/DliLearn
          branch: main
          deploy_on_push: true
      build_command: cd client && npm install && npm run build
      source_dir: client/dist
      http_port: 8080
      resource_type: static_site
      envs:
          - key: VITE_API_URL
            value: ${backend.PUBLIC_URL}/api

    # Untuk statik file serving Kalau perlu
    # - name: static-files
    #   source_dir: public
    #   resource_type: static_site
    #   http_port: 8080

# Database MongoDB
databases:
    - name: db
      engine: MONGODB
      version: '7'
      production: true
      # Opsional: konfigurasi sizing
      # db_instance_size_slug: db-s-1vcpu-1gb
# Domain custom (opsional, setup nanti)
# domains:
#   - domain: your-domain.com
```

**PENTING**: Ganti `YOUR_USERNAME` dengan GitHub username Kamu!

### 1.4 Tambahkan build scripts di package.json

Pastikan `server/package.json` dan `client/package.json` memiliki scripts yang tepat:

**server/package.json:**

```json
{
    "scripts": {
        "start": "node index.js",
        "build": "echo 'Building backend...'",
        "dev": "nodemon index.js"
    }
}
```

**client/package.json:**

```json
{
    "scripts": {
        "dev": "vite",
        "build": "vite build",
        "preview": "vite preview"
    }
}
```

### 1.5 Commit dan Push ke GitHub

```bash
cd ~/DliLearn
git add app.yaml
git add server/package.json client/package.json
git commit -m "Add App Platform configuration"
git push origin main
```

---

## Langkah 2: Setup Environment Variables

### 2.1 Kumpulkan Semua Environment Variables

Sebelum deploy, siapkan semua environment variables yang diperlukan:

**Backend (.env untuk development):**

```env
NODE_ENV=production
PORT=5000
JWT_SECRET=your-random-secret-key-here
FRONTEND_URL=https://your-app.ondigitalocean.app
# Atau custom domain: https://your-domain.com

# Database akan di-set otomatis dari App Platform
# MONGODB_URI akan di-inject sebagai ${db.DATABASE_URL}

# Email Configuration (Kalau ada)
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USER=your-email@gmail.com
MAIL_PASSWORD=your-app-password

# API Keys (Kalau ada)
STRIPE_API_KEY=sk_live_xxxxx
CLOUDINARY_URL=cloudinary://xxx
```

**Frontend (.env untuk development):**

```env
VITE_API_URL=http://localhost:5000/api
```

### 2.2 Generate JWT Secret

Di terminal lokal, generate secret key:

```bash
openssl rand -base64 32
```

Output akan seperti: `aBc123XyZ/+==`

Catat nilai ini untuk diPake di App Platform nanti.

### 2.3 Generate Database Password (untuk MongoDB)

Kalau setup MongoDB manual (bukan managed):

```bash
openssl rand -base64 16
```

---

## Langkah 3: Konfigurasi App Platform

### 3.1 Login ke DigitalOcean

1. Buka [digitalocean.com](https://www.digitalocean.com)
2. Login ke akun Kamu
3. Klik menu **Apps** di sidebar

### 3.2 Authorize GitHub Integration

Kalau belum pernah:

1. Klik **Create App**
2. Pilih **GitHub**
3. Klik **Authorize DigitalOcean**
4. Login GitHub dan approve access
5. Pilih repository **DliLearn** (atau repository Kamu)

### 3.3 Create New App

1. Klik **Create App** → **GitHub**
2. Pilih repository: **YOUR_USERNAME/DliLearn**
3. Branch: **main** (atau branch yang ingin di-deploy)
4. Klik **Next**

### 3.4 Review App Configuration

App Platform akan auto-detect konfigurasi dari `app.yaml`:

- ✅ Backend service (Node.js)
- ✅ Frontend service (Static site)
- ✅ Database MongoDB

Pastikan semua ter-detect dengan benar. Kalau ada error, edit manual:

- Klik service → Edit
- Adjust build command, run command, port, dll.

### 3.5 Configure Environment Variables

Pada tahap configuration, add semua environment variables:

**Backend service:**

1. Klik **backend** service
2. Klik tab **Environment**
3. Add variables:
    - `NODE_ENV` = `production`
    - `PORT` = `5000`
    - `JWT_SECRET` = `<paste-generated-secret>`
    - `FRONTEND_URL` = Akan diisi nanti setelah deploy
    - Tambah variable lainnya

**Frontend service:**

1. Klik **frontend** service
2. Klik tab **Environment**
3. Pastikan `VITE_API_URL` sudah di-set ke `${backend.PUBLIC_URL}/api`

### 3.6 Configure Database

Kalau Pake DigitalOcean Managed Database:

1. Klik **Database** atau tab **Resources**
2. Pilih **MongoDB** (managed service)
3. Setup:
    - **Size**: Minimal $15/bulan
    - **Version**: 7.0 (latest)
    - **Region**: Same region as app
4. Database akan otomatis di-provision

Atau Pake MongoDB Atlas (cloud):

- Sign up di [mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas)
- Create cluster gratis (M0)
- Copy connection string ke backend environment variable `MONGODB_URI`

---

## Langkah 4: Setup Database MongoDB

### 4.1 Opsi A: Pake Managed MongoDB di DigitalOcean

Kalau memilih managed MongoDB saat create app:

1. MongoDB akan auto-provisioned
2. Connection string akan otomatis di-inject ke variable `${db.DATABASE_URL}`
3. Tidak perlu setup manual

**Verify dalam app.yaml:**

```yaml
databases:
    - name: db
      engine: MONGODB
      version: '7'
      production: true
```

### 4.2 Opsi B: Pake MongoDB Atlas (Cloud)

Kalau ingin pakai cloud database:

1. Sign up di [mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas)
2. Create free cluster (M0)
3. Setup network access:
    - Klik **Network Access**
    - Add IP: `0.0.0.0/0` (allow all - untuk production lebih ketat)
4. Get connection string:
    - Klik **Connect**
    - Pilih **Connect your application**
    - Copy connection string
5. Di App Platform, set environment variable:
    - `MONGODB_URI` = `mongodb+srv://username:password@cluster.mongodb.net/dli_learn?retryWrites=true&w=majority`

### 4.3 Opsi C: MongoDB di Komputer Lokal (Testing)

Skip untuk production. Hanya untuk development.

---

## Langkah 5: Deploy Aplikasi

### 5.1 Review Final Configuration

Sebelum deploy, pastikan:

- ✅ `app.yaml` sudah di-commit dan push ke GitHub
- ✅ Environment variables sudah di-set
- ✅ Database sudah ter-configure
- ✅ Build commands benar
- ✅ Run commands benar

### 5.2 Deploy Aplikasi

Di DigitalOcean App Platform:

1. Klik **Create App**
2. Review semua konfigurasi
3. Klik **Create Resources**

**Tunggu 5-10 menit untuk deployment**

App Platform akan:

- Pull code dari GitHub
- Install dependencies
- Build frontend
- Build backend
- Start services
- Configure networking

### 5.3 Monitor Deployment Progress

1. Klik **Deployments** tab
2. Lihat progress:
    - Image building
    - Service starting
    - Health checks
3. Kalau ada error, lihat logs:
    - Klik **Logs** tab
    - Filter by service (backend/frontend)

### 5.4 Catat Public URLs

Setelah deployment selesai:

1. Klik **Settings** → **App Info**
2. Catat **Live App** URL (misal: `https://dli-learn-abc123.ondigitalocean.app`)

**URLs:**

- Frontend: `https://dli-learn-abc123.ondigitalocean.app`
- Backend: `https://dli-learn-abc123.ondigitalocean.app/api` (via proxy)
- Backend service URL: Lihat di service details

---

## Langkah 6: Testing Aplikasi

### 6.1 Test Frontend

1. Buka di browser: `https://dli-learn-abc123.ondigitalocean.app`
2. Pastikan:
    - ✅ Frontend muncul dengan benar
    - ✅ No error di console
    - ✅ CSS & assets loading properly
    - ✅ SSL/HTTPS working

### 6.2 Test Backend API

Pake curl atau Postman:

```bash
curl https://dli-learn-abc123.ondigitalocean.app/api/
# Or test specific endpoint
curl https://dli-learn-abc123.ondigitalocean.app/api/users
```

Expected response:

- ✅ Status 200-201 (atau error handling yang tepat)
- ✅ JSON response
- ✅ No timeout

### 6.3 Test Database Connection

1. Di backend, buat endpoint test:

    ```javascript
    app.get('/api/health', async (req, res) => {
        try {
            await mongoose.connection.db.admin().ping();
            res.json({ status: 'ok', database: 'connected' });
        } catch (error) {
            res.status(500).json({ status: 'error', message: error.message });
        }
    });
    ```

2. Test: `curl https://dli-learn-abc123.ondigitalocean.app/api/health`

### 6.4 Test Environment Variables

Verify di backend logs:

1. App Platform → **Logs**
2. Filter backend service
3. Pastikan environment variables ter-load:
    ```
    NODE_ENV: production
    MONGODB_URI: connected
    JWT_SECRET: loaded
    ```

---

## Langkah 7: Setup Custom Domain (Opsional)

_Skip Kalau hanya ingin Pake `.ondigitalocean.app` domain._

### 7.1 Beli Domain

- Domain provider: Namecheap, GoDaddy, domain.id, dll.
- Harga: Rp50,000 - Rp200,000/tahun

### 7.2 Setup DNS di App Platform

1. Di DigitalOcean, klik App → **Settings**
2. Klik **Domains**
3. Klik **Add Domain**
4. Masukkan domain Kamu: `your-domain.com`
5. Klik **Add**

App Platform akan generate **DNS records** yang perlu di-add di domain registrar.

### 7.3 Add DNS Records di Domain Registrar

Misal di Namecheap:

1. Login ke Namecheap
2. Klik domain → **Manage**
3. Klik **Advanced DNS**
4. Add CNAME record yang diberikan App Platform:
    - **Host**: @ atau your-domain.com
    - **Type**: CNAME
    - **Value**: DigitalOcean provided CNAME
    - **TTL**: 3600
    - Save

**Tunggu 5-30 menit untuk DNS propagation**

### 7.4 Verify Domain Connection

1. Di App Platform, klik domain
2. Status akan berubah ke **Active** saat DNS propagated
3. Buka di browser: `https://your-domain.com`

**SSL Certificate automatic** untuk custom domain di App Platform!

---

## Langkah 8: Update Frontend & Backend untuk Domain

### 8.1 Update Backend Environment Variable

Kalau Pake custom domain:

1. App Platform → backend service → **Environment**
2. Update `FRONTEND_URL`:
    - Dari: `https://dli-learn-abc123.ondigitalocean.app`
    - Ke: `https://your-domain.com`
3. Save - automatic redeploy

### 8.2 Update Frontend API URL

1. Edit `client/.env.production`:

    ```env
    VITE_API_URL=https://your-domain.com/api
    ```

2. Commit & push:

    ```bash
    git add client/.env.production
    git commit -m "Update API URL for custom domain"
    git push origin main
    ```

3. App Platform akan otomatis redeploy

### 8.3 Configure CORS di Backend

Pastikan backend memperbolehkan domain:

```javascript
const cors = require('cors');

app.use(
    cors({
        origin: [
            'https://your-domain.com',
            'https://www.your-domain.com',
            'https://dli-learn-abc123.ondigitalocean.app',
        ],
        credentials: true,
    }),
);
```

Commit & push, automatic redeploy.

---

## Langkah 9: Automatic Redeployment via Git

### 9.1 Cara Kerja Automatic Deployment

Setiap kali push ke GitHub (`main` branch):

1. GitHub trigger webhook ke App Platform
2. App Platform:
    - Pull latest code
    - Run build command
    - Run health checks
    - Deploy (zero-downtime)
3. Aplikasi updated otomatis

### 9.2 Testing Automatic Deployment

1. Edit file di project (misal `client/src/App.jsx`)
2. Commit & push:
    ```bash
    git add .
    git commit -m "Update UI for deployment test"
    git push origin main
    ```
3. Di App Platform, lihat **Deployments** tab
4. New deployment should trigger otomatis
5. Tunggu selesai (~5-10 menit)
6. Refresh browser, changes harus muncul

---

## Troubleshooting & Monitoring

### 10.1 Check Logs

1. App Platform → **Logs**
2. Filter by service (backend/frontend)
3. Real-time monitoring

**Common error messages:**

- `Connection refused` → Database belum ready
- `Port already in use` → Conflict dengan port
- `Build failed` → Check build commands
- `CORS error` → Update allowed origins

### 10.2 Deployment Failed

Kalau deployment gagal:

1. Klik **Deployments**
2. Klik deployment yang failed
3. Baca error message detail
4. Common issues:
    - Missing environment variable
    - Build command error
    - Dependency not installed
    - Port conflict

**Fix:**

```bash
cd ~/DliLearn
# Test build locally
cd server && npm install && npm run build
cd ../client && npm install && npm run build
# Commit dan push
git add .
git commit -m "Fix build issues"
git push origin main
```

### 10.3 Database Connection Issues

1. Verify connection string:

    ```bash
    # In backend
    console.log(process.env.MONGODB_URI);
    ```

2. Check database status:
    - App Platform → **Resources** → **db**
    - Status should be "Running"

3. For MongoDB Atlas:
    - Verify IP whitelist
    - Check username/password
    - Test connection string locally

### 10.4 API CORS Errors

Kalau frontend dapat't call API:

1. Check browser console error
2. Usually: `Access-Control-Allow-Origin` missing
3. Fix backend CORS:

    ```javascript
    app.use(
        cors({
            origin: process.env.FRONTEND_URL,
            credentials: true,
        }),
    );
    ```

4. Commit, push, auto-redeploy

### 10.5 Slow Performance

1. Check logs untuk bottleneck
2. Monitor resource usage:
    - App Platform → **Insights** → **Metrics**
3. Kalau CPU/Memory tinggi:
    - Upgrade instance size di app.yaml
    - Atau setup multiple instances untuk load balancing

### 10.6 Health Checks

App Platform otomatis health check services:

1. Check status: **App** → service → **Health**
2. Kalau unhealthy:
    - Service might be crashing
    - Check logs
    - Restart service: **Settings** → **Restart App**

---

## Monitoring & Maintenance

### 11.1 Setup Monitoring

App Platform provides built-in monitoring:

1. App Platform → **Insights**
2. Lihat:
    - CPU usage
    - Memory usage
    - Request count
    - Error rate
    - Build duration

### 11.2 Setup Alerts (Opsional)

Untuk production, setup alerts:

1. **Insights** → **Alerts**
2. Create alert untuk:
    - High CPU/Memory
    - High error rate
    - Deployment failed
    - Service down

### 11.3 Backup Database

Kalau Pake managed MongoDB:

1. DigitalOcean → **Databases**
2. Select database
3. **Backups** → Enable automatic backups

Untuk MongoDB Atlas:

- Automatic backups included (free tier 7 days)
- Manual backup via Atlas interface

### 11.4 Update Dependencies

Regular update packages untuk security:

```bash
cd server
npm audit fix
git add package-lock.json
git commit -m "Update dependencies (security)"
git push origin main

cd ../client
npm audit fix
git add package-lock.json
git commit -m "Update dependencies (security)"
git push origin main
```

---

## Scaling & Advanced Configuration

### 12.1 Add More Instances

Untuk high traffic:

Edit `app.yaml`:

```yaml
services:
    - name: backend
      instance_count: 3 # 3 instances load balanced
      instance_size_slug: basic-m # More powerful
```

Commit & push untuk auto-scale.

### 12.2 Setup Environment-specific Configs

Buat branch terpisah untuk staging:

```bash
git checkout -b staging
# Edit app.yaml untuk staging config
git push origin staging
```

Di App Platform, create app lain dari branch `staging`.

### 12.3 Setup CI/CD dengan GitHub Actions (Lanjutan)

Tambahan file `.github/workflows/deploy.yml`:

```yaml
name: Deploy to DigitalOcean

on:
    push:
        branches: [main]

jobs:
    test:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v3
            - uses: actions/setup-node@v3
              with:
                  node-version: 18

            - name: Test Backend
              run: cd server && npm install && npm run test

            - name: Test Frontend
              run: cd client && npm install && npm run test
```
