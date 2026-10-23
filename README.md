<div align="center">

# 🚀 CI/CD Pipeline dengan Jenkins & Docker

### Otomasi Build & Deploy dari `git push` sampai Aplikasi Live

![Jenkins](https://img.shields.io/badge/JENKINS-CI%2FCD-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/DOCKER-COMPOSE-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Ubuntu](https://img.shields.io/badge/UBUNTU-SERVER_24.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![GitHub](https://img.shields.io/badge/GITHUB-WEBHOOK-181717?style=for-the-badge&logo=github&logoColor=white)
![Nginx](https://img.shields.io/badge/NGINX-REVERSE_PROXY-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Go](https://img.shields.io/badge/GO-BACKEND-00ADD8?style=for-the-badge&logo=go&logoColor=white)
![React](https://img.shields.io/badge/REACT-FRONTEND-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![PostgreSQL](https://img.shields.io/badge/POSTGRESQL-DATABASE-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)

*Setiap `git push` otomatis memicu build, test, dan deploy tanpa campur tangan manual*

</div>

---

## 📌 Overview

Repositori ini mendokumentasikan implementasi **CI/CD pipeline** dari nol di atas **Ubuntu Server (VMware)** menggunakan **Jenkins**. Implementasi dilakukan bertahap, dari aplikasi paling sederhana sampai arsitektur multi-service:

| Tahap | Aplikasi | Metode Deploy | Tujuan Belajar |
|-------|----------|---------------|----------------|
| 1 | Node.js + Express | systemd service | Dasar pipeline, webhook, hak akses `sudo` |
| 2 | Go + PostgreSQL + React | Docker Compose | Multi-service, multi-stage build, health check |
| 3 | Website statis Danau Toba | Docker + Nginx | Reuse pola pipeline untuk proyek baru |

Alur kerjanya: developer melakukan `git push` dari Windows, GitHub mengirim **webhook** ke Jenkins, lalu Jenkins menjalankan pipeline yang didefinisikan di `Jenkinsfile` sampai aplikasi ter-deploy di server.

---

## 🏗️ Arsitektur Sistem

```
 [Developer - Windows]
         │  git push
         ▼
    [GitHub Repo]
         │  webhook (POST /github-webhook/)
         ▼
    [ngrok Tunnel]  ← mengekspos Jenkins lokal ke internet
         │
         ▼
 ┌────────────────────────────────────────────┐
 │         Ubuntu Server (VMware, NAT)        │
 │                                            │
 │   [Jenkins :8080]                          │
 │        │  Pipeline (Jenkinsfile)           │
 │        ▼                                   │
 │   Checkout → Build → Deploy → Health Check │
 │        │                                   │
 │        ▼                                   │
 │   [Docker Compose]                         │
 │    ┌──────────┬──────────┬──────────┐      │
 │    │ Frontend │ Backend  │ Postgres │      │
 │    │ (Nginx)  │  (Go)    │  (DB)    │      │
 │    │  :80     │  :8080   │  :5432   │      │
 │    └──────────┴──────────┴──────────┘      │
 └────────────────────────────────────────────┘
         ▲
         │  http://IP-VM
    [User / Browser]
```

---

## ✨ Fitur Utama

- 🔁 **Trigger Otomatis** — push ke GitHub langsung memicu build lewat webhook
- 🐳 **Containerized Deployment** — setiap komponen berjalan di container terisolasi
- 🧱 **Multi-stage Build** — image akhir kecil (Go dan React di-compile, hanya hasilnya yang dibawa)
- ❤️ **Health Check Stage** — pipeline memverifikasi aplikasi benar-benar hidup setelah deploy
- 🔒 **Disable Concurrent Builds** — mencegah dua build berjalan bersamaan
- 📋 **Auto Log on Failure** — saat gagal, pipeline menampilkan log container untuk debugging
- 💾 **Persistent Volume** — data PostgreSQL tetap aman meski container di-restart

---

## 🛠️ Tech Stack

| Komponen | Teknologi | Fungsi |
|----------|-----------|--------|
| CI/CD Server | Jenkins 2.555.x | Menjalankan pipeline otomatis |
| Source Control | GitHub | Menyimpan kode & mengirim webhook |
| Tunnel | ngrok | Mengekspos Jenkins (IP lokal) agar bisa dijangkau GitHub |
| OS Server | Ubuntu Server 24.04 (VMware, NAT) | Host Jenkins & aplikasi |
| Runtime Jenkins | OpenJDK 21 | Syarat minimum Jenkins versi terbaru |
| Container | Docker + Docker Compose | Isolasi & orkestrasi service |
| Web Server | Nginx | Serve static file & reverse proxy `/api` |
| Backend | Go (`net/http`) | REST API |
| Frontend | React (Create React App) | Antarmuka pengguna |
| Database | PostgreSQL 16 | Penyimpanan data |

---

## 📂 Struktur Repositori Proyek

```
todo-cicd/
├── backend/
│   ├── main.go            REST API + auto-migrate tabel
│   ├── go.mod / go.sum
│   └── Dockerfile         Multi-stage: golang → alpine
├── frontend/
│   ├── src/               Kode React
│   ├── nginx.conf         Serve static + proxy /api → backend
│   └── Dockerfile         Multi-stage: node → nginx
├── docker-compose.yml     postgres + backend + frontend
└── Jenkinsfile            Definisi pipeline
```

---

## 🧭 Langkah Implementasi

### Fase 1 — Instalasi Jenkins di Ubuntu Server

**1. Install Java 21** (Jenkins terbaru menolak Java 17)
```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y
java -version
```
> Fungsi: Jenkins ditulis dalam Java, sehingga butuh JRE untuk berjalan.

**2. Tambahkan repository & key Jenkins** (gunakan key **2026**)
```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```
> Fungsi: `apt` memverifikasi paket dengan GPG key. Key yang salah/kedaluwarsa membuat repository ditolak.

**3. Install & jalankan Jenkins**
```bash
sudo apt update
sudo apt install jenkins -y
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

**4. Buka firewall & ambil password awal**
```bash
sudo ufw allow 8080
sudo ufw allow OpenSSH
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**5. Setup wizard di browser** → `http://IP-VM:8080`
Unlock dengan password di atas → *Install suggested plugins* → buat admin user (jangan klik *Skip*).
> ⚠️ Jangan mengedit file `initialAdminPassword` secara manual. File itu hanya dibaca sekali oleh wizard.

---

### Fase 2 — Pipeline Pertama (Node.js + systemd)

**1. Install Node.js di server** (pipeline berjalan *di server*, bukan di laptop)
```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install nodejs -y
```

**2. Buat service systemd** — `/etc/systemd/system/belajar-cicd.service`
```ini
[Unit]
Description=Belajar CICD Node.js App
After=network.target

[Service]
Type=simple
User=jenkins
WorkingDirectory=/var/lib/jenkins/workspace/belajar ci-cd pipeline
ExecStart=/usr/bin/node index.js
Restart=on-failure
Environment=PORT=3000

[Install]
WantedBy=multi-user.target
```
> Fungsi: systemd menjaga proses tetap hidup **terlepas dari Jenkins**, otomatis restart jika crash, dan jalan lagi saat server reboot.

**3. Izinkan user `jenkins` me-restart service tanpa password** — `/etc/sudoers.d/jenkins-restart`
```
jenkins ALL=(ALL) NOPASSWD: /bin/systemctl restart belajar-cicd
jenkins ALL=(ALL) NOPASSWD: /bin/systemctl status belajar-cicd*
```
> Fungsi: pipeline berjalan non-interaktif sehingga tidak bisa mengetik password sudo. Tanda `*` diperlukan agar argumen seperti `--no-pager` ikut cocok.

**4. Jenkinsfile**
```groovy
pipeline {
    agent any
    stages {
        stage('Checkout')             { steps { checkout scm } }
        stage('Install Dependencies') { steps { sh 'npm install' } }
        stage('Deploy') {
            steps {
                sh '''
                    sudo systemctl restart belajar-cicd
                    sleep 2
                    sudo systemctl status belajar-cicd --no-pager
                '''
            }
        }
    }
}
```

**5. Buat job di Jenkins**
`New Item` → `Pipeline` → *Pipeline script from SCM* → SCM `Git` → isi Repository URL → Branch `*/main` → Script Path `Jenkinsfile`.
> Jika opsi Git tidak muncul, install plugin **Git** lewat *Manage Jenkins → Plugins*.

---

### Fase 3 — Trigger Otomatis (Webhook + ngrok)

Jenkins berjalan di IP lokal (`192.168.x.x`) yang tidak bisa dijangkau GitHub. ngrok membuat tunnel publik menuju Jenkins.

**1. Pasang & jalankan ngrok**
```bash
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install ngrok -y
ngrok config add-authtoken <TOKEN_NGROK_ANDA>
ngrok http 8080
```

**2. Set Jenkins URL** — *Manage Jenkins → System → Jenkins URL* = URL ngrok.

**3. Tambah webhook di GitHub** — *Repo → Settings → Webhooks → Add webhook*

| Field | Isi |
|-------|-----|
| Payload URL | `https://<subdomain-ngrok>/github-webhook/` |
| Content type | `application/json` |
| Events | Just the push event |

**4. Aktifkan trigger di job** — *Configure → Build Triggers →* ✅ `GitHub hook trigger for GITScm polling`

> Webhook dibuat **per repository**, jadi setiap repo baru butuh webhook baru. Tunnel ngrok harus selalu aktif; jika mati, push tidak akan memicu build.

---

### Fase 4 — Pipeline Multi-Service (Go + React + PostgreSQL)

**1. Install Docker & beri akses ke user `jenkins`**
```bash
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

sudo usermod -aG docker jenkins
sudo systemctl restart docker jenkins
```
> Fungsi: user `jenkins` perlu masuk grup `docker` agar bisa menjalankan `docker` tanpa `sudo`.

**2. Dockerfile backend (multi-stage)**
```dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/main .
EXPOSE 8080
CMD ["./main"]
```
> Fungsi: tahap pertama meng-compile, tahap kedua hanya membawa binary. Image akhir kecil dan tanpa compiler.

**3. Dockerfile frontend (multi-stage)**
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```
> `package*.json` disalin lebih dulu agar layer `npm install` bisa di-cache selama dependency tidak berubah.

**4. docker-compose.yml**
```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: todo-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks: [todo-network]

  backend:
    build: ./backend
    container_name: todo-backend
    restart: unless-stopped
    environment:
      DB_HOST: postgres          # nama service = hostname di network Docker
      DB_PORT: 5432
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_NAME: ${DB_NAME}
    depends_on: [postgres]
    networks: [todo-network]

  frontend:
    build: ./frontend
    container_name: todo-frontend
    restart: unless-stopped
    ports: ["80:80"]
    depends_on: [backend]
    networks: [todo-network]

volumes:
  postgres_data:
networks:
  todo-network:
```
> Hanya `frontend` yang membuka port ke luar. Backend dan database hanya bisa diakses dari dalam network Docker.

**5. Jenkinsfile (dengan health check)**
```groovy
pipeline {
    agent any
    options { disableConcurrentBuilds() }

    stages {
        stage('Checkout')     { steps { checkout scm } }
        stage('Build Images') { steps { sh 'docker compose build' } }
        stage('Deploy') {
            steps {
                sh '''
                    docker compose down
                    docker compose up -d
                '''
            }
        }
        stage('Health Check') {
            steps {
                sh '''
                    sleep 8
                    curl -f http://localhost/api/health
                '''
            }
        }
    }

    post {
        failure { sh 'docker compose logs --tail=50 || true' }
    }
}
```

---

### Fase 5 — Website Statis (Danau Toba)

Pola yang sama dipakai ulang dengan penyesuaian:

- Dockerfile cukup **satu tahap** (HTML/CSS/JS statis langsung di-serve Nginx)
- Port host diubah ke **8081** agar tidak bentrok dengan aplikasi lain di port 80
- Buat **job Jenkins baru** dan **webhook baru** khusus repo tersebut

---

## 🔧 Troubleshooting (Masalah Nyata yang Ditemui)

| # | Gejala | Penyebab | Solusi |
|---|--------|----------|--------|
| 1 | `NO_PUBKEY 7198F4B714ABFC68`, repo *not signed* | Key `jenkins.io-2023.key` sudah tidak berlaku | Pakai `jenkins.io-2026.key` |
| 2 | Jenkins gagal start, log: *Java 17 older than minimum (Java 21)* | Versi Java terlalu lama | `sudo apt install openjdk-21-jre` |
| 3 | Halaman setup wizard *Offline* | Wizard gagal mengecek update site sesaat | Refresh / restart Jenkins; pastikan VM punya internet |
| 4 | Opsi **Git** tidak ada di dropdown SCM | Plugin Git belum terinstall | *Manage Jenkins → Plugins → Available → Git* |
| 5 | `ERROR: No flow definition, cannot run` | Build dijalankan sebelum konfigurasi SCM tersimpan | Simpan konfigurasi, lalu Build Now ulang |
| 6 | `npm: not found` di stage Install | Node.js hanya ada di laptop, belum di server | Install Node.js di Ubuntu Server |
| 7 | Aplikasi mati setelah pipeline selesai | Proses background (`nohup &`) dibersihkan Jenkins | Kelola proses lewat **systemd** |
| 8 | PM2: `spawn /usr/bin/node EACCES` | Daemon PM2 milik user `jenkins` bermasalah | Beralih ke **systemd** |
| 9 | `sudo: a password is required` | Aturan sudoers tidak cocok dengan argumen (`--no-pager`) | Tambah wildcard `*` di sudoers |
| 10 | App jalan di VM, tak bisa diakses dari Windows | Port belum dibuka di firewall, atau salah port | `sudo ufw allow <port>`; cek port yang dipakai |
| 11 | `missing go.sum entry for module ... lib/pq` | `go.sum` belum ada / tidak lengkap | `docker run --rm -v $(pwd)/backend:/app -w /app golang:1.23-alpine go mod tidy` |
| 12 | `TLS handshake timeout` ke `sum.golang.org` | Koneksi ke checksum DB lambat/terblokir | Lihat catatan keamanan di bawah |
| 13 | `container name "/todo-postgres" is already in use` | Container hasil `compose up` manual bentrok dengan build Jenkins | `docker compose down` di folder lama, hapus container sisa |
| 14 | Push tidak memicu build | Tunnel ngrok mati | Jalankan ulang `ngrok http 8080` |
| 15 | `git push` → `rejected (fetch first)` | Remote punya commit yang belum ada di lokal | `git pull origin main`, lalu push |
| 16 | `Password authentication is not supported` | GitHub menolak password akun untuk HTTPS | Pakai **Personal Access Token** |

---

## ⚠️ Catatan Keamanan & Batasan

Proyek ini dibuat untuk **pembelajaran**. Sebelum dipakai di lingkungan nyata, perhatikan:

- **`GOSUMDB=off`** (opsi untuk masalah #12) mematikan verifikasi checksum dependency Go ke server publik. Ini hanya mengandalkan `go.sum` lokal. Untuk produksi, sebaiknya tetap aktifkan verifikasi atau gunakan `GOPROXY` internal.
- **Kredensial database** jangan di-hardcode di `docker-compose.yml` yang di-commit. Gunakan file `.env` (masuk `.gitignore`) atau **Jenkins Credentials**.
- **ngrok free** memberi URL publik ke Jenkins; aktifkan autentikasi Jenkins dan jangan biarkan tunnel menyala terus-menerus.
- **CORS** pada backend contoh dibuka untuk semua origin; batasi ke domain frontend di produksi.
- Token GitHub (PAT) tidak boleh di-commit; beri masa berlaku dan scope seminimal mungkin.

---

## 🧪 Cara Memverifikasi Pipeline

```bash
# 1. Edit kode di Windows, lalu:
git pull origin main
git add .
git commit -m "update tampilan"
git push

# 2. Pantau Jenkins → job → build baru muncul otomatis
#    Baris pertama Console Output harus:
#    "Started by GitHub push by <username>"

# 3. Pastikan status: Finished: SUCCESS
# 4. Refresh browser → perubahan langsung tampil
```

> Jika baris pertama berbunyi `Started by user ...`, berarti build dipicu manual, bukan lewat webhook.

---

## 📚 Pelajaran Utama

1. **Pipeline berjalan di server, bukan di laptop** — semua tool (Node, Docker, dll.) harus ada di server Jenkins.
2. **Proses jangan dijalankan langsung dari pipeline** — serahkan ke systemd atau Docker agar tidak ikut mati.
3. **Bedakan hak akses per user** — `jenkins` adalah user terpisah dengan izin (sudo, grup docker) yang harus diatur eksplisit.
4. **Webhook per repo, tunnel harus hidup** — dua hal yang paling sering terlupa.
5. **Satu sumber kebenaran** — jalankan deploy hanya lewat Jenkins agar tidak terjadi konflik container.

---

## 📄 Lisensi

MIT License — bebas digunakan untuk keperluan akademik
