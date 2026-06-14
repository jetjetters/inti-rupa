# Backend Technical Notes & UAS Preparation

Dokumen ini disusun sebagai panduan teknis bagi tim Backend (khususnya Lead Backend) untuk menjelaskan arsitektur, keputusan teknologi, dan mekanisme keamanan sistem **Inti Rupa Cloud App** saat evaluasi UAS.

---

## 1. Pertanyaan Arsitektural Utama

### Q1: Mengapa Memilih FastAPI?
**Jawaban Teknis:**
*   **Asynchronous Support (ASGI):** FastAPI dibangun di atas Starlette dan Uvicorn, mendukung pemrograman asinkron (`async/await`) secara native. Hal ini sangat krusial bagi aplikasi kita yang menangani I/O-bound operations yang berat seperti pemanggilan API model LLM/Gemini dan Image Generation (Hugging Face). Dengan arsitektur asinkron, server tidak terblokir (non-blocking) saat menunggu respon dari pihak ketiga, sehingga throughput request per detik (RPS) jauh lebih tinggi dibandingkan framework WSGI seperti Flask.
*   **Kecepatan Eksekusi (High Performance):** FastAPI memiliki performa setara dengan NodeJS dan Go, menjadikannya salah satu framework Python tercepat yang ada saat ini.
*   **Validasi Data Otomatis & Keamanan (Pydantic):** Integrasi erat dengan Pydantic melakukan validasi tipe data (type-hinting) secara otomatis saat request masuk. Jika data tidak valid, FastAPI langsung mengembalikan respon `422 Unprocessable Entity` secara otomatis lengkap dengan detail error-nya. Ini mencegah eksploitasi data buruk (bad data injection).
*   **Dokumentasi OpenAPI Otomatis:** FastAPI secara otomatis menggenerate UI dokumentasi interaktif (Swagger UI di `/docs` dan ReDoc di `/redoc`) berdasarkan type hints dan Pydantic schemas kita. Hal ini mempercepat integrasi dengan tim Frontend tanpa perlu menulis dokumentasi API manual secara terpisah.

---

### Q2: Bagaimana Mekanisme Autentikasi Antar-Service (Inter-Service Auth)?
**Jawaban Teknis:**
Sistem kami menerapkan arsitektur **Decentralized JWT-based Authentication** dengan **API Gateway (Nginx)** sebagai pintu masuk tunggal:
1.  **Gateway Routing:** Semua request dari luar masuk melalui Port `80` (Nginx Gateway). Nginx bertindak sebagai reverse proxy yang mengarahkan path `/auth/*` ke `auth-service` (Port `8001`) dan path `/api/*` ke `ai-service` (Port `8002`).
2.  **JWT Token Generation:** Ketika pengguna login atau mendaftar di `auth-service`, service ini menghasilkan token JWT (JSON Web Token) yang ditandatangani menggunakan kunci simetris (`SECRET_KEY`) dengan algoritma `HS256`. Token ini menyimpan klaim payload seperti `sub` (user_id), `email`, dan `exp` (waktu kadaluarsa).
3.  **Token Validation di AI Service (Inter-service):**
    *   Setiap request ke endpoint terproteksi di `ai-service` harus menyertakan token JWT ini di HTTP Header: `Authorization: Bearer <token>`.
    *   `ai-service` **tidak melakukan request HTTP synchronous tambahan ke `auth-service`** untuk memvalidasi token tersebut.
    *   Sebaliknya, `ai-service` secara mandiri memecahkan dan memverifikasi tanda tangan JWT tersebut menggunakan `SECRET_KEY` yang sama (shared secret key).
    *   Metode ini disebut **Stateless Verification**. Ini mengeliminasi overhead jaringan (network hop) dan menghindari bottleneck komunikasi sinkron antar service.
4.  **Quota Synchronization:** Jika `ai-service` perlu memperbarui data penggunaan API kuota (`api_used`), ia melakukan panggilan API asinkron internal ke `auth-service` pada endpoint internal terproteksi (`PUT /auth/users/me/increment-api`) menggunakan Bearer Token dari user untuk meng-increment penggunaan kuota secara aman.

---

### Q3: Apa yang Terjadi Jika Auth Service Down?
**Jawaban Teknis:**
Desain arsitektur mikroservis kami dirancang dengan prinsip **Fault Tolerance** dan **Service Isolation** untuk menangani kegagalan `auth-service`:
*   **State Saat Ini (State of System):**
    *   Karena verifikasi token di `ai-service` bersifat stateless (menggunakan kunci rahasia bersama `SECRET_KEY` untuk memvalidasi tanda tangan JWT), user yang **sudah memiliki token valid yang belum kadaluarsa** akan tetap dapat menggunakan fitur AI Chat dan Image Generation di `ai-service` secara normal! Verifikasi token tidak akan gagal karena tidak ada dependency sinkron langsung ke database auth untuk validasi token dasar.
    *   Namun, fitur pendaftaran baru (`/auth/register`), login baru (`/auth/login`), dan penambahan kuota penggunaan API (`/auth/users/me/increment-api`) tidak akan berfungsi dan mengembalikan error `502 Bad Gateway` atau `503 Service Unavailable`.
*   **Mitigasi Dampak & Rekomendasi Production Hardening:**
    1.  **Circuit Breaker & Fallback:** Di level Gateway (Nginx) atau menggunakan pustaka internal, kita dapat menerapkan pola Circuit Breaker. Jika `auth-service` gagal merespon berturut-turut, gateway langsung menolak request login/register dengan pesan ramah tanpa membebani server.
    2.  **User State Caching (Redis):** Di lingkungan produksi skala besar, data session user atau status blacklist token disimpan di Redis cache cluster. Jika database atau Auth Service down, `ai-service` dapat memvalidasi status keaktifan user dari cache terdistribusi yang sangat cepat dan memiliki availability tinggi.
    3.  **Graceful Degradation:** Pada `ai-service`, jika panggilan sinkron internal untuk increment kuota (`/auth/users/me/increment-api`) gagal karena `auth-service` down, kita menangkap error tersebut (`try-except`) secara aman, mencatat log warning, dan **tetap mengizinkan** request generation/chat milik user selesai (tidak memblokir layanan bagi user yang sudah terautentikasi), lalu mengantrekan pencatatan kuota ke background worker/message queue (seperti RabbitMQ) untuk diproses saat `auth-service` kembali online.

---

## 2. Ringkasan Kesiapan Endpoint Backend

Semua endpoint backend utama telah diuji secara menyeluruh dan dipastikan berfungsi dengan baik:

| Service | Endpoint | Metode | Proteksi | Deskripsi |
| :--- | :--- | :--- | :--- | :--- |
| **Auth** | `/auth/register` | POST | Publik | Registrasi user baru dengan validasi password & email ketat. |
| **Auth** | `/auth/login` | POST | Publik | Autentikasi user & pembuatan token JWT. |
| **Auth** | `/auth/users/me` | GET | Bearer JWT | Mengambil profil user aktif beserta statistik `api_used`. |
| **Auth** | `/auth/users/me/increment-api` | PUT | Bearer JWT | Increment counter `api_used` user secara aman. |
| **Auth** | `/auth/health` | GET | Publik | Liveness check status `auth-service` & koneksi database. |
| **AI** | `/chats` | POST | Bearer JWT | Membuat sesi chat baru (judul otomatis dari prompt pertama). |
| **AI** | `/chats` | GET | Bearer JWT | Mengambil daftar riwayat sesi chat milik user aktif. |
| **AI** | `/chats/{chat_id}/messages` | POST | Bearer JWT | Mengirim chat prompt ke AI & mendapatkan respon asinkron. |
| **AI** | `/chats/{chat_id}/messages` | GET | Bearer JWT | Mengambil daftar pesan dalam satu sesi chat tertentu. |
| **AI** | `/images/generate` | POST | Bearer JWT | Menghasilkan gambar menggunakan model Diffusion (Hugging Face). |
| **AI** | `/images` | GET | Bearer JWT | Mengambil riwayat generasi gambar user aktif. |
| **AI** | `/health` | GET | Publik | Liveness check status `ai-service` & koneksi database. |
