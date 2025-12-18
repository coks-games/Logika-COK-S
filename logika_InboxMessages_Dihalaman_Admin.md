# Logika Inbox Messages (Pesan Masuk) di Halaman Admin

Dokumen ini menjelaskan bagaimana fitur "Inbox / Messages" bekerja. Fitur ini berfungsi sebagai kotak surat digital tempat Admin menerima pesan, pertanyaan, atau laporan dari pengunjung website.

## 1. Logika Utama Inbox Messages

Sistem ini sangat sederhana, mirip dengan email tapi versi Lite:

1.  **Pengambilan Pesan:** Saat Admin membuka halaman Inbox, sistem mengambil **semua pesan** dari database, diurutkan dari yang paling **baru**.
2.  **Tampilan Responsif:**
    *   Di **Komputer (Desktop):** Pesan ditampilkan dalam bentuk **Tabel** yang rapi agar mudah discan mata.
    *   Di **HP (Mobile):** Pesan berubah bentuk menjadi **Kartu (Card)** agar enak dibaca di layar kecil.
3.  **Aksi Balas Cepat:** Tombol "Reply" tidak membuka form baru di website, melainkan langsung **membuka aplikasi Email** di komputer/HP Admin (seperti Gmail/Outlook) dengan subjek yang sudah terisi otomatis.
4.  **Hapus Permanen:** Admin bisa menghapus pesan yang sudah tidak penting untuk menjaga kebersihan database.

## 2. File-File Terkait

Berikut adalah 4 sekawan file yang menjalankan fitur ini:
1.  **`routes/admin.js`**: Pintu gerbang yang menerima akses ke halaman inbox dan permintaan hapus pesan.
2.  **`controllers/adminController.js`**: "Otak" yang menyuruh Model mengambil data dan mengirimnya ke View.
3.  **`models/Message.js`**: "Tangan" yang langsung mengambil atau menghapus data di tabel SQL `messages`.
4.  **`views/admin/messages.ejs`**: Tampilan visual (Tabel & Kartu) yang dilihat Admin.

## 3. Detail Implementasi & Source Code

Mari kita bongkar mesinnya.

### A. Menampilkan Pesan (`/admin/messages`)

**1. Route (`routes/admin.js`)**
```javascript
// Baris 84: Rute untuk melihat pesan
router.get('/messages', adminController.listMessages);
```

**2. Controller (`controllers/adminController.js`)**
Fungsi `listMessages` bertugas menyiapkan data.
```javascript
listMessages: async (req, res) => {
    try {
        const Message = require('../models/Message');
        // 1. Ambil semua pesan (Urut dari yang terbaru)
        const messages = await Message.getAll();
        
        // 2. Kirim ke tampilan
        res.render('admin/messages', { 
            messages, // Data utama
            currentUrl: '/admin/messages',
            title: 'Inbox Messages'
        });
    } // ... error handling
},
```

**3. Model (`models/Message.js`)**
Fungsi `getAll` melakukan query SQL murni.
```javascript
static async getAll() {
    // SELECT * : Ambil semua kolom
    // ORDER BY created_at DESC : Yang baru di atas, yang lama di bawah
    const query = `SELECT * FROM messages ORDER BY created_at DESC`;
    try {
        const result = await db.query(query);
        return result.rows;
    } catch (error) { throw error; }
}
```

### B. Tampilan (View Logic)

Di file **`views/admin/messages.ejs`**, kita menggunakan logika EJS untuk menangani jika inbox kosong.

```ejs
<!-- Cek dulu, ada pesannya gak? -->
<% if (messages.length > 0) { %>
    <!-- Kalau ada, lakukan looping (perulangan) -->
    <% messages.forEach(msg => { %>
        <tr>
            <!-- Konversi waktu mesin ke waktu manusia yang enak dibaca -->
            <td>
                <%= new Date(msg.created_at).toLocaleDateString() %> <br>
                <%= new Date(msg.created_at).toLocaleTimeString(...) %>
            </td>
            <!-- Tampilkan Nama & Email Pengirim -->
            <td>
                <div class="fw-bold"><%= msg.name %></div>
                <a href="mailto:<%= msg.email %>"><%= msg.email %></a>
            </td>
            <!-- ... kolom lainnya ... -->
            
            <!-- Tombol Action -->
            <td>
                <!-- 1. Reply: Membuka aplikasi email bawaan -->
                <a href="mailto:<%= msg.email %>?subject=Re: <%= msg.subject %>">Reply</a>
                
                <!-- 2. Delete: Form khusus untuk hapus -->
                <form action="/admin/messages/<%= msg.id %>/delete" method="POST" ...>
                    <button type="submit">Delete</button>
                </form>
            </td>
        </tr>
    <% }) %>
<% } else { %>
    <!-- Kalau kosong, tampilkan gambar inbox kosong biar gak sepi -->
    <p>Your inbox is empty.</p>
<% } %>
```

### C. Menghapus Pesan

Saat tombol DELETE ditekan:

**1. Route**
```javascript
// Baris 85: Menerima ID pesan yang mau dihapus
router.post('/messages/:id/delete', adminController.deleteMessage);
```

**2. Controller**
```javascript
deleteMessage: async (req, res) => {
    try {
        const Message = require('../models/Message');
        // Perintah hapus dijalankan
        await Message.delete(req.params.id);
        
        // Kasih notifikasi sukses
        req.session.success = "Message deleted successfully";
        res.redirect('/admin/messages');
    } // ...
}
```
**Alasannya:** Fitur hapus ini bersifat permanen (Hard Delete) karena pesan spam atau pesan lama biasanya memang tidak perlu disimpan selamanya di database.
