# Contributing Guide

Terima kasih sudah meluangkan waktu berkontribusi pada proyek **Aplikasi Peminjaman Buku**. Dokumen ini menjelaskan cara berkontribusi sehingga proses review berjalan cepat dan kode tetap konsisten.

> **Sumber kebenaran teknis:**
> - [`RULES.md`](./RULES.md) — engineering rules (binding)
> - [`README.md`](./README.md) — overview, ERD, kredensial, contoh API

Jika ada konflik antar dokumen, urutan prioritas: `RULES.md` → `README` → kode itu sendiri.

---

## 1. Code of Conduct

- Hormat dalam komunikasi dan review.
- Asumsikan niat baik. Tanya sebelum menyimpulkan.
- Tidak ada bahasa yang merendahkan, diskriminatif, atau ofensif di issue, PR, commit, atau code comment.

---

## 2. Setup Lingkungan

```bash
git clone <repo-url> books-apps
cd books-apps
composer setup
```

Pastikan:
- PHP **8.3** dengan ekstensi `pdo_sqlite`, `sqlite3`, `mbstring`, `openssl`, `tokenizer`, `xml`, `ctype`, `json`
- Composer ≥ 2
- Node ≥ **20.19** (Vite 8)
- SQLite ≥ 3.35

Verifikasi:

```bash
php artisan test --compact     # harus 100% pass
vendor/bin/pint --test --format agent
```

---

## 3. Workflow

### 3.1 Branching

- `main` — selalu deployable, tests pass
- Fitur: `feat/<short-name>`
- Bugfix: `fix/<short-name>`
- Refactor: `refactor/<short-name>`
- Docs: `docs/<short-name>`

Satu branch = satu logical change. Jangan campur fitur dengan refactor.

### 3.2 Commit Message

Format Conventional Commits:

```
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

`<type>`: `feat` · `fix` · `refactor` · `test` · `docs` · `chore` · `perf` · `style`

Contoh:
```
feat(loan): add overdue auto-flag on dashboard

Closes #42
```

Aturan:
- Subject ≤ 70 karakter, imperative ("add" bukan "added").
- Tidak ada `WIP`, `tmp`, `xxx` di main.
- Tidak commit `vendor/`, `node_modules/`, `.env`, `database/database.sqlite`, `storage/logs/*`, `public/build/`.

### 3.3 Pull Request

Sebelum membuka PR, pastikan:

- [ ] `php artisan test --compact` PASS (100%)
- [ ] `vendor/bin/pint --dirty --format agent` clean
- [ ] Tidak ada `dd()` / `dump()` / `var_dump()` tertinggal
- [ ] Branch up-to-date dengan `main`
- [ ] PR description menjelaskan **why** (bukan hanya what), screenshot untuk perubahan UI
- [ ] Reference issue terkait jika ada (`Closes #N`)

PR template (akan auto-loaded jika ada `.github/pull_request_template.md`):

```markdown
## Why
<!-- Konteks bisnis / motivasi -->

## What
<!-- Ringkasan perubahan -->

## How to test
<!-- Langkah verifikasi manual -->

## Screenshots
<!-- Untuk perubahan UI -->

## Checklist
- [ ] Tests pass
- [ ] Pint clean
- [ ] Docs updated (jika menambah/mengubah API atau env var)
```

---

## 4. Coding Standards

Wajib mengikuti [`RULES.md`](./RULES.md). Highlights:

### 4.1 Arsitektur

```
Routes → FormRequest → Controller → Service → Eloquent → SQLite
```

- Controller **tipis** — hanya validate via FormRequest → call Service → return Response.
- Tidak ada query Eloquent / business logic di Controller atau Blade.
- Multi-step / stateful → wajib di Service di `app/Services/`.
- Validasi semua input via `FormRequest` (tidak `$request->validate()` inline).
- Authorization via Policy / middleware (tidak `if ($user->id === ...)` di controller).

### 4.2 PHP

- PHP 8 constructor property promotion wajib.
- Type hints + return types eksplisit di semua method.
- Backed enums untuk status / role / tipe (`PHP 8.1`).
- Curly braces untuk semua control structures (termasuk single-line).
- TitleCase untuk enum keys (`Pending`, `Approved`).
- Tidak ada string magic value — pakai enum / constant.

### 4.3 Database

- Migration atomic, reversible (`up`/`down` simetris).
- Tidak edit migration yang sudah merged — buat migration baru.
- Foreign key wajib dengan `cascadeOnDelete` / `restrictOnDelete` sesuai semantik.
- Index untuk semua kolom di `WHERE` / `ORDER BY` yang dipakai > 1×.
- Multi-tabel update wajib `DB::transaction(fn () => ...)`.
- Atomic stock ops via `Book::decrementStock()` / `incrementStock()`.

### 4.4 Frontend

- Tailwind v4 CSS-first (`@theme`) — tidak ada `tailwind.config.js`.
- Komponen Blade berulang → `<x-...>` di `resources/views/components/`.
- Alpine.js untuk interaksi ringan; jQuery hanya untuk Yajra DataTables + AJAX list.
- Tidak ada inline `<style>` di Blade.
- Setiap input ada `<label>`, button icon-only ada `aria-label`.

---

## 5. Testing

### 5.1 Aturan

- Setiap fitur wajib datang dengan test. PR tanpa test = ditolak.
- Default: feature test. Unit test untuk logic murni.
- Per-endpoint minimal cover:
  1. Happy path
  2. Unauthenticated → 401/302
  3. Wrong guard → 403
  4. Validation fail → 422
  5. Not found → 404
  6. Edge bisnis (stok habis, double-approve, dll)

### 5.2 Konvensi

```php
it('approves a pending request and decrements stock', function () { ... });
```

- Pakai `it()` (sesuai konvensi project).
- Pakai factories, bukan `Model::create([...])` manual.
- `LazilyRefreshDatabase` aktif global di `tests/Pest.php`.
- Helper `actingAsAdmin()` / `actingAsMember()` tersedia.

### 5.3 Run

```bash
php artisan test --compact                    # full suite
php artisan test --compact --filter=approve   # filter
php artisan test tests/Unit                   # unit only
```

---

## 6. Security

Lihat `RULES.md` §3. Wajib:

- Validate semua input di Form Request — tidak trust client.
- Setiap Model punya `$fillable` eksplisit.
- Authorization via Policy / middleware untuk setiap action.
- Password via `Hash::make()` atau cast `'hashed'`.
- Rate limit auth routes & API (`throttle:5,1` login, `throttle:60,1` API).
- Tidak ada secret hardcoded — semua via `.env`.
- Tidak `dd()` / `dump()` di kode yang masuk main.

Jika menemukan kerentanan keamanan, **jangan buka issue publik**. Email ke maintainer secara private terlebih dulu.

---

## 7. Documentation

Update dokumentasi yang relevan ketika menambah/mengubah:

| Perubahan | Update |
|---|---|
| API endpoint baru / berubah | `README.md` API section |
| ERD / schema migration | `README.md` ERD section |
| Aturan engineering baru | `RULES.md` |
| Env var baru | `.env.example` + `README.md` |
| Konvensi UI baru | `RULES.md` §7 |

---

## 8. Pertanyaan?

Buka GitHub Discussions atau issue dengan label `question`.

Selamat berkontribusi! 🎉
