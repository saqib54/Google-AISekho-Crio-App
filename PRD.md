# CIRO — Product Requirements Document (PRD)
**Crisis Intelligence & Response Orchestrator**
**Version:** 1.0 | **Date:** May 2026 | **Challenge:** Google Antigravity Hackathon — Challenge 3

---

## 1. Executive Summary

CIRO is an AI-powered crisis detection and response orchestration platform purpose-built for Pakistani urban centers. It ingests multi-source emergency signals (social reports, weather alerts, traffic feeds), runs a 4-agent AI pipeline powered by **Gemini 2.0 Flash**, and produces a prioritized response plan — all in real time. The primary deliverable is a React Native (Expo) mobile application backed by a Node.js/Express API.

---

## 2. Problem Statement

Pakistani cities — particularly Islamabad and Rawalpindi — face recurring crises: urban flooding, power grid failures, road accidents, and industrial fires. Emergency response today is:

- **Fragmented**: No single system aggregates cross-domain signals.
- **Slow**: Manual triage means delayed decisions during critical minutes.
- **Opaque**: Responders lack AI-assisted situational awareness or resource routing.

CIRO addresses all three gaps with an autonomous, traceable AI orchestration layer.

---

## 3. Product Goals

| Priority | Goal |
|----------|------|
| P0 | Detect active crises from heterogeneous signals using Gemini AI |
| P0 | Generate and simulate a prioritized action plan automatically |
| P1 | Provide a real-time mobile dashboard for field and command-center users |
| P1 | Support Roman Urdu, Urdu, and English signal inputs |
| P2 | Visualize crisis geography and resource positions on a city map |
| P2 | Enable offline/demo mode with full mock fallback |

---

## 4. Target Users

| Persona | Description | Primary Need |
|---------|-------------|--------------|
| **Emergency Coordinator** | NDMA / city authority operator | Real-time crisis overview + action dispatch |
| **First Responder** | Police, rescue, paramedic | Field instructions, route, resource status |
| **Citizen Reporter** | Urban resident | Report an emergency in local language |
| **Hackathon Judge / Demo User** | Evaluator | One-click scenario execution + full trace |

---

## 5. Scope

### 5.1 In Scope (v1.0)
- Multi-source signal ingestion (social, weather, traffic, custom text)
- 4-agent AI pipeline: Ingest → Detect → Plan → Simulate
- Gemini 2.0 Flash integration for crisis detection & action planning
- SQLite persistence for events, signals, actions, logs
- Mobile app with 9 screens (Splash, Home, Report, Map, Dashboard, CrisisDetail, Simulation, AgentTrace, Comparison)
- Demo scenarios: Urban Flooding, Power Outage, Road Accident
- WebSocket live-log streaming
- SSE signal stream endpoint
- Mock/offline fallback mode

### 5.2 Out of Scope (v1.0)
- Live GPS tracking of responders
- Real-time social media API integration (Twitter/X, Facebook)
- Push notifications to citizens
- Multi-city deployment at scale
- User authentication / role-based access control
- Payment or resource management modules

---

## 6. Functional Requirements

### 6.1 Signal Ingestion (FR-01)
- **FR-01.1** System SHALL accept free-text crisis reports in English, Urdu, and Roman Urdu.
- **FR-01.2** System SHALL ingest structured signal arrays (id, text, source, location, lat, lng).
- **FR-01.3** System SHALL auto-detect signal language and tag crisis type (flooding, fire, power_outage, road_blockage, accident).
- **FR-01.4** System SHALL load mock signals from `signals.json` when no custom input is provided.
- **FR-01.5** System SHALL expose a 10-second interval SSE stream (`/api/signals/stream`).

### 6.2 Crisis Detection (FR-02)
- **FR-02.1** System SHALL send processed signals to Gemini 2.0 Flash for crisis analysis.
- **FR-02.2** Gemini SHALL return a structured JSON: `crisis_detected`, `crisis_type`, `severity` (CRITICAL/HIGH/MEDIUM/LOW), `confidence` (0–100), `affected_area`, GPS coords, `summary`.
- **FR-02.3** System SHALL fall back to a dynamic mock response if the Gemini API key is absent or invalid.
- **FR-02.4** System SHALL persist detected crises to the `crisis_events` SQLite table.
- **FR-02.5** If no crisis is detected, the pipeline SHALL terminate and return `status: no_crisis`.

### 6.3 Action Planning (FR-03)
- **FR-03.1** System SHALL generate a prioritized action plan using Gemini based on crisis type + available resources.
- **FR-03.2** Each action SHALL contain: id, type (dispatch_rescue / reroute_traffic / notify_hospital / send_alert), description, priority (1–5), assigned_resource_id, estimated_impact.
- **FR-03.3** System SHALL load available resources from `providers.json` (rescue teams, hospitals, alternate routes).

### 6.4 Execution Simulation (FR-04)
- **FR-04.1** System SHALL simulate each action with a 0.5–2s realistic delay.
- **FR-04.2** Each simulation SHALL produce a `before` and `after` state object.
- **FR-04.3** System SHALL maintain a 95% simulated success rate per action.
- **FR-04.4** Simulation results SHALL be persisted in `actions` and `execution_logs` tables.

### 6.5 Mobile App (FR-05)
- **FR-05.1** App SHALL display live signal feed on Home screen with pull-to-refresh.
- **FR-05.2** App SHALL support free-text crisis reporting via Input screen.
- **FR-05.3** App SHALL render 3 pre-built demo scenarios (G-10 Flood, F-7 Outage, Murree Accident).
- **FR-05.4** App SHALL show a city map with crisis zone markers.
- **FR-05.5** App SHALL display full agent reasoning trace with color-coded log levels.
- **FR-05.6** App SHALL visualize Before/After state comparison for each action.
- **FR-05.7** App SHALL support Dark and Light themes via ThemeContext.
- **FR-05.8** App SHALL provide haptic feedback on refresh and key interactions.

### 6.6 Dashboard (FR-06)
- **FR-06.1** Dashboard SHALL display: Active Crises count, Signals Today, Actions Done, Avg Response Time.
- **FR-06.2** Dashboard SHALL poll `/api/crisis/stats` for live statistics.

---

## 7. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| **Performance** | Full pipeline (4 agents) SHALL complete in < 15 seconds with live Gemini |
| **Availability** | Mock mode SHALL guarantee 100% offline demo availability |
| **Accuracy** | Gemini crisis detection confidence SHOULD be ≥ 85% on test scenarios |
| **Responsiveness** | Mobile UI MUST be usable on screens 375px–428px wide |
| **Language** | System MUST correctly tag Roman Urdu inputs containing Urdu keywords |
| **Logging** | All agent steps SHALL be logged with timestamp, level, and agent ID |
| **Security** | API keys SHALL be stored in `.env` and never committed to source control |
| **Portability** | App SHALL run on iOS (Expo Go), Android (Expo Go), and Web (Expo Web) |

---

## 8. Demo Scenarios

| Scenario | Input | Expected Crisis | Actions |
|----------|-------|-----------------|---------|
| 🌊 Urban Flooding — G-10 | "Massive flooding reported in G-10..." | CRITICAL flooding | 4: rescue + reroute + alert + hospital |
| ⚡ Power Outage — F-7 | "Main power grid failure in F-7..." | HIGH power_outage | 3: restore + notify + backup |
| 🚗 Road Accident — Murree Road | "Multi-vehicle pileup on Murree Road..." | HIGH accident | 3: rescue + reroute + hospital |

---

## 9. Success Metrics

| Metric | Target |
|--------|--------|
| Demo pipeline end-to-end success rate | 100% |
| Agent trace visibility in UI | All 4 agents visible |
| Scenario auto-fill → result time | < 20 seconds |
| Judge comprehension score | ≥ 4/5 on clarity |
| Roman Urdu detection accuracy | ≥ 80% on test inputs |

---

## 10. Constraints & Assumptions

- **Gemini API Key**: Must be provided via `.env`. Without it, mock mode activates automatically.
- **Map**: Google Maps JS API requires a separate valid API key; v1.0 uses a grid-based mock visualization.
- **Real-time Signals**: Not live-streamed in v1.0; signals loaded from mock JSON. Production upgrade: WebSocket push.
- **Pakistani Data**: All coordinates and place names are real Islamabad/Rawalpindi locations.
- **SQLite**: Single-file local database suitable for demo; production would upgrade to PostgreSQL or Firestore.

---

## 11. Roadmap (Post-Hackathon)

| Phase | Feature |
|-------|---------|
| v1.1 | Firebase Authentication + role-based access (Coordinator, Responder, Citizen) |
| v1.2 | Live social media signal ingestion (Twitter API, Facebook API) |
| v1.3 | Real Google Maps integration with resource marker overlays |
| v1.4 | Push notifications to registered citizens in affected zones |
| v2.0 | Multi-city deployment (Karachi, Lahore, Peshawar) + cloud database |
