# Logika Halaman Edit Game: Renovasi dan Koreksi Konten

Edit Game adalah tentang **Kewenangan** dan **Pembersihan**. Berbeda dengan Upload yang mulai dari nol, Edit harus mempertimbangkan data yang sudah ada dan file sampah yang mungkin tertinggal.

Dokumen ini menjelaskan bagaimana kita memastikan hanya pemilik game yang bisa mengedit, dan bagaimana sistem membersihkan file lama saat file baru diupload.

---

## 1. Logika Pengambilan Data (Preparation)

Sebelum form ditampilkan, ada dua pos pemeriksaan ketat:

1.  **Cek Eksistensi:** Cari game berdasarkan slug. Kalau tidak ada, 404/Redirect.
2.  **Cek Kepemilikan (Authorization):** Ini yang paling krusial.
    *   Sistem membandingkan `req.session.user.id` (User yang login) dengan `game.user_id` (Pemilik game).
    *   Jika beda: **TENDANG!** "You are not authorized". Jangan sampai user A mengacak-acak game milik user B.

**Masalah Parsing Data (JSON Fix):**
Di database, `tags` dan `download_config` disimpan sebagai teks JSON. Saat mau ditampilkan kembali di form edit, kita harus mengubahnya kembali menjadi objek/array.
Di sini ada logika *fallback* cerdas: Kalau format JSON-nya rusak (misal manual edit di DB), sistem akan mencoba memperbaikinya secara manual (split string by comma) agar tidak crash.

---

## 2. File-File yang Terlibat

1.  `routes/games.js`: Menangani rute `GET /:slug/edit` dan `PUT /:slug`.
2.  `controllers/gameController.js`: Menangani logika validasi kepemilikan dan penghapusan file lama.
3.  `models/Game.js`: Menangani query SQL `UPDATE`.

---

## 3. Fitur Utama: Implementasi Logika Perubahan

### A. Ubah Game Mode (Play <-> Download)
User berubah pikiran dari Web Game jadi PC Game?
*   **Logika Backend:** Controller akan melihat input `game_mode`.
*   Jika berubah jadi `download`: Sistem akan membaca input `provider_name` dan merakit ulang `download_config` baru.
*   Jika berubah jadi `play`: Sistem akan mengabaikan `download_config` dan fokus memvalidasi `github_url`.

### B. Ubah Price Model (Free <-> Pro)
*   **Paid ke Free:** User set `price_type` ke 'free'. Controller otomatis memaksa `price = 0` sebelum simpan ke database.
*   **Free ke Paid:** User set `price_type` ke 'paid'. Controller memvalidasi `price` harus >= Rp 1.000.

### C. Action Tombol "Save Changes" (Logika Pembersihan)
Ini bedanya Edit dengan Upload. Saat Edit, kita peduli lingkungan.

**Logika File Replacement:**
Jika user mengupload Thumbnail baru:
1.  Cek apakah game punya thumbnail lama?
2.  Jika ada, **HAPUS** file lama tersebut dari folder `/public/uploads/...` menggunakan `fs.unlinkSync`.
3.  Simpan file baru.
4.  Update nama file di database.

Kenapa? Supaya server tidak penuh dengan sampah gambar yang tidak terpakai (Orphaned Files).

---

## 4. Bedah Source Code (Bedah Renovasi)

Mari kita lihat fungsi `update` yang menangani logika penggantian file ini.

```javascript
// File: controllers/gameController.js
const fs = require('fs');
const path = require('path');

update: async (req, res) => {
    // 1. Ambil Data Game Lama
    const game = await Game.findBySlug(req.params.slug);
    
    // 2. Validasi Kepemilikan (Security)
    if (game.user_id !== req.session.user.id) {
       req.session.error = "You are not authorized";
       return res.redirect("/games");
    }

    const { title, game_mode, price_type, price, ...otherFields } = req.body;

    // 3. LOGIKA GANTI FILE (Thumbnail)
    let finalThumbnailUrl = game.thumbnail_url; // Default: Pake yang lama
    
    if (req.files && req.files['thumbnail']) {
       // User upload baru!
       
       // Hapus file lama jika ada di server lokal
       if (game.thumbnail_url && game.thumbnail_url.startsWith('/uploads/')) {
           const oldPath = path.join(__dirname, '../public', game.thumbnail_url);
           // Cek dulu filenya beneran ada gak biar gak error
           if (fs.existsSync(oldPath)) {
               fs.unlinkSync(oldPath); // DELETE file lama
           }
       }
       
       // Set URL baru
       finalThumbnailUrl = '/uploads/thumbnails/' + req.files['thumbnail'][0].filename;
    }

    // 4. LOGIKA HARGA (Price Reset)
    let finalPrice = parseInt(price) || 0;
    if (price_type === 'free') {
        finalPrice = 0; // Paksa jadi 0 kalau free
    }

    // 5. UPDATE DATABASE
    await Game.update(req.params.slug, {
       title,
       thumbnail_url: finalThumbnailUrl,
       price: finalPrice,
       // ... field lainnya
    });
    
    // ... Redirect success
},
```

### JSON Fallback Logic (saat Edit - Display)

Potongan kode ini penting untuk mencegah halaman crash jika data di database korup.

```javascript
// File: controllers/gameController.js (fungsi edit)

// START: KOREKSI FIX UNTUK JSON PARSING
game.parsedTags = [];
// Cek datanya string dan tidak kosong?
if (game.tags && typeof game.tags === "string" && game.tags.trim().length > 0) {
    try {
        // Coba cara normal
        game.parsedTags = JSON.parse(game.tags);
    } catch (e) {
        // JIKA JSON RUSAK: Lakukan cara manual (Fallback)
        // Pisahkan berdasarkan koma
        game.parsedTags = game.tags.split(",").map(t => t.trim());
    }
}
```

## Kesimpulan

Halaman Edit Game mengajarkan kita tentang tanggung jawab data.
1.  **Security Responsibility:** Memastikan hanya pemilik yang bisa akses.
2.  **Storage Responsibility:** Membersihkan file sampah saat update.
3.  **Data Integrity:** Self-healing mechanism saat membaca format data yang mungkin rusak (JSON parsing).

Fitur ini melengkapi siklus hidup (Lifecycle) sebuah konten di COKS.
