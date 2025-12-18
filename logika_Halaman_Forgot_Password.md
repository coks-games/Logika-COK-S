# Logika Halaman Forgot Password: Penyelamat Akun yang Hilang

Dokumen ini membahas fitur darurat: **Forgot Password**. Saat user lupa password, kita tidak bisa sekadar "kirim balik password lama mereka" (karena password di-hash dan kita pun tidak tahu isinya). Jadi, solusinya adalah **Reset Token**.

Mari kita bedah mekanisme penyelamatan ini!

---

## 1. Logika Reset Password (The Rescue Mission)

Mekanisme ini menggunakan sistem **Token Sekali Pakai (One-Time Token)**.

1.  **Request:** User mengetik email mereka.
2.  **Verifikasi:** Sistem mengecek, "Email ini terdaftar nggak?".
3.  **Token Generation:** Kalau terdaftar, sistem membuat kode acak panjang (Token) yang unik. Kode ini punya masa berlaku (expired), misal cuma 1 jam.
4.  **Email:** Sistem mengirim email ke user berisi link khusus: `coks.site/reset-password?token=KODE_ACAK_TADI`.
5.  **Simpan:** Token tadi disimpan di database di baris data milik user tersebut.

Kenapa harus begini? Supaya orang asing tidak bisa sembarangan ganti password orang lain tanpa akses ke email pemilik asli.

---

## 2. File-File yang Terlibat

Tiga komponen bekerja sama dalam misi ini:

1.  `controllers/authController.js`: Si Komandan. Dia mengatur alur pembuatan token dan pengiriman email.
2.  `models/User.js`: Si Arsip. Dia menyediakan tempat untuk menyimpan token sementara di database.
3.  `utils/emailService.js`: Si Kurir. File utilitas khusus yang tugasnya cuma satu: Kirim Email via Nodemailer.

---

## 3. Fitur Utama: Implementasi Logika Pengambilan Data

Di fase awal Forgot Password, pengambilan data fokus pada verifikasi eksistensi user.

**Security Logic:**
Satu hal menarik di sini adalah **"Fake Success Message"**.
Jika user memasukkan email yang *tidak terdaftar*, kita JANGAN bilang "Email tidak ditemukan".
Kenapa? Karena itu memberi tahu hacker email mana yang valid dan tidak.
Kita harus selalu bilang: "Jika email valid, kami sudah mengirimkan link reset." Biar hacker bingung.

---

## 4. Bedah Source Code (Bedah Token)

Mari kita lihat bagaimana token dibuat dan disimpan.

### Bagian 1: Controller (controllers/authController.js)

```javascript
// File: controllers/authController.js

forgotPassword: async (req, res) => {
    try {
      const { email } = req.body;
      // Gunakan modul crypto bawaan Node.js untuk bikin kode acak
      const crypto = require('crypto'); 
      const { sendPasswordResetEmail } = require('../utils/emailService');

      // 1. Cek User (PENGAMBILAN DATA)
      const user = await User.findByEmail(email);
      
      // 2. Security Check (User Gak Ada)
      if (!user) {
        // Fake Success: Pura-pura berhasil biar aman dari User Enumeration Attack.
        req.session.success = "If that email exists, we've sent a password reset link.";
        return res.redirect("/login");
      }

      // 3. Cek User OAuth
      // Kalau dia login pake Google, dia gak punya password buat direset.
      if (!user.password) {
        req.session.error = "This account uses OAuth. Please login with Google/Discord.";
        return res.redirect("/forgot-password");
      }

      // 4. Generate Token
      // Bikin 32 bytes data acak lalu ubah jadi string hexadesimal.
      const token = crypto.randomBytes(32).toString('hex');
      // Set expired 1 jam (3600000 ms) dari sekarang.
      const expires = new Date(Date.now() + 3600000); 

      // 5. Simpan Token ke Database
      await User.setResetToken(email, token, expires);

      // 6. Kirim Email
      const emailSent = await sendPasswordResetEmail(user, token);

      if (!emailSent) {
          // Hanya error ini yang boleh jujur (karena masalah teknis server)
          req.session.error = "Failed to send email. Check server logs.";
          return res.redirect("/forgot-password");
      }

      req.session.success = "Password reset link sent! Check your email inbox.";
      res.redirect("/login");

    } catch (error) {
       // Error handling...
    }
},
```

### Bagian 2: Model (models/User.js)

Model bertugas menyimpan token tersebut ke kolom khusus di tabel users.

```javascript
// File: models/User.js

// Fungsi update token
setResetToken: async (email, token, expires) => {
     // Kita update user berdasarkan emailnya.
     // Kita isi kolom 'reset_token' dan 'reset_token_expires'.
     await db.query(
        "UPDATE users SET reset_token = $1, reset_token_expires = $2 WHERE email = $3",
        [token, expires, email]
     );
},
```

## Kesimpulan

Fitur Forgot Password adalah fitur yang jarang dipakai tapi wajib ada. Logikanya menggabungkan **Keamanan** (Token & Fake Success) dengan **Utilitas** (Email Service).

Dengan sistem token ini, kita menjamin bahwa hanya pemilik email aslilahlah yang bisa mereset password mereka. Aman dan terpercaya!
