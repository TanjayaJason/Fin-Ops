# PRD — Fin Ops (F-Ops)

**Status:** Draft untuk ditinjau  
**Versi:** 0.1  
**Tanggal:** 18 November 2026

## 1. Ringkasan

Fin Ops adalah aplikasi web multi-user untuk mencatat transaksi keuangan pribadi melalui input teks sederhana. Setiap pengguna memiliki ledger pribadi yang terisolasi. Pengguna mengetik transaksi dengan bahasa sehari-hari; backend mengubahnya menjadi data terstruktur, mengusulkan jenis dan kategori, lalu menyimpan transaksi setelah validasi. UI menampilkan ringkasan arus kas dan riwayat transaksi.

Contoh input: `kopi 28rb tadi pagi pakai gopay`. Hasil yang diusulkan: pengeluaran Rp28.000, kategori Makanan & Minuman, waktu hari ini, metode GoPay. Jika nominal atau arti input tidak jelas, sistem meminta pengguna memperbaikinya sebelum disimpan.

## 2. Masalah dan tujuan

Pencatatan manual terasa lambat karena pengguna harus membuka banyak kolom dan memilih kategori setiap kali. Bot chat saja tidak menyediakan pengalaman ringkasan dan riwayat yang mudah ditinjau atau dikoreksi.

**Tujuan MVP:**

- Mencatat pemasukan dan pengeluaran dari satu kolom input pada web.
- Mengklasifikasikan transaksi dan mengubah ungkapan nominal/tanggal menjadi nilai terstruktur.
- Meminta konfirmasi untuk hasil yang ambigu atau berkeyakinan rendah.
- Menyimpan transaksi secara konsisten dan menampilkan ringkasan serta riwayat yang dapat dikoreksi.
- Mendukung banyak pengguna dengan login Google, menjaga data tiap akun terisolasi, serta menyediakan ekspor data.

**Bukan tujuan MVP:** koneksi rekening/bank, pembayaran atau transfer, nasihat investasi, anggaran otomatis kompleks, multi-mata-uang (MVP hanya IDR), dan chatbot percakapan umum.

## 3. Pengguna dan kebutuhan

**Pengguna utama:** beberapa individu yang masing-masing ingin mencatat keuangan pribadi tanpa mengisi formulir panjang. Akun dan data tiap pengguna terpisah; berbagi wallet belum termasuk MVP.

- Sebagai pengguna, saya ingin menulis transaksi dengan bahasa alami agar pencatatan cepat.
- Saya ingin melihat kategori dan nominal sebelum data ambigu masuk ke laporan.
- Saya ingin mengubah atau menghapus transaksi yang salah.
- Saya ingin melihat pemasukan, pengeluaran, dan saldo bersih untuk rentang waktu tertentu.
- Saya ingin mengekspor data saya dan yakin orang lain tidak dapat melihatnya.

## 4. Ruang lingkup MVP

### Termasuk

1. Banyak akun pengguna dengan autentikasi Google (OAuth).
2. Dashboard dengan pemasukan, pengeluaran, arus bersih, serta transaksi terbaru untuk bulan berjalan.
3. Form input teks transaksi; hasil parser ditampilkan sebagai pratinjau yang bisa diedit.
4. Klasifikasi jenis (income/expense), nominal IDR, tanggal, kategori, catatan, dan metode pembayaran opsional.
5. Simpan, edit, hapus, pencarian/filter sederhana, dan ekspor CSV.
6. Kategori bawaan yang dapat dipilih; pengguna dapat mengubah kategori sebelum menyimpan.
7. Penanganan input tidak jelas dengan pertanyaan singkat atau form koreksi.
8. Tanya jawab Bahasa Indonesia atas data keuangan milik pengguna yang sedang login, dengan query baca saja dan jawaban bersumber dari hasil query database.

### Tidak termasuk

Integrasi bank/e-wallet, OCR struk, recurring transaction, pembagian transaksi, anggaran, target tabungan, shared wallet, multi-mata-uang, notifikasi, aplikasi mobile native.

## 5. Kebutuhan fungsional

- **FR-1 Input:** pengguna dapat memasukkan kalimat transaksi dalam Bahasa Indonesia (locale MVP `id-ID`) atau mengisi form terstruktur; mata uang MVP hanya IDR.
- **FR-2 Parsing:** sistem mengekstrak arah transaksi, nominal dalam rupiah, tanggal, deskripsi, kategori kandidat, dan metode pembayaran jika disebut.
- **FR-3 Validasi:** nominal wajib positif dan eksplisit; tanggal tanpa tahun ditafsirkan dalam tahun berjalan dan ditampilkan; nominal yang tidak dapat dipastikan tidak boleh disimpan diam-diam.
- **FR-4 Konfirmasi:** sistem menampilkan pratinjau yang bisa diedit sebelum transaksi disimpan. Kategori/jenis berkeyakinan rendah atau nominal tidak jelas harus ditanyakan.
- **FR-5 Penyimpanan:** hanya transaksi terkonfirmasi yang masuk ke ledger. Operasi simpan harus idempoten agar retry tidak membuat duplikat.
- **FR-6 Ringkasan:** dashboard menghitung total income, expense, net cash flow, dan daftar transaksi menurut periode.
- **FR-7 Koreksi:** pengguna dapat mengedit dan menghapus transaksi miliknya; ringkasan mengikuti perubahan.
- **FR-8 Ekspor:** pengguna dapat mengunduh transaksi miliknya dalam CSV.
- **FR-9 Isolasi:** setiap query dan mutasi data terikat pada pengguna yang terautentikasi melalui Google. Tanya jawab AI hanya dapat membaca data milik pengguna tersebut.

- **FR-10 Tanya jawab:** pengguna dapat bertanya dalam Bahasa Indonesia tentang transaksi dan ringkasannya. AI menerjemahkan pertanyaan menjadi intent/query terstruktur yang diizinkan, backend menjalankan query baca saja dalam scope user aktif, dan AI merangkum hasilnya. AI tidak menjalankan SQL bebas, tidak membaca akun lain, dan tidak mengubah data lewat percakapan.

## 6. Kebutuhan nonfungsional

- UI responsif dan dapat digunakan dengan keyboard; form memiliki label dan status error yang jelas.
- Respons parsing target p95 < 4 detik (sasaran awal, diukur setelah implementasi).
- Waktu dashboard target p95 < 1 detik untuk akun dengan 10.000 transaksi (sasaran awal).
- Penyimpanan uang menggunakan integer rupiah, bukan floating point.
- Rahasia tidak disimpan di repo; data sensitif tidak dicatat di log aplikasi.
- Kegagalan model/provider tidak boleh menghilangkan input pengguna; teks tetap dapat dikoreksi dan disimpan secara manual.

## 7. Aturan klasifikasi dan kepercayaan

- Gunakan kategori yang tersedia; jika tidak cocok, pilih `Lainnya` dan tandai untuk ditinjau.
- Hasil classifier adalah usulan. Pengguna tetap mengonfirmasi pratinjau sebelum penyimpanan.
- Simpan nilai confidence dan versi parser/model untuk diagnosis, bukan sebagai pengganti konfirmasi.
- Jangan menyimpulkan transfer, aset, utang, atau investasi tanpa keterangan eksplisit.
- Jangan membuat dua transaksi dari satu input kecuali pengguna secara eksplisit memasukkan beberapa item dan UI menampilkan semuanya.

## 8. Kriteria keberhasilan

- Minimal 90% dari 50 contoh uji internal menghasilkan nominal dan arah transaksi yang benar sebelum konfirmasi.
- Minimal 80% hasil kategori diterima tanpa perubahan pada set contoh yang disepakati.
- Tidak ada input ambigu yang disimpan tanpa pratinjau dan konfirmasi.
- Pengguna dapat menyelesaikan pencatatan transaksi biasa dalam kurang dari 15 detik pada uji kegunaan.
- Semua operasi ledger memiliki uji isolasi kepemilikan dan tidak memaparkan transaksi akun lain.

## 9. Risiko dan mitigasi

- **Salah tafsir nominal/kategori:** pratinjau wajib, edit cepat, confidence untuk triase, dan jalur form manual.
- **Provider AI gagal/mahal atau jawabannya keliru:** parser deterministik untuk pola sederhana, provider dibatasi di server, fallback manual.
- **Data keuangan terekspos:** autentikasi, otorisasi per query, HTTPS, minimisasi log, backup dan kebijakan retensi.
- **Scope membesar:** pertahankan batas MVP; integrasi bank dan budgeting ditunda.

## 10. Keputusan implementasi

MVP memakai React/Vite, FastAPI/Python, PostgreSQL lokal, Google OAuth, Bahasa Indonesia dan IDR. AI memakai endpoint Chat Completions yang OpenAI-compatible melalui konfigurasi `.env`.

