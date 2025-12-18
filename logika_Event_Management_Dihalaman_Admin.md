# Logika Event Management (Manajemen Acara) di Halaman Admin

Dokumen ini membedah fitur "Event Management" di Halaman Admin. Fitur ini memungkinkan Admin untuk membuat acara, turnamen, atau pengumuman khusus yang akan muncul di halaman depan (Home) website.

## 1. Logika Utama Event Management

Fitur ini memiliki 3 nyawa utama: **Menampilkan Daftar Event**, **Membuat Event Baru**, dan **Mengedit Event**.

### A. Pembukaan Halaman (Data Retrieval)
Saat Admin membuka `/admin/events`, sistem melakukan satu tugas utama:
1.  Mengambil **semua daftar event** dari database, diurutkan dari yang terbaru.


### B. Membuat Event Baru (Create)
Ketika tombol "Create Event" ditekan:
1.  Admin mengisi judul, deskripsi, upload banner/video, dan waktu mulai/selesai.
2.  Sistem otomatis membuatkan link khusus (`target_url`) yang ramah URL (contoh: "M7 Carnival" menjadi `/events/m7-carnival`).
3.  Banner dan Video di-upload ke server, dan lokasinya disimpan di database.

### C. Mengedit Event (Edit)
Fitur edit menggunakan teknik **Modal Dinamis**:
1.  **Saat tombol pensil ditekan:** Sistem tidak me-reload halaman. Javascript langsung mengambil data event yang tersimpan di tombol itu sendiri (data-attributes) dan menempelkannya ke dalam formulir Pop-up.
2.  **Saat tombol Save ditekan:** Formulir dikirim ke server. Jika ada gambar baru yang di-upload, sistem akan pintar untuk **menghapus gambar lama** agar server tidak penuh sampah (file sampah).

---

## 2. File-File Terkait

Berikut adalah orkestrasi file-file MVC yang terlibat:
1.  **`routes/admin.js`**: "Polisi Lalu Lintas" yang mengatur rute URL (Get, Post, Update, Delete).
2.  **`controllers/adminController.js`**: "Otak" yang berpikir (validasi input, upload file, panggil database).
3.  **`models/Event.js`**: "Tangan" yang bekerja langsung mengutak-atik database SQL.
4.  **`views/admin/events.ejs`**: "Wajah" antarmuka yang berinteraksi dengan Admin (Tabel & Modal).

---

## 3. Detail Implementasi & Source Code

Mari kita bedah kodenya satu per satu dengan bahasa manusia.

### A. Menampilkan Halaman (`/admin/events`)

**Controller:** `listEvents` di `controllers/adminController.js`
```javascript
listEvents: async (req, res) => {
    try {
        // 1. Ambil semua event (Otak menyuruh Tangan)
        const events = await Event.getAll();
        
        // 2. Tampilkan halaman dan bawa perbekalan datanya
        res.render("admin/events", {
            title: "Event Management",
            events,                 // Data event untuk tabel
        });
    } // ... error handling
},
```
**Alasannya:** Kita cukup mengambil data events saja karena fitur pemilihan game sudah tidak digunakan di tampilan (UI) yang sekarang.

### B. Logika Tombol "Pensil" (Edit UI)

Bagian ini menarik karena menggunakan **Javascript murni** di sisi browser (Client-Side) untuk kecepatan instan.

**View:** `views/admin/events.ejs`
Perhatikan bagaimana data diselipkan di dalam tombol HTML (`data-title`, `data-banner`, dll).

```html
<!-- Tombol Edit menyimpan 'rahasia' data event-nya sendiri -->
<button type="button" 
        class="edit-event-btn" 
        data-id="<%= e.id %>"
        data-title="<%= e.title %>"
        data-description="<%= e.description %>"
        data-banner="<%= e.banner_url %>"
        ...>
    <i class="fas fa-edit"></i>
</button>

<!-- Script Javascript di bagian bawah file -->
<script>
    // 1. Cari semua tombol edit
    const editButtons = document.querySelectorAll('.edit-event-btn');
    
    // 2. Pasang 'telinga' (event listener) untuk setiap tombol
    editButtons.forEach(btn => {
        btn.addEventListener('click', function() {
            const id = this.dataset.id; // Ambil ID dari tombol
            
            // 3. Ubah tujuan formulir agar mengarah ke ID yang benar
            // Dari action="" menjadi action="/admin/events/12/update"
            if(editForm) editForm.action = `/admin/events/${id}/update`;
            
            // 4. Isi kotak-kotak input dengan data dari tombol
            document.getElementById('editTitle').value = this.dataset.title;
            document.getElementById('editDescription').value = this.dataset.description;
            // ... dst untuk field lain
        });
    });
</script>
```
**Kenapa begini?** Karena jauh lebih cepat daripada harus request ke server lagi hanya untuk mengambil data satu event. Datanya sudah ada di tabel, kita tinggal "pindahkan" ke dalam Modal form.

### C. Logika "Create" & "Save" (Backend Action)

Ketika tombol "Create Event" atau "Save Event" ditekan, data dikirim ke Controller.

**Controller:** `createEvent` & `updateEvent`
Disini ada logika menarik untuk penanganan file dan URL otomatis.

```javascript
// Potongan logika di controllers/adminController.js

// 1. Auto-Slugify: Membuat URL cantik dari Judul
// Contoh: "Turnamen Besar" -> "turnamen-besar"
const slug = title.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '');
const target_url = `/events/${slug}`;

// 2. Logika Upload File
if (req.files) {
    if (req.files.banner) {
        // Jika ini UPDATE dan ada banner lama, HAPUS banner lama dulu!
        // Ini step penting agar harddisk server tidak penuh sampah.
        if(oldEvent && oldEvent.banner_url) {
            fs.unlinkSync(oldPath); // Hapus file fisik
        }
        // Simpan path file baru
        banner_url = '/' + req.files.banner[0].path...
    }
}

// 3. Panggil Model untuk simpan ke SQL
await Event.create({ title, ..., target_url, ... }); 
```

**Alasannya:**
*   **Slugify:** URL yang bisa dibaca manusia (`/events/summer-sale`) jauh lebih baik untuk SEO dan user experience daripada URL acak (`/events/83749`).
*   **Cleanup File:** Menghapus file lama saat update adalah praktik *Good Citizen* dalam pemrograman backend untuk menjaga kesehatan server.
