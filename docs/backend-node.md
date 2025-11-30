````markdown
# 🟢 Node.js (Express) Security Deep Dive

Panduan teknis untuk mengamankan aplikasi Backend berbasis Node.js dan Express. Fokus pada hardening server dan pencegahan serangan umum.

---

## 📋 Daftar Isi
1. [Secure HTTP Headers (Helmet)](#1-secure-http-headers-helmet)
2. [Rate Limiting (Anti Brute-Force)](#2-rate-limiting-anti-brute-force)
3. [Input Validation & Sanitization](#3-input-validation--sanitization)
4. [NoSQL Injection (MongoDB)](#4-nosql-injection-mongodb)
5. [Error Handling (Information Leakage)](#5-error-handling-information-leakage)
6. [Secure Authentication](#6-secure-authentication)

---

## 1. Secure HTTP Headers (Helmet)

Express.js secara default tidak mengatur HTTP headers keamanan. Gunakan `helmet` untuk menutup celah seperti Clickjacking, Sniffing, dan XSS.

**Instalasi:**
```bash
npm install helmet
````

**✅ Implementasi:**

```javascript
const express = require('express');
const helmet = require('helmet');
const app = express();

// Wajib diletakkan di paling atas middleware stack!
app.use(helmet());

// Secara default, helmet mengaktifkan:
// - dnsPrefetchControl
// - frameguard (X-Frame-Options: SAMEORIGIN) -> Cegah Clickjacking
// - hidePoweredBy (Menyembunyikan 'X-Powered-By: Express')
// - hsts (Strict-Transport-Security) -> Paksa HTTPS
// - xssFilter (X-XSS-Protection)
```

-----

## 2\. Rate Limiting (Anti Brute-Force)

Mencegah serangan DDoS level aplikasi dan Brute-Force password dengan membatasi jumlah request dari satu IP.

**Instalasi:**

```bash
npm install express-rate-limit
```

**✅ Implementasi:**

```javascript
const rateLimit = require('express-rate-limit');

// Limiter umum untuk seluruh aplikasi
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 menit
  max: 100, // Limit setiap IP maksimal 100 request
  message: 'Terlalu banyak request dari IP ini, silakan coba lagi nanti.',
  standardHeaders: true, // Return rate limit info di headers `RateLimit-*`
  legacyHeaders: false, // Disable header `X-RateLimit-*`
});

app.use(globalLimiter);

// Limiter khusus untuk Login (Lebih ketat)
const loginLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 jam
  max: 5, // Maksimal 5 kali percobaan login gagal
  message: 'Terlalu banyak percobaan login, akun dikunci sementara.'
});

app.use('/api/auth/login', loginLimiter);
```

-----

## 3\. Input Validation & Sanitization

Jangan pernah percaya input user (`req.body`, `req.query`, `req.params`). Validasi struktur data sebelum diproses.

**Rekomendasi Library:** `Joi` atau `Zod`.

**✅ Contoh dengan Joi:**

```bash
npm install joi
```

```javascript
const Joi = require('joi');

const registerSchema = Joi.object({
  username: Joi.string().alphanum().min(3).max(30).required(),
  email: Joi.string().email().required(),
  password: Joi.string().pattern(new RegExp('^[a-zA-Z0-9]{3,30}$')).required(),
});

app.post('/api/register', (req, res) => {
  const { error } = registerSchema.validate(req.body);
  
  if (error) {
    // Return 400 Bad Request jika validasi gagal
    return res.status(400).json({ error: error.details[0].message });
  }

  // Lanjut proses registrasi...
});
```

-----

## 4\. NoSQL Injection (MongoDB)

Jika menggunakan MongoDB, input seperti `{"$gt": ""}` di kolom password bisa membypass login tanpa mengetahui password.

**Instalasi:**

```bash
npm install express-mongo-sanitize
```

**✅ Implementasi:**

```javascript
const mongoSanitize = require('express-mongo-sanitize');

app.use(express.json());

// Sanitasi data: Menghapus key yang berawalan '$' atau mengandung '.'
app.use(mongoSanitize());

// Atau replace karakter berbahaya dengan karakter lain
app.use(mongoSanitize({
  replaceWith: '_'
}));
```

-----

## 5\. Error Handling (Information Leakage)

Jangan pernah menampilkan *Stack Trace* (detail error coding) kepada user di mode production. Hacker bisa melihat struktur folder dan library yang digunakan.

**❌ Vulnerable Code:**

```javascript
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack }); // Bahaya!
});
```

**✅ Secure Code:**

```javascript
app.use((err, req, res, next) => {
  console.error(err.stack); // Log error di server console/logging system

  const response = {
    message: 'Something went wrong!',
  };

  // Hanya tampilkan detail error jika di mode development
  if (process.env.NODE_ENV === 'development') {
    response.error = err.message;
    response.stack = err.stack;
  }

  res.status(500).json(response);
});
```

-----

## 6\. Secure Authentication

Tips keamanan password:

1.  **Hashing:** Jangan simpan password plain text. Gunakan `bcryptjs` atau `argon2`.
2.  **Timing Attack:** Gunakan fungsi pembanding yang aman (bcrypt `compare`).

**✅ Contoh Bcrypt:**

```bash
npm install bcryptjs
```

```javascript
const bcrypt = require('bcryptjs');

// Hashing saat Register
const salt = await bcrypt.genSalt(10);
const hashedPassword = await bcrypt.hash(req.body.password, salt);

// Verifikasi saat Login
const validPass = await bcrypt.compare(req.body.password, user.password);
if (!validPass) return res.status(400).send('Invalid password');
```

-----

> **Pro Tip:** Selalu gunakan file `.env` untuk menyimpan konfigurasi sensitif (DB URL, JWT Secret) dan masukkan `.env` ke `.gitignore`.

```
```
