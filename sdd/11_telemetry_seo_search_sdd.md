# 11. VPS Telemetry, Search Fallback & Dynamic SEO

## 1. Ruang Lingkup & Komponen Aktual
- **Services**: `TelemetryHealthService`, `TelemetryAlertService`, `TypesenseSearchService`, `SeoMetaService`, `StorefrontCacheService`, `WebPMediaOptimizer`
- **Controllers & CLI**: `TelemetryHealthController` (`/api/v1/health`, `/api/v1/health/ping`), `SeoController` (`/robots.txt`, `/sitemap.xml`), `nusora:telemetry`, `typesense:index`
- **UI**: `Admin\Monitoring\SystemMonitoring`, `Storefront\InstantSearch` (`Ctrl+K`)

## 2. Use Case Diagram

```mermaid
flowchart LR
    Monitor["Admin / Uptime Probe"]
    Visitor["Pembeli & Search Bot"]
    Scheduler["Cron Scheduler"]

    subgraph PlatformModule ["Modul Telemetry, Search & SEO"]
        UC1["Pantau Kesehatan DB, Queue, Disk & OPcache"]
        UC2["Kirim Peringatan Threshold via Webhook/Email"]
        UC3["Pencarian Instan Typesense + Fallback Eloquent"]
        UC4["Generate Dynamic XML Sitemap & JSON-LD Schema"]
    end

    Monitor --> UC1
    Scheduler --> UC2
    Visitor --> UC3
    Visitor --> UC4
```

## 3. Activity Diagram (Resilient Search Fallback)

```mermaid
flowchart TD
    Start([Query Pencarian InstantSearch / Katalog]) --> TryTS[Kirim Query ke Typesense 26+]
    TryTS --> IsOk{Typesense Online & Sukses?}
    IsOk -- Ya --> ReturnTS[Kembalikan Hasil Sub-10ms]
    IsOk -- Tidak --> FallbackDB[Fallback Otomatis ke PostgreSQL Eloquent LIKE/JSONB]
    FallbackDB --> ReturnDB[Kembalikan Hasil Konsisten Tanpa Error]
```

## 4. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Probe as Uptime Monitor / Admin
    participant Ctrl as TelemetryHealthController
    participant Svc as TelemetryHealthService
    participant DB as PostgreSQL & Cache

    Probe->>Ctrl: GET /api/v1/health
    Ctrl->>Svc: collect()
    Svc->>DB: Cek latency SELECT 1, failed_jobs, scheduler_last_pulse
    DB-->>Svc: Metrik Kesehatan Aktual
    Svc-->>Ctrl: Payload Status (healthy / degraded)
    Ctrl-->>Probe: JSON 200 / 503
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    BUSINESS_SETTINGS ||--o{ PRODUCTS : "configures_seo"

    BUSINESS_SETTINGS {
        bigint id PK
        string business_name
        string business_archetype
        char currency_code
        json operational_settings
        json payment_settings
        json shipping_settings
    }
```
