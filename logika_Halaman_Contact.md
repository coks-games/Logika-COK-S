# Logika Halaman Contact: Kotak Saran Digital COKS

Halaman Contact bukan sekadar hiasan. Ini adalah saluran pengaduan resmi user. Bagi sebuah platform, mendengar keluhan user itu vital, makanya kita buatkan sistem penampungan pesan yang rapi.

Dokumen ini akan menjelaskan apa yang terjadi ketika tombol "Send Message" ditekan.

---

## 1. Logika Pengiriman Pesan (The Feedback Loop)

Logika di sini berfungsi seperti **Kotak Pos**.
1.  **Penulisan:** User mengisi formulir (Nama, Email, Subjek, Pesan).
2.  **Validasi:** Sistem memastikan data tidak kosong (meskipun validasi utama ada di frontend HTML `required`).
3.  **Penyimpanan:** Pesan tidak dikirim ke email admin (karena ribet setup SMTP), tapi **disimpan ke Database**.
4.  **Notifikasi:** Admin nanti bisa melihat angka notifikasi "Pesan Belum Dibaca" di dashboard mereka.

Kenapa disimpan di database?
*   **Arsip Permanen:** Pesan tidak bakal hilang / masuk spam folder.
*   **Status Tracking:** Kita bisa menandai mana pesan yang "Sudah Dibaca" (`is_read = true`) dan mana yang belum.

---

## 2. File-File yang Terlibat

Sistem ini melibatkan:
1.  `routes/pages.js`: Si Penerima Surat. Dia yang menangani alamat `/contact` baik saat dibuka (GET) maupun saat surat dikirim (POST).
2.  `models/Message.js`: Si Arsiparis. Dia yang menyusun rapi surat-surat tersebut ke dalam tabel `messages`.

---

## 3. Fitur Utama: Implementasi Logika

Ada dua kejadian di halaman ini:

### A. Membuka Halaman (GET /contact)
Tidak ada pengambilan data spesial di sini. Server hanya menyajikan formulir kosong. Sederhana.

### B. Menekan Tombol "Send Message" (POST /contact)
Inilah inti aksinya!
Saat tombol ditekan:
1.  Browser membungkus data form menjadi paket HTTP POST.
2.  Server menerima paket tersebut (`req.body`).
3.  Server memanggil `Message.create()` untuk menyimpan data.
4.  Server memberikan **Flash Message** (Pesan Kilat): "Thank you! Your message has been received."
5.  Halaman di-refresh (Redirect) agar form kembali bersih.

---

## 4. Bedah Source Code (Bedah Kotak Pos)

Mari kita lihat jeroan kodenya.

### Bagian 1: Route (routes/pages.js)

Route ini menangani lalu lintas data masuk.

```javascript
// File: routes/pages.js
const Message = require('../models/Message');

// 1. Tampilkan Halaman Contact
router.get('/contact', (req, res) => {
    res.render('pages/contact', {
        title: 'Contact Us',
        page: 'contact',
        user: req.session.user || null
    });
});

// 2. Handle Tombol "Send Message" (Action Utama)
router.post('/contact', async (req, res) => {
    try {
        // Ambil data dari form HTML
        const { name, email, subject, message } = req.body;
        
        // Simpan ke Database
        // Kita pakai 'await' biar yakin datanya beneran masuk sebelum kasih respon sukses.
        await Message.create({ name, email, subject, message });
        
        // Kasih Feedback ke User
        req.session.success = "Thank you! Your message has been received. Our team will read it shortly.";
        
        // Redirect balik ke halaman contact (Pola PRG: Post-Redirect-Get)
        // Ini mencegah resubmission form kalau user refresh halaman.
        res.redirect('/contact');
    } catch (error) {
        console.error("Contact Error:", error);
        req.session.error = "Failed to send message. Please try again.";
        res.redirect('/contact');
    }
});
```

### Bagian 2: Model (models/Message.js)

Model ini menggunakan Class statis agar rapi.

```javascript
// File: models/Message.js

class Message {
    // Create new message
    static async create(data) {
        const query = `
            INSERT INTO messages (name, email, subject, message, is_read, created_at)
            VALUES ($1, $2, $3, $4, false, NOW())
            RETURNING *
        `;
        // Penjelasan:
        // 'is_read' defaultnya false (Belum dibaca).
        // 'created_at' otomatis diisi waktu sekarang (NOW()).
        const values = [data.name, data.email, data.subject, data.message];
        
        try {
            const result = await db.query(query, values);
            return result.rows[0];
        } catch (error) {
            throw error;
        }
    }
    
    // ... fungsi lain seperti getUnreadCount ...
}
```

## Kesimpulan

Fitur Contact ini menggunakan pola klasik **Store-and-Forward**. Pesan disimpan (Store) di database, untuk kemudian diteruskan (Forward) ke dashboard admin. Implementasi `is_read` boolean juga menandakan bahwa sistem ini bukan sekadar penampung teks sampah, tapi sistem tiket support sederhana yang fungsional.
