# 04. Orders, State Machine & Immutable Adjustments

## 1. Ruang Lingkup & Komponen Aktual
- **Models & Enums**: `Order`, `OrderItem`, `OrderAdjustment`, `OrderStatus`
- **Actions**: `PlaceOrderAction`, `TransitionOrderStatusAction`, `CancelOrderAction`
- **Invoice**: `OrderInvoiceController` + `resources/views/storefront/invoice.blade.php` (A4 Printable)

## 2. Use Case Diagram

```mermaid
flowchart LR
    Buyer["Pembeli"]
    Admin["Admin OMS"]

    subgraph OrderModule ["Modul Order & State Machine"]
        UC1["Buat Order & Snapshot Produk"]
        UC2["Transisi Status Pesanan Terproteksi"]
        UC3["Kunci Permanen OrderAdjustments saat Confirmed"]
        UC4["Cetak Faktur Penjualan A4 Resmi"]
        UC5["Batalkan Order & Pulihkan Stok"]
    end

    Buyer --> UC1
    Buyer --> UC4
    Admin --> UC2
    Admin --> UC3
    Admin --> UC4
    Admin --> UC5
```

## 3. Activity Diagram (Order State Machine)

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> PendingPayment
    PendingPayment --> Confirmed : Webhook Paid / Konfirmasi Admin
    PendingPayment --> Cancelled : Expired / Batal
    Confirmed --> Processing : Mulai Kemas
    Processing --> Shipped : CreateShipmentAction
    Processing --> Refunded : Refund Khusus
    Shipped --> Completed : Paket Diterima (Delivered)
    Confirmed --> Cancelled : CancelOrderAction (Restore Stock)
```

## 4. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Caller as Webhook / Admin OrderDetail
    participant Trans as TransitionOrderStatusAction
    participant Conv as ConvertReservationToSaleAction
    participant DB as PostgreSQL 16

    Caller->>Trans: execute(order, OrderStatus::Confirmed)
    rect rgb(245, 245, 245)
        Trans->>DB: DB::transaction + Lock Order
        Trans->>DB: UPDATE order_adjustments SET is_locked = true
        Trans->>Conv: Konversi reservasi -> kurangi stock_quantity + tulis InventoryLog
        Trans->>DB: UPDATE orders SET status = 'confirmed', confirmed_at = now()
    end
    Trans-->>Caller: Dispatch OrderConfirmedEvent (Queued Email)
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    ORDERS ||--|{ ORDER_ITEMS : "contains"
    ORDERS ||--o{ ORDER_ADJUSTMENTS : "adjusted_by"

    ORDERS {
        bigint id PK
        char ulid UK
        string order_number UK
        bigint customer_id FK
        string status
        bigint subtotal_amount
        bigint adjustments_total_amount
        bigint total_amount
        json metadata
    }

    ORDER_ITEMS {
        bigint id PK
        bigint order_id FK
        bigint product_id FK
        bigint variant_id FK
        json product_snapshot
        integer quantity
        bigint unit_price_amount
        bigint total_price_amount
    }

    ORDER_ADJUSTMENTS {
        bigint id PK
        bigint order_id FK
        string type "tax | promotion | shipping | fee"
        string label
        bigint amount
        boolean is_locked
    }
```
