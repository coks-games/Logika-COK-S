# Logika Pembayaran (Payment System)

Dokumen ini menjelaskan bagaimana jantung finansial dari website ini bekerja. Sistem pembayaran ini mengatur segalanya mulai dari user memilih game, mendapatkan diskon, memproses transaksi ke DOKU Payment Gateway, hingga memastikan status pembayaran terupdate secara *real-time*.

## 1. Logika Utama Pembayaran

Alur pembayaran didesain agar aman (anti-curang) namun tetap nyaman bagi pengguna.

### A. Halaman Checkout (`/payment/buy/:slug`)
Saat user ingin membeli game:
1.  **Cek Kelayakan:** Sistem memastikan game ini berbayar dan user belum memilikinya.
2.  **Hitung Diskon:** Sistem mengecek apakah ada diskon yang berlaku hari ini (Flash Sale 99%, Daily Deal 10%, atau Event Discount 25%).
3.  **Tampilan:** User disuguhkan harga coret (jika diskon) dan pilihan metode pembayaran (QRIS, VA, E-Wallet, Retail).

### B. Proses Pembayaran (`Proceed to Payment`)
Saat tombol ditekan, ini yang terjadi di belakang layar:
1.  **Validasi Ulang (Critical):** Harga dan diskon dihitung ulang di server. *Kita tidak percaya harga yang dikirim dari browser user karena bisa dimanipulasi.*
2.  **Cek Stok Flash Sale:** Khusus Flash Sale 99%, sistem melakukan "Race Condition Check" untuk memastikan slot pemenang belum diambil orang lain di detik yang sama.
3.  **Request ke DOKU:** Server mengirim data order ke DOKU untuk mendapatkan URL pembayaran / Kode VA.
4.  **Simpan Transaksi:** Data transaksi disimpan ke database dengan status `waiting`.

### C. Halaman Status (`/payment/:orderId/status`)
Halaman ini adalah ruang tunggu user setelah melakukan checkout.
1.  **Countdown Timer:** Menampilkan sisa waktu pembayaran (Default 24 jam / Flash Sale 15 menit).
2.  **Auto-Update:** Halaman ini **tidak perlu di-refresh manual**. Ada script AJAX yang otomatis mengecek status ke server setiap 5 detik. Jika status berubah (misal: sukses bayar), tampilan langsung berubah menjadi hijau (Sukses).

### D. Riwayat Pesanan (`/payment/history`)
Menampilkan daftar semua pembelian user, dari yang sukses sampai yang gagal, diurutkan dari yang terbaru.

---

## 2. File-File Terkait

1.  **`routes/payment.js`**: "Polisi Lalu Lintas" yang mengatur URL pembayaran.
2.  **`controllers/paymentController.js`**: "Otak Utama" yang menghitung diskon, menghubungi DOKU, dan memproses transaksi.
3.  **`models/Transaction.js`**: "Gudang Data" yang menyimpan riwayat transaksi ke database `transactions`.
4.  **`views/payment/checkout.ejs`**: Tampilan formulir pembelian.
5.  **`views/payment/status.ejs`**: Tampilan status pembayaran dengan timer & auto-refresh.
6.  **`views/payment/history.ejs`**: Tampilan daftar riwayat pesanan.

---

## 3. Detail Implementasi & Source Code

Berikut adalah bedah kode dari logika-logika vital di atas.

### A. Persiapan Checkout (Menghitung Diskon)

**File:** `controllers/paymentController.js` (Function `checkout`)

```javascript
// Logika Diskon Pintar
// 1. Cek dulu, user sudah belanja diskon hari ini belum? (Jatah 1x sehari)
const hasClaimed = await Transaction.hasClaimedDiscountToday(req.session.user.id);

if (!hasClaimed) {
   // 2. Tentukan Game Harian (Daily Deal)
   const dailyIndex = DiscountManager.getDailyDealIndex(totalGamesCount);
   const isDaily = (dailyDealGame.id === game.id);

   // 3. Ambil Event Aktif (misal: Event Lebaran)
   const activeEvent = await Event.getActive();

   // 4. Hitung Diskon Final (Prioritas: Flash Sale > Daily > Event)
   discount = DiscountManager.calculateDiscount(
       {...game, isDailyDeal: isDaily}, 
       true, // Cek ketersediaan Flash Sale
       activeEvent
   ); 
}
```
**Alasannya:** Kita ingin memberikan pengalaman terbaik. User langsung tahu harga terbaik yang bisa mereka dapatkan saat membuka halaman checkout.

### B. Memproses Pembayaran (Security & DOKU Integration)

**File:** `controllers/paymentController.js` (Function `processPayment`)

```javascript
// ... (Awal fungsi)
// 1. Mulai Transaksi Database (Safety First)
// Agar jika ada error di tengah jalan, database tidak rusak/inconsistent
await client.query('BEGIN');

// 2. [SECURITY] Hitung Ulang Harga & Diskon di Server
// Jangan pernah percaya input harga dari Frontend!
let finalAmount = originalPrice;
// ... (Logika perhitungan ulang harga sama seperti di checkout) ...

// 3. [ANTI-BENTROK] Cek Stok Pemenang Flash Sale
if (discountType === 'flash_sale') {
    const winnerCount = await client.query(`...cek jumlah pemenang hari ini...`);
    if (winnerCount > 0) {
        // Maaf, Anda telat sedetik! Flash sale hangus, harga kembali normal/diskon event.
        potentialDiscount = DiscountManager.calculateDiscount(..., false, ...);
    }
}

// 4. Panggil DOKU Payment Gateway
const dokuResponse = await createPayment({
    order: {
        invoice_number: orderId,
        amount: parseInt(finalAmount), // Harga final yang aman
        currency: 'IDR'
    },
    // ... data customer ...
});

// 5. Simpan ke Database
await client.query(`INSERT INTO transactions ... VALUES ...`);

// 6. Selesai! Redirect user ke Invoice
res.json({ success: true, redirect_url: `/payment/${orderId}/invoice` });
```
**Alasannya:** Keamanan adalah prioritas #1 dalam pembayaran. Kita menghitung ulang semua harga di server untuk mencegah hacker memanipulasi request dan membeli game mahal seharga Rp 1.

### C. Halaman Status (Auto-Refresh tanpa Reload)

**File:** `views/payment/status.ejs` (Frontend Logic)

```javascript
 <script>
    // 1. Timer berjalan mundur
    setInterval(updateTimer, 1000); // Tiap 1 detik

    // 2. [FITUR UTAMA] Cek Status ke Server Diam-diam (AJAX)
    async function checkPaymentStatus() {
        // Panggil API status
        const response = await fetch(`/payment/${orderId}/check`);
        const data = await response.json();
        
        // Jika status di server beda dengan di layar saat ini
        if (data.status !== currentStatus) {
            // Update Tampilan UI (Ganti Icon & Tombol)
            updateStatusUI(data.status);
        }
    }

    // Jalankan cek status tiap 5 detik
    setInterval(checkPaymentStatus, 5000);
 </script>
```
**Alasannya:** User experience yang modern. User tidak perlu capek-capek menekan F5/Refresh untuk mengecek apakah pembayaran QRIS mereka sudah masuk. Sistem yang bekerja untuk mereka.

### D. Riwayat Pesanan (`Transactions Model`)

**File:** `models/Transaction.js`

```javascript
static async getByUserId(userId) {
    // Ambil transaksi + Join tabel Games (biar dapat judul & gambar game)
    const query = `
      SELECT t.*, g.title as game_title, g.slug as game_slug, g.thumbnail_url
      FROM transactions t
      JOIN games g ON t.game_id = g.id
      WHERE t.user_id = $1
      ORDER BY t.created_at DESC `; // Urutkan dari yang terbaru
    
    const result = await db.query(query, [userId]);
    return result.rows; 
}
```
**Alasannya:** Query `JOIN` digunakan agar dalam satu kali pengambilan data, kita sudah lengkap mendapatkan info transaksi sekaligus info gamenya. Efisien dan cepat.
