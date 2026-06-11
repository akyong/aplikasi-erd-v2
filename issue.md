# ISSUE.md — Perencanaan Pembuatan Aplikasi ERP Berdasarkan ERD

Tanggal: 2026-06-11  
Stack: PHP 8.5.5, Laravel 13, Inertia.js, Vue 3, TailwindCSS, PrimeVue, PostgreSQL, Vite  
Fokus issue: membangun fondasi aplikasi sebelum masuk ke business logic ERP seperti sales, purchase, inventory, accounting, dan finance.

---

## 1. Tujuan Utama

Membangun fondasi ERP multi-company yang aman, scalable, branch-aware, role-aware, siap i18n, dan siap dipakai sebagai dasar semua modul bisnis.

Aplikasi ini harus mendukung:

- Multi-company / multi-tenant berbasis `cifs`.
- Isolasi data antar perusahaan.
- Akses internal untuk pemilik aplikasi / perusahaan internal.
- Akses company untuk perusahaan terdaftar.
- Role-based access control berbasis menu dan action.
- Active company context melalui `active_cif`.
- Branch access per user perusahaan.
- Audit log universal untuk aktivitas penting.
- Desain database long-term dan siap internationalization.

---

## 2. Teknologi

- PHP 8.5.5
- Laravel 13
- Inertia.js
- Vue 3
- TailwindCSS
- PrimeVue
- PostgreSQL
- Vite

Catatan implementasi:

- Inertia + Vue digunakan untuk aplikasi web utama.
- PostgreSQL menjadi database utama.
- PrimeVue digunakan untuk komponen UI seperti dialog, datatable, form input, dropdown, dan confirm dialog.
- TailwindCSS digunakan untuk layout dan styling custom.
- Identifikasi user menggunakan JWT.
- Untuk keamanan web app, JWT sebaiknya disimpan di HTTP-only secure cookie, bukan localStorage.

---

## 3. Istilah Utama

| Istilah | Arti |
|---|---|
| `cifs` | Perusahaan / tenant |
| `users` | Pengguna aplikasi |
| `cif_users` | Membership user pada perusahaan tertentu |
| `branches` | Cabang / unit organisasi perusahaan |
| `cif_user_branches` | Hak akses user perusahaan ke cabang tertentu |
| `roles` | Role dalam sistem, bisa system role atau company role |
| `menus` | Struktur navigasi aplikasi |
| `permissions` | Aksi yang boleh dilakukan, biasanya terkait route/action |
| `role_menus` | Hak role untuk melihat menu |
| `role_permissions` | Hak role untuk menjalankan action tertentu |
| `user_roles` | Role milik user dalam konteks `cif_user` |
| `audit_logs` | Catatan aktivitas create/update/delete/login/action penting |

---

## 4. Role Bawaan

### 4.1 SYS_ADMIN

Role internal tertinggi.

Hak akses:

- Bisa mengakses semua menu.
- Bisa mengakses semua CIF/perusahaan.
- Bisa mengelola menu internal seperti:
  - CIF management
  - Plan management
  - Menu management
  - Role system management
  - User guide translation
  - Internal setting
- Bisa melihat dan mengelola data lintas perusahaan jika diperlukan.
- Tidak boleh dihapus.
- Tidak boleh diubah code-nya.
- Hanya boleh dimiliki user internal.

### 4.2 CORPORATE_ADMIN

Role tertinggi untuk perusahaan terdaftar.

Hak akses:

- Dibuat otomatis ketika perusahaan baru dibuat/terdaftar.
- Minimal setiap CIF harus memiliki 1 user dengan role ini.
- Bisa mengakses semua menu milik perusahaannya, kecuali menu internal.
- Bisa mengelola user, role, branch, dan permission dalam CIF miliknya.
- Tidak bisa melihat CIF/perusahaan lain.
- Tidak bisa mengakses menu internal seperti master plan, menu builder global, user guide translation global, atau data perusahaan lain.
- Tidak boleh dihapus jika menjadi satu-satunya corporate admin dalam CIF tersebut.

---

## 5. Prinsip Arsitektur Multi-Company

### 5.1 CIF sebagai tenant

`cifs` adalah root tenant. Semua data bisnis harus memiliki `cif_id`, baik langsung di tabel utama atau tidak langsung melalui relasi parent.

Contoh:

- `branches.cif_id`
- `warehouses.cif_id`
- `accounts.cif_id`
- `sales_orders.cif_id`
- `purchase_orders.cif_id`
- `journal_entries.cif_id`

### 5.2 Active CIF context

Setiap request user harus memiliki konteks perusahaan aktif.

Context minimal:

- `active_cif_id`
- `active_cif_user_id`
- `active_branch_id` jika user memilih cabang aktif
- `is_internal_user`
- daftar role dan permission aktif

Context ini dapat dibaca dari JWT claim, session context, atau database lookup setelah token valid.

### 5.3 Internal vs company isolation

Aturan utama:

- User internal boleh mengelola semua CIF jika memiliki permission internal.
- User company hanya boleh melihat data CIF miliknya.
- Menu internal hanya boleh muncul untuk user internal.
- Menu seperti `User`, `Role`, dan `Branch` ditampilkan berdasarkan CIF aktif.
- Untuk user company, data `users`, `roles`, dan `branches` harus selalu scoped ke CIF aktif.
- Untuk user internal, modul tertentu boleh menampilkan semua CIF.

### 5.4 Branch-aware access

User dalam CIF bisa memiliki akses cabang:

- `ALL`: user bisa mengakses semua branch dalam CIF tersebut.
- `SELECTED`: user hanya bisa mengakses branch yang ada di `cif_user_branches`.

Aturan:

- Jika mode `SELECTED`, minimal harus ada 1 branch.
- Hanya boleh ada 1 default branch per `cif_user_id`.
- Branch yang dipilih harus milik CIF yang sama dengan `cif_user.cif_id`.

---

## 6. Struktur Menu dan Permission

### 6.1 Menu 3 level

Menu harus mendukung minimal 3 level.

Contoh struktur:

```text
Penjualan
├── Sales Order
├── Delivery Order
└── Sales Invoice

Master / Perlengkapan
├── Produk
│   ├── Barang
│   ├── Satuan
│   ├── Kategori Barang
│   └── Kelompok Barang
└── Gudang

Sistem Internal
├── CIF Management
├── Plan Management
├── Menu Management
└── User Guide Translation
```

Rekomendasi struktur:

- Level 1: module group, contoh `Penjualan`, `Pembelian`, `Master`, `Accounting`, `System`.
- Level 2: page/menu utama, contoh `Sales Order`, `Delivery Order`, `Produk`, `Role`.
- Level 3: sub-page atau child menu khusus jika dibutuhkan.
- Action seperti create/update/delete/approve/post/cancel sebaiknya masuk ke `permissions`, bukan selalu menjadi menu.

### 6.2 Menu permission

`role_menus` menentukan menu apa yang muncul di sidebar.

Aturan:

- Sidebar hanya render menu aktif.
- Sidebar hanya render menu yang dimiliki role user pada CIF aktif.
- Menu internal tidak boleh muncul untuk user non-internal.
- Jika parent tidak punya child yang boleh diakses, parent tidak ditampilkan.

### 6.3 Action permission

`permissions` dan `role_permissions` menentukan action apa yang boleh dilakukan user.

Contoh permission:

```text
system.company.view
system.company.create
system.company.update
system.company.delete
system.role.view
system.role.create
system.role.update
system.role.delete
sales.sales_order.view
sales.sales_order.create
sales.sales_order.update
sales.sales_order.confirm
sales.sales_order.cancel
```

Aturan:

- Route middleware harus mengecek permission route/action.
- UI harus punya helper untuk menampilkan/sembunyikan tombol berdasarkan permission.
- Backend tetap menjadi sumber kebenaran. UI helper hanya untuk kenyamanan.

---

## 7. JWT Authentication Design

### 7.1 JWT claim minimal

JWT perlu membawa identitas minimal:

```json
{
  "sub": 1,
  "active_cif_id": 1,
  "active_cif_user_id": 1,
  "active_branch_id": 1,
  "token_version": 1,
  "iat": 1760000000,
  "exp": 1760003600
}
```

Catatan:

- Jangan masukkan terlalu banyak permission ke JWT jika permission sering berubah.
- Permission lebih aman di-load ulang dari database/cache per request atau per login.
- Gunakan `token_version` atau session table untuk invalidasi token.
- Untuk single-device login, simpan active token/session identifier di database.

### 7.2 Middleware awal

Middleware yang dibutuhkan:

- `auth.jwt`
- `active.cif`
- `internal.only`
- `company.scope`
- `branch.access`
- `permission:permission_code`
- `menu:menu_code` jika masih diperlukan

---

## 8. Milestone Implementasi

## Milestone 0 — Schema Reconciliation & ERD Cleanup

Tujuan: membereskan mismatch antara migration, model, dan target desain sebelum fitur dibuat lebih jauh.

Checklist:

- [ ] Audit semua model `$fillable` vs kolom database.
- [ ] Audit semua relasi model vs foreign key database.
- [ ] Tentukan field mana yang akan dipakai long-term.
- [ ] Buat migration tambahan, jangan edit migration lama jika sudah pernah migrate di environment lain.
- [ ] Tambahkan `max_branches` ke `plans` jika plan harus membatasi jumlah branch.
- [ ] Tambahkan `is_system` ke `roles` untuk protected role.
- [ ] Evaluasi apakah `roles` perlu `cif_id` nullable untuk mendukung role per company.
- [ ] Tambahkan `translation_key` ke `menus` jika menu akan siap i18n.
- [ ] Tambahkan index untuk kolom yang sering dipakai filter: `cif_id`, `branch_id`, `is_active`, `code`, `created_at`.

Acceptance criteria:

- Semua model tidak memiliki fillable yang tidak ada di database, kecuali memang sengaja virtual/derived.
- Semua foreign key yang dipakai di relasi benar-benar ada.
- Seeder berhasil jalan dari database kosong.
- Migration fresh + seed berhasil tanpa error.

---

## Milestone 1 — Authentication & Security

Fitur:

- Login & authentication.
- Password reset via email.
- Route protection.
- JWT-based user identification.
- Token invalidation.
- Optional single-device login.

Checklist backend:

- [ ] Buat login endpoint/controller.
- [ ] Generate JWT setelah login valid.
- [ ] Simpan token/session identifier untuk invalidasi.
- [ ] Implement logout dan revoke token.
- [ ] Implement password reset via email.
- [ ] Tambahkan middleware `auth.jwt`.
- [ ] Tambahkan throttle untuk login.
- [ ] Tambahkan audit log untuk login success, login failed, logout, password reset request, password reset success.

Checklist frontend:

- [ ] Buat halaman login Inertia + Vue.
- [ ] Buat halaman forgot password.
- [ ] Buat halaman reset password.
- [ ] Simpan auth state secara aman.
- [ ] Redirect user ke dashboard setelah login.
- [ ] Tampilkan error login dengan PrimeVue dialog/toast.

Acceptance criteria:

- User aktif bisa login.
- User nonaktif tidak bisa login.
- Password reset email bisa dikirim.
- Route protected tidak bisa diakses tanpa token valid.
- Logout membuat token tidak bisa dipakai lagi.

---

## Milestone 2 — Active Company Context

Fitur:

- Active company context (`active_cif`).
- Switch company untuk user yang punya lebih dari satu CIF.
- Company-specific data isolation.
- Internal bypass untuk user internal yang punya permission.

Checklist:

- [ ] Buat service `CifContextService` atau `ActiveCifService`.
- [ ] Ambil daftar CIF user dari `cif_users`.
- [ ] Set default active CIF setelah login.
- [ ] Buat fitur switch active CIF.
- [ ] Validasi user hanya bisa switch ke CIF yang dimiliki.
- [ ] Inject `active_cif` ke Inertia shared props.
- [ ] Semua query modul company harus scoped ke active CIF.
- [ ] Tambahkan middleware `active.cif`.
- [ ] Tambahkan test isolation antar CIF.

Acceptance criteria:

- User dengan 1 CIF langsung masuk ke CIF tersebut.
- User dengan banyak CIF bisa memilih active CIF.
- User company tidak bisa membaca data CIF lain.
- User internal bisa mengakses modul internal sesuai permission.

---

## Milestone 3 — Multi Company Architecture & CIF Management

Fitur:

- Multi-company support.
- Internal company management.
- Company registration/create flow.
- Company Admin auto-generation.

Checklist create CIF:

- [ ] Validasi `plan_id` aktif.
- [ ] Generate code CIF unik.
- [ ] Create row `cifs`.
- [ ] Create default/head office branch.
- [ ] Create atau attach user owner.
- [ ] Create `cif_users` untuk owner.
- [ ] Assign role `CORPORATE_ADMIN`.
- [ ] Setup default role/menu/permission untuk CIF.
- [ ] Setup default numbering series jika diperlukan.
- [ ] Tulis audit log.
- [ ] Semua proses dibungkus database transaction.

Checklist internal CIF management:

- [ ] SYS_ADMIN bisa melihat semua CIF.
- [ ] SYS_ADMIN bisa create/update/deactivate CIF.
- [ ] SYS_ADMIN bisa assign plan ke CIF.
- [ ] SYS_ADMIN bisa melihat jumlah user dan branch per CIF.
- [ ] Company user tidak bisa membuka daftar CIF lain.

Acceptance criteria:

- CIF baru selalu punya minimal 1 user corporate admin.
- CIF baru selalu punya minimal 1 branch head office.
- Data CIF baru tidak terlihat oleh company lain.
- Plan limit bisa divalidasi saat create user/branch.

---

## Milestone 4 — User & Role Management

Fitur:

- User management.
- Dynamic role management.
- Protected system roles.
- Assign role ke user per CIF.
- Assign branch access ke user.

Checklist user management:

- [ ] Corporate admin bisa membuat user dalam CIF miliknya.
- [ ] Validasi plan `max_users` saat create user baru dalam CIF.
- [ ] Internal SYS_ADMIN bisa membuat user internal.
- [ ] User company hanya melihat user dalam CIF aktif.
- [ ] User internal bisa melihat user lintas CIF jika punya permission.
- [ ] Deactivate user tidak menghapus data historis.
- [ ] Tidak boleh menghapus corporate admin terakhir dalam CIF.

Checklist role management:

- [ ] Role system seperti `SYS_ADMIN` dan `CORPORATE_ADMIN` protected.
- [ ] Role custom bisa dibuat per CIF.
- [ ] Corporate admin bisa membuat role custom untuk CIF sendiri.
- [ ] Role internal tidak bisa dipakai company user.
- [ ] Assign menu ke role melalui `role_menus`.
- [ ] Assign action ke role melalui `role_permissions`.
- [ ] Assign role ke user melalui `user_roles`.

Acceptance criteria:

- Role bisa dikustomisasi per perusahaan.
- Role protected tidak bisa dihapus/diubah sembarangan.
- User company tidak bisa assign role internal.
- Hak akses user berubah sesuai role yang diberikan.

---

## Milestone 5 — Branch Access Foundation

Fitur:

- Branch management.
- Plan-based branch limit.
- Branch access per user.
- Default branch.

Checklist:

- [ ] Tambahkan `max_branches` di `plans` jika belum ada.
- [ ] Validasi `max_branches` saat create branch.
- [ ] Branch code unique per CIF.
- [ ] CIF harus punya tepat satu head office branch.
- [ ] User bisa diberi branch access `ALL` atau `SELECTED`.
- [ ] Jika `SELECTED`, wajib ada minimal 1 branch.
- [ ] Hanya boleh ada 1 default branch per `cif_user`.
- [ ] Validasi branch yang dipilih harus milik CIF yang sama.
- [ ] Tambahkan middleware `branch.access`.
- [ ] Inject daftar branch user ke Inertia shared props.

Acceptance criteria:

- Free plan hanya bisa membuat branch sesuai `max_branches`.
- User dengan akses selected hanya melihat data branch yang diberikan.
- User dengan akses all bisa melihat semua branch dalam CIF aktif.
- Default branch konsisten dan tidak double.

---

## Milestone 6 — Permission System

Fitur:

- Dynamic sidebar rendering.
- Route permission middleware.
- UI permission helpers.
- Menu + action permissions.
- Internal-only system modules.

Checklist backend:

- [ ] Buat `PermissionService`.
- [ ] Buat `MenuService` untuk sidebar tree.
- [ ] Query menu berdasarkan active CIF, role, dan internal flag.
- [ ] Query permission berdasarkan active CIF dan role.
- [ ] Cache permission per request atau per user-active-cif.
- [ ] Buat middleware `permission:{code}`.
- [ ] Buat middleware `internal.only`.
- [ ] Tambahkan helper backend `canPermission($code)`.

Checklist frontend:

- [ ] Share `menus` via Inertia props.
- [ ] Share `permissions` atau helper boolean via Inertia props.
- [ ] Render sidebar berdasarkan menu tree.
- [ ] Buat helper Vue `can('permission.code')`.
- [ ] Sembunyikan tombol create/edit/delete jika user tidak punya permission.
- [ ] Backend tetap validasi permission meskipun tombol disembunyikan.

Acceptance criteria:

- Sidebar berubah sesuai role.
- Route terlindungi permission.
- User tanpa permission tidak bisa akses route langsung.
- Menu internal tidak muncul untuk company user.

---

## Milestone 7 — Audit System

Fitur:

- Universal audit logging.
- Track create/update/delete.
- User activity logs.
- IP & user agent logging.
- Company-aware audit tracking.

Checklist:

- [ ] Buat `Auditable` trait atau observer global per model penting.
- [ ] Log action create/update/delete/deactivate/restore jika ada.
- [ ] Simpan `user_id`.
- [ ] Simpan `cif_id` dari active context.
- [ ] Simpan `auditable_type` dan `auditable_id`.
- [ ] Simpan `old_values` dan `new_values` dalam JSON.
- [ ] Simpan IP address.
- [ ] Simpan user agent.
- [ ] Hindari menyimpan data sensitif seperti password/token.
- [ ] Batasi ukuran JSON jika terlalu besar.
- [ ] Audit ditulis setelah perubahan berhasil, idealnya dalam transaction yang sama.

Acceptance criteria:

- Create/update/delete pada master penting tercatat.
- Audit bisa difilter berdasarkan CIF.
- Audit bisa difilter berdasarkan user.
- Audit bisa menampilkan perubahan old vs new.
- Password/token tidak pernah masuk audit log.

---

## Milestone 8 — i18n Foundation

Fitur:

- Desain siap internationalization.
- Translation key untuk menu dan label penting.
- User guide translations.

Checklist:

- [ ] Tambahkan `translation_key` ke `menus` jika belum ada.
- [ ] Gunakan kode stabil untuk translation key, bukan nama bebas.
- [ ] Buat file bahasa awal `id`, `en`, dan opsional `zh`.
- [ ] Pastikan sidebar bisa menampilkan label sesuai locale.
- [ ] Implement `user_guides` dan `user_guide_translations`.
- [ ] Pastikan slug unique per locale.
- [ ] Siapkan pattern untuk translation master data jika dibutuhkan long-term.

Acceptance criteria:

- User bisa melihat menu dalam bahasa aktif.
- User guide bisa punya versi per locale.
- Struktur menu tidak tergantung pada label bahasa tertentu.

---

## Milestone 9 — UI Foundation

Fitur:

- Layout aplikasi standar.
- Sidebar dinamis.
- Topbar active CIF/branch.
- PrimeVue component standard.
- Reusable dialog & table pattern.

Checklist:

- [ ] Buat `AppLayout.vue` / `AuthenticatedLayout.vue` final.
- [ ] Render dynamic sidebar dari props `menus`.
- [ ] Tampilkan active company di topbar.
- [ ] Tampilkan active branch di topbar jika branch-aware.
- [ ] Buat `PageHeader`.
- [ ] Buat `StatusBadge`.
- [ ] Buat `FlashMessageDialog`.
- [ ] Buat `ConfirmActionDialog`.
- [ ] Buat `ValidationAlertDialog`.
- [ ] Buat standar datatable: search, status, pagination, sort_by, sort_direction, per_page.
- [ ] Buat standar form page create/edit/show.

Acceptance criteria:

- Semua halaman foundation memakai layout yang sama.
- Sidebar tidak hardcode.
- Tombol action mengikuti permission helper.
- Flash/error/validation tampil konsisten.

---

## 9. Checklist Per Tabel Saat Ini

## 9.1 `plans`

Migration saat ini:

- `id`
- `code` unique
- `name`
- `max_users`
- `show_ads`
- `is_active`
- timestamps

Mismatch:

- Model memiliki `max_branches`, tetapi migration belum punya.

Checklist:

- [ ] Tambahkan migration `add_max_branches_to_plans_table`.
- [ ] Default free plan: `max_users = 3`, `max_branches = 1`, `show_ads = true`.
- [ ] Paid plan: `show_ads = false`, limit sesuai paket.
- [ ] Validasi `max_users >= 0`.
- [ ] Validasi `max_branches >= 0`.
- [ ] Gunakan plan limit saat create user dan branch.

---

## 9.2 `cifs`

Migration saat ini:

- `id`
- `plan_id`
- `code` unique
- `name`
- `is_internal`
- `email`
- `phone`
- `address`
- `is_active`
- `created_by`
- `updated_by`
- timestamps

Mismatch:

- Model punya field tambahan seperti `alias`, `fax`, `website`, `city`, `postal_code`, `country_code`, `tax_*`, dan lain-lain yang belum ada di migration.

Relasi penting:

- `belongsTo Plan`
- `hasMany Branch`
- `hasMany CifUser`
- `hasMany Role` jika role dibuat per CIF
- `hasMany Currency`
- `hasMany Warehouse`
- `hasMany AccountGroup`
- `hasMany Account`
- `hasMany FiscalYear`
- `hasMany PostingProfile`

Checklist:

- [ ] Putuskan field tambahan mana yang benar-benar dipakai long-term.
- [ ] Buat migration tambahan untuk field legal/tax/contact jika diperlukan.
- [ ] Validasi `plan_id` wajib dan aktif.
- [ ] Code harus unique.
- [ ] Create CIF harus transaction.
- [ ] Create CIF harus otomatis membuat default branch dan corporate admin.
- [ ] CIF internal minimal punya 1 user `SYS_ADMIN`.

---

## 9.3 `users`

Peran:

- Identitas login pengguna.
- Satu user bisa menjadi anggota beberapa CIF melalui `cif_users`.

Checklist:

- [ ] Pastikan `email` unique.
- [ ] Tambahkan `is_active` jika belum ada.
- [ ] Tambahkan audit untuk create/update/deactivate.
- [ ] Jangan hard delete user yang sudah punya transaksi.
- [ ] Gunakan `cif_users` untuk scope perusahaan, bukan langsung `users.cif_id`.

---

## 9.4 `cif_users`

Migration saat ini:

- `id`
- `cif_id`
- `user_id`
- `branch_access_mode`
- `is_active`
- unique `(cif_id, user_id)`
- timestamps

Relasi:

- `belongsTo Cif`
- `belongsTo User`
- `hasMany UserRole`
- `hasMany CifUserBranch`
- `belongsToMany Branch` via `cif_user_branches`

Checklist:

- [ ] Validasi `user_id` dan `cif_id` eksis.
- [ ] Enforce unique `(cif_id, user_id)`.
- [ ] Saat membuat user perusahaan, assign minimal 1 role.
- [ ] Jika `branch_access_mode = SELECTED`, wajib assign branch.
- [ ] Jika `branch_access_mode = ALL`, pivot branch optional.
- [ ] Semua operasi create/update harus transaction.
- [ ] Tambahkan test update mode `ALL` ke `SELECTED` dan sebaliknya.

---

## 9.5 `cif_user_branches`

Kolom:

- `id`
- `cif_user_id`
- `branch_id`
- `is_default`
- timestamps
- unique `(cif_user_id, branch_id)`

Invariant:

- Hanya boleh ada satu `is_default = true` per `cif_user_id`.
- Branch harus milik CIF yang sama dengan `cif_user.cif_id`.

Checklist:

- [ ] Saat set default branch, clear default lama dalam transaction.
- [ ] Validasi `branch.cif_id === cif_user.cif_id`.
- [ ] Tambahkan index untuk `cif_user_id`, `branch_id`, dan `is_default`.
- [ ] Tambahkan integration test untuk default branch.

---

## 9.6 `branches`

Kolom saat ini:

- `id`
- `cif_id`
- `code`
- `name`
- `email`
- `phone`
- `address`
- `city`
- `province`
- `postal_code`
- `is_head_office`
- `is_active`
- timestamps
- unique `(cif_id, code)`

Invariant:

- Setiap CIF aktif minimal punya 1 branch.
- Setiap CIF sebaiknya punya tepat 1 head office branch.

Checklist:

- [ ] Create branch pertama otomatis `is_head_office = true`.
- [ ] Branch berikutnya default `is_head_office = false`.
- [ ] Jika mengganti head office, clear head office lama dalam transaction.
- [ ] Validasi code unique per CIF.
- [ ] Validasi limit `plans.max_branches`.
- [ ] Company user hanya melihat branch CIF aktif.
- [ ] Internal user bisa melihat branch semua CIF jika punya permission.

---

## 9.7 `roles`

Migration saat ini:

- `id`
- `code` unique
- `name`
- `description`
- `is_active`
- `created_by`
- `updated_by`
- timestamps

Mismatch:

- Model punya `is_system`, tetapi migration belum punya.
- Model/relasi mengarah ke `cif_id`, tetapi migration saat ini belum jelas memiliki `cif_id`.

Rekomendasi long-term:

- Tambahkan `cif_id` nullable.
- `cif_id = null` untuk system/internal/global role.
- `cif_id != null` untuk role custom perusahaan.
- Tambahkan `is_system` untuk role protected.

Checklist:

- [ ] Tambahkan migration `add_cif_id_and_is_system_to_roles_table` jika diputuskan role per CIF.
- [ ] Protected role `SYS_ADMIN` dan `CORPORATE_ADMIN` tidak boleh dihapus.
- [ ] Corporate admin hanya bisa membuat role untuk CIF aktif.
- [ ] Role code unique perlu dievaluasi: global unique atau unique per CIF.
- [ ] Jika code per CIF, gunakan unique `(cif_id, code)`.

---

## 9.8 `user_roles`

Peran:

- Pivot `cif_user` ke `role`.

Checklist:

- [ ] Enforce unique `(cif_user_id, role_id)`.
- [ ] Saat assign role company, validasi role milik CIF yang sama atau role global yang boleh dipakai.
- [ ] User company tidak boleh menerima role internal.
- [ ] Tidak boleh menghapus role `CORPORATE_ADMIN` terakhir dalam CIF.
- [ ] Audit assign/remove role.

---

## 9.9 `menus`

Kolom saat ini:

- `id`
- `parent_id`
- `code` unique
- `name`
- `icon`
- `menu_type`
- `route_name`
- `url`
- `sort_order`
- `is_active`
- timestamps

Mismatch:

- Model punya `translation_key`, tetapi migration belum punya.

Rekomendasi tambahan:

- Tambahkan `is_internal` untuk menandai menu khusus internal.
- Tambahkan `translation_key` untuk i18n.

Checklist:

- [ ] Tambahkan migration `add_translation_key_to_menus_table`.
- [ ] Evaluasi menambah `is_internal`.
- [ ] Build tree retrieval dengan eager load children.
- [ ] Menu parent tidak ditampilkan jika tidak ada child yang bisa diakses.
- [ ] Menu internal hanya untuk SYS_ADMIN/internal user.
- [ ] Gunakan `role_menus` untuk hak tampil sidebar.

---

## 9.10 `role_menus`

Peran:

- Pivot role ke menu.

Checklist:

- [ ] Enforce unique `(role_id, menu_id)`.
- [ ] Assign menu ke role harus scoped sesuai tipe role.
- [ ] Role company tidak boleh diberi menu internal.
- [ ] Audit perubahan role menu.

---

## 9.11 `permissions`

Kolom saat ini:

- `menu_id`
- `code` unique
- `name`
- `route_name` unique
- `is_active`

Checklist:

- [ ] Gunakan permission code stabil.
- [ ] Jangan bergantung hanya pada route name jika action bisa berubah.
- [ ] Tambahkan permission untuk view/create/update/delete per modul foundation.
- [ ] Tambahkan permission khusus seperti approve/post/cancel nanti saat masuk bisnis logic.
- [ ] Permission internal hanya bisa diassign ke role internal.

---

## 9.12 `role_permissions`

Peran:

- Pivot role ke permission/action.

Checklist:

- [ ] Enforce unique `(role_id, permission_id)`.
- [ ] Validasi permission internal tidak diberikan ke role company.
- [ ] Cache permission setelah assign/update.
- [ ] Audit perubahan role permission.

---

## 9.13 `numbering_series`

Kolom & behavior:

- `cif_id`
- `branch_id`
- `document_type`
- `name`
- `prefix_format`
- `next_number`
- `padding`
- `reset_yearly`
- `reset_monthly`
- `is_default`
- `is_active`
- unique `(cif_id, document_type, name)`

Checklist:

- [ ] Buat service generator nomor dokumen.
- [ ] Gunakan database transaction dan row lock saat ambil increment `next_number`.
- [ ] Implement reset yearly/monthly.
- [ ] Pastikan default series hanya satu per document type/cif/branch jika dibutuhkan.
- [ ] Belum wajib dipakai sebelum bisnis logic, tapi foundation service boleh disiapkan.

---

## 9.14 `audit_logs`

Kolom saat ini:

- `user_id` nullable
- `cif_id` nullable
- `action`
- `auditable_type`
- `auditable_id`
- `description`
- `old_values` JSON
- `new_values` JSON
- `ip_address`
- `user_agent`
- timestamps

Checklist:

- [ ] Buat audit trait/observer.
- [ ] Jangan log password/token/secret.
- [ ] Log create/update/delete/deactivate/assign-role/permission-change/login/logout.
- [ ] Pastikan `cif_id` diisi dari active context.
- [ ] Untuk internal action lintas CIF, isi `cif_id` target jika ada.
- [ ] Tambahkan index `user_id`, `cif_id`, `auditable_type`, `auditable_id`, `created_at`.

---

## 9.15 `user_guides` & `user_guide_translations`

Behavior:

- `user_guides` menyimpan metadata panduan.
- `user_guide_translations` menyimpan konten per locale.
- Unique `(user_guide_id, locale)` dan `(locale, slug)`.

Checklist:

- [ ] Implement helper `translation($locale)`.
- [ ] Pastikan slug unique per locale.
- [ ] Menu user guide translation hanya internal.
- [ ] Company user hanya membaca panduan, bukan mengelola translation global.

---

## 10. Modul Foundation yang Harus Ada Sebelum Business Logic

## 10.1 Authentication & Security

- [ ] Login
- [ ] Logout
- [ ] JWT authentication
- [ ] Password reset via email
- [ ] Route protection
- [ ] Token invalidation
- [ ] Login throttling
- [ ] Optional single-device login

## 10.2 Multi Company Architecture

- [ ] CIF management
- [ ] Active CIF context
- [ ] Switch active CIF
- [ ] Internal company management
- [ ] Company-specific data isolation
- [ ] Internal vs company access isolation

## 10.3 User & Role Management

- [ ] User management scoped by CIF
- [ ] Dynamic role management
- [ ] Protected system roles
- [ ] Company admin auto-generation
- [ ] Assign role to user
- [ ] Assign branch access to user
- [ ] Prevent deleting last corporate admin

## 10.4 Permission System

- [ ] Menu-based permissions
- [ ] Action-based permissions
- [ ] Dynamic sidebar rendering
- [ ] Route permission middleware
- [ ] UI permission helpers
- [ ] Internal-only system modules

## 10.5 Audit System

- [ ] Universal audit logging
- [ ] Track create/update/delete
- [ ] User activity logs
- [ ] IP logging
- [ ] User agent logging
- [ ] Company-aware audit tracking

## 10.6 i18n Foundation

- [ ] Menu translation key
- [ ] User guide translation
- [ ] Locale switcher
- [ ] Translation files `id`, `en`, optional `zh`

---

## 11. Rekomendasi Folder Structure Laravel

```text
app/
├── Actions/
├── Enums/
├── Http/
│   ├── Controllers/
│   │   ├── Auth/
│   │   ├── System/
│   │   ├── Company/
│   │   └── Foundation/
│   ├── Middleware/
│   └── Requests/
├── Models/
├── Observers/
├── Policies/
├── Services/
│   ├── Auth/
│   ├── Audit/
│   ├── Company/
│   ├── Menu/
│   ├── Permission/
│   └── Numbering/
└── Support/
    ├── Context/
    └── Helpers/

resources/js/
├── Components/
├── Composables/
├── Layouts/
├── Pages/
│   ├── Auth/
│   ├── Dashboard/
│   ├── System/
│   └── Company/
└── Stores/
```

---

## 12. Middleware Target

```text
auth.jwt
active.cif
internal.only
company.scope
branch.access
permission:{permission_code}
menu:{menu_code}
```

Prioritas implementasi:

1. `auth.jwt`
2. `active.cif`
3. `permission:{permission_code}`
4. `internal.only`
5. `branch.access`

---

## 13. Service Target

```text
AuthService
JwtService
ActiveCifService
CifService
CifUserService
BranchService
RoleService
PermissionService
MenuService
AuditService
NumberingSeriesService
```

Aturan service:

- Controller tidak berisi business logic berat.
- Controller validasi request dan panggil service.
- Service menggunakan transaction untuk operasi multi-table.
- Query list/pagination sebaiknya dipisah ke QueryService jika mulai kompleks.

---

## 14. Seeder Awal

Seeder minimal:

- `PlanSeeder`
- `InternalCifSeeder`
- `SystemRoleSeeder`
- `MenuSeeder`
- `PermissionSeeder`
- `RoleMenuSeeder`
- `RolePermissionSeeder`
- `SysAdminUserSeeder`

Data awal:

```text
Plan:
- FREE: max_users=3, max_branches=1, show_ads=true
- BASIC/PRO: sesuai rencana monetisasi, show_ads=false

Role:
- SYS_ADMIN
- CORPORATE_ADMIN

Internal CIF:
- CIF internal pemilik aplikasi
- is_internal=true

User:
- 1 user SYS_ADMIN internal
```

Acceptance criteria:

- `php artisan migrate:fresh --seed` berhasil.
- User SYS_ADMIN bisa login.
- SYS_ADMIN melihat menu internal.
- Corporate admin test user tidak melihat menu internal.

---

## 15. Testing Plan

## 15.1 Feature tests

- [ ] Login berhasil.
- [ ] Login gagal password salah.
- [ ] User inactive tidak bisa login.
- [ ] Password reset flow.
- [ ] User tanpa token tidak bisa akses protected route.
- [ ] User company tidak bisa akses data CIF lain.
- [ ] SYS_ADMIN bisa akses modul internal.
- [ ] CORPORATE_ADMIN tidak bisa akses modul internal.
- [ ] Create CIF otomatis membuat corporate admin.
- [ ] Create branch terkena limit plan.
- [ ] Create user terkena limit plan.
- [ ] Role protected tidak bisa dihapus.
- [ ] Last corporate admin tidak bisa dihapus.
- [ ] Branch selected mode hanya melihat branch tertentu.
- [ ] Audit log tercatat saat create/update/delete.

## 15.2 Unit tests

- [ ] `ActiveCifService`
- [ ] `PermissionService`
- [ ] `MenuService`
- [ ] `BranchAccessService`
- [ ] `AuditService`
- [ ] `NumberingSeriesService`

---

## 16. Definition of Done Foundation

Foundation dianggap selesai jika:

- [ ] Authentication JWT berjalan.
- [ ] Password reset via email berjalan.
- [ ] Active CIF context berjalan.
- [ ] CIF isolation berjalan dan sudah dites.
- [ ] SYS_ADMIN dan CORPORATE_ADMIN berjalan sesuai aturan.
- [ ] Menu sidebar dinamis berdasarkan role.
- [ ] Route permission middleware berjalan.
- [ ] UI permission helper berjalan.
- [ ] Branch access foundation berjalan.
- [ ] Plan limit `max_users` dan `max_branches` berjalan.
- [ ] Audit log universal berjalan.
- [ ] Mismatch model vs migration penting sudah dibereskan.
- [ ] Seeder awal bisa membuat environment siap pakai.
- [ ] Semua feature test foundation utama pass.

---

## 17. Urutan Kerja yang Disarankan

1. Bereskan schema mismatch.
2. Tambahkan `max_branches`, `roles.is_system`, `menus.translation_key`, dan field penting lain.
3. Rapikan seeder plan, CIF internal, role, menu, permission, sys admin.
4. Implement JWT login/logout/password reset.
5. Implement active CIF context.
6. Implement CIF management internal.
7. Implement user management scoped by CIF.
8. Implement branch management dan branch access.
9. Implement role/menu/permission management.
10. Implement dynamic sidebar dan UI helper permission.
11. Implement audit logging.
12. Implement i18n foundation.
13. Tambahkan tests.
14. Baru lanjut ke business logic ERP: master data, sales, purchase, inventory, accounting.

---

## 18. Catatan Penting untuk Business Logic Nanti

Business logic ERP jangan dimulai sebelum foundation ini stabil, karena semua modul berikut akan bergantung pada:

- `cif_id`
- `branch_id`
- active company context
- permission route/action
- audit log
- numbering series
- role/user isolation

Modul bisnis seperti sales order, delivery order, sales invoice, purchase order, goods receipt, purchase invoice, stock movement, account, journal, dan fiscal period harus selalu mengikuti pola isolation yang sama.

---

## 19. Risiko Teknis

| Risiko | Dampak | Mitigasi |
|---|---|---|
| Model dan migration mismatch | Error runtime dan data tidak konsisten | Schema reconciliation sebelum fitur |
| Permission disimpan terlalu banyak di JWT | Permission stale setelah role diubah | Load permission dari DB/cache |
| Data tidak discoped `cif_id` | Kebocoran data antar perusahaan | Middleware + global scope + tests |
| Branch access tidak divalidasi | User bisa akses branch lain | BranchAccessService + middleware |
| Role system bisa diedit bebas | Keamanan rusak | `is_system` + policy protection |
| Corporate admin terakhir terhapus | CIF tidak punya admin | Guard rule saat delete/deactivate |
| Audit menyimpan password/token | Security incident | Exclude sensitive fields |
| Menu internal muncul ke company user | Kebocoran fitur internal | `is_internal` + permission check |

---

## 20. Open Decisions

Beberapa keputusan perlu dikunci sebelum implementasi detail:

- [ ] Apakah `roles.code` tetap global unique, atau unique per CIF?
- [ ] Apakah `roles.cif_id` nullable akan ditambahkan?
- [ ] Apakah `menus` perlu kolom `is_internal`?
- [ ] Apakah permission akan dicache per request saja atau persistent cache Redis/file?
- [ ] Apakah JWT dipakai penuh untuk web Inertia, atau hybrid dengan HTTP-only cookie?
- [ ] Apakah single-device login wajib dari awal?
- [ ] Field legal/tax/contact apa saja yang benar-benar perlu ditambahkan ke `cifs`?

Rekomendasi awal:

- Tambahkan `roles.cif_id nullable`.
- Tambahkan `roles.is_system`.
- Tambahkan `menus.translation_key`.
- Tambahkan `menus.is_internal`.
- Gunakan HTTP-only secure cookie untuk JWT pada aplikasi web.
- Permission jangan disimpan permanen di JWT; load dari DB/cache.
- Single-device login bisa dibuat dari awal jika memang sudah menjadi requirement produk.

