# Logika Halaman Search: Dibalik Fitur "Live Search" yang Responsif

Dokumen ini akan membedah cara kerja halaman **/search**, tempat di mana user bisa mencari game secara instan. Fitur ini menggunakan teknik **Live Search** (Pencarian Langsung), di mana hasil pencarian muncul *saat itu juga* ketika kamu mengetik, tanpa perlu tekan Enter atau refresh halaman.

Mari kita bongkar rahasianya!

---

## 1. Logika Live Search (Pencarian Instan)

Logika di sini berbeda dengan pencarian biasa di halaman `/games`. Di sini kita mengejar **Kecepatan** dan **Interaktivitas**.

Bayangkan kamu sedang mengetik di kamus digital:
1.  Kamu ketik "A". Kamus langsung membuka halaman huruf A.
2.  Kamu lanjut ketik "Ab". Kamus mempersempit ke kata-kata yang berawalan "Ab".
3.  Kamu ketik "Absurd". Kamus langsung menunjuk kata "Absurd".

Secara teknis, ini disebut **Prefix Matching** (Pencocokan Awalan).
*   **Input:** "sup"
*   **Query SQL:** `ILIKE 'sup%'` (Cari yang **depannya** sup...)
*   **Match:** "Super Mario", "Superman".
*   **Not Match:** "Batmansup", "Ketchup" (Karena "sup"-nya ada di belakang).

Kenapa cuma awalan? Karena ini fitur *Autocomplete*. Kita membantu user menyelesaikan apa yang sedang mereka ketik.

---

## 2. File-File yang Terlibat

Tiga komponen utama yang bekerja sama di sini:

1.  `routes/games.js`: Jalur API. Menyediakan alamat `/api/search` yang siap menerima request kapan saja.
2.  `controllers/gameController.js`: Si Satpam. Dia mengecek "Hei, kamu ngasih teks gak? Kalau kosong, ngapain saya cari?".
3.  `models/Game.js`: Si Pustakawan. Dia yang lari ke rak buku (database) dan mencari judul yang depannya cocok dengan inputan user.

---

## 3. Fitur Utama: Implementasi Logika Pengambilan Data

Prosesnya terjadi dalam hitungan milidetik:

1.  **Frontend (Browser)**: User mengetik sesuatu. Javascript (AJAX) mengirim hembusan sinyal ke server: `GET /games/api/search?q=katakunci`.
2.  **Backend (Server)**:
    *   Menerima `q=katakunci`.
    *   Menjalankan query `SELECT ... WHERE title ILIKE 'katakunci%'`.
    *   Mengembalikan hasil dalam format JSON (Data mentah, bukan HTML).
3.  **Frontend**: Menerima JSON tersebut dan mengubahnya menjadi daftar kartu game di layar.

---

## 4. Bedah Source Code (Bedah Saraf)

Mari kita lihat kodingannya. Ini lebih sederhana dari pencarian filter biasa, tapi sangat spesifik.

### Bagian 1: Controller (controllers/gameController.js)

Fokus controller di sini adalah validasi input dan response format JSON.

```javascript
// File: controllers/gameController.js

// 1. Render Halaman Search (Tampilan Awal)
// Ini cuma buat nampilin kotak input doang. Gak ada logic berat.
searchPage: async (req, res) => {
    const categories = await getCategories();
    res.render('games/search', {
      title: 'Search Games',
      categories
    });
},

// 2. API Live Search (Otak Utamanya)
searchApi: async (req, res) => {
    try {
      const { q } = req.query; // Ambil text yang diketik user
      
      // Validasi Cepat:
      // Kalau user belum ngetik apa-apa (atau cuma spasi), 
      // JANGAN ganggu database. Kasih array kosong aja.
      if (!q || q.length < 1) {
        return res.json([]);
      }

      // Panggil Model dengan text tersebut
      const games = await Game.searchByTitlePrefix(q);
      
      // Kirim balik data mentah (JSON) ke browser
      res.json(games);
    } catch (error) {
      console.error('Search API error:', error);
      res.status(500).json({ error: 'Search failed' });
    }
}
```

### Bagian 2: Model (models/Game.js)

Di sini letak perbedaan utamanya. Perhatikan tanda `%` yang hanya ada di **belakang**.

```javascript
// File: models/Game.js

searchByTitlePrefix: async (query) => {
    // Safety check lagi, kali aja controller lolos.
    if (!query) return [];
    
    // PERHATIKAN INI:
    // Kita pakai query + '%'
    // Artinya: "Cari judul yang BERAWALAN dengan query".
    // Beda dengan '%query%' yang artinya "Cari judul yang MENGANDUNG query".
    // Kenapa? Karena Auto-complete itu biasanya menebak kata selanjutnya, bukan kata yang di tengah.
    // Selain itu, 'string%' jauh lebih cepat secara performa index database daripada '%string%'.
    const result = await db.query(
      `SELECT id, title, slug, thumbnail_url, category, avg_rating 
       FROM games 
       WHERE title ILIKE $1 
       ORDER BY title ASC 
       LIMIT 10`, // Kita batasi 10 hasil saja biar UI gak kepanjangan
      [query + '%']
    );
    return result.rows;
}
```

## Kesimpulan

Halaman Search ini menggunakan pendekatan **API-First** untuk kontennya. Halaman utamanya (`searchPage`) sebenarnya "kosong" saat pertama dibuka. Isinya baru muncul secara dinamis berkat kerjasama antara Javascript di browser dan endpoint API (`searchApi`) di server.

Penggunaan `Prefix Match` (`string%`) adalah keputusan desain yang sengaja dibuat untuk mendukung perilaku *Autocomplete* yang natural dan cepat.
