# Logika Halaman Community: Dibalik Layar Fitur "Global Matrix Chat"

Dokumen ini akan membawamu masuk ke dapur pacu fitur **Community Chat** di COKS. Fitur ini bukan sekadar pajangan, tapi sistem *Real-time Messaging* sederhana yang memungkinkan user saling sapa tanpa refresh halaman (kalau pakai API) atau sekadar papan pesan publik.

Mari kita bedah logikanya sampai ke akar-akarnya!

---

## 1. Logika Pengambilan Data Chat (The Messenger Logic)

Logika di sini cukup *straightforward* alias to-the-point. Kita tidak ingin mempersulit user dengan room private atau enkripsi end-to-end yang ribet. Fokus kita adalah: **Kecepatan** dan **Keterbacaan**.

Prinsip kerjanya:
1.  **Ambil Terakhir:** Kita tidak mengambil *semua* chat dari zaman purba. Kita hanya mengambil **50 pesan terakhir**.
2.  **Urutan Waktu:** Chat ditampilkan dari yang terlama (atas) ke terbaru (bawah), persis seperti WhatsApp atau Discord (`ORDER BY created_at ASC`).
3.  **Join User:** Chat itu cuma text, kita butuh tahu *siapa* yang ngetik. Jadi kita harus "menjahit" (JOIN) tabel `chats` dengan tabel `users` untuk dapat Nama dan Avatar.

---

## 2. File-File yang Terlibat

Dua file ini adalah kunci komunikasi antar user di halaman Community:

1.  `models/Chat.js`: Si Gudang Data. Dia yang bertugas mengambil tumpukan pesan dari database dan menyimpannya kembali.
2.  `routes/pages.js` (Bagian `/community` & `/api/messages`): Si Kurir. Dia yang mengantarkan pesan dari Gudang ke Layar User (Frontend), dan menerima surat balasan dari User untuk disimpan ke Gudang.

---

## 3. Fitur Utama: Implementasi Logika Pengambilan Data (Halaman '/community')

Di halaman ini, kita punya dua mekanisme pengambilan data.

### A. Initial Load (Saat Halaman Pertama Dibuka)
Saat user mengetik `coks.site/community` di browser:
*   Server langsung mengambil 50 pesan terakhir.
*   Pesan-pesan ini "ditanam" langsung ke dalam HTML (`res.render`).
*   **Keuntungan:** User langsung lihat chat tanpa loading tambahan (Skeleton/Spinner).

### B. API Messages (Untuk Update Real-time/AJAX)
Walaupun di request ini kita fokus ke halaman dasarnya, server kita juga punya jalur rahasia `/api/messages`. Jalur ini biasanya dipakai oleh Javascript di frontend untuk mengambil pesan baru tanpa me-refresh halaman (teknik AJAX/Fetch).

---

## 4. Bedah Source Code (Bedah Otak)

Mari kita lihat kodingannya yang ringkas namun padat fungsi.

### Bagian 1: Model (models/Chat.js)

```javascript
// File: models/Chat.js

module.exports = {
  // Fungsi 1: Menyimpan Pesan (Create)
  // Menerima ID pengirim dan Isi Pesannya
  create: async (userId, message) => {
    // Kita pakai INSERT ... RETURNING.
    // RETURNING itu fitur keren PostgreSQL. Habis simpan, dia langsung balikin datanya (ID & Waktu).
    // Jadi kita gak perlu query SELECT lagi buat tahu kapan pesan itu dibuat. Efisien!
    const result = await db.query(
      `INSERT INTO chats (user_id, message) 
       VALUES ($1, $2) 
       RETURNING id, message, created_at`,
      [userId, message]
    );
    return result.rows[0];
  },

  // Fungsi 2: Mengambil Pesan (GetRecent)
  // Default limit 50 pesan biar server gak meledak loading ribuan chat.
  getRecent: async (limit = 50) => {
    // Perhatikan Query JOIN ini:
    // Kita ambil data chat (c) DAN data user (u).
    // Kenapa inner JOIN? Karena chat tanpa user itu chat hantu (data sampah), gak perlu diambil.
    const result = await db.query(
      `SELECT c.id, c.message, c.created_at, u.name as user_name, u.avatar as user_avatar 
       FROM chats c
       JOIN users u ON c.user_id = u.id
       ORDER BY c.created_at ASC
       LIMIT $1`,
      [limit]
    );
    return result.rows;
  }
};
```

### Bagian 2: Route / Controller (routes/pages.js)

Di sini logic pengontrol alur datanya.

```javascript
// File: routes/pages.js

const Chat = require('../models/Chat');

// 1. Route Halaman Utama (Render HTML)
router.get('/community', async (req, res) => {
  try {
    // Panggil Model untuk ambil 50 pesan
    const messages = await Chat.getRecent(50);
    
    // Render file 'views/pages/community.ejs'
    // Kita "suntikkan" data messages ke dalam view.
    res.render('pages/community', { 
      title: 'Community Chat',
      page: 'community',
      user: req.session.user || null, // Kirim data user login (kalau ada)
      messages // <-- Ini data chatnya
    });
  } catch (err) {
    // Error Handling: Kalau database error, jangan blank page.
    // Tampilkan halaman chat kosong saja.
    console.error(err);
    res.render('pages/community', { 
      // ... default params ...
      messages: [] // Kirim array kosong biar gak error loop di EJS
    });
  }
});

// 2. Route API (JSON Response)
// Ini dipakai kalau frontend mau minta data terbaru via Javascript (Fetch API)
router.get('/api/messages', async (req, res) => {
  try {
    const messages = await Chat.getRecent(50);
    res.json(messages); // Balas dengan JSON, bukan HTML
  } catch (err) {
    res.status(500).json({ error: 'Failed to fetch messages' });
  }
});

// 3. Route Kirim Pesan (POST)
router.post('/api/messages', async (req, res) => {
  // Security Check: Login dulu bos!
  if (!req.session.user) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  try {
    const { message } = req.body;
    // Validasi: Jangan kirim pesan kosong atau spasi doang
    if (!message || !message.trim()) {
      return res.status(400).json({ error: 'Message cannot be empty' });
    }

    // Simpan ke database
    const newChat = await Chat.create(req.session.user.id, message);
    
    // Konstruksi object balasan yang lengkap
    // Kita gabungkan data pesan baru (newChat) dengan info user dari session.
    // Supaya frontend bisa langsung nampilin chatnya tanpa perlu refresh.
    const completeMessage = {
      ...newChat,
      user_name: req.session.user.name,
      user_avatar: req.session.user.avatar
    };

    res.json(completeMessage);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Failed to send message' });
  }
});
```

## Kesimpulan

Logika Community Chat ini mengajarkan kita tentang **Kesederhanaan**. Kita tidak menggunakan WebSocket (seperti Socket.io) yang berat untuk setup awal, tapi kita tetap bisa mencapai fungsionalitas chat yang solid dengan kombinasi Database + REST API standar.

Pola `JOIN` di SQL dan pemisahan antara `Render Route` (HTML) dan `API Route` (JSON) adalah fondasi penting yang dipakai di hampir semua aplikasi web modern.
