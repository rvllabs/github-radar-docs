# Design Guidelines - GitHub Radar (V1)

Tema: gelap pekat dengan gradien biru di bagian atas, kartu semi glass. Bersih, modern, fokus ke data.

## 1. Color Palette (globals.css)
Semua warna wajib jadi variabel CSS di `app/globals.css` dan dipanggil lewat semantic class shadcn. Dilarang hardcode warna atau pakai class warna default Tailwind (contoh: `bg-blue-500`) di komponen.

### Warna Dasar
- **Background:** `#0A0B0F` (`bg-background`)
- **Card & Popover:** `rgba(255,255,255,0.04)` + blur, lihat bagian Efek (`bg-card`, `bg-popover`)
- **Muted / Secondary:** `#14161C` (`bg-muted`, `bg-secondary`)
- **Accent (Hover):** `#1B1E27` (`bg-accent`)

### Garis & Fokus
- **Border:** `rgba(255,255,255,0.08)` (`border-border`)
- **Input Border:** `rgba(255,255,255,0.14)` (`border-input`)
- **Focus Ring:** `#6B93F5` (`ring-ring`). Wajib terlihat jelas di semua elemen interaktif.

### Teks
- **Utama:** `#F2F4F8` (`text-foreground`)
- **Kedua / Muted:** `#8A90A0` (`text-muted-foreground`)

### Brand (Biru)
- **Primary:** `#4F7DF3`, teks `#FFFFFF` (`bg-primary text-primary-foreground hover:bg-primary/90`)
- **Sky (terang):** `#A9C7FB`, untuk highlight dan angka penting
- **Royal:** `#2B47B8`, untuk tingkatan garis grafik dan gradien
- **Navy (gelap):** `#0F1E6B`, untuk dasar gradien dan glow
- **Gradien brand:** `linear-gradient(180deg, #A9C7FB, #4F7DF3 55%, #0F1E6B)`, dibuat sebagai utility `.bg-brand-gradient`

### Status
- **Success:** `#30A46C` (`text-success`, `bg-success/15`, `border-success/30`)
- **Warning:** `#F5A524` (`text-warning`, `bg-warning/15`, `border-warning/30`)
- **Destructive:** `#E5484D` (`text-destructive`, `bg-destructive/15`, `border-destructive/30`)
- `success` dan `warning` adalah token tambahan, wajib didaftarkan di konfigurasi Tailwind.
- Grafik: naik = success, turun = destructive, garis utama = primary.
- Grafik multi-repo (peringkat 1 sampai 5): `#F2F4F8`, `#A9C7FB`, `#6B93F5`, `#4F7DF3`, `#2B47B8`. Dilarang pakai warna status atau warna lain untuk garis repo.

### Efek
- **Gradien header** (utility `.bg-header-gradient`): gradien vertikal biru ke hitam di bagian atas halaman, tinggi sekitar 640px di desktop, ikut scroll (tidak fixed). Titik warna: 0% `#6F95F5`, 15% `#4F7DF3`, 35% `#2B47B8`, 52% `#0F1E6B`, 70% ke bawah `#0A0B0F`. Putih hanya kabut tipis di pojok kanan atas (opacity 15%). Tambah grain 3% agar tidak banding. Di bawah gradien, halaman polos `bg-background`.
- **Kartu semi glass** (utility `.card-glass`): background `rgba(10,11,15,0.45)`, border `1px solid rgba(255,255,255,0.12)`, `backdrop-filter: blur(20px)`, shadow `0 12px 32px rgba(0,0,0,0.5)`. Hover: background naik ke 0.55 dan border ke 0.20. Dipakai untuk semua kartu, pill, grup toggle, tombol sekunder, dan tooltip.
- **Glass gelap** (utility `.card-glass-dark`): background `rgba(10,11,15,0.45)`, border `1px solid rgba(255,255,255,0.12)`, `backdrop-filter: blur(20px)`. Dipakai untuk elemen di atas gradien header (nav pill, filter waktu, kartu ringkasan, tooltip).
- **Shadow kartu:** `0 10px 30px rgba(0,0,0,0.4)`
- **Teks di atas gradien:** selalu putih `#F2F4F8` (sekunder 80% opacity), minimal 12px, kontras minimal 4.5:1.

### Radius
- `--radius: 1.25rem` (20px) untuk kartu, panel, dan dialog.
- Tombol, input, tooltip, kotak icon, chip peringkat, dan badge velocity `rounded-xl` (12px).
- Chat bubble `rounded-2xl` (16px).
- Chip bahasa/topik dan badge status `rounded-md` (6px).
- Pill nav, toggle, tab bar, dan tombol ikon bulat `rounded-full`.

## 2. Typography
- **Typeface:** **Geist Sans** (via `next/font/google`, weight 400/500/600/700) untuk deskripsi repo, teks menu, tombol, dan judul grafik.
- **Font Mono:** **Geist Mono** untuk nama repo (`owner/name`), angka star dan fork, angka besar di kartu ringkasan, eyebrow/label uppercase (contoh `REPOS TRACKED`, `RISING NOW`), chip bahasa/topik, badge velocity, dan caption angka (contoh `Showing 6 of 87`).
- **Icons:** **Lucide React**.
- **Aturan:** angka pakai `tabular-nums`. Ukuran teks minimal 12px.

H1 halaman `text-3xl md:text-5xl font-bold tracking-tight`
H2 seksi `text-xl md:text-2xl font-semibold tracking-tight`
Judul kartu `text-base font-semibold`
Label / Eyebrow `font-mono text-xs font-medium uppercase tracking-widest text-muted-foreground`
Body `text-sm md:text-base leading-relaxed text-muted-foreground`
Angka besar `font-mono text-3xl font-bold tabular-nums`
Caption `text-xs text-muted-foreground`

## 3. Spacing & Layout
- Skala basis 4px (skala bawaan Tailwind).
- **Padding halaman:** `px-4` mobile, `px-8` desktop.
- **Padding kartu:** `p-4` mobile, `p-6` desktop.
- **Gap antar kartu:** `gap-4` mobile, `gap-6` desktop.
- **Jarak antar seksi:** `space-y-8` mobile, `space-y-12` desktop.
- **Jarak dalam kartu:** `space-y-3` sampai `space-y-4`.
- Konten maksimal `max-w-6xl mx-auto`. Grid dashboard: 1 kolom (mobile), 2 (tablet), 3 (desktop).
- Tanpa sidebar. Navigasi lewat top bar melayang, di mobile pindah ke tab bar melayang di bawah.
- Tinggi tombol dan input `h-11` (44px).

## 4. Motion & Interaksi
- Transisi 150-200ms untuk hover dan fokus. Kartu naik 2px saat hover.
- Fade-in sekitar 300ms untuk halaman dan grid. Tanpa parallax dan animasi berat.
- Hormati `prefers-reduced-motion`: matikan animasi non-esensial.

## 5. Responsif & Aksesibilitas
- **Mobile first:** desain dan uji mulai dari layar kecil.
- **Target sentuh:** minimal 44px di mobile.
- **Kontras:** teks harus terbaca jelas di atas latar gelap dan di atas gradien header. Jangan taruh teks penting di atas area terang tanpa cek kontras.
- **Status:** jangan andalkan warna saja, tambah ikon atau teks.