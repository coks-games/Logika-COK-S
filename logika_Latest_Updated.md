# Logika Latest Updated: Bedah Tuntas Fitur "Game Terbaru" di Platform COKS

Halo! Dokumen ini akan membedah secara mendalam namun santai mengenai bagaimana fitur **"Latest Updated"** atau **"Game Terbaru"** bekerja di sistem kita. Kita akan bahas logika di balik layar, file-file yang terlibat, dan tentu saja bedah kodingannya.

Langsung saja, mari kita selami logikanya!

---

## 1. Logika Latest Updated (Penjelasan Santai)

Secara sederhana, logika "Latest Updated" ini bekerja seperti prinsip **tumpukan buku**.

Bayangkan kamu sedang menumpuk buku di meja. Setiap kali kamu beli buku baru, kamu taruh di posisi paling **atas**. Ketika temanmu bertanya "Mana buku yang paling baru?", kamu tinggal ambil buku dari tumpukan paling atas, kan?

Di dalam bahasa pemrograman dan database, konsep ini kita sebut **Sorting by Date Descending** (Urutkan Tanggal secara Menurun):
1.  **Database (Rak Buku):** Menyimpan semua data game beserta "stempel waktu" kapan game itu dibuat (`created_at`).
2.  **Query (Perintah Ambil):** "Tolong ambilkan daftar game, tapi tolong urutkan dari yang stempel waktunya paling muda (terbaru) ke yang paling tua."
3.  **Limit (Batasan):** "Dan tolong ambilkan 15 biji saja ya, jangan semuanya."

Jadi, sistem tidak mengacak data, melainkan menyusun ulang barisan data berdasarkan tanggal lahirnya. Yang lahir belakangan (junior), berdiri paling depan!

---

## 2. File-File yang Terlibat (The Suspects)

Berikut adalah daftar file yang bertanggung jawab atas fitur ini. Tanpa mereka, fitur ini tidak akan jalan.

1.  `models/Game.js`: Si Otak. Di sinilah logika query database (SQL) sesungguhnya berada. Dia yang ngobrol langsung sama database.
2.  `controllers/gameController.js`: Si Manajer. Dia yang mengatur alur, menerima request dari user, menyuruh Model (`Game.js`) mengambil data, lalu menyerahkannya ke halaman web.
3.  `routes/games.js` & `server.js`: Si Resepsionis. Mereka yang membukakan pintu untuk alamat URL `/` (Beranda) dan `/games` (Halaman List Game).

---

## 3. Fitur Utama: Implementasi Logika Pengambilan Data

Nah, ini bagian pentingnya. Ada dua tempat utama di mana logika ini dipakai, dan caranya sedikit berbeda meskipun tujuannya sama.

### A. Halaman Beranda (`/`)
Di halaman ini, kita ingin menampilkan **15 game terbaru** secara langsung sebagai *showcase*.

*   **Logic:** Langsung tembak function khusus `getLatest(15)`.
*   **Kenapa:** Karena di halaman depan itu statis, kita sudah tahu pasti kita cuma butuh data terbaru. Tidak perlu filter aneh-aneh. Cepat dan efisien.

### B. Halaman Games (`/games?sort=newest`)
Di halaman ini, logika lebih dinamis karena user bisa memfilter.

*   **Logic:** Menggunakan function `getAll()` yang super fleksibel. Function ini menerima parameter sorting.
*   **Alurnya:**
    1.  User buka `/games`.
    2.  Controller cek parameter `sort`. Kalau kosong, default-nya kita set ke `newest`.
    3.  Data dikirim ke Model.
    4.  Model menyusun query SQL: `ORDER BY created_at DESC`.

---

## 4. Bedah Source Code (The Secret Sauce)

Berikut adalah kode aslinya beserta penjelasan mahasiswa teknik informatika kenapa kode ini ditulis begini.

### Bagian 1: Otaknya (models/Game.js)

Ini adalah fungsi `getAll` yang menangani sorting dinamis, termasuk "newest".

```javascript
// File: models/Game.js

// Function getAll menerima banyak parameter, perhatikan 'sortBy'
getAll: async (search = "", category = "", sortBy = "newest", page = 1, limit = 12, type = "") => {
    // 1. Base Query: Ambil semua data game dan info author-nya
    let query = `
            SELECT games.*, users.name as author_name, users.avatar as author_avatar,
            COUNT(*) OVER() as total_count
            FROM games 
            JOIN users ON games.user_id = users.id 
            WHERE 1=1
        `;
    
    // ... (kode filter search & category dilewati biar fokus) ...

    // 2. Logika Sorting (Disinilah Kuncinya!)
    switch (sortBy) {
        case "oldest":
            query += " ORDER BY games.created_at ASC"; // ASC = Ascending (Tua -> Muda)
            break;
        case "popular":
            query += " ORDER BY games.play_count DESC";
            break;
        case "rating":
            query += " ORDER BY games.avg_rating DESC NULLS LAST";
            break;
        case "newest": // <--- INI DIA TARGET KITA
        default:      // Default juga ke sini kalau user aneh-aneh
            // DESC = Descending (Turun). Artinya dari angka besar (waktu kini) ke kecil (masa lalu).
            // Jadi yang baru dibuat akan muncul paling atas.
            query += " ORDER BY games.created_at DESC"; 
            break;
    }
    
    // 3. Eksekusi Query
    // ... (kode pagination & eksekusi) ...
},

// Function KHUSUS untuk Home Page yang lebih ringan
getLatest: async (limit = 10) => {
    try {
      // Query ini to-the-point. Gak pake basa-basi filter.
      // Langsung: SELECT, JOIN, ORDER BY created_at DESC, LIMIT.
      // EFEKTIF untuk performa karena query-nya simpel banget.
      const result = await db.query(`
        SELECT games.*, users.name as author_name 
        FROM games 
        JOIN users ON games.user_id = users.id
        ORDER BY created_at DESC
        LIMIT $1
      `, [limit]);
      return result.rows;
    } catch (error) {
      console.error("Error getting latest games:", error);
      return [];
    }
},
```

### Bagian 2: Manajernya (controllers/gameController.js)

Bagaimana controller memanggil fungsi di atas?

```javascript
// File: controllers/gameController.js

// 1. Controller untuk Halaman Beranda (Landing)
landing: async (req, res) => {
    try {
        // ... (kode lain ambil featured games) ...

        // Nah ini dia! Kita panggil getLatest dengan angka 15.
        // Artinya: "Tolong ambilkan 15 game paling baru dong!"
        const latestGames = await Game.getLatest(15); 

        // Lalu datanya dikirim ke view (ejs) biar bisa dilihat user
        res.render("index", {
            // ...
            latestGames, // variable ini nanti di-looping di HTML
            // ...
        });
    } catch (error) {
        // Error handling
    }
},

// 2. Controller untuk Halaman Games Index
index: async (req, res) => {
    try {
        // Ambil query dari URL, misal ?sort=newest
        // Kalau gak ada sort, kita biarkan logic di Model yang handle defaultnya.
        // Tapi di sini kita kirim explict "newest" di pemanggilan Game.getAll baris bawah.
        const { search, category, type, page = 1 } = req.query;
        const limit = 6; 
        const currentPage = parseInt(page) || 1;

        // Perhatikan parameter ke-3: "newest"
        // Kita memaksa urutannya newest secara default saat memanggil function ini
        // Kecuali nanti kita modif controller ini untuk membaca req.query.sort
        const { games, total } = await Game.getAll(search, category, "newest", currentPage, limit, type);
        
        // Render ke halaman
        res.render("games/index", {
            // ...
            games, // Data game yang sudah terurut rapi
            // ...
        });
    } catch (error) {
        // Error handling
    }
}
```

## Kesimpulan

Jadi intinya, fitur "Latest Updated" ini mengandalkan perintah SQL `ORDER BY created_at DESC`. Fitur ini krusial banget buat platform game kayak gini supaya user selalu disuguhi konten segar setiap kali masuk, bikin mereka penasaran "Wah ada game apa lagi nih barusan?".

Semoga penjelasan ini mencerahkan! Jangan ragu buat oprek kodingannya, karena cara belajar terbaik adalah dengan merusak kodingan lalu pusing cara benerinnya! Selamat coding!
