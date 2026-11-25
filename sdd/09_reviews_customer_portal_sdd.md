# 09. Verified Fit Reviews & Customer Portal

## 1. Ruang Lingkup & Komponen Aktual
- **Models**: `ProductReview`, `Address`, `User`
- **Storefront UI**: `CustomerLogin`, `CustomerRegister`, `CustomerForgotPassword`, `CustomerResetPassword`, `CustomerVerifyEmail`, `CustomerProfile`, `CustomerAddresses`, `CustomerOrders`, `CustomerWishlist`, `ReviewSubmissionModal`
- **Admin UI**: `Admin\Reviews\ReviewModeration`

## 2. Use Case Diagram

```mermaid
flowchart LR
    Visitor["Pengunjung"]
    Member["Pelanggan"]
    Moderator["Admin Moderasi"]

    subgraph CustomerModule ["Modul Reviews & Customer Portal"]
        UC1["Registrasi & Pemulihan Sandi (Dikontrol allow_registration)"]
        UC2["Kelola Profil Ukuran, Buku Alamat & Wishlist"]
        UC3["Tulis Ulasan Terverifikasi + Foto + Fit Gauge"]
        UC4["Voting Ulasan Membantu (Helpful Votes)"]
        UC5["Moderasi & Balas Ulasan dari Admin"]
    end

    Visitor --> UC1
    Member --> UC2
    Member --> UC3
    Visitor --> UC4
    Moderator --> UC5
```

## 3. Activity Diagram

```mermaid
flowchart TD
    Start([Akses /daftar]) --> CheckReg{allow_registration == true?}
    CheckReg -- False --> ShowClosed[Tampilkan Pendaftaran Ditutup]
    CheckReg -- True --> Validate[Validasi Input + RateLimiter 5/5m]
    Validate --> CreateUser[Buat User Role customer + Kirim Verifikasi Email]
    CreateUser --> MergeCart[Merge Guest Cart & Wishlist ke User]
    MergeCart --> Portal([Masuk ke /akun/pesanan])
```

## 4. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Member as Pembeli
    participant Modal as ReviewSubmissionModal
    participant DB as PostgreSQL 16
    participant Admin as ReviewModeration

    Member->>Modal: Submit rating, tinggi/berat, fit & foto ulasan
    Modal->>Modal: Cek RateLimiter + Order status == Completed
    Modal->>DB: INSERT product_reviews (is_verified_purchase: true)
    Admin->>DB: Approve ulasan & tulis admin_reply
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    USERS ||--o{ ADDRESSES : "saves"
    USERS ||--o{ PRODUCT_REVIEWS : "writes"
    PRODUCTS ||--o{ PRODUCT_REVIEWS : "reviewed_on"

    ADDRESSES {
        bigint id PK
        bigint user_id FK
        string recipient_name
        string phone
        string address_line1
        string city
        string postal_code
        boolean is_default
    }

    PRODUCT_REVIEWS {
        bigint id PK
        bigint product_id FK
        bigint user_id FK
        bigint order_id FK
        integer rating
        string fit_feedback "small | true_to_size | large"
        integer height_cm
        integer weight_kg
        json photos
        boolean is_approved
        text admin_reply
    }
```
