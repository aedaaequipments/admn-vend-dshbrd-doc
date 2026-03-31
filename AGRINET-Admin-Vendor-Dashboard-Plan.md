# AGRINET Platform — Admin Dashboard + Firebase→MQTT Migration Plan

## Context

The TPMC firmware currently uses **Firebase REST API via HTTP AT commands**. We are migrating to **MQTT pub/sub via AT+CMQTT** targeting a **Raspberry Pi 4 Mosquitto broker**. Zero Firebase dependency.

**Critical discovery**: Working MQTT firmware already exists at:
- `~/Downloads/gateway-mqtt-firmware/src/` — SIM7670C MQTT driver + cloud sync
- `~/Downloads/ssmiot2files/sasyamithra-server-FIXED/` — Pi4 server (Fastify + MQTT + InfluxDB)
- `~/Downloads/gateway-mqtt-firmware/docs/` — CONFIG_CHANGES.h + FLASH_CONFIG_CHANGES.c diffs

## New Architecture

```
STM32F103C8 → AT+CMQTT → Pi4 Mosquitto → App (MQTT WebSocket/REST)
                              ↓
                         Pi4 runs:
                         • Mosquitto broker (port 1883)
                         • InfluxDB (time-series)
                         • SQLite (config/users)
                         • Fastify API (REST for app)
                         • MQTT handler (data ingest)
```

## MQTT Topic Tree (replaces Firebase paths)

```
data/{uid}/{fid}/telemetry   — Motor + weather + valve data (QoS 1)
cmd/{uid}/{fid}/set          — Commands from app (QoS 1, PUSH)
alert/{uid}/{fid}/warn       — Critical alerts (QoS 2)
status/{uid}/{fid}/heartbeat — Connectivity + signal (QoS 0)
status/{uid}/{fid}/ack       — Command acknowledgments
ota/{uid}/{fid}/#            — OTA firmware updates
```

---

## Execution Plan — 10 Steps

### Step 1: Replace `credentials.h.template`
**Action**: REPLACE entire file
**Source**: `~/Downloads/gateway-mqtt-firmware/src/credentials.h.template`
**Target**: `TPMC-cld-ai-v3/src/credentials.h.template`
**Why**: Foundation — defines MQTT_DEFAULT_BROKER_IP, PORT, USERNAME, PASSWORD, CLIENT_ID, USER_ID, FARM_ID

### Step 2: Modify `config.h`
**Action**: SURGICAL EDIT (4 changes)
**Reference**: `~/Downloads/gateway-mqtt-firmware/docs/CONFIG_CHANGES.h`
- Line 19: FW_VERSION → "2.0.0-MQTT"
- Lines 165-168: CLOUD_PUSH/POLL → MQTT_PUBLISH_PERIOD, remove CLOUD_POLL_PERIOD_MS
- Lines 198-201: Remove FIREBASE_UID/TOKEN/URL_MAX_LEN, add USER_ID/FARM_ID/MQTT_BROKER_IP/USERNAME/PASSWORD/CLIENT_ID_MAX_LEN
- Lines 327-355: FlashConfig_t struct — replace firebaseUid/Token/Url with userId/farmId/mqttBrokerIp/mqttBrokerPort/mqttUsername/mqttPassword/mqttClientId

### Step 3: Replace `gsm_driver.h` + `gsm_driver.c`
**Action**: REPLACE both files
**Source**: `~/Downloads/gateway-mqtt-firmware/src/`
**Changes**: HTTP state machine → MQTT. GsmState_t: HTTP_ACTIVE → MQTT_CONNECTED. Removes HttpMethod_t, GSM_HttpRequest. Adds GSM_MqttPublish, GSM_MqttSubscribe, GSM_SetRxCallback, GSM_ProcessMqttRx. 335→474 lines.

### Step 4: Replace `offline_queue.h` + `offline_queue.c`
**Action**: REPLACE both files
**Source**: `~/Downloads/gateway-mqtt-firmware/src/`
**Changes**: OfflinePacket_t: removes HttpMethod_t+path, adds char topic[96]+uint8_t qos. Enqueue signature: `(topic, json)` instead of `(method, path, json)`. Flush uses GSM_MqttPublish.

### Step 5: Replace `cloud_sync.h` + `cloud_sync.c`
**Action**: REPLACE both files
**Source**: `~/Downloads/gateway-mqtt-firmware/src/`
**Changes**: Firebase REST → MQTT pub/sub. Removes polling functions (PollMotorCommands, PollValveCommand). Adds MqttRxHandler callback + CloudSync_HandleCommand. Commands arrive via MQTT subscription (instant push, no polling).

### Step 6: Modify `flash_config.c`
**Action**: SURGICAL EDIT (3 sections)
**Reference**: `~/Downloads/gateway-mqtt-firmware/docs/FLASH_CONFIG_CHANGES.c`
- ApplyDefaults: Firebase defaults → MQTT defaults (userId, farmId, broker IP/port/user/pass/clientId)
- Remove SetFirebaseUid/Token/Url functions, add 7 MQTT setters
- ProcessCommand: Update UART provisioning (SET_FARM, SET_BROKER, SET_MQTTPORT, SET_MQTTUSER, SET_MQTTPASS, SET_CLIENTID)

### Step 7: Modify `flash_config.h`
**Action**: SURGICAL EDIT
- Remove 3 Firebase function declarations
- Add 7 MQTT function declarations
- Update docstring

### Step 8: Comment-only updates (4 files)
**Action**: COMMENT CHANGES ONLY (zero logic changes)
- `main.c`: Lines 6, 12, 269 — "Firebase" → "MQTT/Pi4"
- `main.c`: Line 430 — provisioning message string update
- `automation.c`: Line 6 — "Firebase" → "MQTT"
- `automation.h`: Lines 5, 59 — "Firebase" → "MQTT"
- `display.h`: Line 10 — "Firebase status" → "MQTT broker status"

### Step 9: Add LoRaManager_SendValveCommandByIndex stub
**Action**: ADD ~10 lines to `lora_manager.c`
**Why**: New cloud_sync.c HandleCommand references this function for valve-by-index routing. Without it, linker fails.

### Step 10: Rewrite `docs/MASTER_EXECUTION_PLAN.md`
**Action**: COMPLETE REWRITE
- Remove all 80+ Firebase references
- Document Pi4 MQTT architecture
- MQTT topic tree
- Updated memory budget (saves ~6KB flash, ~640B RAM)
- Updated provisioning commands
- Updated verification plan

---

## Files Unchanged (14 files)

motor_control.c/h, power_monitor.c/h, lora_radio.c/h, lora_manager.c/h (except stub), agrinet_protocol.h, node_registry.c/h, display.c, watchdog.c/h, platformio.ini

---

## Commit Strategy

Single atomic commit after all 10 steps:
```
Migrate firmware from Firebase HTTP to MQTT/Pi4 Mosquitto broker

- gsm_driver: AT+HTTP → AT+CMQTT (connect/pub/sub/receive)
- cloud_sync: Firebase REST → MQTT topics, push-based commands
- offline_queue: HTTP buffer → MQTT packet buffer
- config.h: Firebase fields → MQTT broker fields
- flash_config: Firebase setters → MQTT setters + new UART commands
- Zero Firebase dependency, Pi4 is central data hub

Architecture: STM32 → AT+CMQTT → Pi4 Mosquitto → App
Topics: data/{uid}/{fid}/telemetry, cmd/{uid}/{fid}/set
```

---

## Verification

| Check | Command/Action | Expected |
|-------|---------------|----------|
| Compile | `pio run` | Zero errors |
| No Firebase in source | `grep -r "firebase\|Firebase" src/` | Zero matches |
| MQTT present | `grep -r "MQTT\|CMQTT" src/` | Matches in gsm_driver, cloud_sync, config |
| FlashConfig fits 1KB | sizeof(FlashConfig_t) | ~278 bytes < 1024 |
| No removed API calls | grep for GSM_HttpRequest, FlashConfig_SetFirebase* | Zero matches |

## Key Source Files

| Purpose | Existing MQTT Source |
|---------|---------------------|
| MQTT GSM driver | `~/Downloads/gateway-mqtt-firmware/src/gsm_driver.c` (474 lines) |
| MQTT cloud sync | `~/Downloads/gateway-mqtt-firmware/src/cloud_sync.c` (448 lines) |
| Config migration diff | `~/Downloads/gateway-mqtt-firmware/docs/CONFIG_CHANGES.h` (101 lines) |
| Flash config diff | `~/Downloads/gateway-mqtt-firmware/docs/FLASH_CONFIG_CHANGES.c` (264 lines) |
| Pi4 server | `~/Downloads/ssmiot2files/sasyamithra-server-FIXED/` (5 files) |

---

---

# AGRINET Admin + Vendor Dashboards — DETAILED PLAN (v2)

> Supersedes all earlier admin/vendor sections. Incorporates code-review fixes, offline mode, and alert notification system.

---

## Platform Overview — Three Interfaces

| Interface | Who | Scope | Device |
|-----------|-----|-------|--------|
| **Sasyamithra App** | Individual farmer | Own farm | Mobile phone |
| **Admin Dashboard** | Superadmin, agri-expert, technician | All farms cross-tenant | Desktop browser |
| **Vendor Dashboard** | Pump vendor / irrigation contractor | Contracted farms only | Tablet / desktop |

All three hit the same **Pi4 Fastify REST API** (port 3000). Pi4 is the single source of truth.

---

## Part 1 — Pi4 Server Fixes (Pre-Requisites)

These bugs from code review must be fixed BEFORE writing any more frontend code.

### Fix 1: Create `adapters/influx.js` (BLOCKING — server crashes without it)

`admin.js` and `vendor.js` both import `queryTelemetry` from this missing file.
The InfluxDB client package (`@influxdata/influxdb-client`) is already in `package.json`.

**File:** `sasyamithra-server-FIXED/adapters/influx.js`
```
export async function queryTelemetry(deviceId, range)
  → query InfluxDB `sensor_data` measurement filtered by device_id tag
  → supported range strings: '1h', '6h', '24h', '7d', '30d'
  → return array of { time, ...fields }
  → if INFLUX_TOKEN empty, return [] with console.warn
```

### Fix 2: Add `district` column to users table

`getAllUsers()` in db.js queries `u.district` but `init-db.js` creates users without this column → SQL error.

**File:** `sasyamithra-server-FIXED/init-db.js`
Add: `district TEXT, mandal TEXT, village TEXT, farm TEXT, crop TEXT, acreage TEXT` to users table definition (these are profile fields, already stored but column missing from schema).

Also add `ALTER TABLE IF NOT EXISTS` migration in `db/vendors.js` to add missing columns to existing DBs.

### Fix 3: Parameterize vendor SQL queries (SQL injection)

In `db.js`, functions `getVendorOverview`, `getVendorAlerts`, `getVendorEnergy`, `getVendorSchedule` build SQL with string-interpolated farm IDs:
```js
const farmIds = farms.map(f => `'${f.id}'`).join(',');  // ← unsafe
```
**Fix:** Use SQLite `?` params via `db.prepare(...).all(farmIds)` with a prepared `IN (...)` query using a JSON list, or use `farms.map(f => f.id)` passed as repeated `?` params.

### Fix 4: Add auth preHandler to admin/vendor routes

Existing routes use Fastify's `preHandler` hook for authentication (cleaner, prevents accidental bypass). The new admin/vendor routes use manual `if (!requireAdmin(...)) return` inside each handler.

**Refactor:** Convert to:
```js
fastify.get('/admin/farms', { preHandler: requireAdminHook }, async (req, reply) => { ... })
```
where `requireAdminHook` is a preHandler function that attaches `request.admin` or calls `reply.code(401).send(...)`.

### Fix 5: Add `db.js` missing indexes

Add to `db/vendors.js`:
```sql
CREATE INDEX IF NOT EXISTS idx_vendor_farms_farm    ON vendor_farms(farm_id);
CREATE INDEX IF NOT EXISTS idx_issues_assigned      ON issues(assigned_to);
CREATE INDEX IF NOT EXISTS idx_reviews_expert       ON expert_reviews(expert_id);
```

---

## Part 2 — Admin Dashboard (Web App)

### Tech Stack (final)

| Layer | Choice | Reason |
|-------|--------|--------|
| Framework | React 18 + TypeScript + Vite | Matches mobile app stack |
| Styling | Tailwind CSS v3 | No design system overhead, fast dark theme |
| State | Zustand (persist to localStorage) | Auth tokens + UI prefs |
| Server state | TanStack React Query v5 | Auto-refresh, polling, cache |
| Charts | Recharts | Already in mobile app |
| Maps | Leaflet + react-leaflet | Farm GPS pins |
| Real-time | MQTT.js over WebSocket (port 9001) | Push from Pi4 broker |
| Offline | Service Worker (Workbox via vite-plugin-pwa) + IndexedDB | See Part 4 |

**Output directory:** `~/Projects/agrinet-admin/`

---

### Roles & Exact Page Access

| Page | superadmin | agri_expert | technician |
|------|-----------|-------------|------------|
| Overview `/` | ✅ | ✅ | ✅ |
| Farms list `/farms` | ✅ | ✅ | ✅ |
| Farm detail `/farms/:id` | ✅ | ✅ | ✅ |
| Cluster detail `/clusters/:id` | ✅ | ✅ | ✅ |
| Devices `/devices` | ✅ | ❌ | ✅ |
| OTA Firmware `/devices/ota` | ✅ | ❌ | ✅ |
| Alerts `/alerts` | ✅ | ✅ | ✅ |
| Issue Tracker `/issues` | ✅ | ✅ | ✅ (only assigned) |
| Expert Review `/expert` | ✅ | ✅ | ❌ |
| Users `/users` | ✅ | ✅ (read-only) | ❌ |
| Vendor Mgmt `/vendors` | ✅ | ❌ | ❌ |
| Infrastructure `/infra` | ✅ | ❌ | ✅ |
| Audit Log `/audit` | ✅ | ❌ | ❌ |

---

### Pi4 Admin API Routes (complete)

**Prefix:** `/admin/*`  **Auth:** Bearer JWT, role in `[superadmin, agri_expert, technician]`

```
POST   /admin/auth/login                  — role-gated login (8h token)
GET    /admin/overview                    — stat cards data
GET    /admin/farms?district=&search=&page= — paginated farm list
GET    /admin/farms/:farmId               — farm detail + devices + alerts
GET    /admin/farms/:farmId/clusters      — field clusters (station + valve children)
GET    /admin/clusters/:stationId         — single cluster detail
GET    /admin/devices?type=&status=&page= — all devices cross-farm
GET    /admin/alerts?ack=&sev=&type=&limit= — cross-farm alerts
PATCH  /admin/alerts/:id/ack              — acknowledge
GET    /admin/users?search=&district=&page= — all farmers (not superadmin/expert/etc.)
GET    /admin/users/:id                   — user detail + farms + devices
POST   /admin/review                      — post expert review note
GET    /admin/review/:farmId              — reviews for a farm
GET    /admin/issues?status=              — issue list (priority sorted)
POST   /admin/issues                      — create issue
POST   /admin/issues/:id/assign           — assign to technician
PATCH  /admin/issues/:id/progress         — move to in_progress
PATCH  /admin/issues/:id/resolve          — resolve issue
GET    /admin/telemetry/:deviceId?range=  — InfluxDB history
GET    /admin/infra                       — Pi4 CPU/RAM/MQTT/disk stats
GET    /admin/audit?limit=                — audit log (superadmin only)
GET    /admin/vendors                     — list all vendor accounts
POST   /admin/vendors                     — create vendor account
POST   /admin/vendors/:id/farms           — assign farm to vendor
DELETE /admin/vendors/:id/farms/:farmId   — remove farm from vendor
```

---

### Admin Dashboard Pages (complete spec)

#### Page 1: `/admin` — Overview

**Layout:** 3-column grid (stats | map | live feed)

**Row 1 — Stat cards (React Query, 30s refetch):**
```
[ Total Farms: 47 ]  [ Devices: 312 online / 389 total ]
[ Open Alerts: 8 ⚠ ]  [ Gateways online: 12/15 ]
```
Each card: large number + trend arrow (vs yesterday) + colored border by health.

**Row 2 — Map (Leaflet, full width):**
- OpenStreetMap tiles
- Marker per farm at GPS coords
- Marker color: 🟢 no alerts | 🟡 warning | 🔴 critical
- Click marker → navigate to `/admin/farms/:id`
- District outlines (optional GeoJSON for Telangana districts)

**Row 3 — Two panels:**
- Left: MQTT live feed (last 20 messages, auto-scroll, topic + payload snippet + timestamp, color-coded by topic prefix)
- Right: Pi4 infra strip (CPU%, RAM%, MQTT client count, DB size) — polling 10s

---

#### Page 2: `/admin/farms` — Farm Registry

**Filter bar:** Search by name | District dropdown | Crop type | Status (all/online/offline)

**Table columns:**
`Farm name | Owner | District/Mandal | Acres | Crops | Motors | Clusters | Last seen | Status`

Row click → `/admin/farms/:id`

**Pagination:** 50 per page

---

#### Page 3: `/admin/farms/:farmId` — Farm Detail

**Top strip:** Farm name | Owner name + phone | District/Mandal/Village | Area | Crop | Water source

**Left column (60%):**
- **Motors section:** Card per motor
  - Device ID | Name | run state | Power kW | 3-phase voltage (R/Y/B) | Health badge
  - 24h energy sparkline (Recharts LineChart, 100px tall)
  - Last seen timestamp
- **Field Clusters section:** Card per cluster (= 1 weather station + N valves)
  - Weather station: temp, humidity, soil moisture (L1/L2/L3 bars), rainfall today, battery%
  - Valves sub-grid: each valve chip — name | I2C addr (0x21) | open/closed icon | flow rate | control mode badge

**Right column (40%):**
- Active alerts for this farm (sorted by severity)
- Expert review history (reverse chronological) — each note shows expert name, date, recommendation

**Bottom:** 3 Recharts charts in a row
- 24h energy (motor kWh stacked bar)
- 7d soil moisture trend (line per zone/cluster)
- 7d rainfall (bar)

---

#### Page 3a: `/admin/clusters/:stationId` — Field Cluster Detail

**Header card:** Weather station name | device ID | online status | last seen | LoRa signal

**Sensor panel:** 6-cell grid
```
[ Air Temp: 31°C ]  [ Humidity: 68% ]  [ Soil Temp: 27°C ]
[ Soil L1: 52% ]    [ Soil L2: 47% ]   [ Soil L3: 41% ]
[ Leaf Wet 1: 35% ] [ Wind: 2.3 m/s ]  [ Rain today: 4mm ]
[ Battery: 87% ]    [ Light: 8500 lux ] [ Signal: -78 dBm ]
```
Each cell: current value + mini 24h sparkline.

**Valve bus (I2C children):** Horizontal card list, sorted by I2C address
```
┌─────────────────────────────────────────────────────┐
│ 0x21  VLV-001-Z1  "Zone 1 - Paddy"                 │
│ 🟢 OPEN  |  Flow: 12.4 L/min  |  Mode: auto        │
│ Auto thresholds: L2 35%–70%  |  Last: 2m ago        │
└─────────────────────────────────────────────────────┘
```
Each card: I2C addr | device ID | zone name | open/closed toggle (read-only for non-control roles) | flow rate | volume today | control mode | last toggle timestamp | last LoRa heartbeat

**Alerts:** All alerts where `device_id` matches station or any of its valves.

---

#### Page 4: `/admin/devices` — Device Registry

**Tabs:** All | Motors | Valves | Weather Stations

**Table per tab:**
```
Device ID | Name | Farm | Firmware | Last Seen | Signal | Status | Actions
```
Actions: View telemetry | Send command (superadmin/technician only) | Mark maintenance

**OTA sub-page `/admin/devices/ota`** (technician/superadmin):
- File upload for firmware .bin
- Device selector (filter by firmware version)
- Progress indicator
- Rollback option

---

#### Page 5: `/admin/alerts` — Alert Center

**Filter bar:** Severity | Type | Ack status | Farm | Date range

**Table:**
```
Sev badge | Alert name | Farm | Device | Time | Action text | Ack btn | Create issue btn
```

**Bulk actions:** Select all → Acknowledge selected | Assign selected to issue

**Alert types shown:** `device` | `pest` | `weather` | `irrigation` | `system`

---

#### Page 6: `/admin/issues` — Issue Tracker

**View toggle:** Kanban | Table

**Kanban:** 3 columns — Open | In Progress | Resolved
Each card:
```
┌──────────────────────────────────┐
│ ⚠ [Farm: Ravi's Farm, Nellore]  │
│ Motor overload — 3 consecutive   │
│ Priority score: 8.5              │
│ Assigned: K. Ramaiah (tech)      │
│ 3 days open                      │
└──────────────────────────────────┘
```
Drag-drop between columns (react-beautiful-dnd → @hello-pangea/dnd).

Priority score formula: `(sev_weight × 3) + (days_open × 0.5) + (unacked_alerts × 1)`
where sev_weight: critical=3, warning=2, info=1.

**Issue detail modal:** Full description | Linked alert | Timeline of status changes | Comment/notes field | Assign dropdown (lists all technician-role users)

---

#### Page 7: `/admin/expert` — Agri-Expert Review Panel

**Anomaly queue (left panel, 30% width):**
List of farms/devices where any sensor exceeded threshold in last 24h.
Each item: Farm name | Sensor that triggered | Current value vs threshold | Hours since anomaly
Click item → loads that farm's sensor data in main panel.

**Review panel (right, 70% width):**
- Farm info header
- **Sensor chart:** 7-day Recharts LineChart — select which sensors to overlay (temp, humidity, soil moisture L1/L2/L3, leaf wetness)
- **Review form:**
  ```
  Note: [text area — free text]
  Recommendation:
    ○ Increase irrigation  ○ Reduce irrigation  ○ Apply fungicide
    ○ Check soil pH        ○ Inspect device     ○ Custom: [text]
  Severity: ○ info  ○ warning  ○ critical
  [ Push to farmer app ]  [ Save draft ]
  ```
- Push to farmer: creates an advisory entry visible in Sasyamithra mobile app notification center.

**Review history (below form):** Past reviews for this farm, reverse chronological.

---

#### Page 8: `/admin/users` — Farmer Management

**Table:**
```
Name | Email | Phone | District/Mandal | Farms | Motors | Joined | Last active | Status
```
Row click → user detail modal: profile fields + farm list + device counts + recent alerts.

**Export button:** CSV with name, phone, district, mandal, village, farm name, crop, acreage.

---

#### Page 9: `/admin/vendors` — Vendor Management (superadmin only)

**Table:** Vendor name | Email | Phone | District | Farms assigned | Created

**Create vendor form:**
- Name, email, password, phone, district
- Assign farms (multi-select from all farms list)
- Per-farm access level: read | control | full

**Row actions:** Edit | Deactivate | View vendor's farms

---

#### Page 10: `/admin/infra` — Platform Infrastructure

**Pi4 card:**
```
CPU: 34%  ████░░░░░░
RAM: 1.2GB / 4GB (30%)  ███░░░░░░░
Disk: 18GB / 64GB (28%)  ███░░░░░░░
Uptime: 14d 6h 22m
```

**Mosquitto card:**
- Connected clients count | Messages/sec | Broker version
- Status: 🟢 Running

**InfluxDB card:**
- Write rate | Series count | Bucket name | Disk usage

**SQLite card:**
- DB size | Last write timestamp

Polling every 10 seconds via React Query.

---

#### Page 11: `/admin/audit` — Audit Log (superadmin only)

**Table:** Timestamp | Actor (admin name + role) | Action | Target | IP address
Actions logged: login, ack_alert, create_issue, assign_issue, resolve_issue, post_review, motor_toggle, valve_toggle, vendor_create, vendor_farm_assign

---

### Real-Time Architecture (MQTT WebSocket)

**Pi4 Mosquitto config** — add to `/etc/mosquitto/mosquitto.conf`:
```
listener 9001
protocol websockets
allow_anonymous false
```

**Hook:** `src/hooks/useMqttFeed.ts`
```typescript
useMqttFeed(topics: string[], onMessage: (topic, payload) => void)
```
- Connects to `ws://{PI4_HOST}:9001` using MQTT.js
- Admin token used for MQTT auth: `{ username: 'admin', password: accessToken }`
- Reconnects with 5s backoff on disconnect
- Returns `{ connected, lastMessage, messageLog }`

**Topics subscribed by admin:**
- `data/#` — telemetry from all devices (filter by selectedFarmId if set)
- `alert/#` — all alerts
- `status/#` — device heartbeats

**Vite dev proxy** (for local dev where Pi4 is remote):
```typescript
'/ws': { target: 'ws://raspberrypi.local:9001', ws: true }
```

---

## Part 3 — Vendor Dashboard

### Vendor Account Lifecycle

1. Superadmin creates vendor in `/admin/vendors` → sets email/password/farms
2. Vendor receives credentials (out-of-band: SMS/WhatsApp)
3. Vendor logs in at `http://pi4:5174/vendor/login`
4. JWT issued with `role: vendor, vendorId: vnd_xxxxx`
5. All `/vendor/*` routes filter data through `vendor_farms` join table

### Vendor Dashboard Pages (complete spec)

#### `/vendor` — Borewell Control Center

**Top command strip:**
```
● 14 RUNNING  ○ 6 STOPPED  ⚠ 3 ALERTS  ⚡ 142 kWh today
[ Start All Scheduled ]  [ ⛔ Emergency Stop All ]
```
Emergency Stop: requires password re-entry (critical action guard).

**Motor grid (primary view):** Large cards, 2-3 per row, designed for 10" tablets
```
┌─────────────────────────────────────────┐
│  MOT-001-A1  |  Farm: Ravi's Field      │
│                                         │
│   ●  RUNNING         [  STOP  ]         │
│                                         │
│  Phase R: 231V / 4.2A                   │
│  Phase Y: 229V / 4.1A                   │
│  Phase B: 230V / 4.3A                   │
│                                         │
│  Power: 2.8 kW | PF: 0.92 | 🟢 Good    │
│  Run: 4h 32m today | kWh: 12.7         │
└─────────────────────────────────────────┘
```
Color-coded by health: green border = good, yellow = warning, red = fault.

**Right panel:** Live alert feed (MQTT WS) + recent command log.

---

#### `/vendor/farms` — Farm Cards (tablet-optimized)

Grid of farm cards (not table):
```
┌──────────────────────────┐
│  Ravi's Farm             │
│  Nellore, AP             │
│                          │
│  3 pumps ● 2 running     │
│  2 clusters | 8 valves   │
│  Last seen: 1m ago       │
│  ⚠ 1 warning alert       │
└──────────────────────────┘
```
Tap → `/vendor/farms/:id`

---

#### `/vendor/farms/:farmId` — Farm Pump Map

**Zone map (Leaflet or SVG):**
- Each pump plotted at GPS coords
- Color: green=running, grey=stopped, red=fault
- Tap pump → right side panel slides in

**Right side panel (on pump tap):**
- Motor detail: phases, power, runtime
- 24h power chart (Recharts)
- START / STOP button
- Last 10 commands for this motor

**Cluster grid below map:**
- Field cluster cards with weather readings
- Valve open/close chips below each cluster

---

#### `/vendor/energy` — Energy Dashboard

**Summary cards:**
```
Today: 142 kWh  |  Week: 891 kWh  |  Month: 3,241 kWh
Avg PF: 0.91    |  Peak: 8.4 kW   |  Est. cost: ₹2,347
```
Cost = `kWh × ₹/unit` (configurable in vendor settings, default ₹7/unit).

**Stacked bar chart:** Each bar = 1 day, stacked by motor.

**Table:**
```
Motor | Farm | kWh today | kWh week | kWh month | Avg PF | Hours | Status
```

**Export CSV** button — for billing reports.

---

#### `/vendor/schedule` — Irrigation Schedule (Gantt)

Gantt timeline: each valve = one row, x-axis = 24h or 7d.
Scheduled irrigation slots shown as colored bars.

**Conflict panel:** Highlights if multiple high-draw motors scheduled simultaneously.
```
⚠ Conflict: 3 motors scheduled 06:00–08:00 on Farm A
   Combined load: ~14kW — check grid capacity
```

---

#### `/vendor/alerts` — Alert Feed (scoped to vendor farms)

Same layout as admin alerts table.
Extra action: **"Raise Service Ticket"** → creates issue + sends WhatsApp/SMS to vendor contact.

---

## Part 4 — Offline / Low-Connectivity Mode

Pi4 may be on a remote SIM card with intermittent connectivity. Admin/vendor on a remote tablet.

### Strategy

**Service Worker (Workbox via `vite-plugin-pwa`):**
- Cache all static assets (JS/CSS/HTML) → app opens even offline
- Cache last-good API responses for: overview, farms list, farm details
- Show "Stale data — last updated 3m ago" banner when serving from cache

**IndexedDB queue (via `idb` library):**
- Commands (motor toggle, valve toggle, alert ack) that fail when offline are queued in IndexedDB
- On reconnect, the queue is replayed in order
- User sees: "⏳ 2 actions pending — reconnecting..."

**Connectivity detection:**
- Poll `/api/health` every 15 seconds
- `navigator.onLine` + fetch-based check
- Status bar at top of layout: 🟢 Connected | 🟡 Reconnecting | 🔴 Offline (cached data)

**MQTT reconnect:**
- MQTT.js has built-in reconnect (already configured with 5s backoff)
- On reconnect, re-subscribe to all topics

---

## Part 5 — Alert Notification System

### In-App Notifications

- **Notification bell** in top bar: badge count (unacked alerts)
- Badge count polled from `/admin/alerts?ack=false` every 30s
- Clicking bell → slides open notification drawer with last 20 alerts
- Each alert: severity icon | message | farm | time | Ack button

### Browser Push Notifications (Web Push API)

For alerts while admin has tab open in background.

**Service Worker registers for push:**
```typescript
navigator.serviceWorker.register('/sw.js')
  .then(reg => reg.pushManager.subscribe({ userVisibleOnly: true, applicationServerKey: VAPID_PUBLIC_KEY }))
```

**Pi4 server side:**
- Add `web-push` npm package
- New route: `POST /admin/push/subscribe` — saves push subscription to DB
- MQTT handler: on `alert/#` message with `sev=critical` → call `webpush.sendNotification()`

**Notification appearance:**
```
AGRINET ALERT                    [AGRINET logo]
🔴 Motor Fault — Ravi's Farm
MOT-001-A1: Dry run protection triggered
[View]  [Ack]
```

### WhatsApp / SMS (India-specific)

For critical alerts even when dashboard is closed. Via **Twilio / D7 Networks** (common in India).

**Flow:** MQTT handler receives `alert/*/warn` with `sev=critical` → look up farm owner's phone → send WhatsApp via Twilio API or SMS via D7.

**Config (in `config.js`):**
```
TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN, TWILIO_WHATSAPP_FROM
D7_API_KEY, D7_SMS_FROM
NOTIFY_BACKEND: 'twilio' | 'd7' | 'none'
```

**Message template:**
```
🔴 AGRINET Alert: [Alert name]
Farm: [farm name] | Device: [device ID]
Action: [action text]
Reply STOP to unsubscribe
```

### Notification Routing Rules

| Severity | Admin (browser push) | Vendor (browser push) | Farmer (mobile app) | SMS/WhatsApp |
|----------|---------------------|----------------------|---------------------|--------------|
| `critical` | ✅ | ✅ | ✅ | ✅ farmer only |
| `warning` | ✅ | ✅ | ✅ (badge only) | ❌ |
| `info` | ❌ (badge count only) | ❌ | ❌ | ❌ |

---

## Part 6 — Directory Structure (Final)

```
~/Projects/agrinet-admin/
├── package.json
├── vite.config.ts           (proxy /api + /ws)
├── tailwind.config.ts
├── tsconfig.json
├── index.html
└── src/
    ├── main.tsx
    ├── index.css
    ├── App.tsx               (router: /admin/* + /vendor/*)
    │
    ├── api/
    │   ├── client.ts         (axios + JWT interceptor)
    │   ├── adminApi.ts       (all /admin/* calls)
    │   └── vendorApi.ts      (all /vendor/* calls)
    │
    ├── hooks/
    │   ├── useMqttFeed.ts    (MQTT.js WS + reconnect)
    │   ├── useOnlineStatus.ts (navigator.onLine + /health poll)
    │   └── useOfflineQueue.ts (IndexedDB action queue)
    │
    ├── components/
    │   ├── layout/
    │   │   ├── AdminLayout.tsx   (sidebar + topbar + notification bell)
    │   │   └── VendorLayout.tsx  (simpler tablet-first sidebar)
    │   ├── shared/
    │   │   ├── StatusBadge.tsx       (good/warning/critical/offline)
    │   │   ├── MotorCard.tsx         (motor status card — admin + vendor)
    │   │   ├── ValveChip.tsx         (compact valve state chip)
    │   │   ├── ClusterCard.tsx       (weather station + valve children)
    │   │   ├── AlertRow.tsx          (alert table row)
    │   │   ├── MqttFeedPanel.tsx     (scrolling live MQTT log)
    │   │   ├── OfflineBanner.tsx     (stale data warning bar)
    │   │   └── NotificationDrawer.tsx (slide-in alert tray)
    │   ├── charts/
    │   │   ├── EnergyChart.tsx       (kWh bar/line, Recharts)
    │   │   ├── MoistureChart.tsx     (soil L1/L2/L3 lines)
    │   │   ├── PhaseChart.tsx        (3-phase V/A/PF lines)
    │   │   └── Sparkline.tsx         (100px inline chart)
    │   └── map/
    │       ├── FarmMap.tsx           (Leaflet overview)
    │       └── FarmPin.tsx           (colored marker)
    │
    ├── pages/
    │   ├── LoginPage.tsx             (shared login, routes to /admin or /vendor)
    │   ├── admin/
    │   │   ├── OverviewPage.tsx
    │   │   ├── FarmsPage.tsx
    │   │   ├── FarmDetailPage.tsx
    │   │   ├── ClusterDetailPage.tsx
    │   │   ├── DevicesPage.tsx       (includes OTA tab)
    │   │   ├── AlertsPage.tsx
    │   │   ├── IssuesPage.tsx        (Kanban + Table)
    │   │   ├── ExpertPage.tsx
    │   │   ├── UsersPage.tsx
    │   │   ├── VendorsPage.tsx       (superadmin only)
    │   │   ├── InfraPage.tsx
    │   │   └── AuditPage.tsx         (superadmin only)
    │   └── vendor/
    │       ├── ControlCenterPage.tsx (motor grid + stop-all)
    │       ├── VendorFarmsPage.tsx
    │       ├── FarmPumpMapPage.tsx
    │       ├── EnergyPage.tsx
    │       ├── SchedulePage.tsx
    │       └── VendorAlertsPage.tsx
    │
    └── store/
        └── adminStore.ts   (admin token, vendor token, alert count, offline queue)
```

---

## Part 7 — Pi4 Server Files (complete list of changes)

| File | Action | What changes |
|------|--------|-------------|
| `adapters/influx.js` | **CREATE** | `queryTelemetry(deviceId, range)` |
| `adapters/db.js` | **EXTEND** | Admin/vendor query functions (already done, + SQL injection fixes) |
| `db/vendors.js` | **CREATE** | DDL for vendors, vendor_farms, issues, expert_reviews + ALTER migrations |
| `routes/admin.js` | **CREATE** | All `/admin/*` routes + preHandler auth |
| `routes/vendor.js` | **CREATE** | All `/vendor/*` routes + preHandler auth |
| `routes/audit.js` | **CREATE** | `GET /admin/audit` — command + action log |
| `routes/notifications.js` | **CREATE** | `POST /admin/push/subscribe` + VAPID push send |
| `server.js` | **EXTEND** | Register 5 new route files + runVendorMigrations() |
| `init-db.js` | **EXTEND** | Add district/mandal/village/farm/crop/acreage cols to users |
| `mqtt-handler.js` | **EXTEND** | On critical alert → trigger web-push + WhatsApp/SMS |
| `config.js` | **EXTEND** | VAPID_PUBLIC/PRIVATE_KEY, TWILIO_*, NOTIFY_BACKEND |
| `package.json` | **EXTEND** | Add `web-push`, `idb` (for IndexedDB offline queue in frontend) |

---

## Part 8 — Execution Order (Revised)

### Phase 1: Server fixes + new backend (Pi4)
1. Fix `init-db.js` — missing user columns
2. Create `adapters/influx.js`
3. Fix SQL injection in db.js vendor functions
4. Refactor admin/vendor routes to use preHandler auth
5. Add missing indexes to `db/vendors.js`
6. Create `routes/audit.js`
7. Create `routes/notifications.js` + MQTT handler webhook
8. Update `server.js` + `config.js`

### Phase 2: Frontend core
9. Complete scaffolding (already partially done)
10. `App.tsx` — router + auth guards
11. `AdminLayout` + `VendorLayout` + `LoginPage`
12. Shared components (StatusBadge, MotorCard, ClusterCard, AlertRow)
13. Charts (EnergyChart, MoistureChart, Sparkline)
14. Hooks (useMqttFeed, useOnlineStatus, useOfflineQueue)

### Phase 3: Admin pages (priority order)
15. Overview page (stat cards + map + MQTT feed)
16. Farm detail + Cluster detail (highest field value)
17. Alerts page
18. Issues tracker (Kanban)
19. Expert review panel
20. Users + Vendors pages
21. Infra + Audit pages
22. Devices + OTA page

### Phase 4: Vendor pages
23. Control Center (motor grid)
24. Farm card grid + Farm pump map
25. Energy dashboard + CSV export
26. Schedule Gantt view
27. Vendor alerts

### Phase 5: Offline + Notifications
28. Service Worker setup (vite-plugin-pwa)
29. IndexedDB offline queue
30. Connectivity banner
31. Web Push subscription + send
32. SMS/WhatsApp integration (optional, config-driven)

---

## Part 9 — Verification Checklist

### Server
| Check | Expected |
|-------|----------|
| `node server.js` starts without errors | No MODULE_NOT_FOUND |
| `POST /admin/auth/login` with farmer creds | 401 — not admin |
| `POST /admin/auth/login` with superadmin creds | 200, role=superadmin |
| `GET /admin/farms` (no token) | 401 |
| `GET /admin/overview` (superadmin token) | Stats object |
| `GET /admin/infra` (superadmin token) | CPU/RAM/MQTT data |
| `POST /vendor/auth/login` | 200, role=vendor |
| `GET /vendor/farms` (vendor token) | Only contracted farms |
| `GET /vendor/farms` (admin token) | 403 |
| `GET /admin/telemetry/MOT-001-A1?range=24h` | Array of time-series |

### Frontend
| Check | Expected |
|-------|----------|
| Build: `npm run build` | Zero errors |
| Login with superadmin | Redirects to `/admin/` |
| Login with vendor | Redirects to `/vendor/` |
| Login with farmer creds | Error: not an admin/vendor |
| Offline: kill Pi4 server | Banner shows, cached data displayed |
| Offline: toggle motor | Queued, retry on reconnect |
| Farm map | Markers at GPS coords, click → farm detail |
| Live MQTT feed | Messages appear within 2s of device transmit |
| Web push | Critical alert triggers browser notification |
| Cluster detail | Shows weather + I2C valve children |
| Issue Kanban | Drag card between columns, status updates |

| Interface | Audience | Scope |
|-----------|----------|-------|
| **Sasyamithra Mobile App** | Individual farmer | Own farm: motors, valves, weather, alerts |
| **Admin Dashboard** (this plan) | Platform admin, agri-expert, technician | All farms cross-tenant, device registry, issues, expert review |
| **Vendor Dashboard** (this plan) | Borewell/pump vendor, large estate manager | Multi-borewell cluster control, bulk commands, energy across all borewells |

---

## Admin Dashboard Context

Need a web-based admin dashboard that sits alongside the existing Pi4 Fastify REST API. The Sasyamithra mobile app serves individual farmers. The admin dashboard serves:
- **Platform admins** — full system oversight across all farms/customers
- **Agri-experts** — review soil/weather data, approve irrigation recommendations, flag issues
- **Field technicians** — device health, firmware updates, on-site troubleshooting

The Pi4 Fastify server at `sasyamithra-server-FIXED/` exposes REST APIs scoped per-user. Admin needs **cross-user/cross-farm** visibility — requires new admin API routes with elevated privileges.

---

## Field Node Topology (Critical — missed in prior plan)

```
FIELD CLUSTER (per zone / per channel)
┌──────────────────────────────────────────────────────────┐
│  Weather Station Node (AGN-W, STM32L476RG)               │
│  ┌────────────────────────────────────┐                  │
│  │  I2C MASTER                        │                  │
│  │  ├── BME280  (temp/humidity/pres)  │   LoRa TX        │
│  │  ├── AHT20   (air temp/humidity)  │──────────────→   │
│  │  ├── DS18B20 (soil temp)           │   WeatherPayload │
│  │  ├── Soil moisture probes (L1/L2/L3)│  (32 bytes)    │
│  │  ├── Leaf wetness sensors x2       │                  │
│  │  ├── Rain gauge, anemometer        │                  │
│  │  └── I2C Valve slaves:             │                  │
│  │       ├── Valve 1 @ 0x21 (STM8)   │                  │
│  │       ├── Valve 2 @ 0x22 (STM8)   │                  │
│  │       └── ... up to 10 valves      │                  │
│  └────────────────────────────────────┘                  │
└──────────────────────────────────────────────────────────┘
         ↓ LoRa (866MHz, SF12, IN865)
┌────────────────────────────────┐
│  STM32F103C8T6 GATEWAY         │
│  • Receives WeatherPayload      │
│  • Receives ValvePayload        │
│  • Routes both via SIM7670C    │
│  • MQTT → Pi4 Mosquitto        │
└────────────────────────────────┘
         ↓ MQTT
Pi4 → SQLite (stations + valves linked by station_id) + InfluxDB
```

**Key facts:**
- `valves.station_id` FK in SQLite groups 1–10 valve controllers under one weather station
- Weather station = LoRa TX node + I2C master to both sensors AND valve slave boards
- Valve controllers (STM8) are I2C slave-only — no sensor capability
- Gateway receives SEPARATE LoRa packets for weather (32B) and each valve (12B)
- Admin dashboard must represent a **"Field Cluster"** = 1 weather station + N valves

---

## Architecture

```
Admin Browser
      │
      ▼
React + TypeScript + Vite (static, served from Pi4 or CDN)
      │
      ├── REST  → Pi4 Fastify  (port 3000) — existing routes + new /admin/* routes
      └── WS    → Pi4 MQTT WS  (port 9001) — live telemetry for real-time dashboards
                       │
                       └── Mosquitto (port 1883) → STM32 devices via SIM7670C
```

**Tech Stack:**
- React 18 + TypeScript + Vite (matches existing mobile app stack)
- Tailwind CSS (utility-first, no component library overhead)
- Recharts (charts — already used in mobile app)
- React Query (server state, caching, auto-refresh)
- MQTT.js over WebSocket (port 9001) for live device feed
- React Router v6 (SPA routing)
- Leaflet.js (farm map with device pin overlay)

**Output:** Single directory `~/Projects/agrinet-admin/` — standalone Vite project, deployed as static files to Pi4 `/var/www/admin/`

---

## Roles & Access

| Role | Access |
|------|--------|
| `superadmin` | All farms, all users, system config, Pi4 infra stats |
| `agri_expert` | All farm sensor data (read), write review notes, approve/reject recommendations |
| `technician` | Device health, firmware OTA, command log, not farmer PII |
| `farmer` | Existing mobile app only (not admin dashboard) |

Admin JWT: same `JWT_SECRET` from Pi4 `config.js`, but `role` claim must be `superadmin | agri_expert | technician`. New `POST /admin/auth/login` checks role before issuing token.

---

## New Pi4 API Routes Required (`/admin/*`)

These must be added to `sasyamithra-server-FIXED/server.js`:

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/admin/auth/login` | Admin login (role-gated) |
| GET | `/admin/overview` | Totals: farms, devices, active alerts, online % |
| GET | `/admin/farms` | All farms (paginated, filter by district/mandal) |
| GET | `/admin/farms/:farmId` | Single farm + all its devices + latest telemetry |
| GET | `/admin/farms/:farmId/clusters` | All field clusters (stations + their valve children) |
| GET | `/admin/clusters/:stationId` | Single cluster: weather station + valves (by station_id FK) |
| GET | `/admin/devices` | All devices across all farms (filter: type, status, offline) |
| GET | `/admin/alerts` | All unacknowledged alerts (cross-farm, sorted by severity) |
| PATCH | `/admin/alerts/:id/ack` | Acknowledge alert |
| GET | `/admin/users` | All registered farmers |
| GET | `/admin/users/:id` | Farmer profile + farms + device list |
| POST | `/admin/review` | Expert posts review note on farm/device |
| GET | `/admin/review/:farmId` | Get review history for a farm |
| GET | `/admin/issues` | All open issues (alerts + expert flags) prioritized |
| POST | `/admin/issues/:id/assign` | Assign issue to technician |
| PATCH | `/admin/issues/:id/resolve` | Mark issue resolved |
| GET | `/admin/telemetry/:deviceId` | Cross-farm telemetry history (InfluxDB) |
| GET | `/admin/infra` | Pi4 system health: CPU, RAM, disk, MQTT broker status |

---

## Dashboard Pages & Components

### 1. `/` — Overview (Command Center)
**Cards row:**
- Total active farms | Online devices / Total | Open critical alerts | Online gateways %

**Live feed panel:** MQTT WebSocket — shows last 20 MQTT messages in real-time (topic + payload summary)

**Alert heatmap:** Telangana district map (Leaflet) — dots at farm GPS coords, colored by worst active alert severity

**System health strip:** Pi4 CPU %, RAM %, Mosquitto connected clients, InfluxDB write rate

### 2. `/farms` — Farm Registry
**Table:** farmId | Name | District/Mandal/Village | Acres | Crops | Devices count | Last telemetry | Status
**Filters:** district dropdown, crop type, online/offline toggle, search by name
**Row expand:** inline device list

### 3. `/farms/:farmId` — Farm Detail
**Left panel:** Farm info card (location, crop, soil type, water source)
**Center:** Two sections:
  - **Motors** list (with star-delta state, kWh, health)
  - **Field Clusters** — each cluster = 1 weather station + its valve children
    - Cluster card shows: weather station online/offline, sensor readings (temp/humidity/soil moisture)
    - Under each cluster: valve list with I2C address (0x21..0x2A), position%, flow, last toggle
**Right panel:** Active alerts for this farm + expert review history
**Bottom:** 24h energy chart (motor kWh), soil moisture trend per cluster, rainfall bar

### 3a. `/farms/:farmId/clusters/:stationId` — Field Cluster Detail
**Header:** Weather station card — last 24h sensor trend (airT, humidity, soilM L1/L2/L3, rain, wind)
**Valve grid:** Each valve card shows:
  - I2C address (0x21-0x2A), device ID, name
  - Open/closed state, current position%, flow rate L/min
  - Control mode (auto/schedule/quantity/manual)
  - Auto thresholds from associated weather station (L2 min/max %)
  - Last command + last LoRa seen timestamp
**Alerts:** Alerts scoped to this cluster (weather + valve alerts combined)

### 4. `/devices` — Device Registry
**Table:** deviceId | Type | farmId | Firmware | Last seen | Signal strength | Status
**Filters:** type, status (online/offline/fault), firmware version
**Actions:** per-row — view telemetry, send command, mark for maintenance

### 5. `/alerts` — Alert Center
**Table:** Severity badge | Alert name | Farm | Device | Time | Action required | Ack button
**Filters:** severity, type (device/pest/weather), ack status, farm
**Bulk actions:** Acknowledge selected | Assign to technician

### 6. `/issues` — Issue Tracker
Issue = Alert + Expert flag + Field ticket
**Kanban view:** Open → In Progress → Resolved
**Each card:** Issue title | Farm name | Severity | Assigned technician | Days open
**Priority formula:** `score = (severity_weight * 3) + (days_open * 0.5) + (unacked_alerts_count)`
- critical=3, warning=2, info=1

**Create issue:** manually from any alert or from expert review

### 7. `/expert` — Agri-Expert Review Panel
**Queue:** Farms with recent sensor anomalies needing review (soil moisture out of range, leaf wetness spike, soil temp extreme)
**Review form:** Farm selector → sensor graph (7d) → free-text note + recommendation
**Recommendations:** "Increase irrigation frequency", "Apply fungicide", "Check soil pH"
**Approval flow:** Expert submits → superadmin reviews (optional) → pushed to farmer app as advisory

### 8. `/users` — Farmer Management
**Table:** Name | Email | Phone | District | Farms | Devices | Joined | Last active
**Row expand:** Farm list with quick device counts
**Export:** CSV download (name, contact, location, crop)

### 9. `/infra` — Platform Infrastructure
**Pi4 status card:** CPU%, RAM%, Disk%, uptime
**Mosquitto card:** Connected MQTT clients, message rate (msg/s), broker version
**InfluxDB card:** Write rate, series count, disk usage
**SQLite card:** DB size, last write timestamp
**Service health checks:** Live `GET /health` poll every 10s

---

## Real-time (MQTT WebSocket)

Mosquitto must enable WebSocket listener on port 9001 in `/etc/mosquitto/mosquitto.conf`:
```
listener 9001
protocol websockets
```

Admin dashboard subscribes:
- `data/#` — all telemetry (filter by selected farm)
- `alert/#` — live alerts
- `status/#` — device heartbeats

React hook `useMqttFeed()` wraps MQTT.js, re-exports observables per topic prefix.

---

## Execution Plan — 8 Steps

### Step A: Pi4 Server — Add Admin Routes
**File:** `sasyamithra-server-FIXED/server.js` + new `routes/admin.js`
**Add:** All `/admin/*` routes listed above + role middleware guard
**Add:** `/admin/infra` — reads `os` module for CPU/RAM + pings Mosquitto via MQTT

### Step B: Scaffold Vite Project
```
~/Projects/agrinet-admin/
├── src/
│   ├── api/           — axios + react-query hooks (useAdminOverview, useFarms, etc.)
│   ├── components/    — shared UI (StatusBadge, AlertCard, DeviceRow, MapPin)
│   ├── hooks/         — useMqttFeed, useAdminAuth
│   ├── pages/         — Overview, Farms, FarmDetail, Devices, Alerts, Issues, Expert, Users, Infra
│   ├── store/         — zustand (auth token, selected farm, alert count)
│   └── main.tsx
├── tailwind.config.ts
├── vite.config.ts
└── package.json
```

### Step C: Auth + Layout
- Login page → `POST /admin/auth/login` → JWT stored in `localStorage`
- Sidebar nav with role-based menu (technician doesn't see `/users`)
- Top bar: live alert badge count (polling `/admin/alerts?ack=false` every 30s)

### Step D: Overview Page
- 4 stat cards (React Query, 30s refetch)
- Leaflet map with farm markers (click → navigate to `/farms/:id`)
- MQTT live feed panel (last 20 messages)
- Pi4 infra strip (React Query, 10s refetch)

### Step E: Farms + Farm Detail Pages
- Farms table with search/filter
- Farm detail: device list + alert panel + expert notes
- 24h charts using Recharts (energy, moisture, rainfall)

### Step F: Alerts + Issue Tracker
- Alerts table with ack/assign actions
- Issue kanban with drag-drop (react-beautiful-dnd)
- Priority auto-calculated, manual override allowed

### Step G: Expert Review Panel
- Anomaly queue: devices where any sensor exceeded threshold in last 24h
- 7-day sensor graph per device (InfluxDB history)
- Review form: structured note + recommendation + optional push to farmer

### Step H: Infra Page + Mosquitto WS
- Add `listener 9001 / protocol websockets` to Mosquitto config
- Pi4 infra page with polling
- MQTT live feed hook

---

## Directory Structure

```
~/Projects/agrinet-admin/
├── package.json         (React 18, Vite, Tailwind, Recharts, Leaflet, MQTT.js, React Query, Zustand)
├── vite.config.ts       (proxy /api → http://pi4:3000, /mqtt → ws://pi4:9001)
├── tailwind.config.ts
├── index.html
└── src/
    ├── main.tsx
    ├── App.tsx           (router + auth guard)
    ├── api/
    │   ├── client.ts     (axios with JWT interceptor)
    │   ├── admin.ts      (all /admin/* endpoint callers)
    │   └── queries.ts    (react-query hooks)
    ├── hooks/
    │   ├── useMqttFeed.ts
    │   └── useAdminAuth.ts
    ├── components/
    │   ├── layout/       (Sidebar, Topbar, PageWrapper)
    │   ├── cards/        (StatCard, DeviceCard, AlertCard)
    │   ├── charts/       (EnergyChart, MoistureChart, PhaseChart)
    │   └── map/          (FarmMap, DevicePin)
    ├── pages/
    │   ├── LoginPage.tsx
    │   ├── OverviewPage.tsx
    │   ├── FarmsPage.tsx
    │   ├── FarmDetailPage.tsx
    │   ├── ClusterDetailPage.tsx   ← NEW: weather station + child valves
    │   ├── DevicesPage.tsx
    │   ├── AlertsPage.tsx
    │   ├── IssuesPage.tsx
    │   ├── ExpertPage.tsx
    │   ├── UsersPage.tsx
    │   └── InfraPage.tsx
    └── store/
        └── adminStore.ts  (zustand: auth, selectedFarm, alertCount)
```

---

## Verification

| Check | Action | Expected |
|-------|--------|----------|
| Admin login | POST `/admin/auth/login` with superadmin creds | JWT with `role: superadmin` |
| Cross-farm farms list | GET `/admin/farms` | Returns all farms, not just own |
| Live MQTT feed | WS connect to Pi4 port 9001 | Events appear in Overview live panel |
| Farm map | Open `/` Overview | Farm markers appear at correct GPS coords |
| Expert review | Submit note on farm → check farmer app | Advisory appears in farmer app |
| Issue priority | Create 3 issues (critical, warning, info) | Sorted by computed score |
| Infra page | `/infra` | CPU/RAM/MQTT client count populated |
| Role guard | Login as `technician`, visit `/users` | Redirected or 403 |

---

## Implementation Order

1. **First**: Add admin + vendor routes to Pi4 server (Step A) — unblocks all UI work
2. **Then**: Scaffold + auth + layout (Steps B, C) — shared shell for both dashboards
3. **Then**: Overview + Farms + Cluster pages (Steps D, E) — highest value views
4. **Then**: Alerts + Issues (Step F)
5. **Then**: Expert panel (Step G)
6. **Then**: Vendor dashboard scaffold + borewell pages (Step I)
7. **Last**: Infra page + MQTT WS (Step H)

---

---

# AGRINET Vendor Dashboard — Design Plan

## Context

Large farms in Telangana often have **5–20+ borewells** across hundreds of acres. A single farmer may not manage these — instead a **vendor** (pump operator, irrigation contractor, equipment dealer) manages the borewell cluster on their behalf. The vendor needs centralized real-time control without navigating individual farmer accounts.

**Target users:**
- Borewell pump operators managing a customer's multi-pump estate
- Irrigation contractors who operate and maintain equipment
- Equipment dealers offering managed-service contracts to farmers

**Key difference from Admin dashboard:**
- Admin sees ALL customers across the platform (internal staff view)
- Vendor sees ONLY their contracted farms/devices (scoped, customer-facing B2B portal)
- Vendor can ACT (start/stop pumps, open/close valves) but cannot see other vendor's farms

---

## Vendor Account Model

New SQLite table `vendors`:
```
id          TEXT PK  (vnd_{uuid8})
name        TEXT     (company or operator name)
email       TEXT UNIQUE
phone       TEXT
district    TEXT
created_at  TEXT
```

New SQLite table `vendor_farms` (many-to-many):
```
vendor_id   TEXT FK → vendors.id
farm_id     TEXT FK → farms.id
access_level TEXT  ('read' | 'control' | 'full')
assigned_at TEXT
```

JWT role: `vendor` — scoped to vendor's contracted farms only.

New Pi4 API routes prefix: `/vendor/*`

---

## New Pi4 API Routes for Vendor (`/vendor/*`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/vendor/auth/login` | Vendor login (role=vendor) |
| GET | `/vendor/overview` | Totals for vendor's farms: pumps online, alerts, energy today |
| GET | `/vendor/farms` | Vendor's contracted farms list |
| GET | `/vendor/farms/:farmId/motors` | All motors in that farm with live telemetry |
| GET | `/vendor/farms/:farmId/clusters` | Field clusters (weather + valves) in farm |
| POST | `/vendor/motors/:id/toggle` | Start/stop pump (if access_level = control or full) |
| POST | `/vendor/valves/:id/toggle` | Open/close valve |
| GET | `/vendor/energy` | Aggregated kWh across all vendor farms (today/week/month) |
| GET | `/vendor/alerts` | All alerts across vendor's farms |
| PATCH | `/vendor/alerts/:id/ack` | Acknowledge alert |
| GET | `/vendor/history/:deviceId` | Motor/valve telemetry history |
| GET | `/vendor/schedule` | All scheduled irrigation jobs across farms |

---

## Vendor Dashboard Pages

**Separate Vite project:** `~/Projects/agrinet-vendor/` OR second route prefix in same project at `/vendor/*`

**Recommended:** Same monorepo, separate route tree in React Router — shared components (charts, maps, MQTT hook), different auth context.

### 1. `/vendor/` — Borewell Control Center
**Command strip at top:**
- `● 14 RUNNING` | `○ 6 STOPPED` | `⚠ 3 ALERTS` | Energy today: `142 kWh`
- **Quick-action row:** "Start All Scheduled" | "Stop All" (with confirmation dialog — irreversible)

**Motor grid:** Card per borewell pump — large, designed for tablet/large phone
- Motor name, farm zone name
- Big ON/OFF toggle button
- Current power (kW), voltage (V avg), run hours today
- Health badge (good/warning/critical)
- Star-delta state indicator

**Right panel:** Live alert feed (MQTT WS)

### 2. `/vendor/farms` — Farm Overview
**Card grid** (not table — better for field operators on tablet):
- Farm name, village
- Pump count, valve cluster count
- Last telemetry timestamp
- Worst active alert badge
- Tap → `/vendor/farms/:farmId`

### 3. `/vendor/farms/:farmId` — Farm Pump Map
**Top:** Farm name, total area, crop
**Pump list with zone map:** Visual zone layout (simple SVG grid or Leaflet with zone polygons)
  - Each pump plotted at its GPS coords (or zone position)
  - Color: green=running, grey=stopped, red=fault
  - Tap pump → side panel with 24h power chart + control buttons

**Cluster grid below:** Each field cluster card
  - Weather station readings (temp, humidity, soil moisture)
  - Valve sub-list: open/closed state, flow rate

### 4. `/vendor/energy` — Energy Dashboard
**Summary cards:** Today kWh | This week kWh | Avg cost (₹/unit × kWh) | Peak demand (kW)
**Chart:** Stacked bar by motor — daily kWh for last 30 days
**Table:** Motor | Farm | kWh today | kWh week | kWh month | Avg PF | Run hours
**Export:** CSV for billing/reporting

### 5. `/vendor/schedule` — Irrigation Schedule View
**Timeline view:** Gantt-style — each valve/motor as a row, time on x-axis
- Scheduled slots shown as colored bars
- Running = solid green, completed = grey, upcoming = dashed blue
**Conflict detection:** Highlight if multiple high-draw motors scheduled simultaneously (potential grid overload)

### 6. `/vendor/alerts` — Vendor Alert Feed
Same as admin alerts but scoped to vendor's farms only.
**Added feature:** "Raise service ticket" button on critical alerts (sends notification to vendor's support team)

---

## Shared Components Between Admin + Vendor Dashboards

```
src/components/shared/
├── MqttFeedPanel.tsx      — live MQTT message list
├── MotorCard.tsx          — motor status + toggle (reused in both)
├── ValveCard.tsx          — valve state + flow (reused in both)
├── ClusterCard.tsx        — weather station + valve children (reused in both)
├── EnergyChart.tsx        — Recharts bar/line (reused in both)
├── AlertRow.tsx           — alert table row with ack button
├── FarmMap.tsx            — Leaflet farm overview map
└── StatusBadge.tsx        — good/warning/critical/offline badges
```

---

## Updated Directory Structure

```
~/Projects/agrinet-admin/
├── package.json
├── vite.config.ts         (proxy /api → Pi4, /ws → Pi4 WS 9001)
└── src/
    ├── App.tsx             (router: /admin/* + /vendor/* + shared login)
    ├── api/
    │   ├── client.ts
    │   ├── adminApi.ts     (/admin/* calls)
    │   └── vendorApi.ts    (/vendor/* calls)
    ├── components/
    │   ├── layout/
    │   │   ├── AdminLayout.tsx
    │   │   └── VendorLayout.tsx
    │   └── shared/         (shared components listed above)
    ├── hooks/
    │   ├── useMqttFeed.ts
    │   ├── useAdminAuth.ts
    │   └── useVendorAuth.ts
    ├── pages/
    │   ├── admin/          (all admin pages)
    │   └── vendor/         (all vendor pages)
    └── store/
        ├── adminStore.ts
        └── vendorStore.ts
```

---

## Updated Pi4 API Steps

### Step A (extended): Add Admin + Vendor Routes
**Files to add/modify in `sasyamithra-server-FIXED/`:**
- `routes/admin.js` — all `/admin/*` routes
- `routes/vendor.js` — all `/vendor/*` routes
- `db/vendors.js` — vendor + vendor_farms table creation
- `server.js` — register both route files

---

## Verification (Vendor Dashboard)

| Check | Action | Expected |
|-------|--------|----------|
| Vendor login | POST `/vendor/auth/login` | JWT with `role: vendor`, scoped farms |
| Farm isolation | Vendor A lists farms | Only their contracted farms, not Vendor B's |
| Motor toggle | POST `/vendor/motors/:id/toggle` | MQTT cmd published, motor responds |
| Energy chart | GET `/vendor/energy` | Aggregated kWh across all vendor farms |
| Schedule view | Open `/vendor/schedule` | Gantt shows upcoming irrigation slots |
| Alert ack | Ack alert from vendor dashboard | Alert marked ack in DB, mobile app reflects it |
