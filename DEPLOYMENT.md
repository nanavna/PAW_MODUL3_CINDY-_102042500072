# Deployment Guide

Panduan deploy ke **[Laravel Cloud](https://cloud.laravel.com/)**. Tested untuk free/hobby tier.

---

## TL;DR

1. Pakai **PostgreSQL** (managed oleh Laravel Cloud) — bukan SQLite — di production.
2. Set ~10 env vars di Laravel Cloud dashboard (lihat tabel di bawah).
3. Build commands: `composer install --no-dev --optimize-autoloader && npm ci && npm run build`.
4. After-deploy: `php artisan migrate --force` (sekali saat first deploy, lalu juga di setiap deploy).

---

## 1. SQLite vs PostgreSQL di Laravel Cloud

| | SQLite | PostgreSQL (rekomendasi) |
|---|---|---|
| Setup di Cloud | File ephemeral; **hilang setiap redeploy** kecuali pakai persistent disk add-on | Managed, persistent, snapshot otomatis |
| Cocok untuk | Demo singkat (data boleh hilang) | Production, data harus tetap |
| Env override | Tidak butuh perubahan kode — semua via env vars | Sama, semua via env vars |

**Kode tidak perlu diubah** — Laravel `config/database.php` sudah membaca `DB_CONNECTION` dari env. Cukup ganti `DB_CONNECTION=sqlite` (local) → `DB_CONNECTION=pgsql` (Cloud) di env vars deploy.

> Saran: tetap **SQLite untuk local dev** (tidak butuh service tambahan), **PostgreSQL untuk production**.

---

## 2. Environment Variables (Laravel Cloud Dashboard)

Tab **Environment** di project Laravel Cloud, masukkan minimal:

### Wajib

| Key | Value | Catatan |
|---|---|---|
| `APP_NAME` | `Aplikasi Peminjaman Buku` | |
| `APP_ENV` | `production` | |
| `APP_KEY` | (generate via `php artisan key:generate --show` di local, copy hasilnya) | Format: `base64:...` |
| `APP_DEBUG` | `false` | **WAJIB false di production** |
| `APP_URL` | `https://<your-app>.laravel.cloud` | Sesuai domain Cloud |
| `LOG_CHANNEL` | `stderr` | Cloud-friendly |
| `LOG_LEVEL` | `error` | atau `warning` |
| `BCRYPT_ROUNDS` | `12` | |

### Database (PostgreSQL — auto-injected oleh Cloud)

Laravel Cloud otomatis injeksi env DB jika anda attach managed Postgres ke environment. Jika perlu manual:

| Key | Value |
|---|---|
| `DB_CONNECTION` | `pgsql` |
| `DB_HOST` | dari Cloud database panel |
| `DB_PORT` | `5432` |
| `DB_DATABASE` | dari Cloud |
| `DB_USERNAME` | dari Cloud |
| `DB_PASSWORD` | dari Cloud |

### Session / Cache / Queue

| Key | Value | Catatan |
|---|---|---|
| `SESSION_DRIVER` | `database` | atau `redis` jika attach Redis |
| `SESSION_LIFETIME` | `120` | menit |
| `SESSION_DOMAIN` | `.laravel.cloud` atau domain anda | optional |
| `SESSION_SECURE_COOKIE` | `true` | HTTPS production |
| `CACHE_STORE` | `database` | atau `redis` |
| `QUEUE_CONNECTION` | `database` | atau `redis` |

### Sanctum (untuk API SPA, optional)

```
SANCTUM_STATEFUL_DOMAINS=<your-app>.laravel.cloud
```

### Mail (jika butuh notif)

```
MAIL_MAILER=log    # atau: smtp / ses / postmark / dst
```

### Vite

```
VITE_APP_NAME="${APP_NAME}"
```

---

## 3. Build Commands (Laravel Cloud)

Di tab **Settings → Build** project Laravel Cloud:

```bash
composer install --no-interaction --no-dev --prefer-dist --optimize-autoloader
npm ci
npm run build
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache
```

> `--no-dev` aman karena kita tidak butuh Pint/Pest di production.
> `npm ci` lebih reproducible daripada `npm install` di CI.

## 4. After-Deploy Commands

Di tab **Settings → Deploy hooks**:

```bash
php artisan migrate --force
php artisan storage:link
```

> Di first deploy data kosong. Untuk seed demo data, **JANGAN** auto-seed di production. Manual sekali via Cloud SSH:
> ```bash
> php artisan db:seed --class=AdminSeeder --force
> ```
> Hindari `MemberSeeder` & `BookSeeder` di production (mereka generate data dummy).

---

## 5. Permissions / Writable Dirs

Laravel Cloud sudah handle ini otomatis. Tapi pastikan **dir-dir berikut ter-commit** ke git (kosong, dengan `.gitignore` internal masing-masing untuk ignore content-nya):

```
bootstrap/cache/.gitignore       # *  !.gitignore
storage/app/.gitignore           # *  !private/  !public/  !.gitignore
storage/framework/.gitignore     # ignores compiled.php, config.php, dst.
storage/framework/cache/data/.gitignore
storage/framework/sessions/.gitignore
storage/framework/views/.gitignore
storage/logs/.gitignore
```

> ⚠️ **Jangan tambah `/bootstrap/cache/`, `/storage/app/`, `/storage/framework/` ke ROOT `.gitignore`.** Itu akan ignore seluruh dir termasuk `.gitignore` internalnya, sehingga saat clone/deploy dir-nya tidak ada → Laravel error: *"The /var/www/html/bootstrap/cache directory must be present and writable."*

---

## 6. Troubleshooting

### `bootstrap/cache directory must be present and writable`

Penyebab: dir tidak ter-commit ke git. Cek:
```bash
git ls-files bootstrap/cache storage/app storage/framework | head
```
Harus ada `.gitignore` di tiap dir. Jika kosong, **hapus** baris `/bootstrap/cache/`, `/storage/app/`, `/storage/framework/` dari ROOT `.gitignore`, lalu:
```bash
git add bootstrap/cache/.gitignore storage/app/.gitignore storage/framework/.gitignore
git commit -m "chore: ensure writable dirs are tracked"
git push
```

### `No application encryption key has been specified.`

Set `APP_KEY` di Cloud env. Generate value via:
```bash
php artisan key:generate --show     # copy output base64:... ke Cloud env
```

### `SQLSTATE[08006] connection refused` (Postgres)

Pastikan database service di-attach ke environment Cloud. Cek `Settings → Database`.

### Vite manifest not found

`npm run build` belum jalan, atau `public/build/` tidak ter-deploy. Tambahkan ke build commands di atas.

---

## 7. Lokal vs Production — env diff cheatsheet

```diff
# Local (.env)
- APP_ENV=local
- APP_DEBUG=true
- APP_URL=http://localhost
- DB_CONNECTION=sqlite
- LOG_LEVEL=debug
- SESSION_SECURE_COOKIE= (unset)

# Production (Laravel Cloud env)
+ APP_ENV=production
+ APP_DEBUG=false
+ APP_URL=https://<app>.laravel.cloud
+ DB_CONNECTION=pgsql + DB_HOST/PORT/DATABASE/USERNAME/PASSWORD
+ LOG_LEVEL=error
+ SESSION_SECURE_COOKIE=true
```

---

## 8. Checklist Sebelum Deploy

- [ ] `php artisan test --compact` PASS di local
- [ ] `vendor/bin/pint --test --format agent` clean
- [ ] `.env` **tidak** ter-commit (sudah di `.gitignore`)
- [ ] `.env.example` up-to-date dengan semua env baru
- [ ] `bootstrap/cache/.gitignore`, `storage/app/.gitignore`, `storage/framework/.gitignore` **ter-commit**
- [ ] `composer.lock` & `package-lock.json` ter-commit
- [ ] `public/build/` **tidak** ter-commit (akan di-build di Cloud)
- [ ] `APP_DEBUG=false` di Cloud
- [ ] `APP_KEY` di-set di Cloud
- [ ] Migrations idempotent (no destructive change)
- [ ] No `dd()` / `dump()` tertinggal
