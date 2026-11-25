# 08. After-Sales Returns, 7-Day Size Exchange & RMA

## 1. Ruang Lingkup & Komponen Aktual
- **Models & Enums**: `OrderReturn`, `ReturnClaimType` (`size_exchange`, `defect_qc`, `refund`), `ReturnStatus`
- **Actions**: `ProcessWarehouseQcAction`, `CreateExchangeOrderAction`
- **UI**: `Storefront\{RmaRequestForm, SizeGuideModal}`, `Admin\Returns\RmaManager`

## 2. Use Case Diagram

```mermaid
flowchart LR
    Buyer["Pembeli (Member / Tamu)"]
    AdminRMA["Admin Retur & Gudang QC"]

    subgraph RmaModule ["Modul Returns & RMA 7 Hari"]
        UC1["Cek Kelayakan Garansi 7 Hari"]
        UC2["Ajukan Tukar Ukuran (Varian Produk Sama)"]
        UC3["Inspeksi QC Gudang & Restock Otomatis"]
        UC4["Terbitkan Order Pengganti & Resi Baru"]
    end

    Buyer --> UC1
    Buyer --> UC2
    AdminRMA --> UC3
    AdminRMA --> UC4
```

## 3. Activity Diagram

```mermaid
flowchart TD
    Start([Ajukan Retur /akun/retur/ajukan]) --> CheckOrder{Order Completed & <= 7 Hari?}
    CheckOrder -- Tidak --> Deny[Tolak Pengajuan RMA]
    CheckOrder -- Ya --> CheckSameProd{Target Varian == Product ID Sama?}
    CheckSameProd -- Tidak --> DenyCross[Tolak Cross-Product Injection]
    CheckSameProd -- Ya --> CreateRMA[Terbitkan Tiket RMA pending_review]
    CreateRMA --> AdminReview[Admin QC Verifikasi Barang Masuk]
    AdminReview --> Decision{Tipe Klaim?}
    Decision -- size_exchange --> Exchange[CreateExchangeOrderAction]
    Decision -- refund --> Refund[Catat Metadata Refund Idempoten]
```

## 4. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Admin Gudang
    participant UI as RmaManager
    participant QC as ProcessWarehouseQcAction
    participant Ex as CreateExchangeOrderAction
    participant DB as PostgreSQL 16

    Admin->>UI: Terima paket retur & klik Lulus QC
    UI->>QC: execute(rma, passed_restock: true)
    QC->>DB: DB::transaction + lockForUpdate varian lama -> restock + InventoryLog
    opt claim_type == size_exchange
        UI->>Ex: execute(rma)
        Ex->>DB: Kurangi stok varian pengganti + buat shipment pengganti
    end
    UI-->>Admin: Status RMA Completed + RmaStatusUpdatedEvent
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    ORDERS ||--o{ ORDER_RETURNS : "has_rma"
    ORDER_ITEMS ||--o{ ORDER_RETURNS : "claimed_item"
    PRODUCT_VARIANTS ||--o{ ORDER_RETURNS : "exchange_target"

    ORDER_RETURNS {
        bigint id PK
        string rma_number UK
        bigint order_id FK
        bigint order_item_id FK
        bigint target_variant_id FK
        string claim_type "size_exchange | defect_qc | refund"
        string status "pending_review | approved | rejected | item_received | completed"
        text reason
        timestamp created_at
    }
```
