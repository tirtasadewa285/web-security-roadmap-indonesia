# ⚛️ React.js Security Deep Dive

## 1. Cross-Site Scripting (XSS)
React cukup aman secara default karena melakukan escaping string. Namun, developer sering membuka celah melalui `dangerouslySetInnerHTML`.

### 🔴 Kode Berbahaya (Vulnerable)
```jsx
// Jika 'userContent' berisi script alert, script itu akan jalan.
<div dangerouslySetInnerHTML={{ __html: userContent }} />
