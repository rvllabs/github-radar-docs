# Product Requirement Document (PRD) — GitHub Radar

## 1. Gambaran Umum

- **Nama project:** GitHub Radar
- **Ringkasan:** Aplikasi untuk memantau repo GitHub yang lagi naik. Data diambil dari GitHub API secara berkala, disimpan sebagai riwayat (time-series), lalu ditampilkan lewat Dashboard, Watchlist, Detail Repo, AI Chat, dan Digest harian (halaman News).
- **Nilai jual utama:**
  - Punya **riwayat tren** (bukan cuma snapshot hari ini seperti GitHub Trending).
  - Ada **penjelasan AI** kenapa sebuah repo naik, berdasarkan data kita sendiri.
  - Ada **berita harian** ala portal berita: repo apa yang lagi trending dan apa yang baru di repo yang kita pantau.

## 2. Tech Stack dan Infrastruktur

- **Framework:** Next.js 16 (App Router, TypeScript strict). Memakai `after()` dan `connection()` dari Next 16.
- **Database dan jadwal:** Supabase (PostgreSQL, `pg_cron`, `pg_net`, Edge Function). Tanpa Supabase Auth dan tanpa Vault.
- **Sumber data:** GitHub REST API dengan token (classic PAT scope `public_repo`, batas 5000 request per jam).
- **AI:** format OpenAI-compatible lewat SDK `openai`. Provider bebas diganti lewat env (OmniRoute, OpenRouter, dan lain-lain).
- **UI:** Tailwind CSS v4, shadcn/ui (Base UI), Recharts untuk grafik, Framer Motion untuk animasi, Lucide React untuk ikon.
- **Data client:** `@tanstack/react-query` (hanya dipakai di bagian yang memang interaktif).
- **Render markdown:** `react-markdown` + `remark-gfm` + `rehype-raw` + `rehype-sanitize` (HTML dari pihak ketiga selalu disanitasi).
- **Test:** Vitest.
- **Hosting:** Vercel region Singapore (`sin1`), sejajar dengan Supabase region `ap-southeast-1`.

### Konfigurasi `.env` (10 nama)

- `NEXT_PUBLIC_SUPABASE_URL`: alamat project Supabase.
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`: key publik (dipakai browser, hanya baca lewat RLS).
- `SUPABASE_SERVICE_ROLE_KEY`: key server (rahasia, tidak boleh sampai ke browser).
- `GITHUB_TOKEN`: wajib. Tanpa token limit cuma 60 request per jam.
- `AI_BASE_URL`, `AI_API_KEY`, `AI_MODEL`: konfigurasi AI (format OpenAI-compatible).
- `CRON_SECRET`: pengaman endpoint yang dipanggil cron.
- `APP_URL`: alamat app, supaya cron bisa memanggil endpoint digest. Isi localhost saat development, **ganti ke URL produksi setelah deploy**.
- `RATE_LIMIT_HMAC_SECRET`: secret untuk meng-hash IP sebelum disimpan di tabel rate limit.

Secret Edge Function (`GITHUB_TOKEN`, `CRON_SECRET`) diisi manual di Dashboard Supabase dengan nilai yang sama persis dengan `.env.local`.

## 3. Pengguna

- **Pemilik app (satu pengguna):** mengakses semua halaman. tanpa login. Karena itu semua endpoint tulis dilindungi rate limit, dan disarankan menambah proteksi di level hosting (password protection atau Basic Auth) selama belum ada login.

## 4. Struktur Halaman

- `/` — **Dashboard:** kartu ringkasan, grafik star velocity, daftar repo yang lagi naik, filter waktu (Today, 7 Days, 30 Days).
- `/repo/[owner]/[name]` — **Detail Repo:** info repo, grafik 30 hari, commit mingguan, ringkasan AI, rilis terbaru, info repo, README penuh, tombol Ask AI, Add to Watchlist, dan Open on GitHub.
- `/watchlist` — **Watchlist:** repo yang dipantau (maksimal 50), kotak cari, dialog Add repo.
- `/chat` — **AI Chat:** tanya jawab soal repo dan tren, riwayat tersimpan multi-sesi.
- `/digest` — **News** (label di navigasi: "News", judul halaman: "What's new today"): riwayat berita harian dari AI. Dibuka manual.

Navigasi lewat top bar melayang (desktop) dan tab bar melayang di bawah (mobile). Tanpa sidebar.

## 5. Fitur Inti

### Fitur 1: Pengumpulan Data (Otomatis)

- **Alur utama:** `pg_cron` jalan tiap 6 jam, memanggil Edge Function `collect-repos`. Function itu mengambil data repo dari GitHub API, menyimpan snapshot (star, fork, issue), lalu menghitung kecepatan naik (velocity).
- **Catatan:** hanya bagian ini (plus backfill dan digest) yang menembak GitHub secara rutin. Halaman lain membaca dari database. Pengecualian: panggilan sekali jalan saat user menambah repo baru lewat Watchlist atau membuka Detail Repo lewat URL langsung.
- **Backfill (sekali, saat setup):** cari repo yang dibuat dalam 30 hari terakhir lewat GitHub Search API (urut star, ambil 30 teratas), lalu tarik riwayat star lewat endpoint `/stargazers/history` (bintang per hari, dikelompokkan per minggu), simpan sebagai snapshot harian. Dikerjakan bertahap per beberapa repo oleh job `backfill-batch` supaya tidak kena batas waktu Edge Function dan rate limit GitHub.
- **Kenapa `/stargazers/history`:** sejak Juni/Juli 2026 GitHub membatasi endpoint daftar stargazers per-user hanya untuk admin atau kolaborator. Endpoint `/stargazers/history` masih terbuka dan lebih hemat (30 minggu per halaman, tanpa perlu sampling atau interpolasi).
- **Penemuan repo baru (tiap siklus 6 jam):** setelah memperbarui snapshot repo yang sudah dipantau, function mencari repo yang dibuat dalam 7 hari terakhir (urut star). Repo yang belum ada di database dan minimal 50 star ditambahkan (maksimal 10 repo baru per siklus), lalu di-backfill. Total repo aktif di luar watchlist dibatasi 100. Kalau penuh, repo dengan velocity 14 hari terendah dikeluarkan (status `evicted`, baris tidak dihapus).
- **Data pendukung:** commit mingguan, rilis terbaru, potongan README, dan **20 pesan commit terbaru** diambil saat repo pertama masuk daftar pantau, lalu diperbarui paling cepat sekali per 24 jam. Pesan commit terbaru tidak ditampilkan di UI; ia hanya bahan konteks AI supaya bisa menjawab "apa yang baru di repo ini".

### Fitur 2: Dashboard

- **Alur:** buka `/`, app membaca database, tampil kartu ringkasan, grafik, dan daftar repo naik. Klik repo menuju Detail Repo.
- **Baris info:** teks "Terakhir diperbarui N jam yang lalu • disegarkan setiap 6 jam" di kiri, filter waktu di kanan.
- **Filter waktu:** Today / 7 Days / 30 Days (default 7 Days, disimpan di URL `?range=`). Filter menentukan jendela penghitungan velocity (rata-rata star per hari). Yang ikut berubah: Stars gained, Fastest riser, grafik, serta urutan dan angka velocity di daftar. Jumlah repo yang dipantau tidak berubah.
- **Tiga kartu ringkasan:**
  - Repos tracked: jumlah repo aktif dipantau, plus jumlah repo baru dalam periode terpilih (contoh "+ 111 new in 7 days").
  - Stars gained: total star yang didapat semua repo di periode terpilih, plus persentase perubahan dibanding periode sebelumnya yang sama panjang. Kalau periode sebelumnya 0 atau tidak ada, tampil "-". Persen dihitung hanya dari repo yang punya data periode sebelumnya. Persen negatif tampil merah dengan ikon turun.
  - Fastest riser: repo dengan velocity tertinggi beserta angkanya.
- **Grafik Star velocity:** 5 repo teratas. Yang digambar adalah **velocity per interval** (selisih bintang dibagi lama interval), bukan total bintang kumulatif, sehingga repo dengan velocity tertinggi selalu berada paling atas. Ada legenda, tooltip yang mengikuti garis yang ditunjuk kursor, dan label tanggal tepat di bawah titiknya.
- **Daftar Rising now:** diurutkan berdasarkan velocity, hanya repo dengan velocity lebih dari 0. Awalnya 6 teratas, sisanya lewat tombol "Load more" (12 per klik) dengan caption "Showing X of Y".
- **Format velocity di semua tempat:** `+8,103 stars / day` (satu format yang sama di kartu, header, dan digest).
- **Selama backfill berjalan:** Dashboard menampilkan kartu progres setup (bukan data demo). Tidak ada data demo di aplikasi ini.

### Fitur 3: Detail Repo

- **Alur:** klik repo, tampil info, grafik 30 hari, commit mingguan, ringkasan AI, rilis, info repo, dan README. Tombol: Ask AI, Add to Watchlist (toggle), Open on GitHub.
- **Ask AI:** membuka `/chat?repo=owner/name` dengan repo tersebut sudah menjadi konteks.
- **Repo belum ada di database:** app mengambil dulu dari GitHub API lalu menyimpan (dibatasi rate limit). Kalau repo memang tidak ada di GitHub, tampil halaman 404. Repo yang berstatus `evicted` tetap tampil dengan penanda "Tidak lagi dilacak".
- **Ringkasan AI:** disimpan (cache 24 jam) supaya AI tidak dipanggil tiap halaman dibuka. Halaman tampil dulu, ringkasan AI menyusul. Kalau cache basi tapi masih ada isi lama, isi lama tetap tampil dan diperbarui di belakang layar.
- **README:** ditampilkan penuh (diambil langsung dari GitHub, di-cache 1 jam), HTML di dalamnya disanitasi.
- **Kartu Latest release:** selalu tampil. Kalau repo belum punya rilis, tampil pesan kosong dengan link ke halaman rilis GitHub.

### Fitur 4: Watchlist

- **Alur:** klik "Add repo", cari repo lewat GitHub Search, pilih, repo masuk watchlist dan dikumpulkan tiap siklus.
- **Batas:** maksimal 50 repo. Repo watchlist tidak ikut batas 100 repo aktif dan tidak pernah dikeluarkan otomatis.
- **Toggle:** klik tombol simpan sekali lagi untuk menghapus dari watchlist. Repo tidak dihapus dari database; statusnya kembali jadi repo biasa yang tetap dilacak.
- **Kotak cari:** tepat di bawah header halaman, selebar kontainer, memfilter daftar watchlist langsung di client (tanpa request tambahan).
- **Kartu repo:** memakai komponen kartu yang sama persis dengan Dashboard.

### Fitur 5: AI Chat

- **Alur:** user bertanya, server mengambil data repo dari database (snapshot 14 hari, commit mingguan, rilis, pesan commit terbaru, potongan README) atau data tren umum, data diubah jadi teks dan dikirim ke AI bersama pertanyaan, jawaban tampil bertahap (streaming) beserta chip sumber datanya.
- **Batasan AI:** AI tidak punya akses ke database, GitHub, atau internet. Ia hanya membaca teks yang dikirim server, jadi jawabannya adalah analisis dari data, **bukan kepastian penyebab**. Semua panggilan AI lewat satu file `lib/ai.ts`. Tanpa tool calling sehingga jalan di model gratis.
- **Riwayat tersimpan di database, multi-sesi:** ada sidebar percakapan, tombol New chat, dan tombol Bersihkan riwayat. Jawaban yang gagal ikut tersimpan dengan status gagal.
- **Konteks repo menempel di sesi (pin):** sesi yang dimulai dari repo tertentu tetap ingat repo itu walau dibuka lagi tanpa parameter. Kalau request berikutnya membawa repo yang berbeda, dibuat sesi baru. Tombol **New chat selalu mulai bersih** tanpa konteks repo.
- **Aksi pada pesan:** Copy dan Retry (buat ulang jawaban terakhir tanpa menggandakan pesan user), plus timestamp relatif.
- **Rate limit:** 10 per menit dan 100 per hari per IP, plus plafon global per hari.

### Fitur 6: Digest Harian (halaman News)

- **Alur:** `pg_cron` sekali sehari memanggil endpoint app. Server mengambil (a) top 10 repo trending (velocity 7 hari) dan (b) aktivitas repo Watchlist 24 jam terakhir (commit baru dan rilis baru) langsung dari GitHub. AI menulis judul buletin, blurb "tentang repo apa" untuk tiap repo trending, dan headline serta ringkasan untuk tiap repo watchlist, semuanya dalam **satu** panggilan. Hasil disimpan lalu tampil di `/digest`.
- **Dua bagian:** "Trending now" (berita umum) dan "Watchlist updates" (apa yang baru di repo yang dipantau). Di desktop keduanya berdampingan kiri-kanan, di mobile bertumpuk. Masing-masing bisa di-hide atau di-show.
- **Hari sepi:** kalau tidak ada aktivitas watchlist, bagian itu kosong dan buletin tetap dibuat; berita umum tetap tampil.
- **Retry:** cron memanggil dua kali sehari (01:00 dan 01:30 UTC). Panggilan kedua hanya bekerja kalau digest hari itu gagal atau macet.
- **Riwayat:** entri terbaru tampil terbuka, entri lama terlipat, dengan "Load more". Isi digest tidak berubah walau data berubah esok hari (disimpan sebagai salinan).
