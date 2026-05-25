# 🖥️ Homelab Infrastructure — Master Guide
**PC Bekas → Profile Site Live di Internet**
> Fresh graduate TI UGM | DevOps/Cloud Portfolio | Ubuntu 24.04 LTS

---

## 📋 Daftar Isi
1. [Persiapan Hardware & Bahan](#1-persiapan-hardware--bahan)
2. [Install Ubuntu Server 24.04](#2-install-ubuntu-server-2404)
3. [Konfigurasi Awal Server](#3-konfigurasi-awal-server)
4. [Setup Storage (HDD 1TB)](#4-setup-storage-hdd-1tb)
5. [Install Docker & Docker Compose](#5-install-docker--docker-compose)
6. [Setup DuckDNS (Domain Gratis)](#6-setup-duckdns-domain-gratis)
7. [Buat & Deploy Profile Site](#7-buat--deploy-profile-site)
8. [Setup Nginx Reverse Proxy + SSL](#8-setup-nginx-reverse-proxy--ssl)
9. [Verifikasi — Site Live!](#9-verifikasi--site-live)
10. [Struktur Repository GitHub](#10-struktur-repository-github)
11. [Next Steps: Phase 2-4](#11-next-steps-phase-2-4)

---

## 1. Persiapan Hardware & Bahan

### Spek PC Server
| Komponen | Spesifikasi |
|----------|------------|
| RAM | 8GB (2 × 4GB) |
| SSD | 256GB — OS + Docker |
| HDD | 1TB — Data, logs, backup |
| OS Target | Ubuntu Server 24.04 LTS |

### Yang Dibutuhkan Sebelum Mulai
- [ ] USB flashdisk minimal 4GB
- [ ] File ISO Ubuntu Server 24.04 LTS → https://ubuntu.com/download/server
- [ ] Software Rufus (Windows) atau Balena Etcher (Mac/Linux) untuk burn ISO ke USB
- [ ] Monitor + keyboard untuk setup awal (setelah itu bisa remote SSH)
- [ ] Koneksi internet (LAN kabel **sangat direkomendasikan** — lebih stabil dari WiFi)
- [ ] Akun GitHub (gratis)
- [ ] Akun DuckDNS (gratis) → https://www.duckdns.org

> **Catatan koneksi:** Jika hanya ada WiFi, tetap bisa jalan tapi perlu konfigurasi
> tambahan di Netplan. LAN kabel jauh lebih mudah dan stabil untuk server.

---

## 2. Install Ubuntu Server 24.04

### Langkah 2.1 — Burn ISO ke USB
```bash
# Pakai Rufus (Windows) atau Balena Etcher:
# 1. Pilih file ISO Ubuntu Server 24.04
# 2. Pilih USB target
# 3. Klik Flash/Start
```

### Langkah 2.2 — Boot dari USB
1. Colok USB ke PC bekas
2. Nyalakan PC, masuk BIOS (biasanya tekan `F2`, `F12`, `Del`, atau `Esc`)
3. Ubah boot order: **USB first**
4. Save & Exit → PC akan boot dari USB

### Langkah 2.3 — Proses Install Ubuntu Server

Ikuti wizard instalasi:

```
Language: English (lebih mudah untuk dokumentasi)
Keyboard: pilih sesuai keyboard kalian
Network: pilih interface (eth0 untuk LAN, wlan0 untuk WiFi)
  → DHCP dulu tidak apa-apa, nanti kita set static IP
Storage: Custom storage layout
  → SSD 256GB:
      /          → 200GB (ext4)
      /boot      → 1GB (ext4)
      swap       → 8GB (sama dengan RAM)
      sisanya    → /var/lib/docker (ext4)
  → HDD 1TB: biarkan dulu, kita mount manual nanti
Profile setup:
  Your name: nama_kamu
  Server name: homelab (atau apapun)
  Username: devops (atau nama_kamu)
  Password: [buat password kuat]
OpenSSH: ✅ Install OpenSSH server (WAJIB centang)
Featured snaps: skip semua
```

5. Tunggu install selesai (~5-10 menit)
6. Pilih **Reboot Now**, cabut USB saat diminta

### Langkah 2.4 — Login Pertama
```bash
# Login langsung di PC server:
username: devops
password: [password yang dibuat tadi]

# Cek IP address server:
ip addr show
# Catat IP yang muncul di eth0, contoh: 192.168.1.100
```

---

## 3. Konfigurasi Awal Server

### Langkah 3.1 — SSH dari Laptop/PC Utama
```bash
# Dari laptop/PC kamu:
ssh devops@192.168.1.100
# Setelah ini semua perintah dijalankan via SSH
```

### Langkah 3.2 — Update System
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim htop net-tools ufw
```

### Langkah 3.3 — Set Static IP via Netplan

> Ini penting agar IP server tidak berubah setelah restart router.

```bash
# Cek nama interface jaringan
ip link show
# Biasanya: eth0, enp3s0, atau ens3 (tergantung hardware)

# Edit konfigurasi Netplan
sudo nano /etc/netplan/00-installer-config.yaml
```

Isi file (sesuaikan nama interface dan IP):
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:              # ganti dengan nama interface kamu
      dhcp4: no
      addresses:
        - 192.168.1.100/24   # ganti dengan IP yang kamu inginkan
      routes:
        - to: default
          via: 192.168.1.1   # IP router/gateway kamu
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
# Terapkan konfigurasi
sudo netplan apply

# Verifikasi
ip addr show
ping google.com
```

### Langkah 3.4 — Konfigurasi UFW Firewall
```bash
# Aktifkan UFW
sudo ufw enable

# Izinkan port yang diperlukan
sudo ufw allow ssh          # port 22 - SSH
sudo ufw allow 80/tcp       # HTTP
sudo ufw allow 443/tcp      # HTTPS

# Cek status
sudo ufw status verbose
```

### Langkah 3.5 — Hardening SSH (Opsional tapi Direkomendasikan)
```bash
sudo nano /etc/ssh/sshd_config
```

Ubah/tambahkan baris ini:
```
PermitRootLogin no
PasswordAuthentication yes    # bisa diubah ke no jika sudah setup SSH key
MaxAuthTries 3
```

```bash
sudo systemctl restart ssh
```

---

## 4. Setup Storage (HDD 1TB)

### Langkah 4.1 — Identifikasi HDD
```bash
# Lihat disk yang tersedia
lsblk
# HDD 1TB biasanya: /dev/sdb atau /dev/sda (tergantung setup)

sudo fdisk -l | grep "1 TiB"
# Catat nama device, contoh: /dev/sdb
```

### Langkah 4.2 — Format & Partisi HDD
```bash
# Format HDD (HATI-HATI: ini menghapus semua data di /dev/sdb)
sudo mkfs.ext4 /dev/sdb
```

### Langkah 4.3 — Mount ke /data
```bash
# Buat direktori mount point
sudo mkdir -p /data

# Mount manual (temporary, untuk test)
sudo mount /dev/sdb /data

# Buat struktur folder
sudo mkdir -p /data/{projects,db,logs,backup}

# Set permission
sudo chown -R devops:devops /data
```

### Langkah 4.4 — Auto-mount saat Boot
```bash
# Dapatkan UUID disk
sudo blkid /dev/sdb
# Output contoh: UUID="abcd-1234-efgh-5678"

# Edit fstab
sudo nano /etc/fstab
```

Tambahkan baris ini di bagian bawah:
```
UUID=abcd-1234-efgh-5678  /data  ext4  defaults  0  2
```

```bash
# Test fstab tanpa reboot
sudo umount /data
sudo mount -a

# Verifikasi
df -h /data
```

---

## 5. Install Docker & Docker Compose

### Langkah 5.1 — Install Docker Engine
```bash
# Hapus versi lama jika ada
sudo apt remove -y docker docker-engine docker.io containerd runc

# Install dependencies
sudo apt install -y ca-certificates curl gnupg lsb-release

# Tambah Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Tambah Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Langkah 5.2 — Konfigurasi Post-Install
```bash
# Tambah user ke group docker (tidak perlu sudo tiap perintah docker)
sudo usermod -aG docker devops

# Aktifkan Docker start saat boot
sudo systemctl enable docker
sudo systemctl start docker

# PENTING: Logout dan login ulang agar group docker berlaku
exit
ssh devops@192.168.1.100
```

### Langkah 5.3 — Verifikasi Docker
```bash
docker --version
docker compose version
docker run hello-world
```

### Langkah 5.4 — Pindahkan Docker Data ke SSD (Opsional)
Jika ingin Docker images tersimpan di partisi terpisah di SSD:
```bash
sudo systemctl stop docker

# Edit daemon config
sudo nano /etc/docker/daemon.json
```

```json
{
  "data-root": "/var/lib/docker"
}
```

```bash
sudo systemctl start docker
```

---

## 6. Setup DuckDNS (Domain Gratis)

DuckDNS memberikan subdomain gratis yang otomatis update ke IP publik kamu.

### Langkah 6.1 — Daftar DuckDNS
1. Buka https://www.duckdns.org
2. Login dengan akun Google/GitHub
3. Buat subdomain: misalnya `namadevops` → hasilnya `namadevops.duckdns.org`
4. Catat **token** yang diberikan DuckDNS

### Langkah 6.2 — Port Forwarding di Router
> Ini diperlukan agar traffic dari internet bisa masuk ke server kamu.

1. Login ke admin router (biasanya http://192.168.1.1)
2. Cari menu **Port Forwarding** atau **Virtual Server**
3. Tambahkan rules:
   - Port 80 → 192.168.1.100 (IP server)
   - Port 443 → 192.168.1.100 (IP server)

### Langkah 6.3 — Script Auto-Update IP
```bash
# Buat direktori untuk script DuckDNS
mkdir -p /data/projects/duckdns
cd /data/projects/duckdns

# Buat script update
nano duck.sh
```

Isi file `duck.sh`:
```bash
#!/bin/bash
# DuckDNS Auto-Update Script
DOMAIN="namadevops"          # ganti dengan subdomain kamu
TOKEN="your-duckdns-token"   # ganti dengan token dari DuckDNS

echo "$(date): Updating DuckDNS..." >> /data/logs/duckdns.log

RESULT=$(curl -s "https://www.duckdns.org/update?domains=${DOMAIN}&token=${TOKEN}&ip=")

echo "$(date): Result: ${RESULT}" >> /data/logs/duckdns.log
```

```bash
# Set permission executable
chmod +x duck.sh

# Test manual
./duck.sh
cat /data/logs/duckdns.log
# Harus ada output: OK
```

### Langkah 6.4 — Setup Cron Job (Auto-update setiap 5 menit)
```bash
crontab -e
```

Tambahkan baris:
```
*/5 * * * * /data/projects/duckdns/duck.sh >/dev/null 2>&1
```

---

## 7. Buat & Deploy Profile Site

### Langkah 7.1 — Buat Direktori Project
```bash
mkdir -p /data/projects/profile-site
cd /data/projects/profile-site
```

### Langkah 7.2 — Buat HTML Profile Site Sederhana
```bash
mkdir -p src
nano src/index.html
```

Isi `src/index.html`:
```html
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Nama Kamu — DevOps Engineer</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: 'Segoe UI', system-ui, sans-serif;
      background: #0f0f0f;
      color: #e0e0e0;
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .card {
      max-width: 600px;
      padding: 3rem;
      text-align: center;
    }
    .avatar {
      width: 100px;
      height: 100px;
      background: #1a73e8;
      border-radius: 50%;
      margin: 0 auto 1.5rem;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 2.5rem;
    }
    h1 { font-size: 2rem; margin-bottom: 0.5rem; }
    .title { color: #1a73e8; margin-bottom: 1.5rem; }
    .bio { color: #999; line-height: 1.7; margin-bottom: 2rem; }
    .tags { display: flex; flex-wrap: wrap; gap: 0.5rem; justify-content: center; margin-bottom: 2rem; }
    .tag {
      background: #1a1a1a;
      border: 1px solid #333;
      padding: 0.3rem 0.8rem;
      border-radius: 4px;
      font-size: 0.85rem;
    }
    .links { display: flex; gap: 1rem; justify-content: center; }
    .links a {
      color: #1a73e8;
      text-decoration: none;
      padding: 0.5rem 1.2rem;
      border: 1px solid #1a73e8;
      border-radius: 4px;
      transition: all 0.2s;
    }
    .links a:hover { background: #1a73e8; color: white; }
    .homelab-badge {
      margin-top: 3rem;
      font-size: 0.75rem;
      color: #555;
    }
  </style>
</head>
<body>
  <div class="card">
    <div class="avatar">👨‍💻</div>
    <h1>Nama Kamu</h1>
    <p class="title">Fresh Graduate TI UGM · DevOps Enthusiast</p>
    <p class="bio">
      Membangun infrastruktur cloud mandiri dari PC bekas.
      Tertarik pada DevOps, containerization, dan automation.
    </p>
    <div class="tags">
      <span class="tag">Docker</span>
      <span class="tag">Linux</span>
      <span class="tag">Nginx</span>
      <span class="tag">CI/CD</span>
      <span class="tag">Kubernetes</span>
      <span class="tag">Python</span>
    </div>
    <div class="links">
      <a href="https://github.com/username_kamu" target="_blank">GitHub</a>
      <a href="https://linkedin.com/in/username_kamu" target="_blank">LinkedIn</a>
    </div>
    <p class="homelab-badge">
      🖥️ Hosted on homelab · Self-built infrastructure
    </p>
  </div>
</body>
</html>
```

### Langkah 7.3 — Buat Dockerfile
```bash
nano Dockerfile
```

```dockerfile
FROM nginx:alpine

# Copy site ke folder nginx
COPY src/ /usr/share/nginx/html/

# Expose port 80
EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost/ || exit 1
```

### Langkah 7.4 — Buat docker-compose.yml
```bash
nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  profile-site:
    build: .
    container_name: profile-site
    restart: unless-stopped
    ports:
      - "8080:80"   # internal port, Nginx proxy akan forward ke sini
    networks:
      - webnet
    labels:
      - "traefik.enable=false"

networks:
  webnet:
    name: webnet
    driver: bridge
```

### Langkah 7.5 — Build & Jalankan Container
```bash
# Build image dan jalankan
docker compose up -d --build

# Cek status
docker ps
docker logs profile-site

# Test lokal
curl http://localhost:8080
```

---

## 8. Setup Nginx Reverse Proxy + SSL

### Langkah 8.1 — Buat Struktur Folder Nginx
```bash
mkdir -p /data/projects/nginx/{conf.d,ssl,logs}
cd /data/projects/nginx
```

### Langkah 8.2 — Buat docker-compose untuk Nginx + Certbot
```bash
nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./ssl:/etc/letsencrypt:ro
      - ./logs:/var/log/nginx
      - certbot-webroot:/var/www/certbot
    networks:
      - webnet
    depends_on:
      - certbot

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - ./ssl:/etc/letsencrypt
      - certbot-webroot:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  certbot-webroot:

networks:
  webnet:
    external: true
    name: webnet
```

### Langkah 8.3 — Konfigurasi Nginx (HTTP dulu, untuk verifikasi SSL)
```bash
nano conf.d/profile.conf
```

```nginx
server {
    listen 80;
    server_name namadevops.duckdns.org;   # ganti dengan domain kamu

    # Untuk Certbot challenge
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Redirect semua HTTP ke HTTPS (aktifkan setelah SSL berhasil)
    # return 301 https://$host$request_uri;

    # Sementara: langsung proxy ke container profile-site
    location / {
        proxy_pass http://profile-site:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
# Jalankan Nginx
cd /data/projects/nginx
docker compose up -d

# Test: buka http://namadevops.duckdns.org dari browser
```

### Langkah 8.4 — Request SSL Certificate (Let's Encrypt)
```bash
# Jalankan certbot untuk request sertifikat
docker run --rm \
  -v /data/projects/nginx/ssl:/etc/letsencrypt \
  -v certbot-webroot:/var/www/certbot \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email email@kamu.com \
  --agree-tos \
  --no-eff-email \
  -d namadevops.duckdns.org

# Verifikasi sertifikat berhasil dibuat
ls /data/projects/nginx/ssl/live/namadevops.duckdns.org/
```

### Langkah 8.5 — Update Nginx Config untuk HTTPS
```bash
nano /data/projects/nginx/conf.d/profile.conf
```

```nginx
# Redirect HTTP → HTTPS
server {
    listen 80;
    server_name namadevops.duckdns.org;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl;
    server_name namadevops.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/namadevops.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/namadevops.duckdns.org/privkey.pem;

    # SSL Security
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

    # Proxy ke profile-site
    location / {
        proxy_pass http://profile-site:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Logs
    access_log /var/log/nginx/profile-access.log;
    error_log /var/log/nginx/profile-error.log;
}
```

```bash
# Reload Nginx
docker compose exec nginx nginx -s reload
```

---

## 9. Verifikasi — Site Live!

### Checklist Akhir
```bash
# 1. Cek semua container berjalan
docker ps

# 2. Cek log Nginx
docker logs nginx-proxy

# 3. Test koneksi
curl -I https://namadevops.duckdns.org

# 4. Cek SSL certificate
openssl s_client -connect namadevops.duckdns.org:443 -brief
```

Jika semua berjalan:
- ✅ `http://namadevops.duckdns.org` → redirect ke HTTPS
- ✅ `https://namadevops.duckdns.org` → profile site tampil
- ✅ SSL certificate valid (gembok hijau di browser)

### Troubleshooting Umum

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|---------------------|--------|
| Site tidak bisa diakses | Port forwarding belum diset | Cek konfigurasi router |
| SSL gagal | Domain belum propagate | Tunggu 5-10 menit setelah update DNS |
| Container tidak mau start | Port conflict | `docker ps -a` dan `docker rm` container lama |
| `curl` berhasil tapi browser tidak | ISP block port 80/443 | Coba dari jaringan lain (hotspot HP) |

---

## 10. Struktur Repository GitHub

### Repo 1: `homelab-infrastructure`
```
homelab-infrastructure/
├── README.md                    ← Architecture diagram + status badge
├── docs/
│   ├── architecture.md
│   └── troubleshooting.md
├── phase1/
│   ├── README.md
│   ├── nginx/
│   │   ├── docker-compose.yml
│   │   └── conf.d/profile.conf
│   └── scripts/
│       └── duck.sh
├── phase2/                      ← CI/CD (nanti)
├── phase3/                      ← Monitoring (nanti)
└── phase4/                      ← Kubernetes (nanti)
```

### Repo 2: `profile-site`
```
profile-site/
├── README.md
├── Dockerfile
├── docker-compose.yml
├── src/
│   └── index.html
└── .github/
    └── workflows/
        └── deploy.yml           ← Phase 2: CI/CD pipeline
```

### Push ke GitHub
```bash
# Setup git di server
git config --global user.name "Nama Kamu"
git config --global user.email "email@kamu.com"

# Push profile-site
cd /data/projects/profile-site
git init
git remote add origin https://github.com/username/profile-site.git
git add .
git commit -m "feat: initial profile site with Docker setup"
git push -u origin main

# Push infrastructure
mkdir -p /data/projects/homelab-infrastructure
cd /data/projects/homelab-infrastructure
# ... copy file config ke sini
git init
git remote add origin https://github.com/username/homelab-infrastructure.git
git add .
git commit -m "feat: phase 1 foundation - nginx + ssl + duckdns"
git push -u origin main
```

---

## 11. Next Steps: Phase 2-4

### Phase 2 — CI/CD Pipeline
Setelah Phase 1 selesai, goal selanjutnya:
```
git push → GitHub Actions build Docker image → SSH ke server → Deploy otomatis
```
Tech: GitHub Actions, Docker Hub/GitHub Container Registry, SSH deploy

### Phase 3 — Observability
```
Prometheus (metrics) + Grafana (dashboard) + Node Exporter + cAdvisor + Alertmanager
```
Bisa monitor: CPU, RAM, disk, container health, traffic

### Phase 4 — Kubernetes (k3s)
Migrasi dari Docker Compose biasa ke Kubernetes ringan:
```
k3s + Traefik Ingress + HPA (auto-scaling) + Persistent Volume
```

---

## 📌 Quick Reference — Perintah Harian

```bash
# Cek semua container
docker ps

# Restart service tertentu
docker compose restart profile-site

# Lihat log realtime
docker logs -f nginx-proxy

# Update site (setelah edit file)
cd /data/projects/profile-site
docker compose up -d --build

# Cek disk usage
df -h
docker system df

# Cek RAM
free -h

# Cek CPU & proses
htop
```

---

*Dibuat untuk portofolio DevOps | UGM Teknik Informatika 2026*
*Stack: Ubuntu 24.04 · Docker · Nginx · Let's Encrypt · DuckDNS*
