# AGRINET Admin + Vendor Dashboard — Implementation Status

> Last updated: 2026-03-31

---

## Pi4 Server Fixes (sasyamithra-server-FIXED/)

### adapters/influx.js — CREATED
- `queryTelemetry(deviceId, range)` — queries InfluxDB sensor_data measurement
- Supported ranges: 1h, 6h, 24h, 7d, 30d
- Graceful fallback: returns `[]` if INFLUX_TOKEN not set
- Uses `@influxdata/influxdb-client` (already in package.json)

### adapters/db.js — SQL INJECTION FIXED
All vendor functions now use parameterized `?` placeholders:
- `getVendorOverview()` — farm IDs and user IDs via `...farmIds` spread
- `getVendorAlerts()` — user IDs + ack filter parameterized
- `getVendorEnergy()` — farm IDs parameterized
- `getVendorSchedule()` — farm IDs parameterized, changed to `SELECT v.*` for full column mapping
- `getAllIssues()` — status filter via `?` parameter

### routes/admin.js — REFACTORED (preHandler auth)
Replaced manual `if (!requireAdmin(request, reply)) return` with Fastify preHandler hooks:
- `adminAuth` — verifies JWT, checks role in [superadmin, agri_expert, technician]
- `superAdminAuth` — extends adminAuth, requires superadmin role
- `nonTechAuth` — extends adminAuth, blocks technician role (for /users, /users/:id)

All 20+ routes now use `{ preHandler: adminAuth }` or role-specific hooks.

### routes/vendor.js — REFACTORED (preHandler auth)
- `vendorAuth` — verifies JWT, checks role === 'vendor'
- All 12 routes now use `{ preHandler: vendorAuth }`

### db/vendors.js — UPDATED
- Added ALTER TABLE migration for existing DBs (district, mandal, village, farm, crop, acreage)
- Added missing indexes: `idx_vendor_farms_farm`, `idx_issues_assigned`, `idx_reviews_expert`

### init-db.js — UPDATED
- Added 6 columns to users table: district, mandal, village, farm, crop, acreage

---

## Frontend (~/Projects/agrinet-admin/)

**Build status: PASSES** — 969 modules, zero errors

### Project Structure (28 files)

```
src/
├── main.tsx                    — QueryClient + React entry
├── App.tsx                     — Router: /admin/*, /vendor/*, guards
├── api/
│   ├── client.ts               — Axios + JWT interceptor (auto admin/vendor token)
│   ├── adminApi.ts             — 20 admin API callers
│   └── vendorApi.ts            — 12 vendor API callers
├── hooks/
│   ├── useMqttFeed.ts          — MQTT.js WebSocket, reconnect, message buffer
│   └── useOnlineStatus.ts      — /api/health polling + navigator.onLine
├── store/
│   └── adminStore.ts           — Zustand: admin auth, vendor auth, alertBadgeCount
├── components/layout/
│   ├── AdminLayout.tsx          — Sidebar nav (role-filtered) + alert badge + offline banner
│   └── VendorLayout.tsx         — Tablet-optimized sidebar + offline banner
├── pages/
│   ├── LoginPage.tsx            — Admin/Vendor toggle, JWT login
│   ├── admin/
│   │   ├── OverviewPage.tsx     — 4 stat cards + MQTT live feed + Pi4 infra strip
│   │   ├── FarmsPage.tsx        — Searchable/filterable farm table, pagination
│   │   ├── FarmDetailPage.tsx   — Farm info + motor cards + field cluster cards
│   │   ├── ClusterDetailPage.tsx — 12-cell sensor grid + valve bus (I2C children)
│   │   ├── DevicesPage.tsx      — Tabbed device registry (all/motors/valves/stations)
│   │   ├── AlertsPage.tsx       — Filtered alert table + ack actions
│   │   ├── IssuesPage.tsx       — Kanban (Open/InProgress/Resolved) + table view toggle
│   │   ├── ExpertPage.tsx       — Farm selector + review form + review history
│   │   ├── UsersPage.tsx        — Farmer management table
│   │   ├── VendorsPage.tsx      — Vendor management (superadmin only)
│   │   └── InfraPage.tsx        — Pi4 CPU/RAM bars + Mosquitto/SQLite/InfluxDB cards
│   └── vendor/
│       ├── ControlCenterPage.tsx — Motor grid + START/STOP toggle + live MQTT feed
│       ├── VendorFarmsPage.tsx   — Farm card grid with pump/valve counts
│       ├── EnergyPage.tsx        — Summary cards + Recharts bar chart + motor table
│       ├── SchedulePage.tsx      — Valve schedule timeline list
│       └── VendorAlertsPage.tsx  — Scoped alert table + ack
```

### Admin Pages (11)

| Page | Route | Features |
|------|-------|----------|
| Overview | `/admin` | Stat cards (farms, devices, alerts, gateways), MQTT live feed (last 50 messages, color-coded by topic), Pi4 infra strip (CPU/RAM bars, MQTT status, DB size, uptime) |
| Farms | `/admin/farms` | Searchable + filterable table, click → farm detail, pagination |
| Farm Detail | `/admin/farms/:id` | Farm header, motor cards (3-phase V/A, power, PF, kWh), field cluster cards (click → cluster detail) |
| Cluster Detail | `/admin/clusters/:id` | 12-cell sensor grid (airT, humidity, soilM L1/L2/L3, leaf wet, wind, rain, battery, light, signal), valve bus with I2C addresses |
| Devices | `/admin/devices` | Tabbed (All/Motors/Valves/Stations), paginated table |
| Alerts | `/admin/alerts` | Filter by ack/severity, acknowledge button, auto-refresh 15s |
| Issues | `/admin/issues` | Kanban + Table toggle, Start/Resolve actions, severity badges |
| Expert | `/admin/expert` | Farm selector, review note form (recommendation dropdown + severity), review history panel |
| Users | `/admin/users` | Search + district filter, farmer table (superadmin + agri_expert only) |
| Vendors | `/admin/vendors` | Vendor table (superadmin only) |
| Infra | `/admin/infra` | CPU/RAM progress bars, Mosquitto status, SQLite size, InfluxDB config, 10s polling |

### Vendor Pages (5)

| Page | Route | Features |
|------|-------|----------|
| Control Center | `/vendor` | Command strip (running/stopped/alerts/kWh), motor grid with START/STOP toggle, 3-phase readings, MQTT live feed |
| Farms | `/vendor/farms` | Card grid (pump counts, valve counts, alert badges, access level) |
| Energy | `/vendor/energy` | Summary cards (kWh, hours, cost @₹7/unit), Recharts bar chart by motor, detailed table |
| Schedule | `/vendor/schedule` | Timeline list of scheduled/auto valves, start/end times, recurring indicator |
| Alerts | `/vendor/alerts` | Filtered alert table, acknowledge action |

### Auth Flow

1. Login page: Admin/Vendor toggle → POST `/admin/auth/login` or `/vendor/auth/login`
2. JWT stored in Zustand (persisted to localStorage)
3. Axios interceptor auto-selects token based on URL prefix (`/vendor` → vendorToken, else → adminToken)
4. Route guards: `AdminGuard` checks adminToken, `VendorGuard` checks vendorToken, `RoleGuard` checks adminUser.role
5. On 401: clear both auth states, redirect to `/login`

### Role-Based Access

| Page | superadmin | agri_expert | technician |
|------|-----------|-------------|------------|
| Overview, Farms, FarmDetail, Clusters, Devices, Alerts, Issues, Expert, Infra | Yes | Yes | Yes |
| Users | Yes | Yes (read) | No |
| Vendors | Yes | No | No |

---

## Still Pending

| Item | Notes |
|------|-------|
| Service Worker (Workbox) | vite-plugin-pwa not yet configured |
| IndexedDB offline queue | Hook skeleton exists (`useOnlineStatus`), queue not implemented |
| Web Push notifications | VAPID keys + `POST /admin/push/subscribe` route needed |
| WhatsApp/SMS alerts | Twilio/D7 integration (config-driven, optional) |
| Audit log page | `/admin/audit` — route + page not yet built |
| OTA firmware page | `/admin/devices/ota` — sub-tab not yet built |
| Leaflet farm map | On Overview page — map component not yet created |
| Drag-drop Kanban | @hello-pangea/dnd installed but not wired into IssuesPage |
| CSV export | Users + Energy pages — export button not yet wired |
