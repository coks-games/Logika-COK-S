# Logika Diskon (Discount System)

Dokumen ini menjelaskan bagaimana sistem diskon otomatis bekerja di platform ini. Sistem ini dirancang untuk memberikan "surprise" setiap hari kepada user tanpa perlu intervensi manual dari admin setiap saat.

## 1. Logika Utama Diskon

Sistem diskon menggunakan hierarki prioritas untuk menentukan harga terbaik bagi user. Tidak ada diskon yang bertumpuk (stacking), sistem akan selalu memilih satu diskon dengan prioritas tertinggi.

### Hierarki Prioritas:
1.  **Flash Sale (99%):** Prioritas TERTINGGI. Hanya muncul saat Event Khusus & Stok masih ada.
2.  **Special Event (25%):** Muncul saat ada event besar (e.g. 17 Agustus, Natal, Lebaran) untuk game kategori PRO.
3.  **Daily Deal (10%):** Diskon harian yang merotasi game secara acak namun deterministik (semua user melihat game yang sama).

### Fitur "Daily Deal" (Diskon Harian)
Bagaimana sistem memilih game mana yang didiskon hari ini?
*   **Rotasi Otomatis 24 Jam:** Setiap pergantian hari (jam 00:00), game yang didiskon akan berubah.
*   **Deterministik:** Menggunakan tanggal hari ini sebagai "Seed" untuk pengacakan. Artinya, jika Budi membuka website jam 8 pagi dan Wati membukanya jam 8 malam, mereka berdua akan melihat game diskon yang SAMA.
*   **Adil:** Algoritma memastikan setiap game punya peluang yang sama untuk terpilih.

---

## 2. File-File Terkait

1.  **`utils/DiscountManager.js`**: "Otak" dari sistem diskon. Berisi rumus matematika untuk pengacakan harian dan logika penentuan jenis diskon.
2.  **`controllers/gameController.js`**: Menggunakan *DiscountManager* untuk menampilkan harga coret di halaman Home (`landing`), List Game (`index`), dan Detail Game (`show`).
3.  **`controllers/paymentController.js`**: Menggunakan *DiscountManager* untuk menghitung harga final saat user melakukan pembayaran. (PENTING: Harga dihitung ulang di sini demi keamanan).

---

## 3. Detail Implementasi & Source Code

Mari kita bedah kode di balik fitur ini.

### A. Algoritma Daily Deal (RNG Deterministik)

**File:** `utils/DiscountManager.js`

```javascript
static getDailyDealIndex(totalGames) {
    if (totalGames === 0) return -1;
    
    const today = new Date();
    // 1. Buat "Seed" dari Tanggal Hari Ini (Format: YYYYMMDD)
    // Contoh: 18 Desember 2025 -> 20251218
    const seed = parseInt(
        today.getFullYear().toString() + 
        (today.getMonth() + 1).toString().padStart(2, '0') + 
        today.getDate().toString().padStart(2, '0')
    );

    // 2. Linear Congruential Generator (LCG)
    // Rumus matematika sederhana untuk mengacak angka berdasarkan Seed.
    // Karena Seed-nya sama (tanggal hari ini), hasil acaknya pun PASTI SAMA seharian ini.
    const a = 1664525;
    const c = 1013904223;
    const m = 4294967296;
    
    const randomVal = (a * seed + c) % m;
    
    // 3. Modulo dengan Total Game
    // Agar hasilnya tidak melebihi jumlah game yang ada di database.
    return randomVal % totalGames;
}
```
**Alasannya:** Kita tidak menggunakan `Math.random()` standar karena itu akan menghasilkan game berbeda setiap kali page di-refresh. Kita ingin "Deal of the Day" yang konsisten untuk semua user.

### B. Kalkulator Diskon Terpusat

**File:** `utils/DiscountManager.js` - `calculateDiscount`

```javascript
static calculateDiscount(game, isFlashSaleAvailable = false, activeEvent = null) {
    const originalPrice = parseInt(game.price || 0);

    // Cek Apakah Hari Ini Hari Spesial? (Dapat dari database atau kalender manual)
    const eventStatus = this.getEventStatus(activeEvent);

    // PRIORITAS 1 & 2: Special Event (Untuk Game Kategori 'Pro')
    if (eventStatus.isSpecial && (game.category || '').toLowerCase().includes('pro')) {
        if (isFlashSaleAvailable) {
            // Jackpot! Flash Sale 99%
            return {
                finalPrice: Math.floor(originalPrice * 0.01), 
                discountType: 'flash_sale',
                discountPercent: 99
            };
        } else {
            // Diskon Event Regular 25%
            return {
                finalPrice: Math.floor(originalPrice * 0.75), 
                discountType: 'event',
                discountPercent: 25
            };
        }
    }

    // PRIORITAS 3: Daily Deal
    // Flag 'isDailyDeal' dikirim dari Controller yang memanggil fungsi ini
    if (game.isDailyDeal) {
        return {
            finalPrice: Math.floor(originalPrice * 0.90), // Diskon 10%
            discountType: 'daily',
            discountPercent: 10
        };
    }

    // Tidak Ada Diskon
    return {
        finalPrice: originalPrice,
        discountType: 'none',
        discountPercent: 0
    };
}
```
**Alasannya:** Logika terpusat ini mencegah *bug* "harga beda". Jangan sampai di halaman Home diskonnya 50% tapi pas mau bayar cuma 10%. Dengan satu fungsi pusat, konsistensi terjamin.

### C. Implementasi di Controller

**File:** `controllers/gameController.js`

```javascript
// Di fungsi landing (Halaman Home)
const totalGames = await Game.count();

// 1. Cari Index Game Pilihan Hari Ini
const dailyIndex = DiscountManager.getDailyDealIndex(totalGames);

if (dailyIndex >= 0) {
    // 2. Ambil data game dari database berdasarkan index tadi
    const rawGame = await Game.findByOffset(dailyIndex);
    
    // 3. Hitung harga diskonnya
    if (rawGame) { 
        const discountInfo = DiscountManager.calculateDiscount(
            {...rawGame, isDailyDeal: true} // Paksa flag True karena ini memang game pilihannya
        );
        
        // Kirim ke view EJS untuk ditampilkan banner-nya
        dailyDealGame = { ...rawGame, ...discountInfo };
    }
}
```
