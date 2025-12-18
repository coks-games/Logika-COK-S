# Logika Carousel Game (Dokumentasi)

Dokumen ini menjelaskan cara kerja sistem carousel video pada halaman utama (landing page). Penjelasan dibuat sesederhana mungkin agar mudah dipahami.

---

## 1. Logika Carousel (Bahasa Manusia)

Bayangkan carousel ini seperti sebuah panggung pertunjukan yang memiliki antrean pemain:

1.  **Tampilan Awal**: Saat halaman dibuka, "pemain" pertama (game pertama) langsung naik ke panggung.
2.  **Gambar Dulu**: Yang pertama kali muncul adalah **Poster/Gambar** (Thumbnail) dari game tersebut.
3.  **Menunggu Sinyal (3 Detik)**: Sistem akan diam sejenak selama 3 detik. Ini memberikan waktu bagi penonton untuk melihat posternya.
4.  **Perubahan**:
    *   Jika game memiliki video, setelah 3 detik, Poster akan perlahan menghilang (fade out).
    *   Di belakang poster, Video sebenarnya sudah siap dan mulai dimainkan.
    *   Jadi, terlihat seolah-olah poster berubah menjadi video hidup.
5.  **Selesai & Ganti**:
    *   Jika video selesai diputar, sistem otomatis memanggil "pemain" berikutnya (Slide Selanjutnya).
    *   Jika hanya ada 1 game, video akan diputar ulang (Looping).

---

## 2. File-File yang Berhubungan

Ada 3 komponen utama (file) yang bekerja sama untuk membuat fitur ini:

1.  **`views/index.ejs`** (Kerangka Tubuh)
    *   Ini adalah HTML-nya. Tempat kita menyusun kotak-kotak untuk Gambar, Video, dan Tombol.
2.  **`public/js/carousel.js`** (Otak / Logika)
    *   Ini yang mengatur kapan video mulai, kapan berhenti, dan tombol mana yang ditekan.
3.  **`public/css/style.css`** (Baju / Tampilan)
    *   Mengatur posisi agar video pas di tengah, efek transparansi (fade), dan tata letak tombol.

---

## 3. Source Code & Penjelasannya

Berikut adalah potongan kode paling penting beserta alasannya.

### A. Struktur HTML (`views/index.ejs`)

```html
<div class="item active" data-video-url="URL_VIDEO_DISINI">
    <!-- 1. Lapisan Paling Atas: Gambar Thumbnail -->
    <div class="thumb-bg" style="background-image: url('URL_GAMBAR'); ..."></div>
    
    <!-- 2. Lapisan Tengah: Wadah Video (Awalnya disembunyikan) -->
    <div class="video-bg-container">
        <!-- Untuk Video YouTube -->
        <iframe class="youtube-frame" ...></iframe>
        <!-- Untuk Video File Sendiri (MP4) -->
        <video class="local-video" ...></video>
    </div>
</div>
```

**Kenapa begini?**
*   Kita menumpuk elemen. `thumb-bg` ditaruh di atas `video-bg-container`.
*   Atribut `data-video-url` di `div.item` berguna untuk menyimpan link video, supaya Javascript bisa membacanya nanti.

### B. Logika Javascript (`public/js/carousel.js`)

#### 1. Memulai Timer (Logika Peralihan)

```javascript
// Fungsi ini dipanggil setiap kali slide berganti
function startVideoTimer() {
    // 1. Ambil slide yang sedang aktif
    const activeItem = ...; 
    
    // 2. Baca URL videonya
    const videoUrl = activeItem.getAttribute("data-video-url");

    // 3. Tahan dulu 3 detik (3000ms) sebelum memutar video
    videoTimer = setTimeout(() => {
        // Kode di sini jalan setelah 3 detik
        
        // A. Sembunyikan Poster (Gambar)
        thumbBg.style.opacity = "0"; 
        
        // B. Tampilkan Wadah Video
        videoContainer.style.opacity = "1";
        
        // C. Putar Videonya
        playVideo();
        
    }, 3000); 
}
```

**Alasannya:**
*   `setTimeout`: Digunakan untuk memberikan jeda. Kita tidak mau video langsung jalan karena bisa bikin kaget atau berat loadingnya. Kita kasih orang lihat gambarnya dulu.
*   `opacity`: Kita mainkan transparansi (0 = bening, 1 = jelas) supaya perubahannya halus (efek *fade*), tidak *kaget* (jump cut).

---

## 4. Fitur Utama & Penjelasan Mendalam

### Fitur 1: Bagaimana logika pergantian gambar ke video?!

Kuncinya ada di **Opacity (Transparansi)** dan **Z-Index (Tumpukan)**.

1.  Awalnya, Gambar (`thumb-bg`) punya `opacity: 1` (terlihat jelas) dan `z-index: 10` (paling atas).
2.  Video ada di bawahnya, disembunyikan.
3.  Setelah 3 detik, Javascript mengubah CSS:
    *   Gambar diubah jadi `opacity: 0` (menghilang perlahan).
    *   Video diubah jadi `opacity: 1` (muncul perlahan).
4.  Karena posisinya ("absolute") bertumpuk di tempat yang sama, mata kita melihatnya sebagai transisi halus dari gambar diam menjadi gambar bergerak.

### Fitur 2: Bagaimana logika video play tetapi dalam kondisi mute (Tanpa suara) + alasannya?!

**Logika:**
```javascript
// Set video jadi bisu dulu
localVideo.muted = true; 

// Baru boleh di-play
localVideo.play().then(() => {
    // Sukses play! Coba nyalakan suara setelah setengah detik
    setTimeout(() => {
        localVideo.muted = false; // Coba un-mute (kadang berhasil, kadang diblokir browser)
    }, 500);
});
```

**Alasannya (Sangat Penting!):**
*   **Aturan Browser (Chrome/Firefox/Safari)**: Browser modern **MELARANG KERAS** video memutar suara secara otomatis (Autoplay with Sound) tanpa interaksi pengguna (klik/tap).
*   Jika kita paksa `muted = false` dari awal, Browser akan **memblokir** video tersebut. Video tidak akan jalan sama sekali (Blank/Error).
*   **Solusi Jalan Tengah**: Kita "mengalah" dengan memutar video secara bisu (`muted = true`). Setelah video jalan (sudah dianggap aktif), barulah kita coba menyalakan suaranya atau menyediakan tombol volume agar user bisa menyalakan sendiri.

### Fitur 3: Bagaimana dia looping ?!

Carousel ini menggunakan teknik **Memindah Elemen DOM**.

**Logika:**
```javascript
function moveNext() {
    // Ambil semua item
    let items = document.querySelectorAll(".item");
    
    // Pindahkan item PERTAMA ke posisi PALING BELAKANG
    slide.appendChild(items[0]);
}
```

**Bahasanya:**
Bayangkan antrean orang beli tiket: A, B, C, D.
1.  Si A (depan) selesai dilayani.
2.  Bukannya Si A pulang, petugas menyuruh Si A pindah ke **buntut antrean**.
3.  Sekarang urutannya jadi: B, C, D, A.
4.  Si B jadi yang paling depan (Active).

Dengan cara ini, antrean tidak pernah habis. A akan ketemu antrean lagi setelah D selesai. Ini menciptakan efek putaran (loop) yang tak terputus tanpa perlu menduplikasi banyak data.

### Fitur 4: Bagaimana logikanya mengambil data, berdasarkan apa ?!

Data yang ditampilkan di carousel **bukanlah acak**, melainkan data game **Terpopuler (Featured)**.

**Logika di Database (SQL):**
Sistem mengambil 4 game terbaik dengan aturan prioritas berikut:

1.  **Dilihat dari Jumlah Main (`play_count`)**: Game yang paling sering dimainkan akan muncul paling awal.
2.  **Dilihat dari Rating (`avg_rating`)**: Jika jumlah mainnya sama, game dengan bintang/rating tertinggi yang menang.
3.  **Dilihat dari Kebaruan (`created_at`)**: Jika ratingnya juga sama, game yang paling baru di-upload yang akan dipilih.

**Bukti Kode (`models/Game.js`):**
```javascript
ORDER BY 
  games.play_count DESC,  // Prioritas 1: Paling sering dimainkan
  games.avg_rating DESC,  // Prioritas 2: Rating paling bagus
  games.created_at DESC   // Prioritas 3: Paling baru
LIMIT 4                   // Hanya ambil 4 game teratas
```

**Kesimpulan:**
Jadi, game yang muncul di carousel adalah game-game "Populer" yang terbukti paling laris dan disukai pemain.
