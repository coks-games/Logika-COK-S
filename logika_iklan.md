# Dokumentasi Teknis: Manajemen Fitur Iklan (Ads)

Dokumen ini menjelaskan secara teknis bagaimana sistem manajemen iklan bekerja, mulai dari pemilihan data untuk tampilan publik (*Homepage*) hingga logika aktivasi eksklusif pada panel Admin.

---

## 1. Ikhtisar Fungsional

Fitur Iklan memungkinkan admin untuk menempatkan banner promosi pada area strategis situs (seperti "home_top"). Sistem menerapkan logika **Single Active Ad**, di mana hanya satu iklan yang boleh aktif pada satu lokasi tertentu dalam satu waktu, guna menjaga kerapian tata letak antarmuka.

---

## 2. Struktur Berkas Terkait

Berkas-berkas yang bertanggung jawab atas fitur ini meliputi:

1.  **`models/Ad.js`**: Menangani logika database, termasuk mekanisme "sakelar" untuk aktivasi iklan.
2.  **`controllers/gameController.js`**: Mengambil data iklan aktif untuk dikirim ke halaman utama.
3.  **`views/index.ejs`**: Menampilkan banner iklan dan menangani navigasi saat diklik.

---

## 3. Logika Implementasi Data

Berikut adalah bedah tuntas logika yang berjalan di balik layar.

### A. Tampilan Beranda (Landing Page `/`)

Saat pengguna mengakses halaman utama, sistem harus memutuskan iklan mana yang berhak tampil.

**Logika Pengambilan Data:**
Controller meminta model untuk mengambil **satu-satunya** iklan yang berstatus `is_active = TRUE` untuk lokasi `home_top`.

**Implementasi Controller (`controllers/gameController.js`):**
```javascript
landing: async (req, res) => {
    // ...
    // Mengambil 1 iklan aktif khusus untuk lokasi bagian atas beranda
    const activeAd = await Ad.getActive('home_top');
    
    // Data dikirim ke view (index.ejs)
    res.render("index", {
        // ...
        activeAd, 
    });
},
```

**Implementasi Model (`models/Ad.js`):**
Query SQL dirancang sangat spesifik dengan `LIMIT 1` untuk efisiensi.
```javascript
getActive: async (location = 'home_top') => {
    // Ambil semua kolom DARI tabel ads
    // SYARAT 1: is_active harus TRUE
    // SYARAT 2: location harus sesuai request (misal: 'home_top')
    const result = await db.query(
        "SELECT * FROM ads WHERE is_active = TRUE AND location = $1 LIMIT 1", 
        [location]
    );
    return result.rows[0]; // Kembalikan objek iklan atau undefined jika kosong
},
```

---

### B. Navigasi & Tampilan Frontend

Bagaimana jika pengunjung mengklik iklan tersebut?

**Logika Tampilan (`views/index.ejs`):**
Kami menggunakan pengecekan kondisional EJS.
1.  **Jika `activeAd` ada**: Tampilkan gambar banner yang dibungkus *anchor tag* (`<a>`).
2.  **Jika `activeAd` kosong**: Tampilkan *placeholder* default ("SPACE IKLAN") agar layout tidak terlihat rusak/kosong.

**Logika Navigasi:**
Atribut `target="_blank"` disematkan pada link agar saat pengunjung mengklik iklan, browser membuka **tab baru**. Ini krusial agar pengunjung tidak meninggalkan halaman situs utama kita.

**Snippet Tampilan:**
```html
<div class="container ad-container">
    <% if (typeof activeAd !== 'undefined' && activeAd) { %>
        <!-- KASUS: ADA IKLAN AKTIF -->
        <!-- Link pembungkus mengarah ke target_url iklan -->
        <a href="<%= activeAd.target_url || '#' %>" target="_blank" class="...">
            <img src="<%= activeAd.image_url %>" alt="<%= activeAd.title %>" ...>
        </a>
    <% } else { %>
        <!-- KASUS: TIDAK ADA IKLAN (Placeholder) -->
        <div class="ad-placeholder">
            [ SPACE IKLAN 728x250 ]<br />
            <span>Tempatkan bannermu di sini</span>
        </div>
    <% } %>
</div>
```

---

### C. Logika Manajemen di Admin Panel (PENTING!)

Bagian ini menjelaskan mekanisme cerdas di balik tombol **"Activate"**. Sistem tidak mengizinkan admin melakukan kesalahan seperti mengaktifkan dua iklan sekaligus di tempat yang sama.

**Aturan Bisnis:**
"Pada satu lokasi (misal: `home_top`), hanya boleh ada **SATU** iklan yang menyala."

**Mekanisme `activate` (`models/Ad.js`):**
Saat admin mengklik tombol aktivasi untuk Iklan B:
1.  **Langkah 1 (Reset Total)**: Sistem secara paksa mematikan (`FALSE`) **SEMUA** iklan yang berada di lokasi yang sama dengan Iklan B.
2.  **Langkah 2 (Nyalakan Target)**: Setelah semua mati, sistem baru menyalakan (`TRUE`) Iklan B.

Ini disebut logika *Atomic Toggle* untuk menjamin konsistensi data.

**Implementasi Lengkap:**
```javascript
activate: async (id) => {
    // 0. Cek lokasi iklan yang mau diaktifkan (misal: 'home_top')
    const result = await db.query("SELECT location FROM ads WHERE id = $1", [id]);
    if (result.rows.length === 0) return; // Batal jika ID salah

    const location = result.rows[0].location;

    // 1. Matikan SEMUA iklan di lokasi tersebut (Sapu bersih!)
    await db.query("UPDATE ads SET is_active = FALSE WHERE location = $1", [location]);
    
    // 2. Baru nyalakan iklan yang dipilih admin
    await db.query("UPDATE ads SET is_active = TRUE WHERE id = $1", [id]);
},
```

---

## Kesimpulan

Sistem manajemen iklan ini dibangun dengan prioritas pada **Konsistensi Tampilan**. Dengan logika aktivasi *Atomic Toggle*, admin tidak perlu mematikan iklan lama secara manual sebelum menyalakan iklan baru—sistem menanganinya secara otomatis. Di sisi depan (*frontend*), mekanisme *placeholder* memastikan situs tetap terlihat profesional meskipun belum ada iklan yang dipasang.
