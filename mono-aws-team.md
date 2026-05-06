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

⚠️ **Disclaimer**: tutorial ini di jalankan menggunakan WSL archLinux [wsl paling ringan], jadi mungkin akan ada kendala dependency & cara instalasi bagi yang environment berbeda.

✨ **Tips:** 
- Gunakan `US-Virginia (us-east-1)` jika ingin cost paling murah, jadi credit tidak terkuras banyak.
- Tiap pindah fitur AWS Console, Buka tab baru, agar mudah kembali ke tab sebelumya. 
- Gunakan Notepad untuk menyimpan key/config yang akan digunakan lagi.

## Install & Login

**AWS**
<details><summary>Step Instalasi AWS </summary>

Ikuti [Docs setup AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions)

```sh
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
</details>

## Fase 1 — Admin: IAM, VPC, Parameter Store
Admin perlu handle issue aktivasi akun. Coba test dengan Create Cloudfront Distribution. Jika dapat error `You Account Must be Verified before you can add new Cloudfront resources`, lakukan [Aktivasi Akun untuk akses Cloudfront](https://github.com/Leo42night/monorepo/issues/4). Proses ini untuk dapat melakukan [Setup CloudFront untuk HTTPS](#setup-cloudfront-untuk-https)

### 1. Buat IAM Group dan User untuk tiap anggota
Masuk ke AWS Console → IAM → Groups → Create group. Buat 4 group sesuai tugas anggota.

#### **Gunakan policy type "AWS Managed"**
<details><summary>Step Add User Group</summary>

```sh
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
</details>

**Custom Additional Inline Policy**
<details><summary>Step Add Inline Policy</summary>

Add dari "User groups" (jika hanya untuk 1 Group) atau dari "Policies" (Jika dipakai 2 atau lebih Group)
```sh
# AWS Budget butuh policy "ce:DescribeReport"
-> masuk ke group "grp-budget" -> Permissions -> Create Inline Policy
  Select a service "Cost Explorer Service" # di JSON codename nya "ce" (jadi anda perlu search arti dari code nya)
  Search allowed Actions -> "DescribeReport"
  Resource All (*)  
  # Lihat di JSON, kode action nya menjadi "ce:DescribeReport"
  # tabahkan juga action "ce:GetDimensionValues", "ce:GetSavingsPlansPurchaseRecommendation", "ce:GetSavingsPlansCoverage", "ce:DeleteReport", "ce:CreateReport"
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
</details>

**✨ Tips:** Admin bisa cek login ke akun IAM, dengan buka di Incognito Mode, jadi session login lain tidak terpengaruh. 

#### **Buat IAM User**
<details><summary>Step add IAM User</summary>

Lakukan untuk tiap anggota, sesuaikan Group nya.
```sh
IAM → Users → Create user
  Username: database-anggota-b (atau nama asli, kasih job ke nama nya biar jelas role nya)
  Access type: AWS Management Console access
  Console password: custom / auto-generated
  Assign to group: grp-database
  → Download credentials.csv → kirim ke anggota via chat
# Tambahkan user dengan name "asdos", beri policy "AdministratorAccess", kirim csv lewat WA ke Asdos-Leo.
```
</details>

> [!NOTE]
> Kalau tim hanya 5 orang, tugaskan Client E (Integrasi) ke Client D, karena keduanya paling erat berkaitan (build frontend + verifikasi end-to-end).

**Penting:** Admin Root perlu atkifkan `IAM user and role access to Billing information`. Menunya ada di [account setting](https://drive.google.com/file/d/1OGMQqj2ZUSZfcV5p0LDXJlUVU_Uyi0St/view?usp=drive_link).

### 2. Setup VPC dan Security Group
Gunakan default **VPC** (Virtual Private Cloud) jika ada, atau buat VPC baru. Yang penting: Security Group untuk RDS hanya terima koneksi dari Lambda.

**Buat Security Group untuk RDS**
<details><summary>Step SG RDS Internal </summary>

```sh
VPC → Security Groups → Create security group
  Name: sgRdsInternal
  Description: RDS Security Group for internal PostgreSQL access from within VPC
  VPC: (pilih VPC kamu, atau biarkan default)
  
  Inbound rules:
    Type: PostgreSQL (5432)
    Source: Custom → sgLambda (akan diisi setelah Lambda SG dibuat, untuk sekarang skip dulu)
  
  Outbound rules: semua traffic (default)
```
</details>

**Buat Security Group untuk Lambda**
<details><summary>Step SG Lambda</summary>

```sh
Name: sgLambda
Desc: Security group for Lambda functions to access RDS and external APIs
  Inbound: tidak perlu (Lambda dipanggil via URL, bukan VPC ingress)
  Outbound:
    Type: PostgreSQL (5432) → Destination: sgRdsInternal
    Type: HTTPS (443) → Destination: Anywhere-IPv4 (untuk Turso/LibSQL)
```
</details>

> [!NOTE] 
> Setelah `sgLambda` terbentuk, balik ke sgRdsInternal dan edit inbound rule: ubah source dari "Custom" ke sgLambda ID.

**Buat Security untuk Postgres public**
<details><summary>Step SG Postgre Public</summary>

Dipakai untuk migrate database dari local, dan dapat di akses.
```sh
Name: postgrePublic
Desc: Allow Local public Access to RDS PostgreSQL Database
  Inbound: PostgreSQL (5432) → Sources (Anywhere-IPv4)
  Outbound: semua traffic (default)
```
</details>

### 3. Simpan semua env vars ke Parameter Store
Jangan pernah hardcode secret di Lambda env vars. Simpan di Systems Manager Parameter Store, tipe SecureString.

**Buat parameter (tiap baris = satu parameter).**
<details><summary>Step env vars di System Manager</summary>

```sh
AWS Systems Manager → Parameter Store → Create parameter

/monorepo/GOOGLE_CLIENT_ID         → String
/monorepo/GOOGLE_CLIENT_SECRET     → SecureString
/monorepo/GOOGLE_REDIRECT_URI      → String  (isi nanti setelah Lambda URL diketahui)
/monorepo/JWT_SECRET               → SecureString (isi bebas, auntentikasi pengganti session cookie)
/monorepo/DATABASE_URL             → SecureString  (isi setelah Anggota B selesai RDS)
/monorepo/DB_AUTH_TOKEN            → SecureString
/monorepo/API_KEY                  → SecureString
/monorepo/FRONTEND_URL             → String  (isi nanti setelah S3/CloudFront URL diketahui)
```
</details>

- ✅ Screenshot halaman Parameter Store ([*contoh](https://drive.google.com/file/d/1llPkFZ8MGDNJrmHzxW1l7pFC_UzpmDBk/view?usp=drive_link))

**Tambahkan policy baca Parameter Store ke Lambda role (untuk anggota C)**
<details><summary>Step Additional Policy Lambda Role</summary>

```sh
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
</details>

- ✅ Bagikan nama parameter path (/monorepo/...) ke Anggota C
- ✅ Jangan bagikan nilai secret-nya langsung — anggota C cukup tahu nama key-nya

### 4. Buat S3 bucket untuk frontend (persiapan Anggota D)
Admin buat bucket sekarang supaya Anggota D bisa langsung upload nanti.

<details><summary>Step Create S3 Bucked for Frontend</summary>

```sh
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
</details>

> [!NOTE]
> Catat S3 website endpoint (`http://monorepo-frontend-prod.s3-website-us-east-1.amazonaws.com`) → kirim ke Anggota D dan E.

## Fase 2 — Anggota A: AWS Budgets
Paralel dengan fase lain · Bisa selesai dalam 30 menit

### 1. Buat Monthly Cost Budget
Masuk ke `Billing and Cost Management → Budgets → Create budget`.

**Konfigurasi budget**
<details><summary>Step add Budget</summary>

```sh
Setup: Customize (advanced)
Budget type: Cost budget
Budget name: MonorepoTeamBudget
Period: Monthly
Budget renewal type: Recurring
Start month: bulan ini
Budget amount: $10.00 (sesuaikan dengan estimasi tim)

Filters (opsional): bisa filter per service kalau mau spesifik
```
</details>

**Configure alert threshold**
<details><summary>Step add Alert Treshold</summary>

```sh
Alert 1:
  Threshold: 50% of budgeted amount ($5.00)
  Trigger: Actual cost
  Email: [email seluruh tim] (separate pakai comma)

Alert 2:
  Threshold: 80% ($8.00)
  Trigger: Actual cost
  Email: [email seluruh tim]

Alert 3 & 4:
  Threshold: 100% ($10.00)
  Trigger: Actual & Forecast
  Email: [email Admin + seluruh tim]
```
</details>

> ✅ Screenshot halaman Budgets ([*contoh](https://drive.google.com/file/d/1IAjwCWOW1uFctIwI5QPiU2cujQMAVGfP/view?usp=drive_link)) setelah selesai (untuk penilaian, lakukan di H-1 Pengumpulan)

### 2. Cost Explorer - Tugas Khusus (task-cost-report)
Ini bagian dokumentasi/penilaian — tunjukkan kamu paham cara membaca cost.

```sh
Billing and Cost Management -> Cost Explorer Saved Reports (Pilih "Coverage Report")
```
Tugas anda adalah men-setting filter. Screenshoot mencangkup bagian filter yang digunakan dan bagian "Savings Plans coverage breakdown" yang menampilkan "On-Demand Spend" yang mencangkup tiap Service, Instance Family, ataupun Region (seperti [*berikut](https://drive.google.com/file/d/1t3jeeoIKDKsfnut2R9PbsVqOEu1e8fh_/view?usp=drive_link), filter disembunyikan karena tugas anda untuk mencarinya). Semakin detail semakin baik.

- ✅ "Save as a new report" agar budget report dapat di pakai kembali. 
- ✅ Screenshoot ketika H-1 pengumpulan (supaya yang dikumpulkan adalah nilai paling update).

## Fase 3 — Anggota B: RDS Database
Mulai setelah Admin selesai VPC & Security Group
> Tunggu konfirmasi dari Admin bahwa sgRdsInternal dan Parameter Store sudah siap.

### 1. Buat RDS
Buat RDS PostgreSQL Free Tier (lebih aman dari sisi biaya).

**RDS PostgreSQL (Free Tier)**
<details><summary>Step add Databse</summary>

```sh
Region "us-east-1"
Aurora and RDS → Database → Create database (FUll Configuration)
  Engine: PostgreSQL 17
  Database Creation Method: Full Configuration
  Template: Sandbox  ← PENTING untuk Free Tier single-AZ
  DB instance identifier: monorepo-db
  Master username: postgres
  Master password: (simpan baik-baik, jangan pakai simbol: !$"'")
  DB instance class: db.t3.micro
  Storage: 20 GiB gp2
  
  Connectivity:
    VPC: (pilih VPC dari Admin)
    Subnet group: default
    VPC security group: 
      sgRdsInternal (dari Admin)
      postgrePublic (agar dapat migrate dari local)
    Additional configuration:
      Public access: Yes (dapat diakses di Local & Lambda)
  
  Additional configuration
    Initial database name: monorepo_prod
```
</details>

**Cara cek endpoint RDS** (salin endpoint tersebut)
```sh
RDS → Databases → monorepo-db → Connectivity & security
→ Endpoint: monorepo-db.xxxxxxxxx.us-east-1.rds.amazonaws.com
→ Port: 5432
```

### 2. Jalankan migrate ke DB baru

**Setup database local**

<details><summary>Step Migrate using HeidiSQL or psql CLI</summary>

```sh
cd apps/backend

# Jika bun belum ter-install di temrinal, run "curl -fsSL https://bun.com/install | bash"
# Deploy migrasi ke local, dia akan membuat file "deb.db" jika tidak ada)
bunx prisma migrate deploy
# Seed (akan diminta menambahkan config seed: "bun run prisma/seed.ts")
bunx prisma db seed

# Ikuti Tutorial di "Aurora and RDS" -> "Databases" -> "monorepo-db" -> Connectivity & Security
# download global key (karena AWS RDS default nya wajib koneksi SSL terenkripsi)
mkdir -p cert && curl -o cert/global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

**Kirim data dari SQLite Local ke RDS PostgreSQL**
```sh
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
psql "host=$RDSHOST port=5432 dbname=monorepo_prod user=postgres sslmode=verify-full sslrootcert=./cert/global-bundle.pem"
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
</details>

**Setelah selesai, update Parameter Store**
```sh
Minta Admin update parameter /monorepo/DATABASE_URL dengan value:
postgresql://postgres:PASSWORD@ENDPOINT:5432/monorepo_prod

JANGAN kirim password via chat terbuka — gunakan DM atau minta Admin input langsung.
```

- ✅ Screenshot RDS console (status Available) untuk penilaian ([*contoh](https://drive.google.com/file/d/1wSuy_LKIER0q7TSIzBcSq2X6nfUrb9yA/view?usp=drive_link))
- ✅ Kabari Anggota C bahwa `DATABASE_URL` sudah di Parameter Store

## Fase 4 — Anggota C: Lambda Backend (Elysia)
Mulai setelah Admin selesai Parameter Store, dan Anggota B update DATABASE_URL
> Tunggu Anggota B kabari bahwa DATABASE_URL sudah di Parameter Store.

### 1. Build backend Elysia menjadi bundle Lambda
Lambda Node.js butuh handler function sebagai entry point. Elysia sudah export app, kita bungkus dengan adapter.

**a. Beberapa modifikas file:**
#### 1.a. prisma/schema-postgres.prisma
<details><summary>Skema khusus postgres</summary>

```sh
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma-pg"
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
</details>

#### 1.b. prisma/db.ts
<details><summary>Handler Lazy Load & export</summary>

```ts
import { PrismaClient } from "../src/generated/prisma/client";
import { PrismaLibSql } from "@prisma/adapter-libsql";
import path from "path";

export const dbUrl = process.env.DATABASE_URL || `file:${path.resolve(__dirname, "../dev.db")}`;

const adapter = new PrismaLibSql({ url: dbUrl, authToken: process.env.DB_AUTH_TOKEN });

let prisma: PrismaClient;

export const getPrisma = () => {
  if (!prisma) {
    prisma = new PrismaClient({ adapter });
  }
  return prisma;
};
```
</details>

#### 1.c. prisma/dbPostgre.ts
<details><summary>Inisiasi db khusus RDS Postgres</summary>

```ts
// AWS Lambda tidak bisa langsung menggunakan file SQLite, jadi kita buat file baru khusus untuk PostgreSQL yang akan digunakan di Lambda. 
// File ini akan tetap menggunakan Prisma Client dengan skema PostgreSQL.
import { PrismaClient } from "../src/generated/prisma-pg/client";
import { PrismaPg } from "@prisma/adapter-pg";
import fs from "fs";
import path from "path";

const ca = fs.readFileSync(
  path.join(process.cwd(), "cert/global-bundle.pem")
).toString();

// Kita buat singleton Prisma Client agar dipanggil ketika SSM sudah siap, dan tidak dibuat ulang setiap kali handler dipanggil (karena Lambda bisa reuse container).
let prisma: PrismaClient;

export const getPrisma = () => {
  if (!prisma) {
    prisma = new PrismaClient({
      adapter: new PrismaPg({
        connectionString: process.env.DATABASE_URL!,
        ssl: {
          ca, // lokasi file sertifikat SSL untuk RDS, ketika build akan relatif ke folder /apps/backend/dist-lambda/
          rejectUnauthorized: true,
        }
      }),
    });
  }

  return prisma;
};
```
</details>

#### 1.d. src/config.ts
<details><summary>Berisi SSM loader</summary>

```ts
import { SSMClient, GetParametersCommand } from "@aws-sdk/client-ssm";

const ssm = new SSMClient({ region: "us-east-1" });

const SSM_PARAMS = [
  "/monorepo/DATABASE_URL",
  "/monorepo/DB_AUTH_TOKEN",
  "/monorepo/GOOGLE_CLIENT_ID",
  "/monorepo/GOOGLE_CLIENT_SECRET",
  "/monorepo/GOOGLE_REDIRECT_URI",
  "/monorepo/JWT_SECRET",
  "/monorepo/API_KEY",
  "/monorepo/FRONTEND_URL",
];

let isLoaded = false;

export const loadConfig = async () => {
  if (isLoaded) return;

  const command = new GetParametersCommand({
    Names: SSM_PARAMS,
    WithDecryption: true,
  });

  const response = await ssm.send(command);

  response.Parameters?.forEach((param) => {
    if (!param.Name || !param.Value) return;
    const key = param.Name.split("/").pop()!;
    process.env[key] = param.Value;
  });

  isLoaded = true;
};
```
</details>

#### 1.e. src/types.ts
<details><summary>Berisi custom types Prisma Client untuk security</summary>

```ts
export interface DbClient {
  user: {
    findMany: () => Promise<any[]>;
  };
  // tambah model & method lain sesuai kebutuhan
}
```
</details>

#### 1.f. src/index.ts
<details><summary>Berisi shared app factory</summary>

Hanya routes & middleware, terima `getPrisma` via DI
```ts
import { Elysia } from "elysia";
import { cookie } from "@elysiajs/cookie";
import { jwt } from "@elysiajs/jwt";
import { createOAuthClient, getAuthUrl } from "./auth";
import { getCourses, getCourseWorks, getSubmissions } from "./classroom";
import type { ApiResponse, HealthCheck, User } from "shared";
import type { DbClient } from "./types";

// Auth middleware — reusable di semua route yang butuh autentikasi
const makeAuthMiddleware = (jwtInstance: any) =>
  async ({ headers, set }: any) => {
    const authHeader = headers.authorization;
    if (!authHeader) {
      set.status = 401;
      return null;
    }

    const token = authHeader.replace("Bearer ", "");
    const payload = await jwtInstance.verify(token);

    if (!payload) {
      set.status = 401;
      return null;
    }

    return payload;
  };

// Factory menerima `getPrisma` sebagai dependency injection
// sehingga dev pakai LibSQL, prod pakai PostgreSQL — tanpa mengubah routes
export const createApp = (getPrisma: () => DbClient) => {
  const app = new Elysia()
    .use(cookie())
    .use(
      jwt({
        name: "jwt",
        secret: process.env.JWT_SECRET!,
        exp: "1d",
      })
    )

    // Middleware akses kontrol untuk /users
    .onRequest(({ request, set }) => {
      const url = new URL(request.url);
      console.log(`[DEBUG] [${request.method}] ${url.pathname}`);

      // Lewati preflight OPTIONS
      if (request.method === "OPTIONS") return;

      if (!url.pathname.startsWith("/users")) return;

      const origin = request.headers.get("origin");
      const frontendUrl = process.env.FRONTEND_URL ?? "http://localhost:5173";
      const key = url.searchParams.get("key");

      if (origin === frontendUrl) return;

      if (key !== process.env.API_KEY) {
        set.status = 401;
        return { message: "Unauthorized: Access denied without valid API Key" };
      }
    })

    // Health check
    .get("/", (): ApiResponse<HealthCheck> => ({
      data: { status: "ok" },
      message: "server running",
    }))

    // Users
    .get("/users", async () => {
      const users = await getPrisma().user.findMany();
      const response: ApiResponse<User[]> = {
        data: users,
        message: "User list retrieved",
      };
      return response;
    })

    // Auth — redirect ke Google login
    .get("/auth/login", ({ redirect }) => {
      const oauth2Client = createOAuthClient();
      const url = getAuthUrl(oauth2Client);
      return redirect(url);
    })

    // Auth — Google OAuth callback
    .get("/auth/callback", async ({ query, jwt, redirect }) => {
      const { code } = query as any;
      const oauth2Client = createOAuthClient();
      const { tokens } = await oauth2Client.getToken(code);

      const token = await jwt.sign({
        access_token: tokens.access_token,
        refresh_token: tokens.refresh_token,
      });

      return redirect(`${process.env.FRONTEND_URL}/classroom?token=${token}`);
    })

    // Auth — cek sesi user dari JWT
    .get("/auth/me", async ({ headers, jwt, set }) => {
      const auth = makeAuthMiddleware(jwt);
      const user = await auth({ headers, set });
      if (!user) return { loggedIn: false };
      return { loggedIn: true, user };
    })

    // Classroom — daftar courses
    .get("/classroom/courses", async ({ headers, jwt, set }) => {
      const auth = makeAuthMiddleware(jwt);
      const user = await auth({ headers, set });
      if (!user) return;

      const courses = await getCourses(user.access_token);
      return { data: courses };
    })

    // Classroom — submissions per course
    .get("/classroom/courses/:courseId/submissions", async ({ params, headers, jwt, set }) => {
      const auth = makeAuthMiddleware(jwt);
      const user = await auth({ headers, set });
      if (!user) return;

      const { courseId } = params;
      const [courseWorks, submissions] = await Promise.all([
        getCourseWorks(user.access_token, courseId),
        getSubmissions(user.access_token, courseId),
      ]);

      return {
        data: courseWorks.map((cw) => ({
          courseWork: cw,
          submission: submissions.find((s) => s.courseWorkId === cw.id) ?? null,
        })),
      };
    });

  return app;
};
```
</details>

#### 1.g. src/server.ts
<details><summary>dev entry point</summary>

Import `prisma` dari `prisma/db` (LibSQL). Tampilkan detail log.
```ts
import { createApp } from "./index";
import { getPrisma, dbUrl } from "../prisma/db"; // LibSQL
import cors from "@elysiajs/cors";

const app = createApp(getPrisma);

app.use(cors({
  origin: "*",
  allowedHeaders: ["Content-Type", "Authorization"],
}))
.listen(3000);

console.log("🦊 Backend    → http://localhost:3000");
console.log("🦊 FRONTEND_URL →", process.env.FRONTEND_URL);
console.log("🦊 DATABASE_URL →", dbUrl);
console.log("🦊 REDIRECT_URI →", process.env.GOOGLE_REDIRECT_URI);
```
</details>

#### 1.h. src/lambda.ts
<details><summary>prod entry point</summary>

```ts
import { createApp } from "./index";
import { loadConfig } from "./config";       // SSM loader
import { getPrisma } from "../prisma/dbPostgres"; // PostgreSQL

let app: ReturnType<typeof createApp>;

export const handler = async (event: any) => {
  // DEBUG: log seluruh event untuk lihat apakah OPTIONS masuk
  console.log("[EVENT] method:", event.requestContext?.http?.method);
  console.log("[EVENT] path:", event.rawPath);
  console.log("[EVENT] headers:", JSON.stringify(event.headers));

  await loadConfig(); // load SSM sekali, lalu di-cache

  if (!app) {
    app = createApp(getPrisma); // buat app setelah env ready
  }

  // DEBUG ENV
  console.log("[DATABASE_URL]:", process.env.DATABASE_URL);
  console.log("[FRONTEND_URL] env:", process.env.FRONTEND_URL);
  console.log("[API_KEY] env:", process.env.API_KEY);
  console.log("[JWT_SECRET] env:", process.env.JWT_SECRET);

  const frontendUrl = process.env.FRONTEND_URL ?? "http://localhost:5173";

  // Handle preflight OPTIONS langsung di handler — sebelum masuk Elysia
  // Lambda URL CORS config tidak reliable, jadi kita handle manual
  if (event.requestContext.http.method === "OPTIONS") {
    console.log("[OPTIONS] preflight handled for:", event.rawPath);
    return {
      statusCode: 204,
      headers: {
        "Access-Control-Allow-Origin": frontendUrl,
        "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
        "Access-Control-Allow-Headers": "Content-Type, Authorization",
        "Access-Control-Allow-Credentials": "true",
        "Access-Control-Max-Age": "86400",
      },
      body: "",
    };
  }

  const url = `https://${event.headers.host}${event.rawPath}${event.rawQueryString ? "?" + event.rawQueryString : ""
    }`;

  const response = await app.handle(
    new Request(url, {
      method: event.requestContext.http.method,
      headers: event.headers,
      body: event.body
        ? Buffer.from(event.body, event.isBase64Encoded ? "base64" : "utf8")
        : undefined,
    })
  );

  // Inject CORS headers ke semua response dari Elysia
  const resHeaders = Object.fromEntries(response.headers);

  // DEBUG — log headers sebelum inject
  console.log("[RESPONSE] status:", response.status);
  console.log("[RESPONSE] headers before inject:", JSON.stringify(resHeaders));

  resHeaders["Access-Control-Allow-Origin"] = frontendUrl;
  resHeaders["Access-Control-Allow-Credentials"] = "true";

  // DEBUG — log headers setelah inject  
  console.log("[RESPONSE] headers after inject:", JSON.stringify(resHeaders));

  return {
    statusCode: response.status,
    headers: {
      ...Object.fromEntries(response.headers),
      "Access-Control-Allow-Origin": frontendUrl,
      "Access-Control-Allow-Credentials": "true",
    },
    body: await response.text(),
    isBase64Encoded: false,
  };
};
```
</details>


#### 1.i. package.json

<details><summary>Modify Script</summary>

Ubah `dev` dan `dev:turso` ke file `server.ts`.
```json
{
  "scripts": {
    "dev": "bun run --watch src/server.ts",
    "dev:turso": "bun --env-file=.env.production src/server.ts",
  }
}
```

- ✅ Test: `cd apss/backend && bun dev`
  - jika ada error client, coba generate ulang client `bunx --bun brisma generate`
  - `localhost:3000/users?key=learn` Tampil data.
  - `localhost:3000/auth/login` Harus buka popup Google (Jika dapat `error 400 url_mismatch`, Pastikan `http://localhost:3000/auth/callback` ada di GCC -> API -> Cred -> Client ID -> Tambahkan di list `Authorized redirect URIs`).
</details>

#### Install, Generate & Build

<details><summary>Step Build -> Upload Backend</summary>

```sh
cd apps/backend
# 1. Install bebrapa dependency baru
## aws untuk elysia dapat baca env vars di SSM
## adapter-pg utnuk code client Postgres DB
## JWT untuk metode autentikasi baru  
bun add @aws-sdk/client-ssm @prisma/adapter-pg @elysiajs/jwt

# 2. generate client menggunakan schema-postgres.prisma
bunx prisma generate --schema prisma/schema-postgres.prisma
## akan membuat client di `src/generated/prisma-pg`

# 3. build seluruh kode di 1 file (tapi pisahkan prisma dari build code)
## [?] Menggunakan --target node karena kita pakai runtime "Node", bukan "Bun"
## [?] --format cjs, Common JS. mengganti ESM 'import.meta', jadi CJS 'require'
bun build src/lambda.ts --outdir dist-lambda --target node --format cjs --external prisma

# 4. copy Generated Prisma Client (postgres), dependency, & certificate
cp -r src/generated/prisma-pg dist-lambda/generated/prisma-pg

# jika SSH key belum ada
mkdir -p cert && curl -o cert/global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

mkdir -p dist-lambda/cert && cp cert/global-bundle.pem dist-lambda/cert
cp -r node_modules/.prisma dist-lambda/node_modules/.prisma 2>/dev/null || true

# 5. ZIP untuk upload (38MB -> 3.8MB) (install zip, cth di archLinux: `pacmap -S zip`)
cd dist-lambda && zip -r ../lambda-backend.zip . && cd ..
```
</details>

### 2. Buat Lambda function di AWS Console
Buat Function -> Tambah Role -> Upload ZIP konfigurasi env vars & Function URL. 

**Proses Lambda function Backend Elysia Prisma berikut:**
<details><summary>Buat Lambda Function</summary>

```sh
Buka Aws Console -> Select region "us-east-1 (N. Virginia)"
Lambda → Create function → Author from scratch
  Function name: monorepo-backend
  Runtime: Node.js ^24.x  (Latest support, atau pilih Amazon Linux jika ingin "custom" pakai bun layer)
  Architecture: x86_64
  
  Execution role: "Create new role" with basic Lambda permissions
  → setelah dibuat, attach policy SSM read (dari Admin)
```
</details>

<details><summary>Minta admin tambahkan policy ke Role yang baru dibuat</summary>

Biasanya namanya **monorepo-backend-role-xxx**. Supaya Lambda Function dapat akses env vars di SSM.
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
Beri nama `additionalPolicy_LambdaBE`

**✨ Tips**: gunakan fitur search resource biar mudah
</details>

<details><summary>Upload ZIP</summary>

```sh
Lambda → Functions → [nama function]
  -> tab "Code" → Upload from → .zip file → pilih lambda-backend.zip
    -> Runtime Settings -> Edit
      Handler: lambda.handler
  -> tab "Configuration" -> Edit
      Memory: 512 MB (minimum untuk prisma)
      Timeout: 1 menit (default 3 detik terlalu kecil untuk cold start Prisma)
```
</details>

<details><summary>Set environment variables dari Parameter Store</summary>

```sh
Lambda → Configuration → Environment variables:
  NODE_ENV = production
  
  # Untuk secret mengunakan SSM parameter store reference, BUKAN plaintext di sini
  # dynamic load dari SSM sudah di set di config.ts
```
</details>
<details><summary>Buat Lambda Function URL</summary>

```sh
Lambda → Functions -> Masuk ke fungsi yang baru dibuat
  → tab Configuration → Function URL → Create function URL
    Auth type: NONE  (kita pakai API_KEY manual dari kode Elysia)
    CORS: Disabled (CORS di-handle manual dari kode Elysia)

→ Salin Function URL yang muncul (format: https://xxxxxxxx.lambda-url.us-east-1.on.aws)
→ Kirim URL ini ke Anggota D dan Admin
```
</details>

- ✅ Redirect URI didapatkan ("https://FUNCTION_URL/auth/callback"): minta admin update ke AWS Parameter Store `/monorepo/GOOGLE_REDIRECT_URI` & ke GCC -> API Creds -> select Name di "OAuth 2.0 Client IDs" -> tambahkan url ke Authorized redirect URIs. 
- ✅ Test log:
  - Cara 1: run `aws logs tail /aws/lambda/monorepo-backend --follow` (run dulu `aws login --remote`)
  - Cara 2: CloudWatch -> Logs Insights -> search "/aws/lambda/monorepo-backend" -> Run query (jika ter block "..is not authorized to perform", salin policy actions yang dibutuhkan dan minta admin menambahkannye ke akses ke User Group `grp-lambda-be`)
  - 📢 Jika log kosong, coba akses url dulu, log akan di trigger ulang (terutama jika pakai CLI `aws logs ..`)
- ✅ Test: https://FUNCTION_URL → harus dapat response dari Elysia
- ✅ Test: https://FUNCTION_URL/users?key=learn → harus dapat response data dari Prisma
- ✅ Test: https://FUNCTION_URL/auth/login → harus redirect ke Google 
- ✅ Screenshot **Function Lambda** ([*contoh](https://drive.google.com/file/d/1o5a-7kahnrBD94lnW_hkpJhL15wBv8iF/view?usp=drive_link))

## Fase 5 — Anggota D: S3 Cloudfront Frontend (React)
Mulai setelah Anggota C kirim Function URL backend
> Tunggu Anggota C kirim Lambda Function URL sebelum jalankan vite build.

### 1. Build React dengan VITE_BACKEND_URL dari Lambda C
Vite bake env vars saat build time. Pastikan URL dari Anggota C sudah di-set sebelum vite build.

#### 1.a. App3.tsx
<details><summary>apps/frontend/App3.tsx</summary>

**Perbarui `App3.tsx`** Autentikasi berganti dari *session cookie* jadi **JWT Token**
```tsx
import { useCallback, useEffect, useRef, useState } from "react"
import type { Course, CourseWorkWithSubmission, SubmissionAttachmentItem } from "shared"

import { Button } from "@/components/ui/button"
import { Badge } from "@/components/ui/badge"
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from "@/components/ui/card"
import { Separator } from "@/components/ui/separator"
import { ScrollArea } from "@/components/ui/scroll-area"

// ─────────────────────────────────────────────
// Types
// ─────────────────────────────────────────────

type AuthStatus = "loading" | "unauthenticated" | "authenticated"

// ─────────────────────────────────────────────
// Auth helpers — semua interaksi token lewat sini
// ─────────────────────────────────────────────

const TOKEN_KEY = "token"

const tokenStorage = {
  get: () => localStorage.getItem(TOKEN_KEY),
  set: (t: string) => localStorage.setItem(TOKEN_KEY, t),
  clear: () => localStorage.removeItem(TOKEN_KEY),
}

// Wrapper fetch yang otomatis sisipkan Authorization header.
// Lempar error khusus jika 401 agar caller bisa handle logout.
class UnauthorizedError extends Error {}

const authFetch = async (url: string, token: string): Promise<Response> => {
  const res = await fetch(url, {
    headers: { Authorization: `Bearer ${token}` },
  })

  if (res.status === 401) throw new UnauthorizedError("Token tidak valid atau sudah expired")

  return res
}

// ─────────────────────────────────────────────
// Helpers
// ─────────────────────────────────────────────

function formatDueDate(dueDate?: { year: number; month: number; day: number }) {
  if (!dueDate) return "Tidak ada deadline"
  return new Date(dueDate.year, dueDate.month - 1, dueDate.day).toLocaleDateString("id-ID", {
    day: "numeric", month: "long", year: "numeric",
  })
}

function stateLabel(state?: string) {
  const map: Record<string, { label: string; variant: "default" | "secondary" | "destructive" | "outline" }> = {
    TURNED_IN: { label: "Dikumpulkan", variant: "default" },
    RETURNED: { label: "Dinilai", variant: "secondary" },
    CREATED: { label: "Belum Dikumpulkan", variant: "destructive" },
    NEW: { label: "Belum Dimulai", variant: "outline" },
    RECLAIMED_BY_STUDENT: { label: "Ditarik Kembali", variant: "outline" },
  }
  return map[state ?? ""] ?? { label: state ?? "–", variant: "outline" }
}

// ─────────────────────────────────────────────
// Sub-komponen
// ─────────────────────────────────────────────

function AttachmentLink({ att }: { att: SubmissionAttachmentItem }) {
  if (att.driveFile) {
    return (
      <a href={att.driveFile.alternateLink} target="_blank" rel="noopener noreferrer"
        className="flex items-center gap-1 text-blue-600 hover:underline text-sm">
        📄 {att.driveFile.title}
      </a>
    )
  }
  if (att.link) {
    return (
      <a href={att.link.url} target="_blank" rel="noopener noreferrer"
        className="flex items-center gap-1 text-blue-600 hover:underline text-sm">
        🔗 {att.link.title || att.link.url}
      </a>
    )
  }
  if (att.youtubeVideo) {
    return (
      <a href={att.youtubeVideo.alternateLink} target="_blank" rel="noopener noreferrer"
        className="flex items-center gap-1 text-red-600 hover:underline text-sm">
        ▶ {att.youtubeVideo.title}
      </a>
    )
  }
  if (att.form) {
    return (
      <a href={att.form.responseUrl || att.form.formUrl} target="_blank" rel="noopener noreferrer"
        className="flex items-center gap-1 text-green-600 hover:underline text-sm">
        📝 {att.form.title}
      </a>
    )
  }
  return null
}

function CourseWorkCard({ item }: { item: CourseWorkWithSubmission }) {
  const { courseWork, submission } = item
  const { label, variant } = stateLabel(submission?.state)
  const attachments = submission?.assignmentSubmission?.attachments ?? []
  const score = submission?.assignedGrade ?? submission?.draftGrade

  return (
    <Card className="flex flex-col h-full shadow-sm hover:shadow-md transition-shadow overflow-hidden">
      <CardHeader className="pb-2 shrink-0">
        <div className="flex items-start justify-between gap-2">
          <CardTitle className="text-base leading-snug wrap-break-word min-w-0">
            {courseWork.title}
          </CardTitle>
          <Badge variant={variant} className="shrink-0 whitespace-nowrap">
            {label}
          </Badge>
        </div>
        <CardDescription className="text-xs mt-1">
          🗓 {formatDueDate(courseWork.dueDate)}
        </CardDescription>
      </CardHeader>

      <Separator className="shrink-0" />

      <ScrollArea className="flex-1 min-h-0">
        <CardContent className="flex flex-col gap-3 pt-3 pb-4">
          {courseWork.description && (
            <div className="flex flex-col gap-1">
              <p className="text-xs font-semibold text-muted-foreground">DESKRIPSI</p>
              <p className="text-sm text-foreground whitespace-pre-wrap wrap-break-word line-clamp-4">
                {courseWork.description}
              </p>
            </div>
          )}

          {courseWork.materials && courseWork.materials.length > 0 && (
            <div className="flex flex-col gap-1">
              <p className="text-xs font-semibold text-muted-foreground">LAMPIRAN TUGAS</p>
              <div className="flex flex-col gap-1">
                {courseWork.materials.map((mat, i) => {
                  const att: SubmissionAttachmentItem = {
                    driveFile: mat.driveFile?.driveFile,
                    link: mat.link,
                    youtubeVideo: mat.youtubeVideo,
                    form: mat.form
                      ? { formUrl: mat.form.formUrl, title: mat.form.title, responseUrl: "" }
                      : undefined,
                  }
                  return <AttachmentLink key={i} att={att} />
                })}
              </div>
            </div>
          )}

          {submission && (
            <div className="flex items-center gap-2 shrink-0">
              <p className="text-xs font-semibold text-muted-foreground shrink-0">SKOR</p>
              {score !== undefined ? (
                <span className="text-sm font-bold text-primary">
                  {score} / {courseWork.maxPoints ?? "–"}
                </span>
              ) : (
                <span className="text-sm text-muted-foreground">Belum dinilai</span>
              )}
            </div>
          )}

          {attachments.length > 0 && (
            <div className="flex flex-col gap-1">
              <p className="text-xs font-semibold text-muted-foreground">LAMPIRAN SUBMISI KAMU</p>
              <div className="flex flex-col gap-1">
                {attachments.map((att, i) => (
                  <AttachmentLink key={i} att={att} />
                ))}
              </div>
            </div>
          )}

          {submission?.shortAnswerSubmission?.answer && (
            <div className="flex flex-col gap-1">
              <p className="text-xs font-semibold text-muted-foreground">JAWABAN SINGKATMU</p>
              <p className="text-sm italic wrap-break-word">
                "{submission.shortAnswerSubmission.answer}"
              </p>
            </div>
          )}

          {submission?.late && (
            <div className="pt-1">
              <Badge variant="destructive" className="w-fit text-xs">⚠ Terlambat</Badge>
            </div>
          )}
        </CardContent>
      </ScrollArea>
    </Card>
  )
}

// ─────────────────────────────────────────────
// Main App
// ─────────────────────────────────────────────

export default function App() {
  const [authStatus, setAuthStatus] = useState<AuthStatus>("loading")
  const [courses, setCourses] = useState<Course[]>([])
  const [selectedCourse, setSelectedCourse] = useState<string | null>(null)
  const [items, setItems] = useState<CourseWorkWithSubmission[]>([])
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  // Simpan token di ref agar authFetch selalu pakai nilai terbaru
  // tanpa perlu token masuk ke dependency array effect
  const tokenRef = useRef<string | null>(null)

  // Logout — bersihkan semua state & storage
  const logout = useCallback((reason?: string) => {
    tokenStorage.clear()
    tokenRef.current = null
    setAuthStatus("unauthenticated")
    setCourses([])
    setItems([])
    setSelectedCourse(null)
    if (reason) setError(reason)
  }, [])

  // ── 1. Ambil token dari URL (callback dari Google OAuth) atau localStorage ──
  useEffect(() => {
    const url = new URL(window.location.href)
    const tokenFromUrl = url.searchParams.get("token")

    if (tokenFromUrl) {
      // Bersihkan token dari URL sebelum validasi
      window.history.replaceState({}, document.title, "/")
      tokenStorage.set(tokenFromUrl)
      tokenRef.current = tokenFromUrl
    } else {
      tokenRef.current = tokenStorage.get()
    }

    // ── 2. Validasi token ke /auth/me ──
    // Jika tidak ada token sama sekali, langsung ke halaman login
    if (!tokenRef.current) {
      setAuthStatus("unauthenticated")
      return
    }

    // Ada token → verifikasi ke backend
    fetch(`${import.meta.env.VITE_BACKEND_URL}/auth/me`, {
      headers: { Authorization: `Bearer ${tokenRef.current}` },
    })
      .then(async (res) => {
        if (res.status === 401) {
          logout("Sesi berakhir, silakan login kembali.")
          return
        }
        const data = await res.json()
        if (!data.loggedIn) {
          logout()
          return
        }
        setAuthStatus("authenticated")
      })
      .catch(() => {
        // Jika network error saat validasi, anggap token masih valid
        // agar user tidak dipaksa logout karena masalah koneksi sementara
        setAuthStatus("authenticated")
      })
  }, [logout])

  // ── 3. Load courses setelah status authenticated ──
  useEffect(() => {
    if (authStatus !== "authenticated" || !tokenRef.current) return

    authFetch(`${import.meta.env.VITE_BACKEND_URL}/classroom/courses`, tokenRef.current)
      .then((r) => r.json())
      .then((d) => setCourses(d.data ?? []))
      .catch((e) => {
        if (e instanceof UnauthorizedError) logout("Sesi berakhir, silakan login kembali.")
      })
  }, [authStatus, logout])

  // ── 4. Load submissions saat course dipilih ──
  const loadSubmissions = async (courseId: string) => {
    if (!tokenRef.current) return

    setSelectedCourse(courseId)
    setLoading(true)
    setError(null)

    try {
      const res = await authFetch(
        `${import.meta.env.VITE_BACKEND_URL}/classroom/courses/${courseId}/submissions`,
        tokenRef.current
      )
      const d = await res.json()
      if (d.error) throw new Error(d.error)
      setItems(d.data ?? [])
    } catch (e) {
      if (e instanceof UnauthorizedError) {
        logout("Sesi berakhir, silakan login kembali.")
      } else {
        setError(e instanceof Error ? e.message : "Terjadi error")
      }
    } finally {
      setLoading(false)
    }
  }

  const handleLogin = () => {
    window.location.href = `${import.meta.env.VITE_BACKEND_URL}/auth/login`
  }

  // ── Render ──

  if (authStatus === "loading") {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <p className="text-muted-foreground">Memuat...</p>
      </div>
    )
  }

  if (authStatus === "unauthenticated") {
    return (
      <div className="flex flex-col items-center justify-center min-h-screen gap-4">
        <h1 className="text-2xl font-bold">Google Classroom Viewer</h1>
        <p className="text-muted-foreground">Login dengan akun Google kampus kamu</p>
        {error && (
          <p className="text-sm text-destructive">{error}</p>
        )}
        <Button onClick={handleLogin} size="lg">
          🎓 Login dengan Google
        </Button>
      </div>
    )
  }

  return (
    <div className="min-h-screen bg-background p-6">
      <div className="flex items-center justify-between mb-6">
        <h1 className="text-2xl font-bold">📚 Google Classroom Viewer</h1>
        <Button variant="outline" onClick={() => logout()}>Logout</Button>
      </div>

      <div className="mb-6">
        <p className="text-sm font-semibold text-muted-foreground mb-2">PILIH MATA KULIAH</p>
        <div className="flex flex-wrap gap-2">
          {courses.length === 0 && (
            <p className="text-sm text-muted-foreground">Tidak ada mata kuliah ditemukan.</p>
          )}
          {courses.map((c) => (
            <Button
              key={c.id}
              variant={selectedCourse === c.id ? "default" : "outline"}
              size="sm"
              onClick={() => loadSubmissions(c.id)}
            >
              {c.name}
              {c.section && <span className="ml-1 text-xs opacity-70">· {c.section}</span>}
            </Button>
          ))}
        </div>
      </div>

      <Separator className="mb-6" />

      {error && (
        <div className="mb-4 p-3 rounded bg-destructive/10 text-destructive text-sm">{error}</div>
      )}

      {loading && (
        <div className="text-center py-12 text-muted-foreground">Mengambil data tugas...</div>
      )}

      {!loading && items.length > 0 && (
        <>
          <p className="text-sm text-muted-foreground mb-4">{items.length} tugas ditemukan</p>
          <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
            {items.map((item) => (
              <CourseWorkCard key={item.courseWork.id} item={item} />
            ))}
          </div>
        </>
      )}

      {!loading && selectedCourse && items.length === 0 && (
        <div className="text-center py-12 text-muted-foreground">
          Tidak ada tugas di mata kuliah ini.
        </div>
      )}
    </div>
  )
}
```
</details>

#### 1.b. main.tsx
<details><summary>Menggunakan React Router Dom</summary>

```tsx
import { StrictMode, lazy, Suspense } from 'react'
import { createRoot } from 'react-dom/client'
import { BrowserRouter, Routes, Route } from 'react-router-dom'
import './index.css'

// Menggunakan lazy loading agar file hanya diunduh saat dibutuhkan
const ClassroomApp = lazy(() => import('./App3'))
const DefaultApp = lazy(() => import('./App2'))

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <BrowserRouter>
      {/* Suspense wajib ada saat menggunakan lazy loading */}
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          {/* Rute untuk /classroom */}
          <Route path="/classroom" element={<ClassroomApp />} />
          
          {/* Rute default (index) atau rute lain */}
          <Route path="/" element={<DefaultApp />} />
          
          {/* Opsional: Rute 404 jika halaman tidak ditemukan */}
          <Route path="*" element={<DefaultApp />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  </StrictMode>
)
```
</details>

### 2. Upload dist/ ke S3 bucket (dari Admin)
Upload semua file dari folder dist/ ke S3 bucket yang sudah dibuat Admin.

#### **Test Build**
<details><summary>Step Test Build</summary>

**Add Pacakge -> Set env -> test di dev (berhasil) -> Build**
```sh
cd apps/frontend
# tmbahkan package
bun add react-router-dom

cd ../.. && bun install && bun dev
## Test path "/" (tampil data dari backend)
## Test path "/classroom" (dapat login & tampil data course classroom)
# --- Jika berhasil, lanjut ke langkah build ---

# Buat .env.production (atau export langsung), Pastikan url tanpa '/' di akhir
cd apps/frontend
echo "VITE_BACKEND_URL=https://FUNCTION_URL_DARI_C" > .env.production
echo "VITE_CHECK='vite env check'" >> .env.production
# [?] '>': Write ulang, '>>': Append

bun run build
# Output: apps/frontend/dist/
```

**Verifikasi VITE_BACKEND_URL terbake ke bundle**
```sh
grep -r "lambda-url.on.aws" dist/assets/
# Harus muncul URL Lambda (VITE_BACKEND_URL) di dalam JS bundle
# Kalau tidak muncul → build ulang, cek nama variable VITE_*
```
</details>

#### **Buat access Key untuk IAM**
<details><summary>Step Buat access Key</summary>

Butuh access key untuk upload kode frontend lewat CLI.
```sh
IAM > Users > anggota-d > Create access key
  -> Use Case: Command Line Interface (CLI)
  -> Desc: Frontend Access CLI Upload Code to S3 
  -> Download .csv atau salin
  -> Done
```
</details>

#### **Upload via AWS CLI**
<details><summary>Step Upload via CLI</summary>

```sh
aws configure  # masukkan Access Key dari IAM User kamu 
# atau: `aws configure --profile anggota-d` (sesuaikan nama)
  # Masukkan Key ID, Secret, default region (cth: 'us-east-1'), default output 'json'
  # Jika sudah, periksa koneksi (jika tampil file json -> STS berhasil)
  # tambahkan `--profile anggota-d` jika bukan profile default
aws sts get-caller-identity 

# pastikan folder `apps/frontend/dist/` ada.
# sinkronisasi bucked (upload + hapus), hanya upload file yang berubah (cache 1 tahun)
aws s3 sync dist/ s3://s3-monorepo-frontend-prod/ --cache-control "max-age=31536000" --exclude "index.html"
# tambahkan `--profile anggota-d` (sesuaikan nama) jika bukan default profile

# Upload index.html terpisah karena tanpa cache (SPA perlu selalu fresh)
aws s3 cp dist/index.html s3://s3-monorepo-frontend-prod/index.html   --cache-control "no-cache, no-store"
```

**Akses frontend**
```sh
http://s3-monorepo-frontend-prod.s3-website-us-east-1.amazonaws.com

# Atau cek di: S3 → bucket → Properties → Static website hosting → Bucket website endpoint
```
</details>

> [!NOTE]
> S3 website hosting hanya HTTP, bukan HTTPS. Cookie session.secure=true di backend 
> TIDAK akan berfungsi dari S3 URL biasa. kita akan setup CloudFront (lihat langkah berikut).

#### **Setup CloudFront untuk HTTPS**
<details><summary>Step CloudFront pasang HTTPS</summary>

```sh
CloudFront → Create distribution
  Distribution name: monorepo-fe-dist
  Origin domain: pilih S3 bucket (jangan use website endpoint)
  Customize Cache Settings:
    Viewer protocol policy: Redirect HTTP to HTTPS
    Default root object: index.html

Masuk ke distribution yang dibuat:
  -> Tab Error pages → Create custom error response:
      HTTP error code: 403 → Response path: /index.html, Response code: 200
      HTTP error code: 404 → Response path: /index.html, Response code: 200 
      (ini penting untuk React Router SPA)
    
→ Tunggu ~5 menit deploy, catat CloudFront URL (https://xxxx.cloudfront.net)
→ Kirim ke Admin: update /monorepo/FRONTEND_URL = https://xxxx.cloudfront.net (pastikan url tanpa postfix "/")
```
</details>

- ✅ Test buka URL di browser → halaman React muncul (tampil data dari backend) (buka console Ctrl+shift+J untuk cek apa ada error)
- ✅ Test refresh halaman di route `/classroom` → tidak 404 (SPA fallback bekerja)
- ✅ Screenshot **S3 Bucket** ([*contoh](https://drive.google.com/file/d/1A8eH8WYccBpfmT8hMh_naZqVGozolMA2/view?usp=drive_link)) & **CloudFront Distribution** ([*contoh](https://drive.google.com/file/d/1PxAGFm4mjBCKbf0SMMbJWHLj-1QX3S8Y/view?usp=drive_link)) untuk penilaian.

## Fase 6 — Anggota E: Integrasi & Dokumentasi
Mulai setelah fase C dan D selesai · Verifikasi alur end-to-end
> Jalankan setelah Fase 3 (Lambda BE) dan Fase 4 (S3/FE) selesai.

### 1. Check Fitur
Ini titik integrasi paling rawan gagal. Jalankan checklist berikut secara urut.

**Checklist OAuth flow**
1. Buka FRONTEND_URL di browser
2. Klik "Login dengan Google"
3. Pastikan redirect ke accounts.google.com (Fase 4/Anggota C-bukan error CORS)
4. Login, pastikan redirect kembali ke GOOGLE_REDIRECT_URI (Fase 4/Anggota C-Lambda Backend)
5. Lambda set JWT Token → redirect ke `FRONTEND_URL/classroom` → Cek apakah CORS backend berhasil & mengembalikan response data (DevTools → Network → cek request: `/auth/me` (204), `/classroom/courses`  (204))
6. Buka `/classroom` → cek apakah JWT Token ada di localStorage setelah Login (DevTools → Application → LocalStorage)
7. Buka `/users?key=learn` → harus dapat data dari DB (Anggota B-Database)

**Debug CORS jika gagal**
```sh
Gejala: "CORS policy: No 'Access-Control-Allow-Origin'" di browser console

Cek: Lambda C: process.env.FRONTEND_URL sudah sesuai dengan URL yang dibuka?
   → Lambda → Configuration → Environment variables → FRONTEND_URL
```

**Update GOOGLE_REDIRECT_URI di Google Cloud Console**

Jika OAuth callback gagal dengan "redirect_uri_mismatch", ini penyebabnya.
```sh
Google Cloud Console → APIs & Services → Credentials
→ pilih OAuth 2.0 Client ID yang dipakai
→ Authorized redirect URIs → Add URI: https://FUNCTION_URL/auth/callback

→ Save → tunggu ~5 menit propagasi Google (biasanya cepat)
```
> Minta Admin untuk berikan akses ke Google Cloud Console project, atau Admin yang menambahkan URI-nya.

### 2. Dokumentasi
Publish kode ke github repo & gunakan template  `mono-aws-team-template.md` untuk isian file `README.md` sebagai laporan.

### 3. Teardown (Lakukan setelah penilaian)
- RDS: Stop instance (bukan delete, kalau mau simpan data)
- Lambda: tidak ada biaya jika tidak ada request
- CloudFront: Disable distribution
- S3: Biarkan (biaya storage sangat kecil)

## Parameter penilaian
| Semua anggota (termasuk Admin) | Score |
|---|---|
| Layanan berjalan & dapat diakses | 60% |
| Dokumentasi sesuai & rapi / screenshot sesuai instruksi | 20% |
| Keamanan: least privilege IAM, env tidak hardcode | 20% |

Urutan ketergantungan antar anggota yang perlu diperhatikan tim:
- Admin harus selesai dulu karena seluruh anggota butuh IAM credentials, Security Group, dan Parameter Store sebelum bisa mulai. **A_Budget** adalah satu-satunya yang bisa langsung jalan paralel sejak pagi karena tidak bergantung siapa pun.
- Dua titik sinkronisasi kritis di tengah hari: **B_Database** harus **C_Backend** setelah `DATABASE_URL` dimasukkan ke Parameter Store, dan **C_Backend** harus kirim Lambda Function URL ke **D_Frontend** sebelum D menjalankan `vite build` — karena URL itu di-bake ke dalam bundle JavaScript saat build time, bukan runtime.
