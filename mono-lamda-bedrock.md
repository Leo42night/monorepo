# PPWL 11 - AWS Lambda & Bedrock
- Menggunakan fitur AWS Lambda & Bedrock.
- File ini akan di update bertahap, karena ada beberapa tahap yang perlu dirapikan (akan ada info updatenya rutin di grup WA).
- Untuk sekarang anda dapat langsung mengerjakan tahap yang sudah fix di bawah. 

## 1. Install & Login
⚠️ **Disclaimer**: tutorial ini di jalankan menggunakan WSL archLinux [wsl paling ringan], jadi mungkin akan ada kendala dependency & cara instalasi bagi yang environment berbeda.

**Docker & docker-buildx**: Untuk windows yang mau download Docker CLI disarankan pakai WSL (kalo udh ada Docker Desktop gpp, Docker CLI  udh include otomatis).
```bash
# --- Docker ---
docker --version
## Cth:
## | [root@LeoLPC ppwl10-ec2]# docker --version
## | Docker version 29.4.0, build 9d7ad9ff18
docker login
## Login pakai akun Github biar mudah.
## klik url yang tampil di terminal, input-kan one time code yang tampil di terminal.
## Jika berhasil login, harusnya tampil pesan `Login Succeeded`.

# --- Docker Buildx ---
# 1. Install docker-buildx (untuk build image, diperlukan pada versi docker terbaru)
## contoh install di ArchLinux (sesuaikan dengan package manager anda)
sudo pacman -Syu docker-buildx
# 2. Cek Versi
docker buildx version
## Cth:
## | [root@LeoLPC ppwl10-ec2]# docker buildx version
## | github.com/docker/buildx 0.33.0 f7897eba028583e0071642db3c011e860444f8cf
```
Docker & Docker Buildx sudah terinstall. Build image docker jadi lebih detail.

**AWS**: [Docs setup AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions)
```bash
# Khusus untuk Linux/WSL
cd ~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
## Cth:
## | [root@LeoLPC ppwl10-ec2]# aws --version
## | aws-cli/2.34.30 Python/3.14.4 Linux/6.6.87.2-microsoft-standard-WSL2 exe/x86_64.arch
# run `rm awscliv2.zip` untuk hapus file sisa instalasi

#  setelah install AWS CLI, jalankan perintah untuk set default region (cth: `us-east-1`)
aws configure set region us-east-1
# Login ke AWS
aws login --remote
```
