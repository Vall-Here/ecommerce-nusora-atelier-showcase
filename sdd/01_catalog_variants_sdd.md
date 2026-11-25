# 01. Catalog, Dynamic JSONB Attributes & Variant Matrix

## 1. Ruang Lingkup & Komponen Aktual
- **Models**: `App\Domain\Catalog\Models\{Product, ProductVariant, ProductImage, Category}`
- **Actions**: `App\Domain\Catalog\Actions\GenerateVariantMatrixAction`
- **Livewire UI**: `Storefront\{Home, ProductCatalog, ProductDetail}`, `Admin\Products\{ProductList, ProductForm}`, `Admin\Categories\CategoryManager`

## 2. Use Case Diagram

```mermaid
flowchart LR
    Buyer["Pembeli Publik"]
    Admin["Admin Katalog"]

    subgraph CatalogModule ["Modul Catalog & Variant Matrix"]
        UC1["Jelajahi Katalog & Filter Multi-Dimensi"]
        UC2["Lihat Console PDP & Sinkronisasi Foto Warna"]
        UC3["Kelola Hierarki Kategori (Anti-Loop)"]
        UC4["Generate Matriks Varian Warna x Ukuran"]
        UC5["Kelola Atribut Kain JSONB (GSM, Bahan, Size Chart)"]
        UC6["Unggah & Atur Foto Utama Galeri Produk"]
    end

    Buyer --> UC1
    Buyer --> UC2
    Admin --> UC3
    Admin --> UC4
    Admin --> UC5
    Admin --> UC6
```

## 3. Activity Diagram

```mermaid
flowchart TD
    Start([Buka ProductForm]) --> InputBase[Isi Nama, Slug, Kategori, Harga Cent & Atribut JSONB]
    InputBase --> GenMatrix[Jalankan GenerateVariantMatrixAction]
    GenMatrix --> LoopColor[Iterasi Daftar Warna x Ukuran]
    LoopColor --> CheckSku{SKU Varian Sudah Ada?}
    CheckSku -- Ya --> SkipVar[Pertahankan Varian Eksisting]
    CheckSku -- Tidak --> CreateVar[Buat ProductVariant Baru]
    SkipVar --> UploadImg[Unggah Foto & Tandai is_primary]
    CreateVar --> UploadImg
    UploadImg --> SaveProd[Simpan Product & Invalidate Cache Tags]
    SaveProd --> EndDone([Produk Siap Ditampilkan di Storefront])
```

## 4. Sequence Diagram (PDP Color-Image Sync)

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as Pembeli
    participant PDP as ProductDetail (Livewire 4)
    participant Cache as StorefrontCacheService
    participant DB as PostgreSQL 16

    Buyer->>PDP: Klik Swatch Warna (selectColor)
    PDP->>Cache: rememberProduct(slug)
    Cache->>DB: SELECT products, variants, images
    DB-->>PDP: Product + Variants + Images
    PDP->>PDP: Cocokkan variant_id warna atau alt_text dengan nama warna
    PDP-->>Buyer: Perbarui activeImageId & tampilkan stok varian aktif
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    CATEGORIES ||--o{ CATEGORIES : "parent_of"
    CATEGORIES ||--o{ PRODUCTS : "categorizes"
    PRODUCTS ||--|{ PRODUCT_VARIANTS : "has_matrix"
    PRODUCTS ||--o{ PRODUCT_IMAGES : "has_gallery"
    PRODUCT_VARIANTS ||--o{ PRODUCT_IMAGES : "variant_photo"

    CATEGORIES {
        bigint id PK
        bigint parent_id FK
        json name
        string slug UK
        integer position
        boolean is_active
    }

    PRODUCTS {
        bigint id PK
        char ulid UK
        bigint category_id FK
        string sku UK
        json name
        string slug UK
        bigint base_price_amount
        jsonb attributes "GIN Indexed"
        boolean has_variants
        boolean is_active
    }

    PRODUCT_VARIANTS {
        bigint id PK
        bigint product_id FK
        string sku UK
        string name
        json option_values
        bigint price_amount
        integer stock_quantity
        integer weight_grams
        boolean is_active
    }

    PRODUCT_IMAGES {
        bigint id PK
        bigint product_id FK
        bigint variant_id FK
        string image_path
        string alt_text
        integer position
        boolean is_primary
    }
```
