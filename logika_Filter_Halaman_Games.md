# Logika Filter Halaman Games: Bedah Fitur "Pencarian Cerdas" COKS

Dokumen ini akan membedah bagaimana halaman `/games` bisa menampilkan game yang sesuai dengan kemauan user. Entah itu user lagi cari game "Action", atau mencari game yang namanya "Mario", semua diatur di sini.

Langsung saja kita bahas logikanya!

---

## 1. Logika Filter Games (Penjelasan Santai)

Logika filter ini menggunakan prinsip **"Dynamic Query Building"** atau Membangun Query Secara Dinamis.

Bayangkan kamu sedang memesan di warkop:
1.  **Standar:** "Bang, pesan kopi." -> Sistem mengambil semua game.
2.  **Filter 1:** "Bang, pesan kopi **hitam**." (`WHERE category = 'hitam'`)
3.  **Filter 2:** "Bang, pesan kopi **hitam** yang **dingin**." (`WHERE category = 'hitam' AND type = 'dingin'`)

Jadi, kode kita tidak menulis satu per satu kemungkinan pesanan. Kode kita pintar, dia akan melihat apa saja request dari user, lalu menempelkan syarat-syarat (conditions) itu satu per satu ke dalam kalimat perintah database (SQL).

Semakin banyak user minta syarat, semakin panjang kalimat SQL-nya, dan semakin spesifik hasil yang keluar.

---

## 2. File-File yang Terlibat

Seperti biasa, ini kerja tim antara:

1.  `models/Game.js`: Si Eksekutor. Dia yang menyusun serpihan syarat dari user menjadi satu kalimat SQL utuh dan menjalankannya.
2.  `controllers/gameController.js`: Si Penerjemah. Dia membaca URL browser (misal `?category=Action&search=Mario`), membersihkannya, dan menyuapkannya ke Model.
3.  `views/games/index.ejs` (Frontend): Si Penampil. Tempat tombol-tombol filter berada yang akan mengubah URL ketika diklik.

---

## 3. Fitur Utama: Implementasi Logika Pengambilan Data (Halaman '/games')

Di halaman `/games`, data yang diambil sangat bergantung pada apa yang ada di **URL Query Parameters** (tulisan di belakang tanda `?` pada alamat web).

*   **Logic:** Conditional Appending (Penambahan Bersyarat).
*   **Alurnya:**
    1.  User klik kategori "Action". URL berubah jadi `/games?category=Action`.
    2.  Controller sadar ada request masuk dengan parameter `category`.
    3.  Controller panggil Model: `Game.getAll(..., category='Action', ...)`.
    4.  Model memulai kalimat SQL dasar: `SELECT * FROM games WHERE 1=1`.
        *   *Kenapa `WHERE 1=1`? Ini trik programmer supaya kita bisa enak nambahin `AND ...` depannya tanpa takut error syntax.*
    5.  Karena ada request kategori, Model nambahin: `AND category LIKE '%Action%'`.
    6.  Query dijalankan, hasil dikembalikan.

---

## 4. Bedah Source Code (The Engine Room)

Mari kita lihat bagaimana logika dinamis ini ditulis dalam kode JavaScript.

### Bagian 1: Controller (controllers/gameController.js)

Tugas controller di sini adalah "Menangkap" dan "Meneruskan".

```javascript
// File: controllers/gameController.js

index: async (req, res) => {
  try {
    // 1. Menangkap Parameter
    // Kita ambil nilai search, category, type, dan page dari URL.
    // req.query adalah object bawaan Express yang menampung data URL.
    const { search, category, type, page = 1 } = req.query;
    
    // 2. Setting Default
    // Kita tentukan satu halaman isinya 6 game.
    const limit = 6; 
    const currentPage = parseInt(page) || 1;

    // 3. Meneruskan ke Model
    // Perhatikan kita kirim semua parameter itu ke function getAll milik Model.
    // Kita juga kirim "newest" sebagai default sorting.
    const { games, total } = await Game.getAll(search, category, "newest", currentPage, limit, type);
    
    // ... (kode logic diskon & render view) ...
    
    res.render("games/index", {
      // Kita kirim balik data query ke view supaya input search/filter tidak reset (tetap terisi)
      search: search || "",
      category: category || "",
      type: type || "",
      // ...
    });
  } catch (error) {
    // Error handling...
  }
},
```

### Bagian 2: Model (models/Game.js)

Di sinilah struktur kalimat SQL dibangun bata demi bata.

```javascript
// File: models/Game.js

// Function ini menerima banyak parameter opsional (ada default value = "")
getAll: async (search = "", category = "", sortBy = "newest", page = 1, limit = 12, type = "") => {
    
    // 1. Base Query (Fondasi)
    // WHERE 1=1 adalah trik supaya kita bisa langsung sambung dengan AND di bawahnya.
    // SELECT COUNT(*) OVER() teknik pintar buat hitung total data tanpa 2x query.
    let query = `
            SELECT games.*, users.name as author_name, users.avatar as author_avatar,
            COUNT(*) OVER() as total_count
            FROM games 
            JOIN users ON games.user_id = users.id 
            WHERE 1=1
        `;
    
    // Kita pakai array 'params' untuk menyimpan value, ini teknik 'Parameter Binding'.
    // Tujuannya: MENCEGAH SQL INJECTION (hacking database).
    const params = [];
    let paramCount = 1;

    // 2. Filter Search (Jika ada)
    if (search) {
      // Masukkan text pencarian ke koper params
      params.push(`%${search}%`); 
      // Tempelkan logika pencarian ke query string
      query += ` AND (games.title ILIKE $${paramCount} OR games.description ILIKE $${paramCount})`;
      paramCount++; // Naikkan nomor antrian parameter
    }

    // 3. Filter Kategori (Jika ada)
    if (category) {
      params.push(`%${category}%`);
      query += ` AND games.category ILIKE $${paramCount}`;
      paramCount++;
    }

    // 4. Filter Type/Jenis (Jika ada)
    // Khusus PC Games (download), logikanya agak unik.
    if (type) {
      if (type === 'download') {
          // Game download bisa ditandai dari tipe 'download' ATAU punya config download.
          query += ` AND (games.game_type = 'download' OR games.download_config IS NOT NULL)`;
          // Note: Disini kita tidak push params baru karena nilainya hardcode 'download'.
      } else {
          params.push(type);
          query += ` AND games.game_type = $${paramCount}`;
          paramCount++;
      }
    }

    // 5. Sorting (Pengurutan)
    // Switch case untuk menentukan akhiran query (ORDER BY ...)
    switch (sortBy) {
        case "oldest":
            query += " ORDER BY games.created_at ASC";
            break;
        case "popular":
            query += " ORDER BY games.play_count DESC";
            break;
        // ... case lainnya ...
        default:
            query += " ORDER BY games.created_at DESC";
            break;
    }
    
    // 6. Pagination (Halaman)
    // Hitung offset: (Halaman 2 - 1) * 6 = Lewati 6 data pertama.
    const offset = (page - 1) * limit;
    params.push(limit);
    params.push(offset);
    query += ` LIMIT $${paramCount} OFFSET $${paramCount + 1}`;

    // 7. Eksekusi Final
    const result = await db.query(query, params);
    
    return {
        games: result.rows,
        // Ambil total data dari baris pertama (kalau ada)
        total: result.rows.length > 0 ? parseInt(result.rows[0].total_count) : 0
    };
},
```

## Kesimpulan

Sistem filter di halaman Games ini **fleksibel**. Kita tidak membuat query terpisah untuk "Cari Judul", lalu query lain lagi untuk "Cari Kategori". Kita membuat **Satu Query Raksasa** yang bisa membesar dan mengecil sesuai kebutuhan user.

Cara ini sangat efisien dan mudah dikelola (maintainable). Kalau besok kita mau tambah filter baru (misal: "Berdasarkan Harga"), kita tinggal tambah satu blok `if` baru di Model, tanpa merusak fitur lainnya. Keren, kan?
