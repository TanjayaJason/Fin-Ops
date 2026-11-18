# Desain — Fin Ops (F-Ops)

**Status:** Usulan teknis MVP, menunggu pilihan stack final.

## 1. Prinsip desain

- Satu input utama, satu pratinjau yang mudah diperiksa, lalu simpan.
- Tidak ada nilai finansial yang disimpan dari tebakan tersembunyi.
- UI tetap berguna jika parser/LLM tidak tersedia: pengguna dapat mengoreksi form terstruktur.
- Ledger menjadi sumber kebenaran; saldo dan ringkasan dihitung dari transaksi.

## 2. Struktur UI

### Navigasi

Sidebar pada desktop dan navigasi ringkas di mobile: **Dashboard**, **Transaksi**, **Pengaturan**. Nama produk “Fin Ops” dengan subjudul “Catat keuangan dengan cepat”.

### Dashboard

- Pemilih periode dengan default bulan berjalan.
- Tiga kartu: Pemasukan, Pengeluaran, Arus Bersih.
- Input cepat di bagian atas: placeholder `Contoh: makan siang 35rb pakai tunai` dan tombol `Urai transaksi`.
- Area pratinjau hasil parser: jenis, nominal IDR, tanggal, kategori, metode pembayaran, catatan; semua bisa diedit.
- Tombol `Simpan transaksi` dan `Ubah input`.
- Daftar transaksi terbaru dengan arah transaksi, deskripsi, tanggal, kategori, dan nominal.

### Transaksi

Tabel/list dengan filter tanggal, jenis, kategori, pencarian deskripsi; aksi edit dan hapus. Dialog hapus menyebut transaksi yang akan dihapus. Ekspor CSV tersedia dari halaman ini.

### Status dan error

- Parsing: indikator `Sedang memahami transaksi…`.
- Berhasil: tunjukkan pratinjau, jangan langsung menyimpan.
- Ambigu: sorot field bermasalah dan ajukan pertanyaan spesifik (misalnya “Nominalnya 25.000 atau 250.000?”).
- Gagal: pertahankan teks, tawarkan pengisian manual.
- Kosong: panduan singkat dan contoh input.

### Aksesibilitas

Label form eksplisit, fokus keyboard terlihat, validasi tidak hanya memakai warna, format nominal dibacakan dengan jelas, dialog memiliki fokus terkelola, dan kontras mengikuti WCAG AA.

## 3. Alur utama

1. Pengguna login dan membuka Dashboard.
2. Mengetik transaksi lalu memilih `Urai transaksi`.
3. Frontend mengirim teks ke endpoint parse.
4. Orchestrator mengurai dan mengembalikan field terstruktur, confidence, dan field yang perlu klarifikasi.
5. Pengguna memeriksa/mengedit pratinjau.
6. Frontend mengirim data terkonfirmasi ke endpoint create.
7. Backend memvalidasi kembali, menulis transaksi dengan user ID dari sesi, lalu mengembalikan record tersimpan.
8. Dashboard memuat ulang ringkasan dan daftar.

## 4. Arsitektur awal

**Stack awal:** React/Vite untuk UI dan FastAPI/Python untuk API, PostgreSQL lokal untuk database, Google OAuth untuk login, session cookie yang aman. MVP menggunakan Bahasa Indonesia (`id-ID`) dan IDR saja. UI boleh multi-user, tetapi setiap user hanya melihat ledger pribadinya; shared wallet tidak termasuk.

```text
Browser UI
  ├─ POST /api/assistant/query ────> Read-only finance Q&A orchestrator
  │                                  └─ allowlisted query service → PostgreSQL
  ├─ POST /api/transactions/parse ─> Expense Orchestrator
  │                                  ├─ deterministic amount/date parsing
  │                                  ├─ optional LLM category suggestion
  │                                  └─ schema + confidence validation
  ├─ POST /api/transactions ────────> API validation + authz
  └─ GET/PATCH/DELETE /api/transactions
                    │
                 PostgreSQL
      (all records user-scoped)
```

Orchestrator berjalan di backend; kunci provider tidak pernah dikirim ke browser. Parsing dan penyimpanan adalah dua operasi berbeda. Parse tidak menulis ledger. Create menerima payload hasil tinjauan pengguna dan mengotorisasi ulang semua data.

## 5. Kontrak API yang disarankan

- `POST /api/transactions/parse` — body `{ text, locale? }`; respons `{ draft, confidence, needs_clarification[] }`. Tidak menulis DB.
- `POST /api/transactions` — body field terkonfirmasi `{ type, amount_minor, currency, occurred_at, category_id, description?, payment_method?, idempotency_key }`; membuat record untuk user aktif.
- `GET /api/transactions?from=&to=&type=&category=&q=&cursor=` — hanya data user aktif.
- `PATCH /api/transactions/:id` — edit field yang diizinkan, hanya milik user aktif.
- `DELETE /api/transactions/:id` — hapus milik user aktif.
- `GET /api/summary?from=&to=` — agregat dari ledger user aktif.
- `GET /api/transactions/export.csv?from=&to=` — CSV dengan escaping yang aman.

Semua payload divalidasi server-side; tipe transaksi enum (`income`, `expense`), ID kategori harus milik/tersedia untuk user, nilai amount integer > 0, tanggal valid, currency `IDR` pada MVP.

## 6. Model data awal

### `users`

`id UUID PK`, `google_subject UNIQUE`, `email`, `display_name`, `created_at`. Google subject adalah identitas stabil; email dapat berubah.

### `categories`

`id UUID PK`, `user_id UUID NULL` (NULL untuk kategori bawaan), `name`, `type` (`income`/`expense`/`both`), `created_at`. Nama kategori unik per owner/type.

### `transactions`

`id UUID PK`, `user_id UUID NOT NULL`, `type`, `amount_minor BIGINT NOT NULL`, `currency CHAR(3)`, `occurred_at TIMESTAMPTZ`, `category_id UUID`, `description TEXT`, `payment_method TEXT NULL`, `source_text TEXT NULL`, `classifier_confidence NUMERIC NULL`, `classifier_version TEXT NULL`, `idempotency_key TEXT`, `created_at`, `updated_at`.

Index awal: `(user_id, occurred_at DESC)`, `(user_id, type, occurred_at DESC)`, unique `(user_id, idempotency_key)`. Tambahkan FK dan aturan cascade sesuai kebijakan penghapusan akun. Untuk IDR, `amount_minor` menyimpan jumlah rupiah utuh; nama kolom dipertahankan untuk konsistensi model uang lintas mata uang kelak.

## 7. Bentuk hasil orchestrator

```json
{
  "type": "expense",
  "amount_minor": 28000,
  "currency": "IDR",
  "occurred_at": "2026-11-18T00:00:00+07:00",
  "category_suggestion": "Makanan & Minuman",
  "description": "kopi",
  "payment_method": "GoPay",
  "confidence": { "type": 0.99, "amount": 0.98, "category": 0.86 },
  "needs_clarification": []
}
```

Schema response ketat. Field yang tidak dikenal ditolak/dibuang, nominal dinormalisasi tanpa floating point, tanggal memakai zona waktu akun (default Asia/Jakarta sampai ada setting), kategori harus cocok ke katalog, dan teks input diperlakukan sebagai data yang tidak tepercaya. Orchestrator tidak dapat memanggil tool SQL arbitrer atau melakukan write sendiri.

## 8. Keamanan dan privasi

- Ambil `user_id` hanya dari sesi yang diverifikasi di server, jangan dari request body.
- Semua query ledger memasukkan filter `user_id`; pertimbangkan row-level security pada PostgreSQL.
- Enkripsi transport; rahasia via environment/secrets manager; rate-limit endpoint parse.
- Jangan menyimpan prompt/provider payload sensitif di log. Tentukan retensi `source_text` dan sediakan ekspor/hapus akun.
- Batasi ukuran input, validasi output LLM, dan anggap teks transaksi sebagai untrusted prompt content.
- CSV di-escape agar konten transaksi tidak dieksekusi sebagai formula spreadsheet.

## 9. Tanya jawab AI atas database

AI memiliki dua peran produk yang jelas: (1) menyusun draft transaksi dari input, dan (2) menjawab pertanyaan tentang transaksi pengguna. Untuk Q&A, model hanya memilih intent dari katalog yang diizinkan (mis. total pengeluaran per kategori/periode, transaksi terbaru, perbandingan bulan), bukan menulis SQL bebas. Backend memvalidasi parameter, menambahkan `user_id` dari sesi, menjalankan query read-only dengan timeout dan limit, lalu memberikan hasil agregat minimum kepada model untuk diringkas. Jawaban harus menyebut periode yang dipakai; jika data tidak cukup, AI menyatakannya. Tidak boleh membaca data pengguna lain, mengubah transaksi, mengirim query DDL/DML, atau mengklaim angka yang tidak terdapat di hasil query.

`.env.example` memuat nama variabel tanpa secret; `.env` masuk `.gitignore`. Dokumentasikan tiap provider setting, model, dan endpoint base URL. Validasi konfigurasi saat startup dan beri error yang jelas saat belum tersedia. Jangan meneruskan `DATABASE_URL` atau kredensial database ke provider AI.

## 10. Tahapan implementasi

1. Bekukan jawaban pertanyaan terbuka PRD dan pilih stack/auth/hosting.
2. Bangun shell web, auth, skema/migrasi, dan query user-scoped.
3. Bangun input manual + parse draft + konfirmasi + create idempoten.
4. Tambahkan dashboard, pencarian, edit/delete, ekspor.
5. Uji kegunaan, akurasi pada dataset contoh, aksesibilitas, serta isolasi data sebelum rilis.

## 11. Keputusan yang sudah ditetapkan dan terbuka

**Sudah ditetapkan:** multi-user tanpa shared wallet, Bahasa Indonesia dan IDR untuk MVP, Google login, PostgreSQL, hosting lokal sementara, serta provider AI configurable dan OpenAI-compatible.

**Masih terbuka:** aturan retensi source text, konfigurasi timezone per user (default Asia/Jakarta), dan apakah kategori bisa dikustomisasi. Jangan menulis implementasi yang mengunci keputusan terbuka tanpa kebutuhan.




