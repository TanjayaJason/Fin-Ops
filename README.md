# Fin Ops

Catat transaksi dengan kalimat biasa, periksa hasilnya, lalu simpan. Fin Ops menampilkan pemasukan dan pengeluaran bulan berjalan, riwayat transaksi, dan jawaban atas pertanyaan tentang data akun Anda.

Aplikasi berjalan di komputer lokal. Setiap pengguna masuk dengan akun Google dan memiliki transaksi sendiri.

## Yang bisa dilakukan

| Fitur | Cara kerja |
| --- | --- |
| Catat transaksi | Tulis, misalnya, `kopi 28rb tadi pagi pakai gopay`. Periksa jenis, nominal, tanggal, dan kategori sebelum menyimpan. Form manual tersedia jika parser tidak membantu. |
| Lihat ringkasan | Dashboard menghitung pemasukan, pengeluaran, dan arus bersih bulan berjalan. |
| Kelola riwayat | Buka daftar transaksi untuk mengedit, menghapus, atau mengunduh CSV. |
| Tanya AI | Ajukan pertanyaan tentang total, kategori, tren bulanan, atau transaksi terbaru. Backend hanya membaca data akun yang sedang masuk. |

AI membantu mengurai input dan memilih jenis query untuk fitur Tanya AI. Penyimpanan transaksi tetap dilakukan setelah pengguna menyetujui pratinjau. Fitur Tanya AI memerlukan provider yang menyediakan API Chat Completions kompatibel OpenAI; pertanyaan di luar jenis query yang didukung akan ditolak.

## Teknologi

- **Web:** React dan Vite
- **API:** Python dan FastAPI
- **Database:** PostgreSQL
- **Login:** Google OAuth
- **AI:** endpoint `/chat/completions` dengan base URL, model, dan API key dari `.env`

## Jalankan di lokal

Siapkan Node.js 20.19+ atau 22.12+, Python 3.11+, dan Docker Compose. Perintah berikut dijalankan dari root repo, kecuali langkah frontend.

### 1. Konfigurasi

Salin `.env.example` menjadi `.env`:

```powershell
Copy-Item .env.example .env
```

Isi nilai berikut di `.env`:

| Variabel | Kegunaan |
| --- | --- |
| `SESSION_SECRET` | Kunci sesi. Buat string acak panjang; jangan gunakan nilai contoh. |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | Kredensial OAuth Google untuk login. |
| `AI_BASE_URL`, `AI_API_KEY`, `AI_MODEL` | Provider AI. Boleh dibiarkan kosong bila hanya memakai input manual atau parser aturan dasar. Tanya AI memerlukannya. |
| `DATABASE_URL` | Koneksi PostgreSQL. Nilai contoh cocok dengan `compose.yaml`. |

Buat OAuth client bertipe Web application di Google Cloud. Gunakan `http://localhost:8000/api/auth/callback` sebagai *Authorized redirect URI* dan `http://localhost:5173` sebagai *Authorized JavaScript origin*. Jika consent screen masih dalam mode testing, tambahkan akun yang akan dipakai sebagai test user.

Untuk membuat `SESSION_SECRET`:

```powershell
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

### 2. Nyalakan database

```powershell
docker compose up -d db
```

### 3. Jalankan API

Buka terminal PowerShell dari root repo:

```powershell
python -m venv backend/.venv
backend/.venv/Scripts/Activate.ps1
pip install -r backend/requirements.txt
uvicorn backend.main:app --reload --port 8000
```

### 4. Jalankan web

Buka terminal kedua:

```powershell
cd frontend
npm install
npm run dev
```

Buka [http://localhost:5173](http://localhost:5173). Masuk dengan Google, tulis contoh transaksi di atas, periksa pratinjau, lalu pilih **Simpan transaksi**. Dokumentasi API tersedia di [http://localhost:8000/docs](http://localhost:8000/docs).

Gunakan hostname `localhost` untuk kedua layanan agar alur login dan sesi bekerja sesuai konfigurasi contoh.

## Susunan repo

```text
backend/                 API, login, transaksi, dan alur AI
frontend/                UI React
compose.yaml             PostgreSQL lokal
.env.example             Contoh konfigurasi tanpa secret
docs/PRD.md              Kebutuhan produk
docs/DESIGN.md           Desain UI dan arsitektur
AGENTS.md                Aturan kerja di repo
.github/agents/          Definisi agent
.agents/skills/          Skill proyek
```

Aplikasi saat ini disiapkan untuk penggunaan lokal. Migrasi database dan konfigurasi deployment produksi belum tersedia. Kredensial database di `compose.yaml` hanya untuk development lokal.
