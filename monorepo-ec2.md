# Mono Docker EC2 AWS 

## A. Setup Wallet & Billing
Untuk admin AWS: Bisa ikuti tutorial di sini untuk [Setup Kartu Debit/Kredit dan koneksi Billing ke AWS](https://docs.google.com/document/d/1EEtbymKJmg64SjbYX8Qzg4_JhZR_KcImVMgPmti24aE/edit?usp=sharing). Admin dapat beri akses akun IAM ke tim nya untuk akses layanan EC2 AWS.

## B. EC2 + Docker Compose
Jalankan `docker compose up` langsung di EC2, persis seperti di lokal.

### Step 0: Prerequisite
Selesaikan Lebih dulu:
- Setup docker di `monorepo-docker.md` 
- Setup Environement production seperti Turso Database, src/index.ts, & di Frontend `*.tsx` & `vite.config.ts` di `monorepo-4.md` (Menggunakan value Environment Variable, bukan sekadar http://localhost) 

### Step 1: Buat EC2 Instance

Di AWS Console, search halaman `EC2` -> `Instances` (EC2 Feature), lalu klik `Launce Instances`:
- **Name**: Nama User (cth: `user-1`)
- **AMI** (Default): `Amazon Linux 2023`
- **Instance type**: pilih `t3.small` (minimal 2GB Memory RAM)
- Buat atau pilih **Key Pair** (`key.pem`), simpan baik-baik. nama Key Pair = Nama User (cth: `user-1`)
- **Network Setting**: Allow SSH & HTTP.
  - Klik `Edit`. Pastikan **Inbound Security Group**
  - **Security group name** berikan postfix nama user (cth: `launch-wizard-12-user-1`)
  - buka port ini:

| Port | Tujuan      | TYPE       | Source             | 
|------|-------------|------------|--------------------|
| 22   | SSH         | SSH        | Anywhere 0.0.0.0/0 |
| 80   | HTTP (Nginx)| HTTP       | Anywhere 0.0.0.0/0 |
| 5173 | Frontend    | Custom TCP | Anywhere 0.0.0.0/0 |
| 3000 | Backend API | Custom TCP | Anywhere 0.0.0.0/0 |

- Finally: klik `Launce Instance`
- Jika sudah di launch, Klik nama instance (tampil detail instance)-> Klik `Connect` ->
   - Di tab `EC2 Instance Connect` salin `Public IPv4 address`  
   - Klik tab `SSH Client` (salin `Example` koneksi SSH: cth **ssh -i "user-1.pem"...**).

### Step 2: Install Docker di EC2
di halaman AWS Console EC2 `Instances`, Buka instance, lalu Klik tombo `Connect` untuk dapat command connect SSH.
```bash
# di linux atau mac: jalankan dulu protected pem (wajib sebelum koneksi SSH).
chmod 400 key.pem
# di windows: kalian bisa langsung buka cmd di halaman file `*.pem` kalian.

# paste `Example` di halaman aws -> Instance -> Connect -> SSH Client.
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
ssh-keygen -t ed25519 -C "your_github_email@gmail.com"
## Ketik enter saja terus (biarkan default)
## 3. Add Your SSH Key to GitHub 
cat ~/.ssh/id_ed25519.pub
## -> Copy the key
## -> In GitHub: klik profile User (di kanan atas) -> klik `Settings` -> `SSH and GPG keys`
## Add Key: Click New SSH key, give it a title (e.g., "aws user"), and paste your key into the field.
## 4. Test the Connection
ssh -T git@github.com
## -> klik `yes`

# Amazon Linux SSH Login
sudo dnf install -y git
git clone git@github.com:<username>/mono-docker.git
# bagi yang kehilangan proyek atar tertinggal jauh, bisa pakai ini https://github.com/kaniaa-kr/ppwl10-ec2
cd mono-docker
```
Jika ingin melakukan beberapa perubahan file. ubah saja di local, push ke repo. lalu di EC2 pull. terutama:
- Fetch di frontend App2.tsx gunakan VITE_BACKEND_URL seperti di tutorial `Monorepo-4.md -> B. Apps/Frontend`

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
# tekan ctrl+c untuk terminate log dan tampilkan kembali command line
docker compose logs -f frontend

# jika ada perubahan, bisa down dulu lalu lakukan up prefix build
docker compose down
docker compose up --build

# referensi perbaikan docker ada di monorepo-docker.md
```

Akses:
- Frontend: `http://your-ec2-public-ipv4:5173`
   - Lihat root `/` apakah tampil data users dan backend. (path `/classroom` gk dulu, memang perlu setingan lanjutan)
- Backend: `http://your-ec2-public-ipv4:3000`
   - Test periksa path `/users`.

## C. Final
Kumpulkan:
1. **Link** Repo **`https://github.com/<username>/ppwl10-ec2`**
2. **Link** Frontend **`http://public-IPv4:5173`**
3. **Link** Backend /users **`http://public-IPv4:3000/users?key=learn`**
3. **1 Gambar ScreenShot**: Terminal Docker Jalan dengan latar belakang halaman web backend route users. Full Screen. [Contoh Submisi](https://drive.google.com/file/d/10R7xLgCjEVcQhzm9E1Pma52mgEzTsJJh/view?usp=drive_link)

Lihat juga [Bebera SS Proses Setup](https://drive.google.com/drive/folders/1PDZXk2cN-5ieJyH_3CSojy8jU4HPdBD0?usp=drive_link) untuk memmbantu beri gambaran setingan yang tepat.

[Video Tutorial Buat Instance, Setup EC2 Docker & Push kode dari EC2 ke Repo](https://drive.google.com/file/d/1Km4m2t5PpdiTwq44euX-Yul3YQEVCmLq/view?usp=drive_link)

--- 

## Alert
Fitur Google Auth tidak dapat redirect ke ip `http://` jika pakai public ip, jadi perlu setup domain. Tutorial saat ini belum sampai ke situ. Jadi fitur `/classroom` memang belum dapat diakses.
