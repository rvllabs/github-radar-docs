# RULES.md — Aturan Project (jarang berubah, paste sekali di awal chat AI)

## 1. Struktur Folder
- `actions.ts` — semua panggilan Supabase (query/insert/update), **colocated per fitur/route** (contoh: `src/app/admin/(menu)/categories/actions.ts`, `src/app/(customer)/checkout/[slug]/actions.ts`) — bukan folder global `src/actions/` terpisah. Kalau ada logic Supabase yang dipakai lintas-fitur (bukan spesifik 1 route), baru ditarik ke `src/lib/`.
- `constants/` — nilai tetap (status, role, limit). Jangan hardcode di logic.
- `lib/` — function murni non-UI (format tanggal, rupiah, slugify), client Supabase, dan wrapper pihak ketiga. Satu file satu domain.
- `validations/` — skema Zod, pisah dari komponen form
- 1 komponen = 1 tanggung jawab (jangan campur UI+fetch+validasi)

## 2. Naming
- camelCase: function & variable
- PascalCase: komponen & file komponen

## 3. Bug Prevention
- Data Supabase dicek dulu: `data?.field ?? fallback`
- Semua async/await wajib try-catch
- Tiap fetch wajib state loading/success/error
- Cek dependency array useEffect (no infinite loop/fetch berulang)

## 4. Query Supabase
- Hindari N+1 — pakai Promise.all atau join langsung
- Select kolom spesifik, jangan `SELECT *`
- List data wajib pagination
- Index di kolom WHERE/JOIN/ORDER BY (user_id, status, created_at)

## 5. RLS & Security
- Tiap tabel dicek 1-per-1: policy SELECT/INSERT/UPDATE/DELETE
- Tiap policy jelas berlaku buat siapa: anon / authenticated / role spesifik
- Function/trigger cek SECURITY DEFINER vs INVOKER dengan tepat
- Semua FK (user_id, order_id, dll) punya index
- `SUPABASE_SERVICE_ROLE_KEY` cuma server-side, gak pernah ke client

## 6. Performa
- useMemo/useCallback di komponen re-render berat/sering
- Dynamic import buat komponen besar
- Cek props/state penyebab re-render gak perlu

## 7. Lain-lain
- Kode dikasih komentar singkat (1-2 baris) tiap function/logic penting
- Form publik (misal request konsultasi) wajib rate limiting anti-spam bot
- Siapin fallback UI kalau Supabase quota habis (return 402), jangan layar putih


## 8. Idempotency & Race Condition (WAJIB untuk write ke tabel kritis/transaksional)
Tabel kritis = tabel yang nyimpen state transaksi/uang/status penting (contoh: order, payment, invoice, subscription, atau tabel apapun yang perubahannya berdampak besar kalau salah/dobel).
- Setiap INSERT yang bisa ke-trigger dobel (retry network, double-click, race dari 2 request bersamaan) WAJIB pakai unique constraint + tangani error duplicate key (fetch row yang menang, jangan crash).
- Setiap UPDATE status penting WAJIB pakai CAS (compare-and-swap): filter query pakai status lama yang diharapkan, biar gak nimpa perubahan yang udah kejadian duluan dari request lain.
- Retry logic WAJIB dibatasi max attempt (jangan infinite retry) dan kasih fallback error yang jelas ke user.

## 9. Multi-Step Write / Atomicity
Berlaku untuk flow yang nulis ke lebih dari 1 tabel atau lebih dari 1 storage/resource dalam 1 aksi.
- Kalau database yang dipakai gak punya transaction bawaan yang gampang dipanggil dari client, susun urutan step: **paling reversible dulu, paling ireversibel terakhir** (contoh: upload file dulu baru insert row DB, bukan sebaliknya — biar kalau gagal di step DB, file bisa langsung dihapus).
- Kalau salah satu step gagal di tengah, WAJIB ada rollback/cleanup eksplisit — jangan dibiarin data nyangkut setengah jadi.
- Selalu cek return value/error tiap step sebelum lanjut ke step berikutnya, jangan asumsikan step sebelumnya pasti sukses.

## 10. Error Monitoring & Logging
- Semua error di server-side (server action/API route/backend function) WAJIB di-log minimal pakai `console.error` — jangan sampai ada error yang ke-catch tapi didiemin gitu aja (silent fail).
- Log ke tempat yang lebih persisten (tabel log di database, atau service eksternal seperti Sentry) — OPSIONAL, tapi sangat disarankan untuk error di flow kritis (pembayaran, webhook, perubahan status penting), karena `console.error` doang gampang ke-miss dan gak ada histori kalau server restart.
- Log WAJIB include: nama function, input relevan (JANGAN log data sensitif seperti password/token/kartu kredit), pesan error asli, timestamp.
- Error yang ditampilkan ke user WAJIB pesan yang jelas & actionable — jangan tampilkan raw error/stack trace dari server.
- Error di webhook/integrasi pihak ketiga WAJIB log payload masuk + hasil pemrosesan (minimal via `console.error`/`console.log`, OPSIONAL persisten ke DB/service), buat kebutuhan audit kalau ada dispute.

## 11. Environment Variables
- Semua env var dikumpulkan lewat satu objek terpusat (`src/config/environment.ts`), jangan panggil `process.env` langsung di file lain.
- Env var yang isinya **secret/kredensial berbayar** (API key Mayar/Brevo, service role key, dll) TIDAK BOLEH diam-diam fallback ke `""` — minimal beri warning jelas (`console.error`/`throw`) kalau kosong saat dipakai, supaya nggak ada request nembak ke API pihak ketiga dengan key kosong tanpa ketauan. Validasi ketat pakai Zod di satu tempat itu direkomendasikan, tapi tidak wajib bikin seluruh app gagal start kalau env non-kritis (misal `NEXT_PUBLIC_CONTACT_EMAIL`) belum diisi saat development.
- Env var yang berisi secret/key rahasia WAJIB dipastikan cuma dipakai di server-side, dan ditegaskan lewat komentar/penamaan yang jelas (contoh prefix yang beda antara public vs server-only).

## 12. Rate Limiting & Abuse Protection
- WAJIB diterapkan bukan cuma di form publik, tapi juga di: endpoint login/auth, endpoint yang manggil API pihak ketiga berbayar, dan endpoint apapun yang bisa di-spam tanpa autentikasi.
- Terapkan minimal salah satu: rate limit per IP, per user, atau per session — sesuaikan sama konteks endpointnya. Kalau infra rate-limiter (Redis/Upstash) belum ada di project, guard sederhana (contoh: cek duplikat request dalam beberapa menit terakhir) boleh dipakai sementara asal didokumentasikan sebagai keputusan sadar, bukan celah yang kelewat.

## 13. Testing Minimal
- Function murni (di `lib/`) yang punya logic non-trivial (kalkulasi, parsing, validasi kompleks) WAJIB punya unit test.
- Flow otorisasi/akses data (RLS atau access-control custom) WAJIB ditest minimal manual: user A tidak bisa akses data milik user B, role rendah tidak bisa akses fitur role tinggi.
- Flow kritis (pembayaran, perubahan status penting) WAJIB ditest untuk edge case: gagal di tengah, dipanggil dobel, input invalid/kosong.

## 14. Dependency & Secret Hygiene
- Audit dependency berkala (contoh: `npm audit`) sebelum deploy, terutama kalau ada dependency baru ditambahkan.
- Secret/API key WAJIB gak pernah di-hardcode di kode maupun di-commit ke git — selalu lewat env var.
- Kalau ada secret yang ke-expose gak sengaja (commit, log, error message), WAJIB rotate key tersebut, bukan cuma dihapus dari kode.

## 15. Accessibility & SEO (kalau aplikasi consumer-facing)
- Semua image WAJIB ada alt text yang deskriptif.
- Struktur heading (h1/h2/h3) WAJIB berurutan & semantik, jangan cuma buat styling.
- Halaman publik WAJIB ada meta title/description yang relevan.

## 16. Performa Lanjutan
- Semua gambar WAJIB dioptimasi: pakai komponen/format image optimization bawaan framework (contoh: `next/image`) atau format modern (WebP/AVIF), jangan taruh file gambar mentah ukuran besar tanpa kompresi.
- Gambar yang gak keliatan di viewport awal WAJIB lazy-load (kecuali gambar utama di atas layar/hero yang emang harus langsung tampil).
- Data yang jarang berubah (kategori, konfigurasi, konten statis) WAJIB di-cache (cache di level query/fetch, atau HTTP cache header) — jangan fetch ulang dari awal tiap request kalau datanya gak berubah.
- Sebelum nambah dependency/library baru, cek dulu apakah fungsinya udah bisa dicapai dengan yang sudah ada di project — hindari nambah library besar cuma buat 1 fungsi kecil (bikin bundle size bengkak).
- Import library besar WAJIB pakai named/selective import (bukan import seluruh library) kalau library-nya mendukung, biar tree-shaking jalan.

## 17. Keamanan Lanjutan
- Input dari user yang ditampilkan balik ke UI (nama, komentar, teks bebas, dll) WAJIB dianggap tidak aman — jangan pernah di-render sebagai HTML mentah tanpa sanitasi/escape (cegah XSS).
- Upload file WAJIB divalidasi di server-side (bukan cuma client-side): tipe file (whitelist, bukan blacklist), ukuran maksimal, dan kalau memungkinkan validasi isi file (bukan cuma percaya ekstensi/nama file).
- Otorisasi TIDAK BOLEH cuma mengandalkan RLS/policy database — kalau ada aturan akses yang lebih spesifik dari sekadar "kepemilikan data" (contoh: cuma boleh diedit kalau status masih draft, cuma boleh dihapus dalam rentang waktu tertentu), WAJIB dicek eksplisit juga di level action/business logic, jangan asumsi RLS udah cukup.
- Session/token WAJIB ada mekanisme expiry yang jelas, dan aplikasi WAJIB menangani kondisi token expired dengan baik (redirect ke login / minta refresh), bukan error mentah atau state nyangkut.
- Data sensitif (password, token, nomor kartu, dll) TIDAK BOLEH pernah ikut ke-log, ke-return di response API kalau gak perlu, atau ke-expose di client-side state yang gak semestinya.

## 18. Code Quality & Konsistensi
- TypeScript WAJIB strict mode aktif. Hindari `any` — kalau terpaksa pakai (misal tipe dari library eksternal yang gak jelas), kasih komentar alasannya.
- Kalau ada logic yang sama dipakai di 2+ tempat (bukan cuma UI, juga berlaku untuk query pattern, validasi, format data), WAJIB ditarik jadi 1 function/hook yang di-reuse — jangan duplikat.
- Semua function yang bisa gagal (action, API call) WAJIB balikin bentuk error yang konsisten di seluruh project (contoh: selalu `{ data, error }` atau pola lain yang disepakati) — jangan beda-beda shape antar file, biar predictable dipakai di komponen manapun.
- Dead code (import/variable/function/komponen yang gak dipanggil di manapun) WAJIB dihapus, jangan dibiarin nyangkut.