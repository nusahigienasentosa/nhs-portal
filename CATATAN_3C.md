# CATATAN_3C.md — Batch 3C: Payroll

Branch: `batch-3c-payroll` (dari `main`, belum di-merge/push).
Status: kode UI selesai, SQL cuma di-output (belum dijalankan), belum
di-test di browser beneran (lihat poin "Testing" di bawah).

## Ringkasan yang dibangun

- Nav baru **Payroll** (role `nhs` only, sejajar Karyawan/Absensi).
- Form "Hitung gaji baru": pilih karyawan aktif + periode (dari–sampai
  tanggal) → tombol "Hitung dari absensi" → ambil data `attendance`
  di rentang itu, hitung hari masuk = jumlah baris yang `jam_masuk`
  terisi, kali `tarif_harian` karyawan → gaji kotor.
- Potongan: baris dinamis bebas (keterangan + jumlah), bisa tambah
  berapa pun, totalnya dikurangkan dari gaji kotor → gaji bersih.
- Simpan sebagai **draft** dulu (belum masuk ledger).
- **Finalisasi** (ada konfirmasi eksplisit lewat dialog): baru di titik
  ini payroll diposting ke Finance sebagai transaksi `expense` dengan
  `category: "Gaji Karyawan"` — kategori ini terpisah dari `Prive /
  Owner Draw` dan dari kategori PO (`Bahan Produksi` dll), sesuai
  yang diminta. Payroll yang sudah final dikunci di UI (gak ada tombol
  edit/hapus lagi).
- Data demo: nambah seed `demoAttendance` (absensi Juni 2026 buat 3
  karyawan demo) dan `demoPayroll=[]`, supaya alur "Hitung dari
  absensi" langsung ada hasil kalau dites di mode demo — gak nyentuh
  data attendance asli (ini array in-memory demo doang, cuma aktif
  kalau `LIVE=false`, misal buka dengan `#demo` di URL).
- `SQL_3C.md`: skema tabel `payroll` + RLS (nhs only) + trigger kunci
  opsional. Belum dijalankan.

## Keputusan desain yang butuh approval lu

1. **Snapshot vs live-join**: `employee_nama` & `tarif_harian` disimpan
   sebagai snapshot di baris payroll (bukan di-join live ke tabel
   `employees`). Efeknya: kalau nama/tarif karyawan diedit belakangan,
   histori payroll lama tetap nunjukin angka pas saat dihitung, gak
   ikut berubah. Saya anggap ini yang benar buat catatan gaji, tapi
   konfirmasi ya.
2. **Definisi "hari masuk"**: saya hitung dari baris `attendance` yang
   `jam_masuk` terisi, TANPA syarat `jam_keluar` juga harus terisi.
   Jadi kalau karyawan clock-in tapi lupa/gak sempat clock-out, hari
   itu tetap dihitung masuk. Kalau maunya harus lengkap masuk+keluar
   baru dihitung, kasih tau, gampang diubah satu baris kode.
3. **Tanggal posting ke ledger**: saya pakai tanggal hari ini (saat
   tombol Finalisasi diklik), BUKAN tanggal akhir periode gaji. Kalau
   maunya pakai `period_end` sebagai tanggal transaksi ledger, kasih
   tau — juga gampang diubah.
4. **Payroll final terkunci hanya di UI**, bukan di level database.
   Ada draft trigger opsional di `SQL_3C.md` section 3 buat ngunci
   beneran di DB (nolak UPDATE/DELETE kalau `status='final'`), tapi
   belum saya rekomendasikan jalanin — perlu keputusan lu dulu apakah
   mau seketat itu (soalnya kalau suatu saat butuh koreksi payroll
   yang salah, dengan trigger itu satu-satunya jalan adalah hapus
   trigger dulu manual, gak bisa lewat UI sama sekali).
5. **[Update] Alur "Batalkan Finalisasi" sudah dibuat, pola reversing
   entry.** Tombol muncul di baris payroll `status='final'` (role
   `nhs` only, karena view Payroll sendiri cuma kebuka buat `nhs`).
   Klik → konfirmasi eksplisit → sistem BIKIN transaksi pembalik baru
   di `finance_txns` (`type:'income'`, kategori `Gaji Karyawan`,
   deskripsi diawali "PEMBALIK —", nominal = gaji_bersih), lalu
   payroll balik ke `status='draft'` dan `finance_txn_id`/
   `finalized_at` dikosongkan. Transaksi gaji ASLI (yang expense)
   TIDAK dihapus/diedit — jejak lengkap tetap ada di ledger (gaji
   asli + pembaliknya). Lihat `doBatalkanFinalisasi()` di
   `index.html`. Belum di-test di browser beneran (lihat poin
   Testing).

## Titik risiko

- **RLS policy `payroll` saya tulis berdasarkan pola tebakan** (join ke
  `profiles.role = 'nhs'`), karena saya nggak punya akses baca skema
  Supabase live dan nggak boleh run SQL apapun ke sana malam ini. Kalau
  pola RLS yang beneran dipakai di tabel `employees`/`attendance` yang
  sudah ada BEDA dari ini, policy di `SQL_3C.md` harus disesuaikan dulu
  sebelum dijalankan — kalau nggak, bisa ke-generate lebih longgar
  (bocor) atau lebih ketat (nhs sendiri malah kegembok) dari yang
  seharusnya. **Ini yang paling penting dicek sebelum run SQL.**
- Tipe kolom (`employee_id uuid`, `finance_txn_id uuid`) itu asumsi.
  Kalau skema asli pakai tipe lain, sesuaikan dulu.
- Saya TIDAK menyentuh tabel `employees`, `attendance`, atau
  `finance_txns` yang sudah ada — cuma nambah tabel baru (`payroll`)
  dan kolom baru di tabel baru itu sendiri. Jadi data lama aman,
  gak ada risiko kehilangan/berubah data existing dari perubahan ini.
- Belum ada rate-limit/duplikasi guard — kalau admin klik "Hitung" dua
  kali buat karyawan+periode yang sama, bisa kesimpan dua draft payroll
  yang overlap. Gak ada validasi "periode ini udah pernah dihitung
  buat karyawan ini" di versi sekarang. Kalau mau, saya bisa tambahin
  cek itu, tapi butuh keputusan: boleh overlap (misal buat koreksi)
  atau harus dicegah keras?

## Testing

Saya **tidak berhasil buka browser beneran** di environment ini (gak
ada tool otomasi browser yang ke-attach ke sesi ini) — jadi verifikasi
dilakukan lewat:
- Review manual logic per baris (perhitungan hari masuk, gaji kotor,
  potongan, gaji bersih, posting ke ledger).
- Cross-check pola kode baru vs pola existing (PO, karyawan, absensi,
  finance) biar konsisten.
- Sanity check kurung `{}()[]` seimbang di seluruh file (tanda nggak
  ada typo struktural yang jelas).

Ini BUKAN pengganti tes UI beneran. Tolong besok dicoba manual:
1. Buka `index.html` (atau URL live-nya) dengan tambahan `#demo` di
   URL — ini maksa mode demo, gak nyentuh Supabase sama sekali.
2. Login `admin@nhs.co.id` / `demo123`.
3. Buka menu **Payroll** → pilih **Budi Santoso**, periode
   `2026-06-01` s/d `2026-06-30` → klik **Hitung dari absensi**.
   Ekspektasi: 22 hari masuk × Rp120.000 = **Rp2.640.000** gaji kotor.
4. Coba tambah potongan (mis. "Kasbon" Rp200.000) → cek gaji bersih
   ke-update otomatis jadi Rp2.440.000.
5. Simpan draft → cek muncul di tabel riwayat dengan badge "Draft".
6. Klik **Finalisasi** → konfirmasi dialog → cek badge berubah jadi
   "Final · di ledger".
7. Buka menu **Finance** → cek ada baris baru kategori **Gaji
   Karyawan** senilai gaji bersih, dan saldo kas berkurang segitu.
8. Login sebagai `staff@nhs.co.id` atau `purchasing@foodex.co.id` →
   pastikan menu Payroll SAMA SEKALI gak muncul di sidebar (nav_payroll
   cuma ada di grup `nhs`).

## Belum dikerjakan / di luar skop malam ini

- Export/print slip gaji per karyawan (belum diminta, belum dibuat).
- Payroll massal (hitung semua karyawan sekaligus untuk satu periode)
  — sekarang satu-satu per karyawan per klik "Hitung".
- Reverse/void payroll final (lihat poin risiko di atas).

Lanjut ke Tugas 2 (Batch 4) setelah ini.
