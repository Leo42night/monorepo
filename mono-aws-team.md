# PPWL 11 - AWS Team
Instruction Team:
- Proyek ini dikerjakan sesuia Tim yang dibagikan.
- Ada total 6 Fase instruksi (untuk masing-masing anggota).
- Fase 5 & 6 Dapat dikerjakan 1 orang apabila anggota cuma 5.
Brief Project:
- Menggunakan fitur AWS RDS (PostgreSQL), AWS Budgets, AWS S3 Cloudfront atau AWS Lambda.
- 6 Fase/Role:
  1. AWS Admin: Root akses, Setup global (IAM, VPC, Systems Manager), Koordinator tim
  2. IAM Client A (AWS Budgets): Budget Management & Cost Explorer
  3. IAM Client B (Aurora / RDS): PostgreSQL Database layer 
  4. IAM Client C (Lambda — Backend): Lambda Function Elysia API serverless
  5. IAM Client D (Lambda — Frontend): React static via S3+CloudFront
  6. IAM Client E (Opsional, Integrasi & Dokumentasi): Jembatan semua komponen + laporan akhir
- Submisi mungkin akan menggunakan google docs agar rapi (karena ada segmen laporan).

📢 **Announcement**: 
- File ini akan di update bertahap, karena ada beberapa tahap yang perlu dirapikan (akan ada info updatenya rutin di grup WA).
- Untuk sekarang anda dapat langsung mengerjakan tahap yang sudah fix di bawah. 

⚠️ **Disclaimer**: tutorial ini di jalankan menggunakan WSL archLinux [wsl paling ringan], jadi mungkin akan ada kendala dependency & cara instalasi bagi yang environment berbeda.

## Install & Login

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

## Fase 1 — Admin: IAM, VPC, Parameter Store

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

**Custom Additional Inline Policy (add dari "User groups" atau dari "Policies")**
```bash
# AWS Budget butuh policy "ce:DescribeReport"
-> masuk ke group "grp-budget" -> Permissions -> Create Inline Policy
  Select a service "Cost Explorer Service" # di JSON codename nya "ce" (jadi anda perlu search arti dari code nya)
  Search allowed Actions -> "DescribeReport"
  Resource All (*)  
  # Lihat di JSON, kode action nya menjadi "ce:DescribeReport"
  # tabahkan juga action "ce:GetDimensionValues", "ce:GetSavingsPlansPurchaseRecommendation" and "ce:GetSavingsPlansCoverage"
  -> Name: additionalInlinePolicy_grpBudget

# AWS Lambda (fe & be) butuh butuh akses buat role & policy ketiak save lambda function.
-> Policies -> Create Policy
  Service: IAM -> Actions allow: CreateRole, CreatePolicy, AttachRolePolicy
  Resource All
  -> Name "additionalLambdaPolicy"
-> User Group -> "grp-lambda-be" -> Permissions -> Add Attach Policies
  -> tambahkan policy "additionalLambdaPolicy"
-> lakukan hal yang sama untuk User Group "grp-lambda-fe"

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
Gunakan default **VPC** (Virtual Private Cloud) jika ada, atau buat VPC baru. Yang penting: Security Group untuk RDS hanya terima koneksi dari Lambda.

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

**Buat Security untuk Postgres public**: di pakai sementara untuk migrate database dari local
```bash
Name: postgrePublic
Desc: Allow Local public Access to RDS PostgreSQL Database
  Inbound: PostgreSQL (5432) → Sources (Anywhere-IPv4)
  Outbound: semua traffic (default)
```

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

**Tambahkan policy baca Parameter Store ke Lambda role (untuk anggota C)**
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

## Fase 2 — Anggota A: AWS Budgets
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

## Fase 3 — Anggota B: RDS Database
Mulai setelah Admin selesai VPC & Security Group
> Tunggu konfirmasi dari Admin bahwa sgRdsInternal dan Parameter Store sudah siap.

### 1. Buat RDS
Buat RDS PostgreSQL Free Tier (lebih aman dari sisi biaya).

**RDS PostgreSQL (Free Tier)**
```bash
Region "us-east-1"
Aurora and RDS → Database → Create database (FUll Configuration)
  Engine: PostgreSQL 17
  Database Creation Method: Full Configuration
  Template: Sandbox  ← PENTING untuk Free Tier single-AZ
  DB instance identifier: monorepo-db
  Master username: postgres
  Master password: (simpan baik-baik)
  DB instance class: db.t3.micro
  Storage: 20 GiB gp2
  
  Connectivity:
    VPC: (pilih VPC dari Admin)
    Subnet group: default
    VPC security group: 
      sgRdsInternal (dari Admin)
      postgrePublic (sementara agar dapat migrate dari local. setelah migrasi, hapus seleksi)
    Additional configuration:
      Public access: Yes (sementara) ← setelah migrasi, jadikan No agar hanya dapat diakses dari Lambda
  
  Additional configuration
    Initial database name: monorepo_prod
```

**Cara cek endpoint RDS** (salin endpoint tersebut)
```bash
RDS → Databases → monorepo-db → Connectivity & security
→ Endpoint: monorepo-db.xxxxxxxxx.ap-southeast-1.rds.amazonaws.com
→ Port: 5432
```

### 2. Jalankan migrate ke DB baru

**Setup database local**
```bash
cd apps/backend

# Jika bun belum ter-install di temrinal, run "curl -fsSL https://bun.com/install | bash"
# Deploy migrasi ke local, dia akan membuat file "deb.db" jika tidak ada)
bunx prisma migrate deploy
# Seed (akan diminta menambahkan config seed: "bun run prisma/seed.ts")
bunx prisma db seed

# Ikuti Tutorial di "Aurora and RDS" -> "Databases" -> "monorepo-db" -> Connectivity & Security
# download global key (karena AWS RDS default nya wajib koneksi SSL terenkripsi)
curl -o global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

**Kirim data dari SQLite Local ke RDS PostgreSQL**
```bash
# copy database (create & insert) ke file sql
cd apps/backend
sqlite3 dev.db .dump > data.sql
# ❌ Hapus / ubah:
#     - PRAGMA
#     - BEGIN TRANSACTION
#     - COMMIT
#     - AUTOINCREMENT → ganti jadi SERIAL
#     - INTEGER PRIMARY KEY → jadi SERIAL PRIMARY KEY
#! Atau Minta LLM untuk konversi ke format PostgreSQL

# --- Cara 1: HeidiSQL ---
# Gunakan HeidiSQL untuk koneksi (untuk PhpMyAdmin kurleb config nya sama)
"New" Session
  -> Tab "SSL", isi SSL CA certificate dengan path file 'global-bundle.pem'
  -> Tab Settings
    Network Type: PostgreSQL (TCP/IP)
    Library: libpq-12.dll (atau sejenis)
    Hostname: ENDPOINT (cth: monorepo-db.c8nscaw0oxxx.us-east-1.rds.amazonaws.com)
    -> User, Password, Port isi sesuai setingan anda.
  -> Rename Sessions: "AWS RDS"

-> Buka Session -> Database "Public"
  -> Jalankan Query PostgreSQL


# --- Cara 2: psql CLI ---
# Install postgesql, contoh di archLinux (65mb)
sudo pacman -S postgresql
psql --version
# cth output: psql (PostgreSQL) 18.3

export RDSHOST="monorepo-db.c8nscaw0oxxx.us-east-1.rds.amazonaws.com" 
psql "host=$RDSHOST port=5432 dbname=monorepo_prod user=postgres sslmode=verify-full sslrootcert=./global-bundle.pem"
## cth masuk ke terminal: SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
## Tools: jika ingin keluar dari terminal postgres, run "\q" 

# Setelah data.sql jadi format PostgreSQL, dump di terminal postgres
\i data.sql
## cth output:
##   monorepo_prod=> \i data.sql
##   CREATE TABLE
##   INSERT 0 3

# Periksa apakah data sudah ada
SELECT * FROM "User";
```

⚠️ Setelah migrasi, Modify AWS Database "monorepo-db". 
- Hapus seleksi security group "postgrePublic".
- Jadikan Public Access "No". 
Dengan ini, database hanya dapat diakses dari Lambda.

**Setelah selesai, update Parameter Store**
```bash
Minta Admin update parameter /monorepo/DATABASE_URL dengan value:
postgresql://postgres:PASSWORD@ENDPOINT:5432/monorepo_prod

JANGAN kirim password via chat terbuka — gunakan DM atau minta Admin input langsung.
```

- ✅ Screenshot RDS console (status Available) untuk penilaian
- ✅ Kabari Anggota C bahwa DATABASE_URL sudah di Parameter Store

# Fase 4 — Anggota C: Lambda Backend (Elysia)
Mulai setelah Admin selesai Parameter Store, dan Anggota B update DATABASE_URL
> Tunggu Anggota B kabari bahwa DATABASE_URL sudah di Parameter Store.

## 1. Build backend Elysia menjadi bundle Lambda
Lambda Node.js butuh handler function sebagai entry point. Elysia sudah export app, kita bungkus dengan adapter.

**Beberapa modifikas file:**

### a.1. prisma/schema-postgres.prisma
Buat skema khusus postgres:
```sh
generator client {
  provider = "prisma-client-js"
  output   = "../src/generated/prisma"
  engineType = "client"
}

datasource db {
  provider = "postgresql"
}

model User { 
  id    Int     @id @default(autoincrement()) 
  email String  @unique
  name  String?
} 
```
### a.2. prisma/dbPostgre.ts
tambahkan inisiasi db khusus RDS Postgres
```ts
// AWS Lambda tidak bisa langsung menggunakan file SQLite, jadi kita buat file baru khusus untuk PostgreSQL yang akan digunakan di Lambda. 
// File ini akan tetap menggunakan Prisma Client, tapi dengan konfigurasi yang sesuai untuk PostgreSQL.
import { PrismaClient } from "../src/generated/prisma-pg/client";
import { PrismaPg } from "@prisma/adapter-pg";

export const prisma = new PrismaClient({
  adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL })
});
```
### a.3. src/index.ts
Tambahkan modifikasi untuk Load Env, Adapter Lambda & Postgre DB
```ts
// 1 — Ganti import { prisma } from "../prisma/db"; jadi:
import { prisma } from "../prisma/dbPostgre";

// 2.a. — Implement dynamic loader dari SSM (bagian atas sebelum app dipakai)
import { SSMClient, GetParametersCommand } from "@aws-sdk/client-ssm";

const ssm = new SSMClient({ region: "us-east-1" });

const loadSSMParameters = async () => {
  const params = [
    "/monorepo/DATABASE_URL",
    "/monorepo/DB_AUTH_TOKEN",
    "/monorepo/GOOGLE_CLIENT_ID",
    "/monorepo/GOOGLE_CLIENT_SECRET",
    "/monorepo/GOOGLE_REDIRECT_URI",
    "/monorepo/SESSION_SECRET",
    "/monorepo/API_KEY",
    "/monorepo/FRONTEND_URL"
  ];

  const command = new GetParametersCommand({
    Names: params,
    WithDecryption: true,
  });

  const response = await ssm.send(command);

  response.Parameters?.forEach((param: any) => {
    if (!param.Name || !param.Value) return;

    const key = param.Name.split("/").pop(); // ambil nama terakhir
    process.env[key!] = param.Value;
  });
};

// 2.b. — Panggil sebelum app digunakan. Karena Lambda stateless, kita pakai lazy init + caching
let isLoaded = false;

export const initConfig = async () => {
  if (!isLoaded) {
    await loadSSMParameters();
    isLoaded = true;
  }
};

// 3.a. — Pisahkan app builder, bungkus app dalam "createApp"
export const createApp = () => {
  return new Elysia()
    .use(
      cors({
        origin: process.env.FRONTEND_URL || "http://localhost:5173",
        credentials: true,
      }),
    )
    ...
    .get("/users", async () => {
      const users = await prisma.user.findMany();
      const response: ApiResponse<User[]> = {
        data: users,
        message: "User list retrieved",
      };
      return response;
    })
    // isi kode lainnya
};

// 3.b. — Jangan eksport langsung
// ❌ export default app;
export default createApp;

// 4 - Hapus Kode untuk development (app.listen & console.log) & kode lain yang tidak kompatibel dengan struktur sekarang. anda dapat membuat file terpisah untuk development.
```
### a.4. src/lambda.ts
**Tambahkan Lambda handler di **
```ts
// Kode ini adalah adapter yang menghubungkan request dari AWS Lambda (API Gateway) ke aplikasi web kamu (Elysia / Fetch API style).
import createApp, { initConfig } from "./index";

let app: any;

export const handler = async (event: any) => {
  await initConfig(); // 🔥 load SSM dulu

  // 👁️ DEBUG Semetara untuk cek apakah env sudah terbaca dengan benar
  // Periksa di Cloudwatch Logs. Hapus ketika sudah berhasil.
  console.log("ENV DATABASE_URL:", process.env.DATABASE_URL);

  if (!app) {
    app = createApp(); // 🔥 baru buat app setelah env ready
  }

  return app.handle(new Request(
    `https://${event.headers.host}${event.rawPath}${event.rawQueryString ? '?' + event.rawQueryString : ''}`,
    {
      method: event.requestContext.http.method,
      headers: event.headers,
      body: event.body
        ? Buffer.from(event.body, event.isBase64Encoded ? 'base64' : 'utf8')
        : undefined
    }
  )).then(async (res: any) => ({
    statusCode: res.status,
    headers: Object.fromEntries(res.headers),
    body: await res.text(),
    isBase64Encoded: false
  }));
};
```

### b. Install, Generate & Build

```bash
cd apps/backend
# Install bebrapa dependency baru
bun add @aws-sdk/client-ssm
bun add @prisma/adapter-pg

# generate client menggunakan schema-postgres.prisma
bunx prisma generate --schema prisma/schema-postgres.prisma

# build seluruh kode & dependency nya di 1 file (tapi pisahkan kode dari prisma)
## [?] Menggunakan --target node karena kita pakai runtime "Node", bukan "Bun"
## [?] --format cjs, Common JS. mengganti ESM 'import.meta', jadi CJS 'require'
bun build src/lambda.ts --outdir dist-lambda --target node --format cjs --external prisma
# copy Generated Prisma Client (postgres) & dependency
cp -r src/generated/prisma-pg dist-lambda/generated/prisma-pg
cp -r node_modules/.prisma dist-lambda/node_modules/.prisma 2>/dev/null || true

# ZIP untuk upload (38MB -> 3.8MB) (install zip, cth di archLinux: `pacmap -S zip`)
cd dist-lambda && zip -r ../lambda-backend.zip . && cd ..
```

## 2. Buat Lambda function di AWS Console
Upload ZIP dan konfigurasi env vars dari Parameter Store. 
```bash
Buka Aws Console -> Select region "us-east-1 (N. Virginia)"
Lambda → Create function → Author from scratch
  Function name: monorepo-backend
  Runtime: Node.js ^24.x  (Latest support, atau pilih Amazon Linux jika ingin "custom" pakai bun layer)
  Architecture: x86_64
  
  Execution role: "Create new role" with basic Lambda permissions
  → setelah dibuat, attach policy SSM read (dari Admin)
```

Minta admin tambahkan policy ke Role yang baru di buat (biasanya namanya **monorepo-backend-role-xxx**), supaya Lambda Function dapat akses env vars di SSM:
```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "LambdaAccessSSMKey",
			"Effect": "Allow",
			"Action": [
				"ssm:GetParameters",
				"ssm:GetParameter",
				"kms:Decrypt"
			],
			"Resource": [
				"arn:aws:ssm:us-east-1:AWS_ACCOUNT_ID:parameter/monorepo/*"
			]
		}
	]
}
```
**✨ Tips**: gunakan fitur search resource biar mudah

**Upload ZIP**
```bash
Lambda → Functions -> Masuk ke fungsi yang baru dibuat
  -> tab "Code" → Upload from → .zip file → pilih lambda-backend.zip
    -> Runtime Settings -> Edit
      Handler: lambda.handler
  -> tab "Configuration" -> Edit
      Memory: 512 MB (minimum untuk prisma)
      Timeout: 1 menit (default 3 detik terlalu kecil untuk cold start Prisma)
```

**Set environment variables dari Parameter Store**
```bash
Lambda → Configuration → Environment variables:
  NODE_ENV = production
  
  # Untuk secret mengunakan SSM parameter store reference, BUKAN plaintext di sini
  # dynamic load dari SSM sudah di set di index.ts
```

**Buat Lambda Function URL**
```bash
Lambda → Functions -> Masuk ke fungsi yang baru dibuat
  → tab Configuration → Function URL → Create function URL
    Auth type: NONE  (kita pakai API_KEY manual dari kode Elysia)
    CORS: Enable
      Allow origins: * (sementara, nanti ubah ke S3/CloudFront URL)
      Allow headers: Content-Type, Authorization
      Allow methods: *
      Allow credentials: true

→ Salin Function URL yang muncul (format: https://xxxxxxxx.lambda-url.ap-southeast-1.on.aws)
→ Kirim URL ini ke Anggota D dan Admin
```

- ✅ Redirect URI didapatkan: "https://FUNCTION_URL/auth/callback"
  - Update Admin: minta update `/monorepo/GOOGLE_REDIRECT_URI` & tambahkan `GOOGLE_REDIRECT_URI` ke Google Credential Allowed Redirect URI. 
- ✅ Test (log):
  - Cara 1: run `aws logs tail /aws/lambda/monorepo-backend --follow` (run dulu `aws login --remote`)
  - Cara 2: CloudWatch -> Logs Insights -> search "/aws/lambda/monorepo-backend" -> Run query (jika ter block "..is not authorized to perform", salin policy actions yang dibutuhkan dan minta admin menambahkannye ke akses ke User Group `grp-lambda-be`)
- 📢 Jika log kosong, coba akses url dulu, log akan di trigger ulang (terutama jika pakai CLI `aws logs ..`)
- ✅ Test: curl https://FUNCTION_URL → harus dapat response dari Elysia
- ✅ Test: curl https://FUNCTION_URL/auth/login → harus redirect ke Google
- ⚠️ hapus Debug `console.log("ENV DATABASE_URL...` setelah server berhasil running

Fase 5 - 6 on going
