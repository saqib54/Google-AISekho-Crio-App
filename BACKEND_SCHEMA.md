# CIRO — Backend Schema Document
**Crisis Intelligence & Response Orchestrator**
**Version:** 1.0 | **Date:** May 2026

---

## 1. Database Overview

- **Engine:** SQLite (via `better-sqlite3`)
- **File:** `ciro-backend/db/ciro.db` (auto-created on first run)
- **Init:** `db/index.js` runs `schema.sql` on startup
- **Access Pattern:** Synchronous reads, synchronous writes

---

## 2. Table Definitions

### 2.1 `crisis_events`

Stores every detected crisis event from the pipeline.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PRIMARY KEY | Format: `crisis_{timestamp}` e.g. `crisis_1748601234567` |
| `type` | TEXT | NOT NULL | Crisis type: `flooding`, `fire`, `accident`, `power_outage` |
| `severity` | TEXT | NOT NULL | `CRITICAL`, `HIGH`, `MEDIUM`, `LOW` |
| `confidence` | INTEGER | — | AI confidence score 0–100 |
| `lat` | REAL | — | GPS latitude of crisis epicenter |
| `lng` | REAL | — | GPS longitude of crisis epicenter |
| `area` | TEXT | — | Human-readable affected area e.g. `G-10, Islamabad` |
| `summary` | TEXT | — | Gemini-generated plain English summary |
| `status` | TEXT | — | `ACTIVE`, `RESOLVED`, `ESCALATED` |
| `created_at` | DATETIME | DEFAULT CURRENT_TIMESTAMP | Auto-set on insert |

**DDL:**
```sql
CREATE TABLE IF NOT EXISTS crisis_events (
  id         TEXT PRIMARY KEY,
  type       TEXT,
  severity   TEXT,
  confidence INTEGER,
  lat        REAL,
  lng        REAL,
  area       TEXT,
  summary    TEXT,
  status     TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Sample Row:**
```json
{
  "id": "crisis_1748601234567",
  "type": "flooding",
  "severity": "CRITICAL",
  "confidence": 94,
  "lat": 33.6938,
  "lng": 72.9862,
  "area": "G-10, Islamabad",
  "summary": "Severe urban flooding detected in G-10 sector. Multiple residents stranded. Immediate rescue required.",
  "status": "ACTIVE",
  "created_at": "2026-05-30 06:00:00"
}
```

---

### 2.2 `signals`

Stores individual ingested signals. (Populated in extended mode; currently signals are in-memory in v1.0.)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PRIMARY KEY | Format: `sig_{source}_{timestamp}` |
| `text` | TEXT | — | Raw signal text (English/Urdu/Roman Urdu) |
| `source` | TEXT | — | `social`, `weather`, `traffic`, `custom_input` |
| `location` | TEXT | — | Location name string |
| `lat` | REAL | — | GPS latitude |
| `lng` | REAL | — | GPS longitude |
| `crisis_id` | TEXT | FK → crisis_events.id | Associated crisis (nullable) |
| `timestamp` | DATETIME | DEFAULT CURRENT_TIMESTAMP | Signal received time |

**DDL:**
```sql
CREATE TABLE IF NOT EXISTS signals (
  id        TEXT PRIMARY KEY,
  text      TEXT,
  source    TEXT,
  location  TEXT,
  lat       REAL,
  lng       REAL,
  crisis_id TEXT,
  timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

### 2.3 `actions`

Stores each action generated and simulated by the pipeline.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PRIMARY KEY | Format: `{crisisId}_{actionId}_{timestamp}` |
| `crisis_id` | TEXT | FK → crisis_events.id | Parent crisis |
| `type` | TEXT | — | `dispatch_rescue`, `reroute_traffic`, `notify_hospital`, `send_alert` |
| `description` | TEXT | — | Human-readable action description |
| `priority` | INTEGER | — | 1 = highest priority |
| `status` | TEXT | — | `COMPLETED`, `FAILED`, `PENDING` |
| `executed_at` | DATETIME | — | ISO timestamp of execution |

**DDL:**
```sql
CREATE TABLE IF NOT EXISTS actions (
  id          TEXT PRIMARY KEY,
  crisis_id   TEXT,
  type        TEXT,
  description TEXT,
  priority    INTEGER,
  status      TEXT,
  executed_at DATETIME
);
```

**Sample Row:**
```json
{
  "id": "crisis_1748601234567_act_1_1748601240000",
  "crisis_id": "crisis_1748601234567",
  "type": "dispatch_rescue",
  "description": "Deploy Rescue 1122 team to G-10 sector immediately",
  "priority": 1,
  "status": "COMPLETED",
  "executed_at": "2026-05-30T06:00:08.000Z"
}
```

---

### 2.4 `execution_logs`

Stores per-step execution log entries from Agent 4.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PRIMARY KEY | Format: `log_{timestamp}_{random}` |
| `action_id` | TEXT | FK → actions.id | Associated action |
| `agent_name` | TEXT | — | `AGENT_4` |
| `message` | TEXT | — | Log message |
| `level` | TEXT | — | `SUCCESS`, `ERROR`, `INFO`, `WARN` |
| `timestamp` | DATETIME | DEFAULT CURRENT_TIMESTAMP | Auto-set |

**DDL:**
```sql
CREATE TABLE IF NOT EXISTS execution_logs (
  id         TEXT PRIMARY KEY,
  action_id  TEXT,
  agent_name TEXT,
  message    TEXT,
  level      TEXT,
  timestamp  DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

### 2.5 `resources`

Resource catalog (rescue teams, hospitals, routes). Currently read from `providers.json`; stored in DB for future querying.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PRIMARY KEY | e.g. `r1`, `r2`, `h1` |
| `name` | TEXT | — | e.g. `Rescue 1122 Islamabad` |
| `type` | TEXT | — | `rescue_team`, `hospital`, `alternate_route` |
| `status` | TEXT | — | `AVAILABLE`, `DEPLOYED`, `UNAVAILABLE` |
| `lat` | REAL | — | Resource home base latitude |
| `lng` | REAL | — | Resource home base longitude |
| `last_updated` | DATETIME | DEFAULT CURRENT_TIMESTAMP | Last status update |

**DDL:**
```sql
CREATE TABLE IF NOT EXISTS resources (
  id           TEXT PRIMARY KEY,
  name         TEXT,
  type         TEXT,
  status       TEXT,
  lat          REAL,
  lng          REAL,
  last_updated DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## 3. Entity Relationship Diagram

```
crisis_events (1) ──────< signals (many)
     │
     │ (1)
     │
     └──< actions (many)
               │
               │ (1)
               │
               └──< execution_logs (many)
```

---

## 4. Mock Data Schemas

### 4.1 `signals.json`

Located at `ciro-backend/mock-data/signals.json`

**Top-level structure:**
```json
{
  "social_posts": [ ... ],
  "weather_alerts": [ ... ],
  "traffic_reports": [ ... ]
}
```

**Social Post Schema:**
```json
{
  "id": "social_001",
  "text": "G-10 mein pani bhar gaya hai, gaariyan phans gayi hain",
  "source": "social",
  "platform": "Twitter",
  "username": "@IslamabadAlert",
  "location": "G-10, Islamabad",
  "lat": 33.6938,
  "lng": 72.9862,
  "timestamp": "2026-05-30T05:45:00Z",
  "language": "roman_urdu"
}
```

**Weather Alert Schema:**
```json
{
  "id": "weather_001",
  "message": "Heavy rainfall warning for Islamabad. Expected 80mm in next 3 hours.",
  "source": "weather",
  "agency": "PMD",
  "severity": "HIGH",
  "affected_areas": ["G-10", "G-11", "F-7"],
  "lat": 33.7200,
  "lng": 73.0000,
  "timestamp": "2026-05-30T05:00:00Z"
}
```

**Traffic Report Schema:**
```json
{
  "id": "traffic_001",
  "text": "Murree Road completely blocked near Faizabad interchange",
  "source": "traffic",
  "road": "Murree Road",
  "location": "Faizabad, Rawalpindi",
  "congestion_level": "CRITICAL",
  "lat": 33.6200,
  "lng": 73.0700,
  "timestamp": "2026-05-30T05:50:00Z"
}
```

---

### 4.2 `providers.json`

Located at `ciro-backend/mock-data/providers.json`

**Top-level structure:**
```json
{
  "rescue_teams": [ ... ],
  "hospitals": [ ... ],
  "alternate_routes": [ ... ]
}
```

**Rescue Team Schema:**
```json
{
  "id": "r1",
  "name": "Rescue 1122 Islamabad",
  "type": "rescue_team",
  "status": "AVAILABLE",
  "vehicles": 4,
  "personnel": 12,
  "base_location": "G-9, Islamabad",
  "lat": 33.6800,
  "lng": 73.0100,
  "specializations": ["flood_rescue", "urban_search"]
}
```

**Hospital Schema:**
```json
{
  "id": "h1",
  "name": "PIMS Hospital",
  "type": "hospital",
  "status": "AVAILABLE",
  "beds_available": 45,
  "trauma_unit": true,
  "location": "G-8, Islamabad",
  "lat": 33.6900,
  "lng": 73.0300,
  "contact": "+92-51-9261170"
}
```

**Alternate Route Schema:**
```json
{
  "id": "route_1",
  "name": "Islamabad Expressway Bypass",
  "type": "alternate_route",
  "from": "G-10",
  "to": "Faizabad",
  "estimated_time": "12 mins",
  "status": "CLEAR",
  "waypoints": [
    { "lat": 33.6938, "lng": 72.9862 },
    { "lat": 33.6800, "lng": 73.0500 }
  ]
}
```

---

## 5. Gemini API Contracts

### 5.1 Crisis Detection Prompt → Response

**Model:** `gemini-2.0-flash-001`  
**Config:** `responseMimeType: "application/json"`

**Prompt Template:**
```
You are an expert crisis detection AI for Pakistani cities. 
Analyze these emergency signals carefully.
SIGNALS: {JSON.stringify(signals, null, 2)}
Respond with a JSON object ONLY:
{
  "crisis_detected": true,
  "crisis_type": "flooding|fire|accident|power_outage",
  "severity": "CRITICAL|HIGH|MEDIUM|LOW",
  "confidence": 94,
  "affected_area": "Area, City",
  "lat": 33.0,
  "lng": 73.0,
  "summary": "description",
  "key_signals_used": ["sig_id_1"]
}
```

**Response Schema:**
```json
{
  "crisis_detected": "boolean",
  "crisis_type": "string (enum)",
  "severity": "string (enum: CRITICAL|HIGH|MEDIUM|LOW)",
  "confidence": "integer (0-100)",
  "affected_area": "string",
  "lat": "number",
  "lng": "number",
  "summary": "string",
  "key_signals_used": ["array of signal IDs"]
}
```

---

### 5.2 Action Planning Prompt → Response

**Prompt Template:**
```
Generate emergency actions for: {JSON.stringify(crisis)}.
Resources: {JSON.stringify(resources)}.
Respond with JSON:
{
  "actions": [
    {
      "id": "act_1",
      "type": "dispatch_rescue|reroute_traffic|notify_hospital|send_alert",
      "description": "...",
      "priority": 1,
      "assigned_resource_id": "r1",
      "estimated_impact": "..."
    }
  ],
  "reasoning": "...",
  "estimated_resolution_time": "..."
}
```

**Response Schema:**
```json
{
  "actions": [
    {
      "id": "string",
      "type": "string (enum)",
      "description": "string",
      "priority": "integer (1=highest)",
      "assigned_resource_id": "string | null",
      "estimated_impact": "string"
    }
  ],
  "reasoning": "string",
  "estimated_resolution_time": "string"
}
```

---

## 6. In-Memory / Runtime Data Shapes

### 6.1 Processed Signal (Agent 1 Output)
```json
{
  "id": "social_001",
  "text": "G-10 mein pani bhar gaya hai",
  "source": "social",
  "location": "G-10, Islamabad",
  "lat": 33.6938,
  "lng": 72.9862,
  "timestamp": "2026-05-30T05:45:00Z",
  "language": "urdu/roman_urdu",
  "crisis_type_tag": "flooding",
  "severity_score": 7,
  "processed_at": "2026-05-30T06:00:00.000Z"
}
```

### 6.2 Execution Result (Agent 4 Output)
```json
{
  "action_id": "act_1",
  "status": "COMPLETED",
  "before": {
    "status": "STANDBY",
    "location": "Base"
  },
  "after": {
    "status": "EN_ROUTE",
    "eta": "8 mins",
    "ticket_id": "EMG-2025-743"
  },
  "timestamp": "2026-05-30T06:00:08.000Z"
}
```

### 6.3 Agent Trace Log Entry
```json
{
  "agentId": "2",
  "level": "DETECT",
  "message": "Crisis detected (confidence: 94%)",
  "timestamp": "06:00:05"
}
```

### 6.4 Full Pipeline API Response
```json
{
  "status": "success",
  "crisis": {
    "id": "crisis_1748601234567",
    "crisis_detected": true,
    "crisis_type": "flooding",
    "severity": "CRITICAL",
    "confidence": 94,
    "affected_area": "G-10, Islamabad",
    "lat": 33.6938,
    "lng": 72.9862,
    "summary": "..."
  },
  "actions": [ { "id": "act_1", "type": "dispatch_rescue", ... } ],
  "execution_logs": [ { "action_id": "act_1", "status": "COMPLETED", ... } ],
  "agent_traces": [ { "agentId": "1", "level": "INFO", "message": "...", "timestamp": "..." } ]
}
```

---

## 7. Language Detection Logic

**Keyword Sets (Agent 1):**

| Set | Keywords |
|-----|----------|
| Urdu/Roman Urdu | `pani`, `phans`, `bijli`, `khamba`, `khatarnak`, `gaya`, `hai`, `bohot`, `mein`, `ke` |
| English | `flood`, `traffic`, `blocked`, `smoke`, `fire`, `heavy`, `warning` |

**Algorithm:** Count matches in each set → higher score wins. Default: `english`.

**Crisis Type Tags:**

| Condition | Tag |
|-----------|-----|
| `pani` OR `flood` OR `rain` | `flooding` |
| `bijli` OR `power` | `power_outage` |
| `smoke` OR `fire` | `fire` |
| `traffic` OR `blocked` | `road_blockage` |
| None matched | `unknown` |

**Severity Override:**
- `signal.severity === 'CRITICAL'` OR `signal.congestion_level === 'CRITICAL'` → `severity_score = 9`
- `signal.severity === 'HIGH'` OR `signal.congestion_level === 'HIGH'` → `severity_score = 7`

---

## 8. Simulation State Machine (Agent 4)

| Action Type | Before State | After State (Success) | After State (Fail) |
|-------------|-------------|----------------------|-------------------|
| `dispatch_rescue` | `{ status: "STANDBY", location: "Base" }` | `{ status: "EN_ROUTE", eta: "8 mins", ticket_id: "EMG-2025-xxx" }` | `{ status: "DELAYED" }` |
| `reroute_traffic` | `{ status: "BLOCKED", vehicles_affected: N }` | `{ status: "ACTIVE", diversion_time: "3 mins", route: "Alternate" }` | `{ status: "BLOCKED" }` |
| `notify_hospital` | `{ beds_available: 45 }` | `{ beds_available: 35, trauma_team: "ACTIVATED" }` | `{ beds_available: 45 }` |
| `send_alert` (default) | `{ status: "PENDING" }` | `{ status: "COMPLETED" }` | `{ status: "FAILED" }` |

**Success Rate:** 95% (`Math.random() > 0.05`)  
**Execution Delay:** `Math.random() * 1500 + 500` ms per action (0.5s–2s)
