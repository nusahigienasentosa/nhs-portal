# CATATAN_4.md — Batch 4: Finance sub-filter + Supplier master + PO split

Branch: `batch-4-finance-supplier` (dari `main` langsung, TIDAK numpuk
di atas `batch-3c-payroll` — dua branch ini independen, sesuai
instruksi). Status: kode UI selesai, SQL cuma di-output (belum
dijalankan), belum di-test di browser beneran (lihat poin "Testing").

## Temuan penting sebelum mulai kerja

Pas saya baca kode yang sudah ada di `main`, ternyata **dua dari tiga
item scope batch ini sudah terpenuhi sebelum saya sentuh apapun**:

1. **Supplier master (CRUD)** — sudah ada. Modal tambah/ubah supplier,
   fungsi `db.suppliers()/addSup/updSup/delSup`, semua sudah eksis di
   `main` (kelihatannya dibangun bareng batch PO sebelumnya, walau gak
   ada SQL/catatan tertulis buat itu). Yang belum ada adalah SQL migrasi
   tertulisnya — itu yang saya output ke `SQL_4.md`.
2. **PO reference supplier dari tabel `suppliers`** — sudah ada. Form
   PO (`poSup`) itu `<select>` yang isinya daftar `suppliers`, PO
   nyimpen `supplier_id` (FK) + `supplier_name` (snapshot nama). PO
   TIDAK PERNAH ketik nama supplier manual di kode yang saya baca.

Jadi kerjaan malam ini buat batch 4 fokus ke:
- **Motong tampilan** Supplier keluar dari halaman PO jadi menu sendiri
  (task minta "PO: pisahin tampilannya" — sebelumnya daftar supplier &
  daftar PO nempel di satu halaman yang sama).
- **Filter kategori di Finance** (yang beneran belum ada sebelumnya).
- Nulis SQL + RLS buat `suppliers` yang mungkin belum pernah
  didokumentasikan formal.

## Ringkasan yang dibangun

### Finance — filter kategori, tetap satu ledger
- Tab kategori dinamis di atas tabel ledger (Semua / Modal Awal /
  per-kategori PO / Prive-Owner Draw / kategori manual / dst — daftar
  kategori diambil otomatis dari data yang ada, bukan hardcode).
- Klik tab cuma nyaring BARIS yang ditampilkan di tabel. Kartu "Saldo
  Kas Sekarang" dan statistik lain di atas TETAP dihitung dari SELURUH
  transaksi — sesuai instruksi "satu ledger, satu saldo, kategori cuma
  lensa". Kolom "Saldo" per baris juga tetap nunjukin saldo berjalan
  yang sebenarnya (dari urutan transaksi lengkap), bukan saldo
  ter-filter — biar gak menyesatkan seolah ada "saldo per kategori".
- Kategori pengeluaran PO tadinya digeneralisir jadi cuma label `"PO"`
  di ledger (semua PO nyampur satu label). Saya ubah jadi
  `"PO · <kategori PO-nya>"` (Bahan Produksi/Operasional/Konsumsi/
  Lainnya — field yang PO-nya sendiri udah punya dari batch 2B), biar
  filternya beneran berguna motong per jenis pengeluaran, bukan cuma
  motong "PO vs bukan-PO". Ini perubahan tampilan doang, gak ada
  perubahan skema/data.

### Supplier — menu sendiri
- Nav baru **Supplier** (nhs only), berisi CRUD supplier yang
  sebelumnya nempel di halaman PO.
- Halaman **PO** sekarang cuma isinya daftar PO + tombol "+ PO baru".
  Kalau belum ada supplier terdaftar, ada notice ngarahin ke menu
  Supplier dulu.
- Semua redirect setelah simpan/hapus supplier saya arahkan ke
  `go('supplier')` (sebelumnya `go('po')`).

### SQL
- `SQL_4.md`: skema `suppliers` (idempotent, `create table if not
  exists`) + RLS nhs-only. Ditulis dengan asumsi tabel ini mungkin
  SUDAH ada live (karena UI-nya udah jalan) — saya kasih catatan
  eksplisit di file itu buat dicek dulu, khususnya soal RLS-nya udah
  terpasang apa belum.

## Keputusan desain yang butuh approval lu

1. **Label kategori PO di ledger saya ubah** dari `"PO"` jadi
   `"PO · <kategori>"`. Ini keputusan kecil yang saya ambil sendiri
   karena tanpa ini fitur filter kategori jadi kurang berguna (PO
   nyumbang mayoritas pengeluaran tapi semua ke-lump satu label). Kalau
   lu maunya tetap label `"PO"` polos tanpa breakdown kategori,
   gampang dibalikin.
2. **Supplier jadi nav item sendiri** (bukan tab di dalam halaman PO).
   Saya pilih ini karena "pisahin tampilannya" saya baca sebagai pisah
   halaman, konsisten sama pola Karyawan/Absensi yang juga dipisah
   walau terkait. Kalau maunya cuma dipisah jadi 2 tab DALAM 1 halaman
   PO (bukan 2 menu sidebar terpisah), kasih tau — itu ubahan kecil.
3. **Urutan nav**: saya taruh "Supplier" tepat sebelum "PO" di sidebar
   (Inventory → Supplier → PO → Finance → Karyawan → Absensi →
   Tagihan). Confirm urutan ini oke.

## Titik risiko

- **Saya nggak bisa mastiin apakah tabel `suppliers` di Supabase live
  sudah punya RLS atau belum**, karena nggak boleh run SQL/baca skema
  malam ini. Kalau ternyata BELUM ada RLS di tabel itu selama ini,
  berarti ini bukan cuma "tambahan dari batch 4" tapi **celah yang
  sudah ada dari batch sebelumnya** — data supplier (kontak, dll) bisa
  diakses siapapun yang punya token valid (bukan cuma nhs), sejak
  tabelnya dibikin. Prioritaskan cek ini besok pagi.
- SQL di `SQL_4.md` pakai pola RLS yang sama kayak `SQL_3C.md` (tebakan
  berbasis `profiles.role`). Risiko sama seperti dicatat di
  `CATATAN_3C.md` — perlu diverifikasi cocok sama pola RLS yang beneran
  dipakai di tabel lain sebelum dijalankan.
- Perubahan `po()`/`supplier()`/`fin()` di `index.html` murni
  refactor tampilan + penambahan filter di sisi klien. Tidak ada
  fungsi `db.*` yang saya hapus, tidak ada data yang dihapus/diubah —
  cuma nambah 1 fungsi render baru (`V.supplier`) dan motong isi
  `V.po`, plus 1 variable filter baru (`finCatFilter`) di `V.fin`.

## Testing

Sama seperti batch 3C — saya nggak punya akses browser beneran di
environment ini, jadi verifikasi cuma lewat review kode manual +
cross-check pola existing + sanity check kurung `{}()[]` seimbang di
seluruh file. Tolong dites manual besok:

1. Buka index.html dengan `#demo` di URL → login `admin@nhs.co.id` /
   `demo123`.
2. Sidebar harus nunjukin menu **Supplier** terpisah dari **PO**.
3. Buka **Supplier** → harus ada 1 supplier demo ("Heaven Chemical")
   dengan tombol edit/hapus berfungsi seperti sebelumnya (cuma
   pindah halaman).
4. Buka **PO** → harus cuma nunjukin daftar PO (1 PO demo:
   PO-2026-001), gak ada lagi strip supplier di halaman ini. Coba
   "+ PO baru" → dropdown supplier di modal tetap kepilih dari
   `suppliers` (bukan free text).
5. Buka **Finance** → harus ada baris tab kategori di atas tabel
   ledger (mis. "Semua", "PO · Bahan Produksi", dll). Klik salah satu
   tab → tabel ke-filter, tapi kartu "Saldo Kas Sekarang" di atas
   TIDAK berubah nilainya.
6. Login sebagai `staff@nhs.co.id` / `purchasing@foodex.co.id` →
   pastikan menu Supplier/PO/Finance tetap gak muncul (unchanged dari
   sebelumnya, cuma nhs).

## Belum dikerjakan / di luar skop malam ini

- Auto-kirim PO ke email supplier (disebut di teks UI sebagai
  "Fase 3", belum diminta di batch ini, gak disentuh).
- Import/export supplier massal.
- Grafik/breakdown visual per kategori Finance (cuma tab filter teks,
  belum ada chart).

Lanjut ke Tugas 3 (Batch 5 — diagnosa doang) setelah ini.
