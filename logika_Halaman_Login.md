# Logika Halaman Login: Kunci Pintu Masuk COKS

Dokumen ini menjelaskan bagaimana sistem kita memeriksa "Siapa Kamu?". Proses **Login** adalah tentang verifikasi identitas. Kalau Register adalah tentang membuat kunci, Login adalah proses mencocokkan kunci tersebut dengan lubangnya.

Mari kita lihat bagaimana sistem bekerja di balik layar!

---

## 1. Logika Autentikasi (Verification Process)

Kita menggunakan teknik yang disebut **Compare Hash** (Bandingkan Kode Acak).

User memasukkan email "budi@gmail.com" dan password "rahasia123".
*   Langkah 1 (Cari User): Sistem mencari di database, "Ada gak Budi?".
*   Langkah 2 (Ambil Hash): Kalau ada, sistem mengambil password ter-enkripsi milik Budi dari database (Contoh: `$2a$10$X7...`).
*   Langkah 3 (Compare - INTINYA DISINI):
    *   Sistem tidak men-dekripsi hash tersebut (karena tidak bisa).
    *   Sistem memanggil `bcrypt.compare("rahasia123", "$2a$10$X7...")`.
    *   Bcrypt akan menghitung ulang secara matematis. Jika cocok, `return true`.
*   Langkah 4 (Bikin Tiket): Jika cocok, sistem membuat **Session** (tiket masuk). Selama tiket ini belum hilang (expired), Budi bebas keliling website tanpa perlu login lagi.

---

## 2. File-File yang Terlibat

Masih sama dengan Register, dua aktor utamanya adalah:

1.  `controllers/authController.js`: Si Penjaga Pintu. Dia yang menerima input user dan memanggil model. Dia juga yang menerbitkan "Session Token" jika login berhasil.
2.  `models/User.js`: Si Pemegang Kunci. Dia yang punya fungsi `verifyPassword` untuk membandingkan password inputan dengan password asli di database.

---

## 3. Fitur Utama: Implementasi Logika Pengambilan Data (Login Check)

Di halaman ini, logika "Pengambilan Data" terjadi sangat intens di awal:

1.  **Ambil User:** `User.findByEmail(email)`.
    *   Tujuannya untuk memastikan user itu eksis. Kalau gamenya saja belum didownload, gimana mau main? Sama, kalau usernya gak ada, tolak langsung.
2.  **Cek Status Banned:**
    *   Setelah data user diambil, kita cek kolom `is_banned`.
    *   Kalau `true`, tendang keluar! Keamanan nomor satu.

---

## 4. Bedah Source Code (Bedah Autentikasi)

Mari kita telaah kode krusial ini.

### Bagian 1: Controller (controllers/authController.js)

Controller ini bertugas menjadi orkestrator alur login.

```javascript
// File: controllers/authController.js

login: async (req, res) => {
    try {
      const { email, password } = req.body;

      // 1. Validasi Input Kosong
      if (!email || !password) {
        req.session.error = "Email and password are required";
        return res.redirect("/login");
      }

      // 2. Ambil Data User (PENGAMBILAN DATA UTAMA)
      const user = await User.findByEmail(email);
      
      // Kalau user gak ketemu...
      if (!user) {
        // Trik Keamanan: Jangan bilang "Email tidak ditemukan".
        // Bilang "Invalid email or password" biar hacker bingung mana yang salah.
        req.session.error = "Invalid email or password";
        return res.redirect("/login");
      }

      // 3. Cek Tipe Login (OAuth vs Password)
      // Kalau user daftar pake Google, dia gak punya password.
      // Jadi suruh dia login pake Google lagi.
      if (!user.password) {
        req.session.error = "Please login using your OAuth provider";
        return res.redirect("/login");
      }

      // 4. Cek Status Banned (Fitur Security)
      if (user.is_banned) {
        req.session.error = `Account Suspended: ${user.ban_reason}`;
        return res.redirect("/login");
      }

      // 5. Verifikasi Password
      // Panggil fungsi sakti di Model.
      const isValidPassword = await User.verifyPassword(password, user.password);
      
      if (!isValidPassword) {
        req.session.error = "Invalid email or password";
        return res.redirect("/login");
      }

      // 6. Buat Session (TIKET MASUK)
      // Di sinilah status "Logged In" terjadi.
      // Kita simpan data penting di memory server (session).
      req.session.user = {
        id: user.id,
        name: user.name,
        email: user.email,
        role: isAdmin(user) ? "admin" : "user", // Cek admin disini
      };

      res.redirect("/");
    } catch (error) {
      // Error handling...
    }
},
```

### Bagian 2: Model (models/User.js)

Ini adalah fungsi yang melakukan operasi kriptografi.

```javascript
// File: models/User.js

// Fungsi verifikasi password
verifyPassword: async (plainPassword, hashedPassword) => {
    // Safety check: Kalau hash-nya kosong (user OAuth), pasti false.
    if (!hashedPassword) {
      return false; 
    }
    // bcrypt.compare adalah fungsi async yang berat secara komputasi.
    // Sengaja dibuat berat biar tidak bisa di-bruteforce dengan cepat.
    return await bcrypt.compare(plainPassword, hashedPassword);
},
```

## Kesimpulan

Login adalah gerbang pertahanan kedua setelah Register. Logika kuncinya ada pada **Verifikasi Bertingkat**:
1.  Ada gak usernya?
2.  Boleh masuk gak (gak ke-banned)?
3.  Passwordnya bener gak?

Hanya user yang lolos ketiga filter ini yang berhak mendapatkan tiket (Session) untuk masuk ke dalam dunia COKS.
