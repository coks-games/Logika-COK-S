# Dokumentasi Teknis: Fitur Special Event

Dokumen ini menjelaskan arsitektur dan logika implementasi fitur **Special Event**. Penjelasan mencakup mekanisme tampilan pada sisi publik (Homepage), penanganan rute dinamis (Routing), serta manajemen data melalui panel Admin.

---

## 1. Ikhtisar Fungsional

Fitur Special Event dirancang untuk menonjolkan acara atau promosi penting pada halaman utama aplikasi. Fitur ini terdiri dari tiga komponen utama:
1.  **Homepage Display**: Menampilkan event yang sedang aktif secara otomatis.
2.  **Event Detail Page**: Halaman khusus yang memuat informasi rinci mengenai event.
3.  **Admin Management**: Antarmuka bagi administrator untuk membuat, menyunting, dan mengatur status aktivasi event.

---

## 2. Struktur Berkas Terkait

Berikut adalah berkas-berkas kunci yang mengelola fitur ini:

1.  **`models/Event.js`**: Model database yang menangani operasi *Create, Read, Update, Delete* (CRUD) dan logika aktivasi event.
2.  **`controllers/gameController.js`**: Mengontrol logika tampilan pada halaman utama (Landing Page).
3.  **`routes/pages.js`**: Mengelola *routing* untuk halaman detail event publik.
4.  **`controllers/adminController.js`**: Mengelola logika *backend* untuk manajemen event oleh admin.
5.  **`views/admin/events.ejs`**: Antarmuka (View) untuk manajemen event di panel admin.

---

## 3. Logika Implementasi Data

Berikut adalah penjelasan teknis mengenai alur data pada setiap bagian.

### A. Tampilan Beranda (Landing Page `/`)

Sistem menampilkan event pada halaman depan berdasarkan status validasi waktu dan aktivasi.

**Logika Pengambilan Data:**
Saat halaman utama dimuat, sistem melakukan *query* ke database untuk mencari satu event yang memenuhi kriteria:
1.  Status `is_active` bernilai `TRUE`.
2.  Tanggal saat ini (`CURRENT_DATE`) berada dalam rentang `start_date` dan `end_date`.

**Implementasi Kode (`models/Event.js`):**
```javascript
getActive: async () => {
    try {
        const result = await db.query(
            `SELECT * FROM events 
             WHERE is_active = TRUE 
             AND CURRENT_DATE BETWEEN start_date AND end_date
             ORDER BY created_at DESC LIMIT 1`
        );
        return result.rows[0] || null;
    } catch (error) { ... }
},
```

---

### B. Halaman Detail Event (`/events/:slug`)

Sistem menggunakan *Dynamic Routing* berbasis *slug* untuk menangani halaman detail event, sehingga menghasilkan URL yang ramah pengguna (SEO-friendly).

**Logika Routing:**
1.  Pengguna mengakses URL, misalnya `/events/m7-carnival`.
2.  Router menangkap parameter `slug` ("m7-carnival").
3.  Sistem merekonstruksi *Target URL* menjadi `/events/m7-carnival` dan mencari data yang sesuai di database.

**Implementasi Kode (`routes/pages.js`):**
```javascript
router.get('/events/:slug', async (req, res, next) => {
    try {
        const slug = req.params.slug;
        const fullPath = `/events/${slug}`;
        
        // Pencarian data berdasarkan URL target
        const event = await Event.findByTargetUrl(fullPath);
        
        if (!event) {
            return next(); // Mengarahkan ke handler 404 jika tidak ditemukan
        }
        
        res.render('pages/event-detail', {
            title: event.title,
            event: event
        });
    } catch (error) { ... }
});
```

---

### C. Manajemen Event (Admin Panel)

Bagian ini mencakup proses pembuatan event, pembuatan URL otomatis (*slug generation*), dan logika aktivasi tunggal.

#### 1. Pembuatan & Pemrosesan Data (`controllers/adminController.js`)

Saat admin menyimpan event baru, sistem melakukan beberapa proses otomatis:
*   **Slug Generation**: Judul event dikonversi menjadi format URL yang valid (huruf kecil, spasi diganti tanda hubung, menghapus karakter spesial).
*   **File Handling**: Berkas banner dan video disimpan ke direktori publik, dan path-nya disimpan ke database.

**Potongan Kode Generasi Slug:**
```javascript
const slug = title.toLowerCase()
    .replace(/[^a-z0-9]+/g, '-') // Mengganti karakter non-alfanumerik dengan tanda hubung
    .replace(/^-+|-+$/g, '');    // Menghapus tanda hubung di awal/akhir string
const target_url = `/events/${slug}`;
```

#### 2. Logika Aktivasi (Single Active Event)

Sistem menerapkan aturan bisnis bahwa **hanya satu event yang boleh aktif** dalam satu waktu pada Homepage.

**Mekanisme `toggleActive` (`models/Event.js`):**
1.  **Reset Global**: Mengubah status `is_active` menjadi `FALSE` untuk **seluruh** baris data di tabel events.
2.  **Set Active**: Mengubah status `is_active` menjadi `TRUE` hanya untuk ID event yang dipilih.

```javascript
toggleActive: async (id) => {
     // Nonaktifkan semua event terlebih dahulu
     await db.query("UPDATE events SET is_active = FALSE");
     
     // Aktifkan event yang dipilih
     await db.query("UPDATE events SET is_active = TRUE WHERE id = $1", [id]);
}
```

---

## Kesimpulan

Arsitektur fitur Special Event mengintegrasikan validasi waktu pada sisi model, *routing* dinamis pada sisi controller, dan manajemen data yang ketat pada sisi admin untuk memastikan konsistensi tampilan. Pendekatan *Single Active Event* mencegah konflik tampilan pada halaman utama.
