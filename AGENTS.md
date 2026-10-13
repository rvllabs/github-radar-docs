<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

# GitHub Radar — Agent Working Rules

## 1. Wajib dibaca dulu sebelum kerja apapun
Sebelum menyentuh kode, baca urutan ini secara penuh (bukan skimming). Semua ada di folder `docs/`:
1. `docs/PRD.md` — apa yang dibangun, alur fitur, struktur halaman, env.
2. `docs/DESIGN.md` — token warna, tipografi, radius, motion, dan spesifikasi tampilan per halaman. **Sumber kebenaran untuk warna, font, radius, dan tampilan.** Struktur dan layout mengikuti mockup Stitch; kalau mockup dan DESIGN.md berbeda, DESIGN.md menang.
3. `docs/SCHEMA.md` — struktur database, function, cron job, RLS. **Sumber kebenaran untuk semua tabel, query, dan migration.**
4. `docs/RULES.md` — aturan teknis project (struktur folder, security, idempotency, rate limit, dan lain-lain). Berlaku di SEMUA task dan tidak diulang per task. **Sumber kebenaran untuk penempatan file dan konvensi kode.**
5. `docs/TASK.md` — urutan kerja, Aturan Umum, dan task yang sedang aktif.

**Kalau dokumen bentrok:** SCHEMA.md menang untuk data, DESIGN.md menang untuk tampilan, RULES.md menang untuk penempatan file.

**Soal bentrok TASK.md vs RULES.md secara spesifik, bersikap fleksibel, jangan langsung stop:**
- Kalau instruksi teknis di satu task TASK.md sedikit beda penekanan atau gaya dengan RULES.md tetapi tidak mengorbankan keamanan atau data (contoh: detail kecil penamaan, struktur file, atau cara implementasi yang sama-sama masuk akal), ikuti TASK.md apa adanya (dia lebih spesifik ke konteks task itu), lanjut kerja, dan sebutkan di laporan akhir task kalau ada penyesuaian kecil dari RULES.md.
- Kalau bentroknya menyangkut hal kritis (security, RLS, idempotency atau CAS, secret yang bocor ke client, rate limit, dan apapun yang kalau salah bisa merusak data atau membuka celah keamanan), **berhenti dan tanya dulu**. Jangan menebak dan jangan diam-diam menurut salah satu.
- Kalau ragu bentrokan itu masuk kategori "kecil" atau "kritis", anggap kritis dan tanya dulu. Lebih aman bertanya sekali daripada coding salah arah lalu membongkar ulang.

## 2. Satu task, satu waktu — jangan loncat
- Kerjakan `docs/TASK.md` **berurutan dari atas**. Cari task pertama yang checklist Pengerjaan atau Verifikasinya belum penuh tercentang, atau yang baris "Kamu cek manual"-nya masih `[ ]`.
- Jangan mulai atau menyentuh file dari task berikutnya walau kelihatan lebih efisien "sekalian saja". Dependency antar task sengaja disusun linear. Loncat urutan berisiko mengulang kerjaan.
- Task yang baris "Kamu cek manual"-nya **sudah dicentang user** dianggap final dan terkunci. Jangan diubah lagi kecuali user eksplisit meminta revisi ke task itu.
- **Stitch:** baca `STITCH_PROJECT_ID` di bagian Aturan Umum TASK.md. Kalau nilainya masih placeholder (`[ISI_PROJECT_ID_STITCH_KAMU]`), **berhenti dan minta user mengisinya**. Untuk task yang punya acuan layar, ambil desain lewat Stitch MCP (`get_screen`, `fetch_screen_code`, `fetch_screen_image`) langsung. Kalau layar tidak bisa dibuka (MCP belum terhubung atau akses belum aktif), berhenti dan beri tahu user. Jangan menebak atau mengarang layout dari deskripsi teks.
- **Database:** jalankan lewat Supabase MCP (`apply_migration`), satu migration per objek, **jangan dipanggil paralel**, bukan lewat SQL Editor Dashboard. Arsip SQL ke `src/migration/` ditarik dari `supabase_migrations.schema_migrations` (kolom `statements`), bukan ditulis ulang dari ingatan. Nilai secret di arsip diganti `[REDACTED]`.
- **Secret dan env:** **jangan pernah mengedit atau menimpa `.env.local`** dan jangan mencetak nilai secret ke output, log, dokumen, atau commit. AI hanya boleh memverifikasi nama env ada dan membaca nilainya untuk keperluan dev yang memang diminta task. Langkah bertanda **(Kamu)** di TASK.md (isi `.env.local`, secret Edge Function di Dashboard, matikan signup publik, deploy, ganti `APP_URL`) dikerjakan user. AI tidak mengerjakannya; cukup ingatkan dan tunggu konfirmasi.
- **Bagian "Setelah Deploy"** di akhir TASK.md baru dikerjakan setelah user mengabarkan aplikasi sudah live di Vercel. Kerjakan hanya butir bertanda **(AI)**.

## 3. Alur kerja per task (WAJIB diikuti persis, ini yang paling penting)
1. Baca detail task yang aktif di TASK.md secara penuh: semua checklist item, "Acuan", "Lokasi", dan jebakan yang tertulis.
2. Implementasikan checklist **Pengerjaan** satu per satu dari atas ke bawah. Setiap satu item selesai dan benar, langsung ubah `[ ]` menjadi `[x]` item itu di TASK.md sebelum lanjut. Jangan mengerjakan semua dulu lalu mencentang sekaligus, dan jangan loncat urutan item. Ikuti RULES.md di setiap baris kode (struktur folder, naming, idempotency, RLS, rate limit, dan seterusnya).
3. Setelah semua Pengerjaan tercentang, jalankan **trio hijau** (`npx tsc --noEmit` 0 error, `npm run lint` bersih, `npm run build` sukses tanpa warning route dinamis tanpa boundary) plus `npm test`. Lalu cek satu per satu poin di **Verifikasi** task itu dan centang yang lolos. Kalau ada yang gagal, perbaiki dulu sampai lolos. Jangan melapor task selesai kalau ada poin Verifikasi (selain "Kamu cek manual") yang belum tercentang. Data uji sementara wajib dibersihkan.
4. Update bagian **"File yang sudah dibuat"** di paling bawah TASK.md. **Wajib dikelompokkan per task, bukan daftar datar:** buat atau tambah heading `### TASK N — <judul task persis seperti di TASK.md>` (berurutan sesuai urutan task), lalu daftarkan tiap file baru atau diubah dari task itu di bawah heading tasknya. Format satu baris per file: tanda strip, path file dalam backtick, tanda "—", lalu deskripsi singkat dan jujur (contoh deskripsi: "client GitHub murni fetch dengan mapping error"; bukan deskripsi generik seperti "helper file"). Jangan tag `(Task N)` di tiap baris. Kalau task ini menyentuh file yang sama dengan task sebelumnya (misal `package.json`), tetap catat juga di bawah heading task sekarang.
5. Update baris **Status** di bawah judul TASK.md (isi apa yang baru dikerjakan dan tanggalnya, contoh: `Task 3 selesai, menunggu review (10 Oktober 2026)`).
6. Baris **"Kamu cek manual"** milik task itu **SELALU TETAP `[ ]`**. Ini satu-satunya baris di seluruh TASK.md yang TIDAK PERNAH dicentang AI dalam kondisi apapun, walau semua Pengerjaan dan Verifikasi lain sudah tercentang. Itu murni hak review manual user, penanda "AI klaim selesai, menunggu user cek beneran".
7. **STOP di sini.** Jangan lanjut ke task berikutnya dan jangan mulai kerjaan lain. Laporkan ke user secara singkat: apa yang dikerjakan, file apa yang dibuat atau diubah, penyesuaian dari dokumen kalau ada, dan konfirmasi semua Pengerjaan plus Verifikasi sudah tercentang.

## 4. Setelah user review
- Kalau user bilang ok, mencentang "Kamu cek manual" sendiri, atau bilang "lanjut", lanjut ke task berikutnya dan ulangi Bagian 3 dari langkah 1.
- Kalau user memberi koreksi, perbaiki **persis sesuai koreksi yang diberikan** dan jangan menafsirkan scope lebih luas dari yang diminta. Setelah selesai, ulangi langkah 3 sampai 7 di Bagian 3 (file list, Status, "Kamu cek manual" tetap `[ ]`, stop, lapor lagi).
- Jangan pernah menganggap "diam = setuju". Kalau user belum bilang apa-apa soal hasil task, jangan lanjut task berikutnya sendiri.

## 5. Batasan tambahan

- Jangan memodifikasi `docs/PRD.md`, `docs/DESIGN.md`, `docs/SCHEMA.md`, dan `docs/RULES.md` kecuali user eksplisit meminta. Dokumen itu sumber kebenaran, bukan draft yang boleh AI edit sambil jalan. Di `docs/TASK.md`, AI hanya boleh mencentang checklist, mengisi baris Status, dan mengisi bagian "File yang sudah dibuat". **Jangan mengubah isi spesifikasi task.**
- Kalau instruksi di satu task butuh keputusan yang belum ditentukan di dokumen manapun (bukan sekadar detail kecil), berhenti dan tanya user. Jangan mengarang keputusan arsitektur secara diam-diam.
- Selalu commit dan laporkan di granularitas **per task**, bukan beberapa task digabung.

## 6. Tools yang Wajib Dipakai
Tools MCP atau plugin berikut sudah terpasang di environment ini dan **wajib dipakai setiap kali relevan**, bukan opsional:
- **Context7:** sebelum menulis kode yang menyangkut library atau framework apapun (Next.js, Supabase SDK, Tailwind, Zod, React Query, Recharts, dan lain-lain), cek dulu dokumentasi terbaru lewat Context7. Jangan mengasumsikan API atau syntax dari training data, karena banyak breaking changes (Next.js di project ini adalah versi yang baru dan berbeda dari yang kamu kenal).
- **Superpowers:** ikuti metodologi brainstorm, plan, implement, review bawaan skill ini di setiap task, selaras dengan alur di Bagian 3. Kalau Superpowers dan alur project ini kelihatan tumpang tindih, alur project ini (RULES.md dan checklist TASK.md) menang untuk hal spesifik project. Superpowers dipakai untuk disiplin proses berpikir, bukan menggantikan aturan project.
- **Supabase MCP** dan **Stitch MCP:** wajib dipakai sesuai konteksnya (migration dan pengecekan database, ambil desain), seperti di Bagian 2.

Prinsipnya: kalau ada tool yang bisa memberi informasi lebih akurat atau terbaru dibanding asumsi atau ingatan sendiri, **selalu pakai tool itu dulu** sebelum menulis kode atau mengambil keputusan.

## 7. Fleksibilitas terhadap Dokumen
Dokumen (`PRD.md`, `DESIGN.md`, `SCHEMA.md`, `RULES.md`, `TASK.md`) adalah sumber kebenaran default, **bukan aturan mati yang harus diikuti buta** kalau ternyata ada yang kurang tepat:
- Kalau menemukan instruksi di dokumen yang secara teknis **kurang optimal**, **outdated** (misal API atau library yang direferensikan sudah berubah), atau ada **cara yang lebih best-practice**, jangan diam-diam mengikuti yang salah, tetapi juga jangan diam-diam menggantinya sendiri. **Informasikan ke user:** apa yang menurutmu kurang pas, kenapa, dan usulan perbaikannya, supaya user yang memutuskan.
- Ini **beda** dari aturan di Bagian 1 soal bentrok TASK.md vs RULES.md (itu soal dua dokumen saling bentrok). Bagian ini soal satu dokumen yang kelihatan kurang tepat dibanding best practice terkini atau temuan dari tool (Context7 dan lain-lain).
- Untuk hal kecil (naming, gaya penulisan, detail implementasi yang sama-sama valid), boleh langsung menyesuaikan ke yang lebih baik dan sebutkan di laporan akhir task.
- Untuk hal berdampak besar (arsitektur, skema data, keamanan, alur bisnis inti), **tetap wajib tanya dulu**. Jangan mengasumsikan user pasti setuju walau usulannya lebih baik secara teknis.
- Intinya: dokumen ini baseline yang boleh dan seharusnya dikoreksi kalau memang ada yang lebih baik, dengan tetap transparan ke user, bukan diam-diam menyimpang.
