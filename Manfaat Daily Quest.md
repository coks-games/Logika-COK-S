# JinwooStream Social & Content Ecosystem (The "Fame" Update)

## 🧠 Core Philosophy

Transformasi dari "Platform Nonton" menjadi "Social Platform Anime".  
Karena fitur premium (No Ads/Download VIP) tidak relevan, kita beralih ke benefit **Status Sosial & Eksistensi**. User berlomba untuk menjadi **Terkenal (Influencer)** dan **Terkuat (Top Rank)**.

---

## 1. Social Graph System (The Connection)

Membangun jaringan antar Hunter.

- **Follow System:** User A follow User B.
- **Friend System (Mutua):** Terjadi otomatis jika **Saling Follow**.
- **Benefits:**
  - Melihat aktivitas teman.
  - Persaingan di _Circle Leaderboard_.

## 2. Leaderboard System (The Arena)

Halaman khusus `/leaderboard` dengan 3 kategori tab:

- **🌍 Global Rank:** Urutan berdasarkan **EXP / Hunter Rank** (E s/d National Level).
- **👑 Popularity Rank:** Urutan berdasarkan **Jumlah Follower**. (Influencer Chart).
- **🤝 Circle Rank:** Urutan berdasarkan EXP tapi **Hanya di antara Teman Sendiri**.

### 🏆 Follower Milestones (Badges)

Reward visual untuk jumlah follower tertentu (Tampil di Profile/Leaderboard):

- **100:** 🥉 Local Celeb
- **250:** 🥈 Rising Star
- **500:** 🥇 Star Hunter
- **1k:** 💎 Idol
- **5k:** 🔥 Supernova
- **10k:** 👑 King/Queen
- **...10M:** 🌌 National Treasure

---

## 3. Blog / Content Creator System (The Voice)

User bisa menjadi kontributor konten (Jurnalis Anime).

### 🔒 Access Control (Rank Gate)

- **E-Hunter:** ❌ **Viewer Only**. (Bisa baca, like, share, report).
- **D-Hunter (+):** ✅ **Creator Access**. (Bisa tulis & upload blog).

### 🛡️ Moderation System (The Law)

Menjaga profesionalisme tanpa membebani Admin.

1.  **Strict Rule:** Konten Wajib Anime/Jejepangan.
2.  **Community Policing:** Tombol "Report: Non-Anime".
    - _Threshold:_ 5 Report = **Auto-Takedown (Hidden)**.
3.  **Admin Nuke:** Hapus permanen + Ban fitur blog user.

### 📊 Dashboard Integration

Sidebar Menu untuk Creator:

1.  **Upload Blog:**
    - Editor UI (Judul, Thumbnail, Body, Tag).
    - Validation: Min 1 image, min X words.
2.  **View Blog (Analytics):**
    - List artikel yang pernah dibuat.
    - **Live Stats:** Views 👁️, Likes ❤️, Comments 💬, Shares 🔗.
    - Actions: Edit / Delete.

---

## 4. Technical Roadmap

1.  **Database:**
    - `Follow` Model (followerId, followingId).
    - `Blog` Model (title, content, author, stats, reports).
2.  **API:**
    - Social: Follow/Unfollow, Get Followers.
    - Blog: CRUD endpoints with Rank Checks.
3.  **Frontend:**
    - Leaderboard UI.
    - Blog Editor & Viewer UI.
    - Profile Upgrade (Show Badges & Follower Count).
