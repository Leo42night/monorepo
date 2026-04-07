# Mono Docker EC2 AWS 

## A. Setup Wallet & Billing
Bagi yang ingin pakai rekening Bank Konvensional (Mandiri, BCA, dsb.), langsung ke step-2, pastikan kartu sudah mendukung transaksi internasional atau terkoneksi Visa.

### 1. Buat Kartu Jago
- Download Aplikasi Jago. 
- Login: masukkan data diri & alamat yang sesuai dengan KTP dan akun akun google (pribadi, jangan pakai akun UNTAN).
- Buat "Kartu Debit Digital" Visa Bank Jago, jadikan `kantong utama`.
- Pastikan alamat rumah konsisten untuk akun bank Jago dan akun google.

### 2. Verifikasi Kartu
- Verifikasi nomor dengan [Credit Card Validator](https://dnschecker.org/credit-card-validator.php).

### 3. Koneksi Billing
Tutorial koneksi kartu ke AWS Free Tier:
- [Cara Daftar Akun AWS Free Tier 2025](https://www.youtube.com/watch?v=NB7F8RaCY1o)

Tutorial Koneksi kartu ke GCP Free Trial:
- Verifikasi kartu visa dapat digunakan dengan koneksi ke e-wallet google. 
- Pastikan saldo min. `Rp50k` untuk `Tagihan Sementara` untuk google dapat verifikasi kartu. Saldo akan dikembalikan setelah verifikasi berhasil (paling lama 7 hari kerja).
- Setelah rekening terkoneksi ke e wallet, pendaftarkan ke GCP 300$ free trial akan lebih mudah.
- Verifikasi Google Cloud Billing buntuh min. saldo `Rp150k`, data dapat di refund seletah verifikasi berhasil. 

## B. EC2 + Docker Compose
Jalankan `docker compose up` langsung di EC2, persis seperti di lokal.

### Step 0: Prerequisite
Selesaikan Lebih dulu:
- Setup docker di `monorepo-docker.md` 
- Setup Environement production seperti Turso Database & src/index.ts di `monorepo-4.md` 

### Step 1: Buat EC2 Instance

Di AWS Console, search halaman `Instances` (EC2 Feature), lalu klik `Launce Instances`:
- **AMI** (Default): `Amazon Linux 2023`
- **Instance type**: pilih `t3.small` (minimal 2GB Memory RAM)
- Buat atau pilih **Key Pair** (`key.pem`), simpan baik-baik.
- **Network Setting**: Allow SSH & HTTP.
  - Klik `Edit`. Pastikan **Inbound Security Group** — buka port ini:

| Port | Tujuan      | TYPE       | Source             | 
|------|-------------|------------|--------------------|
| 22   | SSH         | SSH        | Anywhere 0.0.0.0/0 |
| 80   | HTTP (Nginx)| HTTP       | Anywhere 0.0.0.0/0 |
| 5173 | Frontend    | Custom TCP | Anywhere 0.0.0.0/0 |
| 3000 | Backend API | Custom TCP | Anywhere 0.0.0.0/0 |

- Finally: klik `Launce Instance`
- Jika sudah di launch, Klik nama instance (tampil detail instance)-> Klik `Connect` -> Klik tab `SSH Client` (ikut & salin command koneksi SSH).

### Step 2: Install Docker di EC2
di halaman AWS Console EC2 `Instances`, Buka instance, lalu Klik tombo `Connect` untuk dapat command connect SSH.
```bash
# (di wsl) copy file `key.pem` (di downloads/ windows) ke ~ (home linux)
# contoh:
/Downloads$ cp key.pem ~   
# masuk ke ~ (home)
cd ~

# paste `Example` di halaman aws -> Instance -> Connect -> SSH Client, 
# contoh (`ec2-user` adalah username default untuk Amazon Linux):
ssh -i "key.pem" ec2-user@ec2-44-xx-xx-xx.compute-1.amazonaws.com
# 🚨🚨 Anda akan sering run command ini, karena SSH akan sering disconnect

# Perintah di bawah khusus untuk Amazon Linux (bukan Ubuntu/Debian)
# 1. Update sistem (Opsional tapi disarankan)
sudo dnf update -y

# 2. Install Docker
sudo dnf install -y docker

# 3. Install Docker Compose
# download the latest version in the global plugins directory
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-$(uname -m) -o /usr/libexec/docker/cli-plugins/docker-compose

# make it executable
sudo chmod +x /usr/libexec/docker/cli-plugins/docker-compose

# check
docker compose version

# 4. Start Docker service
sudo systemctl start docker

# 5. Atur agar Docker otomatis jalan saat server reboot
sudo systemctl enable docker

# 6. Tambah user ke group docker
# Catatan: User default di Amazon Linux adalah 'ec2-user'
sudo usermod -aG docker ec2-user

# 7. Terapkan perubahan grup tanpa logout
newgrp docker
```

### Step 3: Upload source code ke EC2

**Via Git, di EC2:**
```bash
# --- Buat koneksi SSH ke Github ---
## 1. Check for Existing SSH Keys 
ls -al ~/.ssh
## If ada id_ed25519.pub or id_rsa.pub, you can skip to Step 3
## 2. Generate a New SSH Key 
ssh-keygen -t ed25519 -C "your_email@example.com"
## Ketik enter saja terus (biarkan default)
## 3. Add Your SSH Key to GitHub 
cat ~/.ssh/id_ed25519.pub
## -> Copy the key
## -> In GitHub: go to Settings > `SSH and GPG keys`
## Add Key: Click New SSH key, give it a title (e.g., "Work Laptop"), and paste your key into the field.
## 4. Test the Connection
ssh -T git@github.com

# Amazon Linux SSH Login
sudo dnf install -y git
git clone git@github.com:<username>/mono-docker.git
cd mono-docker
```

### Step 4: Buat file `.env` untuk secrets

```bash
# Di EC2, dalam folder mono-docker/
nano .env
# Tips Nano: Ctrl+Shift+V (Paste). Ctrl+X -> Y (Simpan)
```

Contoh data seperti ini (sesuaikan dengan punya anda, gunakan `public IPV4 address`):
```bash
GOOGLE_CLIENT_ID=xxxx48786911-cn9xxxvsheuim2ujs7q7d7na30nmqpfv.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=xxxPX-Iw2F5hx6vEG7PGVW_jta0bJmxxxx
GOOGLE_REDIRECT_URI=http://44.210.xxx.xxx:3000/auth/callback
SESSION_SECRET=xxx6dd5f1554c09543c39a2a105axxx
API_KEY="learn"

DATABASE_URL=libsql://monorepo-xxxx.aws-ap-northeast-1.turso.io
DB_AUTH_TOKEN=xxxxGciOiJFZERTQSIsInR5cCI6IkpXVCJ9
FRONTEND_URL=http://44.210.xxx.xxx:5173

VITE_CHECK="vite env check"
VITE_BACKEND_URL=http://44.210.xxx.xxx:3000
```
Copy .env ke frontend agar `VITE_*` terbaca vite saat build:
```bash
cp .env apps/frontend
```

Update `docker-compose.yml` supaya backend bisa baca `.env` saat runtime:
```yaml
services:
  backend:
    build:
      context: .
      dockerfile: apps/backend/Dockerfile
    ports:
      - "3000:3000"
    env_file:
      - .env          # ← tambahkan ini
    restart: unless-stopped
```
frontend tidak perlu set env_file di docker composer, karena env sudah di convert saat build pakai vite.

### Step 5: Build dan jalankan

```bash
cd mono-docker
docker compose up --build -d

# --- START: FIX BUG Docker Buildx ---
# Jika dapat `requires buildx 0.17.0 or later` -> Upgrade Docker Buildx
# Cek versi buildx sekarang (asdos cek 0.12.1)
docker buildx version 

# Download buildx versi terbaru
mkdir -p ~/.docker/cli-plugins
curl -L "https://github.com/docker/buildx/releases/download/v0.33.0/buildx-v0.33.0.linux-amd64" -o ~/.docker/cli-plugins/docker-buildx

# Beri izin eksekusi
chmod +x ~/.docker/cli-plugins/docker-buildx

# Verifikasi (jika sudah update v0.33.0, run lagi compose up sebelumnya)
docker buildx version
# --- END: FIX BUG Docker Buildx ---
```

Cek status:
```bash
docker compose ps
docker compose logs -f backend

# jika bermasalah build, selalu coba untuk ikut cara di monorepo-docker.md
# bisa coba build backend terlebih dahulu, lalu frontend
# jika proses di Killed (RAM Habis) bisa coba Tambah Swap Space
# Jika ada update, jalankan ulang shortcut up build:
docker compose up --build -d
```

Akses:
- Frontend: `http://your-ec2-public-ipv4:5173`
   - Lihat root `/` apakah tampil data users dan backend.
- Backend: `http://your-ec2-public-ipv4:3000`
   - Test periksa path `/users`.

## C. Final
Kumpulkan:
1. **Link**: 
   - Repo **`https://github.com/<username>/ppwl10-ec2`**
   - Frontend & Backend
2. **1 Gambar ScreenShot**: Terminal Docker Jalan dengan latar belakang halaman web backend route users. Full Screen. [Contoh Submisi](https://drive.google.com/file/d/10R7xLgCjEVcQhzm9E1Pma52mgEzTsJJh/view?usp=drive_link)

Lihat juga [Bebera SS Proses Setup](https://drive.google.com/drive/folders/1PDZXk2cN-5ieJyH_3CSojy8jU4HPdBD0?usp=drive_link) untuk memmbantu beri gambaran setingan yang tepat.

--- 

## Alert
Fitur Google Auth tidak dapat redirect ke ip `http://` jika pakai public ip, jadi perlu setup domain. Tutorial saat ini belum sampai ke situ.
