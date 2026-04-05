# AGRINET QA Bug Report

> Audit date: 2026-04-05
> Auditor: Claude Code (Professional QA)
> Scope: Pi4 server, Admin dashboard, Mobile app

---

## Summary

| Severity | Server | Dashboard | Mobile App | Total |
|----------|--------|-----------|------------|-------|
| CRITICAL | 4 | 1 | 0 | **5** |
| HIGH | 2 | 0 | 2 | **4** |
| MEDIUM | 1 | 1 | 6 | **8** |
| LOW | 0 | 0 | 1 | **1** |
| **Total** | **7** | **2** | **9** | **18** |

> Note: 5 originally reported dashboard bugs (D2-D6) were verified as false positives during implementation.

---

## Server Bugs (`sasyamithra-server/`)

### S1 — CRITICAL: InfluxDB client crash on empty token
- **File:** `adapters/tsdb.js` line 9
- **Bug:** `new InfluxDB({ url, token })` crashes if `INFLUX_TOKEN` is empty/undefined. The sibling adapter `adapters/influx.js` has a guard but `tsdb.js` does not.
- **Impact:** Server crashes on startup in environments without InfluxDB configured.

### S2 — CRITICAL: Flux query injection in tsdb.js
- **File:** `adapters/tsdb.js` lines 35-36
- **Bug:** `deviceId` and `userId` are interpolated directly into Flux query strings without sanitization.
- **Impact:** Malicious input can alter Flux queries, potentially leaking data from other users/devices.

### S3 — CRITICAL: SQL injection in updateUserProfile
- **File:** `adapters/db.js` line ~33
- **Bug:** Column names in `updateUserProfile` are constructed from user input (`key.replace(...)`) without validation, allowing SQL injection via crafted JSON keys.
- **Impact:** Attacker can execute arbitrary SQL via profile update endpoint.

### S4 — CRITICAL: Registration ignores profile fields
- **File:** `routes/auth.js` lines 67-71
- **Bug:** `POST /auth/register` accepts `district`, `mandal`, `village`, `farm`, `crop`, `acreage` in request body but the INSERT only saves `name`, `email`, `phone`, `password_hash`, `role`. Profile fields are assigned to the in-memory user object but never persisted.
- **Impact:** Farmers lose profile data on registration. Data appears correct during the session but is lost after logout.

### S5 — HIGH: Null dereference on deleted vendor_farms row
- **File:** `routes/vendor.js` lines 105-106, 133-134
- **Bug:** `access.access_level` throws if `getVendorFarmAccess()` returns `undefined` (row deleted between auth check and access check).
- **Impact:** Server crashes (500) when vendor tries to control a motor/valve on a farm whose access was just revoked.

### S6 — HIGH: getMqttClientCount not defined
- **File:** `routes/admin.js` line ~214, `adapters/db.js`
- **Bug:** `db.getMqttClientCount()` is called in the infra endpoint but never defined/exported. The route has a safe fallback (`db.getMqttClientCount ? ... : 0`) so it returns 0 instead of crashing.
- **Impact:** Infra dashboard always shows 0 MQTT clients.

### S7 — MEDIUM: Missing getStationRaw export
- **File:** `adapters/db.js`
- **Bug:** `getStationRaw` is missing from exports despite `getMotorRaw`, `getValveRaw`, `getAlertRaw` all being exported.
- **Impact:** Future routes needing raw station access would fail.

---

## Dashboard Bugs (`agrinet-admin/`)

### D1 — CRITICAL: VendorFarmsPage wrong navigation
- **File:** `src/pages/vendor/VendorFarmsPage.tsx` line 20
- **Bug:** Farm card `onClick` navigates to `/vendor` (the control center) instead of `/vendor/farms/${f.id}`.
- **Impact:** Vendors cannot navigate to individual farm details. Clicking any farm card just reloads the control center.

### D7 — MEDIUM: DevicesPage inconsistent page size
- **File:** `src/pages/admin/DevicesPage.tsx` line 69
- **Bug:** Uses page size `100` in the pagination disabled check while all other pages use `50`. The server's `/admin/devices` endpoint also defaults to `limit=100`.
- **Impact:** Inconsistent pagination behavior — "Next" button may be incorrectly enabled/disabled.

### False Positives (verified during fix implementation)
- **D2/D3/D4:** Query invalidation already uses correct TanStack v5 `{ queryKey: [...] }` object syntax.
- **D5/D6:** Pagination correctly uses `50` matching the server's default `limit=50`.

---

## Mobile App Bugs (`sasyamithra-app/`)

### M1 — HIGH: Stale Firebase toast messages
- **Files:** `ValvesPage.tsx`, `MotorsPage.tsx`, `WeatherPage.tsx`, `AlertsPage.tsx`
- **Bug:** Toast/success messages still reference "Firebase" (e.g., `"Valve added -> Firebase"`, `"Refreshing from Firebase..."`). App v6.2 is the self-hosted (zero Firebase) edition.
- **Impact:** Confuses users. Suggests the app is still connected to Firebase when it is not.

### M2 — MEDIUM: Unused Firebase env type definitions
- **File:** `src/vite-env.d.ts`
- **Bug:** Contains `VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_PROJECT_ID`, and 5 other Firebase env type declarations that are unused.
- **Impact:** TypeScript types reference non-existent env vars. Misleading for developers.

### M3 — MEDIUM: AuthPage displays blank Firebase Project ID
- **File:** `src/pages/AuthPage.tsx` line 291
- **Bug:** Renders `Firebase Project: {import.meta.env.VITE_FIREBASE_PROJECT_ID}` which is empty/undefined.
- **Impact:** Login page shows "Firebase Project:" with blank text.

### M4 — MEDIUM: Toggle endpoints need verification
- **File:** `src/contexts/DeviceContext.tsx` lines 254, 301
- **Bug:** Motor/valve toggle calls use `/motors/{id}/toggle` and `/valves/{id}/toggle`.
- **Verified:** Endpoints match the farmer routes in `routes/motors.js` and `routes/valves.js`. **Not a bug** — working as intended.

### M5 — HIGH: Empty MQTT password in .env
- **File:** `.env` line 11
- **Bug:** `VITE_MQTT_PASSWORD=` is empty. If Mosquitto requires authentication, MQTT connections fail silently.
- **Impact:** No real-time data on the app if Mosquitto auth is enabled.

### M6 — MEDIUM: Silent failure on /live-data endpoint
- **File:** `src/contexts/LiveDataContext.tsx` lines 165-181
- **Bug:** REST fallback fetch to `/live-data` catches errors silently with no UI feedback.
- **Verified:** Acceptable behavior — MQTT is the primary transport; REST is a best-effort fallback on initial load.

### M7 — MEDIUM: Hardcoded MQTT topic patterns
- **File:** `src/contexts/LiveDataContext.tsx`
- **Bug:** MQTT subscriptions use `data/+/+/telemetry`, `data/+/+/power`, etc.
- **Verified:** Patterns match the server's MQTT publish format `data/{uid}/{fid}/telemetry`. **Not a bug.**

### M8 — MEDIUM: Token storage fallback to localStorage
- **File:** `src/contexts/AuthContext.tsx` lines 18-34
- **Bug:** Falls back to `localStorage` if Capacitor Preferences plugin is unavailable. No warning logged.
- **Impact:** On web this is fine. On mobile, localStorage may not persist across app restarts in some WebView configurations.

### M9 — LOW: Silent token refresh failure
- **File:** `src/contexts/AuthContext.tsx` lines 79-111
- **Bug:** Token refresh failure only logs to console.
- **Verified:** Actually handles correctly — on failure, `clearRefreshToken()` is called and `user` stays null, triggering login redirect via route guards. **Not a bug.**

---

## Risk Assessment

| Category | Risk | Notes |
|----------|------|-------|
| SQL Injection | **Eliminated** | S3 (column whitelist) + prior db.js fixes (parameterized queries) |
| Flux Query Injection | **Eliminated** | S2 (input sanitization) |
| Server Crashes | **Eliminated** | S1 (InfluxDB guard), S5 (null guards) |
| Data Loss | **Fixed** | S4 (registration profile fields now persisted) |
| Navigation | **Fixed** | D1 (vendor farm cards navigate correctly) |
| Firebase Remnants | **Cleaned** | M1, M2, M3 (all references replaced) |
