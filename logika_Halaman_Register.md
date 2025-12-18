# Logika Halaman Register: Gerbang Utama Menjadi Member COKS

Dokumen ini akan mengupas tuntas proses **Register** (Pendaftaran). Ini adalah proses paling krusial karena menyangkut keamanan data user. Kita tidak bisa main-main di sini, password harus diamankan dan validasi harus ketat.

Mari kita lihat bagaimana sistem menerima warga baru!

---

## 1. Logika Registrasi (The Gatekeeper)

Logika pendaftaran di sini menggunakan standar keamanan industri modern: **Hashing**.

Bayangkan user mendaftar dengan password "rahasia123".
1.  **Validasi:** Sistem ngecek dulu, "Passwordnya minimal 8 karakter nggak? Sama nggak sama konfirmasinya? Emailnya udah dipake orang belum?".
2.  **Hashing:** Sistem TIDAK MENYIMPAN "rahasia123" ke database. Itu bahaya! Kalau database bocor, tamatlah riwayat user.
    *   Sistem mengubah "rahasia123" menjadi kode acak panjang: `$2a$10$X7...`.
    *   Proses ini satu arah. Kita bisa ubah password jadi hash, tapi tidak bisa balikin hash jadi password.
3.  **Create User:** Hash itulah yang disimpan ke database.
4.  **Auto Login:** Setelah sukses daftar, ngapain suruh login lagi? Langsung aja kita buatkan session biar user senang.

---

## 2. File-File yang Terlibat

Dua aktor utama penjaga gerbang:

1.  `controllers/authController.js`: Si Petugas Pendaftaran. Dia yang memeriksa formulir (validasi), menolak kalau ada yang salah, dan melapor ke atasan kalau semua oke.
2.  `models/User.js`: Si Brankas. Dia yang punya mesin penghancur (Bcrypt) untuk meng-hash password sebelum menyimpannya ke dalam lemari besi (Database).

---

## 3. Fitur Utama: Implementasi Logika Pengambilan Data

Tunggu, halaman Register kan *mengirim* data (POST), bukan mengambil (GET)? Betul! Tapi ada satu logika pengambilan data (Read) yang terselip:

**Cek Duplikasi Email:**
Sebelum menyimpan user baru, sistem **MENGAMBIL** data user lama dengan email yang sama (`User.findByEmail`).
*   Jika DITEMUKAN: Stop! Tampilkan error "Email already registered".
*   Jika TIDAK DITEMUKAN: Lanjut proses pendaftaran.

---

## 4. Bedah Source Code (Bedah Keamanan)

Mari kita lihat kode aslinya. Perhatikan penggunaan `bcryptjs`.

### Bagian 1: Controller (controllers/authController.js)

Tugas controller adalah validasi, validasi, dan validasi.

```javascript
// File: controllers/authController.js

register: async (req, res) => {
    try {
      const { name, email, password, confirmPassword } = req.body;

      // 1. Validasi Input Dasar
      // Pastikan semua kolom diisi.
      if (!name || !email || !password || !confirmPassword) {
        req.session.error = "All fields are required";
        return res.redirect("/register");
      }

      // 2. Validasi Match Password
      if (password !== confirmPassword) {
        req.session.error = "Passwords do not match";
        return res.redirect("/register");
      }

      // 3. Validasi Keamanan Password
      // Minimal 8 karakter biar gak gampang ditebak.
      if (password.length < 8) {
        req.session.error = "Password must be at least 8 characters";
        return res.redirect("/register");
      }

      // 4. Cek Duplikasi Database (PENGAMBILAN DATA)
      // Kita tanya Model: "Ada gak orang pake email ini?"
      const existingUser = await User.findByEmail(email);
      if (existingUser) {
        req.session.error = "Email already registered";
        return res.redirect("/register");
      }

      // 5. Eksekusi Pendaftaran
      // Kalau semua aman, suruh Model simpan.
      // Perhatikan: Kita kirim password MENTAH ke model. Model yang akan nge-hash.
      const user = await User.create({ name, email, password });

      // 6. Auto Login (Session)
      // User langsung dianggap login (punya session) setelah daftar.
      req.session.user = {
        id: user.id,
        name: user.name,
        email: user.email,
        role: "user", // Default role
      };

      res.redirect("/games");
    } catch (error) {
      console.error("Register error:", error);
      res.redirect("/register");
    }
},
```

### Bagian 2: Model (models/User.js)

Di sinilah sihir hashing terjadi.

```javascript
// File: models/User.js

const bcrypt = require("bcryptjs"); // Library standar industri untuk hashing

module.exports = {
  // Fungsi utama membuat user
  create: async (userData) => {
    const { name, email, password } = userData;

    // 1. HASHING (SECURITY CRITICAL)
    // Angka 10 itu "Salt Rounds". Semakin tinggi semakin aman tapi semakin lambat.
    // 10 adalah standar yang pas (aman dan cepat).
    const hashedPassword = await bcrypt.hash(password, 10);

    // 2. Simpan ke Database
    // Perhatikan: Yang disimpan adalah 'hashedPassword', BUKAN 'password'.
    const result = await db.query(
      `INSERT INTO users (name, email, password, created_at, updated_at) 
       VALUES ($1, $2, $3, NOW(), NOW()) 
       RETURNING id, name, email`,
      [name, email, hashedPassword]
    );

    return result.rows[0];
  },

  // Fungsi bantu untuk cek email
  findByEmail: async (email) => {
    const result = await db.query("SELECT * FROM users WHERE email = $1", [email]);
    return result.rows[0] || null;
  }
};
```

## Kesimpulan

Halaman Register bukan sekadar form isian. Di baliknya ada benteng pertahanan pertama aplikasi kita.
1.  **Validasi Input** di Controller mencegah data sampah masuk.
2.  **Pengecekan Duplikasi** mencegah konflik identitas.
3.  **Hashing Password** di Model menjamin keamanan privasi user selamanya.

Sistem yang baik itu paranoid. Dia tidak percaya input user sampai benar-benar dicek validitasnya!
