# SCHEMA.md - GitHub Radar
> Database: Supabase (PostgreSQL). Semua tabel di bawah `public` schema kecuali disebutkan lain.
> Dua jalur akses saja: **browser** (anon key, hanya baca lewat RLS) dan **server/Edge Function** (service role, semua tulis).
> Extension yang dipakai: `pg_cron` (jadwal), `pg_net` (cron memanggil Edge Function/endpoint app), Supabase Vault (menyimpan `CRON_SECRET` dan `APP_URL` untuk dipakai cron).

## 1. repos
Master data tiap repo GitHub yang dipantau. Satu baris = satu repo. Dipakai Dashboard, Watchlist, dan Detail Repo.
- `id` (uuid, PK, default `gen_random_uuid()`)
- `github_id` (bigint, not null, **unique**) — ID repo dari GitHub. Stabil walau repo diganti nama/pindah owner, jadi dipakai sebagai kunci anti-duplikat.
- `owner` (text, not null) — contoh: "vercel".
- `name` (text, not null) — contoh: "ai-sdk".
- `description` (text, nullable)
- `html_url` (text, not null) — dipakai tombol "Open on GitHub".
- `primary_language` (text, nullable) — contoh: "TypeScript".
- `topics` (text[], not null, default `'{}'`) — topik repo dari GitHub (chip topik).
- `license` (text, nullable) — kode lisensi (contoh: "MIT"), dipakai kartu Repo info.
- `default_branch` (text, nullable) — contoh: "main".
- `stars_count` (integer, not null, default 0) — nilai terbaru, di-update tiap siklus. Disalin dari snapshot terakhir supaya daftar repo tidak perlu join ke snapshot (hindari N+1).
- `forks_count` (integer, not null, default 0) — sama, nilai terbaru.
- `open_issues_count` (integer, not null, default 0) — nilai terbaru. **Catatan:** angka ini dari GitHub dan sudah termasuk PR yang masih terbuka.
- `pushed_at` (timestamptz, nullable) — waktu push terakhir ("Last pushed 3 hours ago").
- `github_created_at` (timestamptz, not null) — tanggal repo dibuat di GitHub ("Created 8 days ago"), juga dipakai pencarian repo baru.
- `status` (text, not null, default `'active'`) — `active` (dilacak) atau `evicted` (dikeluarkan karena batas 100 repo aktif). Baris **tidak dihapus** saat dikeluarkan: riwayat tetap ada dan repo tidak ditambahkan ulang oleh penemuan repo baru ("belum ada di database" tetap berlaku).
- `is_watched` (boolean, not null, default false) — `true` = masuk Watchlist. Repo yang di-unsave jadi `false` tetapi `status` tetap `active` (jadi repo biasa, tetap dilacak).
- `watched_at` (timestamptz, nullable) — diisi saat masuk Watchlist, `NULL` kalau tidak.
- `discovery_source` (text, not null) — `backfill` (30 repo awal saat setup), `discovery` (ditemukan otomatis tiap siklus), atau `manual` (ditambahkan user lewat Watchlist/Detail Repo).
- `discovered_at` (timestamptz, not null, default now()) — kapan repo pertama masuk daftar pantau. Dipakai angka "14 new in 7 days" di kartu Repos tracked.
- `backfill_status` (text, not null, default `'pending'`) — `pending`, `running`, `done`, `failed`.
- `backfill_attempts` (smallint, not null, default 0) — jumlah percobaan backfill. Maksimal 3, setelah itu status jadi `failed` (tidak retry tanpa batas).
- `backfill_started_at` (timestamptz, nullable) — dipakai mendeteksi backfill yang macet (lihat Bagian 10).
- `backfilled_at` (timestamptz, nullable) — diisi saat `backfill_status` jadi `done`.
- `last_synced_at` (timestamptz, nullable) — kapan snapshot repo ini terakhir berhasil diambil. Dipakai (a) teks "Last updated 2 hours ago" di Dashboard (`max` dari kolom ini untuk repo aktif), (b) urutan proses siklus berikutnya: yang paling lama belum di-sync diproses duluan (`NULL` paling awal), supaya kalau rate limit habis di tengah siklus, repo sisanya tidak kelaparan.
- `created_at` (timestamptz, default now())
- `updated_at` (timestamptz, default now(), di-update lewat trigger)

**Constraint:**
- CHECK `status IN ('active','evicted')`, `discovery_source IN ('backfill','discovery','manual')`, `backfill_status IN ('pending','running','done','failed')`.
- CHECK `stars_count >= 0`, `forks_count >= 0`, `open_issues_count >= 0`.
- CHECK `NOT is_watched OR status = 'active'` (repo Watchlist tidak pernah berstatus `evicted`).
- CHECK `is_watched = (watched_at IS NOT NULL)` (dua kolom ini selalu searah).
- CHECK `backfill_attempts BETWEEN 0 AND 3`.

**Index:**
- Unique index di `github_id`.
- Unique index di `(lower(owner), lower(name))` — GitHub tidak membedakan huruf besar/kecil, dan dipakai untuk URL `/repo/[owner]/[name]`.
- Index di `status`.
- Partial index di `is_watched` `WHERE is_watched` (hitung "7 of 50 repos" dan daftar Watchlist).
- Partial index di `backfill_status` `WHERE backfill_status IN ('pending','running')` (antrean backfill).
- Index di `discovered_at`.
- Index di `last_synced_at` (`NULLS FIRST`).

**Akses:** publik boleh baca (data repo publik GitHub). Tulis hanya lewat server/Edge Function.
**Idempotency:** insert repo (dari penemuan otomatis, backfill, atau tambah manual) wajib `INSERT ... ON CONFLICT (github_id)` supaya double-click atau dua request bersamaan tidak membuat duplikat dan tidak crash. Batas Watchlist 50 repo dan batas 100 repo aktif ditegakkan lewat function (Bagian 9), bukan cek di aplikasi, supaya aman dari race condition.
**Repo ganti nama:** saat sinkronisasi, `owner` dan `name` di-update berdasarkan `github_id`. Kalau update bentrok dengan unique index `(lower(owner), lower(name))` (nama lama dipakai repo lain), siklus mencatat `console.error` + `job_runs.details`, melewati update nama repo itu, dan lanjut ke repo berikutnya. Tidak crash dan tidak membatalkan siklus.

## 2. repo_snapshots
Riwayat angka tiap repo dari waktu ke waktu (time-series). Ini sumber semua grafik dan perhitungan velocity. Nilai unik produk ini dibanding GitHub Trending.
- `repo_id` (uuid, FK ke `repos.id`, on delete cascade, not null)
- `captured_at` (timestamptz, not null) — waktu snapshot. Untuk snapshot dari cron, dibulatkan ke awal slot 6 jam (00:00, 06:00, 12:00, 18:00 UTC), jadi retry di siklus yang sama menimpa baris yang sama. Untuk snapshot backfill, diisi akhir hari (UTC) dari tanggal bintang. **Hari ini tidak dibuatkan snapshot backfill** (akhir hari belum terjadi), hari berjalan diisi oleh snapshot cron.
- `stars_count` (integer, not null)
- `forks_count` (integer, nullable) — `NULL` pada snapshot backfill (riwayat fork tidak tersedia dari endpoint stargazers).
- `open_issues_count` (integer, nullable) — `NULL` pada snapshot backfill, alasan sama.
- `source` (text, not null, default `'cron'`) — `cron` atau `backfill`.
- **Primary key gabungan:** (`repo_id`, `captured_at`)

**Constraint:** CHECK `source IN ('cron','backfill')`, `stars_count >= 0`.
**Index:** PK gabungan sudah menjadi index (`repo_id`, `captured_at`) untuk query "snapshot terakhir" dan "snapshot N hari lalu" per repo; index tambahan di `captured_at` (descending).
**Akses:** publik boleh baca. Tulis hanya lewat server/Edge Function.
**Idempotency:** insert wajib `INSERT ... ON CONFLICT (repo_id, captured_at) DO UPDATE` (siklus yang di-retry atau backfill yang diulang tidak menggandakan baris).
**Biaya backfill:** endpoint stargazers dipaginasi 100 bintang per halaman, jadi repo dengan puluhan ribu bintang memakan ratusan request. Batas halaman maksimal per repo per percobaan disimpan di `constants/`. Repo yang melewati batas diambil dengan sampling halaman (bukan semua halaman), lalu hasilnya diinterpolasi ke snapshot harian. Alasannya supaya satu repo besar tidak menghabiskan kuota GitHub untuk repo lain.
**Retensi:** di V1 tidak ada penghapusan. Perkiraan volume: 150 repo x 4 snapshot/hari = sekitar 600 baris/hari, masih sangat kecil.

## 3. repo_metrics
Hasil hitung kecepatan naik (velocity) per repo per jendela waktu, dihitung ulang di akhir tiap siklus. Disimpan (bukan dihitung tiap halaman dibuka) supaya Dashboard cepat dan konsisten untuk filter Today / 7 hari / 30 hari.
- `repo_id` (uuid, FK ke `repos.id`, on delete cascade, not null)
- `window_days` (smallint, not null) — jendela waktu: `1` (Today), `7`, atau `30`.
- `stars_gained` (integer, not null) — bintang yang didapat dalam jendela itu. Bisa negatif.
- `velocity` (numeric(10,2), not null) — rata-rata bintang per hari dalam jendela itu. Bisa negatif (contoh: `-12/day`).
- `prev_stars_gained` (integer, nullable) — bintang pada periode sebelumnya yang sama panjang, untuk persentase perubahan di kartu Stars gained. `NULL` kalau riwayat belum cukup.
- `is_partial` (boolean, not null, default false) — `true` kalau riwayat repo lebih pendek dari jendela (repo baru), jadi velocity dihitung dari data yang ada saja.
- `computed_at` (timestamptz, not null)
- **Primary key gabungan:** (`repo_id`, `window_days`)

**Constraint:** CHECK `window_days IN (1,7,30)`.
**Index:** PK gabungan; index di (`window_days`, `velocity` descending, `repo_id`) untuk daftar Rising now (urut velocity, `repo_id` sebagai pemecah seri supaya "Muat lebih banyak" 12 per klik tidak menggandakan atau melewatkan baris).
**Akses:** publik boleh baca. Tulis hanya lewat function `compute_repo_metrics()` (Bagian 9).
**Aturan baca:**
- Query selalu digabung dengan `repos.status = 'active'`.
- Daftar "repo naik" hanya mengambil `velocity > 0`.
- Nomor peringkat (#1, #2, "Rank #1 overall") **tidak disimpan**, dihitung dari urutan velocity pada jendela terpilih.
- Persentase perubahan di kartu Total stars gained dihitung dari repo yang `prev_stars_gained`-nya tidak `NULL` saja (supaya periode dibandingkan apple to apple). Kalau total periode sebelumnya `0` atau tidak ada, UI menampilkan "-" bukan angka persen.

**Aturan hapus:** baris metrics repo yang dikeluarkan (`evicted`) dihapus oleh function eviction.

## 4. repo_details
Data pendukung per repo yang tidak berubah tiap 6 jam: commit mingguan, rilis terbaru, potongan README. Relasi 1-ke-1 dengan `repos`. Diambil saat repo pertama masuk daftar pantau, lalu diperbarui paling cepat sekali per 24 jam.
- `repo_id` (uuid, PK, FK ke `repos.id`, on delete cascade)
- `weekly_commits` (integer[], nullable) — jumlah commit 12 minggu terakhir, urut dari yang paling lama ke terbaru (12 elemen). `NULL` kalau GitHub belum menyiapkan statistiknya, dicoba lagi di pembaruan harian berikutnya.
- `latest_release_tag` (text, nullable) — contoh: "v5.0.3". `NULL` kalau repo belum punya rilis (kartu Latest release disembunyikan).
- `latest_release_published_at` (timestamptz, nullable)
- `latest_release_notes` (text, nullable) — isi catatan rilis, dipotong aplikasi sebelum disimpan.
- `readme_excerpt` (text, nullable) — potongan awal README (markdown mentah).
- `details_refreshed_at` (timestamptz, nullable) — dipakai siklus untuk memilih repo yang datanya lebih tua dari 24 jam.

**Constraint:** CHECK `weekly_commits IS NULL OR cardinality(weekly_commits) = 12`, `char_length(readme_excerpt) <= 4000`, `char_length(latest_release_notes) <= 4000`.
**Index:** PK sudah cukup (join satu arah dari Detail Repo); index di `details_refreshed_at` untuk memilih repo yang perlu di-refresh.
**Akses:** publik boleh baca. Tulis hanya lewat server/Edge Function.
**Keamanan konten:** README, catatan rilis, deskripsi, dan topik berasal dari pihak ketiga, jadi dianggap tidak aman. Wajib di-render sebagai teks/markdown yang disanitasi, tidak pernah sebagai HTML mentah (RULES.md Bagian 17).

## 5. ai_summaries
Cache ringkasan AI per repo di halaman Detail Repo ("AI summary", label "CACHED - OCT 1"), supaya AI tidak dipanggil tiap halaman dibuka.
- `repo_id` (uuid, PK, FK ke `repos.id`, on delete cascade) — satu repo satu ringkasan aktif.
- `content` (text, nullable) — isi ringkasan (poin-poin markdown). `NULL` selama belum pernah berhasil dibuat.
- `model` (text, nullable) — nama model yang dipakai (dari `AI_MODEL`), untuk jejak audit.
- `status` (text, not null, default `'generating'`) — `generating`, `ready`, `failed`.
- `generating_started_at` (timestamptz, nullable) — kunci waktu untuk mendeteksi proses yang macet.
- `generated_at` (timestamptz, nullable) — dipakai label "CACHED - [tanggal]".
- `based_on_snapshot_at` (timestamptz, nullable) — snapshot terakhir yang dipakai sebagai bahan, untuk menentukan ringkasan sudah basi atau belum.

**Constraint:** CHECK `status IN ('generating','ready','failed')`.
**Akses:** publik boleh baca. Tulis hanya lewat server.
**Idempotency / biaya AI:**
- Pembuatan ulang wajib pakai CAS: `UPDATE ai_summaries SET status = 'generating', generating_started_at = now() WHERE repo_id = ? AND (status <> 'generating' OR generating_started_at < now() - interval '2 minutes')`.
- Kalau 0 baris terpengaruh, request lain sedang membuat, jadi request ini berhenti dan menampilkan cache lama (tidak memanggil AI kedua kali).
- Untuk repo yang belum punya baris: `INSERT ... ON CONFLICT (repo_id) DO NOTHING`, yang menang lanjut memanggil AI.
- Batas kedaluwarsa cache (contoh awal: 24 jam) ada di `constants/`, bukan di database.

**Catatan:** tidak ada kolom pesan error di tabel ini (tabel ini bisa dibaca publik). Error AI dicatat lewat `console.error` di server (RULES.md Bagian 10). Kalau AI gagal tapi sudah ada `content` lama, `content` lama dipertahankan dan tetap ditampilkan.

## 6. digests
Ringkasan harian dari AI untuk halaman `/digest`. Satu baris = satu hari. Data pendukung disimpan sebagai salinan (snapshot) saat digest dibuat, supaya riwayat tidak berubah walau velocity berubah di hari berikutnya.
- `id` (uuid, PK, default `gen_random_uuid()`)
- `digest_date` (date, not null, **unique**) — tanggal digest (UTC). Kunci anti-duplikat: cron yang terpanggil dua kali tidak membuat dua digest.
- `title` (text, nullable) — judul ringkasan (contoh: "AI tooling keeps climbing, with ai-sdk leading the week").
- `content` (text, nullable) — isi ringkasan bahasa manusia. `NULL` selama belum berhasil dibuat.
- `tags` (text[], not null, default `'{}'`) — chip topik di daftar riwayat.
- `top_risers` (jsonb, not null, default `'[]'`) — salinan daftar top repo naik saat itu. Bentuk tiap elemen: `{ "repo_id": uuid, "full_name": "vercel/ai-sdk", "velocity": 480.0, "stars_gained": 3360 }`.
- `watchlist_changes` (jsonb, not null, default `'[]'`) — salinan perubahan repo Watchlist. Bentuk tiap elemen: `{ "repo_id": uuid, "full_name": "...", "stars_delta": 120, "forks_delta": 4 }`.
- `model` (text, nullable)
- `status` (text, not null, default `'generating'`) — `generating`, `ready`, `failed`.
- `generating_started_at` (timestamptz, nullable)
- `generated_at` (timestamptz, nullable)
- `created_at` (timestamptz, default now())

**Constraint:** CHECK `status IN ('generating','ready','failed')`, `jsonb_typeof(top_risers) = 'array'`, `jsonb_typeof(watchlist_changes) = 'array'`.
**Index:** unique index di `digest_date` (juga dipakai urutan terbaru ke terlama dan pagination berbasis tanggal).
**Akses:** publik boleh baca. Tulis hanya lewat endpoint cron di server.
**Idempotency:**
1. `INSERT (digest_date, status = 'generating') ON CONFLICT (digest_date) DO NOTHING`.
2. Panggil AI sekali.
3. `UPDATE ... SET status = 'ready' ... WHERE id = ? AND status = 'generating'` (CAS).
4. Baris berstatus `failed`, atau `generating` lebih dari 10 menit, boleh diambil alih oleh panggilan cron berikutnya (cron memanggil dua kali sehari untuk retry, lihat Bagian 10, jadi retry-nya terbatas).

**Catatan:**
- Seperti `ai_summaries`, tidak ada kolom pesan error di sini. Error dicatat di `job_runs`.
- Halaman `/digest` hanya menampilkan baris berstatus `ready`. Baris `generating` atau `failed` tidak tampil ke pengunjung.

## 7. job_runs
Log tiap eksekusi job otomatis (pengumpulan data, backfill, digest). Dipakai untuk audit, mencegah job jalan tumpang tindih, dan melacak kegagalan tanpa harus membuka log server (RULES.md Bagian 10).
- `id` (uuid, PK, default `gen_random_uuid()`)
- `job_type` (text, not null) — `collect`, `backfill`, atau `digest`.
- `status` (text, not null, default `'running'`) — `running`, `success`, `partial` (selesai tapi sebagian repo gagal atau terpotong rate limit), `failed`.
- `started_at` (timestamptz, not null, default now())
- `finished_at` (timestamptz, nullable)
- `repos_processed` (integer, not null, default 0)
- `error_message` (text, nullable) — pesan error asli untuk log internal. Tidak pernah ditampilkan ke user.
- `details` (jsonb, not null, default `'{}'`) — rincian: jumlah repo ditemukan/dikeluarkan, sisa kuota GitHub, repo yang gagal. **Dilarang** berisi token atau secret.

**Constraint:** CHECK `job_type IN ('collect','backfill','digest')`, `status IN ('running','success','partial','failed')`.
**Index:**
- **Partial unique index** di `job_type` `WHERE status = 'running'` (hanya satu job tipe yang sama boleh berjalan, insert kedua gagal dengan duplicate key dan job itu berhenti dengan tenang).
- Index di (`job_type`, `started_at` descending).

**Akses:** hanya server (tanpa policy).
**Catatan:**
- Job `backfill` yang tidak punya antrean (tidak ada repo `pending`) tidak perlu mencatat baris, supaya tabel tidak penuh noise tiap beberapa menit.
- Job `collect` dan `backfill` memakai kuota GitHub yang sama. Sisa kuota dari header respons dicatat di `details` supaya terlihat kalau keduanya saling berebut.

## 8. rate_limits
Penghitung rate limit per IP berbasis database. Jawaban untuk aturan PRD (chat wajib dibatasi per IP) dan RULES.md Bagian 12 (endpoint tanpa login yang memanggil API berbayar atau API pihak ketiga). Dipakai sebagai guard sederhana sampai ada infrastruktur khusus.
- `key` (text, not null) — gabungan nama endpoint, jenis jendela, dan hash IP, contoh: `chat:min:9f2c...` atau `chat:day:9f2c...`. IP diubah jadi hash (HMAC dengan secret server) sebelum dipakai. **IP mentah tidak pernah disimpan atau di-log.**
- `window_start` (timestamptz, not null) — awal jendela waktu (dibulatkan sesuai panjang jendela).
- `count` (integer, not null, default 0)
- **Primary key gabungan:** (`key`, `window_start`)

**Akses:** hanya server (tanpa policy). Semua pengecekan lewat function `check_rate_limit()` (Bagian 9) yang atomik.
**Pembersihan:** baris yang lebih tua dari 1 hari dihapus cron harian (Bagian 10).
**Batas dua jendela:** endpoint yang punya batas per menit dan per hari (Chat AI) memanggil `check_rate_limit()` dua kali dengan key berbeda. Request ditolak kalau salah satunya `false`.
**Batas global:** endpoint yang menghabiskan token AI (Chat AI) juga punya key tanpa IP (contoh: `chat:global:day`) sebagai plafon total per hari untuk semua pengunjung, supaya biaya tetap terbatas walau penyerang berganti-ganti IP. Pakai tabel dan function yang sama.

**Endpoint yang wajib dibatasi** (angka contoh awal, nilai asli di `constants/`):
- **Chat AI** — alasan: menghabiskan token AI. Batas awal: 10 per menit dan 100 per hari per IP.
- **Pencarian repo (dialog Add repo)** — alasan: memanggil GitHub Search API (kuotanya kecil). Batas awal: 20 per menit per IP.
- **Tambah/hapus Watchlist** — alasan: memanggil GitHub dan menulis database. Batas awal: 30 per menit per IP.
- **Buka repo yang belum ada di database (URL langsung)** — alasan: memicu fetch GitHub dan insert. Batas awal: 10 per menit per IP.
- **Buat ulang ringkasan AI** — alasan: menghabiskan token AI. Batas awal: 10 per jam per IP.

## 9. Function & Trigger
Semua function di bawah memakai `SET search_path = ''` dan nama objek lengkap dengan schema. Function bertanda **server-only** berstatus `SECURITY DEFINER`, dan hak `EXECUTE`-nya di-**revoke** dari `PUBLIC`, `anon`, dan `authenticated`, lalu diberikan hanya ke `service_role` (default Postgres memberi EXECUTE ke semua orang, jadi revoke wajib eksplisit).

- **`set_updated_at()`** (trigger, reusable) — mengisi `updated_at` tiap baris `repos` berubah.
- **`set_watchlist(p_repo_id uuid, p_watched boolean)`** — *server-only.* Satu-satunya jalan menyalakan/mematikan Watchlist.
  - Memakai advisory lock transaksi supaya hitungan 50 repo aman dari dua request bersamaan.
  - Menyalakan: kalau sudah `is_watched`, langsung sukses (idempotent). Kalau jumlah watched sudah 50, menolak dengan hasil `limit_reached`. Selain itu set `is_watched = true`, `watched_at = now()`, `status = 'active'` (repo yang pernah dikeluarkan otomatis aktif lagi).
  - Mematikan: set `is_watched = false`, `watched_at = NULL`. `status` tidak diubah, jadi repo kembali jadi repo biasa yang tetap dilacak (dan ikut batas 100 repo aktif).
  - Hasil: `ok`, `limit_reached`, atau `not_found`.
- **`claim_backfill_batch(p_limit int)`** — *server-only.* Mengambil repo `pending` sebanyak `p_limit` dengan `FOR UPDATE SKIP LOCKED`, set `backfill_status = 'running'`, `backfill_started_at = now()`, `backfill_attempts + 1`, lalu mengembalikan daftarnya. Aman dipanggil bersamaan tanpa dua worker mengambil repo yang sama. Repo dengan `backfill_attempts >= 3` tidak diambil lagi dan ditandai `failed`.
- **`compute_repo_metrics()`** — *server-only.* Menghitung ulang `repo_metrics` untuk semua repo aktif, jendela 1, 7, dan 30 hari. Idempotent (upsert). Aturan hitung:
  - Titik acuan jendela adalah `captured_at` snapshot terbaru repo itu, bukan `now()`, supaya hasilnya konsisten walau siklus terlambat.
  - `stars_gained` = bintang snapshot terbaru dikurangi bintang snapshot terakhir sebelum awal jendela. Kalau riwayat belum sampai awal jendela, pakai snapshot paling awal dan set `is_partial = true`.
  - `velocity` = `stars_gained` dibagi lama (hari) yang benar-benar tercakup (lama 0 hari menghasilkan velocity 0, bukan error pembagian).
  - `prev_stars_gained` dihitung dengan cara sama untuk periode sebelumnya, `NULL` kalau riwayat belum cukup.
- **`evict_excess_repos(p_max_active int default 100)`** — *server-only.* Memilih repo `active` yang tidak `is_watched`. Kalau jumlahnya melebihi `p_max_active`, repo dengan velocity 14 hari terendah (seri: `discovered_at` paling lama) diubah jadi `evicted` dan baris `repo_metrics`-nya dihapus. Repo yang masih `pending`/`running` backfill tidak ikut dipertimbangkan (jangan dikeluarkan sebelum riwayatnya terisi). Velocity 14 hari dihitung langsung dari `repo_snapshots` di dalam function ini (tabel `repo_metrics` hanya punya jendela 1, 7, 30). Mengembalikan daftar repo yang dikeluarkan.
- **`check_rate_limit(p_key text, p_limit int, p_window_seconds int)`** — *server-only.* Atomik: `INSERT ... ON CONFLICT (key, window_start) DO UPDATE SET count = count + 1 RETURNING count`, hasilnya `true` kalau `count <= p_limit`. Tidak ada jalur "baca dulu baru tulis".
- **`get_star_series(p_repo_ids uuid[], p_days int)`** — *boleh dipanggil publik* (`SECURITY INVOKER`, jadi RLS berlaku). Mengembalikan titik harian bintang (snapshot terakhir tiap hari) untuk banyak repo sekaligus, dipakai sparkline kartu repo dan grafik Star velocity. Satu panggilan untuk semua repo di layar, bukan satu panggilan per kartu (hindari N+1). Karena bisa dipanggil publik, function ini membatasi input: `p_days` maksimal 90 dan jumlah `p_repo_ids` maksimal 50, di atas itu ditolak.

## 10. Cron Job (`pg_cron`, zona waktu UTC)
Semua job memanggil Edge Function atau endpoint app lewat `pg_net`. Header `Authorization` berisi `CRON_SECRET` yang **dibaca dari Vault saat job jalan**, tidak ditulis langsung di perintah cron (isi tabel `cron.job` bisa dilihat, jadi secret tidak boleh ada di sana). Endpoint/Edge Function wajib menolak request tanpa secret yang cocok.

Daftar job:
- **`collect-repos`** — jadwal `0 */6 * * *`. Satu siklus pengumpulan (lihat langkah di bawah).
- **`backfill-batch`** — jadwal `*/5 * * * *`. Memproses beberapa repo antrean backfill per panggilan (`claim_backfill_batch`). Kalau antrean kosong, langsung selesai.
- **`daily-digest`** — jadwal `0 1 * * *` dan `30 1 * * *`. Memanggil endpoint digest di app. Panggilan kedua (30 menit kemudian) hanya bekerja kalau digest hari itu `failed` atau macet, jadi retry terbatas satu kali. 01:00 UTC = 08:00 WIB.
- **`reap-stale-jobs`** — jadwal `*/10 * * * *`. SQL murni: `job_runs` yang `running` lebih dari 15 menit jadi `failed`; repo `backfill_status = 'running'` yang `backfill_started_at` lebih dari 15 menit dikembalikan ke `pending` (atau `failed` kalau percobaannya sudah 3).
- **`cleanup-housekeeping`** — jadwal `0 3 * * *`. SQL murni: hapus `rate_limits` yang lebih tua dari 1 hari, dan hapus `cron.job_run_details` yang lebih tua dari 7 hari (tabel bawaan `pg_cron` ini tumbuh terus, `backfill-batch` saja menambah 288 baris per hari).

**Langkah satu siklus `collect-repos`:**
1. Daftarkan diri di `job_runs` (gagal duplicate key berarti siklus sebelumnya masih jalan, berhenti). Kalau `GITHUB_TOKEN` kosong, langsung `failed` dengan pesan jelas (tanpa token, limit hanya 60 request per jam).
2. Kalau tabel `repos` masih kosong (setup pertama): cari repo 30 hari terakhir (urut bintang, 30 teratas), simpan dengan `discovery_source = 'backfill'`, lalu berhenti. Backfill ditangani job `backfill-batch`.
3. Ambil snapshot repo aktif, yang `last_synced_at`-nya paling lama duluan. Berhenti rapi (status `partial`) kalau sisa kuota GitHub menipis.
4. Perbarui `repo_details` untuk repo yang `details_refreshed_at`-nya lebih tua dari 24 jam atau masih kosong.
5. Penemuan repo baru: repo 7 hari terakhir (urut bintang), minimal 50 bintang, belum ada di database, maksimal 10 per siklus, `discovery_source = 'discovery'`.
6. Jalankan `evict_excess_repos(100)`, lalu `compute_repo_metrics()`.
7. Tutup baris `job_runs` (`success`/`partial`/`failed`).

**Status setup untuk kartu progres Dashboard** (dihitung dari `repos` dengan `discovery_source = 'backfill'`, tanpa tabel tambahan):
- Belum ada repo sama sekali: "Finding new repos".
- Masih ada yang `pending`/`running`: "Fetching star history" (angka "12 of 30 repos" = jumlah `done` dibanding total).
- Tidak ada yang `pending`/`running` tetapi `repo_metrics` masih kosong: "Calculating velocity".
- Selain itu: setup selesai, tampil Dashboard normal. Repo yang akhirnya `failed` tidak menahan Dashboard selamanya.
- Repo dari penemuan otomatis atau tambah manual **tidak** memunculkan kartu progres.


## 11. Row Level Security (RLS)
**RLS wajib di-ENABLE di semua tabel** (`ALTER TABLE ... ENABLE ROW LEVEL SECURITY`), termasuk tabel server-only. Tabel tanpa policy dengan RLS aktif = deny total untuk `anon` dan `authenticated`. Tabel server-only (`job_runs`, `rate_limits`) juga di-`REVOKE ALL` dari `anon` dan `authenticated` sebagai lapis kedua.

Dua kelas akses saja, karena V1 tanpa login:
- **Publik (anon)** — browser pemilik app memakai `NEXT_PUBLIC_SUPABASE_ANON_KEY`. Hanya boleh **baca** tabel yang memang untuk ditampilkan. Karena anon key terlihat di browser, siapa pun yang memegang URL project bisa memanggil API database langsung, jadi hanya data non-sensitif yang boleh punya policy baca.
- **Server-only (service role)** — dipakai Server Action, Route Handler, dan Edge Function untuk semua tulis (simpan snapshot, ubah watchlist, panggil function server-only, catat log). Service role selalu *bypass* RLS dan `SUPABASE_SERVICE_ROLE_KEY` tidak pernah sampai ke browser. Tabel/function yang statusnya "server-only" **sengaja tidak diberi policy** (atau di-revoke EXECUTE-nya) supaya default deny berlaku untuk `anon` dan `authenticated`.
- Policy baca ditulis untuk `anon` dan `authenticated` sekaligus, supaya akun hasil signup tidak punya hak berbeda dari pengunjung biasa.

> **Keamanan (wajib):** matikan signup publik di Supabase Auth (Dashboard → Authentication → Sign In / Providers → "Allow new users to sign up" = OFF). Aplikasi tidak memakai Auth, jadi tidak ada alasan membiarkannya terbuka.

> **Perlu disadari (V1 tanpa login):** semua endpoint tulis (cari repo, tambah/hapus Watchlist, buka repo lewat URL, chat, buat ulang ringkasan) bisa dipanggil siapa pun yang tahu alamat app, dan tiap panggilan memakai kuota GitHub atau token AI. Mitigasi minimal di dokumen ini: `rate_limits` di semua endpoint tersebut (Bagian 8). Disarankan juga proteksi di level hosting (misalnya password protection atau Basic Auth) selama belum ada login.

Aturan per tabel:

### `repos`, `repo_snapshots`, `repo_metrics`, `repo_details`
- **Read:** publik (anon dan authenticated), semua baris.
- **Insert/Update/Delete:** tidak ada policy, hanya server (service role).

### `ai_summaries`, `digests`
- **Read:** publik, semua baris (isinya analisis atas data publik, dan memang tampil di halaman publik). Filter `status = 'ready'` dilakukan di query aplikasi.
- **Insert/Update/Delete:** tidak ada policy, hanya server.

### `job_runs`
- **Server-only, tanpa policy sama sekali.** Berisi `error_message` dan `details` internal yang tidak boleh terbaca dari browser.

### `rate_limits`
- **Server-only, tanpa policy sama sekali.** Semua akses lewat `check_rate_limit()`.

### Function
- **Server-only** (EXECUTE hanya `service_role`): `set_watchlist`, `claim_backfill_batch`, `compute_repo_metrics`, `evict_excess_repos`, `check_rate_limit`.
- **Boleh publik:** `get_star_series` (read-only, `SECURITY INVOKER`, tunduk pada RLS tabel snapshot, input dibatasi).
- **Trigger:** `set_updated_at` (tidak dipanggil sebagai RPC).

### Ringkasan pola
- **`repos`** — publik baca: ya. Tulis: tanpa policy (deny client). Jalur: server saja (+ `set_watchlist`, `evict_excess_repos`).
- **`repo_snapshots`** — publik baca: ya. Tulis: tanpa policy. Jalur: server saja.
- **`repo_metrics`** — publik baca: ya. Tulis: tanpa policy. Jalur: server saja (`compute_repo_metrics`).
- **`repo_details`** — publik baca: ya. Tulis: tanpa policy. Jalur: server saja.
- **`ai_summaries`** — publik baca: ya. Tulis: tanpa policy. Jalur: server saja, wajib CAS.
- **`digests`** — publik baca: ya. Tulis: tanpa policy. Jalur: endpoint cron di server, wajib CAS.
- **`job_runs`** — publik baca: tidak. Tulis: tanpa policy (deny total). Jalur: server saja.
- **`rate_limits`** — publik baca: tidak. Tulis: tanpa policy (deny total). Jalur: server saja (`check_rate_limit`).
- **`set_watchlist`, `claim_backfill_batch`, `compute_repo_metrics`, `evict_excess_repos`, `check_rate_limit`** — publik: tidak. EXECUTE hanya `service_role`. Jalur: server saja.
- **`get_star_series`** — publik: ya. Tidak menulis, read-only.

## Catatan Umum
- Semua `status`/jenis di tabel (`repos.status`, `backfill_status`, `discovery_source`, `job_runs`, `ai_summaries`, `digests`) disimpan sebagai `text` dengan CHECK constraint (bukan Postgres `enum`), supaya gampang ditambah nilai baru tanpa `ALTER TYPE`.
- Batas bisnis (Watchlist 50, repo aktif 100, penemuan maksimal 10 per siklus dengan minimal 50 bintang, batas percobaan backfill 3, batas halaman backfill, batas rate limit, umur cache AI) disimpan di `constants/`, bukan di database. Yang butuh aman dari race condition (50 Watchlist, 100 repo aktif) ditegakkan lewat function di Bagian 9.
- Semua tulis multi-langkah mengikuti urutan RULES.md Bagian 9: paling reversible dulu. Contoh: simpan repo (`ON CONFLICT`), baru jadwalkan backfill, baru nyalakan Watchlist. Kegagalan di tengah tidak meninggalkan data setengah jadi, karena tiap langkah idempotent dan bisa diulang.
- **Tidak ada seed data demo.** PRD melarang data demo di Dashboard. Dashboard kosong selalu menampilkan kartu progres setup.
- Pagination: Dashboard memakai pagination 12 per klik (urut `velocity` lalu `repo_id`), `/digest` memakai pagination berbasis `digest_date`. Watchlist maksimal 50 baris jadi boleh diambil sekaligus.
- Select kolom spesifik, bukan `SELECT *` (RULES.md Bagian 4). Terutama `repo_details.readme_excerpt` dan `ai_summaries.content`, hanya diambil di halaman Detail Repo.
- Semua FK sudah punya index karena kolom FK adalah kolom pertama di PK atau unique index masing-masing tabel.
- Data hasil hitung (`repo_metrics`) dan data historis (`repo_snapshots`, `digests`) tidak pernah diedit manual. Kalau salah, hitung ulang dengan function atau buat ulang digest.
- Error di semua job dan function server dicatat lewat `console.error` dan `job_runs` (RULES.md Bagian 10). Secret dan token tidak pernah ikut ke log.


### Uji minimal (RULES.md Bagian 13)
- `set_watchlist`: dua request bersamaan di repo ke-50 (hanya satu berhasil), menyalakan dua kali (idempotent), mematikan lalu menyalakan lagi, repo `evicted` yang di-watch jadi aktif lagi.
- `compute_repo_metrics`: repo baru dengan riwayat pendek (`is_partial`), velocity negatif, periode sebelumnya kosong, snapshot dengan jarak waktu 0.
- `evict_excess_repos`: repo Watchlist tidak pernah ikut keluar, repo yang masih backfill tidak ikut keluar.
- `check_rate_limit`: batas tepat, jendela berganti, dua panggilan bersamaan.
- `get_star_series`: `p_days` di atas 90 ditolak, array repo lebih dari 50 ditolak.
- RLS: `anon` bisa baca tabel publik tapi gagal tulis ke semua tabel, `anon` gagal baca `job_runs` dan `rate_limits`, `anon` gagal memanggil function server-only.
- Constraint `repos`: `is_watched = true` dengan `status = 'evicted'` ditolak, `is_watched` dan `watched_at` tidak bisa tidak searah.
- Idempotency: snapshot yang dikirim dua kali di slot yang sama, dua panggilan digest di hari yang sama, dua panggilan pembuatan ringkasan AI untuk repo yang sama.