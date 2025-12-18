# Logika Reviews & Ratings: Suara Jujur Para Gamer

Fitur ini memberi nyawa pada game. **Reviews & Ratings** membiarkan user memberi bintang (1-5) dan komentar tertulis. Ini bukan sekadar angka, tapi algoritma sosial yang menentukan apakah sebuah game layak dimainkan atau tidak.

Dokumen ini akan mengupas bagaimana satu klik rating mengubah nasib sebuah game di database.

---

## 1. Logika Pengambilan Data (Review yang Tampil)

Review tidak berdiri sendiri. Mereka menempel pada halaman Game Detail (`/games/:slug`).
Jadi, logika pengambilan datanya terjadi di `gameController.show`, BUKAN di `ratingController`.

Logikanya:
1.  **Load Game:** Ambil data game utama (Judul, Deskripsi, dll).
2.  **Load Reviews:** Ambil semua review untuk game ID tersebut. JOIN-kan dengan tabel `users` supaya kita tahu siapa yang nulis, dan apa avatar mereka.
3.  **Cek Review Saya:** Cek apakah user yang sedang login *sudah pernah* memberi rating?
    *   Jika SUDAH: Form berubah jadi "Edit Rating Anda".
    *   Jika BELUM: Form tampil bersih "Rate This Game".
4.  **Hitung Statistik:** Berapa rata-rata bintangnya? Berapa total yang ngevote? (`AVG(rating)`).

---

## 2. File-File yang Terlibat

1.  `routes/ratings.js`: Jalur khusus untuk *Submit* dan *Delete* rating.
2.  `controllers/ratingController.js`: Wasit. Dia yang menerima input bintang dari user.
3.  `models/Rating.js`: Juru Tulis. Dia punya query canggih (`UPSERT`) untuk menangani rating ganda.
4.  `models/Game.js`: Menerima update rata-rata rating (`avg_rating`) agar sorting populer bekerja cepat.

---

## 3. Fitur Utama: Implementasi Aksi Tombol

### Apa yang terjadi saat tombol "Submit Rating" ditekan?
User memilih 5 Bintang dan mengetik "Game ini mantap!". Klik Submit.

**Logika Backend (POST /ratings):**
1.  **Validasi:** Pastikan `game_id` ada dan `rating` adalah angka 1-5.
2.  **UPSERT (Update or Insert):** Ini logika kuncinya!
    *   Sistem tidak cek satu-satu "User ini udah review belum ya?". Kelamaan.
    *   Sistem langsung pakai perintah SQL: "**MASUKKAN** data ini. **TAPI KALAU** user ini udah pernah review game ini sebelumnya, **UPDATE** aja review lamanya."
    *   Efisiensi tingkat dewa. Satu query menangani dua skenario (Rating Baru & Ganti Rating).
3.  **Recalculate Average:** Setelah rating masuk, kita hitung ulang nilai rata-rata game tersebut dan simpan di tabel `games`. Kenapa? Biar pas sorting "Top Rated", database gak perlu hitung ulang jutaan baris tiap detik.

---

## 4. Bedah Source Code (Bedah UPSERT)

Teknik `ON CONFLICT` di sini adalah juara utamanya.

### Bagian 1: Controller (controllers/ratingController.js)

```javascript
// File: controllers/ratingController.js

store: async (req, res) => {
    try {
      const { game_id, rating, review } = req.body;
      const user_id = req.session.user.id;

      // 1. Panggil Model dengan teknik UPSERT
      await Rating.createOrUpdate({
        user_id,
        game_id,
        rating: parseInt(rating), // Pastikan jadi integer
        review: review || null,
      });

      // 2. Update Cache Rata-rata Rating di Tabel Game
      // Ini penting buat performa sorting di halaman depan.
      await Game.updateAverageRating(game_id);

      req.session.success = "Rating submitted successfully! ⭐";
      // Redirect balik ke halaman game tadi
      res.redirect('back'); 
    } catch (error) {
      // Error handling...
    }
},
```

### Bagian 2: Model (models/Rating.js)

Perhatikan query SQL yang "smart" ini.

```javascript
// File: models/Rating.js

createOrUpdate: async (ratingData) => {
    const { user_id, game_id, rating, review } = ratingData;

    // QUERY SAKTI: INSERT ... ON CONFLICT
    const result = await db.query(
      `INSERT INTO ratings (user_id, game_id, rating, review, created_at, updated_at) 
       VALUES ($1, $2, $3, $4, NOW(), NOW())
       
       ON CONFLICT (user_id, game_id) 
       DO UPDATE SET 
          rating = $3, 
          review = $4, 
          updated_at = NOW()
       
       RETURNING *`,
      [user_id, game_id, rating, review]
    );

    return result.rows[0];
},
```
*   `ON CONFLICT (user_id, game_id)`: Artinya, jika kombinasi User A dan Game B sudah ada di database...
*   `DO UPDATE`: ...Jangan error, tapi update saja isinya.

## Kesimpulan

Sistem Reviews & Ratings ini didesain anti-duplikasi. Satu user hanya bisa punya satu suara per game. Logika `UPSERT` memastikan data tetap konsisten tanpa perlu kode validasi if-else yang panjang di backend. Cerdas, ringkas, dan cepat.
