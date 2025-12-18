# Logika Upload Game: Jantung Ekosistem COKS

Ini adalah fitur paling kompleks di seluruh aplikasi. Di sinilah konten lahir. Halaman **Upload Game** (`/games/create/new`) bukan sekadar form, tapi mesin pemroses metadata yang fleksibel.

Dokumen ini akan membongkar bagaimana sistem menangani dua jenis game (Browser vs PC) dan dua model bisnis (Free vs Pro) dalam satu form yang sama.

---

## 1. Logika Tampilan & Game Modes

Formulir ini dinamis. Pilihan user di satu kolom akan mempengaruhi kolom lainnya.

### A. Game Mode: Play vs Download
Saat user memilih mode, backend sebenarnya hanya menerima string `'play'` atau `'download'`. Tapi dampaknya besar pada validasi:

1.  **Play (Browser Mode):**
    *   **Wajib:** `github_url`.
    *   **Logika:** Game ini akan dimainkan via iFrame. URL harus mengarah ke *GitHub Pages* atau hosting statis.
    *   **Auto-Correction:** Sistem pintar di backend akan mencoba "memperbaiki" link GitHub biasa menjadi link GitHub Pages yang valid agar bisa dimainkan.

2.  **Download (PC Mode):**
    *   **Opsional:** `github_url` (Boleh kosong).
    *   **Wajib:** Konfigurasi Download (Link download eksternal, ukuran file, OS target).
    *   **Logika:** Game ini adalah file `.exe` atau `.zip`. COKS tidak menghosting file game besar (mahal bandwidth!), jadi kita hanya menyimpan *link-nya*.

### B. Price Model: Free vs Pro
1.  **Free:** Harga otomatis diset `0`.
2.  **Pro (Paid):**
    *   User wajib isi harga.
    *   **Validasi:** Harga minimal Rp 1.000. Kalau user iseng masukin Rp 50, sistem akan menolak.

---

## 2. File-File yang Terlibat

1.  `routes/games.js`: Jalur distribusi. Di sini kita pasang middleware `multer` untuk menangkap file yang diupload (Thumbnail, Icon, Video).
2.  `controllers/gameController.js`: Otak pemrosesan. Menangani validasi rumit dan logika "Slug Unik".
3.  `models/Game.js`: Tukang simpan. Mengeksekusi query SQL `INSERT`.

---

## 3. Fitur Utama: Action Tombol "Upload Game"

Saat tombol ditekan, inilah perjalanan datanya:

1.  **File Hosting (Multer):** Thumbnail, Icon, dan Video tidak masuk database. Mereka disimpan ke folder `/public/uploads/...`. Database cuma pegang *nama file*-nya saja.
2.  **Slug Generation:**
    *   Judul: "Super Mario" -> Slug: `super-mario`.
    *   Cek Database: "Ada gak `super-mario`?"
    *   Jika ada -> Ubah jadi `super-mario-1`.
    *   Cek lagi -> Masih ada? Ubah jadi `super-mario-2`. (Looping sampai unik).
3.  **Mode Detection:**
    *   Sistem membaca input `game_mode`.
    *   Jika `download`, sistem merakit JSON `download_config` dari inputan array (Link 1, Link 2, dst).
4.  **Save:** Semua data yang sudah rapi disimpan ke PostgreSQL.

---

## 4. Bedah Source Code (Bedah Logika Kompleks)

Mari kita lihat controller `store` yang menangani semua kerewetan ini.

```javascript
// File: controllers/gameController.js

store: async (req, res) => {
    try {
      const { 
        title, github_url, game_mode, price_type, price, // ... input lainnya
        provider_name, provider_url // Array untuk link download
      } = req.body;

      // 1. GENERATE SLUG UNIK
      // Slug adalah URL cantik. Kita loop sampai nemu yang belum dipake.
      let slug = slugify(title, { lower: true, strict: true });
      let slugExists = await Game.slugExists(slug);
      let counter = 1;

      while (slugExists) {
        slug = `${slugify(title, { lower: true, strict: true })}-${counter}`;
        slugExists = await Game.slugExists(slug);
        counter++;
      } 
      
      // 2. DETEKSI TIPE GAME
      let game_type = "playable";
      if (game_mode === 'download') {
          game_type = "download";
      }

      // 3. KONSTRUKSI DOWNLOAD CONFIG (JSON)
      // Kalau mode download, kita rakit data link-nya jadi format JSON.
      let download_config = null;
      if (game_type === 'download' && provider_name) {
          // Logika untuk menangani apakah inputnya Single atau Array
          // (Karena form HTML kadang kirim string kalau cuma 1 input, array kalau >1)
          const links = [];
          if (Array.isArray(provider_name)) {
              for (let i = 0; i < provider_name.length; i++) {
                  links.push({ name: provider_name[i], url: provider_url[i] });
              }
          } else {
              links.push({ name: provider_name, url: provider_url });
          }
          
          // Stringify biar bisa masuk kolom TEXT di database
          download_config = JSON.stringify({
              links: links,
              // ... info lain (size, os)
          });
      }

      // 4. PRICE HANDLING
      // Pastikan kalau free, harganya 0.
      const finalPrice = (price_type === 'free') ? 0 : (parseInt(price) || 0);

      // 5. SIMPAN KE DATABASE
      await Game.create({
        slug, // Slug unik tadi
        game_type,
        download_config, // JSON siap simpan
        price: finalPrice,
        // ... data lainnya (thumbnail URL dari req.files)
      });

      req.session.success = "Game uploaded successfully! 🎮";
      res.redirect("/games");

    } catch (error) {
      // Error handling...
    }
},
```

## Kesimpulan

Fitur Upload Game ini adalah contoh **Polymorphic Input Handling**.
Satu form melayani dua dunia (Web Game & PC Game).
Kuncinya ada di Controller yang "pintar" memilah:
*   Kalau Web Game -> Fokus validasi `github_url`.
*   Kalau PC Game -> Fokus rakit JSON `download_config`.

Dengan pendekatan ini, kita tidak perlu membuat dua halaman upload terpisah. Efisien dan User Friendly!
