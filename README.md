# AGRINET Admin + Vendor Dashboard Documentation

## Project Status

| Item | Status |
|------|--------|
| Pi4 Server fixes (SQL injection, auth, schema) | Done |
| Admin Dashboard (11 pages, full build) | Done |
| Vendor Dashboard (5 pages, full build) | Done |
| Frontend build (`npm run build`) | Passes — zero errors |
| Offline mode (hooks created, banner wired) | Partial — SW + IndexedDB queue pending |
| Alert notifications (Web Push, WhatsApp/SMS) | Planned — not yet implemented |

## Files in this folder

| File | Contents |
|------|----------|
| `DASHBOARD-SPEC.md` | **Complete dashboard specification** — all pages, components, API routes, offline mode, notifications. Start here. |
| `AGRINET-Admin-Vendor-Dashboard-Plan.md` | Full project plan including firmware migration + dashboard. |
| `IMPLEMENTATION-STATUS.md` | **What was built** — file-by-file summary of server fixes and frontend pages. |

## Architecture Summary

```
Sasyamithra Mobile App  ──┐
Admin Dashboard         ──┤──→  Pi4 Fastify API (port 3000)
Vendor Dashboard        ──┘         │
                                    ├── SQLite (farms, motors, valves, users, vendors, issues)
                                    ├── InfluxDB (time-series telemetry)
                                    └── Mosquitto (port 1883 + 9001 WS)
                                             ↑
                                    STM32 gateway via SIM7670C AT+CMQTT
```

**Zero Firebase. Pi4 is the single source of truth.**

## Three Dashboards

| Dashboard | Role | URL |
|-----------|------|-----|
| Sasyamithra App | Farmer | Mobile app (Capacitor) |
| Admin Dashboard | superadmin / agri_expert / technician | `http://pi4:5174/admin/` |
| Vendor Dashboard | pump vendor / irrigation contractor | `http://pi4:5174/vendor/` |

## Key Decisions

- **Pi4-only** — no cloud dependency, works offline in remote farms
- **MQTT WebSocket** (port 9001) for real-time telemetry in dashboard
- **Field Cluster** = 1 weather station (I2C master) + up to 10 valve controllers (I2C slaves at 0x21-0x2A)
- **Offline mode** — Service Worker + IndexedDB queue for commands when Pi4 unreachable
- **Alerts** — in-app notification bell + browser Web Push + WhatsApp/SMS for critical faults
- **Vendor accounts** — scoped to contracted farms via `vendor_farms` table (access_level: read/control/full)

## Server Fixes — All Complete

- [x] Create `adapters/influx.js` — InfluxDB telemetry query adapter
- [x] Add `district/mandal/village/farm/crop/acreage` columns to `init-db.js` users table
- [x] Fix SQL injection in `db.js` vendor query functions (parameterized queries)
- [x] Refactor admin/vendor routes to use Fastify `preHandler` auth hooks
- [x] Add missing indexes in `db/vendors.js`
- [x] Fix `getVendorSchedule` — `SELECT v.*` so `rowToValve()` has all columns
- [x] Fix `getAllIssues` — parameterized status filter

## Frontend Tech Stack

| Layer | Choice |
|-------|--------|
| Framework | React 18 + TypeScript + Vite |
| Styling | Tailwind CSS v3 (dark theme) |
| State | Zustand (persist to localStorage) |
| Server state | TanStack React Query v5 |
| Charts | Recharts |
| Real-time | MQTT.js over WebSocket (port 9001) |
| Routing | React Router v6 |

## Source Locations

| Component | Path |
|-----------|------|
| Pi4 Server (fixed) | `~/Downloads/ssmiot2files/sasyamithra-server-FIXED/` |
| Admin + Vendor Frontend | `~/Projects/agrinet-admin/` |
| This documentation | `~/Projects/admn-vend-dshbrd-doc/` |
