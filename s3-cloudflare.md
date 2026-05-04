# PPWL 13
AWS S3 & Cloudflare Domain (+SSL). Lanjutan PPWL11. Mengganti endpoint frontend dari Cloudfront ke Cloudflare. 
**Cara kerja:**
- User → HTTPS (Cloudflare SSL) → Cloudflare
- Cloudflare → HTTP → S3
- Jadi tetap HTTPS dari sisi user

**Langkah:**
- Beli domain murah di Namecheap (beri nama tim dan kelas, cth: "ppwl-a1.store" Kelas A, kelompok 1)
- Daftarkan domain ke Cloudflare:
    - Login ke Cloudflare: Disarankan sign-in manual di cloudflare (nama & password), pakai OAuth sering tidak bekerja.
    - Ikuti [Tutorial](https://youtu.be/dowXP-kKw5E?si=nREc-L575VbSPPST) cara pakai nameserver Cloudflare. 
    - Hasilnya bisa cek di menu Cloudflare -> Domain (jika tulisan status aktif berarti berhasil, [contoh](https://drive.google.com/file/d/1uJfomrjbSMrza7tomHjPh0rUJz2r838J/view?usp=drive_link))
    
Langkah selanjutnya masih di update..
