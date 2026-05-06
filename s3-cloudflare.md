# PPWL 13 (Khusus yg tidak dapat akses AWS Cloudfront)
AWS S3 & Cloudflare Domain (+SSL). Lanjutan PPWL11. Mengganti endpoint frontend dari Cloudfront ke Cloudflare. 

**Cara kerja:**
- User → HTTPS (Cloudflare SSL) → Cloudflare
- Cloudflare → HTTP → S3
- Jadi tetap HTTPS dari sisi user

**Langkah:**
- Beli domain murah di Namecheap (beri nama tim dan kelas, cth: "ppwl-a1.store" Kelas A, kelompok 1)
- Buka domain anda di browser (cth: `ppwl.com`). Jika dapat `Whois veriﬁcation is pending`. Periksa email, jika tidak ada, klik `Didn't get the email?` di tampilan web tersebut. Aktivasi bisa dari 24 - 48 hours.
- Daftarkan domain ke Cloudflare:
    - Login ke Cloudflare: Disarankan sign-in manual di cloudflare (nama & password), pakai OAuth sering tidak bekerja.
    - Ikuti [Tutorial](https://youtu.be/dowXP-kKw5E?si=nREc-L575VbSPPST) cara pakai nameserver Cloudflare. 
    - Hasilnya bisa cek di menu Cloudflare -> Domain (jika tulisan status aktif berarti berhasil, [contoh](https://drive.google.com/file/d/1uJfomrjbSMrza7tomHjPh0rUJz2r838J/view?usp=drive_link))
    

## Config S3 AWS + Cloudflare
Anda akan buat 2 bucker, sesuaikan nama domain:
1. www.domain.com → bucket utama (isi website)
2. domain.com → redirect ke www
Tujuannya agar web dapat diakses tanpa perlu menulis prefix www

### STEP 1 — Bucket utama (`www.domain.com`)
1. Beri nama sesuai `www.domain.com` (sesuaikan domain).
2. Block all public access: OFF (kita perlu public baca). Centang "I acknowledge that the current ... becoming public."
3. Properties → Static website hosting
    - Enable
    - Index document: `index.html`
    - (optional) error: `index.html`
4. Tambahkan bucket policy
<details><summary>Contoh Bucket Policy</summary>

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "Statement1",
			"Principal": "*",
			"Effect": "Allow",
			"Action": [
				"s3:GetObject"
			],
			"Resource": [
				"arn:aws:s3:::www.ppwl-a1.store/*"
			]
		}
	]
}
```
</details>

### 🔁 STEP 2 — Bucket redirect (`domain.com`)
1. Beri nama sesuai `www.domain.com` (sesuaikan domain).
2. Block all public access: OFF (kita perlu public baca). Centang "I acknowledge that the current ... becoming public."
3. Properties → Static website hosting
    - Enable
    - Pilih: **Redirect requests**
    - Target domain: `www.domain.com`
    - Protocol: **http**

Jadi: `domain.com → redirect → www.domain.com`

### 🌐 STEP 3 — Ambil endpoint S3
Format endpoint:
```sh
http://BUCKET.s3-website-REGION.amazonaws.com
```

Lihat di bagian **Properties -> Static website hosting**, contoh:
```sh
www.ppwl.com.s3-website-us-east-1.amazonaws.com
ppwl.com.s3-website-us-east-1.amazonaws.com
```

### 🌐 STEP 4 — Setup DNS di Cloudflare
Masuk ke Cloudflare → DNS -> Records
Hapus beberapa record default (jika ada) yang mengarah ke IP lama, contoh:
- `A www → 198.54.117.242` (Hapus! karena mengarah ke server lama)
- `A * → 198.54.117.242` (Hapus! Wildcard * akan override semua subdomain)
- `A domain.com → 198.54.117.242` (Hapus! redirect ke IP lama)

Tambahkan record baru yang mengarah ke S3 Bucket:
1. ✔️ Record 1 (www → S3 utama)
    - Type: `CNAME`
    - Name: `www`
    - Target: `www.ppwl.com.s3-website-REGION.amazonaws.com`
    - Proxy: ON (☁️ orange)
    - tekan Save
2. ✔️ Record 2 (root → S3 redirect)
    - Type: `CNAME`
    - Name: `@`
    - Target: `ppwl.com.s3-website-REGION.amazonaws.com`

### 🔒 STEP 5 — Aktifkan & Force HTTPS
Menu SSL ada ketika anda di dalamn domain ([*contoh](https://drive.google.com/file/d/1i7z2gAsojSUJNJereLwZXAC0am_RjSuZ/view?usp=drive_link))
```sh
Aktifkan HTTPS:
- Masuk: SSL/TLS → Overview → Configure
- Pilih: Flexible

Simpan. Salu lihat, Current encryption mode: `Flexible` (menghilangkan security warning di browser & google OAuth)

Force HTTPS:
- Masuk: SSL/TLS → Edge Certificates
- Aktifkan: ✅ Always Use HTTPS
```

## Flow akhirnya
buka domain anda di browser (cthL: `ppwl.com`). Jika dapat `Whois veriﬁcation is pending`. Periksa email, jika tidak ada, klik `Didn't get the email?` di tampilan web tersebut. Aktivasi bisa dari 24 - 48 hours.
