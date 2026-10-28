# NEXA — BEYOND TOMORROW

Sistem operasi digital futuristik dalam bentuk website: command center, AI Assistant, Smart Tools, Security Center, Future Lab, workspace pribadi, dan Admin Panel.

- **Tanpa build tools.** HTML + CSS + JavaScript modular (ES modules). Edit file → deploy.
- **Backend:** serverless functions Vercel di folder `api/` (AI, admin, konfigurasi, log error).
- **Cloud (opsional):** Supabase untuk login + database dengan Row Level Security.
- **Berjalan tanpa konfigurasi apa pun** dalam **demo mode** (data di `localStorage`, AI berupa template lokal yang berlabel jelas).

## Apa yang aktif, dan kapan

| Fitur | Tanpa konfigurasi | Setelah dikonfigurasi |
|---|---|---|
| Overview, Workspace, Smart Tools, Security Center, Future Lab | ✅ berfungsi penuh (lokal) | sama |
| Penyimpanan | Demo: `localStorage` (hanya di perangkat itu) | Cloud: Supabase, dipisah per akun (RLS) |
| AI Assistant | **DEMO MODE** (template lokal, bukan AI) | AI sungguhan lewat `/api/ai` bila `ANTHROPIC_API_KEY` diisi |
| Login / akun | tidak tersedia | Supabase Auth |
| Admin Panel | tidak tersedia | butuh Supabase + service role key + akun ber-role `admin` |

NEXA **tidak** mengklaim cloud/AI aktif sebelum benar-benar terkonfigurasi. Security Center hanya melakukan pemeriksaan lokal (format URL, kekuatan password, checklist buatan Anda); ia **tidak** memindai perangkat, jaringan, atau akun. Future Lab adalah materi edukasi; tidak ada kontrol perangkat pintar/VR.

## Struktur

```
index.html            aplikasi utama (SPA, router berbasis #hash)
admin.html            Admin Panel (halaman terpisah, /admin)
manifest.webmanifest  + icons/   (bisa "Tambahkan ke layar utama")
css/style.css
js/
  main.js  util.js  store.js  cloud.js  focus.js  search.js
  passgen.js  urlcheck.js  planet.js  labdata.js  admin.js
  pages/  overview workspace assistant tools security lab settings
api/                  serverless functions (Vercel)
  config.js           konfigurasi publik (tanpa secret)
  ai.js               proxy AI + streaming (API key hanya di server)
  admin.js            semua operasi admin, diverifikasi di server
  log.js              laporan error klien (opt-in)
  _lib.js             util bersama
supabase/schema.sql   tabel + RLS + trigger (jalankan sekali)
tests/api.test.mjs    tes tanpa dependensi (npm test)
dev-server.mjs        server lokal tanpa dependensi (npm run dev)
```

## Jalankan lokal (opsional)

```bash
npm run dev        # http://localhost:3000  (membaca .env bila ada)
npm test           # 23 tes: URL checker, password, kalkulator, kontrol akses API
```

Server statis biasa (mis. `python3 -m http.server`) juga bisa untuk demo mode, tetapi `/api/*` hanya tersedia lewat `npm run dev` atau Vercel.

## Deploy ke Vercel (cukup dari HP Android)

1. **GitHub:** buat repositori (privat disarankan) di aplikasi/situs GitHub, lalu unggah seluruh isi folder ini (di HP: GitHub web → *Add file → Upload files*, atau aplikasi Termux/git). `.env` jangan ikut diunggah (sudah di `.gitignore`).
2. **Vercel:** buka vercel.com → *Add New → Project* → impor repositori. *Framework Preset* = **Other**, tanpa build command, output directory dikosongkan. Klik **Deploy**.
3. Situs langsung jalan dalam **demo mode**. Lanjut ke langkah berikut untuk cloud/AI/admin.
4. Setiap Anda mengubah file di GitHub, Vercel otomatis deploy ulang. Perubahan **Environment Variables** baru berlaku setelah *Redeploy*.

## Mengaktifkan Cloud mode (Supabase)

1. Buat project di supabase.com.
2. **SQL Editor → New query**: tempel seluruh isi `supabase/schema.sql` → **Run**. (Aman dijalankan ulang.) Ini membuat tabel, constraint validasi, dan kebijakan RLS.
3. **Project Settings → API**: salin *Project URL*, kunci **anon public**, dan **service_role**.
4. Di Vercel → *Project → Settings → Environment Variables*, tambahkan:

   | Nama | Isi | Catatan |
   |---|---|---|
   | `SUPABASE_URL` | Project URL | publik |
   | `SUPABASE_ANON_KEY` | kunci anon | publik, dilindungi RLS |
   | `SUPABASE_SERVICE_ROLE_KEY` | kunci service_role | **RAHASIA**, hanya dibaca fungsi server |

5. **Redeploy**. Di *Settings → Akun & Cloud* sekarang muncul form Masuk/Daftar. Lencana di pojok kanan atas berubah dari **DEMO · LOKAL** menjadi **CLOUD** hanya setelah Anda login.
6. *(Disarankan)* Supabase → *Authentication → URL Configuration*: isi **Site URL** dengan domain Vercel Anda agar tautan konfirmasi email/reset password benar.

### Keamanan data cloud
- Semua tabel data (`tasks`, `notes`, `goals`, `events`, `links`, `chats`) memakai RLS: `user_id = auth.uid()`. Pengguna hanya dapat membaca/ubah miliknya.
- Validasi dilakukan di klien **dan** di database (CHECK constraint, panjang, format, URL http/https).
- Pengguna tidak dapat mengubah `role` miliknya (trigger `protect_profile_columns`).
- Tabel admin (`app_settings`, `audit_log`, `error_log`, `usage_log`) tanpa kebijakan RLS untuk anon/authenticated → hanya `service_role` (server) yang dapat mengakses.

## Menjadikan akun Anda administrator

Tidak ada password admin di source code. Admin = akun Supabase yang kolom `profiles.role`-nya `admin`.

1. Daftar lewat aplikasi (Settings → Akun & Cloud) dan konfirmasi email.
2. Supabase → SQL Editor, jalankan (ganti emailnya):
   ```sql
   update public.profiles set role = 'admin'
   where id = (select id from auth.users where email = 'email-anda@contoh.com');
   ```
3. Login ulang, lalu buka **/admin** (atau menu *Admin Panel* yang muncul di sidebar). Tombol itu hanya kosmetik: **setiap permintaan `/api/admin` diperiksa ulang di server** (JWT ke Supabase Auth + role dari tabel `profiles`). Akun biasa yang memanggil API langsung mendapat `403`.

Admin Panel: ringkasan penggunaan, pengelolaan pengguna (jadikan admin/user, nonaktifkan/aktifkan), pengelolaan konten Future Lab, pengumuman & saklar fitur (AI, mode pemeliharaan), audit log (100 tindakan terakhir), status layanan, dan error log.

## Mengaktifkan AI sungguhan

Tambahkan di Vercel (lalu Redeploy):

| Nama | Isi |
|---|---|
| `ANTHROPIC_API_KEY` | API key Anda (**RAHASIA**, tidak pernah dikirim ke browser) |
| `AI_MODEL` | opsional, default `claude-haiku-5-5` |
| `AI_REQUIRE_LOGIN` | default `true`: hanya pengguna login yang boleh memakai AI (mencegah orang asing menghabiskan biaya Anda). Set `false` hanya bila Anda siap menanggung risiko biaya. |

- Respons di-*stream* dari `/api/ai`. Ada pembatas laju per pengguna/IP (best-effort) dan batas panjang pesan.
- Admin dapat mematikan AI kapan saja dari Admin Panel.
- Tanpa key (atau tanpa login bila wajib), Assistant tetap berjalan sebagai **DEMO MODE** dengan label jelas dan tidak berpura-pura sebagai AI.
- **Privasi:** teks yang Anda kirim ke Assistant diteruskan ke penyedia AI hanya saat Anda menekan Kirim.

## Pengembangan dari HP Android

- Edit langsung di GitHub (tombol ✏️ pada file) atau editor seperti Acode/Termux; commit → Vercel deploy otomatis.
- Konten Future Lab dapat diubah dari **Admin Panel** tanpa menyentuh kode (setelah cloud aktif).
- Tidak ada langkah build, jadi tidak ada yang perlu diinstal.

## Aksesibilitas & performa

- Kontras tinggi, fokus terlihat, tautan "lewati ke konten", label ARIA, dan `prefers-reduced-motion` dihormati.
- **Mode hemat** otomatis aktif (bisa diatur di Settings) bila perangkat meminta reduced motion, RAM ≤ 2 GB, atau CPU ≤ 2 inti: bintang & animasi planet dikurangi.
- Tanpa webfont dan tanpa framework; halaman dimuat per-modul sesuai kebutuhan.
- Library eksternal hanya dimuat saat dibutuhkan: `supabase-js` (saat cloud dikonfigurasi) dan pembuat QR (saat tab QR dipakai), dari CDN dengan versi dikunci.

## Batasan yang jujur

- Demo mode menyimpan data **hanya di browser itu**; hapus data situs = data hilang. Gunakan *Ekspor* di Settings.
- Pembatas laju di fungsi serverless bersifat best-effort (memori per instance). Untuk trafik besar tambahkan pembatas di sisi platform.
- Konten Future Lab berstatus *draf* tidak menyembunyikan topik bawaan; ia hanya menahan konten tambahan/penimpa agar tidak tampil.
- Pemeriksa URL membaca pola penulisan saja; bukan pemindai reputasi dan tidak menjamin sebuah situs aman.
