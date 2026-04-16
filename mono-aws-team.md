# PPWL 11 - AWS Team
Instruction Team:
- Proyek ini dikerjakan sesuia Tim yang dibagikan.
- Ada total 6 Fase instruksi (untuk masing-masing anggota).
- Fase 5 & 6 Dapat dikerjakan 1 orang apabila anggota cuma 5.
Brief Project:
- Menggunakan fitur AWS RDS (PostgreSQL), AWS Budgets, AWS S3 Cloudfront atau AWS Lambda.
- 6 Fase/Role:
  - 1. AWS Admin: Root akses · Setup global · Koordinator tim
  - 2. IAM Client A (AWS Budgets): Cost management
  - 3. IAM Client B (Aurora / RDS): Database layer 
  - 4. IAM Client C (Lambda — Backend): Elysia API serverless
  - 5. IAM Client D (Lambda — Frontend): React static via S3+CloudFront atau Lambda
  - 6. IAM Client E (Opsional, Integrasi & Dokumentasi): Jembatan semua komponen + laporan akhir
- Submisi mungkin akan menggunakan google docs agar rapi (karena ada segmen laporan).

⚠️ **Announcement**: 
- File ini akan di update bertahap, karena ada beberapa tahap yang perlu dirapikan (akan ada info updatenya rutin di grup WA).
- Untuk sekarang anda dapat langsung mengerjakan tahap yang sudah fix di bawah. 

## Install & Login
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

Tips ✨: 
- Gunakan `US-Virginia (us-east-1)` jika ingin cost paling murah, jadi credit tidak terkuras banyak.
- Tiap pindah fitur AWS Console, Buka tab baru, agar mudah kembali ke tab sebelumya. 
- Gunakan Notepad untuk menyimpan key/config yang akan digunakan lagi.

## Fase 0 — Admin: IAM, VPC, Parameter Store

### 1. Buat IAM Group dan User untuk tiap anggota
Masuk ke AWS Console → IAM → Groups → Create group. Buat 4 group sesuai tugas anggota.

**Gunakan policy type "AWS Managed"**
```bash
Group: grp-budget
  → AWSBudgetsActionsWithAWSResourceControlAccess
  → AWSCostAndUsageReportAutomationPolicy

Group: grp-database  
  → AmazonRDSFullAccess
  → AmazonVPCReadOnlyAccess

Group: grp-lambda-be
  → AWSLambda_FullAccess
  → AmazonSSMReadOnlyAccess
  → AmazonVPCReadOnlyAccess

Group: grp-lambda-fe
  → AWSLambda_FullAccess
  → AmazonS3FullAccess
  → CloudFrontFullAccess
```

**Custom Additional Inline Policy**
```bash
# AWS Budget butuh policy "ce:DescribeReport"
-> masuk ke group "grp-budget" -> Permissions -> Create Inline Policy
    Select a service "Cost Explorer Service" # di JSON codename nya "ce" (jadi anda perlu search arti dari code nya)
    Search allowed Actions -> "DescribeReport"
    Resource All (*)  
    # Lihat di JSON, kode action nya menjadi "ce:DescribeReport"
    # tabahkan juga action "ce:GetDimensionValues", "ce:GetSavingsPlansPurchaseRecommendation" and "ce:GetSavingsPlansCoverage"
    -> Name: additionalInlinePolicy_grpBudget
# Lakukan hal yang sama untuk policy lain jika IAM User butuh akses fitur.
```
**✨ Tips:** Admin bisa cek login ke akun IAM, dengan buka di Incognito Mode, jadi session login lain tidak terpengaruh. 

**Buat IAM User**, contoh (lakukan juga untuk Anggota lain, sesuaikan Group nya):
```bash
IAM → Users → Create user
  Username: database-anggota-b (atau nama asli, kasih job ke nama nya biar jelas role nya)
  Access type: AWS Management Console access
  Console password: custom / auto-generated
  Assign to group: grp-database
  → Download credentials.csv → kirim ke anggota via chat

# Tambahkan user dengan name "asdos", beri policy "AdministratorAccess", kirim csv lewat WA ke Asdos-Leo. 
```
> Kalau tim hanya 5 orang, tugaskan Client E (Integrasi) ke Client D, karena keduanya paling erat berkaitan (build frontend + verifikasi end-to-end).

### 2. Setup VPC dan Security Group
Gunakan default VPC jika ada, atau buat VPC baru. Yang penting: Security Group untuk RDS hanya terima koneksi dari Lambda.

**Buat Security Group untuk RDS**
```bash
VPC → Security Groups → Create security group
  Name: sgRdsInternal
  Description: RDS Security Group for internal PostgreSQL access from within VPC
  VPC: (pilih VPC kamu, atau biarkan default)
  
  Inbound rules:
    Type: PostgreSQL (5432)
    Source: Custom → sgLambda (akan diisi setelah Lambda SG dibuat, untuk sekarang skip dulu)
  
  Outbound rules: semua traffic (default)
```

**Buat Security Group untuk Lambda**
```bash
Name: sgLambda
Desc: Security group for Lambda functions to access RDS and external APIs
  Inbound: tidak perlu (Lambda dipanggil via URL, bukan VPC ingress)
  Outbound:
    Type: PostgreSQL (5432) → Destination: sgRdsInternal
    Type: HTTPS (443) → Destination: Anywhere-IPv4 (untuk Turso/LibSQL)
```
> Setelah sgLambda terbentuk, balik ke sgRdsInternal dan edit inbound rule: ubah source dari "Custom" ke sgLambda ID.

### 3. Simpan semua env vars ke Parameter Store
Jangan pernah hardcode secret di Lambda env vars. Simpan di Systems Manager Parameter Store, tipe SecureString.

**Buat parameter (tiap baris = satu parameter).**
```bash
AWS Systems Manager → Parameter Store → Create parameter

/monorepo/GOOGLE_CLIENT_ID         → String
/monorepo/GOOGLE_CLIENT_SECRET     → SecureString
/monorepo/GOOGLE_REDIRECT_URI      → String  (isi nanti setelah Lambda URL diketahui)
/monorepo/SESSION_SECRET           → SecureString
/monorepo/DATABASE_URL             → SecureString  (isi setelah Anggota B selesai RDS)
/monorepo/DB_AUTH_TOKEN            → SecureString
/monorepo/API_KEY                  → SecureString
/monorepo/FRONTEND_URL             → String  (isi nanti setelah S3/CloudFront URL diketahui)
```

**Tambahkan policy baca Parameter Store ke Lambda role (nanti)**
```bash
IAM → Policies → Create policy → Visual:
  Service: System Manager
  Action Allowed: GetParameter, GetParameters, GetParametersByPath
  Resource: -> add ARN 
    Resource Region: centang "Any Region"
    Resource Parameter: "monorepo/*"
  -> Next -> Review and Create
    Name: AmazonSSMParameterStoreRead_Monorepo
    Desc: Allows Lambda functions to read configuration parameters under the monorepo path from AWS Systems Manager Parameter Store.  
```

- ✅ Bagikan nama parameter path (/monorepo/...) ke Anggota C
- ✅ Jangan bagikan nilai secret-nya langsung — anggota C cukup tahu nama key-nya


### 4. Buat S3 bucket untuk frontend (persiapan Anggota D)
Admin buat bucket sekarang supaya Anggota D bisa langsung upload nanti.
```bash
S3 → Create bucket
  Region: us-east-1
  Bucket name: s3-monorepo-frontend-prod (harus globally unique)
  Block all public access: OFF (kita perlu public baca)
  Centang "I acknowledge that the current ... becoming public."
  
→ Setelah bucket dibuat:
  → <dalam bucked> tab Permissions → Bucket Policy → edit:
    -> Add New Statement -> Choose a service "S3"
    -> Search action "GetObject"
    -> Add Resource 
        -> Service S3
        -> Type "object"
        -> Resource ARN "arn:aws:s3:::s3-monorepo-frontend-prod/*" 
    -> edit JSON "Principal": "*" 

  → <dalam bucked> tab Properties → Static website hosting → Enable
    Index document: index.html
    Error document: index.html  ← penting untuk SPA React Router
```
> Catat S3 website endpoint (http://monorepo-frontend-prod.s3-website-ap-southeast-1.amazonaws.com) → kirim ke Anggota D dan E.

## Fase 1 — Anggota A: AWS Budgets
Paralel dengan fase lain · Bisa selesai dalam 30 menit

### 1. Buat Monthly Cost Budget
Masuk ke `Billing and Cost Management → Budgets → Create budget`.

**Konfigurasi budget**
```bash
Setup: Customize (advanced)
Budget type: Cost budget
Budget name: MonorepoTeamBudget
Period: Monthly
Budget renewal type: Recurring
Start month: bulan ini
Budget amount: $10.00 (sesuaikan dengan estimasi tim)

Filters (opsional): bisa filter per service kalau mau spesifik
```

**Configure alert threshold**
```bash
Alert 1:
  Threshold: 50% of budgeted amount ($5.00)
  Trigger: Actual cost
  Email: [email seluruh tim] (separate pakai comma)

Alert 2:
  Threshold: 80% ($8.00)
  Trigger: Actual cost
  Email: [email seluruh tim]

Alert 3:
  Threshold: 100% ($10.00)
  Trigger: Actual + Forecasted
  Email: [email Admin + seluruh tim]
```
- ✅ Screenshot halaman Budgets ([*contoh](https://drive.google.com/file/d/1IAjwCWOW1uFctIwI5QPiU2cujQMAVGfP/view?usp=drive_link)) setelah selesai (untuk penilaian, lakukan di H-1 Pengumpulan)

### 2. Eksplorasi Cost Explorer untuk dokumentasi
Ini bagian dokumentasi/penilaian — tunjukkan kamu paham cara membaca cost.

```bash
Billing and Cost Management → Cost and Usage Analysis -> Coverage Report
```
Tugas anda adalah men-setting filter. Screenshoot pada bagian "Coverage" dan "Savings Plans coverage breakdown" yang menampilkan "On-Demand Spend" yang mencangkup tiap Service, Instance Family, ataupun Region. Semakin detail semakin baik.
- ✅ Setelah modifikasi filter, simpan sebagai report baru.
- ✅ Screenshoot ketika H-1 pengumpulan (supaya yang dikumpulkan adalah nilai paling update).
