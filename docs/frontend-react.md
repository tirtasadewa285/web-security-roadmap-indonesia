````markdown
# ⚛️ React.js Security Deep Dive

Panduan lengkap pengamanan aplikasi Frontend berbasis React.js. Meskipun React memiliki mekanisme keamanan bawaan, kesalahan implementasi developer seringkali membuka celah fatal.

---

## 📋 Daftar Isi
1. [Mencegah Cross-Site Scripting (XSS)](#1-mencegah-cross-site-scripting-xss)
2. [Penyimpanan Token & Autentikasi](#2-penyimpanan-token--autentikasi)
3. https://nordvpn.com/cybersecurity/glossary/url-injection/(#3-url-injection--redirects)
4. [Dependensi & Supply Chain](#4-dependensi--supply-chain)
5. [Sensitive Data Exposure](#5-sensitive-data-exposure)
6. [Tools Keamanan React](#6-tools-keamanan-react)

---

## 1. Mencegah Cross-Site Scripting (XSS)

React secara default melakukan *auto-escaping* pada data yang dirender menggunakan JSX (`{variable}`). Namun, ada beberapa skenario di mana fitur ini bisa di-bypass.

### A. Bahaya `dangerouslySetInnerHTML`
Nama properti ini sudah menjadi peringatan. Jangan gunakan kecuali benar-benar perlu (misal: render konten blog dari CMS).

**❌ Vulnerable Code:**
```jsx
const UserBio = ({ content }) => {
  // Jika content berisi: <img src=x onerror=alert(1)>
  // Script akan dieksekusi!
  return <div dangerouslySetInnerHTML={{ __html: content }} />;
};
````

**✅ Secure Code (Sanitization):**
Gunakan library `dompurify` untuk membersihkan HTML string sebelum dirender.

```bash
npm install dompurify
```

```jsx
import DOMPurify from 'dompurify';

const UserBio = ({ content }) => {
  const cleanContent = DOMPurify.sanitize(content);
  return <div dangerouslySetInnerHTML={{ __html: cleanContent }} />;
};
```

### B. Props Spreading

Hati-hati saat menyebarkan props (`{...props}`) ke elemen dasar HTML, karena user bisa menyisipkan atribut event handler berbahaya.

**❌ Berisiko:**

```jsx
// Jika props berisi { onMouseOver: alert('XSS') }
<a {...props}>Click me</a>
```

**✅ Solusi:**
Definisikan props secara eksplisit atau whitelist props yang diizinkan.

-----

## 2\. Penyimpanan Token & Autentikasi

Salah satu perdebatan terbesar: Di mana menyimpan JWT (Access Token)?

### A. LocalStorage / SessionStorage (❌ Tidak Direkomendasikan)

Menyimpan token di `localStorage` sangat mudah, tetapi **tidak aman** terhadap XSS.

  * **Risiko:** Jika hacker berhasil menyisipkan script (XSS) di halaman mana pun, mereka bisa membaca seluruh isi `localStorage` dengan `window.localStorage` dan mencuri token user.

### B. HttpOnly Cookies (✅ Direkomendasikan)

Simpan token di dalam Cookie dengan flag `HttpOnly`.

  * **Keunggulan:** Cookie ini **tidak bisa dibaca oleh JavaScript** (`document.cookie` akan kosong), sehingga token aman meskipun web terkena XSS.
  * **Catatan:** Anda perlu mengatur strategi mitigasi CSRF (seperti menggunakan SameSite cookie atau CSRF Token) jika menggunakan metode ini.

-----

## 3\. URL Injection & Redirects

Hacker bisa menggunakan skema URL `javascript:` untuk mengeksekusi kode saat user mengklik link.

**❌ Vulnerable Code:**

```jsx
// Jika userProfile.website = "javascript:alert('Hacked')"
<a href={userProfile.website}>Kunjungi Website</a>
```

**✅ Secure Code:**
Validasi protokol URL sebelum merender. Pastikan hanya `http:` atau `https:`.

```jsx
const validateUrl = (url) => {
  const parsed = new URL(url);
  return ['http:', 'https:'].includes(parsed.protocol) ? url : '#';
};

<a href={validateUrl(userProfile.website)}>Kunjungi Website</a>
```

-----

## 4\. Dependensi & Supply Chain

Frontend modern bergantung pada ribuan paket npm. Satu paket berbahaya bisa mengkompromikan seluruh aplikasi.

**Langkah Pencegahan:**

1.  **Rutin Audit:**
    Jalankan perintah ini sebelum setiap deployment:
    ```bash
    npm audit
    ```
2.  **Kunci Versi:**
    Gunakan `package-lock.json` atau `yarn.lock` untuk memastikan versi package konsisten di semua environment.
3.  **Review Package Baru:**
    Jangan asal install package yang jarang di-maintain atau memiliki sedikit download.

-----

## 5\. Sensitive Data Exposure

Jangan pernah menyimpan "rahasia" di kode frontend. Ingat: **Apapun yang dikirim ke browser bisa dilihat oleh user.**

**❌ Jangan lakukan ini di `.env` frontend:**

```properties
REACT_APP_STRIPE_SECRET_KEY=sk_live_123456...
REACT_APP_AWS_SECRET=AKIA...
```

*Siapapun bisa klik kanan \> "View Source" atau inspect bundle JS untuk menemukan kunci ini.*

**✅ Solusi:**
Simpan API Key rahasia di Backend (Node/PHP). Frontend hanya memanggil endpoint backend, lalu backend yang berkomunikasi dengan layanan pihak ketiga (Stripe/AWS).

-----

## 6\. Tools Keamanan React

Gunakan alat bantu otomatis untuk menjaga kualitas kode:

1.  **ESLint Plugin Security:**
    Mendeteksi pola koding yang tidak aman.

    ```bash
    npm install eslint-plugin-security --save-dev
    ```

2.  **Snyk:**
    Scanning kerentanan pada dependencies secara mendalam.

3.  **React Scanner:**
    Mencari penggunaan props atau komponen yang deprecated/unsafe.

-----

> **Ingat:** Keamanan Frontend adalah garis pertahanan pertama, namun validasi sesungguhnya (Authorization & Validation) **wajib** dilakukan kembali di sisi Backend.

```
```
