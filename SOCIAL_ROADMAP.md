# 🗺️ JinwooStream Social & Content Roadmap (The "Fame" Update)

_Status Document for User & Agent Tracking_

This document tracks the implementation of the massive Social Ecosystem update.

---

## ✅ Phase 1: Social Core (Fondasi Sosial)

Target: **User bisa saling Follow & punya Profile Stats.**

- [ ] **Database Schema:** Create `Follow` Model.
- [ ] **User Model:** Update User schema to support follower/following counts.
- [ ] **API:** `POST /api/social/follow` (Toggle Follow).
- [ ] **API:** `GET /api/social/stats` (Get Counts).
- [ ] **UI (Profile):** Show Follower/Following Count.
- [ ] **UI (Profile):** Add "Follow" Button on public profiles.

---

## ⏳ Phase 2: The Leaderboard (Panggung Kompetisi)

Target: **User berlomba masuk Ranking.**
_Prerequisites: Phase 1 Complete_

- [ ] **Frontend:** Create `/leaderboard` Page.
- [ ] **Tab 1 (Global):** Fetch & Sort Users by EXP (Rank system).
- [ ] **Tab 2 (Popularity):** Fetch & Sort Users by Follower Count.
- [ ] **Tab 3 (Circle):** Filter Global Rank by "Friends Only" (Mutuals).
- [ ] **Components:** Design `LeaderboardRow` with Rank Badge styling.

---

## ⏳ Phase 3: Blog System (Suara Komunitas)

Target: **D-Hunter+ bisa menulis artikel.**
_Prerequisites: Phase 1 & 2 Complete (Recommended)_

- [ ] **Database Schema:** Create `Blog` Model.
- [ ] **Access Control:** Middleware to check `Rank >= D-Hunter` before Create/Update.
- [ ] **API:** `POST /api/blog/create` (With Validation).
- [ ] **API:** `GET /api/blog` (Public Feed).
- [ ] **Sidebar Menu:** Add "Upload Blog" & "View Blog" (Conditional Rendering).
- [ ] **Page (Upload):** Simple Editor UI.
- [ ] **Page (View):** Analytics Table (Views, Likes, Comments).
- [ ] **Moderation:** Implement "Report" Logic (Auto-hide threshold).

---

## 📝 Notes & Decisions

- **Philosophy:** "From Features to Fame".
- **Friend Definition:** Mutual Follow (A follows B, B follows A).
- **Restriction:** Blog feature only for D-Hunter and above.
