# 02. Inventory Concurrency, TTL Reservations & Stock Opname

## 1. Ruang Lingkup & Komponen Aktual
- **Models**: `App\Domain\Inventory\Models\{InventoryReservation, InventoryLog}`
- **Actions**: `ReserveInventoryAction`, `ConvertReservationToSaleAction`, `ReleaseExpiredReservationsAction`, `AdjustInventoryAction`
- **Scheduler**: `inventory:cleanup-expired-reservations` (tiap menit), `inventory:check-low-stock` (tiap jam)
- **Livewire UI**: `Admin\Inventory\ReservationControl` (3 Tab: Reservasi Aktif, Stok Menipis & Opname, Riwayat Mutasi)

## 2. Use Case Diagram

```mermaid
flowchart LR
    Checkout["Checkout Engine"]
    Cron["Laravel Scheduler"]
    WhAdmin["Admin Gudang"]

    subgraph InventoryModule ["Modul Inventory & Opname"]
        UC1["Kunci Stok Sementara 15 Menit (lockForUpdate)"]
        UC2["Konversi Reservasi ke Penjualan Permanen"]
        UC3["Pelepasan Otomatis Reservasi Kedaluwarsa"]
        UC4["Input Restock Batch & Stok Opname Fisik"]
        UC5["Pantau Peringatan Stok Menipis & Ekspor Ledger CSV"]
    end

    Checkout --> UC1
    Checkout --> UC2
    Cron --> UC3
    WhAdmin --> UC3
    WhAdmin --> UC4
    WhAdmin --> UC5
```

## 3. Activity Diagram (Pessimistic Reservation)

```mermaid
flowchart TD
    Start([Request Reservasi Item]) --> BeginTx[Mulai DB::transaction]
    BeginTx --> LockVar[SELECT product_variants FOR UPDATE]
    LockVar --> CalcReserved[Hitung SUM quantity reservasi aktif belum expired]
    CalcReserved --> CalcAvail[available = stock_quantity - reserved]
    CalcAvail --> Enough{available >= requestedQty?}
    Enough -- Tidak --> ThrowErr[Throw InsufficientStockException & Rollback]
    Enough -- Ya --> InsertRes[INSERT inventory_reservations TTL 15 menit]
    InsertRes --> CommitTx[Commit Transaction]
    CommitTx --> EndDone([Stok Terkunci Aman])
```

## 4. Sequence Diagram (Stock Opname & Audit Ledger)

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Admin Gudang
    participant UI as ReservationControl (Tab Opname)
    participant Act as AdjustInventoryAction
    participant DB as PostgreSQL 16

    Admin->>UI: Input stok fisik aktual, klik Sesuaikan
    UI->>Act: execute(variantId, delta, reason: stock_opname)
    rect rgb(245, 245, 245)
        Act->>DB: DB::transaction + SELECT product_variants FOR UPDATE
        Act->>DB: UPDATE product_variants SET stock_quantity = actual
        Act->>DB: INSERT inventory_logs (before, change, after, note)
    end
    Act-->>UI: ProductVariant updated
    UI-->>Admin: Toast sukses & perbarui tabel ledger
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    PRODUCT_VARIANTS ||--o{ INVENTORY_RESERVATIONS : "locked_by"
    PRODUCT_VARIANTS ||--o{ INVENTORY_LOGS : "audited_in"
    PRODUCTS ||--o{ INVENTORY_LOGS : "tracked_in"

    INVENTORY_RESERVATIONS {
        bigint id PK
        bigint variant_id FK
        bigint order_id
        string session_id
        integer quantity
        string status "reserved | converted | released"
        timestamp expires_at
    }

    INVENTORY_LOGS {
        bigint id PK
        bigint product_id FK
        bigint variant_id FK
        string type "sale | restock | manual_adjustment | order_cancellation | reservation_release"
        integer quantity_before
        integer quantity_change
        integer quantity_after
        string reference_type
        bigint reference_id
        text note
        timestamp created_at
    }
```
