# SQL_4.md — Batch 4: Finance sub-filter + Supplier master + PO split

Status: **belum dijalankan sama sekali.** Cuma output buat direview &
dijalankan manual di Supabase SQL Editor. Assistant tidak menyentuh
Supabase malam ini.

## Konteks penting: cek dulu sebelum run

Tabel `suppliers` **kemungkinan besar sudah ada** di database live —
UI CRUD-nya (modal tambah/ubah supplier, dropdown di form PO) sudah
ada duluan di `index.html` sebelum batch ini (kode `db.suppliers()`,
`db.addSup()`, dll sudah dipakai di halaman PO). Batch 4 di sisi kode
cuma MEMINDAHKAN tampilan supplier ke menu sendiri (bukan bikin fitur
baru), jadi kemungkinan tabelnya sudah eksis.

**Sebelum run apapun di bawah**: cek dulu di Supabase apakah tabel
`suppliers` sudah ada.
- Kalau **sudah ada**: `create table if not exists` di section 1 aman
  dilewatin/no-op, tapi tetap cek apakah RLS di section 2 SUDAH
  terpasang atau belum — kalau belum, itu berarti selama ini tabel
  `suppliers` mungkin tanpa proteksi row-level (bisa diakses role
  manapun yang punya token valid). Section 2 penting dijalankan kalau
  ternyata belum ada RLS di tabel ini.
- Kalau **belum ada**: jalankan section 1 & 2 seperti biasa.

Saya nggak punya akses baca skema live, jadi nggak bisa mastiin sendiri
mana yang berlaku — mohon dicek manual dulu.

## 1. Tabel `suppliers` (idempotent — aman kalau sudah ada)

```sql
create table if not exists public.suppliers (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  contact text,
  phone text,
  email text,
  note text,
  created_at timestamptz not null default now()
);

create index if not exists suppliers_name_idx on public.suppliers(name);
```

Kolom di atas mengikuti field yang sudah dipakai UI (`supName`,
`supContact`, `supPhone`, `supEmail`, `supNote`). Kalau skema live
ternyata sudah pakai nama kolom lain, JANGAN run — sesuaikan dulu,
atau balikin ke saya.

## 2. RLS — cuma role `nhs` yang boleh akses

```sql
alter table public.suppliers enable row level security;

create policy "nhs full access suppliers"
  on public.suppliers
  for all
  using (
    exists (select 1 from public.profiles p where p.id = auth.uid() and p.role = 'nhs')
  )
  with check (
    exists (select 1 from public.profiles p where p.id = auth.uid() and p.role = 'nhs')
  );
```

Pola sama persis kayak yang saya tulis buat `payroll` di `SQL_3C.md`
(dan sama seperti tebakan pola untuk `employees`/`attendance`) — kalau
pola RLS live yang beneral BEDA, sesuaikan konsisten di semua tabel,
jangan cuma di sini.

## Tidak ada perubahan skema lain di batch ini

- **Finance ledger**: sub-tab/filter kategori di halaman Keuangan itu
  murni perhitungan sisi klien (JavaScript), TIDAK ada tabel baru,
  TIDAK ada kolom baru, TIDAK mengubah `finance_txns` sama sekali.
  Satu ledger tetap satu ledger — filter cuma nyaring baris yang
  ditampilkan, saldo kas tetap dihitung dari SEMUA transaksi.
- **PO → Supplier reference**: sudah pakai `supplier_id` (foreign key
  ke `suppliers.id`) sejak sebelum batch ini — PO tidak pernah/tidak
  lagi ketik nama supplier manual. Tidak ada migrasi data diperlukan.
- **Tidak ada tabel yang dihapus atau kolom yang di-drop/di-rename.**

## Urutan run yang disarankan

1. Cek dulu manual apakah `suppliers` & RLS-nya sudah ada di Supabase.
2. Kalau belum ada tabelnya: jalankan section 1.
3. Kalau RLS belum terpasang (baik tabel baru maupun lama): jalankan
   section 2.
