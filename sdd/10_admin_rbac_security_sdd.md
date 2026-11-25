# 10. Admin Cockpit, RBAC, Demo Read-Only Guard & Audit Trail

## 1. Ruang Lingkup & Komponen Aktual
- **Auth & Middleware**: `EnsureAdminAccess`, `AdminLogin`, `SecurityHeadersMiddleware`, `AppServiceProvider` (`Gate::before`, `Livewire::listen('call')`, Eloquent write-blocker)
- **Models & Support**: `User`, `AuditLog`, `BusinessSetting`, `CsvExport` (UTF-8 BOM + CWE-1236)
- **Admin UI**: 17 Native Livewire 4 Components (tanpa Filament)

## 2. Use Case Diagram

```mermaid
flowchart LR
    SuperAdmin["SuperAdmin / Admin"]
    DemoViewer["Demo Admin (demo_admin)"]

    subgraph AdminSecurityModule ["Modul Admin RBAC & Showcase Guard"]
        UC1["Kelola Peran & Akses Staf (Spatie RBAC)"]
        UC2["Toggle Pendaftaran User & Mode Demo Showcase"]
        UC3["Akses Read-Only Seluruh Dashboard (demo@nusora.test)"]
        UC4["Ekspor Laporan CSV Kompatibel Excel (UTF-8 BOM)"]
        UC5["Catat Audit Trail Mutasi Sensitif"]
    end

    SuperAdmin --> UC1
    SuperAdmin --> UC2
    SuperAdmin --> UC4
    SuperAdmin --> UC5
    DemoViewer --> UC3
    DemoViewer --> UC4
```

## 3. Activity Diagram (3-Layer Demo Read-Only Guard)

```mermaid
flowchart TD
    Start([Request ke Admin Cockpit]) --> CheckAuth{Authenticated & Active?}
    CheckAuth -- Tidak --> RedirectLogin[Redirect /admin/login]
    CheckAuth -- Ya --> CheckDemo{Role == demo_admin?}
    CheckDemo -- Tidak --> AllowFull[Izinkan Akses Penuh Sesuai Permission]
    CheckDemo -- Ya --> CheckHTTP{HTTP GET / HEAD?}
    CheckHTTP -- Tidak --> Abort403[HTTP 403 Read-Only]
    CheckHTTP -- Ya --> CheckLW{Livewire Method di Whitelist Read-Only?}
    CheckLW -- Tidak --> BlockEarly[Return Early + Toast Error + Eloquent Write-Blocker]
    CheckLW -- Ya --> RenderView[Tampilkan Data & Banner Read-Only]
```

## 4. Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Demo as Demo Viewer (demo_admin)
    participant MW as EnsureAdminAccess
    participant App as AppServiceProvider (Livewire Hook)
    participant DB as Eloquent Model Listener

    Demo->>MW: GET /admin/orders
    MW-->>Demo: 200 OK + Banner AKUN DEMO READ-ONLY
    Demo->>App: Klik tombol mutasi (e.g. saveSettings / transitionTo)
    App->>App: Cek whitelist readOnlyMethods -> Tidak Cocok
    App-->>Demo: returnEarly() + Toast Error (DB tidak tersentuh)
    Note over DB: Lapis ke-3: eloquent.creating/updating/deleting return false
```

## 5. Entity Relationship Diagram

```mermaid
erDiagram
    USERS ||--o{ AUDIT_LOGS : "performs"
    USERS ||--o{ MODEL_HAS_ROLES : "assigned"
    ROLES ||--o{ MODEL_HAS_ROLES : "granted"

    USERS {
        bigint id PK
        string name
        string email UK
        string phone
        boolean is_active
        timestamp last_login_at
        string last_login_ip
        json metadata
    }

    AUDIT_LOGS {
        bigint id PK
        bigint user_id FK
        string event
        string auditable_type
        bigint auditable_id
        json old_values
        json new_values
        timestamp created_at
    }
```
