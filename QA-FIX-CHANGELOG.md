# AGRINET QA Fix Changelog

> Fix date: 2026-04-05
> Related: [QA-BUG-REPORT.md](QA-BUG-REPORT.md)

---

## Server Fixes (`sasyamithra-server/`)

### adapters/tsdb.js — S1 + S2

**S1: InfluxDB guard**
```javascript
const INFLUX_ENABLED = !!config.INFLUX_TOKEN;
// All exports check INFLUX_ENABLED before calling InfluxDB client
// writeTelemetry(), queryHistory(), flushWrites() return early if disabled
```

**S2: Flux query sanitization**
```javascript
function sanitizeFluxParam(str) {
  if (typeof str !== 'string') return '';
  return str.replace(/[^a-zA-Z0-9_\-]/g, '');
}
// Applied to deviceId, userId, range before Flux interpolation
```

### adapters/db.js — S3 + S6 + S7

**S3: Column name whitelist for updateUserProfile**
```javascript
const PROFILE_COLUMNS = new Set([
  'name', 'phone', 'district', 'mandal', 'village', 'farm', 'crop', 'acreage'
]);
// Unknown columns are silently skipped — no SQL injection possible
```

**S6: getMqttClientCount + setMqttClientCount**
```javascript
let _mqttClientCount = 0;
export const setMqttClientCount = (n) => { _mqttClientCount = n; };
export const getMqttClientCount = () => _mqttClientCount;
// Infra route now returns actual count instead of always 0
```

**S7: getStationRaw export**
```javascript
export const getStationRaw = (id) =>
  db.prepare('SELECT * FROM stations WHERE id = ?').get(id);
```

### routes/auth.js — S4

**Registration now persists profile fields:**
```javascript
db.createUser({ id, name, email, phone, password_hash, role, plan, mqtt_username });

// NEW: Save profile fields to database
const profileData = { district, mandal, village, farm, crop, acreage };
const hasProfile = Object.values(profileData).some(v => v !== undefined && v !== null);
if (hasProfile) {
  db.updateUserProfile(id, profileData);
}

const user = db.getUserById(id); // now returns full profile from DB
```

### routes/vendor.js — S5

**Null guard before access_level checks (2 locations):**
```javascript
// Motor toggle — line 106
const access = db.getVendorFarmAccess(request.vendor.vendorId, motorRow.farm_id);
if (!access) return reply.code(403).send({ error: 'Farm access revoked' });

// Valve toggle — line 134
const access = db.getVendorFarmAccess(request.vendor.vendorId, valveRow.farm_id);
if (!access) return reply.code(403).send({ error: 'Farm access revoked' });
```

---

## Dashboard Fixes (`agrinet-admin/`)

### VendorFarmsPage.tsx — D1

**Fixed navigation target:**
```tsx
// BEFORE: onClick={() => navigate(`/vendor`)}
// AFTER:
onClick={() => navigate(`/vendor/farms/${f.id}`)}
```

### DevicesPage.tsx — D7

**Standardized pagination page size:**
```tsx
// BEFORE: disabled={devices.length < 100}
// AFTER:
disabled={devices.length < 50}
```

---

## Mobile App Fixes (`sasyamithra-app/`)

### Toast messages — M1

**Replaced all Firebase references across 4 files:**

| File | Before | After |
|------|--------|-------|
| ValvesPage.tsx | `-> Firebase` (5 occurrences) | `-> server` |
| MotorsPage.tsx | `-> Firebase`, `from Firebase` (4 occurrences) | `-> server`, `from server` |
| WeatherPage.tsx | `-> Firebase` (1 occurrence) | `-> server` |
| AlertsPage.tsx | `-> Firebase` (1 occurrence) | `-> server` |

### vite-env.d.ts — M2

**Replaced Firebase env types with Pi4 server types:**
```typescript
// BEFORE: VITE_FIREBASE_API_KEY, VITE_FIREBASE_AUTH_DOMAIN, etc. (7 Firebase vars)
// AFTER:
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_MQTT_WS_URL: string;
  readonly VITE_MQTT_USERNAME: string;
  readonly VITE_MQTT_PASSWORD: string;
  readonly VITE_APP_NAME: string;
  readonly VITE_SENSOR_POLL_INTERVAL: string;
}
```

### AuthPage.tsx — M3

**Replaced Firebase project ID display:**
```tsx
// BEFORE: Firebase Project: {import.meta.env.VITE_FIREBASE_PROJECT_ID}
// AFTER:
Server: {import.meta.env.VITE_API_BASE_URL || 'Pi4 Self-Hosted'}
```

### .env — M5

**Set default MQTT password:**
```env
# BEFORE: VITE_MQTT_PASSWORD=
# AFTER:
# IMPORTANT: Set VITE_MQTT_PASSWORD to your Mosquitto password_file entry
VITE_MQTT_PASSWORD=changeme
```

### AuthContext.tsx — M8

**Added warning log for localStorage fallback:**
```typescript
} catch {
  console.warn('Capacitor Preferences unavailable — falling back to localStorage');
  localStorage.setItem('sasy_refresh_token', token);
}
```

---

## Verification Results

| Check | Result |
|-------|--------|
| `npm run build` (agrinet-admin) | 969 modules, zero errors |
| `npx vite build` (sasyamithra-app) | Build passes, PWA generated |
| Grep "Firebase" in app UI code | Zero matches |
| Grep SQL interpolation in tsdb.js | All params sanitized |
| Server startup (no InfluxDB) | Graceful fallback, no crash |
