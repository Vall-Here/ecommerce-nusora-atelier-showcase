# NusoraCommerce Software Design Document (SDD) Matrix

Panduan spesifikasi teknis dan desain arsitektur perangkat lunak (*Software Design Document*) untuk platform **NusoraCommerce** (Single-Instance Universal Modular Monolith).

Seluruh spesifikasi mencerminkan implementasi kode aktual:
- **Backend**: Laravel 12 (PHP 8.2/8.3+), 12 Domain Modules (`app/Domain/`), Thin Livewire / Fat Domain Action.
- **Frontend & Admin**: Livewire 4.4 (`wire:navigate`), Alpine.js, Tailwind CSS v4 (`@tailwindcss/vite`), 100% Native Livewire Admin Cockpit (tanpa Filament).
- **Database & Infra**: PostgreSQL 16+ (JSONB + GIN Index), Redis/Database Queue & Cache, Typesense 26+ (opsional dengan fallback Eloquent), Xendit/Midtrans/Stripe, JNE & J&T.

---

## Indeks Dokumen Desain Modul (12 Dokumen)

| No | Modul | Berkas SDD | Cakupan Diagram |
| :---: | :--- | :--- | :--- |
| **00** | **System Architecture Overview** | [00_system_architecture_overview.md](./00_system_architecture_overview.md) | Topologi Monolith, Global Data Pipeline, Invariant Enforcement |
| **01** | **Catalog, JSONB & Variant Matrix** | [01_catalog_variants_sdd.md](./01_catalog_variants_sdd.md) | Use Case, Activity, Sequence, ERD (`products`, `product_variants`, `product_images`, `categories`) |
| **02** | **Inventory, Reservations & Opname** | [02_inventory_reservation_sdd.md](./02_inventory_reservation_sdd.md) | Use Case, Activity, Sequence, ERD (`inventory_reservations`, `inventory_logs`, `lockForUpdate`) |
| **03** | **Cart & 3-Column Checkout** | [03_cart_checkout_sdd.md](./03_cart_checkout_sdd.md) | Use Case, Activity, Sequence, ERD (`carts`, `cart_items`, Guest Merge, Demo Mode) |
| **04** | **Orders, State Machine & Adjustments** | [04_orders_state_machine_sdd.md](./04_orders_state_machine_sdd.md) | Use Case, Activity, Sequence, ERD (`orders`, `order_items`, `order_adjustments`, A4 Invoice) |
| **05** | **Payments, Webhooks & Reconciliation** | [05_payments_reconciliation_sdd.md](./05_payments_reconciliation_sdd.md) | Use Case, Activity, Sequence, ERD (`payments`, Xendit/Midtrans/Stripe, Overpayment & Manual Refund) |
| **06** | **Shipping, Fulfillment & Print** | [06_shipping_fulfillment_sdd.md](./06_shipping_fulfillment_sdd.md) | Use Case, Activity, Sequence, ERD (`shipments`, JNE/J&T, Thermal A6 & Packing Slip A4) |
| **07** | **Pricing, PPN 11% & Promotions** | [07_promotions_pricing_tax_sdd.md](./07_promotions_pricing_tax_sdd.md) | Use Case, Activity, Sequence, ERD (`Money`, `promotions`, `promotion_rules`, `TaxCalculatorService`) |
| **08** | **After-Sales Returns & RMA** | [08_returns_rma_sdd.md](./08_returns_rma_sdd.md) | Use Case, Activity, Sequence, ERD (`order_returns`, 7-Day Size Exchange, Warehouse QC) |
| **09** | **Reviews & Customer Portal** | [09_reviews_customer_portal_sdd.md](./09_reviews_customer_portal_sdd.md) | Use Case, Activity, Sequence, ERD (`product_reviews`, `addresses`, Wishlist, `allow_registration`) |
| **10** | **Admin Cockpit, RBAC & Demo Guard** | [10_admin_rbac_security_sdd.md](./10_admin_rbac_security_sdd.md) | Use Case, Activity, Sequence, ERD (`users`, Spatie RBAC, `demo_admin` Read-Only, `audit_logs`) |
| **11** | **Telemetry, Search & Dynamic SEO** | [11_telemetry_seo_search_sdd.md](./11_telemetry_seo_search_sdd.md) | Use Case, Activity, Sequence, ERD (VPS Health Probes, Typesense Fallback, Sitemap & JSON-LD) |
