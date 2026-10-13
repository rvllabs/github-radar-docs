# SCHEMA.md — GitHub Radar

Database: Supabase (PostgreSQL), semua tabel di schema `public`. Langsung bikin tabel dengan kolom seperti di bawah (tidak perlu bikin versi lama lalu `ALTER`).

Dua jalur akses saja:
- **Browser:** anon key, hanya baca lewat RLS.
- **Server dan Edge Function:** service role, semua tulis.

Extension: `pg_cron` (jadwal) dan `pg_net` (cron memanggil Edge Function dan endpoint app). **Tanpa Vault.** `CRON_SECRET` dan `APP_URL` ditulis langsung di perintah cron (lihat Bagian 10).

## 1. repos

Master data tiap repo GitHub yang dipantau. Satu baris satu repo.

- `id` (uuid, PK, default `gen_random_uuid()`)
- `github_id` (bigint, not null, **unique**): ID repo dari GitHub. Stabil walau repo ganti nama atau pindah owner, jadi dipakai sebagai kunci anti-duplikat.
- `owner` (text, not null), contoh "vercel".
- `name` (text, not null), contoh "ai-sdk".
- `description` (text, nullable)
- `html_url` (text, not null): dipakai tombol "Open on GitHub".
- `primary_language` (text, nullable)
- `topics` (text[], not null, default `'{}'`)
- `license` (text, nullable): kode lisensi, contoh "MIT".
- `default_branch` (text, nullable)
- `stars_count`, `forks_count`, `open_issues_count` (integer, not null, default 0): nilai terbaru, disalin dari snapshot terakhir supaya daftar repo tidak perlu join ke snapshot. `open_issues_count` dari GitHub sudah termasuk PR yang masih terbuka.
- `pushed_at` (timestamptz, nullable)
- `github_created_at` (timestamptz, not null): tanggal repo dibuat di GitHub.
- `status` (text, not null, default `'active'`): `active` atau `evicted`. Baris tidak dihapus saat dikeluarkan, jadi riwayat tetap ada dan repo tidak ditambahkan ulang oleh penemuan repo baru.
- `is_watched` (boolean, not null, default false): true berarti masuk Watchlist.
- `watched_at` (timestamptz, nullable)
- `discovery_source` (text, not null): `backfill` (30 repo awal), `discovery` (ditemukan otomatis), atau `manual` (ditambahkan user).
- `discovered_at` (timestamptz, not null, default now()): dipakai angka "new in N days" di kartu Repos tracked.
- `backfill_status` (text, not null, default `'pending'`): `pending`, `running`, `done`, `failed`.
- `backfill_attempts` (smallint, not null, default 0): maksimal 3, lalu `failed`.
- `backfill_started_at` (timestamptz, nullable): dipakai mendeteksi backfill yang macet.
- `backfilled_at` (timestamptz, nullable)
- `last_synced_at` (timestamptz, nullable): kapan snapshot terakhir berhasil diambil. Dipakai untuk teks "Last updated" dan untuk urutan proses (yang paling lama belum di-sync duluan, NULL paling awal).
- `created_at`, `updated_at` (timestamptz, default now(); `updated_at` diisi trigger). Dua kolom ini nullable dengan default now().

**Constraint:**
- CHECK `status IN ('active','evicted')`, `discovery_source IN ('backfill','discovery','manual')`, `backfill_status IN ('pending','running','done','failed')`.
- CHECK `stars_count >= 0`, `forks_count >= 0`, `open_issues_count >= 0`.
- CHECK `NOT is_watched OR status = 'active'` (repo watchlist tidak pernah evicted).
- CHECK `is_watched = (watched_at IS NOT NULL)` (dua kolom selalu searah).
- CHECK `backfill_attempts BETWEEN 0 AND 3`.

**Index:**
- Unique di `github_id`.
- Unique di `(lower(owner), lower(name))`: GitHub tidak membedakan huruf besar kecil, dan dipakai untuk URL `/repo/[owner]/[name]`.
- Index di `status`.
- Partial index di `is_watched` `WHERE is_watched`.
- Partial index di `backfill_status` `WHERE backfill_status IN ('pending','running')`.
- Index di `discovered_at`.
- Index di `last_synced_at` (`NULLS FIRST`).

**Akses:** publik boleh baca. Tulis hanya lewat server atau Edge Function.

**Idempotency:** insert repo wajib `INSERT ... ON CONFLICT (github_id)` supaya double-click atau dua request bersamaan tidak membuat duplikat. Batas Watchlist 50 dan batas 100 repo aktif ditegakkan lewat function (Bagian 9), bukan cek di aplikasi.

**Repo ganti nama:** saat sinkronisasi, `owner` dan `name` di-update berdasarkan `github_id`. Kalau update bentrok dengan unique index `(lower(owner), lower(name))`, siklus mencatat `console.error` dan `job_runs.details`, melewati update nama repo itu, dan lanjut. Tidak crash dan tidak membatalkan siklus.

## 2. repo_snapshots

Riwayat angka tiap repo dari waktu ke waktu. Sumber semua grafik dan perhitungan velocity.

- `repo_id` (uuid, FK ke `repos.id`, on delete cascade, not null)
- `captured_at` (timestamptz, not null): untuk snapshot cron dibulatkan ke awal slot 6 jam UTC (00:00, 06:00, 12:00, 18:00), jadi retry di siklus yang sama menimpa baris yang sama. Untuk snapshot backfill diisi akhir hari UTC (23:59:59.999). Hari ini tidak dibuatkan snapshot backfill; hari berjalan diisi snapshot cron.
- `stars_count` (integer, not null)
- `forks_count` (integer, nullable): NULL pada snapshot backfill.
- `open_issues_count` (integer, nullable): NULL pada snapshot backfill.
- `source` (text, not null, default `'cron'`): `cron` atau `backfill`.
- **PK gabungan:** (`repo_id`, `captured_at`)

**Constraint:** CHECK `source IN ('cron','backfill')`, `stars_count >= 0`.
**Index:** PK gabungan, plus index di `captured_at` (descending).
**Akses:** publik baca, tulis hanya server.
**Idempotency:** insert wajib `ON CONFLICT (repo_id, captured_at) DO UPDATE`.

**Biaya backfill:** endpoint `/stargazers/history` dipaginasi 30 minggu per halaman, mundur dari tanggal pembuatan repo. Repo bertahun-tahun pun cukup beberapa halaman. **Tidak ada sampling dan tidak ada interpolasi.** Batas halaman maksimal per repo per percobaan (`BACKFILL_MAX_PAGES_PER_REPO`) tetap disimpan di `constants/` sebagai pengaman.

**Retensi:** Tanpa penghapusan. Perkiraan 150 repo kali 4 snapshot per hari, sekitar 600 baris per hari.

## 3. repo_metrics

Hasil hitung velocity per repo per jendela waktu, dihitung ulang di akhir tiap siklus.

- `repo_id` (uuid, FK ke `repos.id`, on delete cascade, not null)
- `window_days` (smallint, not null): `1`, `7`, atau `30`.
- `stars_gained` (integer, not null): bisa negatif.
- `velocity` (numeric(10,2), not null): rata-rata bintang per hari. Bisa negatif.
- `prev_stars_gained` (integer, nullable): bintang di periode sebelumnya yang sama panjang. NULL kalau riwayat belum cukup.
- `is_partial` (boolean, not null, default false): true kalau riwayat lebih pendek dari jendela.
- `computed_at` (timestamptz, not null)
- **PK gabungan:** (`repo_id`, `window_days`)

**Constraint:** CHECK `window_days IN (1,7,30)`.
**Index:** PK, plus index di (`window_days`, `velocity` descending, `repo_id`) untuk daftar Rising now. `repo_id` jadi pemecah seri supaya "Load more" tidak menggandakan atau melewatkan baris.
**Akses:** publik baca. Tulis hanya lewat `compute_repo_metrics()`.

**Aturan baca:**
- Query selalu digabung dengan `repos.status = 'active'`.
- Daftar repo naik hanya mengambil `velocity > 0`.
- Nomor peringkat tidak disimpan; dihitung dari urutan velocity pada jendela terpilih (rank = jumlah repo dengan velocity lebih besar, ditambah 1).
- Persen perubahan di kartu Stars gained hanya dari repo yang `prev_stars_gained`-nya tidak NULL. Kalau total periode sebelumnya 0 atau tidak ada, UI menampilkan "-".

**Aturan hapus:** baris metrics repo yang dikeluarkan (`evicted`) dihapus oleh function eviction.

## 4. repo_details

Data pendukung per repo yang tidak berubah tiap 6 jam. Relasi 1-ke-1 dengan `repos`. Diambil saat repo pertama masuk, lalu diperbarui paling cepat sekali per 24 jam.

- `repo_id` (uuid, PK, FK ke `repos.id`, on delete cascade)
- `weekly_commits` (integer[], nullable): jumlah commit 12 minggu terakhir, urut dari paling lama ke terbaru. NULL kalau GitHub belum menyiapkan statistiknya.
- `latest_release_tag` (text, nullable): NULL kalau belum ada rilis.
- `latest_release_published_at` (timestamptz, nullable)
- `latest_release_notes` (text, nullable): dipotong aplikasi sebelum disimpan.
- `readme_excerpt` (text, nullable): potongan awal README (markdown mentah), dipotong 4000 karakter. Dipakai sebagai cadangan kalau pengambilan README penuh gagal.
- `details_refreshed_at` (timestamptz, nullable)
- `recent_commits` (jsonb, not null, default `'[]'`): pesan commit terbaru (maksimal 20). Tiap elemen `{ "message": string, "date": ISO-8601 }` berisi **baris pertama** pesan commit (dipotong 200 karakter) dan `commit.committer.date`. **Tanpa nama atau email penulis.** Tidak ditampilkan di UI; hanya bahan konteks AI.

**Constraint:** CHECK `weekly_commits IS NULL OR cardinality(weekly_commits) = 12`, `jsonb_typeof(recent_commits) = 'array'`, `char_length(readme_excerpt) <= 4000`, `char_length(latest_release_notes) <= 4000`.
**Index:** PK, plus index di `details_refreshed_at`.
**Akses:** publik baca. Tulis hanya server.

**Keamanan konten:** README, catatan rilis, deskripsi, topik, dan pesan commit berasal dari pihak ketiga, jadi dianggap tidak aman. Di UI selalu dirender sebagai markdown yang disanitasi, tidak pernah HTML mentah. Saat dikirim ke AI, semuanya diperlakukan sebagai **data, bukan instruksi**.

## 5. ai_summaries

Cache ringkasan AI per repo di Detail Repo.

- `repo_id` (uuid, PK, FK ke `repos.id`, on delete cascade)
- `content` (text, nullable): isi ringkasan (poin markdown). NULL selama belum pernah berhasil.
- `model` (text, nullable)
- `status` (text, not null, default `'generating'`): `generating`, `ready`, `failed`.
- `generating_started_at` (timestamptz, nullable)
- `generated_at` (timestamptz, nullable): dipakai label "CACHED - [tanggal]".
- `based_on_snapshot_at` (timestamptz, nullable): snapshot terakhir yang dipakai sebagai bahan.

**Constraint:** CHECK `status IN ('generating','ready','failed')`.
**Akses:** publik baca. Tulis hanya server.

**Basi (stale) kalau:** `generated_at` lebih tua dari 24 jam, atau `based_on_snapshot_at` lebih lama dari snapshot terbaru repo.

**Idempotency dan biaya AI:**
- Pembuatan ulang wajib CAS: `UPDATE ai_summaries SET status='generating', generating_started_at=now() WHERE repo_id=? AND (status <> 'generating' OR generating_started_at < now() - interval '2 minutes')`.
- Kalau 0 baris terpengaruh, request lain sedang membuat. Request ini berhenti dan menampilkan cache lama.
- Repo yang belum punya baris: `INSERT ... ON CONFLICT (repo_id) DO NOTHING`; yang menang lanjut memanggil AI.
- Batas umur cache ada di `constants/`, bukan di database.

**Perilaku UI (stale-while-revalidate):** pada pembukaan halaman biasa, kalau cache basi maka generate dijadwalkan di belakang layar (`after()`) dan halaman langsung menampilkan isi lama dengan badge "memperbarui". Tombol regenerate manual menunggu hasilnya dan dibatasi rate limit.

**Catatan:** tidak ada kolom pesan error di tabel ini karena tabel ini bisa dibaca publik. Error AI dicatat lewat `console.error`. Kalau AI gagal tapi sudah ada `content` lama, `content` lama dipertahankan. Percobaan AI otomatis diulang sekali sebelum status jadi `failed`.

## 6. digests

Feed berita harian dari AI untuk halaman `/digest`. Satu baris satu hari. Punya dua bagian: (A) berita umum trending, (B) update Watchlist. Keduanya disimpan sebagai salinan (snapshot), jadi riwayat tidak berubah walau data berubah di hari berikutnya.

- `id` (uuid, PK, default `gen_random_uuid()`)
- `digest_date` (date, not null, **unique**): tanggal digest (UTC). Kunci anti-duplikat.
- `title` (text, nullable): judul buletin hari itu.
- `content` (text, nullable): lede atau pembuka (markdown). NULL selama belum berhasil.
- `tags` (text[], not null, default `'{}'`): chip topik.
- `trending` (jsonb, not null, default `'[]'`): berita umum, top repo trending (velocity 7 hari, maksimal 10). Tiap elemen: `{ "repo_id": uuid, "full_name": "owner/name", "description": string|null, "primary_language": string|null, "topics": string[], "velocity": number, "stars_gained": number, "blurb": string }`. `blurb` adalah tulisan AI "repo ini tentang apa dan kenapa naik".
- `stories` (jsonb, not null, default `'[]'`): update Watchlist per repo. Tiap elemen: `{ "repo_id": uuid, "full_name": "owner/name", "kind": "commits" | "release" | "commits_and_release", "headline": string, "summary": string, "occurred_at": ISO-8601, "commits": [{ "message": string, "date": ISO-8601 }], "release": { "tag": string, "name": string|null, "published_at": ISO-8601, "url": string } | null }`. `commits` berisi pesan baris pertama yang sudah dipotong (maksimal 10 per repo); `stories` maksimal 12 dan hanya memuat repo yang punya aktivitas (kosong pada hari sepi). Data `commits` tetap disimpan walau UI sekarang hanya menampilkan jumlahnya.
- `model` (text, nullable)
- `status` (text, not null, default `'generating'`): `generating`, `ready`, `failed`.
- `generating_started_at`, `generated_at` (timestamptz, nullable)
- `created_at` (timestamptz, default now())

**Constraint:** CHECK `status IN ('generating','ready','failed')`, `jsonb_typeof(trending) = 'array'`, `jsonb_typeof(stories) = 'array'`.
**Index:** unique di `digest_date`.
**Akses:** publik baca. Tulis hanya endpoint cron di server.

**Idempotency (empat langkah):**
1. `INSERT (digest_date, status='generating') ON CONFLICT (digest_date) DO NOTHING`.
2. Ambil top trending dari `repo_metrics` dan commit serta rilis segar untuk repo Watchlist, lalu panggil AI **sekali**. Hari tanpa aktivitas tetap memanggil AI.
3. `UPDATE ... SET title, content, tags, trending, stories, status='ready' WHERE id=? AND status='generating'` (CAS).
4. Baris `failed`, atau `generating` lebih dari 10 menit, boleh diambil alih panggilan cron berikutnya. Kalau digest hari itu sudah `ready`, endpoint selesai tanpa memanggil AI.

**Catatan:**
- Tidak ada kolom pesan error. Error dicatat di `job_runs`.
- `/digest` hanya menampilkan baris `ready`.
- Pesan commit, nama repo, dan catatan rilis adalah data pihak ketiga: diperlakukan sebagai data saat dikirim ke AI dan dirender aman sebagai teks di UI.

## 7. job_runs

Log tiap eksekusi job otomatis (collect, backfill, digest).

- `id` (uuid, PK, default `gen_random_uuid()`)
- `job_type` (text, not null): `collect`, `backfill`, atau `digest`.
- `status` (text, not null, default `'running'`): `running`, `success`, `partial` (selesai tapi sebagian gagal atau terpotong rate limit), `failed`.
- `started_at` (timestamptz, not null, default now())
- `finished_at` (timestamptz, nullable)
- `repos_processed` (integer, not null, default 0)
- `error_message` (text, nullable): pesan error asli untuk log internal, tidak pernah ditampilkan ke user.
- `details` (jsonb, not null, default `'{}'`): rincian (jumlah ditemukan dan dikeluarkan, sisa kuota GitHub, repo gagal). **Dilarang** berisi token atau secret.

**Constraint:** CHECK `job_type IN ('collect','backfill','digest')`, `status IN ('running','success','partial','failed')`.
**Index:**
- Partial unique di `job_type` `WHERE status = 'running'`: hanya satu job tipe yang sama boleh berjalan, insert kedua gagal dengan duplicate key (kode 23505) dan job itu berhenti tenang.
- Index di (`job_type`, `started_at` descending).

**Akses:** hanya server (tanpa policy).
**Catatan:** job `backfill` yang antreannya kosong tidak mencatat baris. Job `collect` dan `backfill` memakai kuota GitHub yang sama; sisa kuota dicatat di `details`.

## 8. rate_limits

Penghitung rate limit per IP berbasis database.

- `key` (text, not null): gabungan nama endpoint, jenis jendela, dan hash IP, contoh `chat:min:9f2c...`. IP di-hash dengan HMAC (secret server). **IP mentah tidak pernah disimpan atau di-log.**
- `window_start` (timestamptz, not null)
- `count` (integer, not null, default 0)
- **PK gabungan:** (`key`, `window_start`)

**Akses:** hanya server. Semua pengecekan lewat `check_rate_limit()` yang atomik.
**Pembersihan:** baris yang lebih tua dari 1 hari dihapus cron harian.
**Dua jendela:** endpoint dengan batas per menit dan per hari memanggil `check_rate_limit()` dua kali dengan key berbeda; ditolak kalau salah satunya false.
**Batas global:** endpoint yang menghabiskan token AI punya key tanpa IP (contoh `chat:global:day`) sebagai plafon total per hari.
**Kalau RPC rate limit error:** aplikasi fail-open (tetap diizinkan) dan mencatat `console.error`.

**Endpoint yang wajib dibatasi** (angka awal, nilai asli di `constants/`):
- Chat AI: 10 per menit dan 100 per hari per IP, plus global 500 per hari.
- Pencarian repo (dialog Add repo): 20 per menit.
- Tambah atau hapus Watchlist: 30 per menit.
- Buka repo yang belum ada di database lewat URL: 10 per menit.
- Buat ulang ringkasan AI: 10 per jam.

## 8b. chat_sessions dan chat_messages

Riwayat Chat AI, multi-sesi. Keduanya **server-only** karena riwayat chat bersifat privat.

### chat_sessions
- `id` (uuid, PK, default `gen_random_uuid()`)
- `title` (text, not null, default `'New chat'`): diambil dari pesan user pertama, dipotong.
- `repo_owner`, `repo_name` (text, nullable): konteks repo yang **menempel di sesi (pin)**. Diisi saat sesi dibuat dari `?repo=`, lalu dipakai ulang di semua turn sesi itu (termasuk saat sesi dibuka lagi tanpa parameter). Kalau request berikutnya membawa repo yang berbeda, server membuat **sesi baru**, bukan menimpa sesi lama.
- `created_at`, `updated_at` (timestamptz, not null, default now(); `updated_at` diisi trigger `set_updated_at`)

**Index:** (`updated_at` descending) untuk urutan sidebar.

### chat_messages
- `id` (uuid, PK, default `gen_random_uuid()`)
- `session_id` (uuid, FK ke `chat_sessions.id`, on delete cascade, not null)
- `role` (text, not null): `user` atau `assistant`.
- `content` (text, not null)
- `status` (text, not null, default `'ready'`): `ready` atau `failed` (jawaban AI gagal). Pesan user selalu `ready`.
- `sources` (jsonb, not null, default `'[]'`): salinan chip sumber `[{owner, name}]`.
- `created_at` (timestamptz, not null, default now())

**Constraint:** CHECK `role IN ('user','assistant')`, `status IN ('ready','failed')`, `jsonb_typeof(sources) = 'array'`.
**Index:** (`session_id`, `created_at`).

**Akses kedua tabel:** RLS aktif, tanpa policy, dan `REVOKE ALL` dari `anon` dan `authenticated`. Dibaca dan ditulis lewat admin client di server.

**Alur tulis (`POST /api/chat`):**
- Sesi dibuat saat pesan pertama (atau dipakai ulang).
- Pesan user disimpan **sebelum** AI dipanggil. Jawaban assistant disimpan **setelah** stream selesai; kalau gagal, `status='failed'` dengan isi pesan aman (pesan error asli hanya di `console.error`).
- Gagal menyimpan hanya di-log dan tidak menggagalkan jawaban.
- **Retry:** request dengan `retry: true` tidak menyimpan pesan user baru. Server menghapus pesan assistant **terakhir** sesi, lalu menulis jawaban baru. Tidak butuh ID pesan.
- **Bersihkan riwayat:** menghapus semua baris `chat_sessions` (cascade ke pesan).

## 9. Function dan Trigger

Semua function memakai `SET search_path = ''` dan nama objek lengkap dengan schema. Function **server-only** berstatus `SECURITY DEFINER`, hak `EXECUTE`-nya di-`REVOKE` dari `PUBLIC`, `anon`, dan `authenticated`, lalu di-`GRANT` hanya ke `service_role` (default Postgres memberi EXECUTE ke semua orang, jadi revoke wajib eksplisit).

- **`set_updated_at()`** (trigger, reusable): mengisi `updated_at` di `repos` dan `chat_sessions`. Tanpa REVOKE karena ini trigger function, bukan RPC.
- **`set_watchlist(p_repo_id uuid, p_watched boolean)`** (server-only): satu-satunya jalan menyalakan atau mematikan Watchlist. Pakai advisory lock transaksi supaya hitungan 50 aman. Menyalakan: kalau sudah watched, sukses (idempotent); kalau sudah 50, hasil `limit_reached`; selain itu set `is_watched=true`, `watched_at=now()`, `status='active'` (repo evicted otomatis aktif lagi). Mematikan: set `is_watched=false`, `watched_at=NULL`, `status` tidak diubah. Hasil: `ok`, `limit_reached`, atau `not_found`.
- **`claim_backfill_batch(p_limit int)`** (server-only): ambil repo `pending` sebanyak `p_limit` dengan `FOR UPDATE SKIP LOCKED`, set `running`, `backfill_started_at=now()`, `backfill_attempts+1`, lalu kembalikan daftarnya. Dua worker tidak pernah mengambil repo yang sama. Repo dengan `backfill_attempts >= 3` tidak diambil lagi dan ditandai `failed`.
- **`compute_repo_metrics()`** (server-only): hitung ulang `repo_metrics` untuk semua repo aktif yang punya snapshot, jendela 1, 7, 30. Idempotent (upsert). Aturan:
  - Titik acuan jendela adalah `captured_at` snapshot terbaru repo itu, bukan `now()`.
  - `stars_gained` = bintang snapshot terbaru dikurangi bintang snapshot terakhir sebelum awal jendela. Kalau riwayat belum sampai awal jendela, pakai snapshot paling awal dan `is_partial=true`.
  - `velocity` = `stars_gained` dibagi lama (hari) yang benar-benar tercakup. Lama 0 hari menghasilkan velocity 0, bukan error.
  - `prev_stars_gained` dihitung dengan cara sama untuk periode sebelumnya, NULL kalau riwayat belum cukup.
- **`evict_excess_repos(p_max_active int default 100)`** (server-only): pilih repo `active` yang tidak `is_watched` dan bukan `pending`/`running` backfill. Kalau jumlahnya melebihi batas, repo dengan velocity 14 hari terendah (seri: `discovered_at` paling lama) diubah jadi `evicted` dan baris `repo_metrics`-nya dihapus. Velocity 14 hari dihitung langsung dari `repo_snapshots`. **Hati-hati urutan sortir:** yang dipertahankan adalah velocity tertinggi (urut `DESC`), sisanya yang dikeluarkan. Angka `14 hari` di function ini hardcoded di SQL (dua tempat); `EVICT_VELOCITY_WINDOW_DAYS` di constants hanya rujukan dokumentasi.
- **`check_rate_limit(p_key text, p_limit int, p_window_seconds int)`** (server-only): atomik, `INSERT ... ON CONFLICT (key, window_start) DO UPDATE SET count = count + 1 RETURNING count`, hasil true kalau `count <= p_limit`. Tidak ada jalur "baca dulu baru tulis".
- **`get_star_series(p_repo_ids uuid[], p_days int)`** (boleh publik, `SECURITY INVOKER`, RLS berlaku): titik harian bintang (snapshot terakhir tiap hari) untuk banyak repo sekaligus. Satu panggilan untuk semua repo di layar. Input dibatasi: `p_days` antara 1 dan 90, dan maksimal 50 repo; di luar itu ditolak.

## 10. Cron Job (`pg_cron`, zona waktu UTC)

Semua job yang memanggil Edge Function atau endpoint app memakai `pg_net`, dengan header `Authorization: Bearer <CRON_SECRET>` yang **ditulis langsung** di `cron.job.command`. Ini keputusan sadar tanpa Vault. Risikonya: isi tabel `cron.job` terbaca oleh pemilik project lewat SQL Editor, itu saja. Konsekuensi wajib: **arsip migration di repo memakai placeholder `[REDACTED]`** untuk nilai secret, dan nilai secret tidak pernah dicetak ke log, output, atau dokumen. Endpoint dan Edge Function wajib menolak request tanpa secret yang cocok.

Daftar job:
- **`collect-repos`**: `0 */6 * * *`. Satu siklus pengumpulan.
- **`backfill-batch`**: `*/5 * * * *`. Memproses beberapa repo antrean backfill per panggilan. Antrean kosong langsung selesai.
- **`daily-digest`**: `0 1 * * *` (01:00 UTC = 08:00 WIB). Memanggil `{APP_URL}/api/cron/digest` dengan `timeout_milliseconds := 60000` pada `net.http_post` (wajib — lihat Catatan penting).
- **`daily-digest-retry`**: `30 1 * * *`. Panggilan kedua yang hanya bekerja kalau digest hari itu `failed` atau macet; juga memakai `timeout_milliseconds := 60000`. (Dua job terpisah karena `cron.schedule` dengan nama sama akan meng-update, bukan menambah.)
- **`reap-stale-jobs`**: `*/10 * * * *`. SQL murni: `job_runs` yang `running` lebih dari 15 menit jadi `failed`; repo `backfill_status='running'` dengan `backfill_started_at` lebih dari 15 menit dikembalikan ke `pending` (atau `failed` kalau percobaan sudah 3).
- **`cleanup-housekeeping`**: `0 3 * * *`. SQL murni: hapus `rate_limits` yang lebih tua dari 1 hari dan `cron.job_run_details` yang lebih tua dari 7 hari.

**Catatan penting:**
- `APP_URL` di job digest harus alamat publik. Selama masih localhost, job digest tidak akan berfungsi. Setelah deploy: ganti `APP_URL` di `.env` dan URL di `cron.job` kedua job digest. Sebelum `UPDATE cron.job`, uji dulu `cron.alter_job` (di beberapa project GRANT ke `cron.*` tidak menempel).
- `net.http_post` default `timeout_milliseconds` pg_net adalah **2000 ms (2 detik)**. Efeknya beda tergantung tujuannya:
  - **Job ke Edge Function** (`collect-repos`, `backfill-batch`): timeout pg_net **tidak fatal**. Request tetap terkirim dan function lanjut sampai selesai (mis. `collect-repos` sekitar 9 detik), jadi `net._http_response` boleh `timed_out = true` sementara `job_runs` tetap `success`. Sumber kebenaran status adalah `job_runs`.
  - **Job ke endpoint app di Vercel** (`daily-digest`, `daily-digest-retry`): timeout **fatal**. Begitu pg_net memutus koneksi, Vercel menghentikan fungsi, sehingga `job_runs` nyangkut `running` sampai di-reap setelah 15 menit. Karena digest butuh ~50 detik, kedua job digest **wajib** memakai `timeout_milliseconds := 60000`, selaras `maxDuration = 60` pada route `api/cron/digest`.
- Command job digest disusun dengan menyalin header dari command job `collect-repos` lalu mengganti URL-nya, jadi tidak ada literal secret di arsip migration.

**Langkah satu siklus `collect-repos`:**
1. Daftarkan diri di `job_runs` (duplicate key berarti siklus sebelumnya masih jalan, berhenti tenang). Kalau `GITHUB_TOKEN` kosong, langsung `failed` dengan pesan jelas.
2. Kalau tabel `repos` masih kosong (setup pertama): cari repo 30 hari terakhir (urut bintang, 30 teratas), simpan dengan `discovery_source='backfill'`, lalu berhenti. Backfill ditangani `backfill-batch`.
3. Ambil snapshot repo aktif, yang `last_synced_at` paling lama duluan, paralel kecil (5). Berhenti rapi (status `partial`) kalau sisa kuota GitHub di bawah ambang (50).
4. Perbarui `repo_details` untuk repo yang `details_refreshed_at`-nya lebih tua dari 24 jam atau masih kosong (termasuk `recent_commits`).
5. Penemuan repo baru: repo 7 hari terakhir (urut bintang), minimal 50 bintang, belum ada di database (termasuk yang `evicted`), maksimal 10 per siklus, `discovery_source='discovery'`. Dilewati kalau kuota menipis.
6. Jalankan `evict_excess_repos(100)`, lalu `compute_repo_metrics()`. Langkah ini tetap jalan walau langkah 5 dilewati.
7. Tutup baris `job_runs` (`success`, `partial`, atau `failed`) dengan `details` berisi jumlah ditemukan, dikeluarkan, sisa kuota, dan repo gagal.

**Status setup untuk kartu progres Dashboard** (dihitung dari `repos` dengan `discovery_source='backfill'`, tanpa tabel tambahan):
- Belum ada repo sama sekali: "Finding new repos".
- Masih ada yang `pending` atau `running`: "Fetching star history" (angka "12 of 30 repos" = jumlah `done` dibanding total).
- Tidak ada yang `pending`/`running` tapi `repo_metrics` masih kosong: "Calculating velocity".
- Selain itu: setup selesai, tampil Dashboard normal. Repo yang akhirnya `failed` tidak menahan Dashboard.
- Repo dari penemuan otomatis atau tambah manual tidak memunculkan kartu progres.

## 11. Row Level Security (RLS)

**RLS wajib di-ENABLE di semua tabel**, termasuk tabel server-only. Tabel tanpa policy dengan RLS aktif berarti deny total untuk `anon` dan `authenticated`. Tabel server-only (`job_runs`, `rate_limits`, `chat_sessions`, `chat_messages`) juga di-`REVOKE ALL` dari `anon` dan `authenticated` sebagai lapis kedua.

Dua kelas akses saja (Tanpa login):
- **Publik (anon):** browser memakai anon key, hanya boleh baca tabel yang memang untuk ditampilkan. Policy baca ditulis untuk `anon` dan `authenticated` sekaligus.
- **Server-only (service role):** dipakai Server Action, Route Handler, dan Edge Function untuk semua tulis. Service role bypass RLS dan tidak pernah sampai ke browser.

**Keamanan wajib:** matikan signup publik di Supabase Auth (Dashboard, Authentication, Sign In / Providers, "Allow new users to sign up" = OFF). App tidak memakai Auth, jadi tidak ada alasan membiarkannya terbuka.

**Perlu disadari:** semua endpoint tulis (cari repo, watchlist, buka repo lewat URL, chat, hapus riwayat chat, buat ulang ringkasan) bisa dipanggil siapa pun yang tahu alamat app, dan tiap panggilan memakai kuota GitHub atau token AI. Mitigasinya rate limit di semua endpoint itu, ditambah proteksi di level hosting selama belum ada login.

Aturan per tabel:
- `repos`, `repo_snapshots`, `repo_metrics`, `repo_details`: baca publik semua baris; insert, update, delete tanpa policy (hanya server).
- `ai_summaries`, `digests`: baca publik semua baris (filter `status='ready'` di query aplikasi); tulis hanya server.
- `job_runs`, `rate_limits`, `chat_sessions`, `chat_messages`: server-only, tanpa policy sama sekali.

Function:
- **Server-only** (EXECUTE hanya `service_role`): `set_watchlist`, `claim_backfill_batch`, `compute_repo_metrics`, `evict_excess_repos`, `check_rate_limit`.
- **Publik:** `get_star_series` (read-only, tunduk RLS, input dibatasi).
- **Trigger:** `set_updated_at`.

Advisor security Supabase akan menampilkan INFO `rls_enabled_no_policy` untuk tabel server-only. Itu disengaja.


## 12. Catatan Umum

- Semua kolom status dan jenis disimpan sebagai `text` dengan CHECK constraint (bukan Postgres `enum`), supaya mudah menambah nilai baru.
- Batas bisnis (Watchlist 50, repo aktif 100, penemuan maksimal 10 per siklus dengan minimal 50 bintang, percobaan backfill 3, batas halaman backfill, rate limit, umur cache AI) disimpan di `constants/`, bukan di database. Yang butuh aman dari race condition (50 Watchlist, 100 repo aktif) ditegakkan lewat function.
- Tulis multi-langkah mengikuti urutan "paling reversible dulu": simpan repo (`ON CONFLICT`), baru jadwalkan backfill, baru nyalakan Watchlist. Tiap langkah idempotent dan bisa diulang.
- **Tidak ada seed data demo.** Dashboard kosong menampilkan kartu progres setup.
- Pagination: Dashboard 6 awal lalu 12 per klik (urut `velocity` lalu `repo_id`), `/digest` berbasis `digest_date` (5 per halaman), Watchlist maksimal 50 baris jadi diambil sekaligus.
- Select kolom spesifik, bukan `SELECT *`. Terutama `readme_excerpt` dan `ai_summaries.content`, hanya diambil di Detail Repo.
- Semua FK punya index (kolom FK adalah kolom pertama di PK atau unique index).
- Data hasil hitung (`repo_metrics`) dan historis (`repo_snapshots`, `digests`) tidak diedit manual. Kalau salah, hitung ulang atau buat ulang.
- Error di semua job dan function server dicatat lewat `console.error` dan `job_runs`. Secret dan token tidak pernah ikut ke log.

