<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

# RVL Labs — Agent Working Rules

## 0. Wajib dibaca dulu sebelum kerja apapun
Sebelum menyentuh kode, baca urutan ini secara penuh (bukan skimming):
1. `docs/PRD.md` — apa yang dibangun, flow fitur, page structure.
2. `docs/DESIGN.md` — token warna, tipografi, motion, aturan visual. Sumber kebenaran untuk styling & Disesuaikan juga dengan desain dari stich.
3. `docs/SCHEMA.md` — struktur database, storage bucket, RLS. Sumber kebenaran untuk semua query/migration.
4. `docs/RULES.md` — aturan teknis project (struktur folder, security, idempotency, dll). Berlaku di SEMUA task, tidak diulang per-task.
5. `docs/TASK.md` — urutan kerja & task yang sedang aktif.
6. `docs/STRUCTURE.md` — aturan dasar penempatan file.


**Soal bentrok TASK.md vs RULES.md secara spesifik — bersikap fleksibel, jangan langsung stop:**
- Kalau instruksi teknis di satu task TASK.md sedikit beda penekanan/gaya sama RULES.md tapi nggak mengorbankan keamanan atau data (contoh: detail kecil penamaan, struktur file, atau cara implementasi yang sama-sama masuk akal) → ikuti TASK.md apa adanya (dia lebih spesifik ke konteks task itu), lanjut kerja, cukup sebutkan di laporan akhir task kalau ada penyesuaian kecil dari RULES.md.
- Kalau bentroknya menyangkut hal kritis (security, RLS, idempotency/CAS, secret yang bocor ke client, dll — apapun yang kalau salah bisa bikin data/uang rusak atau celah keamanan) → **berhenti dan tanya dulu**, jangan menebak, jangan diam-diam nurut salah satu.
- Kalau ragu bentrokan itu masuk kategori "kecil" atau "kritis" → anggap kritis, tanya dulu. Lebih aman nanya sekali daripada ngoding salah arah lalu harus dibongkar ulang.

## 1. Satu task, satu waktu — jangan loncat
- Kerjakan `docs/TASK.md` **berurutan dari atas**. Cari task pertama yang checkbox utamanya belum penuh dicentang DAN "Verifikasi dari Gua"-nya masih `[ ] Belum`.
- Jangan mulai atau menyentuh file dari task berikutnya walau kelihatan lebih efisien "sekalian aja". Dependency antar-task sengaja disusun linear di TASK.md — loncat urutan = resiko bikin ulang kerjaan.
- Task yang "Verifikasi dari Gua"-nya **sudah dicentang** dianggap final dan terkunci. Jangan diubah-ubah lagi kecuali user eksplisit minta revisi ke task itu.
- Untuk task yang butuh Stitch mockup: ambil desain via Stitch MCP (`get_screen`/`fetch_screen_code`/`fetch_screen_image`) langsung, jangan menebak/mengarang layout dari deskripsi teks saja.
- Untuk task yang butuh migration/DB: jalankan lewat Supabase MCP (`apply_migration`, dst), bukan nulis manual ke SQL Editor Dashboard — kecuali task itu sendiri bilang beda.

## 2. Alur kerja per task (WAJIB diikuti persis, ini yang paling penting)
1. Baca detail task yang aktif di TASK.md secara penuh, termasuk semua checklist item, "Acuan", dan "Dependency"-nya.
2. Implementasikan checklist item **satu per satu secara berurutan dari atas ke bawah**. Setiap satu item selesai dan benar, langsung centang `[ ]` → `[x]` item itu juga di TASK.md sebelum lanjut ke item berikutnya — jangan kerjakan semua dulu baru centang belakangan sekaligus, dan jangan loncat urutan item dalam satu task. Ikuti RULES.md di setiap baris kode yang ditulis (struktur folder, naming, idempotency, RLS, dll — RULES.md tidak diulang di sini, itu berlaku otomatis).
3. Setelah semua checklist item task itu tercentang, cek ulang sendiri satu-satu poin di "Verifikasi/Acceptance/Test" milik task itu, dan centang tiap poin yang lolos. Kalau ada yang gagal, perbaiki dulu sampai lolos sebelum lanjut ke langkah 4 — jangan laporkan task selesai kalau ada acceptance test yang belum tercentang.
4. Update section **"File yang sudah dibuat"** di bagian paling bawah TASK.md — **wajib dikelompokkan per task, bukan daftar datar**: buat/tambah heading `### TASK N — <judul task persis seperti di TASK.md>` (berurutan sesuai urutan task, satu heading per task), lalu daftarkan tiap file baru/diubah dari task itu sebagai baris `- \`path/file\` — deskripsi singkat dan jujur (bukan generik "helper file")` tepat di bawah heading tasknya. Kalau heading task itu belum ada, buat baru setelah heading task terakhir yang sudah ada; kalau sudah ada, cukup tambahkan barisnya. Jangan tag `(Task N)` di tiap baris (sudah tercakup heading). Kalau task yang sama menyentuh file yang sama dengan task sebelumnya (misal `package.json`), tetap dicatat juga di bawah heading task sekarang.
5. Update baris **"Status"** dan **"Last updated"** di header TASK.md sesuai task yang baru dikerjakan.
6. Isi bagian **"Verifikasi dari Gua"** milik task itu, tapi checkbox-nya **SELALU TETAP `[ ] Belum`** — ini satu-satunya checkbox di seluruh TASK.md yang TIDAK PERNAH dicentang AI dalam kondisi apapun, walau semua checklist item dan acceptance test di atasnya sudah tercentang penuh. Ini murni hak review manual dari user, penanda "AI klaim selesai, menunggu user cek beneran".
7. **STOP di sini.** Jangan lanjut ke task berikutnya, jangan mulai kerjaan lain. Laporkan ke user secara singkat: apa yang dikerjakan, file apa saja yang dibuat/diubah, dan konfirmasi semua checklist + acceptance test task ini sudah tercentang.

## 3. Setelah user review
- Kalau user bilang ok / centang checkbox "Verifikasi dari Gua" sendiri / bilang "lanjut" → lanjut ke task berikutnya, ulangi Alur di Bagian 2 dari langkah 1.
- Kalau user kasih koreksi → perbaiki **persis sesuai koreksi yang diberikan**, jangan reinterpretasi scope lebih luas dari yang diminta. Setelah perbaikan selesai, ulangi langkah 3–7 di Bagian 2 (update file list, status, isi ulang Verifikasi dari Gua tetap kosong, stop, lapor lagi).
- Jangan pernah menganggap "diam = setuju". Kalau user belum bilang apa-apa soal hasil task, jangan lanjut task berikutnya sendiri.

## 4. Batasan tambahan
- Jangan modifikasi `docs/PRD.md`, `docs/DESIGN.md`, `docs/SCHEMA.md`, `docs/RULES.md` kecuali user eksplisit minta — dokumen-dokumen itu adalah sumber kebenaran, bukan draft yang boleh AI edit sambil jalan.
- Kalau instruksi di satu task ternyata butuh keputusan yang belum ditentukan di dokumen manapun (bukan cuma detail kecil), berhenti dan tanya ke user — jangan mengarang keputusan arsitektur sendiri secara diam-diam.
- Selalu commit/laporkan di granularitas **per task**, bukan per beberapa task sekaligus digabung.

## 5. Tools yang Wajib Dipakai
Tools MCP/plugin berikut sudah terpasang di environment ini — **wajib dipakai setiap kali relevan**, bukan opsional:
- **Context7** — sebelum menulis kode yang menyangkut library/framework apapun (Next.js, Supabase SDK, Tailwind, Zod, React Hook Form, dll), cek dulu docs terbaru lewat Context7. Jangan asumsi API/syntax dari training data — banyak breaking changes yang mungkin belum ke-cover.
- **Superpowers** — ikuti metodologi brainstorm→plan→implement→review bawaan skill ini di setiap task, selaras sama Alur Kerja Bagian 2 di atas. Kalau Superpowers dan alur RULES.md ini kelihatan tumpang tindih, RULES.md yang menang untuk hal spesifik project ini (struktur folder, checklist TASK.md, dst) — Superpowers dipakai buat disiplin proses berpikirnya, bukan menggantikan aturan project.
- **Supabase MCP** & **Stitch MCP** — sudah disebut di Bagian 0 & 1, tetap wajib dipakai sesuai konteksnya (migration, ambil desain).

Prinsipnya: kalau ada tool yang bisa kasih informasi lebih akurat/update dibanding asumsi/ingatan sendiri, **selalu pakai tool itu dulu** sebelum nulis kode atau ambil keputusan.

## 6. Fleksibilitas terhadap Dokumen
Dokumen (`PRD.md`, `DESIGN.md`, `SCHEMA.md`, `RULES.md`, `TASK.md` ,`TASK.md`) adalah sumber kebenaran default, **bukan aturan mati yang harus diikuti buta** kalau ternyata ada yang kurang tepat:
- Kalau nemu instruksi di dokumen yang secara teknis **kurang optimal**, **outdated** (misal API/library yang direferensikan sudah berubah), atau ada **cara yang lebih best-practice**, jangan diam-diam ikutin yang salah — tapi juga jangan diam-diam ganti sendiri. **Informasikan ke user**: apa yang menurut lo kurang pas, kenapa, dan usulan perbaikannya — biar user yang putuskan mau ikut usulan itu atau tetap sesuai dokumen asli.
- Ini **beda** dari aturan Bagian 0 soal bentrok TASK.md vs RULES.md (itu spesifik soal dua dokumen saling bentrok satu sama lain). Bagian ini soal ketika **satu dokumen sendiri** kelihatan kurang tepat dibanding best practice terkini atau temuan dari tool (Context7, dst).
- Untuk hal kecil (naming, gaya penulisan, detail implementasi yang sama-sama valid) — boleh langsung sesuaikan ke yang lebih baik, sebutkan di laporan akhir task.
- Untuk hal yang berdampak besar (arsitektur, skema data, keamanan, alur bisnis inti) — **tetap wajib tanya dulu**, jangan asumsi user pasti setuju walau usulannya lebih baik secara teknis.
- Intinya: dokumen ini bukan buat diikuti kaku tanpa mikir, tapi jadi baseline yang boleh dan seharusnya dikoreksi kalau memang ada yang lebih baik — dengan tetap transparan ke user, bukan diam-diam menyimpang.