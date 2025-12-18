# Logika Populer Category: Bedah Fitur "Kategori Terlaris" di COKS

Dokumen ini mengupas tuntas fitur **"Populer Category"** yang ada di halaman depan. Fitur ini cukup pintar karena sifatnya dinamis: dia bisa berubah-ubah sendiri tergantung tren apa yang sedang dimainkan user. Kalau bulan ini semua orang main game "Action", maka section ini otomatis berubah jadi "Action". Keren kan?

Mari kita bedah logikanya!

---

## 1. Logika Populer Category (Penjelasan Santai)

Logika fitur ini bekerja dalam **dua tahap** (2-Step Process). Bayangkan kompetisi antar kelas di sekolah:

1.  **Tahap 1: Cari Juara Umum (Aggregating)**
    Sistem akan mengumpulkan nilai (jumlah main/`play_count`) dari *semua* game. Lalu nilai itu dikelompokkan berdasarkan kategorinya.
    *   Game A (Action): 100 main
    *   Game B (Action): 50 main
    *   **Total Action: 150 main**
    *   Game C (Puzzle): 200 main
    *   **Total Puzzle: 200 main**
    
    Di kasus ini, **Puzzle** menang. Sistem mencatat: "Oke, kategori terpopuler saat ini adalah PUZZLE".

2.  **Tahap 2: Tampilkan Anggota Sang Juara (Filtering)**
    Setelah tahu pemenangnya adalah Puzzle, sistem kemudian mengambil daftar game yang *hanya* ber-tag Puzzle. Game-game ini kemudian diurutkan lagi dari yang paling populer, supaya yang muncul di depan adalah game Puzzle terbaik.

---

## 2. File-File yang Terlibat

Dua aktor utama di sini masih sama, karena pola MVC (Model-View-Controller) kita konsisten:

1.  `models/Game.js`: Tempat query SQL yang melakukan perhitungan berat (menjumlahkan play_count) dan pengambilan data.
2.  `controllers/gameController.js`: Pengatur strategi. Dia yang menyuruh Model "Cari dulu kategorinya, kalau sudah ketemu, baru ambil gamenya".

---

## 3. Fitur Utama: Implementasi Logika Pengambilan Data (Halaman Beranda '/')

Di halaman Beranda, kita tidak ingin membebani user dengan memilih kategori. Kita langsung suguhi apa yang lagi *hype*.

*   **Logic:** Chained Execution (Eksekusi Berantai).
*   **Alurnya:**
    1.  Controller memanggil fungsi `getMostPopularCategory()`.
    2.  Database menghitung total `play_count` per kategori dan mengembalikan **satu nama kategori** pemenang (misal: "Adventure").
    3.  Controller menerima nama "Adventure".
    4.  Controller lanjut memanggil fungsi kedua `getByCategory('Adventure')`.
    5.  Database mengambil game-game Adventure.
    6.  Data dikirim ke frontend untuk ditampilkan dengan judul besar "Populer Category: Adventure".

---

## 4. Bedah Source Code (Deep Dive)

Berikut adalah kode lengkapnya. Perhatikan penggunaan `SUM` dan `GROUP BY` yang menjadi inti dari fitur ini.

### Bagian 1: Model (models/Game.js)

```javascript
// File: models/Game.js

module.exports = {
  // ... fungsi lain ...

  /**
   * Fungsi 1: Mencari Nama Kategori Terpopuler
   * Ini langkah krusial untuk menentukan "Judul" section.
   */
  getMostPopularCategory: async () => {
    try {
      // Query SQL Analitik:
      // 1. SELECT category: Kita mau ambil nama kategorinya.
      // 2. SUM(play_count): Kita jumlahkan play_count semua game di kategori itu.
      // 3. GROUP BY category: Kita kelompokkan perhitungan per kategori.
      // 4. ORDER BY total_plays DESC: Urutkan dari jumlah main terbanyak.
      // 5. LIMIT 1: Ambil JUARA 1 saja.
      const result = await db.query(`
        SELECT category, SUM(play_count) as total_plays 
        FROM games 
        WHERE category IS NOT NULL 
        GROUP BY category 
        ORDER BY total_plays DESC 
        LIMIT 1
      `);
      
      // Jika ada hasil, kembalikan nama kategorinya (misal: "RPG").
      // Jika database kosong, kembalikan null.
      return result.rows.length > 0 ? result.rows[0].category : null;
    } catch (error) {
      console.error("Error getting popular category:", error);
      return null;
    }
  },

  /**
   * Fungsi 2: Mengambil Game berdasarkan Kategori
   * Setelah tahu kategorinya, kita ambil isi game-nya pakai ini.
   */
  getByCategory: async (category, limit = 12) => {
    try {
      // Query ini standar tapi spesifik:
      // WHERE category = $1: Filter mutlak hanya untuk kategori tersebut.
      // ORDER BY play_count DESC: Tampilkan game terpopuler di kategori itu duluan.
      const result = await db.query(`
        SELECT games.*, users.name as author_name
        FROM games 
        JOIN users ON games.user_id = users.id
        WHERE category = $1
        ORDER BY play_count DESC
        LIMIT $2
      `, [category, limit]);
      
      return result.rows; 
    } catch (error) {
      console.error("Error getting games by category:", error);
      return [];
    }
  },
};
```

### Bagian 2: Controller (controllers/gameController.js)

Di sini kita lihat bagaimana Controller merangkai dua fungsi di atas menjadi satu alur logika yang utuh.

```javascript
// File: controllers/gameController.js

landing: async (req, res) => {
    try {
      // ... (kode featured games dll) ...

      // [STEP 1] Cek dulu, kategori apa yang lagi ngetren?
      // Kita panggil fungsi analitik tadi.
      const popularCategoryName = await Game.getMostPopularCategory();
      
      let popularCategoryGames = [];

      // [STEP 2] Validasi & Eksekusi Lanjutan
      // Kita harus cek 'if (popularCategoryName)' untuk menghindari error
      // kalau database masih kosong melompong.
      if (popularCategoryName) {
        // Kalau ketemu (misal: "Strategy"), langsung gas ambil game-game Strategy.
        // Kita ambil 12 biji biar pas grid layout-nya.
        popularCategoryGames = await Game.getByCategory(popularCategoryName, 12); 
      }

      // [STEP 3] Kirim ke View
      // Kita kirim dua variabel penting:
      // 1. Namanya (untuk judul "Populer Category: [Name]")
      // 2. Isinya (untuk ditampilkan sebagai card game)
      res.render("index", {
        // ...
        popularCategoryName, 
        popularCategoryGames, 
        // ...
      });

    } catch (error) {
        // Handle error...
    }
},
```

## Kesimpulan

Fitur **Populer Category** ini adalah contoh cerdas dari penggunaan data untuk driving content. Kita tidak manual setting "Minggu ini tema Action ya!", tapi membiarkan user behavior yang menentukan.

Logikanya sederhana: **Agregasi dulu (SUM & GROUP BY), baru Filter kemudian (WHERE)**. Pola ini sangat umum dipakai di dashboard analitik atau fitur rekomendasi.

Sekarang kamu paham kan kenapa halaman depan bisa berubah sendiri kategorinya? Itu bukan sihir, itu SQL!
