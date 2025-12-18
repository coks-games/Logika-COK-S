# Dokumentasi Teknis: Fitur Game Khusus PC (Download)

Dokumen ini menguraikan logika sistem yang menangani kategori **Game Khusus PC (Download)**. Fokus utama dokumen ini adalah menjelaskan bagaimana sistem memfilter, mengambil, dan menyajikan game yang bertipe unduhan (*downloadable*) baik pada halaman utama maupun halaman pencarian.

---

## 1. Ikhtisar Fungsional

Kategori "Game Khusus PC" dirancang untuk memisahkan game berbasis *web-browser* (HTML5) dengan game yang memerlukan instalasi (file .exe, .zip, .rar). Sistem menggunakan penanda khusus (`game_type` atau `download_config`) untuk mengidentifikasi game jenis ini secara otomatis.

---

## 2. Struktur Berkas Terkait

Logika fitur ini tersebar pada tiga lapisan arsitektur utama:

1.  **`controllers/gameController.js`**: Mengontrol logika permintaan data dari *client* (browser).
2.  **`models/Game.js`**: Menangani logika *query* database untuk memfilter tipe game.
3.  **`views/index.ejs` & `views/games/index.ejs`**: Menampilkan antarmuka daftar game kepada pengguna.

---

## 3. Analisis Source Code & Logika Data

Berikut adalah penjelasan teknis mengenai implementasi kode pada setiap bagian.

### A. Tampilan Beranda (Landing Page `/`)

Pada halaman utama, sistem menampilkan 8 game PC terbaru dalam sebuah *section* khusus.

**Logika Pengambilan Data:**
Controller memanggil fungsi khusus `Game.getType('download', 8)` untuk mendapatkan data secara spesifik dan efisien.

**Implementasi Controller (`controllers/gameController.js`):**
```javascript
landing: async (req, res) => {
  // ...
  // Mengambil 8 game tipe 'download' untuk section PC Games
  const pcGames = await Game.getType('download', 8);
  // ...
}
```

**Implementasi Model (`models/Game.js`):**
Fungsi `getType` menggunakan logika *OR* untuk memastikan tidak ada game unduhan yang tertinggal.

```javascript
getType: async (type, limit = 8) => {
    try {
      let query = `
        SELECT games.*, users.name as author_name
        FROM games 
        JOIN users ON games.user_id = users.id
        WHERE 
      `;
      const params = [];

      // LOGIKA FILTER KHUSUS DOWNLOAD
      if (type === 'download') {
          // Sistem memeriksa DUA kondisi:
          // 1. game_type secara eksplisit diset 'download'
          // 2. ATAU memiliki konfigurasi file download (download_config tidak kosong)
          query += `(game_type = 'download' OR download_config IS NOT NULL)`;
          params.push(limit);
      } else {
          // Logika untuk tipe lain
          query += `game_type = $1`;
          params.push(type);
          params.push(limit);
      }

      query += ` ORDER BY created_at DESC LIMIT $${params.length}`;

      const result = await db.query(query, params);
      return result.rows;
    } catch (error) { ... }
},
```

**Alasan Penggunaan Kode Tersebut:**
Kita memeriksa `download_config IS NOT NULL` sebagai pengaman (*fallback*). Terkadang, status `game_type` mungkin belum terupdate, namun jika game tersebut memiliki data file unduhan, sistem tetap harus menganggapnya sebagai game PC. Ini menjamin akurasi data yang ditampilkan.

---

### B. Halaman Pencarian Game (`/games?type=download`)

Saat pengguna mengklik "See All" atau memfilter pencarian, logika penanganan sedikit berbeda karena terintegrasi dengan filter lain (seperti kategori atau pencarian nama).

**Logika Pengambilan Data:**
Sistem menggunakan fungsi general `Game.getAll` yang dinamis, menerima parameter `type` dari URL *query string*.

**Implementasi Controller (`controllers/gameController.js`):**
```javascript
index: async (req, res) => {
    // Menangkap parameter 'type' dari URL (misal: ?type=download)
    const { search, category, type, page = 1 } = req.query;
    
    // ...
    
    // Memanggil Model dengan filter type
    const { games, total } = await Game.getAll(search, category, "newest", currentPage, limit, type);
    
    // ...
}
```

**Implementasi Model (`models/Game.js`):**
Fungsi `getAll` bertugas menyusun *query* SQL secara dinamis (Dynamic Query Building).

```javascript
getAll: async (search = "", category = "", sortBy = "newest", page = 1, limit = 12, type = "") => {
    // Query Dasar
    let query = `SELECT games.* ... FROM games ... WHERE 1=1`;
    
    // ... (kode filter search & category) ...

    // FILTER TIPE GAME
    if (type) {
      if (type === 'download') {
          // Logika konsisten: Cek game_type ATAU keberadaan konfigurasi download
          query += ` AND (games.game_type = 'download' OR games.download_config IS NOT NULL)`;
      } else {
          params.push(type);
          query += ` AND games.game_type = $${paramCount}`;
          paramCount++;
      }
    }

    // ... (kode sorting & pagination) ...
}
```

**Pentingnya Logika Dynamic Query:**
Pendekatan `WHERE 1=1` diikuti dengan `AND ...` memungkinkan sistem untuk menggabungkan berbagai filter tanpa merusak sintaks SQL. Contoh hasil query yang terbentuk secara otomatis:
`SELECT * FROM games WHERE 1=1 AND category = 'Action' AND (game_type = 'download' OR ...)`

---

## Kesimpulan

Sistem Game Khusus PC dibangun dengan logika filter yang **inklusif**. Sistem tidak hanya bergantung pada label `game_type`, tetapi juga memverifikasi keberadaan data teknis (`download_config`). Hal ini memastikan bahwa setiap game yang memiliki fail unduhan akan selalu muncul di kategori PC, memberikan pengalaman eksplorasi yang lengkap bagi pengguna.
