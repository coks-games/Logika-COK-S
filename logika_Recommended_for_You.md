# Logika "Recommended for You" (Dokumentasi)

Dokumen ini menjelaskan bagaimana fitur rekomendasi game yang bisa digeser-geser (scroll) itu bekerja. Kita bongkar rahasianya dengan bahasa yang manusiawi!

---

## 1. Logika Singkat (Bahasa Manusia)

Fitur ini sebenarnya sederhana namun pintar. Bayangkan sebuah rak buku panjang yang isinya game-game pilihan acak:

1.  **Pengacakan**: Setiap kali kamu refresh halaman, sistem akan mengocok kartu game dan mengambil 6 game secara acak. Jadi rekomendasinya tidak membosankan dan selalu berubah.
2.  **Rak Geser**: Game-game ini disusun berjejer ke samping (horizontal). Karena layar HP atau Laptop terbatas, tidak semua game muat ditampilkan sekaligus.
3.  **Scroll Alami**: Kita tidak pakai tombol panah kiri-kanan yang ribet. Kita pakai fitur bawaan browser (seperti di HP) di mana kamu bisa mengusap (swipe) atau scroll ke samping dengan mulus.

---

## 2. File-File yang Berhubungan

Hanya 3 file utama yang bermain peran di sini:

1.  **`views/index.ejs`** (Tampilan): Tempat kita membuat "Wadah" atau rel untuk game berjejer.
2.  **`controllers/gameController.js`** (Pelayan): Yang menyuruh koki (database) untuk mengambilkan game acak.
3.  **`models/Game.js`** (Koki Database): Yang punya resep rahasia untuk mengacak urutan game.

---

## 3. Source Code & Penjelasannya

Mari kita bedah kodenya satu per satu.

### A. Otak Penggerak (`controllers/gameController.js`)

Di sini kita meminta data game acak.

```javascript
// Di dalam function landing:
const recommendedGames = await Game.getRandom(6);
```

**Penjelasan:**
*   Kode ini memerintahkan Model Game: *"Hei, tolong ambilkan saya 6 game apa saja, terserah, acak ya!"*
*   Hasilnya disimpan di variabel `recommendedGames` yang siap dikirim ke layar.

---

### B. Resep Rahasia (`models/Game.js`)

Inilah fungsi sakti yang melakukan pengacakan.

```javascript
getRandom: async (limit = 5) => {
    try {
      const result = await db.query(`
        SELECT games.*, users.name as author_name 
        FROM games 
        JOIN users ON games.user_id = users.id
        ORDER BY RANDOM()  -- <--- INI KUNCINYA!
        LIMIT $1
      `, [limit]);
      return result.rows;
    } catch (error) {
      // ... error handling
    }
},
```

**Kenapa pakai `ORDER BY RANDOM()`?**
*   Ini adalah perintah SQL paling efisien untuk mengacak baris data.
*   Sitem database akan memberikan nomor acak pada setiap game, lalu mengurutkannya. Hasilnya? Urutan game selalu berubah setiap kali dipanggil. Sangat efektif untuk fitur "Discovery" agar user menemukan game-game lawas yang mungkin tenggelam.

---

### C. Tampilan Rak Geser (`views/index.ejs`)

Ini bagian paling menarik. Kita membuat wadah yang bisa digeser TANPA Javascript yang berat.

**CSS (Style):**
```css
.horizontal-scroll-container {
    display: flex;             /* 1. Bariskan ke samping */
    overflow-x: auto;          /* 2. Izinkan scroll meluber ke samping */
    gap: 24px;                 /* 3. Jarak antar kartu game */
    
    scroll-snap-type: x mandatory; /* 4. Magnet! (Biar pas berhenti di tengah kartu) */
    scroll-behavior: smooth;       /* 5. Gerakan halus */
    
    /* Sembunyikan Scrollbar (biar rapi) */
    -ms-overflow-style: none;
    scrollbar-width: none;
}
```

**Elemen HTML:**
```html
<div class="horizontal-scroll-container">
    <% recommendedGames.forEach((game) => { %>
        <!-- Kartu Game Disini -->
        <a href="..." class="playstore-card">
           ...
        </a>
    <% }); %>
</div>
```

---

## 4. Fitur Utama & Penjelasan Mendalam

### Fitur 1: Bagaimana logika pergeseran gambar ke kanan atau ke kiri?!

Jawabannya: **Murni CSS (Cascading Style Sheets)!** Kita tidak menggunakan Javascript sama sekali untuk pergeseran ini. Sistem ini memanfaatkan fitur *Native Scrolling* dari browser.

Logikanya begini:
1.  **`display: flex`**: Memaksa semua anak elemen (kartu game) untuk berdiri sejajar dalam satu baris, bahu-membahu, dan tidak boleh turun ke baris baru.
2.  **`overflow-x: auto`**: Ini intinya. Kita bilang ke browser: *"Hei, kalau anak-anakku ini kepanjangan dan gak muat di layar, jangan dipotong! Tapi jadikan area ini bisa discroll ke samping."*
3.  **`scroll-snap-type: x mandatory`**: Ini fitur **"Magnet"**. Tanpa ini, kalau kamu scroll dan lepas jari, posisinya bisa berhenti nanggung (tengah-tengah antar dua game). Dengan fitur ini, browser akan memaksa posisi scroll untuk *snap* (menempel) pas di awal kartu terdekat. Rasanya jadi "ceklek-ceklek" premium seperti di Play Store atau Netflix.

### Fitur 2: Bagaimana logika pengambilan data yang di implementasikan di section tersebut?!

Jawabannya: **Random Sampling (Pengambilan Acak).**

Kita tidak melihat popularitas, tidak melihat rating, dan tidak melihat tanggal upload. Kita benar-benar "menutup mata" dan mengambil apa saja yang ada di kantong database.

**Kenapa begini?**
*   Tujuannya adalah **Pemerataan Eksposur**.
*   Game populer sudah punya tempat di Carousel (atas).
*   Game baru sudah punya tempat di "New Releases".
*   Section "Recommended" ini bertugas menjadi pahlawan bagi game-game lain (underdog) supaya tetap punya kesempatan untuk dilihat dan dimainkan oleh user.
