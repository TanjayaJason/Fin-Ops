# Fin Ops (F-Ops)

Aplikasi web multi-user untuk mencatat pemasukan dan pengeluaran dalam Bahasa Indonesia dan rupiah. Frontend React menampilkan input cepat, ringkasan bulanan, riwayat, serta halaman tanya jawab. Backend FastAPI memakai PostgreSQL, login Google, dan provider AI yang kompatibel dengan OpenAI Chat Completions.

AI memiliki dua peran: mengusulkan draft transaksi dari teks dan memilih query baca untuk menjawab pertanyaan tentang data akun yang sedang login. Pengguna memeriksa draft sebelum menyimpan. Jika AI belum dikonfigurasi, input transaksi sederhana tetap dapat diproses dengan aturan dasar atau diisi manual; fitur Tanya AI memerlukan provider.

## Menjalankan di lokal

Prasyarat: Node.js 20.19+ atau 22.12+, Python 3.11+, dan Docker dengan Compose. Gunakan hostname `localhost` untuk frontend dan backend.

1. Salin `.env.example` menjadi `.env`. Isi `SESSION_SECRET` dengan nilai acak panjang, misalnya dari `python -c "import secrets; print(secrets.token_urlsafe(48))"`.
2. Buat OAuth client bertipe **Web application** di Google Cloud. Tambahkan `http://localhost:8000/api/auth/callback` sebagai **Authorized redirect URI** dan `http://localhost:5173` sebagai **Authorized JavaScript origin**. Isi `GOOGLE_CLIENT_ID` dan `GOOGLE_CLIENT_SECRET` di `.env`. Bila OAuth consent masih mode testing, masukkan akun Anda sebagai test user.
3. Isi `AI_BASE_URL`, `AI_API_KEY`, dan `AI_MODEL` di `.env` untuk provider OpenAI-compatible. `AI_BASE_URL` berakhir di path versi API, misalnya `https://api.openai.com/v1`. Provider harus menyediakan endpoint `/chat/completions`. Nilai ini opsional untuk pencatatan manual, tetapi wajib untuk fitur Tanya AI.
4. Jalankan PostgreSQL: `docker compose up -d db`.
5. Di terminal backend: `python -m venv backend/.venv`, aktifkan virtual environment (`backend/.venv/Scripts/Activate.ps1` pada PowerShell), lalu `pip install -r backend/requirements.txt` dan `uvicorn backend.main:app --reload --port 8000` dari root repo.
6. Di terminal frontend: `cd frontend`, jalankan `npm install` lalu `npm run dev`.
7. Buka [http://localhost:5173](http://localhost:5173), login dengan Google, lalu coba input `kopi 28rb tadi pagi pakai gopay`. Periksa pratinjau dan klik **Simpan transaksi**.

API tersedia di [http://localhost:8000/docs](http://localhost:8000/docs). Database lokal memakai kredensial development pada `compose.yaml`; ganti sebelum mengakses dari jaringan lain. Aplikasi ini dikonfigurasi untuk penggunaan lokal dan belum memiliki migrasi database atau deployment produksi.

## Struktur repo

- `frontend/`: React + Vite.
- `backend/`: FastAPI, OAuth, transaksi, orchestrator AI, dan query data.
- `compose.yaml`: PostgreSQL lokal.
- `docs/PRD.md`: kebutuhan produk.
- `docs/DESIGN.md`: desain UI dan arsitektur.
- `AGENTS.md`, `.github/agents/`, `.agents/skills/`: instruksi agent dan skill proyek.
