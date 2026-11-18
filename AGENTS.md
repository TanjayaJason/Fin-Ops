# Panduan agent repo Fin Ops

- Baca `docs/PRD.md` dan `docs/DESIGN.md` sebelum mengubah perilaku produk atau skema.
- Anggap semua angka uang sebagai integer rupiah; jangan memakai floating point untuk aritmetika finansial.
- Operasi parsing tidak boleh menulis transaksi. Hanya endpoint penyimpanan setelah konfirmasi pengguna yang boleh membuat ledger entry.
- Ambil identitas user dari sesi terverifikasi. Setiap baca/ubah/hapus transaksi harus dibatasi ke pemiliknya.
- Perlakukan teks transaksi dan keluaran model sebagai input tak tepercaya; validasi di server dengan schema eksplisit.
- Pertahankan jalur input manual saat parser gagal dan jangan mengarang nilai wajib yang tidak disebutkan.
- Jangan menambahkan integrasi bank, pembayaran, atau nasihat finansial tanpa perubahan scope produk yang disetujui.
- Jangan menyimpan secret, data transaksi contoh riil, atau payload sensitif ke repository/log.
- Perbarui dokumen terkait jika keputusan produk atau API berubah.
