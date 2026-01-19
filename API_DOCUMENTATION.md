# Dokumentasi API & Database E-Commerce

Dokumen ini menjelaskan struktur API, role akses, library yang digunakan, dan skema database untuk backend e-commerce ini.

## Teknologi & Library
- **Runtime**: Node.js
- **Framework**: Express.js
- **Database**: MongoDB (via Mongoose)
- **Autentikasi**: JSON Web Token (JWT)
- **Upload File**: Multer
- **Payment Gateway**: Midtrans Client
- **Hashing**: Bcryptjs
- **Environment**: Dotenv
- **Slug**: Slugify
- **CORS**: Cors

---

## Skema Database (Mongoose Models)

### 1. User
Menyimpan data pengguna (admin dan customer).
- **Schema**: `username`, `email`, `password` (hashed), `full_name`, `phone_number`, `address`, `role` (`admin` / `customer`).

### 2. Product
Menyimpan informasi utama produk.
- **Schema**: `name`, `slug`, `description`, `category_id` (ref Category), `price_min`, `price_max`, `total_stock`, `thumbnail`, `status`.

### 3. ProductVariant
Menyimpan variasi spesifik dari produk (misal: ukuran, warna).
- **Schema**: `product_id` (ref Product), `sku`, `attributes` (Map), `price`, `stock`, `weight`, `is_active`.

### 4. ProductImage
Menyimpan gambar-gambar produk dan varian.
- **Schema**: `product_id`, `variant_id`, `image_url`, `is_primary`, `sort_order`.

### 5. Category
Kategori produk.
- **Schema**: `name`, `slug`, `description`.

### 6. Cart
Keranjang belanja user.
- **Schema**: `user_id`, `variant_id`, `quantity`, `is_checked`.

### 7. Order
Pesanan utama.
- **Schema**: `user_id`, `order_number`, `total_amount`, `shipping_address`, `status` (pending/paid/shipped/etc), `snap_token` (Midtrans), `payment_status`.

### 8. OrderItem
Detail item dalam pesanan (snapshot harga/produk saat beli).
- **Schema**: `order_id`, `variant_id`, `product_name`, `quantity`, `price`, `subtotal`.

### 9. PaymentLog
Log transaksi pembayaran dari Midtrans (untuk audit/debugging).
- **Schema**: `order_id`, `transaction_id`, `status_code`, `gross_amount`, `raw_notification`.

---

## API Routes & Akses Role

Base URL: `/jshope`

### 1. Authentication (`/jshope/auth`)
| Method | URL | Akses | Fungsi | Library/Middleware |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/register` | Public | Mendaftar user baru (default role: customer) | `bcryptjs` |
| `POST` | `/login` | Public | Login dan mendapatkan JWT Token | `bcryptjs`, `jsonwebtoken` |

### 2. Products (`/jshope/product`)
| Method | URL | Akses | Fungsi | Library/Middleware |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/` | Public | Mengambil semua produk aktif | `mongoose` |
| `GET` | `/:id` | Public | Mengambil detail produk (termasuk varian & gambar) | `mongoose` |
| `POST` | `/` | **Admin** | Membuat produk baru (support upload gambar/video) | `auth`, `multer`, `slugify` |
| `PUT` | `/:id` | **Admin** | Mengupdate produk (support upload gambar/video) | `auth`, `multer`, `slugify` |
| `DELETE` | `/:id` | **Admin** | Menghapus produk, varian, dan gambar terkait | `auth` |

### 3. Categories (`/jshope/categories`)
| Method | URL | Akses | Fungsi | Middleware |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/` | Public | List semua kategori | - |
| `GET` | `/:id` | Public | Detail kategori | - |
| `POST` | `/` | **Admin** | Tambah kategori baru | `auth` |
| `PUT` | `/:id` | **Admin** | Edit kategori | `auth` |
| `DELETE` | `/:id` | **Admin** | Hapus kategori | `auth` |

### 4. Cart (`/jshope/cart`)
| Method | URL | Akses | Fungsi | Middleware |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/` | **Customer** | Lihat isi keranjang user login | `auth` |
| `POST` | `/add` | **Customer** | Tambah item ke keranjang | `auth` |
| `PATCH` | `/:id/increase` | **Customer** | Tambah qty item (+1) | `auth` |
| `PATCH` | `/:id/decrease` | **Customer** | Kurangi qty item (-1) | `auth` |
| `PATCH` | `/:id/toggle-checked` | **Customer** | Pilih/Hapus pilihan item untuk checkout | `auth` |
| `DELETE` | `/:id` | **Customer** | Hapus item dari keranjang | `auth` |

### 5. Orders (`/jshope/orders`)
| Method | URL | Akses | Fungsi | Middleware |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/` | **Customer** | History pesanan user login | `auth` |
| `GET` | `/review` | **Customer** | Review item yang akan di-checkout (dari cart) | `auth` |
| `POST` | `/checkout` | **Customer** | Buat pesanan baru & dapatkan Snap Token | `auth`, `midtrans-client` |
| `GET` | `/:id` | **Customer** | Detail pesanan spesifik | `auth` |
| `PATCH` | `/:id/pay` | **Customer** | Update status setelah bayar (opsional/manual) | `auth` |
| `PATCH` | `/:id/cancel` | **Customer** | Batalkan pesanan | `auth` |
| `GET` | `/admin/all` | **Admin** | Lihat semua pesanan (semua user) | `auth` |
| `GET` | `/admin/:id` | **Admin** | Detail pesanan (view admin) | `auth` |
| `PATCH` | `/admin/:id/status` | **Admin** | Update status pesanan (kirim resi, dll) | `auth` |

### 6. Payment / Midtrans (`/jshope/midtrans`)
| Method | URL | Akses | Fungsi | Library/Middleware |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/notification` | **Public** | Webhook dari Midtrans (status update otomatis) | `midtrans-client` |
| `GET` | `/client-key` | Public | Ambil Client Key untuk frontend | - |
| `GET` | `/status/:orderNumber` | User/Admin | Cek status transaksi ke Midtrans API | `auth`, `midtrans-client` |
| `POST` | `/cancel/:orderNumber` | **Admin** | Batalkan transaksi di Midtrans | `auth`, `midtrans-client` |

### 7. Dashboard (`/jshope/dashboard`)
| Method | URL | Akses | Fungsi | Middleware |
| :--- | :--- | :--- | :--- | :--- |
| `GET` | `/` | **Admin** | Statistik penjualan, total user, order, dll | `auth` |

---
**Catatan Keamanan:**
- Middleware `auth` memverifikasi token JWT.
- Controller untuk route Admin melakuan pengecekan tambahan `if (req.user.role !== 'admin')`.
