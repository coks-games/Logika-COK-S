# Logika Statistics Section Halaman About: Mengapa Angka Bisa Bicara?

Dokumen ini membedah bagaimana kita memunculkan angka-angka keren di halaman `/about` (seperti Total Games, Active Gamers, dll). Sekilas terlihat sederhana, "Ah cuma nampilin angka". Padahal di belakang layar, ada teknik optimalisasi yang sangat krusial agar website tidak lemot.

Yuk kita bongkar logikanya!

---

## 1. Logika Parallel Processing (Konsep Utama)

Kunci utama dari section ini adalah **Parallel Execution** (Eksekusi Paralel).

Bayangkan kamu memesan paket lengkap di restoran cepat saji (Burger, Kentang, Cola, Es Krim).
*   **Cara Lambat (Sequential):** Pelayan ambil Burger dulu -> Balik ke dapur ambil Kentang -> Balik lagi ambil Cola -> Balik lagi ambil Es Krim. Lama banget, keburu lapar!
*   **Cara Cepat (Parallel):** Pelayan bawa nampan besar dan mengambil Burger, Kentang, Cola, dan Es Krim **sekaligus dalam satu perjalanan**.

Di kode kita, kita menggunakan `Promise.all()`. Kita menyuruh database mengerjakan 4 tugas hitungan secara BERSAMAAN, bukan antri satu-satu. Hasilnya? Loading page jadi super ngebut!

---

## 2. File-File yang Terlibat

1.  `routes/pages.js`: Pusat kendali. Di file inilah logika tersebut ditulis karena halaman About termasuk "halaman statis" (bukan CRUD game), jadi logic-nya nempel langsung di router.
2.  `config/database.js`: Jalur penghubung ke database PostgreSQL.
3.  `views/pages/about.ejs`: Tempat angka-angka hasil hitungan tadi ditempelkan ke HTML agar terlihat cantik oleh user.

---

## 3. Fitur Utama: Implementasi Logika Pengambilan Data (Halaman '/about')

Kita membutuhkan 4 data statistik yang berbeda. Begini cara kita mengambilnya dari database:

1.  **Games Hosted (Total Game):** Kita hitung seberapa banyak baris di tabel `games`.
    *   Query: `SELECT COUNT(*) FROM games`
2.  **Active Gamers (Total User):** Kita hitung seberapa banyak user yang terdaftar.
    *   Query: `SELECT COUNT(*) FROM users`
3.  **Total Plays (Total Dimainkan):** Kita jumlahkan kolom `play_count` dari SEMUA game.
    *   Query: `SELECT SUM(play_count) FROM games`
4.  **Indie Devs (Developer Unik):** Ini menarik. Kita hitung ada berapa user yang *pernah* upload game.
    *   Logic: Kalau User A upload 10 game, dia tetap dihitung 1 Developer, bukan 10. Makanya kita pakai `DISTINCT`.
    *   Query: `SELECT COUNT(DISTINCT user_id) FROM games`

---

## 4. Bedah Source Code (The Secret Recipe)

Mari kita lihat kode aslinya di `routes/pages.js`. Perhatikan betapa efisiennya kode ini.

```javascript
// File: routes/pages.js

// Route untuk halaman About
router.get('/about', async (req, res) => {
  try {
    // 1. Siapkan wadah kosong untuk stats
    // Ini nilai default kalau-kalau database meledak (error).
    const stats = {
      gamesHosted: 0,
      activeGamers: 0,
      totalPlays: 0,
      indieDevs: 0
    };

    // 2. EKSEKUSI PARALEL (The Magic 🪄)
    // Promise.all menerima array berisi perintah-perintah database.
    // Dia akan "menembak" ke-4 perintah ini secara bersamaan ke database.
    // Code akan menunggu (await) sampai KEEMPAT-EMPATNYA selesai.
    const [gamesRes, usersRes, playsRes, devsRes] = await Promise.all([
      db.query("SELECT COUNT(*) FROM games"),                     // Tugas 1
      db.query("SELECT COUNT(*) FROM users"),                     // Tugas 2
      // COALESCE(..., 0) artinya: Kalau hasilnya NULL (karena blm ada game), ganti jadi 0.
      db.query("SELECT COALESCE(SUM(play_count), 0) as total FROM games"), // Tugas 3
      db.query("SELECT COUNT(DISTINCT user_id) FROM games")       // Tugas 4
    ]);

    // 3. Parsing Data
    // Data dari database (PostgreSQL) kadang formatnya String untuk angka besar (BigInt).
    // Kita paksa jadi Integer (parseInt) biar aman dipakai di Javascript.
    stats.gamesHosted = parseInt(gamesRes.rows[0].count);
    stats.activeGamers = parseInt(usersRes.rows[0].count);
    stats.totalPlays = parseInt(playsRes.rows[0].total);
    stats.indieDevs = parseInt(devsRes.rows[0].count);

    // 4. Render ke Layar
    // Kirim data 'stats' yang sudah terisi penuh ke file tampilan (ejs).
    res.render('pages/about', { 
      title: 'About Cok\'s',
      page: 'about', // Untuk highlight menu navbar
      stats // <-- Ini dia datanya!
    });

  } catch (error) {
    // Error Handling: Kalau gagal, tetap buka halamannya tapi angkanya 0 semua.
    // Jangan biarkan satu error kecil bikin seluruh halaman crash!
    console.error('Error fetching about stats:', error);
    res.render('pages/about', { 
      title: 'About Cok\'s',
      page: 'about',
      stats: { gamesHosted: 0, activeGamers: 0, totalPlays: 0, indieDevs: 0 }
    });
  }
});
```

## Kesimpulan

Logika di halaman About ini mengajarkan kita satu prinsip penting dalam Backend Programming: **Efisiensi**.

Kita bisa saja menulis kodenya berurutan (`await query1`, lalu `await query2`, dst), tapi itu pemborosan waktu yang fatal. Dengan `Promise.all`, kita memanfaatkan kemampuan database untuk multitasking. Hasilnya adalah halaman yang *snappy* (cepat) dan data yang akurat. Simple namun powerful!
