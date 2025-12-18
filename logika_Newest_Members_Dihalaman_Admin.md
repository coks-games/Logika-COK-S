# Logika Newest Members (Anggota Terbaru) di Halaman Admin

Dokumen ini menjelaskan secara teknis bagaimana fitur "Newest Members" pada halaman Dashboard Admin bekerja. Penjelasan dibuat langsung pada intinya dan mudah dipahami.

## 1. Logika Newest Members Game
Secara sederhana, sistem ini melakukan query (permintaan) ke database untuk mengambil data pengguna. Kuncinya ada pada pengurutan data. Sistem meminta database untuk:
1.  Mengambil tabel `users`.
2.  Mengurutkan data berdasarkan kolom `created_at` (waktu pendaftaran).
3.  Jenis pengurutannya adalah **DESC** (Descending), artinya dari yang **terbesar/terbaru** ke yang **terkecil/terlama**.
4.  Membatasi hasil pengambilan hanya **5 data** teratas (`LIMIT 5`).

Hasilnya adalah daftar 5 orang yang baru saja mendaftar ke aplikasi ini.

## 2. File-File Terkait
Berikut adalah file yang bertanggung jawab atas fitur ini dalam arsitektur MVC (Model-View-Controller) project ini:
1.  **`routes/admin.js`** (Route): Mengatur alamat URL `/admin`.
2.  **`controllers/adminController.js`** (Controller): Otak utama yang mengambil data dari database.
3.  **`views/admin/dashboard.ejs`** (View): Tampilan antarmuka yang dilihat admin.

## 3. Detail Implementasi & Source Code
Berikut adalah bedah kode lengkap beserta alasannya.

### A. Route (`routes/admin.js`)
Route ini berfungsi sebagai pintu gerbang. Ketika URL `/admin` diakses, kode ini akan memanggil Controller.

```javascript
// Baris 62 pada routes/admin.js
router.get("/", adminController.dashboard);
```
**Alasan:** Menggunakan method `GET` karena kita hanya mengambil data untuk ditampilkan, bukan mengirim data form. `adminController.dashboard` adalah fungsi tujuan kita.

### B. Controller (`controllers/adminController.js`)
Ini adalah bagian terpenting. Controller melakukan query SQL secara efisien.

```javascript
// Baris 21-56 pada controllers/adminController.js
dashboard: async (req, res) => {
    try {
        // ... (kode statistik lainnya diabaikan untuk fokus ke Newest Members)

        // LOGIKA UTAMA: Mengambil 5 User Terbaru
        const recentUsersResult = await db.query(
            "SELECT id, name, email, avatar, created_at, is_banned FROM users ORDER BY created_at DESC LIMIT 5"
        );

        // Render tampilan dan kirim datanya
        res.render("admin/dashboard", {
            title: "Admin Dashboard",
            page: "dashboard",
            layout: "admin/layout",
            stats, // Data statistik
            recentUsers: recentUsersResult.rows // <-- INI DATANYA. Dikirim ke EJS sebagai variabel 'recentUsers'
        });
    } catch (error) {
        console.error("Admin Dashboard Error:", error);
        res.render("error", { message: "Failed to load admin dashboard", error });
    }
},
```

**Penjelasan Detail Function & Query SQL:**
*   `SELECT id, name, email...`: Kita hanya mengambil kolom yang dibutuhkan untuk ditampilkan (nama, email, foto, status). Kita tidak mengambil password atau data sensitif lain agar lebih aman dan ringan.
*   `ORDER BY created_at DESC`: Ini instruksi kuncinya. Urutkan berdasarkan tanggal buat, dari yang paling baru.
*   `LIMIT 5`: Kita membatasi hanya 5 baris.
    *   **Kenapa LIMIT 5?** Bayangkan jika ada 1000 user dan kita ambil semua tanpa LIMIT, server akan berat (lemot) hanya untuk menampilkan widget kecil ini. Query ini sangat efisien.

### C. View (`views/admin/dashboard.ejs`)
File ini menerima data `recentUsers` dari controller dan menampilkannya menjadi tabel HTML.

```ejs
<!-- Baris 52-70 pada views/admin/dashboard.ejs -->
<tbody>
    <% recentUsers.forEach(u => { %>  <!-- Looping setiap user -->
    <tr>
        <td>
            <div class="d-flex align-items-center gap-3">
                <!-- Tampilkan Avatar. Jika kosong, pakai gambar placeholder -->
                <img src="<%= u.avatar || 'https://via.placeholder.com/40' %>" class="rounded-circle" width="40" height="40">
                <span class="fw-bold text-white"><%= u.name %></span>
            </div>
        </td>
        <td class="text-white-50"><%= u.email %></td>
        <!-- Format Tanggal agar enak dibaca manusia -->
        <td class="text-white-50"><%= new Date(u.created_at).toLocaleDateString() %></td>
        <td>
            <!-- Logika UI: Merah untuk Banned, Hijau untuk Active -->
            <% if(u.is_banned) { %>
                <span class="badge bg-danger">Banned</span>
            <% } else { %>
                <span class="badge bg-success">Active</span>
            <% } %>
        </td>
    </tr>
    <% }) %>
</tbody>
```

**Kenapa kodenya begini?**
*   `forEach`: Kita tidak perlu menulis kode tabel 5 kali secara manual. Jika nanti mau ubah jadi limit 10, tampilan otomatis menyesuaikan.
*   `toLocaleDateString()`: Database menyimpan format waktu mesin (ISO String). Fungsi ini mengubahnya jadi format tanggal lokal yang mudah dibaca.
*   `Ternary Logic (bagan Banned/Active)`: Memberikan feedback visual langsung kepada Admin tentang status user tanpa perlu membuka detail user.
