# Product Requirement Document (PRD) - GitHub Radar (V1)

## 1. Project Overview
- **Project Name:** GitHub Radar
- **Summary:** Aplikasi untuk memantau repo GitHub yang lagi naik. Data diambil dari GitHub API, disimpan sebagai riwayat, lalu ditampilkan lewat dashboard, watchlist, AI chat, dan digest harian.
- **Unique Value Proposition:**
  - Punya **riwayat tren** (bukan cuma snapshot hari ini seperti GitHub Trending).
  - Ada **penjelasan AI** kenapa sebuah repo naik, berdasarkan data kita sendiri.

## 2. Tech Stack & Infrastructure
- **Framework:** Next.js (App Router)
- **Database & Jadwal:** Supabase (PostgreSQL, `pg_cron`, Edge Function)
- **Sumber Data:** GitHub REST API (wajib pakai token agar batas request cukup)
- **AI:** Format OpenAI-compatible lewat SDK `openai`. Provider bebas diganti (contoh: OmniRoute, OpenRouter, dll).
- **UI/Styling:** Tailwind CSS, shadcn/ui (Recharts untuk grafik, Framer Motion untuk animasi)
- **Konfigurasi (`.env`):**
  - `NEXT_PUBLIC_SUPABASE_URL`: alamat project Supabase
  - `NEXT_PUBLIC_SUPABASE_ANON_KEY`: key publik (dipakai browser)
  - `SUPABASE_SERVICE_ROLE_KEY`: key server (rahasia, tidak boleh sampai ke browser)
  - `GITHUB_TOKEN`: wajib, dibutuhkan untuk backfill dan pengumpulan data (tanpa token limit cuma 60 request per jam)
  - `AI_BASE_URL`, `AI_API_KEY`, `AI_MODEL`: konfigurasi AI (format OpenAI-compatible)
  - `CRON_SECRET`: pengaman endpoint yang dipanggil cron
  - `APP_URL`: alamat app, supaya cron bisa memanggil endpoint digest

## 3. User Roles
- **Pemilik app (satu pengguna):** Mengakses semua halaman.

## 4. Page Structure
- `/` (Dashboard): Daftar repo yang lagi naik, kartu ringkasan, grafik star velocity, filter waktu (hari ini, 7 hari, 30 hari).
- `/repo/[owner]/[name]` (Detail Repo): Info repo, grafik 30 hari, link ke GitHub, tombol Tambah Watchlist dan Tanya AI, ringkasan AI.
- `/watchlist`: Repo yang dipantau.
- `/chat` (AI Chat): Tanya jawab soal repo dan tren.
- `/digest`: Riwayat ringkasan harian dari AI. Halaman ini dibuka manual, belum ada notifikasi di V1.

## 5. Core Features

### Feature 1: Pengumpulan Data (Otomatis)
- **Flow:** `pg_cron` jalan tiap 6 jam -> Edge Function ambil data repo dari GitHub API -> simpan snapshot (star, fork, issue) ke database -> hitung kecepatan naik (velocity).
- **Catatan:** Hanya bagian ini yang menembak GitHub secara rutin. Halaman lain membaca dari database. Pengecualian: panggilan sekali jalan saat user menambah repo baru (lewat Watchlist atau Detail Repo).
- **Backfill (sekali, saat setup):** Cari repo yang dibuat dalam 30 hari terakhir lewat GitHub Search API (urut star, ambil 30 teratas) -> tarik riwayat star bertanggal lewat endpoint stargazers -> simpan sebagai snapshot harian. Dikerjakan bertahap per beberapa repo supaya tidak kena batas waktu Edge Function dan rate limit GitHub.
- **Penemuan repo baru (tiap siklus, 6 jam):** Setelah memperbarui snapshot repo yang sudah dipantau, Edge Function mencari repo yang dibuat dalam 7 hari terakhir lewat GitHub Search API (urut star). Repo yang belum ada di database dan minimal 50 star ditambahkan ke daftar pantau (maksimal 10 repo baru per siklus) lalu di-backfill. Total repo aktif (di luar watchlist) dibatasi maksimal 100. Kalau penuh, repo dengan velocity terendah selama 14 hari dikeluarkan.
- **Data pendukung:** Commit mingguan, rilis terbaru, dan potongan README diambil saat repo pertama masuk daftar pantau, lalu diperbarui sekali sehari (bukan tiap 6 jam, supaya hemat request).

### Feature 2: Dashboard
- **Flow:** Buka `/` -> App baca database -> Tampil kartu ringkasan, grafik, daftar repo naik -> Klik repo -> Detail Repo.
- **Filter waktu:** Hari ini / 7 hari / 30 hari (default 7 hari). Menentukan jendela waktu penghitungan velocity (rata-rata star per hari dalam periode itu). Yang ikut berubah: Total stars gained, Fastest riser, grafik, dan urutan serta angka velocity di daftar repo naik. Jumlah repo yang dipantau tidak ikut berubah.
- **Kartu ringkasan:** (1) Repos tracked: jumlah repo yang dipantau, plus jumlah repo baru yang ditemukan dalam periode terpilih. (2) Total stars gained: total star yang didapat semua repo di periode terpilih, plus persentase perubahan dibanding periode sebelumnya yang sama panjang. (3) Fastest riser: repo dengan velocity tertinggi di periode terpilih beserta angkanya (stars/day).
- **Grafik:** star velocity 5 repo teratas di periode terpilih.
- **Daftar repo naik:** diurutkan berdasarkan velocity, hanya repo dengan tren naik. Awalnya 6 teratas, sisanya dimuat bertahap lewat "Muat lebih banyak" (12 per klik).
- **Catatan:** "Naik" dihitung sendiri dari selisih snapshot, bukan salinan halaman GitHub Trending. Grafik langsung terisi data asli lewat backfill (lihat Feature 1), dan selama backfill berjalan dashboard menampilkan status progres, bukan data demo. Waktu pembaruan data terakhir ditampilkan. Repo lama yang baru viral tidak otomatis muncul, tapi bisa ditambahkan lewat watchlist.

### Feature 3: Detail Repo
- **Flow:** Klik repo -> Tampil info, grafik 30 hari, commit mingguan, ringkasan AI -> Tombol "Buka di GitHub", "Tambah Watchlist", atau "Tanya AI".
- **Tombol "Tanya AI":** membuka `/chat` dengan repo tersebut sudah terpilih sebagai konteks pertanyaan.
- **Catatan:** Kalau repo belum ada di database, app ambil dulu dari GitHub API lalu simpan. Ringkasan AI disimpan (cache) supaya tidak memanggil AI setiap halaman dibuka.

### Feature 4: Watchlist
- **Flow:** Klik "Tambah" -> Cari repo (lewat GitHub Search) -> Pilih -> Masuk watchlist -> Repo dipantau dan datanya ikut dikumpulkan tiap siklus `pg_cron`.
- **Catatan:** Repo watchlist tidak ikut batas 100 repo aktif dan tidak pernah dikeluarkan otomatis, tapi dibatasi maksimal 50 repo supaya rate limit GitHub aman. Pemantauan topik belum ada di V1.
- **Hapus dari watchlist:** klik tombol simpan sekali lagi (toggle). Repo tidak dihapus dari database, statusnya kembali jadi repo biasa: tetap dilacak, ikut batas 100 repo aktif, dan bisa dikeluarkan otomatis kalau velocity-nya paling rendah.

### Feature 5: AI Chat
- **Flow:** User bertanya -> Server ambil data repo dari database (snapshot 14 hari, commit, rilis, potongan README) -> Data diubah jadi teks dan dikirim ke AI bersama pertanyaan -> AI menyusun jawaban -> Jawaban tampil bertahap (streaming) beserta sumber datanya.
- **Catatan:** AI tidak punya akses ke database, GitHub, atau internet. Ia hanya membaca teks yang dikirim server, jadi jawabannya adalah analisis dari data, **bukan kepastian penyebab**. Semua panggilan AI lewat satu file `lib/ai.ts`. Tidak bergantung pada tool calling, jadi jalan di model gratis. Karena V1 tanpa login, endpoint chat wajib dibatasi rate limit per IP supaya token AI tidak habis disalahgunakan.

### Feature 6: Digest Harian
- **Flow:** `pg_cron` sekali sehari memanggil endpoint app -> Server ambil top repo naik dan perubahan watchlist -> AI merangkum dalam satu panggilan -> Disimpan ke database -> Tampil di halaman `/digest`.
- **Catatan:** Beda dengan dashboard: dashboard menampilkan angka dan grafik, digest menampilkan ringkasan bahasa manusia. Satu panggilan AI per hari, jadi biaya token kecil.
