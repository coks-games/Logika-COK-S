# Logika Logout (Keluar Sistem)

Dokumen ini menjelaskan bagaimana fitur "Logout" bekerja. Fitur ini sederhana namun krusial untuk keamanan akun pengguna, memastikan sesi mereka benar-benar berakhir saat mereka memutuskan untuk keluar.

## 1. Logika Utama Logout

Proses logout di sistem ini sangat "to-the-point":

1.  **Pemicu (Trigger):** Pengguna menekan tombol "Logout" yang ada di menu profil (pojok kanan atas).
2.  **Pemusnahan Sesi:** Server menerima permintaan tersebut dan langsung **menghancurkan (destroy)** sesi aktif pengguna.
3.  **Pengusiran Halus:** Setelah sesi hancur, pengguna langsung dilempar (redirect) kembali ke halaman Login. Mereka tidak bisa lagi mengakses halaman profil atau fitur khusus member sampai mereka login ulang.

## 2. File-File Terkait

Hanya ada 3 file utama yang terlibat dalam operasi senyap ini:
1.  **`views/partials/navbar.ejs`**: Tempat tombol Logout bersemayam.
2.  **`routes/auth.js`**: Jalur khusus yang menangani permintaan `/logout`.
3.  **`controllers/authController.js`**: Eksekutor yang memusnahkan sesi.

## 3. Detail Implementasi & Source Code

Mari kita lihat kode di balik layar.

### A. Tombol Logout (Frontend)

Di dalam file **`views/partials/navbar.ejs`**, tombol logout bersembunyi di dalam dropdown profil. Ia hanyalah sebuah link sederhana `<a>`, bukan form.

```html
<!-- Baris 139: Link Logout -->
<a class="dropdown-item text-danger hover-bg-dark" href="/logout">
    <i class="fas fa-sign-out-alt me-2"></i> Logout
</a>
```
**Kenapa pakai GET (`<a>`) bukan POST?**
Untuk logout standar yang tidak mengubah data di server (hanya menghapus sesi cookie di browser), request GET sudah cukup dan lebih cepat diimplementasikan.

### B. Rute Logout (Route)

Di file **`routes/auth.js`**, kita mendefinisikan jalurnya.

```javascript
const authController = require("../controllers/authController");

// Baris 43: Mengarahkan traffic '/logout' ke controller
router.get("/logout", authController.logout);
```

### C. Eksekusi Logout (Controller)

Di file **`controllers/authController.js`**, inilah fungsi `logout` yang bekerja.

```javascript
// Baris 168
logout: (req, res) => {
    // 1. Hancurkan Sesi!
    req.session.destroy((err) => {
        if (err) {
            // Kalau gagal hancurkan (jarang terjadi), catat errornya
            console.error("Logout error:", err);
        }
        
        // 2. Redirect ke Halaman Login
        // "Silakan masuk lagi kalau mau main"
        res.redirect("/login");
    });
},
```

**Penjelasan Kode:**
*   `req.session.destroy()`: Ini adalah perintah inti. Ia menghapus data sesi yang tersimpan di server (dan secara otomatis membuat cookie di browser user menjadi tidak valid).
*   Callback `(err) => { ... }`: Fungsi yang dijalankan SETELAH sesi berhasil (atau gagal) dihancurkan.
*   `res.redirect("/login")`: Mengembalikan user ke pintu depan.

---

### Kesimpulan
Logika logout ini mengandalkan mekanisme session management dari `express-session`. Saat `destroy` dipanggil, "kartu akses" user langsung dicabut, sehingga jika mereka mencoba menekan tombol *Back* di browser, mereka tetap tidak akan bisa melakukan aksi yang butuh login.
