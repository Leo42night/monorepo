# PPWL 11/Monorepo - Aws Teams (template)
[![PPWL 2026 Asdos - 11](https://img.shields.io/badge/PPWL_2026_Asdos-11-blue?style=for-the-badge&logo=github)](https://github.com/USERNAME/REPO-ANDA)
[![Date](https://img.shields.io/badge/Date-16%20April%202026-blue?style=flat-square&logo=calendar-check)](https://github.com/USERNAME/REPO-ANDA)

> [!NOTE]
> Sesuaikan template ini dengan tim anda, dapat menambahkan segmen bila perlu.  Hapus NOTE ini dan instruksi lain yang tidak perlu, ganti dengan informasi yang diminta.

## Class A / Team 1

| Nama | NIM | Job |
|------|-----|-----|
| Budi | H1101240000 | Admin |
| Other | H110124000 | Budget & Cost |
| Other | H110124000 | RDS Database |
| Other | H110124000 | Lambda Backend |
| Other | H110124000 | S3+CloudFront Frontend |
| Other | H110124000 | Test & Docs |

[Canva - Arsitektur Diagram AWS](#)
> [!NOTE]
> Gunakan Canva Papan Tulis.
> Isinya Berisi diagram yang diminta. Masukkan Screenshoot yang diminta pada tiap fase ke tiap node terkait.
> (*[contoh](https://canva.link/xdtc48nrltbss4k). Contoh hanyalah referensi, gabungan arsitektur & flow diagram. Tidak perlu sekompleks ini, lihat contoh Arsitektur Diagram di internet, tidak perlu ikuti standar baku, desain sesuai kebutuhan style kalian)
> Screenshot lainnya dapat ditambahkan bila perlu, tapi Screenshot yang wajib ada adalah (s3-frontend, rds-database, parameter_store, lambda-backend, cloudfront-frontend, budget, & task-cost-report). Pasangkan screenshot dengan tiap node arsitektur, buat struktur flow bila perlu.

**Komponen yang harus muncul di diagram**
```sh
Internet → CloudFront (HTTPS) → S3 (React SPA)
Internet → Lambda Function URL → Lambda BE (Elysia)
Lambda BE → RDS/Aurora (VPC internal, port 5432)
Lambda BE → Turso/LibSQL (via internet, HTTPS)
Lambda BE → SSM Parameter Store (secret injection)
Lambda BE → Google OAuth (redirect flow)
AWS Budgets → SNS → Email alerts
```

## Report Bug
Laporkan jika masih menemukan bug/error. Sertakan pesan bug/error, Lokasi Bug yang ditemukan, Alur sebelum bug didapatkan, & Screenshot bila perlu. Jelaskan juga solusi yang sudah dicoba.
