# SQL_3C.md — Batch 3C: Payroll

Status: **belum dijalankan sama sekali.** Semua statement di bawah ini
cuma OUTPUT buat direview & dijalankan manual oleh admin di Supabase
SQL Editor. Assistant tidak menyentuh Supabase malam ini (sesuai aturan).

## Asumsi skema (WAJIB diverifikasi dulu sebelum run)

Saya nggak punya akses baca ke skema live Supabase, jadi SQL di bawah
disusun berdasarkan pola yang kepakai di `index.html` (nama tabel/kolom
yang dipanggil lewat `sbGet`/`sbWrite`, dan bentuk data demo). Sebelum
run, cek dulu di Supabase:

- `employees.id` — saya asumsikan `uuid`. Kalau ternyata `bigint`/`text`,
  ubah tipe `employee_id` di tabel `payroll` di bawah biar match.
- `finance_txns.id` — saya asumsikan `uuid`. Kalau beda, ubah FK
  `finance_txn_id`.
- `profiles(id, role)` — saya asumsikan tabel `profiles` dengan kolom
  `role` (nilainya `'nhs'` / `'staff'` / `'client'` / `'kiosk'`) yang
  dipakai buat cek hak akses, karena pola ini konsisten sama gimana
  `employees`/`attendance` cuma di-fetch pas `role==='nhs'` di kode app.
  **Ini poin paling penting buat dicek** — kalau RLS tabel `employees`/
  `attendance` yang udah ada polanya beda (misal pakai custom JWT claim,
  bukan join ke `profiles`), policy `payroll` di bawah harus disesuaikan
  supaya konsisten, jangan sampai lebih longgar atau malah lebih ketat
  dari yang lain.

Kalau salah satu asumsi di atas meleset, JANGAN paksa run — sesuaikan
dulu manual, atau balikin ke saya buat saya perbaiki skripnya.

## 1. Tabel `payroll`

```sql
create table if not exists public.payroll (
  id uuid primary key default gen_random_uuid(),
  employee_id uuid not null references public.employees(id),
  employee_nama text not null,                 -- snapshot nama karyawan saat dihitung
  period_start date not null,
  period_end date not null,
  hari_masuk integer not null default 0,
  tarif_harian numeric not null default 0,      -- snapshot tarif harian saat dihitung
  gaji_kotor numeric not null default 0,
  potongan jsonb not null default '[]'::jsonb,  -- fleksibel: [{"label":"Kasbon","amount":100000}, ...]
  total_potongan numeric not null default 0,
  gaji_bersih numeric not null default 0,
  status text not null default 'draft' check (status in ('draft','final')),
  finance_txn_id uuid references public.finance_txns(id),  -- diisi pas finalisasi
  finalized_at timestamptz,
  created_at timestamptz not null default now(),
  created_by uuid references auth.users(id),
  constraint payroll_period_valid check (period_start <= period_end),
  constraint payroll_gaji_bersih_non_negative check (gaji_bersih >= 0)
);

create index if not exists payroll_employee_id_idx on public.payroll(employee_id);
create index if not exists payroll_period_idx on public.payroll(period_start, period_end);
create index if not exists payroll_status_idx on public.payroll(status);
```

Catatan desain:
- `employee_nama` dan `tarif_harian` di-snapshot pas dihitung (bukan
  di-join live ke `employees`), biar histori payroll gak ikut berubah
  kalau nama/tarif karyawan diedit belakangan.
- `potongan` disimpan sebagai JSONB array bebas (label + amount) —
  gak dibikin tabel terpisah, karena skopenya cuma "field fleksibel,
  kasbon dll" tanpa kebutuhan tracking per-jenis potongan.
- Tidak ada `update`/`delete` policy khusus yang membedakan status
  `final` di level SQL — penguncian "final gak bisa dihapus/diedit"
  cuma ditegakkan di level UI (tombol disembunyikan). Kalau mau
  dikunci beneran di DB (misal via trigger yang menolak UPDATE/DELETE
  saat `status='final'`), itu belum saya buat — lihat CATATAN_3C.md
  poin risiko.

## 2. RLS — cuma role `nhs` yang boleh akses

```sql
alter table public.payroll enable row level security;

create policy "nhs full access payroll"
  on public.payroll
  for all
  using (
    exists (
      select 1 from public.profiles p
      where p.id = auth.uid() and p.role = 'nhs'
    )
  )
  with check (
    exists (
      select 1 from public.profiles p
      where p.id = auth.uid() and p.role = 'nhs'
    )
  );
```

## 3. (Opsional) trigger kunci baris final

Kalau mau payroll yang sudah `status='final'` bener-bener gak bisa
diubah/dihapus siapapun (termasuk lewat SQL editor manual, bukan cuma
disembunyikan di UI), ini contoh trigger opsional — **belum saya
rekomendasikan run otomatis**, ini draft buat didiskusikan dulu:

```sql
create or replace function public.payroll_block_final_mutation()
returns trigger as $$
begin
  if OLD.status = 'final' then
    raise exception 'Payroll % sudah final, tidak bisa diubah/dihapus.', OLD.id;
  end if;
  return coalesce(NEW, OLD);
end;
$$ language plpgsql;

create trigger payroll_lock_final
  before update or delete on public.payroll
  for each row execute function public.payroll_block_final_mutation();
```

## Urutan run yang disarankan

1. Section 1 (buat tabel)
2. Section 2 (aktifkan RLS + policy)
3. Section 3 — opsional, cuma kalau disetujui setelah baca CATATAN_3C.md
