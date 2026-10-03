# STRUCTURE.md — Aturan Umum Struktur Kode

Aturan dasar penempatan file untuk project Next.js (App Router) + Supabase.
Prinsip utama: file yang cuma dipakai satu fitur ditaruh dekat fitur itu. File yang dipakai dua fitur atau lebih ditaruh di folder global.

## 1. Struktur Folder Dasar
src/
├── app/            # route: page, layout, actions, _components per fitur
├── components/
│   ├── ui/         # komponen shadcn (hasil CLI)
│   ├── common/     # komponen reusable lintas fitur
│   └── providers/  # React provider
├── config/         # akses env var
├── constants/      # nilai tetap
├── hooks/          # custom hook generik
├── lib/            # logic non-UI, client Supabase, integrasi pihak ketiga
├── types/          # type TypeScript lintas fitur
├── validations/    # skema Zod
└── migration/      # arsip SQL migration

## 2. Aturan Per Folder
### app/
- Satu fitur satu folder route. Di dalamnya ada `page.tsx`, `actions.ts`, dan `_components/`.
- `page.tsx` berupa Server Component: ambil data awal, lalu render komponen utama fitur.
- `actions.ts` berisi semua panggilan Supabase untuk fitur itu (query, insert, update, delete). Tidak boleh ada query Supabase langsung di komponen UI.
- `_components/` berisi komponen yang hanya dipakai fitur itu. Awalan `_` artinya folder privat, tidak jadi route.
- Sub-route (create, edit, detail) punya `_components/` sendiri.
- Gunakan route group `(nama)` untuk mengelompokkan halaman tanpa mengubah URL, misalnya `(customer)` dan `(menu)`.
- Halaman login admin ditaruh di luar route group yang punya guard auth, supaya tidak terjadi redirect loop.
- Guard auth dipasang di layout route group, bukan di tiap halaman.

### components/
- `ui/` hanya untuk komponen hasil CLI shadcn.
- `common/` hanya untuk komponen yang dipakai di dua fitur atau lebih.
- Kalau komponen cuma dipakai satu fitur, taruh di `_components/` fitur itu, bukan di `common/`.
- Satu komponen satu tanggung jawab. Jangan campur UI, fetch, dan validasi dalam satu file.

### config/
- Satu file untuk env var publik (`NEXT_PUBLIC_*`), satu file lain untuk env var rahasia yang diberi `import "server-only"`.
- Dilarang memanggil `process.env` langsung di file lain.
- Secret tidak boleh masuk file env publik.

### constants/
- Satu file per fitur dengan nama `<fitur>-constant.ts`.
- Isinya label, opsi dropdown, default form, initial state, limit, dan teks statis.
- Jangan hardcode angka, teks, atau status di dalam logic.

### hooks/
- Hanya hook generik yang dipakai lintas fitur, misalnya debounce dan state tabel.
- Hook yang cuma dipakai satu komponen ditulis di komponen itu.
- Nama file `use-<nama>.ts`.

### lib/
- Isinya logic non-UI: client Supabase (browser dan server), wrapper API pihak ketiga, util murni (format tanggal, format uang, slugify).
- Satu file satu domain.
- Tidak boleh ada JSX atau komponen di sini.
- Logic yang dipakai dua tempat atau lebih ditarik ke satu function di sini.

### types/
- Satu file per fitur dengan nama `<fitur>.ts`.
- Type yang cuma dipakai satu file boleh tetap lokal di file itu.

### validations/
- Skema Zod ditaruh di sini dengan nama `<fitur>-validation.ts`, lalu export type hasil `z.infer`.
- Skema tidak boleh ditulis di dalam komponen.

### migration/
- Arsip SQL migration dengan nama `NNN_deskripsi.sql`, urut nomor.
- Isinya ditarik dari database, bukan ditulis ulang dari ingatan.
- Migration lama tidak diubah. Koreksi dibuat sebagai migration baru.


## 3. Aturan Isi actions.ts
- Awali dengan `"use server"`.
- Server Action adalah endpoint publik. Validasi sendiri di dalam action, jangan mengandalkan layout guard.
- Action admin: cek user dan role dulu, baru boleh pakai client service role.
- Tabel yang tidak punya policy RLS hanya boleh diakses lewat service role di server, dengan validasi manual.
- Action publik cukup pakai client biasa dan tetap menulis filter eksplisit (misalnya status published) di query.
- Return bentuk konsisten, misalnya `{ data, error }`, atau state form untuk `useActionState`.
- Setiap error path wajib `console.error`. Jangan log password, token, atau secret.
- Pesan error ke user harus jelas, bukan stack trace mentah.
- `redirect()` ditaruh di luar try/catch, atau error redirect di-rethrow.
- Select kolom spesifik, bukan `select *`. Hindari N+1 dengan `Promise.all` atau nested select.
- List data wajib pakai pagination.


## 4. Aturan Komponen
- Server Component sebagai default. Tambah `"use client"` hanya kalau butuh state, effect, atau event.
- Ambil data awal di `page.tsx`. Interaksi seperti filter dan pagination ditangani di komponen client.
- Form pakai `react-hook-form`, `zodResolver`, dan `useActionState`.
- Styling hanya pakai token dari `globals.css`. Dilarang hardcode warna hex.
- Komponen yang memakai `useSearchParams` dibungkus `<Suspense>`.
- Setiap fetch punya state loading, success, dan error.

## 5. Penamaan
- File komponen: `kebab-case.tsx`, nama export PascalCase.
- File non-komponen: `kebab-case.ts`.
- Function dan variable: camelCase.
- Komponen, type, dan interface: PascalCase.
- Konstanta nilai: UPPER_SNAKE_CASE.

## 6. Dilarang
- Memanggil `process.env` langsung di luar `config/`.
- Menaruh secret di env publik atau bundle client.
- Menulis query Supabase di komponen UI.
- Menulis skema Zod di dalam komponen.
- Menaruh komponen atau JSX di `lib/`.
- Menduplikasi komponen atau util yang sudah ada. Reuse dulu.
- Mengubah migration lama.
- Memakai `any` tanpa komentar alasan.
- Membiarkan dead code (import, variable, atau komponen yang tidak dipakai).

## 7. Checklist Sebelum Bikin File Baru
1. Dipakai satu fitur atau lebih? Tentukan `_components/` atau `components/common/`.
2. Ada query Supabase? Taruh di `actions.ts` fitur itu.
3. Ada nilai tetap atau teks statis? Taruh di `constants/`.
4. Ada validasi form atau filter? Taruh di `validations/`.
5. Ada type baru yang dipakai lintas file? Taruh di `types/`.
6. Ada env var baru? Publik atau rahasia, taruh di file `config/` yang sesuai.
7. Ada util murni atau integrasi pihak ketiga? Taruh di `lib/`.
8. Sudah ada yang serupa? Reuse, jangan bikin ulang.