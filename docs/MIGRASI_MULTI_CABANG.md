# Rencana Migrasi POS Multi-Cabang

Status: cabang kerja `feature/multi-cabang-pos`. Dokumen ini tidak mengubah produksi.

## Cabang
- `PV001` — Pam MIM: stok awal dari stok opname fisik.
- `TV002` — Toko Miko: stok lama dipetakan ke cabang ini.
- `PV002` — Pam Miko: stok awal dari stok opname fisik.

## Aturan migrasi
1. Jangan menggandakan stok lama ke semua cabang.
2. Pertahankan harga, subtotal, diskon, nota, dan tanggal historis sebagaimana tercatat.
3. Jangan otomatis menetapkan cabang penjualan historis tanpa bukti. Catat data yang belum diketahui sebagai belum terpetakan.
4. Setiap transaksi baru wajib membawa identitas cabang yang ditetapkan server dari sesi pengguna.
5. Frontend menyembunyikan pilihan cabang untuk kasir, tetapi backend tetap wajib memvalidasi peran dan cabang untuk setiap operasi.
6. Transaksi offline menyimpan cabang saat transaksi dibuat; sinkronisasi ulang tidak boleh mengganti cabang asal.
7. Permintaan ulang dengan idempotency key yang sama tidak boleh menerapkan perubahan stok dua kali.
8. Perubahan stok (penjualan, barang masuk, retur, koreksi, transfer) harus mencatat mutasi dan menjaga konsistensi dengan transaksi sumber.

## Skema target (rancangan)
- `Cabang(cabang_id, nama_cabang, status)`
- Tambahan penugasan cabang pada pengguna; Owner dapat melihat semua cabang sesuai hak akses.
- `Stok_Cabang(cabang_id, id_barang, stok, updated_at, updated_by)`, kunci unik pasangan cabang + barang.
- `Mutasi_Stok(mutasi_id, cabang_id, id_barang, jenis, qty_delta, sumber_id, idempotency_key, petugas, waktu)`.
- `Transfer_Header(transfer_id, cabang_asal, cabang_tujuan, status, dibuat_oleh, dikirim_pada, diterima_oleh, diterima_pada, idempotency_key)`.
- `Transfer_Detail(transfer_id, id_barang, qty_kirim, qty_terima, selisih, keterangan)`.
- Transaksi penjualan baru menyimpan `cabang_id`. Kolom historis yang ada tidak ditimpa.

## Status transfer
`DRAFT -> IN_TRANSIT -> RECEIVED`; jika jumlah berbeda, simpan jumlah diterima dan selisih. Pengiriman dan penerimaan harus idempotent. Stok keluar dicatat saat pengiriman dan stok masuk saat penerimaan; stok dalam perjalanan tetap dapat diaudit.

## Hak akses wajib di backend
- Owner: akses lintas cabang sesuai otorisasi.
- Admin/User/Kasir: hanya cabang yang ditetapkan.
- Cabang dari payload klien tidak dipercaya; server memvalidasi sesi/otorisasi dan menolak cabang yang tidak diizinkan.
- Jangan menganggap penyembunyian menu pada HTML sebagai kontrol keamanan.

## Rencana urutan implementasi
1. Inventarisasi endpoint GAS dan kontrak data frontend.
2. Buat skema cabang dan migrasi yang idempotent tanpa menghapus data lama.
3. Implementasi otorisasi server dan penugasan cabang.
4. Implementasi pembacaan stok dan semua mutasi berdasarkan cabang.
5. Implementasi transaksi offline dengan cabang tetap dan idempotency key.
6. Implementasi alur transfer, penerimaan, selisih, dan audit trail.
7. Tambahkan filter laporan Owner dan pembatasan data untuk cabang lain.
8. Jalankan uji regresi pada salinan/staging dan rekonsiliasi hasil.
9. Deployment hanya setelah fungsi GAS dan frontend diuji bersama serta URL deployment diverifikasi.

## Uji penerimaan
- Kasir tidak bisa membaca/mengubah stok cabang lain dengan mengubah payload.
- Penjualan hanya mengurangi stok cabang asal.
- Barang masuk, retur, dan koreksi mengubah stok cabang yang benar.
- Transaksi offline mempertahankan cabang dan tidak ganda saat retry.
- Transfer tidak menggandakan stok; penerimaan parsial dan selisih tercatat.
- Owner dapat memfilter laporan per cabang; pengguna cabang tidak melihat data cabang lain.
- Data historis dan harga historis tidak berubah setelah migrasi.
- Backup bisa dipulihkan dan rekonsiliasi jumlah sebelum/sesudah sesuai.

## Batasan saat ini
Spreadsheet aktif, hak akses deployment GAS, dan versi GAS yang benar-benar terpasang belum diverifikasi lewat konektor langsung. Karena itu dokumen ini adalah spesifikasi kerja; belum ada migrasi data atau deployment produksi yang dijalankan.
