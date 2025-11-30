````markdown
# 🐘 PHP Security Deep Dive

Panduan teknis keamanan untuk ekosistem PHP Modern (PHP 8+). PHP sering menjadi target serangan karena popularitasnya, namun dengan praktik yang benar, PHP bisa menjadi sangat aman.

---

## 📋 Daftar Isi
1. [SQL Injection (The King of Vulnerabilities)](#1-sql-injection-the-king-of-vulnerabilities)
2. [Cross-Site Scripting (XSS) Mitigation](#2-cross-site-scripting-xss-mitigation)
3. [Session Security & Hijacking](#3-session-security--hijacking)
4. [File Upload yang Aman (Mencegah RCE)](#4-file-upload-yang-aman-mencegah-rce)
5. [Hardening php.ini](#5-hardening-phpini)
6. [Dependency Management](#6-dependency-management)

---

## 1. SQL Injection (The King of Vulnerabilities)

Kesalahan paling klasik dan fatal di PHP adalah menggabungkan variabel user langsung ke dalam string query database.

### ❌ Kode Berbahaya (Vulnerable)
```php
$email = $_POST['email'];
// Hacker input: ' OR '1'='1
$sql = "SELECT * FROM users WHERE email = '$email'"; 
$result = $conn->query($sql);
````

### ✅ Solusi: Prepared Statements (PDO)

Gunakan **PDO (PHP Data Objects)**. Database akan memisahkan antara perintah SQL dan data, sehingga input hacker diperlakukan sebagai teks biasa, bukan perintah.

```php
$email = $_POST['email'];

// 1. Prepare
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');

// 2. Bind & Execute
$stmt->execute(['email' => $email]);

// 3. Fetch
$user = $stmt->fetch();
```

-----

## 2\. Cross-Site Scripting (XSS) Mitigation

PHP tidak melakukan *escaping* otomatis kecuali Anda menggunakan template engine (seperti Blade di Laravel atau Twig). Jika menggunakan PHP Native:

### ❌ Vulnerable Output

```php
// Jika nama user adalah <script>alert('Hacked')</script>
echo "Halo, " . $user['name']; 
```

### ✅ Solusi: htmlspecialchars()

Selalu encode karakter khusus HTML sebelum ditampilkan ke browser.

```php
// Mengubah < menjadi &lt;, > menjadi &gt;, dll.
echo "Halo, " . htmlspecialchars($user['name'], ENT_QUOTES, 'UTF-8');
```

-----

## 3\. Session Security & Hijacking

Session PHP default tersimpan di file server. Jika tidak dikonfigurasi dengan benar, session bisa dicuri (Hijacking) atau ditetapkan paksa (Fixation).

### ✅ Konfigurasi Aman (Wajib di awal script)

```php
// Konfigurasi cookie agar tidak bisa diakses JS dan hanya via HTTPS
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_secure', 1); // Aktifkan jika menggunakan HTTPS
ini_set('session.use_only_cookies', 1);

session_start();

// Mencegah Session Fixation:
// Ganti ID session setiap kali user login atau level akses berubah
if ($user_just_logged_in) {
    session_regenerate_id(true); // true = hapus session lama
    $_SESSION['user_id'] = $new_user_id;
}
```

-----

## 4\. File Upload yang Aman (Mencegah RCE)

Fitur upload file adalah pintu masuk favorit hacker untuk menanam **Web Shell / Backdoor**.

**Daftar Periksa Keamanan Upload:**

1.  **Jangan percaya `$_FILES['file']['type']`**: Header ini dikirim oleh browser dan bisa dipalsukan. Gunakan `finfo` (File Info) dari server.
2.  **Ganti Nama File**: Jangan gunakan nama asli dari user. Gunakan hash atau `uniqid()`.
3.  **Whitelist Ekstensi**: Hanya izinkan ekstensi tertentu.

### ✅ Contoh Implementasi Aman

```php
$target_dir = "uploads/";

// 1. Cek Error Upload
if ($_FILES['upfile']['error'] !== UPLOAD_ERR_OK) {
    die("Upload gagal.");
}

// 2. Validasi MIME Type (Cek isi file sebenarnya)
$finfo = new finfo(FILEINFO_MIME_TYPE);
$mime = $finfo->file($_FILES['upfile']['tmp_name']);

$allowed_mimes = [
    'jpg' => 'image/jpeg',
    'png' => 'image/png',
    'pdf' => 'application/pdf'
];

if (!in_array($mime, $allowed_mimes)) {
    die("Format file tidak valid / berbahaya.");
}

// 3. Ganti Nama File (Randomize)
$ext = array_search($mime, $allowed_mimes);
$new_filename = sprintf('%s.%s', sha1_file($_FILES['upfile']['tmp_name']), $ext);

// 4. Pindahkan
if (!move_uploaded_file($_FILES['upfile']['tmp_name'], $target_dir . $new_filename)) {
    die("Gagal memindahkan file.");
}

echo "File berhasil diupload dengan aman.";
```

-----

## 5\. Hardening php.ini

Konfigurasi server PHP memegang peranan penting. Pastikan setting berikut diterapkan di `php.ini` server production:

| Setting | Value | Penjelasan |
| :--- | :--- | :--- |
| `expose_php` | `Off` | Menyembunyikan versi PHP di header HTTP (Security by Obscurity). |
| `display_errors` | `Off` | Jangan tampilkan error coding ke user. |
| `log_errors` | `On` | Catat error ke file log server. |
| `allow_url_fopen` | `Off` | Mencegah PHP membuka file dari URL remote (RFI Protection). |
| `session.cookie_httponly` | `1` | Mencegah XSS mencuri session cookie. |

-----

## 6\. Dependency Management

Jangan copy-paste library manual. Gunakan **Composer**.

### ✅ Mencegah Vulnerable Packages

Gunakan plugin `roave/security-advisories` yang mencegah Anda menginstall library versi lama yang memiliki celah keamanan.

```bash
composer require --dev roave/security-advisories:dev-latest
```

Jika Anda mencoba menginstall library yang vulnerable, Composer akan menolak instalasi tersebut.

-----

> **Catatan Penting:** Keamanan PHP sangat bergantung pada bagaimana kode ditulis. Selalu validasi input di awal, dan encode output di akhir.

````
