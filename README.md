# Halaman Isolir Klien PPPoE dengan MikroTik & Docker 

## 1. Tujuan Proyek

Menyajikan **halaman isolir** kepada klien PPPoE di MikroTik yang berada dalam subnet tertentu. 
Halaman disajikan oleh **lighttpd dalam Docker**, dan routing dilakukan via config MikroTik.  
**README fokus pada logika isolir via MikroTik**, bukan setup lighttpd.

## 2. Komponen Utama

- **Layanan lighttpd** via Docker Compose (lihat dokumentasi terkait) – cukup menyajikan file HTML.
- **Docker network eksternal** (`cloudflared_net`) untuk integrasi.
- **Konfigurasi MikroTik**: 
  - **ROS6**: mendukung `redirect-to` dalam `ip proxy access`.
  - **ROS7**: tidak ada opsi `redirect-to`; perlu workaround seperti mengganti halaman error atau menggunakan NAT ke webserver khusus.

## 3. Mekanisme Isolir (aliran logika)

1. Klien dari subnet target mengakses halaman HTTP (`:80`).
2. MikroTik **dst-nat redirect** ke port proxy lokal (transparan).
3. Proxy menentukan apabila `src-address` sesuai subnet, klien diarahkan ke halaman isolir.
4. Halaman isolir disajikan oleh Docker/lighttpd.


## 4. Konfigurasi MikroTik ROS6 (contoh)

    // Enable proxy & transparent redirect
    /ip proxy set enabled=yes port=8080
    /ip firewall nat add chain=dstnat protocol=tcp src-address=<SUBNET> dst-port=80 action=redirect to-ports=8080
    /ip proxy access add src-address=<SUBNET> action=deny redirect-to="http://<IP_LIGHTTPD_OR_HOSTNAME>"


`redirect-to` (tersedia di ROS6) mengarahkan langsung ke page isolir.
 

## 5. Konfigurasi MikroTik ROS7 – Workaround **tanpa** `redirect-to`

RouterOS 7 masih memiliki fitur `redirect-to` dalam `ip proxy access`, tetapi jika periodenya tidak bekerja secara konsisten atau dihilangkan, solusi alternatif:

### A. Transparent proxy + custom error page

1. **Enable proxy & NAT redirect:**
    ```
    /ip proxy set enabled=yes port=8080
    /ip firewall nat add chain=dstnat protocol=tcp src-address=<SUBNET> dst-port=80 action=redirect to-ports=8080
    ```
2. **Reset dan edit halaman error proxy:**
- Reset HTML default:
  ```
  /ip proxy reset-html
  ```
- Ganti isi `error.html` di direktori `webproxy/error.html` dengan HTML isolir kamu. Saat klien diblokir via proxy access, halaman ini otomatis tampil :contentReference[oaicite:2]{index=2}.

3. **Tambahkan rule blocking di proxy access:**
    ```
    /ip proxy access add src-address=<SUBNET> action=deny
    ```

Dengan cara ini, klien dari subnet tertentu akan melihat halaman isolir saat permintaan HTTP disaring/ditolak oleh proxy.

### B. (Opsional) Script RouterOS untuk force redirect

Jika kamu ingin kontrol lebih manual, kamu bisa pakai script yang memantau koneksi dan meng-redirect NAT ke webserver lighttpd. Namun untuk implementasi umum, opsi A sudah cukup andal dan mudah.

---

## 6. Template `index.html` Halaman Isolir

Simpan file di folder `html/index.html`:

```html
<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<title>Isolir Layanan</title>
<style>
 body { font-family: Arial, sans-serif; text-align: center; padding-top: 50px; }
 .container { max-width: 600px; margin: auto; }
</style>
</head>
<body>
<div class="container">
 <h1>Akses Anda Terisolir</h1>
 <p>Silakan hubungi administrator untuk aktivasi kembali layanan.</p>
</div>
</body>
</html>
```

## 7. Langkah Penerapan
- Siapkan html/index.html dengan konten isolir.
- Jalankan Docker Compose untuk lighttpd (dokumentasi terpisah).
- Di MikroTik ROS7:
- Aktifkan proxy dan redirect NAT.
- Reset dan ganti halaman error.html.
- Tambahkan ip proxy access deny rule untuk subnet target.
- Uji klien di subnet isolasi — akses HTTP akan menampilkan halaman isolir.

## 8. Catatan Penting 
- ROS7 tidak butuh redirect-to, karena digantikan oleh custom error page.
- HTTPS tidak bisa disisipi redirect secara transparan tanpa sertifikat khusus—hal ini terbatas oleh desain TLS dan keamanan browser  
