# CIRO — Crisis Intelligence & Response Orchestrator
### Google Antigravity Hackathon — Challenge 3

```
   ██████╗██╗██████╗  ██████╗ 
  ██╔════╝██║██╔══██╗██╔═══██╗
  ██║     ██║██████╔╝██║   ██║
  ██║     ██║██╔══██╗██║   ██║
  ╚██████╗██║██║  ██║╚██████╔╝
   ╚═════╝╚═╝╚═╝  ╚═╝ ╚═════╝ 
  Crisis Intelligence & Response Orchestrator
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    CIRO MOBILE APP (Expo)                   │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐  ┌──────────┐  │
│  │  Home    │  │  Input   │  │  Crisis    │  │Dashboard │  │
│  │ Screen   │  │  Screen  │  │  Detail    │  │  Screen  │  │
│  └────┬─────┘  └────┬─────┘  └─────┬──────┘  └────┬─────┘  │
│       └─────────────┴──────────────┴───────────────┘        │
│                        API Service (axios)                   │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTP (localhost:3000)
┌───────────────────────────▼─────────────────────────────────┐
│                 CIRO BACKEND (Node.js + Express)             │
│  POST /api/crisis/analyze                                    │
│       ↓                                                      │
│  ┌──────────┐  ┌────────────┐  ┌───────────┐  ┌──────────┐  │
│  │ Agent 1  │→ │  Agent 2  │→ │  Agent 3  │→ │ Agent 4  │  │
│  │ Signal   │  │  Crisis   │  │  Action   │  │ Exec Sim │  │
│  │Ingestion │  │Detection  │  │ Planning  │  │          │  │
│  └──────────┘  └─────┬─────┘  └─────┬─────┘  └────┬─────┘  │
│                       │              │              │        │
│              ┌────────▼──────────────▼──────────────▼──────┐ │
│              │         Gemini 1.5 Pro API                  │ │
│              └─────────────────────────────────────────────┘ │
│                                                              │
│  ┌──────────────────────┐  ┌────────────────────────────┐   │
│  │  mock-data/           │  │  SQLite DB                 │   │
│  │  signals.json         │  │  crisis_events, signals,   │   │
│  │  providers.json       │  │  actions, execution_logs   │   │
│  └──────────────────────┘  └────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## How Antigravity (Gemini) Was Used

CIRO uses **Gemini 1.5 Pro** for two critical agent tasks:

1. **Agent 2 — Crisis Detection**: Receives processed multi-source signals and uses Gemini to detect if a crisis exists, classify the crisis type (flooding, fire, power outage, etc.), assign a severity level (LOW→CRITICAL), score confidence (0-100%), identify the affected area with coordinates, and summarize in plain English.

2. **Agent 3 — Action Planning**: Receives the crisis object + real available resources (rescue teams, hospitals, alternate routes) and uses Gemini to generate a prioritized response plan with specific action types, resource assignments, and estimated impact.

Both agents fall back to mock responses if the API key is missing, ensuring the app works in demo mode.

---

## APIs Used

| API | Purpose |
|-----|---------|
| **Google Gemini 1.5 Pro** | Crisis detection + action planning (Agents 2 & 3) |
| **Google Maps JS API** | (Mock mode) Map visualization of crisis zones |
| **SQLite** | Local storage of crisis events, signals, actions, logs |

---
## Screenshots

<img width="45%" src="https://github.com/user-attachments/assets/2a9e20b3-e8ad-449b-b737-02a7e78db1f0" /> <img width="45%" src="https://github.com/user-attachments/assets/d1556960-e71d-4a34-9d05-da3a1878e77e" />

<img width="45%" src="https://github.com/user-attachments/assets/829945cb-b825-42c9-88f9-684a39ffb389" /> <img width="45%" src="https://github.com/user-attachments/assets/19a4793c-cbe8-4a9a-b22b-22c5baac5e0c" />

<img width="45%" src="https://github.com/user-attachments/assets/60957e2b-9a86-4cbd-afd9-3df11aea6449" /> <img width="45%" src="https://github.com/user-attachments/assets/4624b8d8-0004-4734-8ef0-b13fc3bb3e13" />

<img width="45%" src="https://github.com/user-attachments/assets/459c80a6-a243-4953-aad5-959b4f98a5df" /> <img width="45%" src="https://github.com/user-attachments/assets/781bd76e-adff-4f58-9f79-2de2b2631768" />

<img width="45%" src="https://github.com/user-attachments/assets/801df4ad-405f-415d-a2a9-1272d72d6e3b" /> <img width="45%" src="https://github.com/user-attachments/assets/c199b1d8-93e6-4712-ab60-6eb4165a3e79" />

<img width="45%" src="https://github.com/user-attachments/assets/9e5812f8-f135-49f3-9028-65f7ede66969" /> <img width="45%" src="https://github.com/user-attachments/assets/2b79063b-cc68-4ec4-8c16-ba23e9d05bad" />

<img width="45%" src="https://github.com/user-attachments/assets/db563ac6-5edd-4806-9954-efcea5419a7d" />

---


## Folder Structure

```
antigravity-Ai Sekho project/
├── ciro-backend/
│   ├── agents/
│   │   ├── SignalIngestionAgent.js    ← Agent 1
│   │   ├── CrisisDetectionAgent.js   ← Agent 2 (Gemini)
│   │   ├── ActionPlanningAgent.js    ← Agent 3 (Gemini)
│   │   └── ExecutionSimulationAgent.js ← Agent 4
│   ├── db/
│   │   ├── index.js                  ← SQLite init
│   │   └── schema.sql                ← Table definitions
│   ├── mock-data/
│   │   ├── signals.json              ← Pakistani city signals
│   │   └── providers.json            ← Rescue, hospitals, routes
│   ├── routes/
│   │   └── api.js                    ← REST endpoints
│   ├── services/
│   │   ├── geminiService.js          ← Gemini API wrapper
│   │   └── scenarios.js              ← Demo scenarios
│   ├── .env                          ← API keys
│   ├── .env.example
│   └── index.js                      ← Express server
│
└── ciro-mobile/
    ├── screens/
    │   ├── HomeScreen.js             ← Live signal feed
    │   ├── InputScreen.js            ← Report + AI trigger
    │   ├── CrisisDetailScreen.js     ← Crisis intelligence
    │   ├── SimulationScreen.js       ← Execution theater
    │   ├── AgentTraceScreen.js       ← Agent reasoning log
    │   └── DashboardScreen.js        ← City overview
    ├── components/
    │   ├── theme.js
    │   ├── SeverityBadge.js
    │   ├── AgentLogItem.js
    │   ├── CrisisCard.js
    │   ├── ExecutionTimeline.js
    │   ├── StatCard.js
    │   └── LoadingAgents.js
    ├── services/
    │   ├── apiService.js             ← Backend connector + mock fallback
    │   └── scenarios.js              ← Demo scenario data
    └── App.js                        ← Navigation root
```

---

## How to Run Locally

### Prerequisites
- Node.js 18+
- Expo CLI (`npm install -g @expo/cli`)
- A Gemini API key from [Google AI Studio](https://aistudio.google.com)

### Step 1: Start the Backend
```bash
cd ciro-backend
# Edit .env and add your GEMINI_API_KEY
npm start
# Server runs on http://localhost:3000
```

### Step 2: Start the Mobile App
```bash
cd ciro-mobile
npx expo start
# Press 'w' for web, 'a' for Android emulator
```

### Step 3: Test the API (optional)
```bash
# Test mock signals
curl http://localhost:3000/api/signals/mock

# Run full analysis pipeline
curl -X POST http://localhost:3000/api/crisis/analyze \
  -H "Content-Type: application/json" \
  -d '{"custom_text": "G-10 mein pani bhar gaya hai, gaariyan phans gayi hain"}'
```

---

## Demo Scenario Instructions

In the app, go to **Report** tab → scroll down to **DEMO SCENARIOS**:

| Scenario | Expected Output |
|----------|----------------|
| 🌊 Urban Flooding - G10 | CRITICAL flood → 4 actions (rescue + reroute + alert + hospital) |
| ⚡ Power Outage - F7 | HIGH power_outage → 3 actions |
| 🚗 Road Accident - Murree Road | HIGH accident → 3 actions including hospital |

Each scenario auto-fills signals and triggers the full 4-agent pipeline.

---

## Agent Trace Explanation

Each agent logs its reasoning at every step. View them in the **Agent Trace** screen:

| Color | Meaning |
|-------|---------|
| 🔵 Blue | AGENT_1 (Signal Ingestion) |
| 🟠 Orange | AGENT_2 (Crisis Detection) |
| 🟢 Green | AGENT_3 (Action Planning) |
| 🔴 Red | AGENT_4 (Execution Simulation) |
| White | INFO log |
| Yellow | WARNING |
| Green text | SUCCESS |
| Red text | ERROR |

---

## Hackathon Checklist

- ✅ Multi-source signal ingestion (social, weather, traffic)
- ✅ 4 distinct agents with traceable reasoning
- ✅ Gemini-powered crisis detection with confidence score
- ✅ Action planning with priority ranking
- ✅ Simulation of ALL action types
- ✅ Before/After state visualization
- ✅ Complete execution logs (exportable)
- ✅ Mobile app as primary deliverable
- ✅ Roman Urdu / Urdu / English support
- ✅ Demo scenarios for judges

---

## Assumptions & Limitations

- **Mock Map**: Google Maps integration uses a static grid visualization (mock) — real map requires a valid Maps API key.
- **Real-time**: Signals are not live-streamed; they are loaded from mock JSON. In production, WebSockets would push live signals.
- **Pakistani Data**: All mock data uses real Islamabad/Rawalpindi coordinates and authentic Pakistani city names.
- **MOCK_MODE=true**: When `MOCK_MODE=true`, Gemini calls are replaced with realistic pre-built responses for offline demo.
