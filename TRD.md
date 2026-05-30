# CIRO — Technical Requirements Document (TRD)
**Crisis Intelligence & Response Orchestrator**
**Version:** 1.0 | **Date:** May 2026

---

## 1. System Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                   CIRO MOBILE (React Native / Expo)              │
│  SplashScreen → HomeScreen → InputScreen → CrisisDetailScreen    │
│  MapScreen → DashboardScreen → SimulationScreen → AgentTrace    │
│                   ComparisonScreen                                │
│              apiService.js (axios + mock fallback)               │
└────────────────────────────┬─────────────────────────────────────┘
                             │ HTTP REST + WebSocket
                             │ localhost:3000 (dev)
┌────────────────────────────▼─────────────────────────────────────┐
│              CIRO BACKEND (Node.js 18 + Express 4)               │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐   │
│  │  REST Routes │  │  WebSocket   │  │  SSE Signal Stream    │   │
│  │  /api/*      │  │  (ws lib)    │  │  /api/signals/stream  │   │
│  └──────┬───────┘  └──────────────┘  └───────────────────────┘   │
│         │                                                         │
│  ┌──────▼──────────────────────────────────────────────────────┐  │
│  │                4-Agent Pipeline                              │  │
│  │  Agent1 (SignalIngestion) → Agent2 (CrisisDetection/Gemini) │  │
│  │  → Agent3 (ActionPlanning/Gemini) → Agent4 (ExecSimulation) │  │
│  └──────────────────────┬──────────────────────────────────────┘  │
│                         │                                         │
│  ┌──────────────────────▼───────────────┐  ┌──────────────────┐  │
│  │  SQLite DB (better-sqlite3)          │  │  Gemini 2.0 Flash│  │
│  │  crisis_events, signals, actions,    │  │  (Google AI SDK) │  │
│  │  execution_logs, resources           │  └──────────────────┘  │
│  └──────────────────────────────────────┘                         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Technology Stack

### 2.1 Frontend (Mobile)
| Component | Technology | Version |
|-----------|------------|---------|
| Framework | React Native (Expo) | SDK 52 |
| Language | JavaScript (ES2022) | — |
| Navigation | React Navigation v6 | Stack + BottomTab |
| HTTP Client | Axios | ^1.x |
| Icons | @expo/vector-icons (Ionicons) | — |
| Animations | React Native Animated API | built-in |
| Haptics | expo-haptics | — |
| Blur | expo-blur | — |
| Build | EAS Build (Expo Application Services) | — |
| Web Target | Expo Web (React DOM) | — |

### 2.2 Backend
| Component | Technology | Version |
|-----------|------------|---------|
| Runtime | Node.js | 18 LTS |
| Framework | Express | ^4.x |
| Database ORM | better-sqlite3 (sync) | ^8.x |
| WebSocket | ws | ^8.x |
| AI SDK | @google/generative-ai | ^0.x |
| Config | dotenv | ^16.x |
| CORS | cors | ^2.x |
| Container | Docker (Alpine Node 18) | — |
| Hosting | Google Cloud Run (optional) | — |

### 2.3 External Services
| Service | Purpose | Auth Method |
|---------|---------|-------------|
| Google Gemini 2.0 Flash | Crisis detection + action planning | `GEMINI_API_KEY` in `.env` |
| Google Maps JS API | Map visualization (mock in v1.0) | `MAPS_API_KEY` in `.env` |
| Firebase Hosting | Web app delivery (Expo Web) | `.firebaserc` project config |

---

## 3. Environment Configuration

### 3.1 Backend `.env`
```env
GEMINI_API_KEY=your_gemini_key_here
PORT=3000
MOCK_MODE=false          # Set true for full offline demo
```

### 3.2 Mobile `constants/`
```js
// API_BASE_URL resolved at runtime
const API_BASE_URL = process.env.EXPO_PUBLIC_API_URL || 'http://localhost:3000';
```

---

## 4. API Specification

Base URL: `http://localhost:3000/api`

---

### 4.1 `POST /crisis/analyze` — Full Pipeline

**Description:** Runs the complete 4-agent pipeline and returns a crisis object, actions, execution logs, and agent traces.

**Request Body:**
```json
{
  "custom_text": "G-10 mein pani bhar gaya hai",   // Option A: raw text
  "signals": [ ... ],                               // Option B: structured signals
  // (no body) → loads mock signals                 // Option C: mock
}
```

**Response 200 — Crisis Detected:**
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
    "summary": "Severe flooding detected in G-10 sector..."
  },
  "actions": [
    {
      "id": "act_1",
      "type": "dispatch_rescue",
      "description": "Deploy Rescue 1122 to G-10",
      "priority": 1,
      "assigned_resource_id": "r1",
      "estimated_impact": "Save lives in 10 mins"
    }
  ],
  "execution_logs": [
    {
      "action_id": "crisis_xxx_act_1_yyy",
      "status": "COMPLETED",
      "before": { "status": "STANDBY", "location": "Base" },
      "after": { "status": "EN_ROUTE", "eta": "8 mins", "ticket_id": "EMG-2025-743" },
      "timestamp": "2026-05-30T06:00:00.000Z"
    }
  ],
  "agent_traces": [
    { "agentId": "1", "level": "INFO", "message": "Starting ingestion pipeline...", "timestamp": "06:00:01" }
  ]
}
```

**Response 200 — No Crisis:**
```json
{ "status": "no_crisis", "agent_traces": [ ... ] }
```

**Response 500:**
```json
{ "error": "Internal server error message" }
```

---

### 4.2 `GET /crisis/stats` — Dashboard Statistics

**Response 200:**
```json
{
  "activeCrises": 2,
  "signalsToday": 42,
  "actionsDone": 128,
  "avgResponseTime": "12m"
}
```

---

### 4.3 `GET /crisis/signals` — Initial Signal Feed

**Response 200:** Array of processed signal objects (same schema as Agent 1 output).

---

### 4.4 `GET /crisis/history` — Crisis Event History

**Response 200:** Array of `crisis_events` rows, ordered by `created_at DESC`.

---

### 4.5 `GET /signals/mock` — Raw Mock Signals

**Response 200:** Array of raw signals from `signals.json`.

---

### 4.6 `GET /signals/stream` — SSE Signal Stream

**Protocol:** Server-Sent Events  
**Content-Type:** `text/event-stream`  
**Interval:** Every 10 seconds  
**Event Format:**
```
data: {"id":"sig_xxx","text":"...","source":"social","location":"G-10","lat":33.69,"lng":72.98}

```

---

### 4.7 `GET /scenarios/:name` — Pre-built Demo Scenarios

**Valid `:name` values:** `flood_g10`, `power_f7`, `accident_murree`

**Response 200:**
```json
{
  "text": "Massive flooding reported in sector G-10 after heavy rain.",
  "type": "Flood",
  "location": "G-10"
}
```

---

### 4.8 `GET /execution/logs` — Execution Log History

**Response 200:** Array of `execution_logs` rows, ordered by `timestamp DESC`.

---

## 5. WebSocket Protocol

**Endpoint:** `ws://localhost:3000`

**Server → Client (on pipeline run):**
```json
{ "agentId": "2", "level": "DETECT", "message": "Crisis detected (confidence: 94%)", "timestamp": "06:00:05" }
```

**Server → Client (periodic heartbeat, every 5s):**
```json
{ "agentId": 1, "level": "INFO", "message": "Scanning sector G-10 for anomalies...", "timestamp": "..." }
```

**Log Levels:**
| Level | Color | Meaning |
|-------|-------|---------|
| INFO | White | General system message |
| DETECT | Orange | Agent 2 detection event |
| ACTION | Green | Agent 3 planning event |
| SUCCESS | Green | Successful execution |
| WARN | Yellow | Warning / non-critical issue |
| ERROR | Red | Failure or exception |

---

## 6. Agent Pipeline — Technical Specification

### Agent 1 — SignalIngestionAgent
- **Type:** Synchronous, rule-based
- **Input:** Raw text OR structured signal array
- **Processing:**
  1. Language detection (Urdu keyword scoring vs English keyword scoring)
  2. Crisis type tagging via keyword matching
  3. Severity scoring (0–10)
- **Output:** Array of enriched signal objects with `language`, `crisis_type_tag`, `severity_score`, `processed_at`

### Agent 2 — CrisisDetectionAgent
- **Type:** Async, Gemini-powered
- **Input:** Enriched signal array from Agent 1
- **Prompt Template:** Structured JSON-constrained prompt sent to `gemini-2.0-flash-001` with `responseMimeType: "application/json"`
- **Output:** Crisis object OR `null`
- **Fallback:** `getDynamicMock()` — keyword-driven mock based on signal text

### Agent 3 — ActionPlanningAgent
- **Type:** Async, Gemini-powered
- **Input:** Crisis object + `providers.json` resource catalog
- **Prompt Template:** Crisis JSON + resources JSON → action array
- **Output:** `{ actions: [...], reasoning: "...", estimated_resolution_time: "..." }`
- **Fallback:** 2 hardcoded default actions (dispatch_rescue + send_alert)

### Agent 4 — ExecutionSimulationAgent
- **Type:** Async, rule-based simulation
- **Input:** Action plan from Agent 3 + crisis ID
- **Processing per action:**
  1. Random delay simulation (500–2000ms)
  2. 95% success probability
  3. Type-specific before/after state generation
  4. DB write to `actions` and `execution_logs`
- **Output:** Array of execution result objects

---

## 7. Mobile App Screen Architecture

```
NavigationRoot
├── SplashScreen (timed, no nav)
└── NavigationContainer
    ├── Stack: Main
    │   ├── BottomTab: Tabs
    │   │   ├── HomeScreen          [home-outline icon]
    │   │   ├── InputScreen         [add-circle-outline icon]
    │   │   ├── MapScreen           [map-outline icon]
    │   │   └── DashboardScreen     [stats-chart-outline icon]
    │   ├── CrisisDetailScreen      [pushed from Home/Input]
    │   ├── SimulationScreen        [pushed from CrisisDetail]
    │   ├── AgentTraceScreen        [pushed from CrisisDetail]
    │   └── ComparisonScreen        [pushed from Simulation]
    └── Stack: Crises (compat layer for InputScreen.navigate)
        ├── CrisisDetailScreen
        ├── SimulationScreen
        ├── AgentTraceScreen
        └── ComparisonScreen
```

---

## 8. Data Flow — Full Request Lifecycle

```
User types: "G-10 mein pani bhar gaya"
         ↓
InputScreen.js → POST /api/crisis/analyze { custom_text: "..." }
         ↓
SignalIngestionAgent.ingestCustomSignal()
  → detectLanguage() → "urdu/roman_urdu"
  → processSignal()  → { crisis_type_tag: "flooding", severity_score: 7 }
         ↓
CrisisDetectionAgent.detect(signals)
  → geminiService.analyzeCrisis(signals)
  → Gemini 2.0 Flash returns JSON crisis object
         ↓
DB INSERT INTO crisis_events(...)
         ↓
ActionPlanningAgent.plan(crisis)
  → geminiService.planActions(crisis, resources)
  → returns { actions: [...] }
         ↓
ExecutionSimulationAgent.simulate(plan, crisisId)
  → per-action: delay + state simulation + DB write
         ↓
Response JSON → InputScreen renders CrisisDetailScreen
         ↓
User navigates to SimulationScreen / AgentTraceScreen / ComparisonScreen
```

---

## 9. Docker Deployment

```dockerfile
# ciro-backend/Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

**Build & Run:**
```bash
docker build -t ciro-backend .
docker run -p 3000:3000 -e GEMINI_API_KEY=your_key ciro-backend
```

---

## 10. Firebase Hosting (Expo Web)

```json
// ciro-mobile/firebase.json
{
  "hosting": {
    "public": "dist",
    "rewrites": [{ "source": "**", "destination": "/index.html" }]
  }
}
```

**Deploy steps:**
```bash
npx expo export -p web      # Outputs to dist/
firebase deploy --only hosting
```

---

## 11. Security Considerations

| Risk | Mitigation |
|------|------------|
| Gemini API key exposure | Stored in `.env`, listed in `.gitignore` |
| CORS misconfiguration | `cors()` middleware with permissive config for demo; restrict in prod |
| SQL injection | SQLite parameterized queries via `?` placeholders |
| XSS in log messages | Logs rendered as text nodes in React Native (no innerHTML) |
| Rate limiting | Not implemented in v1.0; add `express-rate-limit` for production |

---

## 12. Performance Benchmarks (Target)

| Operation | Target Latency |
|-----------|---------------|
| `/api/signals/mock` | < 50ms |
| `/api/crisis/analyze` (mock mode) | < 5s |
| `/api/crisis/analyze` (Gemini live) | < 15s |
| Mobile screen render | < 100ms |
| WebSocket connection | < 500ms |
