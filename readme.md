

# CollabSpark Backend API

Selamat datang di dokumentasi resmi untuk CollabSpark Backend API. Aplikasi ini dibangun menggunakan Rust dengan framework Axum dan database PostgreSQL. API ini menyediakan serangkaian endpoint RESTful untuk mengelola proyek, anggota, tugas, dan fitur kolaborasi lainnya secara real-time.

---

## 1. Menjalankan Backend

### Prasyarat
- [Rust & Cargo](https://www.rust-lang.org/tools/install)
- [PostgreSQL](https://www.postgresql.org/download/)
- [sqlx-cli](https://github.com/launchbadge/sqlx/blob/main/sqlx-cli/README.md) (`cargo install sqlx-cli`)

### Langkah-langkah Instalasi
1.  **Clone Repository**
    ```sh
    git clone <url-repository-anda>
    cd <nama-folder-backend>
    ```

2.  **Konfigurasi Environment**
    Salin file `.env.example` menjadi `.env` dan sesuaikan variabel di dalamnya, terutama `DATABASE_URL` dan `JWT_SECRET`.
    ```sh
    cp .env.example .env
    ```
    Contoh isi `.env`:
    ```env
    DATABASE_URL="postgres://user:password@localhost/collabspark_db"
    JWT_SECRET="rahasia-super-panjang-dan-aman"
    # Tambahkan variabel lain sesuai kebutuhan
    ```

3.  **Setup Database**
    Buat database di PostgreSQL sesuai dengan nama yang ada di `DATABASE_URL`. Kemudian, jalankan migrasi database.
    ```sh
    sqlx database create
    sqlx migrate run
    ```
    Ini akan membuat semua tabel yang diperlukan dan mengisi data awal seperti `permission_tier`.

4.  **Jalankan Server**
    ```sh
    cargo run
    ```
    Server akan berjalan di `http://127.0.0.1:7878` (atau sesuai konfigurasi Anda).

---

## 2. Struktur & Fitur API

### Catatan Umum
-   Semua endpoint yang mengubah state (POST, PATCH, DELETE) dilindungi oleh **CSRF & JWT Authentication**.
-   Endpoint `GET` umumnya hanya memerlukan JWT Authentication.
-   Data dikirim dan diterima dalam format `application/json`.
-   ID yang digunakan adalah `INTEGER` (auto-increment), bukan UUID.
-   Timestamp menggunakan format ISO 8601 (misal: `2025-08-20T10:00:00Z`).

---

### Bagian 1: Autentikasi & Manajemen Pengguna

#### **Register & Login**

1.  **Register Pengguna Baru**
    -   `POST /api/auth/register`
    -   **Body:**
        ```json
        {
          "username": "userbaru",
          "email": "user.baru@example.com",
          "password": "PasswordKuat123!"
        }
        ```
    -   **Response:** Data pengguna yang baru dibuat (tanpa hash password).

2.  **Login Pengguna**
    -   `POST /api/auth/login`
    -   **Body:**
        ```json
        {
          "email": "user.baru@example.com",
          "password": "PasswordKuat123!"
        }
        ```
    -   **Response:** Pesan sukses dan `auth-token` (JWT). `auth-token` juga akan diatur sebagai `HttpOnly` cookie.

3.  **Logout Pengguna**
    -   `POST /api/auth/logout`
    -   **Response:** `204 No Content`. Menghapus cookie autentikasi.

#### **Manajemen Sesi & Info Pengguna**

1.  **Get Info Pengguna Saat Ini**
    -   `GET /api/auth/me`
    -   **Response:** Data pengguna yang sedang login.

2.  **Get Info Pengguna Berdasarkan ID**
    -   `GET /api/users/:id`

3.  **Get Token CSRF**
    -   `GET /api/csrf/token`
    -   **Response:** Mengatur cookie CSRF dan mengembalikan token mentah untuk digunakan di header atau body request.

#### **Reset Password**
1.  **Minta Reset Password**
    -   `POST /api/auth/forgot-password`
    -   **Body:**
        ```json
        { "email": "user.lupa@example.com" }
        ```

2.  **Lakukan Reset Password**
    -   `POST /api/users/reset-password`
    -   **Body:**
        ```json
        {
          "token": "token-reset-dari-email",
          "new_password": "PasswordBaruKuat123!"
        }
        ```

---

### Bagian 2: Manajemen Proyek

#### **CRUD Proyek**
-   `GET /api/projects`: Mendapatkan semua proyek di mana pengguna adalah anggota.
-   `POST /api/projects`: Membuat proyek baru.
-   `GET /api/projects/:project_id`: Mendapatkan detail satu proyek.
-   `PATCH /api/projects/:project_id`: Mengedit detail proyek.
-   `DELETE /api/projects/:project_id`: Menghapus proyek.

#### **Contoh Body `POST /api/projects`**
```json
{
  "project_name": "Proyek Peluncuran Produk Baru",
  "business_name": "PT. Inovasi Jaya",
  "description": "Deskripsi lengkap tentang tujuan proyek peluncuran produk."
}
```

#### **Fitur Lainnya**
-   `POST /api/projects/:project_id/transfer-ownership`: Memindahkan kepemilikan proyek ke anggota lain.
-   `GET /api/projects/search?q=<query>`: Mencari proyek.
-   `GET /api/users/me/recent-projects`: Mendapatkan daftar proyek yang baru saja dikunjungi.

---

### Bagian 3: Manajemen Anggota & Peran

#### **Manajemen Anggota**
-   `GET /api/projects/:project_id/members`: Mendapatkan daftar anggota proyek.
-   `POST /api/projects/:project_id/members`: Menambahkan anggota baru ke proyek.
-   `PATCH /api/projects/:project_id/members/:member_id`: Mengedit `full_name`, `role_id`, atau `permission_tier_id` anggota.
-   `POST /api/projects/:project_id/members/:member_id/ban`: Memblokir anggota.
-   `POST /api/projects/:project_id/members/:member_id/unban`: Membuka blokir anggota.

#### **Manajemen Peran (Job Roles)**
-   `GET /api/projects/:project_id/roles`: Mendapatkan semua peran kustom untuk proyek.
-   `POST /api/projects/:project_id/roles`: Membuat peran kustom baru.
-   `PATCH /api/projects/:project_id/roles/:role_id`: Mengedit peran kustom.
-   `DELETE /api/projects/:project_id/roles/:role_id`: Menghapus peran kustom (dengan opsi migrasi anggota).
-   `GET /api/projects/:project_id/roles/:role_id/member-count`: Menghitung jumlah anggota dengan peran tertentu.

#### **Manajemen Undangan (Invites)**
-   `GET /api/permission-tiers`: Mendapatkan daftar tingkat izin sistem (Admin, Editor, Viewer, Member).
-   `POST /api/projects/:project_id/invites`: Membuat tautan undangan baru.
-   `GET /api/invites/:invite_code/accept`: Menerima undangan (memerlukan autentikasi).

---

### Bagian 4: Manajemen Tugas & Sub-tugas

#### **Tugas (Tasks)**
-   `GET /api/projects/:project_id/tasks`: Mendapatkan semua tugas aktif dalam proyek.
-   `POST /api/projects/:project_id/tasks`: Membuat tugas baru.
-   `PATCH /api/tasks/:task_id`: Mengedit detail tugas (judul, deskripsi, status, penanggung jawab).
-   `DELETE /api/tasks/:task_id`: Menghapus tugas.

#### **Fitur Tugas Lainnya**
-   `GET /api/projects/:project_id/tasks/archived`: Mendapatkan tugas yang sudah diarsipkan.
-   `POST /api/tasks/:task_id/archive`: Mengarsipkan tugas secara manual.
-   `POST /api/tasks/:task_id/unarchive`: Mengembalikan tugas dari arsip.
-   `POST /api/tasks/:task_id/contributors`: Menambahkan kontributor ke tugas.
-   `DELETE /api/tasks/:task_id/contributors`: Menghapus kontributor dari tugas.
-   `POST /api/tasks/:task_id/required-roles`: Menambahkan peran yang dibutuhkan untuk tugas.
-   `DELETE /api/tasks/:task_id/required-roles`: Menghapus peran yang dibutuhkan.

#### **Sub-tugas (Subtasks)**
-   `GET /api/tasks/:task_id/subtasks`: Mendapatkan semua sub-tugas dari sebuah tugas.
-   `POST /api/tasks/:task_id/subtasks`: Membuat satu sub-tugas baru.
-   `POST /api/tasks/:task_id/subtasks/bulk`: Membuat beberapa sub-tugas sekaligus.
-   `PATCH /api/subtasks/:subtask_id`: Mengedit sub-tugas (deskripsi, status selesai).
-   `DELETE /api/subtasks/:subtask_id`: Menghapus sub-tugas.

---

### Bagian 5: Real-time Updates dengan WebSockets

-   **Endpoint:** `GET /api/projects/:project_id/ws`
-   Setelah koneksi WebSocket berhasil dibuat, server akan mengirim pesan JSON setiap kali ada perubahan pada data proyek.
-   **Contoh Pesan:**
    ```json
    {
      "type": "task_updated",
      "data": { /* Objek TaskResponse lengkap */ }
    }
    ```
-   **Tipe Pesan yang Didukung:** `project_updated`, `member_created`, `member_updated`, `member_banned`, `member_unbanned`, `role_created`, `role_updated`, `role_deleted`, `task_created`, `task_updated`, `task_deleted`, `task_archived`, `task_unarchived`.

---

### Git Workflow

1.  Setiap fitur dikerjakan di branch terpisah (misal: `feat/real-time-tasks`, `fix/login-cookie`).
2.  Setelah selesai, buat Pull Request (PR) ke branch `development`.
3.  Setelah di-review dan di-merge ke `development`, fitur akan diuji bersama fitur lain.
4.  Jika `development` sudah stabil dan siap rilis, buat PR dari `development` ke `main`.
5.  Branch `main` selalu mencerminkan kode produksi yang stabil.
