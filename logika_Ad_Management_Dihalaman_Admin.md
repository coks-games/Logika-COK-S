# Logika Ad Management (Manajemen Iklan) di Halaman Admin

Dokumen ini menjelaskan bagaimana fitur "Ad Management" bekerja. Fitur ini dibuat agar Admin bisa dengan mudah mengganti-ganti banner iklan di halaman utama website tanpa perlu menyentuh kodingan sama sekali.

## 1. Logika Utama Ad Management

Ada tiga konsep sederhana yang perlu dipahami:

1.  **Satu Tempat, Banyak Pilihan:** Admin bisa meng-upload banyak banner iklan untuk satu lokasi (misalnya: "Homepage Top"). Tapi...
2.  **Hanya Satu Pemenang:** Hanya ada **SATU** iklan yang boleh aktif ("Active") di satu lokasi pada satu waktu.
3.  **Otomatisasi Saklar:** Saat Admin mengaktifkan Iklan A, sistem akan otomatis mematikan Iklan B, C, dan D yang ada di lokasi yang sama. Jadi tidak akan ada iklan yang bertumpuk.

### Dimana Iklan Tampil?
Saat ini, iklan disetting untuk muncul menyambut user di:
*   **Homepage Top (Header):** Banner besar yang muncul di bagian paling atas halaman utama website. Ukuran saran: 728x250 pixels.

## 2. File-File Terkait

Fitur ini berjalan berkat kerjasama file-file berikut:
1.  **`routes/admin.js`**: Pintu gerbang yang menerima upload file gambar dan data form.
2.  **`controllers/adminController.js`**: "Otak" yang memproses upload dan memerintahkan database.
3.  **`models/Ad.js`**: "Tangan" yang menyimpan data ke database dan mematikan/menghidupkan status iklan.
4.  **`views/admin/ads.ejs`**: Tampilan "Ad Manager" yang dilihat Admin.

## 3. Detail Implementasi & Source Code

Mari kita bedah kodenya agar tidak ada sihir di antara kita.

### A. Menampilkan Daftar Iklan (`/admin/ads`)

**Controller:** `listAds` di `controllers/adminController.js`
```javascript
listAds: async (req, res) => {
    try {
        const Ad = require('../models/Ad');
        // 1. Ambil semua iklan dari database (terurut dari yang terbaru)
        const ads = await Ad.getAll();
        
        // 2. Render tampilan
        res.render("admin/ads", {
            title: "Ad Management",
            ads // Bawa data iklan ke tampilan
        });
    } // ... error handling
}
```
**Alasannya:** Sangat straight-forward. Kita butuh melihat semua koleksi banner yang pernah di-upload, baik yang aktif maupun yang sudah kadaluarsa (inactive).

### B. Upload Iklan Baru (New Ad)

Saat Admin klik "New Ad" -> Isi Form -> Klik "Upload Ad", inilah yang terjadi:

**1. Route (`routes/admin.js`)**
Disini ada "Satpam" bernama `multer` yang memeriksa file yang di-upload.
```javascript
// Konfigurasi Multer (Penyaring File)
const upload = multer({ 
    // ...
    fileFilter: (req, file, cb) => {
        // Cek apakah file-nya gambar?
        if (file.fieldname === "ad_image") {
            if (!file.originalname.match(/\.(jpg|jpeg|png|gif)$/)) {
                return cb(new Error('Only image files are allowed for ads!'), false);
            }
        }
        cb(null, true);
    }
});
```
**Alasannya:** Keamanan. Kita tidak mau ada orang iseng upload file virus `.exe` atau script `.php` ke server kita. Hanya gambar yang boleh lewat.

**2. Controller (`createAd`)**
```javascript
createAd: async (req, res) => {
    // ...
    // 1. Ambil path gambar yang baru di-upload oleh Multer
    if (req.file) {
        image_url = '/' + req.file.path.replace(/\\/g, '/')....
    }

    // 2. Validasi: Gambar wajib ada!
    if (!image_url) { ... error ... }

    // 3. Simpan ke Database (Status default: INACTIVE)
    await Ad.create({ title, image_url, target_url, location });
    
    // ... redirect kembali ke halaman ads
}
```
**Kenapa Status Defaultnya INACTIVE?**
Agar Admin bisa me-review dulu tampilannya (preview) di dashboard sebelum benar-benar menayangkannya ke publik.

### C. Logika "Saklar" (Aktivasi Iklan)

Ini adalah bagian paling genius dari sistem ini. Bagaimana caranya agar hanya ada satu iklan yang aktif?

**Model:** `activate` di `models/Ad.js`
```javascript
activate: async (id) => {
    // Langkah 1: Cek dulu iklan ini lokasinya dimana (misal: 'home_top')
    const result = await db.query("SELECT location FROM ads WHERE id = $1", [id]);
    const location = result.rows[0].location;

    // Langkah 2: MATIKAN SEMUA iklan lain yang ada di lokasi tersebut
    // "Semua minggir, bos baru mau lewat!"
    await db.query("UPDATE ads SET is_active = FALSE WHERE location = $1", [location]);
    
    // Langkah 3: Baru AKTIFKAN iklan yang dipilih ini
    await db.query("UPDATE ads SET is_active = TRUE WHERE id = $1", [id]);
},
```
**Alasannya:** Dengan logika "Matikan Semua Dulu, Baru Hidupkan Satu", kita menjamin konsistensi data. Tidak akan pernah terjadi bug dimana ada 2 banner rebutan tempat di halaman Home. Simple tapi efektif!
