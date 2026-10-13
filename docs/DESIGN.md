# Design Guidelines — GitHub Radar

Tema: gelap pekat dengan gradien biru di bagian atas, kartu semi glass. Bersih, modern, fokus ke data. Dark-only (tidak ada mode terang).

Struktur dan layout mengikuti mockup Stitch. Warna, font, dan radius mengikuti dokumen ini. Kalau mockup dan dokumen ini berbeda, dokumen ini menang.

## 1. Palet Warna (`globals.css`)

Semua warna wajib jadi variabel CSS di `app/globals.css` dan dipanggil lewat semantic class shadcn. Dilarang hardcode warna atau pakai class warna default Tailwind (contoh `bg-blue-500`) di komponen. Blok `.dark` dan `prefers-color-scheme` bawaan scaffold dihapus; token di `:root` dengan `color-scheme: dark`.

### Warna dasar
- **Background:** `#0A0B0F` (`bg-background`)
- **Card dan Popover:** `rgba(255,255,255,0.04)` plus blur (`bg-card`, `bg-popover`)
- **Muted / Secondary:** `#14161C` (`bg-muted`, `bg-secondary`)
- **Accent (hover):** `#1B1E27` (`bg-accent`)
- **Star (amber):** `#FBBF24`, untuk ikon bintang dan angka star count.

### Garis dan fokus
- **Border:** `rgba(255,255,255,0.08)` (`border-border`)
- **Input border:** `rgba(255,255,255,0.14)` (`border-input`)
- **Focus ring:** `#6B93F5` (`ring-ring`). Wajib terlihat jelas di semua elemen interaktif (outline 2px, offset 2px).

### Teks
- **Utama:** `#F2F4F8` (`text-foreground`)
- **Kedua / muted:** `#8A90A0` (`text-muted-foreground`)

### Brand (biru)
- **Primary:** `#4F7DF3`, teks `#FFFFFF` (`bg-primary text-primary-foreground`)
- **Sky (terang):** `#A9C7FB`, untuk highlight, eyebrow penting, dan angka penting
- **Royal:** `#2B47B8`, untuk tingkatan garis grafik dan gradien
- **Navy (gelap):** `#0F1E6B`, untuk dasar gradien dan glow
- **Gradien brand:** `linear-gradient(180deg, #A9C7FB, #4F7DF3 55%, #0F1E6B)` sebagai utility `.bg-brand-gradient`

### Status
- **Success:** `#30A46C` (`text-success`, `bg-success/15`, `border-success/30`)
- **Warning:** `#F5A524`
- **Destructive:** `#E5484D`
- `success`, `warning`, `destructive` adalah token tambahan yang wajib didaftarkan di `@theme inline`.
- Grafik: naik = success, turun = destructive. Jangan andalkan warna saja: selalu sertakan tanda (+/−), ikon panah, atau teks.
- **Grafik multi-repo (peringkat 1 sampai 5):** `#F2F4F8`, `#A9C7FB`, `#6B93F5`, `#4F7DF3`, `#2B47B8` (token `--chart-1` sampai `--chart-5`). Dilarang pakai warna status atau warna lain untuk garis repo.

### Efek (utility di `globals.css`)

- **`.bg-header-gradient`:** gradien vertikal biru ke hitam di bagian atas halaman, tinggi sekitar 640px di desktop, ikut scroll (tidak fixed). Titik warna: 0% `#6F95F5`, 15% `#4F7DF3`, 35% `#2B47B8`, 52% `#0F1E6B`, 70% ke bawah `#0A0B0F`. Kabut putih tipis (opacity 15%) di pojok kanan atas. Overlay dot-grain `radial-gradient(rgba(255,255,255,0.12) 1px, transparent 1px)` tile 24x24px agar tidak banding. Overlay fade ke bawah `linear-gradient(to bottom, transparent 0%, rgba(10,11,15,0.2) 50%, #0A0B0F 100%)`. Di bawah gradien, halaman polos `bg-background`.
- **`.card-glass`:** background `rgba(10,11,15,0.45)`, border `1px solid rgba(255,255,255,0.12)`, `backdrop-filter: blur(20px)`, shadow `0 12px 32px rgba(0,0,0,0.5)` plus inset hairline `0 1px 0 rgba(255,255,255,0.04) inset`. Hover: background naik ke 0.55, border ke 0.20, kartu naik 2px. **Ini satu-satunya gaya kartu** untuk semua kartu (ringkasan, repo, grafik, AI summary, rilis, info, README, digest, dialog panel). Semua kartu harus terlihat seragam.
- **`.card-glass-dark`:** nilai sama dengan `.card-glass`. Dipakai untuk elemen di atas gradien header (kartu ringkasan, filter waktu).
- **`.card-glass-sm`:** varian blur 16px untuk elemen kecil (tooltip, chip glass). Dipakai tooltip shadcn.
- **`.card-glass-featured`:** background `rgba(255,255,255,0.05)`, border `rgba(169,199,251,0.35)`, glow `0 0 35px -5px rgba(79,125,243,0.22), 0 8px 24px rgba(0,0,0,0.4)`. Utility tetap didefinisikan, tetapi tidak dipakai saat ini (kartu unggulan sengaja diseragamkan dengan kartu lain).
- **`.nav-pill`:** background `rgba(10,11,15,0.35)`, border `rgba(255,255,255,0.15)`, `blur(24px)`, shadow `0 12px 32px rgba(0,0,0,0.5)`. Dipakai satu bar top bar.
- **`.tab-bar`:** background `rgba(10,11,15,0.45)`, border `rgba(255,255,255,0.12)`, `blur(20px)`, tanpa hover-lift. Dipakai tab bar mobile.
- **`.pill-active-shadow`:** shadow `0 4px 12px rgba(0,0,0,0.3)` untuk item aktif (pill putih) di nav.
- **Tier blur:** kartu utama 20px, elemen kecil 16px, top bar 24px.
- **Teks di atas gradien:** selalu putih `#F2F4F8` (sekunder 80% opacity), minimal 10px, kontras minimal 4.5:1. **Zona gradien 0 sampai sekitar 35% (kira-kira 224px pertama) kontrasnya kurang untuk teks polos**, jadi teks di zona itu harus berada di balik glass; teks polos baru aman di bawah zona itu.

### Radius
- `--radius: 1.25rem` (20px) untuk kartu, panel, dan dialog (`rounded-3xl` = 20px, atau `rounded-[20px]`).
- Tombol, input, tooltip, kotak ikon, badge velocity: `rounded-xl` (12px).
- **Pill nav, grup toggle (filter waktu), tab bar mobile, dan chip saran chat: `rounded-xl` (12px), bukan `rounded-full`.** Segmen aktif di dalam kontainer toggle memakai `rounded-lg` (8px) supaya sejajar dengan sudut luar (radius dalam = radius luar dikurangi padding) dan tidak tampak seperti pil bulat.
- Chat bubble `rounded-2xl` (16px).
- Chip bahasa/topik dan badge status `rounded-md` (6px).
- `rounded-full` hanya untuk elemen yang memang bulat: tombol panah di kartu repo, tombol kirim chat, dot indikator, dan pil "Rank #N overall" di kartu Velocity detail repo.

### Komponen shadcn (penyesuaian)
- Tinggi tombol dan input 44px (`h-11`, ikon `size-11`) untuk semua ukuran standar. Ukuran `xs` boleh kompak.
- Button, input, tooltip, badge: `rounded-xl`. Dialog: `rounded-3xl`.
- Panel dialog dan sheet (Riwayat chat, konfirmasi hapus, Add repo): `bg-background/95 backdrop-blur-xl border border-white/10` supaya cukup gelap dan terbaca.

## 2. Tipografi

- **Sans:** Geist Sans (`next/font/google`, variable font, tidak perlu menyebut weight per berat) untuk deskripsi, menu, tombol, judul.
- **Mono:** JetBrains Mono (`next/font/google`) untuk nama repo (`owner/name`), angka star dan fork, angka besar di kartu, eyebrow/label uppercase, chip bahasa/topik, badge velocity, caption angka (contoh `Showing 6 of 111`), dan teks tombol Load more.
- **Ikon:** Lucide React. Pengecualian: ikon "auto_awesome" (tiga bintang padat, untuk Ask AI dan AI summary) memakai SVG custom `AutoAwesomeIcon` (`fill="currentColor"`, satu komponen bersama dipakai navbar, header detail, dan kartu AI); ikon "travel_explore" di kartu Repos tracked juga SVG custom. Alasannya Lucide tidak punya padanan yang sama persis.
- Angka pakai `tabular-nums`. Ukuran teks minimal 10px (10px hanya badge kecil, 11px label chart dan eyebrow, normal 12px ke atas).

Skala teks:
- H1 halaman: `text-[28px] md:text-[36px] leading-[34px] md:leading-[44px] font-bold tracking-tight`
- H2 seksi: `text-xl md:text-2xl font-semibold tracking-tight`
- Judul kartu: `text-base font-semibold`
- Label / eyebrow: `font-mono text-[11px] font-semibold uppercase tracking-[0.2em] text-muted-foreground` (varian `tracking-wider`)
- Body: `text-sm md:text-base leading-relaxed text-muted-foreground`
- Angka besar: `font-mono text-[30px] font-semibold` (di kartu ringkasan Dashboard diperbesar, lihat bagian halaman)
- Caption: `text-xs text-muted-foreground`

## 3. Spacing dan Layout

- Skala basis 4px.
- **Padding halaman:** `px-4` mobile, `px-8` desktop. Padding atas konten `pt-24` mobile, `pt-[112px]` desktop. Padding bawah `pb-44` mobile (supaya tidak tertutup tab bar), `pb-28` desktop.
- **Padding kartu:** `p-4` mobile, `p-6` desktop.
- **Gap antar kartu:** `gap-4` mobile, `gap-6` desktop. Di Detail Repo semua jarak antar kartu **seragam 24px** walau tinggi kartu berbeda.
- **Jarak antar seksi:** `space-y-8` mobile, `space-y-12` desktop.
- **Lebar konten:** `max-w-[1536px] mx-auto`. Top bar memakai lebar yang sama supaya sejajar dengan konten.
- Grid Dashboard dan Watchlist: 1 kolom (mobile), 2 (tablet), 3 (desktop).
- Tanpa sidebar. Navigasi lewat top bar melayang, di mobile ditambah tab bar melayang di bawah.
- Tinggi tombol dan input 44px. Top bar 48px desktop dan 44px mobile.

## 4. Motion dan Interaksi

- Transisi 150 sampai 200ms (basis 180ms) untuk hover dan fokus. Kartu naik 2px saat hover.
- Fade-in sekitar 300ms untuk halaman dan grid. Tanpa parallax dan animasi berat.
- Hormati `prefers-reduced-motion`: matikan animasi non-esensial, termasuk pulse skeleton dan lift kartu.
- Skeleton: bar teks `rounded bg-muted animate-pulse`. Kartu skeleton memakai `.card-glass` dengan radius dan padding yang sama dengan kartu aslinya, dan **meniru layout halaman asli** (tidak ada layout yang loncat saat konten masuk).

## 5. Responsif dan Aksesibilitas

- **Mobile first:** desain dan uji mulai dari layar kecil.
- **Target sentuh:** minimal 44px di mobile.
- **Kontras:** teks terbaca jelas di atas latar gelap dan gradien. Jangan taruh teks penting di area terang tanpa cek kontras.
- **Status:** jangan andalkan warna saja, tambah ikon atau teks.
- Semua ikon atau gambar bermakna punya teks alternatif (`aria-label` atau `alt`). Tombol ikon wajib `aria-label`.
- Struktur heading berurutan dan semantik.

## 6. Spesifikasi per Halaman

### Shell (top bar dan tab bar)
- **Top bar:** satu bar melayang tunggal (`.nav-pill`, `rounded-[20px]`, `flex justify-between`): logo "GitHub Radar" (ikon GitHub SVG inline) di kiri, nav tengah (Dashboard, Watchlist, Chat, News) di desktop saja, tombol "Ask AI" di kanan. Hover per item `hover:bg-foreground/10`. Item aktif = pill putih (`bg-foreground text-background` plus `.pill-active-shadow`). Di mobile nav tengah disembunyikan (hanya logo dan Ask AI). Posisi `fixed` `top-4` mobile, `top-6` desktop, wrapper `pointer-events-none` dengan tiap elemen `pointer-events-auto`.
- Halaman `/repo/*` menandai "Dashboard" sebagai aktif.
- **Tab bar mobile:** hanya di bawah `md`, melayang, container `rounded-xl`, item `h-11` (44px). Item aktif = pill putih ikon plus label; item lain ikon saja dengan `aria-label`.
- Ikon nav: `LayoutDashboard`, `Bookmark`, `MessageCircle`, `BookOpen`.

### Dashboard
- **Baris atas:** teks info "Terakhir diperbarui N jam yang lalu • disegarkan setiap 6 jam" sebagai teks polos (`text-foreground/85`, ikon jam putih, bullet pemisah `foreground/40`), tanpa pil gelap. Filter waktu di kanan.
- **Filter waktu:** kontainer `bg-muted/80 border border-white/10 backdrop-blur-md rounded-xl p-1`. Segmen aktif `bg-foreground text-background font-semibold` dengan `rounded-lg`. Desktop: kontainer selebar isi, font sans 14px. Mobile: kontainer full-width, tiga segmen sama lebar (`flex-1`, pill aktif 1/3 kontainer, sekitar 113x28px), font mono 11px, tinggi kontainer sekitar 38px.
- **Tiga kartu ringkasan** (`.card-glass-dark`, `rounded-[20px]`, tanpa sparkline): baris eyebrow dengan kotak ikon 18px di kanan (hover scale), baris angka besar di bawah. Teks diperbesar responsif: eyebrow `12px` (`13px` di md), angka besar `32px` (`36px` di md), subline `14px` (`15px` di md), nama repo Fastest riser `text-xl` (22px di md). Subline Repos tracked dan Fastest riser berwarna sky; subline Stars gained mengikuti tanda persen (hijau + ikon naik, merah + ikon turun). Kartu Fastest riser memakai gaya kartu yang sama dengan dua lainnya.
- **Kartu Star velocity:** header satu baris (`border-b border-border pb-6`): judul dan subjudul "Comparative trajectory of top 5 repos" di kiri, legenda plain inline di kanan (dot `size-2.5` seragam, label repo; label pemimpin `text-foreground`, lainnya muted). Chart `h-64`, gridline horizontal putus-putus (`4 4`, warna `--border`), lima garis lebar seragam 2.5px dengan opasitas menurun 100/90/80/75/60 persen lewat `color-mix`, area gradient tipis di bawah garis pemimpin, crosshair putih putus-putus. Tooltip dan dot aktif mengikuti garis yang paling dekat dengan kursor (menampilkan satu repo, gain harian `+N stars`, dan badge persen). Label tanggal berupa overlay HTML di dasar kartu, posisinya dihitung dengan rumus skala titik Recharts (`i/(n-1)`) supaya tepat di tengah crosshair. Rentang sampai 8 hari ditampilkan penuh tanpa tanggal yang hilang; rentang lebih panjang disampling maksimal 7 label. Jarak plot ke label sekitar 30px, jarak bawah label 24px. Garis menggambar velocity per interval, bukan total kumulatif.
- **Header "RISING NOW":** baris `items-center`, judul putih `tracking-wider`, caption `- N repos` setelah judul, "Sorted by velocity" mono di kanan (disembunyikan di mobile). Jarak kartu velocity ke header sama dengan jarak header ke grid, yaitu 24px.
- **Kartu repo** (komponen bersama Dashboard dan Watchlist, `.card-glass`, `rounded-[20px]`, `p-4 md:p-6`, `flex flex-col justify-between gap-4`): baris atas berisi peringkat `#N` (mono 11px, muted; tidak ada di Watchlist), tombol simpan (bookmark, putih/amber bila tersimpan), dan label velocity di kanan (ikon tren plus `+N stars / day`, hijau atau merah). Judul `owner / name` mono, link ke detail, hover `text-primary`. Deskripsi `line-clamp-2`. Chip bahasa dan topik (`rounded-md border-border bg-muted`, mono 11px). Footer `border-t border-border pt-3`: kiri statistik (bintang amber, fork dengan ikon Lucide `Split`, mono 12px) dan baris "Created X ago" (11px); kanan tombol panah bulat `size-11` putih (`bg-foreground text-background`), hover menjadi biru `bg-primary` dengan ikon putih. Tanpa sparkline, tanpa varian featured.
- **Tombol Load more:** `<button>` polos `rounded-xl border-white/10 bg-white/5 px-6 h-11 shadow-lg shadow-black/40`, hover `bg-white/10`, `active:scale-95`, teks mono 12px, ikon `ChevronDown` 18px yang bergeser turun saat hover. Caption "Showing X of Y" di **bawah** tombol. Tombol berubah "Loading…" saat memuat dan hilang saat data habis.
- **Kartu progres setup (saat backfill):** kartu `.card-glass`, judul "Collecting star history" dan deskripsi statis; tiga langkah ber-ikon status (selesai `CheckCircle2`, aktif `RefreshCw` berputar, menunggu `Circle`), progress bar (`bg-border` dan `bg-primary`) dengan label mono "N of M repos".

### Detail Repo
- **Header:** link "Dashboard" kembali (putih, `font-medium`, ikon panah geser saat hover, hover biru). Baris meta: chip peringkat `#N` (`bg-primary/20 text-sky`), badge velocity, dan "Created X ago" berwarna putih. Judul H1 mono dua bobot (`owner` normal, `name` bold, dipisah `/`). Deskripsi `max-w-[640px]`. Chip topik. Tombol: **Ask AI** (primary, ikon `AutoAwesomeIcon` putih, shadow biru), **Add to Watchlist / In Watchlist** (tombol sekunder), **Open on GitHub** (sekunder, link luar `noopener noreferrer`). Di mobile tombol sekunder memakai `.card-glass`, di desktop `bg-white/[0.04]`. Repo `evicted` menampilkan chip "Tidak lagi dilacak".
- **Empat kartu statistik** (`rounded-[20px]`, `p-4 md:p-6`, mobile 2 kolom, desktop 4 kolom, gap 24px): Stars (ikon bintang amber), Forks (ikon `Split` putih), Open issues (ikon `CircleAlert` muted), Velocity (ikon `Zap` putih terisi, plus pil `Rank #N overall` dengan `rounded-full bg-white/[0.06] text-sky`). Sub-baris delta 7 hari dari snapshot dengan warna sesuai tanda (hijau atau merah); kalau riwayat kurang dari dua titik, sub-baris dihilangkan. Nilai kosong tampil "—".
- **Star history (30 hari):** eyebrow `30-DAY` (sky) dan judul, indikator "Live Sync" (dot primary berdenyut) di kanan. Garis putih dengan glow tipis, area gradient primary (0.35 ke 0), grid putus-putus, crosshair biru putus-putus, pill tooltip di tepi atas (`tanggal: +angka`), titik aktif berisi primary dengan cincin putih. Baris tanggal di bawah dengan garis pembatas `border-t border-white/5`: semua tanggal berurutan tanpa dilewati (seri sampai 12 titik memakai format `Sep 24`; seri lebih panjang memakai angka tanggal dan menampilkan bulan saat berganti bulan); label terakhir `text-primary font-semibold`; posisi label tepat di bawah titik datanya.
- **Weekly commits:** header satu baris (eyebrow `LAST 12 WEEKS` sky dan judul di kiri, "Avg. N commits / wk" mono di kanan). Bar `var(--primary)` dengan opasitas 0.7, lebar penuh kolom, sudut atas 6px, hover `--ring`, tanpa sorotan kolom kursor, tooltip pill. Tinggi `h-44 md:h-56`. Data NULL menampilkan empty state "Statistik belum tersedia".
- **AI summary:** `.card-glass`. Header ikon `AutoAwesomeIcon` dan judul, badge kanan (`CACHED - OCT 10`, `MEMBUAT…`, `MEMPERBARUI…`, `GAGAL`, atau `BELUM ADA`) plus tombol regenerate. Isi berupa poin markdown (dot primary, lead bold), garis pemisah, disclaimer italic Indonesia ("Analisis berdasarkan data yang dilacak, bukan kepastian penyebab."), dan tombol lebar penuh "Ask AI about this repo". Ada state generating (skeleton dan teks info), gagal (pesan aman dan arahan), dan kosong.
- **Latest release:** header dengan ikon `Tag` sky di kiri dan tanggal rilis mono di kanan, badge versi `rounded-[6px] border-primary/30 bg-muted text-primary`, catatan rilis markdown yang dipotong tinggi (`max-h-44`). Kalau belum ada rilis, tetap tampil dengan teks "Belum ada rilis di GitHub." dan link "Lihat rilis di GitHub".
- **Repo info:** header judul dan ikon `Info` 18px muted di kanan. Baris `divide-y divide-white/5` (`py-2.5`), label 13px muted, nilai mono 13px medium. Language dengan titik warna bahasa (warna ala linguist GitHub), License, Default branch sebagai badge (`bg-muted text-primary`), Last pushed.
- **README:** header ikon `FileText` sky dan judul, isi README **penuh tanpa dipotong**, HTML disanitasi, dan baris bawah `border-t border-white/5` berisi link "Read full README on GitHub". Isi README memakai body sans (`text-sm text-muted-foreground`).
- **Susunan desktop:** dua kolom independen (kiri 8 kolom: Star history, Weekly commits, README; kanan 4 kolom: AI summary, Latest release, Repo info), masing-masing `flex-col gap-6`. **Urutan mobile:** AI summary, Star history, Weekly commits, Latest release, Repo info, README.

### Watchlist
- Header: judul "Your watchlist" dan caption mono "N of 50 repos" di kiri, tombol "Add repo" di kanan (sekunder dengan ikon plus).
- Kotak cari tepat di bawah header, selebar kontainer: `card-glass-sm`, input transparan `h-11 rounded-xl`, ikon `Search` di kiri, tombol clear `X` saat ada teks, placeholder "Search by owner or repo name…". Empty state "No matching repos".
- Grid kartu repo yang sama dengan Dashboard. Empty state dengan tombol Add repo.
- **Dialog Add repo:** judul, tombol tutup bulat, input cari (debounce 500ms), daftar hasil (`owner / name` dengan nama bold, deskripsi terpotong, chip bahasa, bintang amber, tombol "+ Add" atau penanda "Watching"), footer mono "N of 50 repos". Di mobile tampil sebagai sheet dari bawah. Ada state loading, kosong, error, dan hint.

### Chat
- Judul tengah "Ask about repos and trends". Pill "Konteks: owner/name" (mono) kalau sesi punya repo.
- **Layout desktop:** sidebar kiri (label "CHATS", tombol "New chat", daftar sesi dengan judul dan waktu relatif, tombol "Bersihkan riwayat" di bawah) dan kolom chat penuh di kanan, rapat dengan sidebar (gap 24px), sejajar dengan lebar top bar. **Hanya daftar pesan yang scroll**; sidebar dan composer tetap. Di mobile, sidebar menjadi sheet "Riwayat".
- **Bubble:** pesan user di kanan (`rounded-2xl`, biru gelap); jawaban AI berupa kartu `.card-glass` lebar penuh tanpa avatar atau ikon bintang. Di bawah jawaban ada baris "SOURCES:" berisi chip repo (ikon database, mono) yang menaut ke Detail Repo.
- **Aksi pesan:** timestamp relatif, tombol Retry dan Copy (ikon `RefreshCw` dan `Copy` yang berubah jadi `Check` hijau). Muncul saat hover di desktop dan selalu terlihat di mobile.
- **Composer:** kapsul `.card-glass` dengan input dan tombol kirim bulat biru; saat streaming tombol berubah jadi Stop. Chip saran pertanyaan hanya tampil saat percakapan masih kosong. Di mobile, header menjadi satu baris tipis saat sudah ada pesan supaya area pesan lebih luas, dan area pesan diberi jarak `pr-3` dari scrollbar.
- Catatan kecil di bawah: "Jawaban adalah analisis dari data, bukan kepastian penyebab."

### Digest (halaman News)
- Judul "What's new today" dan subjudul "Ringkasan berita harian yang ditulis AI dari repo yang kamu pantau." Konten selebar top bar.
- Entri terbaru tampil terbuka di `.card-glass` dengan label "Latest" dan tombol lipat di kanan; entri lama terlipat (tanggal sky mono, judul, meta "N trending · M watchlist", chevron). Tidak ada titik timeline.
- Isi entri: tanggal (`OCT 10, 2026`), judul buletin, lede, chip tag, lalu dua panel berdampingan di desktop (satu kolom di mobile): **TRENDING NOW** (hingga 10 repo: peringkat, `owner/name` link, velocity hijau `+N stars / day`, blurb AI, deskripsi asli muted, chip bahasa/topik) dan **WATCHLIST UPDATES** (nama repo link, badge `COMMITS` atau `RELEASE`, headline tebal, ringkasan AI, baris meta "N commits" atau tombol rilis; daftar commit mentah tidak ditampilkan). Tiap panel punya header yang bisa di-hide atau di-show dengan jumlah item di sebelahnya.
- Footer entri: "Generated at HH:MM UTC · 1 AI call".
- Bawah daftar: tombol "Load more" dan caption "Showing X of Y". Empty state "No digest yet".
