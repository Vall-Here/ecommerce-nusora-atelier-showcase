# 05. Multi-Gateway Payments, Webhooks & Reconciliation

## 1. Ruang Lingkup & Komponen Aktual
- **Gateways**: `XenditPaymentGateway`, `MidtransPaymentGateway`, `StripePaymentGateway`
- **Controllers & Actions**: `PaymentWebhookController`, `ProcessPaymentWebhookAction`, `PaymentReconcilerService`, `CircuitBreaker`
- **Admin UI**: `OrderList` (Tab `Perlu Rekonsiliasi`), `OrderDetail` (Panel Rekonsiliasi Pembayaran + `AuditLog`)

## 2. Use Case Diagram

```mermaid
flowchart LR
    Gateway["Xendit / Midtrans / Stripe"]
    Cron["Reconciler Scheduler"]
    Finance["Admin Keuangan"]

    subgraph PaymentModule ["Modul Payment & Rekonsiliasi"]
        UC1["Terima & Verifikasi Signature Webhook"]
        UC2["Proses Pembayaran Idempoten (idempotency_key)"]
        UC3["Deteksi Overpayment & Late Payment"]
        UC4["Rekonsiliasi Otomatis Order Gantung (5m)"]
        UC5["Selesaikan Antrean Refund Manual & Catat AuditLog"]
    end

    Gateway --> UC1
    UC1 --> UC2
    UC2 --> UC3
    Cron --> UC4
    Finance --> UC5
```

## 3. Activity Diagram

```mermaid
flowchart TD
    Start([Webhook Masuk]) --> VerifySig{Signature / Token Valid?}
    VerifySig -- Tidak --> Return401[Tolak HTTP 401 Unauthorized]
    VerifySig -- Ya --> ParseDTO[Parse ke PaymentWebhookResult]
    ParseDTO --> CheckIdem{idempotency_key Sudah Diproses?}
    CheckIdem -- Ya --> SkipReturn[Return 200 OK Idempoten]
    CheckIdem -- Tidak --> CheckAmount{Nominal > Tagihan Order?}
    CheckAmount -- Ya --> FlagOverpay[Set metadata overpayment_detected = true]
    CheckAmount -- Tidak --> CheckLate{Order Cancelled & Stok Habis?}
    FlagOverpay --> ConfirmOrder[Konfirmasi Order]
    CheckLate -- Ya --> FlagManual[Set metadata requires_manual_refund = true]
    CheckLate -- Tidak --> ConfirmOrder
```

## 4. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant PGW as Xendit / Midtrans
    participant Ctrl as PaymentWebhookController
    participant Act as ProcessPaymentWebhookAction
    participant Admin as OrderDetail (Admin)
    participant DB as PostgreSQL 16

    PGW->>Ctrl: POST /api/webhooks/payment/{gateway}
    Ctrl->>Ctrl: verifyWebhookSignature() & parseWebhook()
    Ctrl->>Act: execute(PaymentWebhookResult)
    Act->>DB: DB::transaction + Lock Order + updateOrCreate Payment
    alt Overpayment / Late Payment
        Act->>DB: Simpan flag overpayment_detected / requires_manual_refund di metadata
    end
    Admin->>DB: resolveReconciliation() + AuditLog::record(order.reconciliation_resolved)
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    ORDERS ||--o{ PAYMENTS : "paid_via"
    ORDERS ||--o{ AUDIT_LOGS : "reconciled_in"

    PAYMENTS {
        bigint id PK
        bigint order_id FK
        string gateway "xendit | midtrans | stripe"
        string transaction_id
        string idempotency_key UK
        string status "pending | paid | failed | expired | refunded"
        bigint amount
        json gateway_response
        timestamp paid_at
    }
```
