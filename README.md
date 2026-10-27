# Lensora — Setiap Foto Punya Cerita

Aplikasi komunitas fotografi: member mengunggah foto beserta cerita di baliknya, mengumpulkan poin, naik level otomatis, memberi rating bintang, berkomentar, dan menukarkan poin dengan hadiah.

- **Backend**: Laravel 12 (REST API) + JWT auth + Spatie Permission (RBAC dinamis)
- **Frontend**: Next.js 16 (App Router, Server Components, BFF proxy)
- **Database**: MySQL 8+ (utama) / SQLite (opsional untuk demo cepat)

---

## Daftar Isi

1. [Daftar Kasus](#daftar-kasus)
2. [Fitur](#fitur)
3. [Tech Stack](#tech-stack)
4. [Arsitektur](#arsitektur)
5. [Struktur Proyek](#struktur-proyek)
6. [Persyaratan](#persyaratan)
7. [Instalasi](#instalasi)
8. [Akun Demo](#akun-demo)
9. [Aturan Bisnis](#aturan-bisnis)
10. [RBAC & Permission](#rbac--permission)
11. [API Reference](#api-reference)
12. [Autentikasi JWT di Frontend](#autentikasi-jwt-di-frontend)
13. [Testing](#testing)
14. [Troubleshooting](#troubleshooting)
15. [Batasan & Rencana Lanjutan](#batasan--rencana-lanjutan)

---

## Daftar Kasus

Kasus-kasus yang menjadi dasar aplikasi ini:

1. **Upload foto bercerita** — Member mengunggah foto (JPG/PNG/WebP, maks 5 MB) wajib menyertakan cerita minimal 20 karakter.
2. **Poin otomatis** — Setiap foto yang berhasil dipublikasikan memberi **100 poin** dan dicatat di buku besar (`point_transactions`).
3. **Level otomatis** — Total poin seumur hidup menentukan level: **Bronze (0) → Silver (1.000) → Gold (5.000) → Platinum (10.000)**. Level naik tanpa aksi manual.
4. **Penukaran hadiah** — Member yang sudah mengumpulkan minimal **10.000 poin** dapat menukar hadiah. Poin dipotong, penukaran berstatus `pending` sampai diproses admin.
5. **Rating bintang** — Hanya member yang **sudah login** yang bisa memberi rating 1–5 bintang pada foto milik orang lain. Satu user satu rating per foto (bisa diubah), dan rating bisa dihapus.
6. **Komentar** — Hanya member yang **sudah login** yang bisa berkomentar. Komentar bisa diedit pemiliknya dan dihapus oleh pemilik atau moderator/admin.
7. **RBAC dinamis** — Admin dapat membuat role baru dan memilih permission-nya langsung dari panel admin, tanpa mengubah kode atau deploy ulang.
8. **Pencabutan poin saat foto dihapus** — Menghapus foto mencabut 100 poin dan menurunkan `lifetime_points`, sehingga level ikut turun bila melewati ambang. Ini mencegah farming level.
9. **Penolakan penukaran** — Jika admin menolak penukaran, poin dikembalikan penuh dan stok hadiah dipulihkan.
10. **JWT auth** — Register/login mengembalikan JWT. Frontend menyimpannya di cookie `httpOnly` lewat BFF, bukan di localStorage.
11. **Rate limiting** — Login/register dibatasi 5 request/menit per email+IP; endpoint API 120 request/menit.
12. **Proteksi data** — Email hanya terlihat oleh pemilik akun dan admin. Foto berstatus `hidden` hanya terlihat oleh pemilik/admin.

---

## Fitur

### Member
- Registrasi & login (JWT)
- Unggah foto + cerita + lokasi + tanggal foto
- Dasbor pribadi: saldo poin, total poin, progres level, daftar foto, riwayat poin, status penukaran
- Galeri publik dengan pencarian, filter sorotan, dan urutan terbaru/populer
- Halaman profil publik (`/u/username`)
- Papan peringkat berdasarkan total poin
- Rating bintang interaktif
- Komentar (kirim, edit milik sendiri, hapus milik sendiri)
- Penukaran hadiah dengan validasi saldo minimal

### Admin / Moderator
- Dashboard ringkasan (pengguna, foto, poin, penukaran)
- Manajemen pengguna + pengaturan role
- Manajemen role & permission secara dinamis (create/update/delete)
- Manajemen hadiah (create, aktif/nonaktif, hapus, stok)
- Manajemen penukaran (setujui, selesai, tolak + refund otomatis)
- Moderasi foto (feature/unfeature, publish/hide, hapus)
- Moderasi komentar

### Teknis
- REST API terdokumentasi dan teruji (27 feature test)
- BFF Next.js: token JWT tidak pernah menyentuh JavaScript browser
- Progres level dihitung dari `lifetime_points` (penukaran hadiah tidak menurunkan level)
- Buku besar poin (`point_transactions`) untuk audit saldo
- Seed data demo realistis (5 akun, 10 foto, rating, komentar, 1 penukaran)

---

## Tech Stack

| Layer | Teknologi |
|---|---|
| Backend | PHP 8.2, Laravel 12.70.1 |
| Auth | `php-open-source-saver/jwt-auth` v2.8.3 |
| RBAC | `spatie/laravel-permission` v6 |
| Database | MySQL 8+ (utama) / SQLite (opsional) |
| Storage | Laravel public disk (`storage/app/public`) |
| Frontend | Next.js 16.4.3, React 19.3.0, TypeScript 5 |
| Styling | Tailwind CSS v4 |
| Font | Fraunces (display) + Geist (body) |

---

## Arsitektur

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│        Browser (React)      │        │        Laravel API           │
│  Server Components + Client │        │  /api/*  (JWT guard: api)    │
└──────────────┬──────────────┘        │  Spatie roles & permissions  │
               │                        │  MySQL (utama) / SQLite      │
               │ fetch (same-origin)    │  storage/app/public          │
┌──────────────▼──────────────┐        └──────────────▲───────────────┘
│        Next.js BFF          │   Authorization:      │
│  /api/auth/login|register   │   Bearer <jwt>        │
│  /api/auth/logout|me        │───────────────────────┘
│  /api/proxy/[...path]       │
│  Cookie httpOnly:           │
│  lensora_token              │
└─────────────────────────────┘
```

**Alur login**

1. Form login memanggil `POST /api/auth/login` (Next.js route handler).
2. Route handler meneruskan kredensial ke Laravel `POST /api/auth/login`.
3. Laravel mengembalikan `access_token` (JWT).
4. Next.js menyimpan token di cookie `httpOnly`, `SameSite=Lax`, `Secure` di production.
5. Server Component membaca cookie dan memanggil Laravel dengan header `Authorization: Bearer <token>`.
6. Client Component memanggil `/api/proxy/*`; route handler menambahkan header Authorization.

Konsekuensi: token tidak bisa dibaca JavaScript, sehingga aman dari XSS pencurian token.

---

## Struktur Proyek

```
photography/
├── README.md
├── backend/                     # Laravel 12 REST API
│   ├── app/
│   │   ├── Http/
│   │   │   ├── Controllers/     # Auth, Photo, Comment, Rating, Reward, Admin/*
│   │   │   ├── Requests/        # Validasi form request
│   │   │   └── Resources/       # API resource (User, Photo, Comment, dst.)
│   │   ├── Models/              # User, Photo, Comment, Rating, Level, Reward, Redemption
│   │   ├── Policies/            # PhotoPolicy, CommentPolicy
│   │   └── Services/            # PointService, LevelService, PhotoService, RewardService
│   ├── config/photography.php   # photo_points, min_points_to_redeem, max upload
│   ├── database/
│   │   ├── migrations/
│   │   └── seeders/             # Level, RolePermission, Reward, DemoContent
│   ├── routes/api.php
│   └── tests/Feature/           # Auth, PhotoPoint, RatingComment, Rbac, Redemption
└── frontend/                    # Next.js 16 App Router
    ├── src/app/
    │   ├── page.tsx             # Beranda
    │   ├── photos/[id]/         # Detail foto + rating + komentar
    │   ├── dashboard/           # Dasbor member
    │   ├── upload/              # Form unggah
    │   ├── rewards/             # Katalog hadiah
    │   ├── leaderboard/         # Peringkat
    │   ├── u/[username]/        # Profil publik
    │   ├── admin/               # Panel admin
    │   └── api/                 # BFF: auth + proxy
    ├── src/components/          # PhotoCard, RatingStars, CommentSection, dll.
    └── src/lib/                 # api.ts, auth.ts, types.ts, format.ts
```

---

## Persyaratan

- PHP **8.2+** dengan ekstensi `pdo_mysql` (utama) atau `pdo_sqlite` (opsional), `gd`, `fileinfo`
- Composer **2.x**
- Node.js **20+** (disarankan 22/24) dan npm
- MySQL **8.0+** (server berjalan di port 3306; MAMP biasanya 8889)

---

## Instalasi

### 1. Backend (Laravel)

```bash
cd backend

# Dependensi
composer install

# Environment
cp .env.example .env
php artisan key:generate

# JWT secret (WAJIB)
php artisan jwt:secret

# Pastikan nilai berikut ada di .env
# APP_URL=http://localhost:8000
# AUTH_GUARD=api
# FILESYSTEM_DISK=public
# FRONTEND_URL=http://localhost:3000
# JWT_TTL=10080
# DB_CONNECTION=mysql
# DB_HOST=127.0.0.1
# DB_PORT=3306
# DB_DATABASE=lensora
# DB_USERNAME=root
# DB_PASSWORD=

# Nyalakan MySQL terlebih dahulu (pilih salah satu)
mysql.server start          # Homebrew
# atau buka aplikasi MAMP lalu klik Start Servers

# Buat database MySQL (sesuaikan kredensial)
mysql -uroot -e "CREATE DATABASE lensora CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Migrasi + data demo
php artisan migrate:fresh --seed

# Symlink untuk gambar publik
php artisan storage:link

# Jalankan API
php artisan serve --port=8000
```

API berjalan di `http://localhost:8000/api`.

### 2. Frontend (Next.js)

```bash
cd frontend

npm install

cp .env.example .env.local
# Isi:
# API_URL=http://localhost:8000/api

npm run dev
```

Aplikasi berjalan di `http://localhost:3000`.

> Jalankan backend dan frontend di dua terminal terpisah. Frontend membutuhkan backend aktif karena semua data diambil lewat API.

> **Catatan port:** jika `8000` sudah dipakai proses lain, jalankan backend di `8001` (`php artisan serve --port=8001`), lalu samakan `APP_URL` (backend) dan `API_URL` (frontend) ke `http://localhost:8001`. Setup lokal proyek ini memakai port `8001` karena `8000` dipakai layanan lain.

### 3. Opsi SQLite (cepat, tanpa server database)

Jika ingin menjalankan tanpa MySQL:

```dotenv
DB_CONNECTION=sqlite
# hapus/komentari DB_HOST, DB_PORT, DB_DATABASE, DB_USERNAME, DB_PASSWORD
```

```bash
touch database/database.sqlite
php artisan migrate:fresh --seed
```

SQLite cocok untuk demo cepat, tetapi MySQL disarankan untuk penggunaan nyata.

### 4. Build production (opsional)

```bash
cd backend && php artisan optimize
cd ../frontend && npm run build && npm run start
```

---

## Akun Demo

Semua akun memakai password: **`password`**

| Email | Role | Level | Keterangan |
|---|---|---|---|
| `admin@lensora.test` | admin | Bronze | Akses penuh panel admin |
| `rani@lensora.test` | user | Platinum | 10.200 poin total (200 tersisa), 4 foto, 1 penukaran pending |
| `budi@lensora.test` | user | Gold | 5.400 poin, 3 foto |
| `sari@lensora.test` | user | Silver | 1.400 poin, 2 foto |
| `dimas@lensora.test` | user | Bronze | 300 poin, 1 foto |

Data demo mencakup foto contoh (gambar digenerate via GD), rating, dan komentar sehingga beranda langsung terlihat hidup.

---

## Aturan Bisnis

### Poin

| Aksi | Poin | Catatan |
|---|---|---|
| Unggah foto + cerita | **+100** | Diberikan saat foto dipublikasikan |
| Bonus (seed/admin) | variabel | Tipe transaksi `bonus` |
| Tukar hadiah | `-points_cost` | Tidak mengurangi `lifetime_points` |
| Tolak penukaran | `+points_spent` | Refund penuh, tipe `redemption_refund` |
| Hapus foto | `-100` | Saldo tidak boleh negatif; `lifetime_points` berkurang 100 |

Semua perubahan tercatat di `point_transactions` beserta `balance_after` untuk audit.

### Level

Level dihitung dari **`lifetime_points`** (total poin yang pernah dikumpulkan), bukan saldo.

| Level | Ambang poin |
|---|---|
| Bronze | 0 |
| Silver | 1.000 |
| Gold | 5.000 |
| Platinum | 10.000 |

Level tersimpan di tabel `levels`, sehingga ambang dan warna badge dapat diubah tanpa mengubah kode. Sinkronisasi otomatis terjadi setiap ada perubahan poin (`LevelService::sync`).

### Penukaran Hadiah

- Minimal saldo **10.000 poin** (`config/photography.php` → `min_points_to_redeem`).
- Saldo juga harus ≥ `points_cost` hadiah.
- Hadiah harus aktif dan stok tersedia.
- Status awal: `pending` → admin dapat `approved`, `fulfilled`, atau `rejected`.
- `rejected` mengembalikan poin dan stok secara otomatis.
- Hadiah yang pernah ditukar tidak dapat dihapus (hanya dinonaktifkan) agar riwayat tetap utuh.

### Rating

- Hanya user login (`permission:ratings.give`).
- Skala 1–5, satu rating per user per foto; mengirim ulang akan memperbarui rating (upsert).
- Tidak boleh menilai foto sendiri.
- Rata-rata dan jumlah penilai ditampilkan di kartu foto dan halaman detail.

### Komentar

- Hanya user login (`permission:comments.create`).
- Panjang 1–1.000 karakter.
- Pemilik dapat mengedit; pemilik atau pemegang `comments.moderate` dapat menghapus.

---

## RBAC & Permission

RBAC memakai **Spatie Permission** dengan guard `api`. Role bersifat dinamis: admin bisa membuat role baru dan memilih permission dari panel `/admin/roles`.

**Daftar permission yang di-seed**

```
photos.view            photos.create          photos.update
photos.delete          photos.moderate        ratings.give
comments.create        comments.moderate      rewards.view
rewards.manage         redemptions.request    redemptions.manage
users.view             users.assign-role      roles.view
roles.manage           dashboard.view
```

**Role bawaan**

| Role | Permission |
|---|---|
| `admin` | Semua permission |
| `user` | photos.view/create/update/delete, ratings.give, comments.create, rewards.view, redemptions.request |
| `moderator` | photos.view, photos.moderate, comments.moderate, rewards.view, dashboard.view |

Catatan penting:

- `Gate::before` di `AppServiceProvider` membuat role **admin** selalu lolos pengecekan permission, termasuk permission baru yang dibuat runtime.
- Role `admin` dan `user` dilindungi: tidak bisa dihapus atau diganti namanya.
- Middleware `permission:*` terpasang di route; pengecekan kepemilikan resource tetap dilakukan lewat Policy (`PhotoPolicy`, `CommentPolicy`).

---

## API Reference

Base URL: `http://localhost:8000/api`
Autentikasi: header `Authorization: Bearer <access_token>`

### Auth

| Method | Endpoint | Akses | Deskripsi |
|---|---|---|---|
| POST | `/auth/register` | publik | Daftar + dapat JWT |
| POST | `/auth/login` | publik | Login + dapat JWT |
| GET | `/auth/me` | login | Profil user saat ini |
| POST | `/auth/logout` | login | Invalidate token |
| POST | `/auth/refresh` | login | Token baru |
| PATCH | `/profile` | login | Ubah nama/username/bio/avatar |

Contoh login:

```bash
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"email":"rani@lensora.test","password":"password"}'
```

### Publik

| Method | Endpoint | Deskripsi |
|---|---|---|
| GET | `/levels` | Daftar level |
| GET | `/photos` | Galeri (`?q=`, `?user=`, `?featured=1`, `?sort=popular|oldest`, `?per_page=`) |
| GET | `/photos/{id}` | Detail foto (+ `average_rating`, `ratings_count`, `my_rating` jika token dikirim) |
| GET | `/photos/{id}/comments` | Komentar sebuah foto |
| GET | `/rewards` | Hadiah aktif |
| GET | `/rewards/{id}` | Detail hadiah |
| GET | `/users/{username}` | Profil publik + foto terbit |
| GET | `/leaderboard?limit=20` | Peringkat berdasarkan lifetime points |
| GET | `/stats` | Statistik komunitas |

### Member (login)

| Method | Endpoint | Permission | Deskripsi |
|---|---|---|---|
| GET | `/my/photos` | photos.view | Foto milik sendiri |
| POST | `/photos` | photos.create | Unggah foto (multipart) → +100 poin |
| PUT/PATCH | `/photos/{id}` | photos.update | Ubah foto sendiri |
| DELETE | `/photos/{id}` | photos.delete | Hapus foto → poin dicabut |
| POST | `/photos/{id}/comments` | comments.create | Kirim komentar |
| PUT/PATCH | `/comments/{id}` | pemilik | Edit komentar |
| DELETE | `/comments/{id}` | pemilik / `comments.moderate` | Hapus komentar |
| POST | `/photos/{id}/rating` | ratings.give | Beri/ubah rating 1–5 |
| DELETE | `/photos/{id}/rating` | login | Hapus rating sendiri |
| GET | `/points` | login | Ringkasan poin + progres level |
| GET | `/points/transactions` | login | Riwayat poin |
| GET | `/redemptions` | redemptions.request | Penukaran milik sendiri |
| POST | `/redemptions` | redemptions.request | Tukar hadiah |
| GET | `/redemptions/{id}` | pemilik/admin | Detail penukaran |

Contoh upload foto:

```bash
curl -X POST http://localhost:8000/api/photos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  -F "title=Kabut Pagi" \
  -F "story=Pagi itu berkabut dan sunyi, aku menunggu matahari menembus lapisan tipis." \
  -F "location=Bromo" \
  -F "taken_at=2026-08-01" \
  -F "image=@/path/foto.jpg"
```

### Admin

| Method | Endpoint | Permission | Deskripsi |
|---|---|---|---|
| GET | `/admin/dashboard` | dashboard.view | Statistik + aktivitas terbaru |
| GET | `/admin/users` | users.view | Daftar pengguna (`?q=`, `?role=`) |
| GET | `/admin/users/{id}` | users.view | Detail pengguna |
| PUT | `/admin/users/{id}/roles` | users.assign-role | Ganti role user |
| GET | `/admin/roles` | roles.view | Daftar role + permission |
| POST | `/admin/roles` | roles.manage | Buat role dinamis |
| PUT | `/admin/roles/{id}` | roles.manage | Ubah nama/permission role |
| DELETE | `/admin/roles/{id}` | roles.manage | Hapus role (bukan bawaan) |
| GET | `/admin/permissions` | roles.view | Daftar permission |
| POST | `/admin/permissions` | roles.manage | Buat permission baru |
| DELETE | `/admin/permissions/{id}` | roles.manage | Hapus permission |
| GET | `/admin/rewards` | rewards.manage | Semua hadiah |
| POST | `/admin/rewards` | rewards.manage | Buat hadiah |
| PUT/PATCH | `/admin/rewards/{id}` | rewards.manage | Ubah hadiah |
| DELETE | `/admin/rewards/{id}` | rewards.manage | Hapus hadiah (jika belum ditukar) |
| GET | `/admin/redemptions` | redemptions.manage | Semua penukaran (`?status=`) |
| PATCH | `/admin/redemptions/{id}` | redemptions.manage | `approved` / `fulfilled` / `rejected` |
| GET | `/admin/photos` | photos.moderate | Semua foto |
| PATCH | `/admin/photos/{id}` | photos.moderate | Ubah status / sorotan |
| DELETE | `/admin/photos/{id}` | photos.moderate | Hapus foto |

Contoh membuat role dinamis:

```bash
curl -X POST http://localhost:8000/api/admin/roles \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"name":"curator","permissions":["photos.view","photos.moderate","dashboard.view"]}'
```

Format error validasi Laravel:

```json
{
  "message": "The story field is required.",
  "errors": { "story": ["The story field is required."] }
}
```

---

## Autentikasi JWT di Frontend

Frontend tidak memakai localStorage. Token disimpan sebagai cookie `httpOnly` oleh route handler Next.js.

| Route BFF | Fungsi |
|---|---|
| `POST /api/auth/login` | Teruskan ke Laravel, set cookie `lensora_token` |
| `POST /api/auth/register` | Sama untuk pendaftaran |
| `POST /api/auth/logout` | Invalidate token Laravel + hapus cookie |
| `GET /api/auth/me` | Profil user dari cookie |
| `/api/proxy/[...path]` | Proxy terautentikasi untuk JSON dan multipart (upload) |

Halaman yang membutuhkan login memanggil `requireUser()` dari `src/lib/auth.ts`, yang membaca cookie, memvalidasi ke Laravel, lalu redirect ke `/login` bila gagal.

`next.config.ts` mengaktifkan `cacheComponents` + `partialPrefetching`; karena itu semua data dinamis dibungkus `<Suspense>` agar shell halaman tetap statis.

---

## Testing

Backend memiliki feature test untuk aturan bisnis utama.

```bash
cd backend
php artisan test
```

Cakupan:

- **AuthTest** — register, login, login gagal, proteksi endpoint
- **PhotoPointTest** — +100 poin, validasi cerita, naik level ke Platinum, pencabutan poin saat hapus, proteksi foto orang lain
- **RatingCommentTest** — guest ditolak, komentar, upsert rating, larangan menilai foto sendiri, validasi 1–5, hapus komentar (pemilik/moderator)
- **RbacTest** — pembuatan role dinamis, blokir permission, assign role, proteksi role bawaan
- **RedemptionTest** — minimal 10.000 poin, pemotongan poin, refund + restore stok saat ditolak

Hasil terakhir: **27 test lulus, 101 assertion**.

---

## Troubleshooting

| Gejala | Penyebab & solusi |
|---|---|
| `401 Unauthenticated` padahal sudah login | Pastikan `AUTH_GUARD=api` di `.env` dan `JWT_SECRET` terisi (`php artisan jwt:secret`). Jalankan `php artisan config:clear`. |
| Gambar tidak muncul | Jalankan `php artisan storage:link`, cek `APP_URL=http://localhost:8000`, dan pastikan `next.config.ts` mengizinkan `localhost:8000/storage/**`. |
| Frontend error "fetch failed" | Backend belum jalan. Jalankan `php artisan serve --port=8000`. Cek `API_URL` di `frontend/.env.local`. |
| CORS error saat memanggil API langsung | BFF seharusnya menghindari CORS. Jika memanggil API langsung, samakan `FRONTEND_URL` di `.env` backend. |
| Login selalu 429 | Rate limit 5/menit per email+IP. Tunggu satu menit atau bersihkan cache. |
| Data demo ingin direset | `php artisan migrate:fresh --seed` |
| `Class "App\Providers\AppServiceProvider" not found` | Jalankan `composer dump-autoload`. |
| `Access denied for user 'root'@'localhost'` | Kredensial MySQL salah. Homebrew default: root tanpa password. MAMP default: `root` / `root` (port **3306** atau **8889** tergantung setelan MAMP). Sesuaikan `DB_USERNAME`, `DB_PASSWORD`, `DB_PORT`. |
| Port 8000/3000 dipakai | Ganti port backend dan sesuaikan `API_URL` (frontend) serta `APP_URL` (backend), atau jalankan `npm run dev -- -p 3001`. |

---

## Batasan & Rencana Lanjutan

Yang **belum** termasuk:

- Verifikasi email dan lupa password
- Like/favorit dan balasan komentar bertingkat
- Notifikasi realtime
- Moderasi otomatis konten (NSFW)
- Optimasi gambar sisi Next.js (`unoptimized` dipakai agar tidak bergantung pada image optimizer)
- Pagination UI lanjutan (API sudah mendukung `page`/`per_page`)
- Test E2E frontend
- MySQL production tuning / queue worker untuk proses berat

Rencana lanjutan yang masuk akal:

1. Notifikasi saat foto diberi rating/komentar.
2. Koleksi/album dan seri foto.
3. Leaderboard bulanan dengan hadiah musiman.
4. Audit log admin (siapa mengubah role/hadiah).
5. Object storage (S3/R2) untuk gambar.
6. Verifikasi email + reset password.

---

Dibangun dengan Laravel + Next.js. Setiap foto punya cerita — dan setiap cerita punya poin.
# Lensora-Photography
