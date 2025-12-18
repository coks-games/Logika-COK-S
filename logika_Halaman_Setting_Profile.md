# Logika Halaman Setting Profile: Dapur Pribadi User

Halaman Setting Profile adalah ruang pribadi setiap user. Di sini mereka bisa ganti nama, pasang foto profil keren, atau nulis biografi singkat.

Dokumen ini akan membahas bagaimana alur data mengalir dari database ke form edit, lalu kembali ke database setelah dipermak.

---

## 1. Logika Pengambilan Data (Persiapan Sebelum Edit)

Sebelum user bisa mengedit profilnya, kita harus menampilkan dulu data yang *sekarang* ada. Kan lucu kalau form edit-nya kosong.

Logikanya:
1.  **Identifikasi User:** Sistem membaca Session yang sedang aktif (`req.session.user.id`).
2.  **Ambil Data:** Sistem mengambil data user tersebut dari Database (`User.findById`).
3.  **Inject ke Form:** Data (Nama, Email, Avatar URL, Bio) dimasukkan ke dalam atribut `value="..."` di HTML.
4.  **Kunci Email:** Email ditampilkan tapi di-disable (tidak bisa diedit), karena email adalah identitas unik login.

---

## 2. File-File yang Terlibat

1.  `routes/profile.js`: Gerbang tol. Dia mengatur rute `/profile/edit/me` dan `/profile/update/me`.
2.  `controllers/profileController.js`: Koki utama. Dia yang mengambil data user dan mengolah upload foto.
3.  `config/supabase.js`: Gudang foto. Kita tidak simpan foto di server sendiri, tapi di Supabase Storage (Cloud) biar hemat tempat.

---

## 3. Fitur Utama: Implementasi Aksi & Logika

Ada dua kejadian besar di halaman ini.

### A. Membuka Halaman Edit (GET /profile/edit/me)
Seperti yang dijelaskan di poin 1, ini cuma proses "Membaca dan Menampilkan". User tidak boleh melihat data orang lain, makanya kuncinya ada pada `req.session.user.id` (ambil ID diri sendiri).

### B. Menekan Tombol "Save Changes" (PUT /profile/update/me)
Inilah bagian tersulitnya.

1.  **Terima Data:** Server menerima Nama, Bio, dan *File Gambar* (kalau ada).
2.  **Cek Gambar:**
    *   Jika user upload gambar baru, kita proses dulu.
    *   **Resize:** Gambar dikecilkan maks 500x500px biar ringan (pakai library `sharp`).
    *   **Upload Cloud:** Gambar yang sudah optimal di-upload ke Supabase.
    *   **Dapat Link:** Supabase ngasih link publik, misal `supabase.co/storage/.../foto.jpg`.
3.  **Update Database:** Link foto tadi + Nama + Bio disimpan ke tabel `users`.
4.  **Update Session:** JANGAN LUPA! Session yang aktif juga harus diperbarui namanya dan avatarnya, supaya navbar di pojok kanan atas langsung berubah fotonya tanpa harus logout dulu.

---

## 4. Bedah Source Code (Bedah Operasi Plastik)

Kita fokus ke bagian Update yang paling kompleks karena melibatkan upload file.

### Bagian 1: Route (routes/profile.js)

Route ini pakai Middleware khusus buat handle upload.

```javascript
// File: routes/profile.js

// Middleware 'uploadMiddleware' ini tugasnya nerima file dari form HTML
// Kita batasi ukuran file handling di memory maksimal 20MB.
router.put("/update/me", isAuthenticated, uploadMiddleware, profileController.update);
```

### Bagian 2: Controller (controllers/profileController.js)

```javascript
// File: controllers/profileController.js

update: async (req, res) => {
    try {
      const { name, bio } = req.body;
      let avatarUrl = req.body.avatar; // Default: Pakai URL lama kalau gak ada upload baru

      // 1. Handle File Upload (Jika ada file baru)
      if (req.file) {
        // Optimize Image: Kecilkan ukuran biar hemat kuota dan cepat loading
        const sharp = require('sharp');
        const optimizedBuffer = await sharp(req.file.buffer)
          .resize(500, 500, { fit: 'cover' }) // Crop jadi kotak 500px
          .jpeg({ quality: 80 }) // Kompres jadi JPEG kualitas 80%
          .toBuffer();

        // Nama file unik: ID User + Timestamp
        const filePath = `profiles/${req.session.user.id}-${Date.now()}.jpg`;
        
        // Upload ke Supabase Storage
        await supabase.storage
          .from('avatars')
          .upload(filePath, optimizedBuffer, { contentType: 'image/jpeg' });

        // Ambil URL Publiknya
        const { data } = supabase.storage
          .from('avatars')
          .getPublicUrl(filePath);
          
        avatarUrl = data.publicUrl; // Update variabel URL
      }

      // 2. Update Database
      const updatedUser = await User.update(req.session.user.id, {
        name,
        avatar: avatarUrl,
        bio
      });

      // 3. Update Session (PENTING!)
      // Biar perubahan langsung terasa tanpa relogin.
      req.session.user.name = updatedUser.name;
      req.session.user.avatar = updatedUser.avatar;

      req.session.success = "Profile updated successfully! 👤";
      res.redirect(`/profile/${req.session.user.id}`);
    } catch (error) {
      // Error handling...
    }
},
```

## Kesimpulan

Halaman Setting Profile ini mendemonstrasikan integrasi sistem yang komplit:
1.  **Database CRUD:** Mengambil dan Mengupdate data teks.
2.  **Cloud Storage:** Mengelola file media (foto) ke layanan eksternal (Supabase).
3.  **Session Management:** Sinkronisasi data antara Database dan Session browser agar UX-nya mulus.

Fitur ini memberi user kendali penuh atas identitas digital mereka di platform ini.
