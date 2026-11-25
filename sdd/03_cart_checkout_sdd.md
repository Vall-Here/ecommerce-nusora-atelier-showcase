# 03. Persistent Cart & 3-Column Checkout Wizard

## 1. Ruang Lingkup & Komponen Aktual
- **Models & Services**: `App\Domain\Cart\Models\{Cart, CartItem}`, `App\Domain\Cart\Services\CartManager`
- **Livewire UI**: `Storefront\{CartDrawer, CartTrigger, CheckoutWizard}`

## 2. Use Case Diagram

```mermaid
flowchart LR
    Guest["Pembeli Tamu"]
    Member["Pelanggan Terdaftar"]

    subgraph CartCheckoutModule ["Modul Cart & Checkout"]
        UC1["Tambah / Ubah Qty Item di Keranjang"]
        UC2["Sinkronisasi Keranjang Tamu ke Akun saat Login"]
        UC3["Isi Alamat & Pilih Kurir JNE / J&T"]
        UC4["Terapkan Kode Kupon & Hitung PPN 11%"]
        UC5["Pratinjau Mode Demo (Berhenti di Checkout)"]
    end

    Guest --> UC1
    Member --> UC1
    Member --> UC2
    Guest --> UC3
    Member --> UC3
    Guest --> UC4
    Member --> UC4
    Guest --> UC5
```

## 3. Activity Diagram

```mermaid
flowchart TD
    Start([Buka /checkout]) --> LoadCart[Muat Cart & Hitung Subtotal]
    LoadCart --> CheckMember{Sudah Login?}
    CheckMember -- Ya --> FillAddr[Muat Buku Alamat Default]
    CheckMember -- Tamu --> InputAddr[Input Alamat Manual]
    FillAddr --> SelectCourier[Pilih Kurir JNE/J&T & Layanan]
    InputAddr --> SelectCourier
    SelectCourier --> ApplyPromo[Opsional: Validasi Kupon via PromotionEvaluatorService]
    ApplyPromo --> ClickPay[Klik Buat Pesanan & Bayar]
    ClickPay --> IsDemo{demo_mode Aktif?}
    IsDemo -- Ya --> ShowDemoNotice[Tampilkan Notif Mode Demo & Hentikan Transaksi]
    IsDemo -- Tidak --> RunPlaceOrder[Eksekusi PlaceOrderAction + Atomic Cache Lock]
```

## 4. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as Pembeli
    participant Wizard as CheckoutWizard
    participant Lock as Cache Atomic Lock
    participant Ship as ShippingRateService
    participant Tax as TaxCalculatorService
    participant OrderAct as PlaceOrderAction

    Buyer->>Wizard: Klik Buat Pesanan
    Wizard->>Wizard: Validasi input & cek BusinessSetting demo_mode
    Wizard->>Lock: acquire checkout_lock_{userOrSession} (10s)
    Wizard->>Ship: calculateShippingFee(cart, carrier, service, city)
    Wizard->>Tax: calculateTax(subtotal - discount)
    Wizard->>OrderAct: execute(...)
    OrderAct-->>Wizard: Order ORD-XXXXXX
    Wizard-->>Buyer: Redirect ke /order/{orderNumber}
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    USERS ||--o{ CARTS : "owns"
    CARTS ||--|{ CART_ITEMS : "contains"
    PRODUCT_VARIANTS ||--o{ CART_ITEMS : "referenced_by"

    CARTS {
        bigint id PK
        char ulid UK
        bigint user_id FK
        string session_id
        string coupon_code
        timestamp expires_at
    }

    CART_ITEMS {
        bigint id PK
        bigint cart_id FK
        bigint product_id FK
        bigint variant_id FK
        integer quantity
    }
```
