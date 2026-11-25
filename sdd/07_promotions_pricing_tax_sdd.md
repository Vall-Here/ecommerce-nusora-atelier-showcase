# 07. Pricing Value Object, PPN 11% & Rule-Driven Promotions

## 1. Ruang Lingkup & Komponen Aktual
- **Value Object**: `App\Domain\Pricing\ValueObjects\Money` (integer cents, zero floating-point)
- **Services & Models**: `TaxCalculatorService`, `PromotionEvaluatorService`, `Promotion`, `PromotionRule`
- **Admin UI**: `Admin\Promotions\PromotionManager`, `Admin\Analytics\FinancialReports`

## 2. Use Case Diagram

```mermaid
flowchart LR
    Admin["Admin Promosi"]
    Buyer["Pembeli"]
    Checkout["Checkout Engine"]

    subgraph PricingPromoModule ["Modul Pricing, Tax & Promotions"]
        UC1["Kelola Kupon & Rule Belanja Minimum"]
        UC2["Evaluasi Kelayakan & Batas Pemakaian Kupon"]
        UC3["Hitung PPN 11% dari Dasar Pengenaan Pajak"]
        UC4["Format & Hitung Uang Presisi Integer Cents"]
    end

    Admin --> UC1
    Buyer --> UC2
    Checkout --> UC2
    Checkout --> UC3
    Checkout --> UC4
```

## 3. Activity Diagram

```mermaid
flowchart TD
    Start([Input Kode Kupon]) --> FindPromo{Kupon Aktif & Dalam Periode?}
    FindPromo -- Tidak --> Reject[Tolak Kupon]
    FindPromo -- Ya --> CheckQuota{Kuota Global & Per-User Aman?}
    CheckQuota -- Penuh --> Reject
    CheckQuota -- Aman --> CheckRules[Evaluasi PromotionRule min_order_amount]
    CheckRules -- Gagal --> Reject
    CheckRules -- Lolos --> CalcDiscount[Hitung Diskon + Max Cap]
    CalcDiscount --> CalcTax[Hitung PPN 11% atas subtotal - diskon]
```

## 4. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as Pembeli
    participant Wizard as CheckoutWizard
    participant Eval as PromotionEvaluatorService
    participant Tax as TaxCalculatorService
    participant DB as PostgreSQL 16

    Buyer->>Wizard: applyCoupon(code)
    Wizard->>Eval: evaluate(code, subtotal, user, email)
    Eval->>DB: SELECT promotions + promotion_rules
    Eval-->>Wizard: PromotionResult (eligible, discountAmountCents)
    Wizard->>Tax: calculateTax(max(0, subtotal - discount))
    Tax-->>Wizard: PPN 11% (integer cents)
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    PROMOTIONS ||--o{ PROMOTION_RULES : "enforces"

    PROMOTIONS {
        bigint id PK
        string code UK
        string name
        string type "percentage | fixed"
        bigint value
        bigint max_discount_amount
        integer usage_limit
        integer usage_count
        timestamp starts_at
        timestamp ends_at
        boolean is_active
    }

    PROMOTION_RULES {
        bigint id PK
        bigint promotion_id FK
        string type "min_order_amount | min_subtotal"
        json configuration
    }
```
