# Engineering Rules — Aplikasi Peminjaman Buku

Aturan ini wajib diikuti dari awal sampai akhir proyek. Tujuan: kode **scalable, secure, sustainable, DRY, testable**. Setiap PR / commit yang melanggar harus diperbaiki sebelum lanjut.

> Skop: Laravel 13 + PHP 8.3 + SQLite + Sanctum 4 + Tailwind v4 + Alpine.js + Yajra DataTables + jQuery (khusus list AJAX).

---

## 1. Architecture & Layering

**Lapisan wajib (1 arah dependency):**

```
Routes → FormRequest → Controller → Service → Model/Repository → Database
                                              ↘ Event/Job (async)
```

- **Controllers tipis.** Hanya: validasi input (via FormRequest), panggil Service, return Response/Resource. Tidak ada query Eloquent / business logic di Controller.
- **Service layer wajib** untuk operasi multi-step / stateful (approve pengajuan, konfirmasi pengembalian, generate kode). Tempat: `app/Services/`. Method statelss & idempotent kalau memungkinkan.
- **Form Requests** untuk SEMUA endpoint yang menerima input. Tidak boleh validate di controller via `$request->validate()`.
- **Eloquent API Resources** untuk SEMUA response API. Tidak return Model/array mentah dari controller API.
- **Policies** untuk SEMUA otorisasi. Tidak boleh `if ($user->id === $resource->user_id)` di controller.
- **Enums (PHP 8.1 backed enum)** untuk status / role / tipe. Tidak ada string magic value di code.
- **Events + Listeners** untuk side-effects (kirim email, log audit, update cache). Jangan panggil side-effect langsung di Service.

**File tidak boleh > 300 LOC.** Jika lebih, ekstrak.

## 2. Naming & Code Style

- **Pint wajib** sebelum commit: `vendor/bin/pint --dirty --format agent`.
- **Method**: deskriptif & verb-first (`approveLoanRequest`, `confirmReturn`, `isAvailable`). Hindari `handle()`, `process()` tanpa konteks.
- **Variable bool**: `is`, `has`, `can` prefix.
- **Enum keys**: TitleCase (`Pending`, `Approved`, `Rejected`, `Borrowed`, `Returned`, `Overdue`).
- **Class**: PascalCase. **DB columns**: snake_case. **Routes**: kebab-case di URL, dot.case di name (`admin.loan-requests.approve`).
- Type hint & return type **wajib** di semua method (termasuk private). PHPDoc array-shape untuk array kompleks.
- Constructor property promotion wajib.
- Tidak ada `// TODO` masuk ke main; tracking di task list, bukan di kode.

## 3. Security (non-negotiable)

- **CSRF** aktif untuk semua form web (default Laravel). Tidak boleh `VerifyCsrfToken::except` kecuali endpoint API ber-Sanctum.
- **Mass assignment**: setiap Model wajib punya `$fillable` eksplisit. Hindari `$guarded = []`.
- **Authorization**: setiap action admin/member dijaga middleware `auth:admin` / `auth:member` + Policy. Tidak ada endpoint "trust by default".
- **Password hashing** lewat `Hash::make()` atau cast `'password' => 'hashed'` di Model. Bcrypt rounds dari `.env` (min 12).
- **Rate limit**: API `throttle:60,1`. Login route `throttle:5,1` per IP+email.
- **Validation** di Form Request **selalu** untuk: `min`, `max`, `exists`, `unique`, dan whitelist (`in:`) untuk enum-like field. Filter input di server, jangan trust client.
- **No raw SQL** kecuali ada justifikasi tertulis di PHPDoc + parameter binding wajib. Default: query builder / Eloquent.
- **No `dd()` / `dump()`** di kode yang masuk ke main.
- **Secret di `.env` saja.** Tidak boleh hardcode token/key/password (termasuk di seeder — pakai `env()` atau random).
- **Logging audit** untuk approve/reject/confirm-return (siapa, kapan, target). Pakai event listener → table `audit_logs` atau Laravel log channel khusus. *(Phase 2+, bukan Phase 0)*
- **API response** tidak expose internal id jika sudah ada `*_code`. Default expose `book_code` bukan `id`.
- **CORS** dikonfigurasi minimal (whitelist origin). Default Laravel 13 sudah aman; jangan loosen tanpa alasan.

## 4. Database & Performance

- **Migrasi atomic** per perubahan, reversible (`up`/`down` simetris). Tidak edit migrasi yang sudah merged.
- **Foreign key wajib** dengan `cascadeOnDelete()` / `restrictOnDelete()` sesuai semantik. Default: `restrictOnDelete` untuk data master (books, members), `cascadeOnDelete` untuk data turunan (loan dari loan_request).
- **Index wajib** untuk: foreign keys, kolom `*_code` (unique), kolom `status`, kolom yang dipakai di `WHERE`/`ORDER BY` di query yang dipanggil > 1x.
- **N+1 dilarang.** Pakai `->with()` eager loading. Test wajib assert `DB::getQueryLog()` tidak meledak (gunakan `->loadCount` jika perlu).
- **Pagination wajib** untuk semua list. Tidak ada `Model::all()` di response.
- **Transaksi DB** wajib untuk operasi multi-tabel (approve = update status + decrement stok + create loan). Pakai `DB::transaction(fn () => ...)`.
- **Optimistic locking / atomic decrement** untuk stok: `Book::where('id', $id)->where('stock', '>', 0)->decrement('stock')` — cek return value.
- **SQLite caveat**: tidak support `ALTER COLUMN` rumit, foreign key harus di-enable per koneksi (`'foreign_key_constraints' => true` di config/database.php). Test pakai `:memory:`.

## 5. DRY & Reuse

- **Trait `HasGeneratedCode`** untuk model dengan kode auto-generate (`book_code`, `member_code`, `request_code`, `loan_code`). Format & sequence di-handle satu tempat.
- **Trait `HasUlid` / `HasFactory`** untuk testability.
- **Blade components** (`<x-...>`) untuk UI berulang: card, table, alert, modal, form-input, btn. Tidak copy-paste markup.
- **Form Request base class** kalau ada validasi yang dipakai > 2x (mis. `BookFormRequest` parent untuk Store/Update).
- **Resource base** untuk pattern response yang sama.
- **Magic strings** → konstanta / enum. Status, role, guard name (`'admin'`, `'member'`) → konstanta di Enum / config.
- **Repeated query → query scope** di Model (`scopeAvailable`, `scopeOverdue`, `scopePending`).

## 6. Testing Strategy (Pest 4)

**Coverage target: 100% controller paths, 100% service methods, semua state transitions.**

- **Setiap fitur** wajib datang dengan test. PR tanpa test → ditolak.
- **Feature test default.** Unit test untuk logic murni (CodeGenerator, calculation late_days, dll).
- **Test cases wajib** per endpoint:
  1. Happy path (200/201)
  2. Unauthenticated → 401/302
  3. Wrong guard (admin akses endpoint member, dst) → 403
  4. Validation fail → 422 + struktur error benar
  5. Not found → 404
  6. Edge: stok habis, double approve, return tanpa loan, dll (bisnis-spesifik)
- **Pakai factories**, jangan `Model::create([...])` di test.
- **`RefreshDatabase` trait** aktif di `tests/Pest.php` untuk semua Feature.
- **Arch tests wajib** (Pest 4):
  - `app/Http/Controllers/**` tidak import `Illuminate\Support\Facades\DB` langsung.
  - `app/Models/**` tidak import `Request`.
  - Tidak ada `dd`, `dump`, `var_dump` di `app/`.
- **DB SQLite `:memory:`** untuk test (lebih cepat, isolasi sempurna). Set di `phpunit.xml`.
- **Browser test (Pest 4)** untuk smoke admin & member dashboard, login flow, dan happy-path ajukan→approve→return.
- **Test naming**: `it('approves a pending loan request and decrements stock')`. Bukan `test_approve()`.
- **Run sebelum commit**: `php artisan test --compact`. Wajib 0 fail.

## 7. Frontend Rules

- **TailwindCSS v4 (CSS-first)**: konfigurasi di `resources/css/app.css` via `@theme`. Tidak buat `tailwind.config.js`.
- **Alpine.js** untuk interaktivitas ringan (toggle, dropdown, modal). Tidak pakai jQuery untuk hal-hal yang Alpine bisa.
- **jQuery** *hanya* untuk inisialisasi Yajra DataTables + AJAX list/refresh (memenuhi syarat brief). Tidak untuk DOM manipulation umum.
- **Reusable Blade components** di `resources/views/components/`. Naming: `x-table`, `x-form.input`, `x-btn`, `x-modal`, `x-alert`.
- **Layout**: `layouts/app.blade.php` (base) → `layouts/admin.blade.php` & `layouts/member.blade.php` (extend). Sidebar/topbar adalah komponen, bukan duplikasi.
- **No inline styles**, no `<style>` blok di blade.
- **No CDN** untuk asset critical (semua via npm + Vite). CDN hanya boleh untuk fallback.
- **Accessibility**: tiap input ada `<label>`, tiap button punya `aria-label` kalau icon-only.
- **SweetAlert2** untuk konfirmasi destruktif (delete, reject) dan toast sukses.

## 8. Git Hygiene

- Commit message: `<type>(<scope>): <subject>` — `feat(book): add CRUD admin endpoints`. Type: feat, fix, refactor, test, docs, chore.
- Satu commit = satu logical change. Tidak `feat: bunch of stuff`.
- Tidak commit: `vendor/`, `node_modules/`, `.env`, `database/database.sqlite`, `storage/logs/*`, `public/build/`.
- `.env.example` selalu sinkron dengan `.env` (key-nya).

## 9. Definition of Done (per fitur)

Sebuah fitur DONE kalau:

- [ ] Migration + Model + Factory + Seeder (kalau perlu)
- [ ] Form Request validasi lengkap
- [ ] Policy + middleware terpasang di route
- [ ] Controller tipis, business logic di Service
- [ ] Resource untuk API / Component untuk UI
- [ ] Pest test cover 6 cases di §6
- [ ] `php artisan test --compact` PASS
- [ ] `vendor/bin/pint --dirty --format agent` clean
- [ ] Manual smoke (UI: lihat di browser)
- [ ] Tidak ada N+1 (verifikasi via Telescope/Debugbar/log)
- [ ] PRD section terkait (jika berubah) sudah di-update

## 10. Forbidden Patterns

- ❌ `Model::all()` di response (selalu paginasi)
- ❌ `$request->all()` ke `create()`/`update()` (selalu `->validated()`)
- ❌ `firstOrFail()` tanpa try/log untuk endpoint user-facing yang error message-nya penting
- ❌ Route closure (gunakan controller class)
- ❌ String `'admin'` / `'member'` ditulis langsung — pakai `Guard::Admin->value`
- ❌ Logic di Blade (`@php ... @endphp` untuk hitungan/query)
- ❌ Comment yang menerangkan WHAT (bukan WHY)
- ❌ Test yang dependent satu sama lain (urutan eksekusi tidak boleh penting)
- ❌ Skip migration / edit history-nya — selalu migration baru

---

**Saat ragu**: cek ulang §1-§4. Jika masih ragu, tanya. Jangan menebak arsitektur.
