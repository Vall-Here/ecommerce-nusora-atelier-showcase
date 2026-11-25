# NusoraCommerce — Architecture & Technical Documentation

Dokumentasi arsitektur perangkat lunak dan spesifikasi teknis (*Software Design Document*) untuk sistem e-commerce **Single-Instance Modular Monolith** berbasis **Laravel 12** (PHP 8.2/8.3+), **Livewire 4**, **Tailwind CSS v4**, dan **PostgreSQL 16+**.

---

## 1. Batas Arsitektur

Aplikasi ini dibangun untuk satu instansi bisnis per deployment (bukan SaaS multi-tenant):
- **Modular Monolith (`app/Domain/`)**: 12 modul domain terpisah (`Catalog`, `Inventory`, `Cart`, `Order`, `Pricing`, `Promotion`, `Payment`, `Shipping`, `Tax`, `Review`, `Customer`, `Settings`).
- **Polymorphic Order Adjustments**: Pajak PPN 11%, diskon promosi, dan ongkos kirim disimpan di `order_adjustments` dan dikunci (`is_locked = true`) saat pesanan terkonfirmasi.
- **Native Livewire 4 Backoffice**: Admin Cockpit dibangun dengan Livewire 4 dan Tailwind CSS v4 tanpa Filament.

---

## 2. Invarian Sistem

1. **Pessimistic Inventory Locking**: Reservasi stok checkout (TTL 15 menit) dan mutasi stok dieksekusi di dalam `DB::transaction()` menggunakan `lockForUpdate()`.
2. **Integer-Cents Money**: Nominal uang disimpan sebagai `BIGINT` satuan terkecil dan diformat melalui Value Object `Money`.
3. **Immutable Order Adjustments**: `OrderAdjustment::booted()` menolak perubahan atau penghapusan baris setelah `is_locked = true`.
4. **Idempotent Payment Webhooks**: Webhook Xendit, Midtrans, dan Stripe memvalidasi signature kriptografis serta `idempotency_key` sebelum memperbarui status pesanan.
5. **Thin Livewire, Fat Action**: Komponen Livewire mengelola state antarmuka dan validasi; logika transaksi berada di `app/Domain/**/Actions`.
6. **Showcase & Shared Hosting Controls**: Pengaturan Admin menyediakan toggle `allow_registration`, `demo_mode` (membatasi transaksi hingga tahap Checkout), serta akun `demo_admin` read-only dengan guard UI, Livewire `call` hook, dan Eloquent write-blocker.
7. **Print & Excel CSV**: Faktur penjualan A4 (`@media print`), label thermal A6, packing slip A4, dan ekspor CSV via `CsvExport` (UTF-8 BOM dan sanitasi CWE-1236).

---

## 3. Direktori Dokumentasi

- [Portal Dokumentasi & Diagram Interaktif](./index.html)
- [Diagram Arsitektur Global (Archify)](./architecture/nusora-architecture.html)
- [Diagram Arsitektur Backend (Archify)](./backend/backend-architecture.html)
- [Diagram Arsitektur Frontend & Admin UI (Archify)](./frontend/frontend-architecture.html)
- [Matriks Software Design Document (12 Dokumen SDD)](./sdd/README.md)
- [Panduan Deployment Shared Hosting](../DEPLOYMENT_SHARED_HOSTING.md)
