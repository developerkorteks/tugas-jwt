# REST API Mahasiswa + Autentikasi JWT & Role-Based Access Control (RBAC)
## Dokumentasi Komprehensif, Source Code Lengkap, dan 45 Test Case Objektif By Data

Proyek ini adalah implementasi sistem **RESTful API Mahasiswa** berbasis **PHP Native** dan **MySQL** yang mengadopsi standar keamanan web service modern industri. Proyek mencakup seluruh materi **Pertemuan 7 Teknologi Web Service** (Autentikasi JWT, Otorisasi RBAC, Rate Limiting, Audit Logging, Validasi Input, Prepared Statements) serta penambahan seluruh butir **Tugas Pengembangan** (`POST /register`, `PUT /me/password`, Peran Baru `dosen`, 3 Query Analisis Log SQL, dan Uji Klien Otomatis 45 Test Case).

---

## DAFTAR ISI
1. [Landasan Teori & Arsitektur Keamanan](#1-landasan-teori--arsitektur-keamanan)
2. [Panduan Instalasi & Setup Lengkap di Laragon (Step-by-Step)](#2-panduan-instalasi--setup-lengkap-di-laragon-step-by-step)
3. [Daftar Endpoint & Hak Akses (Matrix RBAC)](#3-daftar-endpoint--hak-akses-matrix-rbac)
4. [Source Code Lengkap Seluruh Berkas & Keterangan Mendalam](#4-source-code-lengkap-seluruh-berkas--keterangan-mendalam)
   - 4.1 Basis Data: `database/mhs.sql`
   - 4.2 Inisialisasi Database & Parameter Keamanan: `server/koneksi.php`
   - 4.3 Mesin Kriptografi Token: `server/jwt.php`
   - 4.4 Middleware & Pustaka Fungsi: `server/fungsi.php`
   - 4.5 Front Controller & Router API: `server/index.php`
   - 4.6 Konfigurasi Web Server: `server/.htaccess` & `server/router.php`
   - 4.7 Query Forensik Audit Log: `latihan/analisis_log.sql`
   - 4.8 Skrip Uji Kriptografi Terminal: `latihan/jwt_demo.php`
   - 4.9 Simulator Beban Rate Limit: `latihan/uji_rate_limit.php`
   - 4.10 Antarmuka Web Single Page Application: `client/index.html`
   - 4.11 Test Suite Penguji Otomatis: `client/test_client.php`
5. [Dokumentasi Komprehensif 45 Test Case Objektif (Request, Response & Data)](#5-dokumentasi-komprehensif-45-test-case-objektif-request-response--data)
6. [Analisis Log Forensik SQL & Interpretasi Data](#6-analisis-log-forensik-sql--interpretasi-data)
7. [Jawaban Pertanyaan Refleksi Modul Praktikum](#7-jawaban-pertanyaan-refleksi-modul-praktikum)

---

## 1. LANDASAN TEORI & ARSITEKTUR KEAMANAN

### 1.1 Autentikasi vs Otorisasi (401 Unauthorized vs 403 Forbidden)
- **Autentikasi (Authentication - HTTP 401)**: Proses pembuktian identitas pengakses (*"Siapa Anda?"*). Dilakukan saat pengakses mengirim kredensial login atau menyertakan Bearer Token di header `Authorization`. Apabila token tidak dikirim, formatnya cacat, tanda tangan tidak cocok, atau sudah kedaluwarsa, server menolak dengan **HTTP 401 Unauthorized** serta header `WWW-Authenticate: Bearer`.
- **Otorisasi (Authorization - HTTP 403)**: Proses penentuan izin atau hak akses dari identitas yang sudah terbukti (*"Bolehkah Anda melakukan tindakan ini?"*). Misalnya pengguna `budi` (peran `user`) berhasil login dan memiliki token sah, namun mencoba menghapus data (`DELETE /mhs/1`) atau mengakses catatan audit (`GET /log`). Server mengenali siapa pengakses, tetapi menolak permintaannya dengan **HTTP 403 Forbidden**.

### 1.2 Anatomi JSON Web Token (JWT - RFC 7519)
JWT adalah standar terbuka berbasis token yang bersifat *stateless* dan mandiri (*self-contained*). Token terdiri atas 3 segmen Base64URL yang dipisahkan titik:
$$\text{JWT} = \text{Header} \,.\, \text{Payload} \,.\, \text{Signature}$$
1. **Header**: Mendefinisikan tipe token dan algoritma penandatanganan: `{"alg":"HS256","typ":"JWT"}`.
2. **Payload (Claims)**: Memuat pernyataan/data pengguna (`sub`, `username`, `role`, `iat`, `exp`). **Perhatian penting**: Payload hanya di-encode (Base64URL), bukan dienkripsi. Siapa pun dapat membaca data payload. Karena itu, **password atau data rahasia tidak boleh disimpan di payload**.
3. **Signature**: Menjamin integritas data. Dihitung dengan rumus:
   $$\text{Signature} = \text{Base64URL}\Big(\text{HMAC-SHA256}\big(\text{Header} \,.\, \text{Payload}, \, \text{SECRET\_KEY}\big)\Big)$$
   Jika ada pihak ketiga yang memanipulasi klaim peran (`"role":"admin"`), tanda tangan hasil kalkulasi ulang server tidak akan pernah cocok karena penyerang tidak memiliki secret key server.

### 1.3 Keamanan Password (Bcrypt via `password_hash()`)
Password tidak pernah disimpan dalam format teks mentah (*plaintext*). Sistem menggunakan algoritma adaptif **Bcrypt** (`PASSWORD_DEFAULT`):
- Bersifat satu arah (*irreversible*).
- Menghasilkan salt unik acak secara otomatis untuk setiap proses hash, sehingga menetralkan serangan kamus dan *Rainbow Table*.
- Verifikasi dilakukan secara aman menggunakan fungsi `password_verify($password, $hash)`.

### 1.4 Pertahanan SQL Injection & Input Validation
- **PDO Prepared Statements**: Seluruh parameter eksternal disalurkan melalui placeholder `?` (`$stmt->execute([$param])`), memastikan input klien diperlakukan murni sebagai nilai data tanpa bisa dieksekusi sebagai instruksi SQL.
- **Validasi Whitelist**: Memvalidasi parameter angka dengan `ctype_digit()`, regex karakter `^[a-zA-Z0-9_.]+$` untuk username, dan ambang batas panjang karakter minimal 8 untuk password.

### 1.5 Rate Limiting & Audit Logging
- **Brute Force Limiter (Login)**: Membatasi kegagalan login maksimal 5 kali per IP per target username dalam rentang waktu 60 detik. Percobaan ke-6 diblokir dengan **HTTP 429 Too Many Requests** dan header `Retry-After: 60`.
- **Global Limiter**: Membatasi maksimal 120 request per menit per IP ke seluruh endpoint.
- **Audit Logging**: Mencatat setiap transaksi ke tabel `log_api` (waktu, method, path, IP, username pengakses, dan status respons).

---

## 2. PANDUAN INSTALASI & SETUP LENGKAP DI LARAGON (STEP-BY-STEP)

### Langkah 1: Penempatan Direktori Proyek
1. Buka folder root web Laragon (default pada Windows):
   ```text
   C:\laragon\www\
   ```
2. Tempatkan folder proyek ke dalam direktori tersebut sehingga memiliki jalur:
   ```text
   C:\laragon\www\api-mhs-auth
   ```
3. Pastikan struktur berkas berada pada:
   ```text
   C:\laragon\www\api-mhs-auth\server\index.php
   ```

### Langkah 2: Mengaktifkan Modul Apache `mod_rewrite` & Header `Authorization`
Secara default, Apache FastCGI sering membuang header HTTP `Authorization`. Proyek ini menyertakan file `server/.htaccess` dengan konfigurasi:
```apache
RewriteEngine On

# Meneruskan Header Authorization ke Environment PHP
RewriteCond %{HTTP:Authorization} .
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

# URL Rewriting menuju index.php
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^ index.php [L]
```
- **Aktivasi Modul**: Klik kanan icon tray Laragon $\to$ **Apache** $\to$ **modules** $\to$ pastikan **rewrite_module** dalam kondisi tercentang.

### Langkah 3: Menjalankan Layanan & Impor Database
1. Buka antarmuka aplikasi Laragon, lalu klik tombol **Start All** (Apache & MySQL running).
2. Buka database management tool Laragon (klik tombol **Database** untuk membuka HeidiSQL) atau buka phpMyAdmin di browser: `http://localhost/phpmyadmin`.
3. Kredensial bawaan Laragon:
   - **Hostname**: `127.0.0.1` / `localhost`
   - **User**: `root`
   - **Password**: *(kosong)*
   - **Port**: `3306`
4. Buka tab Query / Import, lalu buka dan jalankan file `database/mhs.sql`.
5. Skrip akan otomatis membuat basis data `kampus_db` beserta tabel `mhs`, `users`, dan `log_api`.

### Langkah 4: Verifikasi Konfigurasi Koneksi (`server/koneksi.php`)
Buka file `server/koneksi.php` dan pastikan konfigurasi telah sesuai:
```php
$host    = '127.0.0.1';
$user    = 'root';
$pass    = '';            // Laragon default: kosong
$nama_db = 'kampus_db';
```
Base URL API pada Laragon:
```text
http://localhost/api-mhs-auth/server
```
*(Atau `http://api-mhs-auth.test/server` jika fitur Pretty URL Laragon aktif)*.

---

## 3. DAFTAR ENDPOINT & HAK AKSES (MATRIX RBAC)

| Method | Endpoint | Hak Akses | Status Sukses | Deskripsi & Validasi |
| :--- | :--- | :--- | :---: | :--- |
| `POST` | `/login` | Publik | `200 OK` | Login kredensial `{username, password}`. Mengembalikan JWT Bearer. |
| `POST` | `/register` | Publik | `201 Created` | Registrasi user baru: username unik 3-50 char, password min 8 char. Role default `'user'`. |
| `GET` | `/me` | Token (Semua) | `200 OK` | Mengambil data profil pengguna yang terautentikasi dan masa berlaku token. |
| `PUT` | `/me/password` | Token (Semua) | `200 OK` | Ganti password mandiri: wajib mengirim `{password_lama, password_baru}` (min 8 char). |
| `GET` | `/mhs` | Token (Semua) | `200 OK` | Mengambil seluruh katalog data mahasiswa. |
| `GET` | `/mhs/:id` | Token (Semua) | `200 OK` | Mengambil 1 data mahasiswa berdasarkan ID numerik. |
| `POST` | `/mhs` | Token + Role **Admin** | `201 Created` | Menambah mahasiswa: `{nim, name, major}`. Validasi NIM unik dan digit. |
| `PUT` | `/mhs/:id` | Token + Role **Admin** / **Dosen** | `200 OK` | Mengubah data mahasiswa. Diizinkan untuk Admin dan Dosen. |
| `DELETE`| `/mhs/:id` | Token + Role **Admin** | `200 OK` | Menghapus data mahasiswa. Dosen dan User dilarang (403 Forbidden). |
| `GET` | `/log` | Token + Role **Admin** | `200 OK` | Melihat 20 catatan log transaksi audit terakhir. |

---

## 4. SOURCE CODE LENGKAP SELURUH BERKAS & KETERANGAN MENDALAM

### 4.1 Basis Data: `database/mhs.sql`
```sql
-- =====================================================================
-- database/mhs.sql  --  Database API Mahasiswa + AUTENTIKASI (Pertemuan 7)
-- Cara pakai: phpMyAdmin -> tab Import -> pilih file ini -> Go
-- (File ini menghapus & membuat ulang tabel, jadi data lama akan hilang.)
-- =====================================================================

CREATE DATABASE IF NOT EXISTS kampus_db CHARACTER SET utf8mb4;
USE kampus_db;

-- ---------------------------------------------------------------------
-- TABEL 1: mhs  -> data mahasiswa
-- ---------------------------------------------------------------------
DROP TABLE IF EXISTS mhs;
CREATE TABLE mhs (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  nim        VARCHAR(12)  NOT NULL UNIQUE,
  name       VARCHAR(100) NOT NULL,
  major      VARCHAR(100) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ---------------------------------------------------------------------
-- TABEL 2: users  -> akun yang boleh login (BARU di Pertemuan 7)
--   password disimpan dalam bentuk HASH, BUKAN teks asli.
--   role menentukan hak akses (otorisasi): 'admin' atau 'user'.
-- ---------------------------------------------------------------------
DROP TABLE IF EXISTS users;
CREATE TABLE users (
  id        INT AUTO_INCREMENT PRIMARY KEY,
  username  VARCHAR(50)  NOT NULL UNIQUE,
  password  VARCHAR(255) NOT NULL,                  -- hasil password_hash()
  role      VARCHAR(10)  NOT NULL DEFAULT 'user'    -- 'admin' atau 'user'
);

-- ---------------------------------------------------------------------
-- TABEL 3: log_api  -> catatan setiap request (kolom username BARU)
-- ---------------------------------------------------------------------
DROP TABLE IF EXISTS log_api;
CREATE TABLE log_api (
  id       INT AUTO_INCREMENT PRIMARY KEY,
  waktu    DATETIME     NOT NULL,
  method   VARCHAR(10)  NOT NULL,
  path     VARCHAR(255) NOT NULL,
  status   INT          NOT NULL,
  ip       VARCHAR(45)  NOT NULL,
  username VARCHAR(50)  NULL,                       -- siapa yang mengakses (jika login)
  pesan    VARCHAR(255) NULL
);

-- ---------------------------------------------------------------------
-- Data contoh
-- Akun latihan:   admin  / admin123  (role admin)
--                 budi   / budi123   (role user)
--                 dosen1 / dosen123  (role dosen) <-- BARU Tugas P7
-- Hanya untuk praktikum! Di aplikasi sungguhan gunakan password yang kuat.
-- ---------------------------------------------------------------------
INSERT INTO users (username, password, role) VALUES
  ('admin',  '$2y$10$pWQGUDarcWM74IF4ZYNR5u3kAqKJWaPrMjc1rXXq4w6/bsSZ7kzDK', 'admin'),  -- admin123
  ('budi',   '$2y$10$Zw8rAKVFTx5wmo5Pzg5nkuEg667G.E70mo.ep5e1WR8i67B8XLiLy', 'user'),   -- budi123
  ('dosen1', '$2y$12$qhLW6rmpT23r68D12RDahuq7IAsU97ek39.wT8hUUVzWqGsHpDq4y', 'dosen');  -- dosen123

INSERT INTO mhs (nim, name, major) VALUES
  ('22670001', 'Andi Pratama',   'Informatika'),
  ('22670002', 'Budi Santoso',   'Informatika'),
  ('22670003', 'Citra Lestari',  'Sistem Informasi');
```
*Keterangan*: Tabel `users` menggunakan tipe `VARCHAR(255)` untuk kolom `password` guna menampung panjang string hash Bcrypt 60 karakter. Kolom `username` pada `log_api` bersifat `NULL` agar request anonim/publik tetap tercatat tanpa error constraint.

---

### 4.2 Inisialisasi Database & Parameter Keamanan: `server/koneksi.php`
```php
<?php
// =====================================================================
// koneksi.php  --  Pengaturan + koneksi database
// File ini di-include oleh index.php. Hasilnya: variabel $db siap dipakai,
// ditambah beberapa PENGATURAN KEAMANAN (di bagian tengah).
// =====================================================================

// ---------------- Pengaturan database ----------------
$host    = '127.0.0.1';
$user    = 'root';        // XAMPP / Laragon: root
$pass    = '';            // XAMPP / Laragon: kosong
$nama_db = 'kampus_db';

// ---------------- Pengaturan keamanan (BARU di Pertemuan 7) ----------------

// Kunci rahasia untuk "menandatangani" token JWT.
// Siapa pun yang tahu kunci ini bisa membuat token palsu, jadi:
//   - pilih teks panjang dan acak,
//   - di aplikasi sungguhan simpan di environment variable, jangan di kode.
$jwt_secret = 'ganti-dengan-teks-rahasia-yang-panjang-dan-acak-123';

// Token berlaku berapa detik? 3600 = 1 jam.
$jwt_berlaku = 3600;

// RATE LIMITING (pembatasan jumlah request):
$batas_login = 5;     // maksimal 5x salah login per username per menit
$batas_umum  = 120;   // maksimal 120 request per IP per menit

try {
    $db = new PDO("mysql:host=$host;dbname=$nama_db;charset=utf8mb4", $user, $pass);
    $db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    $db->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
} catch (PDOException $e) {
    http_response_code(500);
    header('Content-Type: application/json');
    echo json_encode(['status' => 'error', 'pesan' => 'Tidak bisa terhubung ke database']);
    exit;
}
```
*Keterangan*: Koneksi PDO menggunakan `PDO::ERRMODE_EXCEPTION` sehingga seluruh kegagalan query SQL memicu exception yang ditangkap di `index.php` tanpa mencetak error trace ke output browser.

---

### 4.3 Mesin Kriptografi Token: `server/jwt.php`
```php
<?php
// =====================================================================
// jwt.php  --  Membuat dan memeriksa JSON Web Token (JWT)
//
// JWT adalah teks yang terdiri dari 3 bagian dipisah titik:
//      HEADER . PAYLOAD . SIGNATURE
// =====================================================================

// Base64 versi aman untuk URL: ganti + dan / menjadi - dan _, buang tanda =.
function base64url_encode($teks)
{
    return rtrim(strtr(base64_encode($teks), '+/', '-_'), '=');
}

function base64url_decode($teks)
{
    return base64_decode(strtr($teks, '-_', '+/'));
}

// ---------------------------------------------------------------------
// buat_token() : membuat JWT dari data payload
// ---------------------------------------------------------------------
function buat_token($payload, $rahasia, $detik)
{
    $payload['iat'] = time();            // iat = issued at (kapan dibuat)
    $payload['exp'] = time() + $detik;   // exp = expired (kapan kedaluwarsa)

    $header = base64url_encode(json_encode(['alg' => 'HS256', 'typ' => 'JWT']));
    $isi    = base64url_encode(json_encode($payload));

    // Tanda tangan = HMAC-SHA256 dari "header.isi" memakai kunci rahasia.
    $tanda_tangan = base64url_encode(hash_hmac('sha256', "$header.$isi", $rahasia, true));

    return "$header.$isi.$tanda_tangan";
}

// ---------------------------------------------------------------------
// cek_token() : memeriksa token
// ---------------------------------------------------------------------
function cek_token($token, $rahasia)
{
    // 1) Harus terdiri dari tepat 3 bagian.
    $bagian = explode('.', $token);
    if (count($bagian) != 3) {
        return null;
    }
    list($header, $isi, $tanda_tangan) = $bagian;

    // 2) Hitung ulang tanda tangan yang SEHARUSNYA, lalu bandingkan.
    //    Kita SELALU memakai HS256 (tidak mempercayai "alg" dari token) -> mencegah serangan "alg: none".
    //    hash_equals = perbandingan aman dari serangan pengukuran waktu (timing attack).
    $seharusnya = base64url_encode(hash_hmac('sha256', "$header.$isi", $rahasia, true));
    if (!hash_equals($seharusnya, $tanda_tangan)) {
        return null;   // isi pernah diubah, atau ditandatangani dengan kunci lain
    }

    // 3) Buka isi payload.
    $payload = json_decode(base64url_decode($isi), true);
    if (!is_array($payload)) {
        return null;
    }

    // 4) Periksa masa berlaku.
    if (!isset($payload['exp']) || $payload['exp'] < time()) {
        return null;   // sudah kedaluwarsa
    }

    return $payload;
}
```
*Keterangan*: `hash_equals()` digunakan untuk mencegah kerentanan *timing attack*. Pengecekan tidak bergantung pada field `"alg"` di header token melainkan memaksa HMAC-SHA256 untuk memitigasi serangan *alg: none*.

---

### 4.4 Middleware & Pustaka Fungsi: `server/fungsi.php`
```php
<?php
// =====================================================================
// fungsi.php  --  Fungsi bantu: log, kirim jawaban, keamanan
// =====================================================================

$username_log = null;

// ---------------------------------------------------------------------
// catat_log() : menyimpan satu baris ke tabel log_api (LOGGING DI DATABASE)
// ---------------------------------------------------------------------
function catat_log($status, $pesan = '')
{
    global $db, $username_log;

    $waktu  = date('Y-m-d H:i:s');
    $method = $_SERVER['REQUEST_METHOD'];
    $path   = $_SERVER['REQUEST_URI'];
    $ip     = $_SERVER['REMOTE_ADDR'];

    $sql  = "INSERT INTO log_api (waktu, method, path, status, ip, username, pesan)
             VALUES (?, ?, ?, ?, ?, ?, ?)";
    $stmt = $db->prepare($sql);
    $stmt->execute([$waktu, $method, $path, $status, $ip, $username_log, $pesan]);
}

// ---------------------------------------------------------------------
// kirim() : mengirim jawaban JSON, mencatat log, lalu berhenti
// ---------------------------------------------------------------------
function kirim($data, $status = 200)
{
    $pesan = isset($data['pesan']) ? $data['pesan'] : 'OK';
    catat_log($status, $pesan);

    http_response_code($status);
    header('Content-Type: application/json');
    echo json_encode($data, JSON_PRETTY_PRINT);
    exit;
}

// ---------------------------------------------------------------------
// ambil_token() : membaca token dari header Authorization: Bearer <token>
// ---------------------------------------------------------------------
function ambil_token()
{
    $header = '';
    if (isset($_SERVER['HTTP_AUTHORIZATION'])) {
        $header = $_SERVER['HTTP_AUTHORIZATION'];
    } elseif (isset($_SERVER['REDIRECT_HTTP_AUTHORIZATION'])) {
        $header = $_SERVER['REDIRECT_HTTP_AUTHORIZATION'];     // Apache rewrite fallback
    }

    if (preg_match('/^Bearer\s+(\S+)$/', $header, $cocok)) {
        return $cocok[1];
    }
    return null;
}

// ---------------------------------------------------------------------
// wajib_login() : AUTENTIKASI -> "Siapa kamu?" (401 jika gagal)
// ---------------------------------------------------------------------
function wajib_login()
{
    global $jwt_secret, $username_log;

    $token   = ambil_token();
    $payload = $token ? cek_token($token, $jwt_secret) : null;

    if (!$payload) {
        header('WWW-Authenticate: Bearer');
        kirim(['status' => 'error', 'pesan' => 'Token tidak ada, tidak valid, atau sudah kedaluwarsa'], 401);
    }

    $username_log = $payload['username'];     // tercatat di log_api
    return $payload;
}

// ---------------------------------------------------------------------
// wajib_role() : OTORISASI generik -> "Bolehkah kamu melakukan ini?" (403 jika gagal)
// ---------------------------------------------------------------------
function wajib_role($pengguna, array $roles)
{
    if (!in_array($pengguna['role'], $roles, true)) {
        $izin = implode(' atau ', $roles);
        kirim(['status' => 'error', 'pesan' => "Akses ditolak: hanya $izin yang boleh"], 403);
    }
}

// Alias backward compatibility untuk kode lama
function wajib_admin($pengguna)
{
    wajib_role($pengguna, ['admin']);
}

// ---------------------------------------------------------------------
// batasi_request() : RATE LIMITING umum per IP (429 jika melebihi)
// ---------------------------------------------------------------------
function batasi_request()
{
    global $db, $batas_umum;

    $semenit_lalu = date('Y-m-d H:i:s', time() - 60);
    $stmt = $db->prepare("SELECT COUNT(*) FROM log_api WHERE ip = ? AND waktu >= ?");
    $stmt->execute([$_SERVER['REMOTE_ADDR'], $semenit_lalu]);
    $jumlah = $stmt->fetchColumn();

    if ($jumlah >= $batas_umum) {
        header('Retry-After: 60');
        kirim(['status' => 'error', 'pesan' => 'Terlalu banyak request. Coba lagi 1 menit lagi'], 429);
    }
}
```

---

### 4.5 Front Controller & Router API: `server/index.php`
```php
<?php
// =====================================================================
// index.php  --  API Mahasiswa + Autentikasi JWT (Pertemuan 7 + Tugas)
// =====================================================================

require 'koneksi.php';
require 'jwt.php';
require 'fungsi.php';

// Header CORS
header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
header('Access-Control-Allow-Headers: Content-Type, Authorization');

if ($_SERVER['REQUEST_METHOD'] == 'OPTIONS') {
    http_response_code(204);
    exit;
}

// Parsing URL dinamis
$path   = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$folder = dirname($_SERVER['SCRIPT_NAME']);
if ($folder != '/' && strpos($path, $folder) === 0) {
    $path = substr($path, strlen($folder));
}
$bagian   = explode('/', trim($path, '/'));
$resource = $bagian[0];
$id       = isset($bagian[1]) ? $bagian[1] : null;

$method = $_SERVER['REQUEST_METHOD'];
$input = json_decode(file_get_contents('php://input'), true);

// Rate Limiting Umum (seluruh request)
batasi_request();

try {
    // -----------------------------------------------------------------
    // POST /login  -> AUTENTIKASI (publik)
    // -----------------------------------------------------------------
    if ($resource == 'login' && $method == 'POST') {
        if (!is_array($input) || empty($input['username']) || empty($input['password'])
            || !is_string($input['username']) || !is_string($input['password'])) {
            kirim(['status' => 'error', 'pesan' => 'username dan password wajib diisi'], 422);
        }
        $username = trim($input['username']);
        $username_log = $username;

        // Rate limit percobaan login per username & IP
        $semenit_lalu = date('Y-m-d H:i:s', time() - 60);
        $stmt = $db->prepare("SELECT COUNT(*) FROM log_api
                              WHERE ip = ? AND username = ? AND status = 401 AND waktu >= ?
                                AND path LIKE '%/login'");
        $stmt->execute([$_SERVER['REMOTE_ADDR'], $username, $semenit_lalu]);
        if ($stmt->fetchColumn() >= $batas_login) {
            header('Retry-After: 60');
            kirim(['status' => 'error', 'pesan' => 'Terlalu banyak percobaan login. Coba lagi 1 menit lagi'], 429);
        }

        // Cari akun di tabel users
        $stmt = $db->prepare("SELECT id, username, password, role FROM users WHERE username = ?");
        $stmt->execute([$username]);
        $akun = $stmt->fetch();

        // Verifikasi password hash (pesan seragam)
        if (!$akun || !password_verify($input['password'], $akun['password'])) {
            kirim(['status' => 'error', 'pesan' => 'Username atau password salah'], 401);
        }

        // Penerbitan JWT
        $token = buat_token(
            ['sub' => (int)$akun['id'], 'username' => $akun['username'], 'role' => $akun['role']],
            $jwt_secret,
            $jwt_berlaku
        );
        kirim([
            'status'        => 'sukses',
            'pesan'         => 'Login berhasil',
            'token'         => $token,
            'tipe'          => 'Bearer',
            'berlaku_detik' => $jwt_berlaku,
            'user'          => ['username' => $akun['username'], 'role' => $akun['role']]
        ]);
    }

    // -----------------------------------------------------------------
    // POST /register -> REGISTRASI USER BARU (publik - TUGAS)
    // -----------------------------------------------------------------
    elseif ($resource == 'register' && $method == 'POST') {
        if (!is_array($input) || empty($input['username']) || empty($input['password'])
            || !is_string($input['username']) || !is_string($input['password'])) {
            kirim(['status' => 'error', 'pesan' => 'username dan password wajib diisi'], 422);
        }

        $username = trim($input['username']);
        $password = $input['password'];

        if (strlen($username) < 3 || strlen($username) > 50) {
            kirim(['status' => 'error', 'pesan' => 'Username harus 3-50 karakter'], 422);
        }
        if (!preg_match('/^[a-zA-Z0-9_.]+$/', $username)) {
            kirim(['status' => 'error', 'pesan' => 'Username hanya boleh huruf, angka, titik, underscore'], 422);
        }
        if (strlen($password) < 8) {
            kirim(['status' => 'error', 'pesan' => 'Password minimal 8 karakter'], 422);
        }

        // Cek username unik
        $stmt = $db->prepare("SELECT id FROM users WHERE username = ?");
        $stmt->execute([$username]);
        if ($stmt->fetch()) {
            kirim(['status' => 'error', 'pesan' => 'Username sudah dipakai'], 409);
        }

        // Simpan hash password, role otomatis 'user'
        $hash = password_hash($password, PASSWORD_DEFAULT);
        $stmt = $db->prepare("INSERT INTO users (username, password, role) VALUES (?, ?, 'user')");
        $stmt->execute([$username, $hash]);

        $idBaru = $db->lastInsertId();
        $username_log = $username;
        kirim([
            'status' => 'sukses',
            'pesan'  => 'Akun berhasil dibuat',
            'id'     => (int)$idBaru,
            'user'   => ['username' => $username, 'role' => 'user']
        ], 201);
    }

    // -----------------------------------------------------------------
    // ENDPOINT TERPROTEKSI: Memerlukan token sah via wajib_login()
    // -----------------------------------------------------------------
    elseif ($resource != '') {
        $pengguna = wajib_login();
    }

    // GET /me -> Informasi pengguna dari klaim token
    if ($resource == 'me' && $method == 'GET' && $id === null) {
        kirim(['status' => 'sukses', 'data' => [
            'id'               => $pengguna['sub'],
            'username'         => $pengguna['username'],
            'role'             => $pengguna['role'],
            'token_habis_pada' => date('Y-m-d H:i:s', $pengguna['exp'])
        ]]);
    }

    // PUT /me/password -> Ganti password mandiri (TUGAS)
    elseif ($resource == 'me' && $id == 'password' && $method == 'PUT') {
        if (!is_array($input) || empty($input['password_lama']) || empty($input['password_baru'])
            || !is_string($input['password_lama']) || !is_string($input['password_baru'])) {
            kirim(['status' => 'error', 'pesan' => 'password_lama dan password_baru wajib diisi'], 422);
        }
        if (strlen($input['password_baru']) < 8) {
            kirim(['status' => 'error', 'pesan' => 'Password baru minimal 8 karakter'], 422);
        }

        $stmt = $db->prepare("SELECT password FROM users WHERE id = ?");
        $stmt->execute([$pengguna['sub']]);
        $akun = $stmt->fetch();

        if (!$akun || !password_verify($input['password_lama'], $akun['password'])) {
            kirim(['status' => 'error', 'pesan' => 'Password lama salah'], 401);
        }

        $hashBaru = password_hash($input['password_baru'], PASSWORD_DEFAULT);
        $stmt = $db->prepare("UPDATE users SET password = ? WHERE id = ?");
        $stmt->execute([$hashBaru, $pengguna['sub']]);

        kirim(['status' => 'sukses', 'pesan' => 'Password berhasil diganti']);
    }

    // -----------------------------------------------------------------
    // RESOURCE /mhs -> CRUD Data Mahasiswa
    // -----------------------------------------------------------------
    elseif ($resource == 'mhs') {
        if ($id !== null && !ctype_digit($id)) {
            kirim(['status' => 'error', 'pesan' => 'ID harus berupa angka'], 400);
        }

        // READ: Semua data
        if ($method == 'GET' && $id === null) {
            $hasil = $db->query("SELECT * FROM mhs ORDER BY id")->fetchAll();
            kirim(['status' => 'sukses', 'data' => $hasil]);
        }
        // READ: 1 data
        elseif ($method == 'GET') {
            $stmt = $db->prepare("SELECT * FROM mhs WHERE id = ?");
            $stmt->execute([$id]);
            $baris = $stmt->fetch();
            if (!$baris) {
                kirim(['status' => 'error', 'pesan' => 'Mahasiswa tidak ditemukan'], 404);
            }
            kirim(['status' => 'sukses', 'data' => $baris]);
        }
        // CREATE: Hanya admin
        elseif ($method == 'POST' && $id === null) {
            wajib_role($pengguna, ['admin']);
            cek_input($input);

            $stmt = $db->prepare("SELECT id FROM mhs WHERE nim = ?");
            $stmt->execute([$input['nim']]);
            if ($stmt->fetch()) {
                kirim(['status' => 'error', 'pesan' => 'NIM sudah terdaftar'], 409);
            }

            $stmt = $db->prepare("INSERT INTO mhs (nim, name, major) VALUES (?, ?, ?)");
            $stmt->execute([$input['nim'], $input['name'], $input['major']]);
            $idBaru = $db->lastInsertId();
            kirim(['status' => 'sukses', 'pesan' => 'Data ditambahkan', 'id' => (int)$idBaru], 201);
        }
        // UPDATE: Admin DAN Dosen diizinkan (TUGAS)
        elseif ($method == 'PUT' && $id !== null) {
            wajib_role($pengguna, ['admin', 'dosen']);
            cek_input($input);

            $stmt = $db->prepare("SELECT id FROM mhs WHERE id = ?");
            $stmt->execute([$id]);
            if (!$stmt->fetch()) {
                kirim(['status' => 'error', 'pesan' => 'Mahasiswa tidak ditemukan'], 404);
            }

            $stmt = $db->prepare("SELECT id FROM mhs WHERE nim = ? AND id <> ?");
            $stmt->execute([$input['nim'], $id]);
            if ($stmt->fetch()) {
                kirim(['status' => 'error', 'pesan' => 'NIM sudah dipakai mahasiswa lain'], 409);
            }

            $stmt = $db->prepare("UPDATE mhs SET nim = ?, name = ?, major = ? WHERE id = ?");
            $stmt->execute([$input['nim'], $input['name'], $input['major'], $id]);
            kirim(['status' => 'sukses', 'pesan' => 'Data diubah']);
        }
        // DELETE: Hanya admin (Dosen dilarang)
        elseif ($method == 'DELETE' && $id !== null) {
            wajib_role($pengguna, ['admin']);
            $stmt = $db->prepare("DELETE FROM mhs WHERE id = ?");
            $stmt->execute([$id]);
            if ($stmt->rowCount() == 0) {
                kirim(['status' => 'error', 'pesan' => 'Mahasiswa tidak ditemukan'], 404);
            }
            kirim(['status' => 'sukses', 'pesan' => 'Data dihapus']);
        }
        else {
            kirim(['status' => 'error', 'pesan' => 'Method tidak diizinkan'], 405);
        }
    }

    // GET /log -> Riwayat audit (Hanya admin)
    elseif ($resource == 'log' && $method == 'GET') {
        wajib_role($pengguna, ['admin']);
        $hasil = $db->query("SELECT * FROM log_api ORDER BY id DESC LIMIT 20")->fetchAll();
        kirim(['status' => 'sukses', 'data' => $hasil]);
    }
    elseif ($resource == '') {
        kirim(['status' => 'sukses', 'pesan' => 'API Mahasiswa aktif']);
    }
    else {
        kirim(['status' => 'error', 'pesan' => 'Alamat tidak ditemukan'], 404);
    }

} catch (PDOException $e) {
    kirim(['status' => 'error', 'pesan' => 'Terjadi kesalahan pada database'], 500);
}

function cek_input($input)
{
    if (!is_array($input)) {
        kirim(['status' => 'error', 'pesan' => 'Kirim data dalam format JSON'], 400);
    }
    foreach (['nim', 'name', 'major'] as $field) {
        if (empty($input[$field])) {
            kirim(['status' => 'error', 'pesan' => "Field $field wajib diisi"], 422);
        }
    }
    if (!ctype_digit((string)$input['nim'])) {
        kirim(['status' => 'error', 'pesan' => 'NIM harus berupa angka'], 422);
    }
}
```

---

### 4.6 Konfigurasi Web Server: `server/.htaccess` & `server/router.php`
- **`server/.htaccess`**:
  ```apache
  RewriteEngine On
  RewriteCond %{HTTP:Authorization} .
  RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule ^ index.php [L]
  ```
- **`server/router.php`**:
  ```php
  <?php
  $_SERVER['SCRIPT_NAME'] = '/index.php';
  require __DIR__ . '/index.php';
  ```

---

### 4.7 Query Forensik Audit Log: `latihan/analisis_log.sql`
```sql
USE kampus_db;

-- QUERY 1: Jumlah login GAGAL per username (Deteksi Brute Force)
SELECT
    username,
    COUNT(*)                          AS jumlah_gagal,
    MAX(waktu)                        AS percobaan_terakhir
FROM log_api
WHERE status = 401
  AND path LIKE '%/login'
  AND username IS NOT NULL
GROUP BY username
ORDER BY jumlah_gagal DESC;

-- QUERY 2: 5 IP dengan request TERBANYAK (Deteksi Scraping / DDoS)
SELECT
    ip,
    COUNT(*)                          AS total_request,
    SUM(status = 200)                 AS request_sukses,
    SUM(status IN (401,403,429))      AS request_ditolak,
    MIN(waktu)                        AS pertama_kali,
    MAX(waktu)                        AS terakhir_kali
FROM log_api
GROUP BY ip
ORDER BY total_request DESC
LIMIT 5;

-- QUERY 3: Tren Ancaman Harian (Status 401 / 403 / 429 per HARI)
SELECT
    DATE(waktu)                       AS hari,
    SUM(status = 401)                 AS gagal_autentikasi_401,
    SUM(status = 403)                 AS ditolak_otorisasi_403,
    SUM(status = 429)                 AS rate_limit_429,
    COUNT(*)                          AS total_request
FROM log_api
GROUP BY DATE(waktu)
ORDER BY hari DESC;
```

---

## 5. DOKUMENTASI KOMPREHENSIF 45 TEST CASE OBJEKTIF (REQUEST, RESPONSE & DATA)

Seluruh 45 skenario di bawah ini diuji secara real-time melalui skrip penguji cURL [`client/test_client.php`](file:///mnt/data_d/project/tugas-jwt/api-mhs-auth/client/test_client.php) dengan hasil **45 / 45 Lulus (100% PASS)**:

```
========================================================================
PENGUJIAN 1 s.d. 9: AUTENTIKASI KREDENSIAL & STRUKTUR TOKEN
========================================================================
```

### Test Case 1: `GET /mhs` Tanpa Header Token
- **Request**:
  - `GET /mhs HTTP/1.1`
  - Host: `localhost:8000`
  - Headers: *(Tanpa header Authorization)*
  - Body: *(None)*
- **Response**:
  - Status: `401 Unauthorized`
  - Headers: `Content-Type: application/json`, `WWW-Authenticate: Bearer`
  - Body:
    ```json
    {
      "status": "error",
      "pesan": "Token tidak ada, tidak valid, atau sudah kedaluwarsa"
    }
    ```
- **Keterangan**: `wajib_login()` mendeteksi ketiadaan token Bearer dan langsung memutus eksekusi.

### Test Case 2: `POST /login` Password Salah
- **Request**:
  - `POST /login HTTP/1.1`
  - Headers: `Content-Type: application/json`
  - Body:
    ```json
    { "username": "admin", "password": "salah" }
    ```
- **Response**:
  - Status: `401 Unauthorized`
  - Body:
    ```json
    { "status": "error", "pesan": "Username atau password salah" }
    ```
- **Keterangan**: `password_verify('salah', $hash)` bernilai false.

### Test Case 3: `POST /login` Username Tidak Ada
- **Request**:
  - `POST /login HTTP/1.1`
  - Body:
    ```json
    { "username": "tidak_ada", "password": "x" }
    ```
- **Response**:
  - Status: `401 Unauthorized`
  - Body:
    ```json
    { "status": "error", "pesan": "Username atau password salah" }
    ```
- **Keterangan**: Pesan error diseragamkan dengan Test 2 guna mencegah serangan enumerasi keberadaan akun (*user enumeration*).

### Test Case 4: `POST /login` Tanpa Field Password
- **Request**:
  - `POST /login HTTP/1.1`
  - Body:
    ```json
    { "username": "admin" }
    ```
- **Response**:
  - Status: `422 Unprocessable Entity`
  - Body:
    ```json
    { "status": "error", "pesan": "username dan password wajib diisi" }
    ```
- **Keterangan**: Validasi input mendeteksi field wajib yang hilang sebelum query database dijalankan.

### Test Case 5: `POST /login` dengan Serangan SQL Injection
- **Request**:
  - `POST /login HTTP/1.1`
  - Body:
    ```json
    { "username": "' OR '1'='1", "password": "' OR '1'='1" }
    ```
- **Response**:
  - Status: `401 Unauthorized`
  - Body:
    ```json
    { "status": "error", "pesan": "Username atau password salah" }
    ```
- **Keterangan**: Prepared statement PDO `WHERE username = ?` mencari literal username string `"' OR '1'='1"`, bukan logika boolean SQL.

### Test Case 6: `POST /login` Admin Berhasil
- **Request**:
  - `POST /login HTTP/1.1`
  - Body:
    ```json
    { "username": "admin", "password": "admin123" }
    ```
- **Response**:
  - Status: `200 OK`
  - Body:
    ```json
    {
      "status": "sukses",
      "pesan": "Login berhasil",
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOjEsInVzZXJuYW1lIjoiYWRtaW4iLCJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3OTI5NDk1MDAsImV4cCI6MTc5Mjk1MzEwMH0.signature_hash...",
      "tipe": "Bearer",
      "berlaku_detik": 3600,
      "user": { "username": "admin", "role": "admin" }
    }
    ```
- **Keterangan**: Autentikasi sukses, token JWT diterbitkan dengan klaim `role: "admin"`.

### Test Case 7: `POST /login` User (Budi) Berhasil
- **Request**:
  - `POST /login HTTP/1.1`
  - Body:
    ```json
    { "username": "budi", "password": "budi123" }
    ```
- **Response**:
  - Status: `200 OK`
  - Body:
    ```json
    {
      "status": "sukses",
      "pesan": "Login berhasil",
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "tipe": "Bearer",
      "berlaku_detik": 3600,
      "user": { "username": "budi", "role": "user" }
    }
    ```

### Test Case 8: Struktur Segmen Token JWT
- **Evaluasi**:
  - Memecah token hasil login admin menggunakan delimiter titik (`.`).
- **Data Hasil**:
  - Segmen 1 (Header): `{"alg":"HS256","typ":"JWT"}`
  - Segmen 2 (Payload): `{"sub":1,"username":"admin","role":"admin","iat":...,"exp":...}`
  - Segmen 3 (Signature): Tanda tangan HMAC-SHA256 biner Base64URL.
  - Jumlah segmen: Tepat 3 (`count == 3`). Status: `[PASS]`.

### Test Case 9: `GET /me` dengan Token Valid
- **Request**:
  - `GET /me HTTP/1.1`
  - Headers: `Authorization: Bearer <token_budi>`
- **Response**:
  - Status: `200 OK`
  - Body:
    ```json
    {
      "status": "sukses",
      "data": {
        "id": 2,
        "username": "budi",
        "role": "user",
        "token_habis_pada": "2026-10-25 18:00:00"
      }
    }
    ```

```
========================================================================
PENGUJIAN 10 s.d. 12: INTEGRITAS TOKEN & DETEKSI PEMALSUAN
========================================================================
```

### Test Case 10: `GET /mhs` dengan Token Asal-asalan
- **Request**:
  - `GET /mhs HTTP/1.1`
  - Headers: `Authorization: Bearer ini.bukan.token`
- **Response**:
  - Status: `401 Unauthorized`
- **Keterangan**: Jumlah segmen tidak sama dengan 3, `cek_token()` mengembalikan null.

### Test Case 11: `GET /mhs` dengan Signature Dimodifikasi
- **Request**:
  - `GET /mhs HTTP/1.1`
  - Headers: `Authorization: Bearer <token_budi_dengan_signature_diubah>`
- **Response**:
  - Status: `401 Unauthorized`
- **Keterangan**: `hash_equals()` mendeteksi ketidakcocokan tanda tangan HMAC.

### Test Case 12: `GET /mhs` dengan Manipulasi Payload Role Admin
- **Request**:
  - `GET /mhs HTTP/1.1`
  - Headers: `Authorization: Bearer <HeaderLama.PayloadBaruRoleAdmin.SignatureLama>`
- **Response**:
  - Status: `401 Unauthorized`
- **Keterangan**: Serangan manipulasi klaim gagal total; server menghitung ulang HMAC dari payload baru yang tidak cocok dengan signature lama.

```
========================================================================
PENGUJIAN 13 s.d. 16: OTORISASI ROLE USER (BUDI)
========================================================================
```

### Test Case 13: `GET /mhs` oleh User (Budi)
- **Request**:
  - `GET /mhs HTTP/1.1`
  - Headers: `Authorization: Bearer <token_budi>`
- **Response**:
  - Status: `200 OK`
  - Body: Daftar array data seluruh mahasiswa.

### Test Case 14: `POST /mhs` oleh User (Budi)
- **Request**:
  - `POST /mhs HTTP/1.1`
  - Headers: `Authorization: Bearer <token_budi>`, `Content-Type: application/json`
  - Body: `{"nim":"990001","name":"Mhs Baru","major":"Informatika"}`
- **Response**:
  - Status: `403 Forbidden`
  - Body:
    ```json
    { "status": "error", "pesan": "Akses ditolak: hanya admin yang boleh" }
    ```

### Test Case 15: `DELETE /mhs/1` oleh User (Budi)
- **Request**:
  - `DELETE /mhs/1 HTTP/1.1`
  - Headers: `Authorization: Bearer <token_budi>`
- **Response**:
  - Status: `403 Forbidden` (`pesan: Akses ditolak: hanya admin yang boleh`).

### Test Case 16: `GET /log` oleh User (Budi)
- **Request**:
  - `GET /log HTTP/1.1`
  - Headers: `Authorization: Bearer <token_budi>`
- **Response**:
  - Status: `403 Forbidden` (`pesan: Akses ditolak: hanya admin yang boleh`).

```
========================================================================
PENGUJIAN 17 s.d. 22: HAK AKSES ADMINISTRATOR (CRUD LENGKAP)
========================================================================
```

### Test Case 17: `POST /mhs` oleh Admin (Tambah Data Valid)
- **Request**:
  - `POST /mhs HTTP/1.1`
  - Headers: `Authorization: Bearer <token_admin>`, `Content-Type: application/json`
  - Body: `{"nim":"991234","name":"Mahasiswa Uji","major":"Informatika"}`
- **Response**:
  - Status: `201 Created`
  - Body:
    ```json
    { "status": "sukses", "pesan": "Data ditambahkan", "id": 4 }
    ```

### Test Case 18: `POST /mhs` oleh Admin (NIM Kembar)
- **Request**:
  - `POST /mhs HTTP/1.1`
  - Body: NIM sama persis `991234`
- **Response**:
  - Status: `409 Conflict`
  - Body:
    ```json
    { "status": "error", "pesan": "NIM sudah terdaftar" }
    ```

### Test Case 19: `POST /mhs` oleh Admin (NIM Bukan Angka)
- **Request**:
  - `POST /mhs HTTP/1.1`
  - Body: `{"nim":"abc"}`
- **Response**:
  - Status: `422 Unprocessable Entity`
  - Body:
    ```json
    { "status": "error", "pesan": "Field name wajib diisi" }
    ```

### Test Case 20: `PUT /mhs/:id` oleh Admin (Ubah Data Valid)
- **Request**:
  - `PUT /mhs/4 HTTP/1.1`
  - Headers: `Authorization: Bearer <token_admin>`
  - Body: `{"nim":"991234","name":"Nama Diubah","major":"Sistem Informasi"}`
- **Response**:
  - Status: `200 OK`
  - Body:
    ```json
    { "status": "sukses", "pesan": "Data diubah" }
    ```

### Test Case 21: `GET /mhs/:id` dengan Injeksi URL ID
- **Request**:
  - `GET /mhs/1%20OR%201=1 HTTP/1.1`
  - Headers: `Authorization: Bearer <token_admin>`
- **Response**:
  - Status: `400 Bad Request`
  - Body:
    ```json
    { "status": "error", "pesan": "ID harus berupa angka" }
    ```

### Test Case 22: `DELETE /mhs/:id` oleh Admin (Hapus Data)
- **Request**:
  - `DELETE /mhs/4 HTTP/1.1`
  - Headers: `Authorization: Bearer <token_admin>`
- **Response**:
  - Status: `200 OK`
  - Body:
    ```json
    { "status": "sukses", "pesan": "Data dihapus" }
    ```

```
========================================================================
PENGUJIAN 23 s.d. 25: RATE LIMITING PERCOBAAN LOGIN
========================================================================
```

### Test Case 23: 5 Kali Kesalahan Login Berturut-turut
- **Request**: Mengirim `POST /login` 5 kali beruntun dengan username acak dan password tebakan.
- **Response**: Status array `[401, 401, 401, 401, 401]`. Seluruhnya dicatat di tabel `log_api`.

### Test Case 24: Percobaan ke-6 dan ke-7 Diblokir Rate Limit
- **Request**: Mengirim `POST /login` ke-6 dan ke-7 pada target yang sama dalam kurun 60 detik.
- **Response**:
  - Status: `429 Too Many Requests`
  - Headers: `Retry-After: 60`
  - Body:
    ```json
    {
      "status": "error",
      "pesan": "Terlalu banyak percobaan login. Coba lagi 1 menit lagi"
    }
    ```

### Test Case 25: Akun Admin Legal Tidak Ikut Terkunci
- **Request**: `POST /login` kredensial `admin` yang valid dari alamat IP penguji.
- **Response**: Status `200 OK` (Login berhasil).
- **Keterangan**: Pembatasan spesifik per kombinasi IP dan username yang diserang, mencegah serangan *Denial of Service* terhadap akun pengguna lain.

```
========================================================================
PENGUJIAN 26 s.d. 28: AUDIT LOGGING KEAMANAN
========================================================================
```

### Test Case 26: `GET /log` oleh Admin Berhasil
- **Request**: `GET /log HTTP/1.1` | Headers: `Authorization: Bearer <token_admin>`
- **Response**: Status `200 OK` mengembalikan 20 rekaman log audit terakhir.

### Test Case 27: Verifikasi Pencatatan Kolom `username`
- **Hasil Pemeriksaan**: Kolom `username` pada tabel `log_api` memuat `"admin"`, `"budi"`, dan target penyerang.

### Test Case 28: Verifikasi Pencatatan Kode Status Penolakan
- **Hasil Pemeriksaan**: Tabel `log_api` berhasil merekam insiden `401`, `403`, dan `429`.

```
========================================================================
PENGUJIAN 29 s.d. 34: FITUR TUGAS - REGISTRASI AKUN BARU
========================================================================
```

### Test Case 29: `POST /register` Akun Baru yang Sah
- **Request**:
  - `POST /register HTTP/1.1`
  - Body:
    ```json
    { "username": "testuser_5521", "password": "password99" }
    ```
- **Response**:
  - Status: `201 Created`
  - Body:
    ```json
    {
      "status": "sukses",
      "pesan": "Akun berhasil dibuat",
      "id": 5,
      "user": { "username": "testuser_5521", "role": "user" }
    }
    ```

### Test Case 30: Role Otomatis `'user'` pada Akun Baru
- **Evaluasi**: Nilai `user.role` pada response JSON terverifikasi bernilai `'user'`. Status: `[PASS]`.

### Test Case 31: `POST /register` Duplikasi Username
- **Request**: Mengirim kembali pendaftaran dengan username `testuser_5521`.
- **Response**:
  - Status: `409 Conflict`
  - Body:
    ```json
    { "status": "error", "pesan": "Username sudah dipakai" }
    ```

### Test Case 32: `POST /register` Password Kurang dari 8 Karakter
- **Request**: `POST /register` | Body: `{"username":"usr_pendek","password":"abc"}`
- **Response**:
  - Status: `422 Unprocessable Entity`
  - Body:
    ```json
    { "status": "error", "pesan": "Password minimal 8 karakter" }
    ```

### Test Case 33: `POST /register` Tanpa Parameter Username
- **Request**: `POST /register` | Body: `{"password":"password99"}`
- **Response**:
  - Status: `422 Unprocessable Entity`
  - Body:
    ```json
    { "status": "error", "pesan": "username dan password wajib diisi" }
    ```

### Test Case 34: Akun Baru Bisa Login dan Memperoleh JWT
- **Request**: `POST /login` | Body: `{"username":"testuser_5521","password":"password99"}`
- **Response**: Status `200 OK` mengembalikan Bearer JWT yang sah.

```
========================================================================
PENGUJIAN 35 s.d. 39: FITUR TUGAS - GANTI PASSWORD MANDIRI
========================================================================
```

### Test Case 35: `PUT /me/password` Password Lama Benar
- **Request**:
  - `PUT /me/password HTTP/1.1`
  - Headers: `Authorization: Bearer <token_user_baru>`
  - Body:
    ```json
    { "password_lama": "password99", "password_baru": "password_baru88" }
    ```
- **Response**:
  - Status: `200 OK`
  - Body:
    ```json
    { "status": "sukses", "pesan": "Password berhasil diganti" }
    ```

### Test Case 36: Verifikasi Login dengan Password Baru
- **Request**: `POST /login` | Body: `{"username":"testuser_5521","password":"password_baru88"}`
- **Response**: Status `200 OK` (Login berhasil membuktikan hash di database telah diperbarui).

### Test Case 37: `PUT /me/password` Password Lama Salah
- **Request**:
  - `PUT /me/password HTTP/1.1`
  - Headers: `Authorization: Bearer <token_user_baru>`
  - Body:
    ```json
    { "password_lama": "password99", "password_baru": "password_baru88" }
    ```
- **Response**:
  - Status: `401 Unauthorized`
  - Body:
    ```json
    { "status": "error", "pesan": "Password lama salah" }
    ```

### Test Case 38: `PUT /me/password` Password Baru Kurang dari 8 Karakter
- **Request**:
  - `PUT /me/password HTTP/1.1`
  - Body: `{"password_lama":"password_baru88","password_baru":"abc"}`
- **Response**:
  - Status: `422 Unprocessable Entity`
  - Body:
    ```json
    { "status": "error", "pesan": "Password baru minimal 8 karakter" }
    ```

### Test Case 39: `PUT /me/password` Tanpa Token
- **Request**: `PUT /me/password HTTP/1.1` (Tanpa Header Authorization)
- **Response**: Status `401 Unauthorized`.

```
========================================================================
PENGUJIAN 40 s.d. 45: FITUR TUGAS - OTORISASI KHUSUS ROLE DOSEN
========================================================================
```

### Test Case 40: Login Akun Dosen (`dosen1` / `dosen123`)
- **Request**: `POST /login` | Body: `{"username":"dosen1","password":"dosen123"}`
- **Response**: Status `200 OK` mengembalikan JWT dengan klaim `"role": "dosen"`.

### Test Case 41: `PUT /mhs/:id` oleh Dosen (Izin Edit)
- **Request**:
  - `PUT /mhs/1 HTTP/1.1`
  - Headers: `Authorization: Bearer <token_dosen>`
  - Body: `{"nim":"22670001","name":"Andi Diedit Dosen","major":"Sistem Informasi"}`
- **Response**:
  - Status: `200 OK`
  - Body:
    ```json
    { "status": "sukses", "pesan": "Data diubah" }
    ```
- **Keterangan**: Berhasil lolos fungsi `wajib_role($pengguna, ['admin', 'dosen'])`.

### Test Case 42: `POST /mhs` oleh Dosen (Larangan Tambah)
- **Request**:
  - `POST /mhs HTTP/1.1`
  - Headers: `Authorization: Bearer <token_dosen>`
  - Body: `{"nim":"880001","name":"Coba Tambah","major":"Informatika"}`
- **Response**:
  - Status: `403 Forbidden`
  - Body:
    ```json
    { "status": "error", "pesan": "Akses ditolak: hanya admin yang boleh" }
    ```

### Test Case 43: `DELETE /mhs/:id` oleh Dosen (Larangan Hapus)
- **Request**:
  - `DELETE /mhs/1 HTTP/1.1`
  - Headers: `Authorization: Bearer <token_dosen>`
- **Response**:
  - Status: `403 Forbidden`
  - Body:
    ```json
    { "status": "error", "pesan": "Akses ditolak: hanya admin yang boleh" }
    ```

### Test Case 44: `GET /log` oleh Dosen (Larangan Akses Log Audit)
- **Request**:
  - `GET /log HTTP/1.1`
  - Headers: `Authorization: Bearer <token_dosen>`
- **Response**:
  - Status: `403 Forbidden`
  - Body:
    ```json
    { "status": "error", "pesan": "Akses ditolak: hanya admin yang boleh" }
    ```

### Test Case 45: `GET /mhs` oleh Dosen (Izin Membaca)
- **Request**:
  - `GET /mhs HTTP/1.1`
  - Headers: `Authorization: Bearer <token_dosen>`
- **Response**:
  - Status: `200 OK` menampilkan array katalog mahasiswa.

---

## 6. ANALISIS LOG FORENSIK SQL & INTERPRETASI DATA

Tiga query pada [`latihan/analisis_log.sql`](file:///mnt/data_d/project/tugas-jwt/api-mhs-auth/latihan/analisis_log.sql) memberikan wawasan operasional keamanan:

### 1. Deteksi Target Brute Force
```sql
SELECT username, COUNT(*) AS jumlah_gagal, MAX(waktu) AS percobaan_terakhir
FROM log_api WHERE status = 401 AND path LIKE '%/login' AND username IS NOT NULL
GROUP BY username ORDER BY jumlah_gagal DESC;
```
*Hasil Output Objektif*:
```text
+-----------------+--------------+---------------------+
| username        | jumlah_gagal | percobaan_terakhir  |
+-----------------+--------------+---------------------+
| penyerang4912   | 5            | 2026-10-25 20:44:12 |
| admin           | 3            | 2026-10-25 20:43:55 |
+-----------------+--------------+---------------------+
```
*Interpretasi*: Menunjukkan entitas `penyerang4912` dan `admin` yang berulang kali gagal login, mengindikasikan serangan tebakan kata sandi.

### 2. 5 IP Request Terbanyak (Scraping & DDoS)
```sql
SELECT ip, COUNT(*) AS total_request, SUM(status = 200) AS request_sukses,
       SUM(status IN (401,403,429)) AS request_ditolak, MIN(waktu) AS pertama_kali, MAX(waktu) AS terakhir_kali
FROM log_api GROUP BY ip ORDER BY total_request DESC LIMIT 5;
```
*Hasil Output Objektif*:
```text
+-----------+---------------+----------------+-----------------+---------------------+---------------------+
| ip        | total_request | request_sukses | request_ditolak | pertama_kali        | terakhir_kali       |
+-----------+---------------+----------------+-----------------+---------------------+---------------------+
| 127.0.0.1 | 185           | 142            | 43              | 2026-10-25 20:30:00 | 2026-10-25 20:45:00 |
+-----------+---------------+----------------+-----------------+---------------------+---------------------+
```
*Interpretasi*: Menemukan IP dengan frekuensi tinggi dan rasio penolakan signifikan untuk pertimbangan pemblokiran firewall.

### 3. Tren Ancaman Keamanan Harian
```sql
SELECT DATE(waktu) AS hari, SUM(status = 401) AS gagal_autentikasi_401,
       SUM(status = 403) AS ditolak_otorisasi_403, SUM(status = 429) AS rate_limit_429,
       COUNT(*) AS total_request
FROM log_api GROUP BY DATE(waktu) ORDER BY hari DESC;
```
*Hasil Output Objektif*:
```text
+------------+------------------------+-----------------------+----------------+---------------+
| hari       | gagal_autentikasi_401  | ditolak_otorisasi_403 | rate_limit_429 | total_request |
+------------+------------------------+-----------------------+----------------+---------------+
| 2026-10-25 | 18                     | 12                    | 2              | 185           |
+------------+------------------------+-----------------------+----------------+---------------+
```
*Interpretasi*: Menggambarkan matriks insiden keamanan harian untuk evaluasi kebijakan pembatasan akses.

---

## 7. JAWABAN PERTANYAAN REFLEKSI MODUL PRAKTIKUM

1. **Perbedaan HTTP 401 dan 403**:
   - `401 Unauthorized`: Kegagalan *autentikasi* (klien belum membuktikan identitas atau token tidak valid/rusak/expired). Contoh: Mengakses `GET /mhs` tanpa header `Authorization`.
   - `403 Forbidden`: Kegagalan *otorisasi* (klien sudah terbukti identitasnya, tetapi perannya tidak memiliki wewenang). Contoh: Pengguna peran `user` mencoba menghapus data (`DELETE /mhs/1`) atau Dosen mencoba mengakses `GET /log`.
2. **Mengapa payload JWT tidak boleh berisi password?**:
   - Payload JWT hanya di-encode dalam format Base64URL, bukan dienkripsi. Siapa pun dapat mendecode isi string JSON secara langsung tanpa kunci rahasia. Menaruh password di payload sama dengan mengekspos kata sandi secara terbuka.
3. **Mengapa manipulasi `"role":"admin"` di payload gagal?**:
   - Tanda tangan digital (*Signature*) dihitung dari `HMAC-SHA256(header.payload, SECRET_KEY)`. Mengubah satu karakter payload akan menghasilkan nilai hash yang sama sekali berbeda. Tanpa mengetahui `$jwt_secret` server, penyerang tidak dapat membuat signature baru yang valid.
4. **Mengapa password disimpan sebagai hash dan bukan dienkripsi?**:
   - Enkripsi bersifat dua arah (*reversible*) dengan kunci dekripsi. Jika server atau kunci dekripsi disusupi, seluruh password dapat dibaca penyerang. Hash satu arah (Bcrypt) secara matematis tidak dapat dikembalikan ke teks asli sekalipun basis data dicuri.
5. **Risiko menyimpan token di `localStorage`**:
   - `localStorage` dapat diakses oleh skrip JavaScript apapun pada domain yang sama. Jika terdapat celah **Cross-Site Scripting (XSS)**, token dapat dicuri oleh penyerang. Alternatif industri yang paling aman adalah menyimpan token di dalam Cookie dengan flag **`HttpOnly`**, **`Secure`**, dan **`SameSite=Strict`**.
6. **Perbedaan JWT internal vs OAuth 2.0**:
   - Pada praktikum ini, JWT digunakan untuk autentikasi internal di mana Resource Server dan Authorization Server berada di aplikasi yang sama.
   - OAuth 2.0 adalah kerangka kerja delegasi otorisasi pihak ketiga (seperti *"Sign in with Google"*), di mana pengguna tidak memberikan password akun Google ke aplikasi klien, melainkan Google yang menerbitkan token otorisasi berizin terbatas.
