# 06. Shipping, Fulfillment Desk & Print Engine

## 1. Ruang Lingkup & Komponen Aktual
- **Models & Services**: `Shipment`, `ShippingRateService`, `JneCourierDriver`, `JtExpressCourierDriver`, `BarcodeGeneratorService`, `IndonesiaCity`
- **Actions & Controllers**: `CreateShipmentAction`, `AddTrackingCheckpointAction`, `CourierWebhookController`, `ShippingPrintController`
- **UI**: `Admin\Shipping\FulfillmentDesk`, `print-label.blade.php` (Thermal A6), `print-packing-slip.blade.php` (A4)

## 2. Use Case Diagram

```mermaid
flowchart LR
    Warehouse["Staf Fulfillment"]
    Courier["Webhook JNE / J&T"]
    Buyer["Pembeli"]

    subgraph ShippingModule ["Modul Shipping & Fulfillment"]
        UC1["Hitung Ongkir Regional & Berat Gramasi"]
        UC2["Terbitkan Resi & Buat Shipment"]
        UC3["Cetak Label Thermal A6 & Packing Slip A4"]
        UC4["Terima Checkpoint Transit Kurir Idempoten"]
        UC5["Sinkronisasi Delivered ke Garansi 7 Hari"]
    end

    Warehouse --> UC2
    Warehouse --> UC3
    Courier --> UC4
    UC4 --> UC5
    Buyer --> UC1
```

## 3. Activity Diagram

```mermaid
flowchart TD
    Start([Order Confirmed / Processing]) --> OpenDesk[Buka FulfillmentDesk]
    OpenDesk --> SelectOrders[Pilih Order & Kurir JNE / J&T]
    SelectOrders --> CreateShip[Eksekusi CreateShipmentAction]
    CreateShip --> GenBarcode[Generate Code 39 SVG Barcode]
    GenBarcode --> PrintDocs[Cetak Label Thermal A6 & Packing Slip A4]
    PrintDocs --> Transit[Kurir Kirim Webhook Checkpoint]
    Transit --> Dedup{Checkpoint Duplikat?}
    Dedup -- Ya --> SkipHook[Abaikan Retry Webhook]
    Dedup -- Tidak --> SaveCP[Simpan Checkpoint & Cek Delivered]
```

## 4. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Kurir as JNE / J&T Webhook
    participant Ctrl as CourierWebhookController
    participant Act as AddTrackingCheckpointAction
    participant DB as PostgreSQL 16

    Kurir->>Ctrl: POST /api/webhooks/courier/{carrier} (X-Courier-Token)
    Ctrl->>Ctrl: Validasi COURIER_WEBHOOK_TOKEN
    Ctrl->>Act: execute(shipment, status, description)
    Act->>DB: Deduplikasi checkpoint identik
    alt Status == delivered
        Act->>DB: Update delivered_at & order status Completed
        Act-->>Ctrl: Dispatch OrderDeliveredEvent (mulai window RMA 7 hari)
    end
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    ORDERS ||--o| SHIPMENTS : "fulfilled_by"
    SHIPMENTS ||--o{ SHIPMENT_CHECKPOINTS : "tracked_via"

    SHIPMENTS {
        bigint id PK
        bigint order_id FK
        string carrier "JNE | J&T"
        string service_type
        string tracking_number
        string status
        timestamp shipped_at
        timestamp delivered_at
    }

    SHIPMENT_CHECKPOINTS {
        bigint id PK
        bigint shipment_id FK
        string status
        string location
        text description
        timestamp recorded_at
    }
```
