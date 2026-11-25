# 00. System Architecture Overview & Global Design

## 1. Ikhtisar Arsitektur Sistem

**NusoraCommerce** dibangun menggunakan arsitektur **Single-Instance Universal Modular Monolith** di atas **Laravel 12** (PHP 8.2/8.3+), **Livewire 4**, **Tailwind CSS v4**, dan **PostgreSQL 16+**. Seluruh domain bisnis dipisahkan secara eksplisit di bawah `app/Domain/` dengan komunikasi melalui Domain Actions, Value Objects, dan Domain Events.

```mermaid
graph TB
    subgraph ClientLayer ["Client Layer (Browser)"]
        SF["Storefront Publik (Livewire 4 + Alpine.js)"]
        ADM["Admin Cockpit (Native Livewire 4, No Filament)"]
        PRINT["Print Engine (Faktur A4, Thermal A6, Packing Slip A4)"]
    end

    subgraph AppLayer ["Application & Security Layer"]
        MW["Middleware (EnsureAdminAccess, SecurityHeaders, RateLimiter)"]
        GUARD["Showcase Guard (demo_admin Read-Only + demo_mode + allow_registration)"]
        HOOKS["Thin Livewire Components & Thin Webhook Controllers"]
    end

    subgraph DomainLayer ["12 Core Domain Modules (app/Domain/)"]
        D_CAT["Catalog (JSONB + Variant Matrix)"]
        D_INV["Inventory (15m TTL Lock + Opname + Ledger)"]
        D_CART["Cart (Guest/Member + Auto-Merge)"]
        D_ORD["Order (State Machine + Locked Adjustments)"]
        D_PAY["Payment (Xendit/Midtrans/Stripe + Reconciler)"]
        D_SHIP["Shipping (JNE/J&T + Barcode + Tracking)"]
        D_PROMO["Pricing, Tax PPN 11% & Promotions"]
        D_RMA["Returns & 7-Day Size Exchange RMA"]
        D_REV["Verified Fit Reviews & Customer Portal"]
        D_SET["BusinessSettings, AuditLog & Telemetry"]
    end

    subgraph InfraLayer ["Persistence, Search & External Gateways"]
        PG[("PostgreSQL 16+ (JSONB + GIN Index)")]
        CACHE[("Redis 7+ / Database (Cache, Session, Queue)")]
        TS["Typesense 26+ (Sub-10ms Search + Eloquent Fallback)"]
        EXT["Xendit / Midtrans / Stripe & JNE / J&T Webhooks"]
    end

    SF --> MW
    ADM --> MW
    PRINT --> MW
    MW --> GUARD
    GUARD --> HOOKS
    HOOKS --> DomainLayer
    DomainLayer --> PG
    DomainLayer --> CACHE
    DomainLayer --> TS
    D_PAY --> EXT
    D_SHIP --> EXT
```

---

## 2. Alur Transaksi & Invarian Inti

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as Pembeli
    participant UI as CheckoutWizard (Livewire 4)
    participant Action as PlaceOrderAction
    participant Inv as ReserveInventoryAction
    participant DB as PostgreSQL 16
    participant Pay as ProcessPaymentWebhookAction

    Buyer->>UI: Isi alamat, pilih kurir & kupon, klik Bayar
    alt Mode Demo Aktif (demo_mode = true)
        UI-->>Buyer: Hentikan transaksi & tampilkan notifikasi Mode Demo
    else Mode Normal
        UI->>Action: execute(cart, customer, shipping, promo)
        rect rgb(240, 248, 255)
            Note over Action,DB: Transaksi Terisolasi (DB::transaction)
            Action->>DB: Buat Order (pending_payment, ULID) & OrderItems (snapshot)
            Action->>Inv: Kunci stok varian (lockForUpdate, TTL 15 menit)
            Inv->>DB: INSERT inventory_reservations (status: reserved)
            Action->>DB: INSERT order_adjustments (tax, shipping, promotion)
        end
        Action-->>UI: OrderPlacedEvent -> redirect /order/{orderNumber}
        Pay->>DB: Webhook validasi signature & idempotency_key
        Pay->>DB: TransitionOrderStatusAction(Confirmed) + lock adjustments
        Pay->>DB: ConvertReservationToSaleAction (kurangi stock_quantity + tulis InventoryLog)
    end
```
