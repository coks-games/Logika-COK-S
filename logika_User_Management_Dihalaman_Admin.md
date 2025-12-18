# Logika User Management (Manajemen Pengguna) di Halaman Admin

Dokumen ini menjelaskan bagaimana fitur "User Management" bekerja, mulai dari menampilkan daftar user hingga memblokir (ban) user yang bermasalah. Penjelasan ini dibuat khusus agar mudah dipahami, langsung pada intinya, dan tanpa jargon yang membingungkan.

## 1. Logika Utama User Management

Ada dua proses utama yang terjadi di sini:

1.  **Menampilkan Daftar User:** Saat Admin membuka halaman `/admin/users`, sistem akan mengambil *semua* data pengguna dari database dan menampilkannya dalam bentuk tabel. Data diurutkan dari yang paling baru mendaftar (`DESC`). Admin bisa melihat foto, nama, email, dan status apakah user tersebut Aktif atau Banned.
2.  **Memblokir (Ban) User:** Jika ada user nakal, Admin bisa menekan tombol "Ban". Saat tombol ditekan, sistem akan:
    *   Menerima ID user dan alasan pemblokiran.
    *   Mengubah status user di database menjadi "Banned" (`is_banned = TRUE`).
    *   Menyimpan alasan pemblokiran.
    *   User tersebut otomatis tidak bisa login lagi (tergantung implementasi login check).

## 2. File-File Terkait

Fitur ini melibatkan beberapa file yang bekerja sama (MVC Pattern):
1.  **`routes/admin.js`**: Pintu masuk yang mengatur alamat URL (seperti `/admin/users` dan `/admin/users/:userId/ban`).
2.  **`controllers/adminController.js`**: "Otak" yang memproses permintaan Admin.
3.  **`models/User.js`**: "Gudang" yang berinteraksi langsung dengan database SQL untuk mengambil atau mengubah data user.
4.  **`views/admin/users.ejs`**: Tampilan antarmuka (tabel dan tombol) yang dilihat Admin.

## 3. Detail Implementasi & Source Code

Berikut adalah penjelasan teknis (bedah kode) untuk kedua fitur utama di atas.

### A. Menampilkan Daftar User (Data Retrieval)

**Alur:** Admin akses `/admin/users` -> Route panggil Controller -> Controller minta data ke Model -> Controller kirim data ke View.

**1. Route (`routes/admin.js`)**
```javascript
// Baris 66: Router menerima request GET
router.get("/users", adminController.listUsers);
```

**2. Controller (`controllers/adminController.js`)**
Ini adalah fungsi `listUsers`.
```javascript
listUsers: async (req, res) => {
    try {
        // Panggil Model untuk ambil data
        // Kita pakai query manual di sini agar lebih spesifik (bisa ambil ban_reason juga)
        const result = await db.query(`
            SELECT id, name, email, avatar, created_at, is_banned, ban_reason 
            FROM users 
            ORDER BY created_at DESC
        `);
      
        // Render file EJS dan kirim datanya
        res.render("admin/users", {
            title: "User Management",
            page: "users",      // Untuk highlight menu sidebar
            layout: "admin/layout",
            users: result.rows  // Data user dikirim dengan nama variabel 'users'
        });
    } catch (error) {
        // Error handling standar
        console.error("List Users Error:", error);
        res.status(500).send("Server Error");
    }
},
```
**Alasannya:**
*   Menggunakan `SELECT ... FROM users` untuk mengambil data mentah.
*   `ORDER BY created_at DESC`: Agar user yang baru daftar muncul paling atas (memudahkan Admin memantau pendaftaran baru).

### B. Fitur Ban User

**Alur:** Admin klik "Confirm Ban" di Modal -> Form mengirim data ke Route -> Route panggil Controller -> Controller suruh Model update DB -> Kembali ke halaman list.

**1. View (`views/admin/users.ejs`)**
Bagian ini adalah tombol dan Modal form. Perhatikan `action` pada form.
```ejs
<!-- Tombol pemicu Modal -->
<button type="button" class="btn btn-sm btn-danger" data-bs-toggle="modal" data-bs-target="#banModal<%= u.id %>">
    <i class="fas fa-ban"></i> Ban
</button>

<!-- Form dalam Modal -->
<!-- Perhatikan URL action-nya dinamis sesuai ID user -->
<form action="/admin/users/<%= u.id %>/ban" method="POST">
    <!-- Input alasan ban -->
    <textarea name="reason" class="form-control" required></textarea>
    <button type="submit" class="btn btn-danger">Confirm Ban</button>
</form>
```

**2. Route (`routes/admin.js`)**
```javascript
// Baris 67: Menerima request POST dari form tadi
// :userId adalah parameter dinamis (misal: 10, 11, dst)
router.post("/users/:userId/ban", adminController.banUser);
```

**3. Controller (`controllers/adminController.js`)**
Fungsi `banUser` mengeksekusi logika pemblokiran.
```javascript
banUser: async (req, res) => {
    try {
        const { userId } = req.params; // Ambil ID dari URL
        const { reason } = req.body;   // Ambil alasan dari Form
      
        // VALIDASI PENTING: Jangan sampai Admin mem-banned dirinya sendiri!
        if (parseInt(userId) === req.session.user.id) {
            req.session.error = "You cannot ban yourself!";
            return res.redirect("/admin/users");
        }

        // Panggil Model untuk update database
        await User.ban(userId, reason || "Violation of community rules");
      
        // Beri notifikasi sukses dan refresh halaman
        req.session.success = "User has been banned successfully.";
        res.redirect("/admin/users");
    } catch (error) {
        console.error("Ban User Error:", error);
        req.session.error = "Failed to ban user.";
        res.redirect("/admin/users");
    }
},
```

**4. Model (`models/User.js`)**
Fungsi ini yang melakukan perubahan ke database sesungguhnya.
```javascript
ban: async (userId, reason) => {
    // Jalankan perintah SQL UPDATE
    await db.query(
      "UPDATE users SET is_banned = TRUE, ban_reason = $1, updated_at = NOW() WHERE id = $2",
      [reason, userId]
    );
},
```
**Alasannya:**
*   Menggunakan `UPDATE`, bukan `DELETE`. Kita tidak menghapus user, tapi hanya mengubah statusnya menjadi `TRUE` pada kolom `is_banned`.
*   Data user tetap aman, history-nya tetap ada, tapi aksesnya yang dicabut. Ini praktik standar keamanan (Audit Trail).
