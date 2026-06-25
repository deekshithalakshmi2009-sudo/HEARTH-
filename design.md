# Design Document: Hearth Homelessness Prevention Platform

## Overview

Hearth is a full-stack real-time homelessness prevention platform with the tagline "Predict. Prevent. Protect." It combines a Python FastAPI backend—featuring a weighted-scoring risk engine and a continuous city-block simulator—with a React 18 / Vite frontend that delivers a cinematic, scroll-driven map dashboard.

The platform is designed for two primary audiences:
- **City outreach coordinators** who need at-a-glance situational awareness and one-click dispatch capability.
- **Policymakers** who need quantified cost-benefit data to justify preventive spending.

### Key Design Goals

1. **Real-time feedback loop** — Risk scores update every 3 seconds via WebSocket; the map and sidebar reflect changes within 500ms.
2. **Resilience** — The simulator is fault-tolerant; a single bad RiskEngine call never halts the tick. The frontend reconnects automatically with exponential backoff.
3. **Correctness** — The risk formula is deterministic and bounded; all outputs are clamped to [0, 100]; level thresholds are unambiguous.
4. **Cinematic quality** — Every transition, scroll animation, and data change is intentionally animated using Framer Motion; the visual bar is set by Vercel and Linear.
5. **Deployability** — Backend ships with `render.yaml`; frontend ships with `vercel.json` and `.env.example`; zero manual configuration required.

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Browser (React 18 + Vite)             │
│                                                         │
│  App ──► Navbar, Hero, MapSection, StatsSection,        │
│          CostCalculator, SuccessStories, Footer          │
│                                                         │
│  useWebSocket hook ◄──────────── WebSocket /ws/live     │
│  fetch() calls ◄──────────────── REST /api/*            │
└────────────────────────┬────────────────────────────────┘
                         │ HTTPS / WSS
┌────────────────────────▼────────────────────────────────┐
│                Python FastAPI (Uvicorn)                  │
│                                                         │
│  REST Endpoints:                                        │
│    GET  /api/blocks          → all 16 blocks            │
│    GET  /api/blocks/{id}     → single block             │
│    POST /api/dispatch/{id}   → outreach dispatch        │
│                                                         │
│  WebSocket:                                             │
│    /ws/live → connection pool + 3s broadcast task       │
│                                                         │
│  ┌─────────────────┐   ┌────────────────────────────┐  │
│  │  CitySimulator  │──►│      RiskEngine             │  │
│  │  (16 blocks,    │   │  (weighted score formula)   │  │
│  │   random walk)  │   │                             │  │
│  └─────────────────┘   └────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Technology Choices

| Layer | Choice | Rationale |
|---|---|---|
| Backend framework | FastAPI (Python 3.11+) | Async-native, first-class WebSocket support, automatic OpenAPI docs |
| ASGI server | Uvicorn | Single-worker async; no threading complexity needed for simulation |
| Frontend framework | React 18 + Vite | Concurrent rendering; Vite's HMR accelerates dev cycle |
| Map library | Mapbox GL JS | GeoJSON source + paint expressions allow data-driven color without re-renders |
| Animation library | Framer Motion | Declarative `useScroll` / `whileInView` API maps directly to requirements |
| Styling | Tailwind CSS v3 | Utility classes; custom design tokens in `tailwind.config.js` |
| WebSocket (client) | Native browser `WebSocket` | No extra dependency; hook encapsulates all lifecycle complexity |
| Deployment (backend) | Render (render.yaml) | Free tier, environment variables, automatic deploys |
| Deployment (frontend) | Vercel (vercel.json) | SPA rewrite rule, CDN, preview URLs |

---

## Components and Interfaces

### Backend Components

#### `RiskEngine` (`backend/risk_engine.py`)

Stateless utility class. All methods are pure functions of their arguments.

```python
class RiskEngine:
    # Weight constants (sum = 1.0)
    WEIGHT_EVICTION   = 0.40
    WEIGHT_WAITLIST   = 0.30
    WEIGHT_INCIDENTS  = 0.20
    WEIGHT_ENCAMPMENT = 0.10

    def calculate(
        self,
        eviction_rate: float,    # normalised 0-100
        wait_days: float,        # normalised 0-100
        calls: float,            # normalised 0-100
        encampment: float,       # normalised 0-100
    ) -> float:
        """Return score clamped to [0, 100]."""

    def get_level(self, score: float) -> str:
        """Return 'critical'|'high'|'medium'|'low'|'safe'."""

    def get_color(self, level: str) -> str:
        """Return hex color string for the given level."""

    def get_action(self, level: str) -> str:
        """Return human-readable recommended action string."""

    def get_cost_savings(self) -> dict:
        """Return {'prevention': 3500, 'emergency': 45000, 'savings': 41500}."""
```

**Threshold table** (boundary belongs to higher severity):

| Score range | Level |
|---|---|
| [80, 100] | `critical` |
| [60, 79] | `high` |
| [40, 59] | `medium` |
| [20, 39] | `low` |
| [0, 19] | `safe` |

**Color map:**

| Level | Hex |
|---|---|
| `critical` | `#dc2626` |
| `high` | `#f97316` |
| `medium` | `#eab308` |
| `low` | `#84cc16` |
| `safe` | `#22c55e` |

---

#### `CitySimulator` (`backend/simulator.py`)

Stateful class owning the 16-block ground truth. Thread-safe via `asyncio`—all mutations happen inside the single event-loop thread.

```python
class CitySimulator:
    def __init__(self) -> None:
        """Seed 16 blocks with initial random factor values."""

    def get_block(self, block_id: str) -> dict | None:
        """Return block dict or None if id unknown."""

    def get_all_blocks(self) -> list[dict]:
        """Return shallow copy of all 16 block dicts."""

    def update(self) -> None:
        """
        Random-walk tick:
          1. For each block, add U(-10,+10) to each numeric factor.
          2. Clamp each factor to its valid range.
          3. Call RiskEngine.calculate(); on exception, log + retain old score.
          4. Update risk_score and risk_level fields.
        """

    def dispatch_outreach(self, block_id: str) -> dict | None:
        """
        Reduce each numeric factor by 15% (floor 0) and recalculate.
        Returns updated block dict, or None if id unknown.
        """
```

**Block state schema** (Python dict / Pydantic model):

```
id            : str           — kebab-case name (e.g. "downtown")
name          : str           — display name (e.g. "Downtown")
lat           : float         — [37.70, 37.82]
lng           : float         — [-122.52, -122.36]
evictions     : float         — [0, 100] eviction rate
wait_days     : int           — [0, 365] shelter waitlist days
calls         : int           — [0, 200] mental health incidents
encampment    : int           — [0, 10] encampment count
population    : int           — static display value
income        : int           — median income, static display value
risk_score    : float         — [0, 100]
risk_level    : str           — one of the 5 levels
```

**16 Block definitions** (name → approximate coordinates):

| # | Name | lat | lng |
|---|---|---|---|
| 1 | Downtown | 37.7749 | -122.4194 |
| 2 | Northside | 37.8044 | -122.4712 |
| 3 | Southside | 37.7082 | -122.4637 |
| 4 | East End | 37.7562 | -122.3826 |
| 5 | West Park | 37.7599 | -122.5014 |
| 6 | Riverside | 37.7295 | -122.4002 |
| 7 | Hilltop | 37.7954 | -122.4056 |
| 8 | Lakeside | 37.7249 | -122.4764 |
| 9 | Maplewood | 37.7412 | -122.4301 |
| 10 | Cedar Heights | 37.7841 | -122.4512 |
| 11 | Oak Grove | 37.7188 | -122.4513 |
| 12 | Brookline | 37.7681 | -122.4399 |
| 13 | Fairmont | 37.7892 | -122.4052 |
| 14 | Kingsley | 37.7631 | -122.4823 |
| 15 | Pleasant Hill | 37.7515 | -122.4621 |
| 16 | Sunnydale | 37.7135 | -122.4267 |

---

#### FastAPI Application (`backend/main.py`)

```python
# Lifespan context manager
@asynccontextmanager
async def lifespan(app: FastAPI):
    simulator = CitySimulator()
    app.state.simulator = simulator
    task = asyncio.create_task(broadcast_loop(simulator))
    yield
    task.cancel()

app = FastAPI(lifespan=lifespan)

# CORS middleware — origin read from FRONTEND_URL env var
app.add_middleware(CORSMiddleware, allow_origins=[...], ...)

# Connection pool
connected_clients: set[WebSocket] = set()

# REST endpoints
GET  /api/blocks          → list[BlockSchema]
GET  /api/blocks/{id}     → BlockSchema | 404
POST /api/dispatch/{id}   → BlockSchema | 404

# WebSocket endpoint
@app.websocket("/ws/live")
async def ws_live(websocket: WebSocket): ...

# Broadcast loop (asyncio task)
async def broadcast_loop(simulator: CitySimulator):
    while True:
        await asyncio.sleep(3)
        simulator.update()
        payload = build_broadcast_payload(simulator)
        await broadcast_to_all(payload)
```

**`broadcast_loop` detail:**
1. `await asyncio.sleep(3)` — yields control to event loop.
2. `simulator.update()` — synchronous; completes quickly (16 blocks × cheap arithmetic).
3. Build JSON payload: list of all blocks + UTC timestamp.
4. Iterate `connected_clients`; call `ws.send_json(payload)` in a gathered task; on `WebSocketDisconnect` or any send error, remove from pool silently.

---

### Frontend Components

#### Component Hierarchy

```
App
├── Navbar
├── Hero
├── MapSection
│   ├── Map                   (Mapbox GL JS)
│   └── BlockDetails (Sidebar)
│       └── DispatchButton
├── StatsSection
│   └── AnimatedCounter (×3)
├── CostCalculator
│   └── AnimatedCounter (×2)
├── SuccessStories
│   └── StoryCard (×4)
└── Footer
```

#### `App` (root component)

Owns top-level state and the dispatch handler:

```typescript
const [selectedBlock, setSelectedBlock] = useState<Block | null>(null);
const [familiesHelped, setFamiliesHelped] = useState(47);
const [moneySaved, setMoneySaved]         = useState(1_950_000);
const [totalSaved, setTotalSaved]         = useState(0);  // for CostCalculator

function handleDispatchSuccess() {
  setFamiliesHelped(n => n + 1);
  setMoneySaved(n => n + 3_500);
  setTotalSaved(n => n + 41_500);
  triggerConfetti();
}
```

#### `useWebSocket` hook (`src/hooks/useWebSocket.ts`)

```typescript
interface UseWebSocketOptions {
  url: string;
  onMessage: (data: BroadcastPayload) => void;
  onConnectionError?: () => void;
}

interface UseWebSocketReturn {
  connectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'error';
}

// Reconnection parameters
const INITIAL_DELAY_MS  = 1_000;
const BACKOFF_MULTIPLIER = 2;
const MAX_DELAY_MS      = 30_000;
const MAX_RETRIES       = 5;
const HEARTBEAT_INTERVAL_MS = 30_000;
const HEARTBEAT_TIMEOUT_MS  =  5_000;
```

**State machine:**

```
CONNECTING
    │ open ──────────────────────► CONNECTED
    │                                  │
    │                              heartbeat ping every 30s
    │                              if no pong in 5s → RECONNECTING
    │
    │ close/error ────────────────► RECONNECTING
    │                                  │
    │                              delay = min(initial × 2^attempt, 30s)
    │                              retry ≤ 5 times
    │                              success → CONNECTING
    │                              exhausted → ERROR
```

Heartbeat implementation: send `JSON.stringify({type:'ping'})` on interval; track `lastPongAt`; if `Date.now() - lastPongAt > HEARTBEAT_TIMEOUT_MS` after ping, close socket and trigger reconnect.

#### `Map` component (`src/components/Map.tsx`)

- Initialises `mapboxgl.Map` once via `useEffect`; stores ref.
- Adds GeoJSON `FeatureCollection` source (`"blocks"`) after map loads.
- Adds circle layer with data-driven paint:
  ```json
  "circle-color": ["match", ["get", "risk_level"],
    "critical", "#dc2626",
    "high",     "#f97316",
    "medium",   "#eab308",
    "low",      "#84cc16",
    "safe",     "#22c55e",
                "#22c55e"]
  ```
- On WebSocket update: call `map.getSource("blocks").setData(newGeoJSON)` — Mapbox re-renders only changed features.
- Critical pulse: maintain a CSS class on the `<canvas>` overlay via a `Set<string>` of critical block IDs; a `@keyframes pulse` animation (scale 1→1.4→1, opacity 1→0→1) is applied to a transparent overlay `<div>` per critical marker via absolute positioning.
- Hover popup: `map.on("mouseenter", ...)` → `new mapboxgl.Popup(...)` showing name + score; `map.on("mouseleave", ...)` → `popup.remove()`.
- Click selection: `map.on("click", "blocks-layer", ...)` → calls `onSelectBlock(feature.properties)`.

#### `Navbar`

- Framer Motion `motion.nav` with `useScroll`; `backgroundColor` interpolated from `"rgba(0,0,0,0)"` → `"rgba(15,23,42,0.95)"` over first 100px of scroll.

#### `Hero`

- Full-viewport `<section>` with `background-image: url('/hero-bg.jpg')` and a `linear-gradient(rgba(0,0,0,0.6), rgba(0,0,0,0.6))` overlay.
- Fallback background: `background-color: #0f172a` applied unconditionally; image layered on top via CSS.
- CTA button: Tailwind `animate-pulse` with `animation-duration: 2s`.
- Scroll arrow: Tailwind `animate-bounce`.
- Hero opacity tied to scroll via `useTransform(scrollY, [0, vh], [1, 0])`.

#### `ScrollReveal` (`src/components/ScrollReveal.tsx`)

```typescript
// Uses Framer Motion whileInView + viewport once:true
<motion.div
  initial={{ opacity: 0, y: 30 }}
  whileInView={{ opacity: 1, y: 0 }}
  viewport={{ once: true, amount: 0.2 }}
  transition={{ duration: 0.5, ease: "easeOut" }}
>
  {children}
</motion.div>
```

`once: true` satisfies requirement 6.6 (no re-trigger).

#### `BlockDetails` / Sidebar

- Glassmorphism card: `backdrop-blur-md bg-white/10 border border-white/20 rounded-3xl`.
- Animated progress bars: Framer Motion `motion.div` width animated from `0%` to `${value}%` on mount/update.
- Dispatch button: local `isDispatching` state; disabled during in-flight call; spinner via `animate-spin` class.
- Success/error toasts: absolute-positioned `motion.div` with auto-dismiss via `setTimeout`.

#### `CostCalculator`

- Bar widths: prevention = `(3500 / 45000) * 100 ≈ 7.78%`; emergency = `100%`.
- Counter animation: custom `useCountUp` hook using `requestAnimationFrame` with ease-out interpolation.
- `once` guard: `useRef<boolean>` to prevent re-trigger after first viewport entry.

#### `StatsSection`

- Three metric cards with `useCountUp` on `"Families Helped"` (target 47) and `"Money Saved"` (target 1,950,000).
- "Blocks Monitored" is static (16); no counter animation needed.
- Dispatch increments propagated from App via props.

---

## Data Models

### Pydantic Models (backend)

```python
from pydantic import BaseModel

class BlockSchema(BaseModel):
    id:          str
    name:        str
    lat:         float
    lng:         float
    evictions:   float      # 0-100
    wait_days:   int        # 0-365
    calls:       int        # 0-200
    encampment:  int        # 0-10
    population:  int
    income:      int
    risk_score:  float      # 0-100, clamped
    risk_level:  str        # critical|high|medium|low|safe

class BroadcastPayload(BaseModel):
    timestamp: str          # ISO-8601 UTC
    blocks:    list[BlockSchema]

class DispatchResponse(BaseModel):
    id:         str
    risk_score: float
    risk_level: str
    evictions:  float
    wait_days:  int
    calls:      int
    encampment: int

class ErrorResponse(BaseModel):
    error: str
```

### TypeScript Types (frontend)

```typescript
// src/types/index.ts

export interface Block {
  id:         string;
  name:       string;
  lat:        number;
  lng:        number;
  evictions:  number;
  wait_days:  number;
  calls:      number;
  encampment: number;
  population: number;
  income:     number;
  risk_score: number;
  risk_level: RiskLevel;
}

export type RiskLevel = 'critical' | 'high' | 'medium' | 'low' | 'safe';

export interface BroadcastPayload {
  timestamp: string;
  blocks:    Block[];
}

export const RISK_COLORS: Record<RiskLevel, string> = {
  critical: '#dc2626',
  high:     '#f97316',
  medium:   '#eab308',
  low:      '#84cc16',
  safe:     '#22c55e',
};
```

### WebSocket Message Contract

**Backend → Frontend (broadcast):**
```json
{
  "timestamp": "2024-01-15T18:30:00.000Z",
  "blocks": [
    {
      "id": "downtown",
      "name": "Downtown",
      "lat": 37.7749,
      "lng": -122.4194,
      "evictions": 72.3,
      "wait_days": 45,
      "calls": 120,
      "encampment": 6,
      "population": 48000,
      "income": 62000,
      "risk_score": 68.4,
      "risk_level": "high"
    }
    // ... 15 more blocks
  ]
}
```

**Frontend → Backend (heartbeat):**
```json
{ "type": "ping" }
```

**Backend → Frontend (heartbeat ack):**
```json
{ "type": "pong" }
```

---

### Folder Structure

```
hearth/
├── backend/
│   ├── main.py              # FastAPI app, CORS, endpoints, WebSocket, broadcast loop
│   ├── risk_engine.py       # RiskEngine class
│   ├── simulator.py         # CitySimulator class
│   ├── requirements.txt     # fastapi, uvicorn, pydantic, python-dotenv
│   └── render.yaml          # Render deployment manifest
├── frontend/
│   ├── public/
│   │   └── hero-bg.jpg      # (placeholder; developer-supplied)
│   ├── src/
│   │   ├── components/
│   │   │   ├── Navbar.tsx
│   │   │   ├── Hero.tsx
│   │   │   ├── MapSection.tsx
│   │   │   ├── Map.tsx
│   │   │   ├── BlockDetails.tsx
│   │   │   ├── StatsSection.tsx
│   │   │   ├── CostCalculator.tsx
│   │   │   ├── SuccessStories.tsx
│   │   │   ├── StoryCard.tsx
│   │   │   ├── ScrollReveal.tsx
│   │   │   ├── AnimatedCounter.tsx
│   │   │   └── Footer.tsx
│   │   ├── hooks/
│   │   │   ├── useWebSocket.ts
│   │   │   └── useCountUp.ts
│   │   ├── utils/
│   │   │   ├── riskColors.ts
│   │   │   └── formatters.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── package.json
│   ├── vite.config.js
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   └── vercel.json
├── .env.example
├── README.md
├── PITCH.md
└── SUBMISSION.md
```

---

### Tailwind Design Tokens (`tailwind.config.js`)

```javascript
theme: {
  extend: {
    colors: {
      primary:   { DEFAULT: '#f59e0b', ... },  // amber
      secondary: { DEFAULT: '#0f172a', ... },  // navy
      cream:     { DEFAULT: '#fef9f0', ... },
      risk: {
        critical: '#dc2626',
        high:     '#f97316',
        medium:   '#eab308',
        low:      '#84cc16',
        safe:     '#22c55e',
      },
    },
    fontFamily: {
      display: ['Playfair Display', 'Georgia', 'serif'],
      body:    ['Inter', 'system-ui', 'sans-serif'],
    },
    borderRadius: {
      '4xl': '2rem',   // 32px — glassmorphism cards
    },
    animation: {
      'pulse-slow': 'pulse 2s cubic-bezier(0.4,0,0.6,1) infinite',
      'marker-pulse': 'markerPulse 1.5s ease-in-out infinite',
    },
    keyframes: {
      markerPulse: {
        '0%, 100%': { transform: 'scale(1)',   opacity: '1'   },
        '50%':      { transform: 'scale(1.4)', opacity: '0.6' },
      },
    },
  },
},
```

### Framer Motion Variants (`src/utils/motionVariants.ts`)

```typescript
export const fadeInUp = {
  hidden:  { opacity: 0, y: 30 },
  visible: { opacity: 1, y: 0, transition: { duration: 0.5, ease: 'easeOut' } },
};

export const slideUp = {
  hidden:  { opacity: 0, y: 30 },
  visible: { opacity: 1, y: 0, transition: { duration: 0.5, ease: 'easeOut' } },
};

export const staggerChildren = {
  visible: { transition: { staggerChildren: 0.1 } },
};

export const cardReveal = {
  hidden:  { opacity: 0, y: 20 },
  visible: { opacity: 1, y: 0, transition: { duration: 0.4, ease: 'easeOut' } },
};
```


---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

The following properties are derived from the testable acceptance criteria. Each is universally quantified and suitable for property-based testing with a minimum of 100 iterations per run.

---

### Property 1: Score Formula Correctness

*For any* valid combination of eviction rate, waitlist days, mental health incidents, and encampment count (each normalised to [0, 100]), the value returned by `RiskEngine.calculate()` SHALL equal `clamp(0.4 * evictions + 0.3 * wait_days + 0.2 * calls + 0.1 * encampment, 0, 100)`.

**Validates: Requirements 1.1**

---

### Property 2: Score Bounds Invariant

*For any* combination of input factors (including boundary values and any values within valid ranges), `RiskEngine.calculate()` SHALL return a value in the closed interval [0, 100].

**Validates: Requirements 1.2**

---

### Property 3: Score Determinism

*For any* combination of input factors, calling `RiskEngine.calculate()` twice with the same inputs SHALL return the same value both times — the function has no side effects and no randomness.

**Validates: Requirements 1.5**

---

### Property 4: Level–Score Threshold Consistency

*For any* float `s` in [0, 100], `RiskEngine.get_level(s)` SHALL return a level that is consistent with the threshold table: `critical` if s ≥ 80, `high` if 60 ≤ s < 80, `medium` if 40 ≤ s < 60, `low` if 20 ≤ s < 40, and `safe` if s < 20. Boundary values (80, 60, 40, 20) SHALL be assigned to the higher severity level.

**Validates: Requirements 1.3, 1.5**

---

### Property 5: Simulator Factor Bounds Preservation

*For any* initial simulator state and any positive integer number of `update()` ticks, all Block risk scores SHALL remain in [0, 100] and all numeric factors SHALL remain within their defined valid ranges (`evictions` in [0, 100], `wait_days` in [0, 365], `calls` in [0, 200], `encampment` in [0, 10]) after every tick.

**Validates: Requirements 2.3, 2.5**

---

### Property 6: Simulator Error Isolation

*For any* Block ID in the simulator, if `RiskEngine.calculate()` raises an exception when computing that Block's score during a tick, the simulator SHALL retain the Block's previous `risk_score` unchanged, AND all other Blocks in the same tick SHALL be updated normally (their `risk_score` values SHALL differ from their pre-tick values with probability proportional to the random walk).

**Validates: Requirements 2.6**

---

### Property 7: Fetch Block by Valid ID Returns Correct Block

*For any* block ID drawn from the set of the 16 known block IDs, a `GET /api/blocks/{id}` request SHALL return HTTP 200 and a JSON object whose `id` field equals the requested ID, and whose fields include `name`, `lat`, `lng`, `risk_score`, `risk_level`, `evictions`, `wait_days`, `calls`, and `encampment`.

**Validates: Requirements 3.2**

---

### Property 8: Unknown Block ID Returns 404

*For any* string that is not one of the 16 known block IDs, both `GET /api/blocks/{id}` and `POST /api/dispatch/{id}` SHALL return HTTP 404 and a JSON response body containing an `error` field whose value is a non-empty string identifying the missing block ID.

**Validates: Requirements 3.3, 3.5**

---

### Property 9: Dispatch Reduces Risk Factors

*For any* valid block ID, calling `POST /api/dispatch/{id}` SHALL return HTTP 200, and the returned `evictions`, `wait_days`, `calls`, and `encampment` values SHALL each be less than or equal to their values immediately before the dispatch was applied (the dispatch action never increases any risk factor).

**Validates: Requirements 3.4**

---

### Property 10: Broadcast Payload Completeness

*For any* broadcast message received from the `/ws/live` WebSocket endpoint, the message SHALL be valid JSON containing a `blocks` array of exactly 16 elements and a `timestamp` string, where each element contains at minimum the fields `id`, `risk_score`, `risk_level`, and each `risk_score` is in [0, 100], and each `risk_level` is one of the five valid level strings.

**Validates: Requirements 4.2**

---

### Property 11: WebSocket Disconnection Isolation

*For any* set of N ≥ 2 connected WebSocket clients, disconnecting any subset of those clients SHALL not prevent the remaining connected clients from receiving subsequent broadcasts. The remaining clients SHALL continue to receive at least one additional broadcast after the disconnection occurs.

**Validates: Requirements 4.3**

---

### Property 12: Exponential Backoff Delay Formula

*For any* retry attempt number `n` in {1, 2, 3, 4, 5}, the reconnection delay computed by the `useWebSocket` hook SHALL equal `min(1000 × 2^(n−1), 30000)` milliseconds: 1000, 2000, 4000, 8000, 16000 for attempts 1–5 respectively (all below the 30 000ms cap).

**Validates: Requirements 4.4**

---

### Property 13: Number Formatter Correctness

*For any* non-negative integer `n`, the formatting utility used by `StatsSection` and `CostCalculator` SHALL return a string that (a) contains the correct numeric value, (b) uses comma separators for groups of three digits for values ≥ 1 000, and (c) for currency formatting, is prefixed with `$` with no space between the prefix and the first digit.

**Validates: Requirements 10.4**

---

## Error Handling

### Backend Error Handling

#### RiskEngine exceptions in simulator ticks
The `CitySimulator.update()` method wraps each per-block `RiskEngine.calculate()` call in a `try/except Exception` block. On exception:
- The block retains its previous `risk_score` and `risk_level`.
- The error is logged via Python's `logging` module at `ERROR` level with the block ID and exception details.
- The tick continues to the next block.

This prevents a single malformed block state from halting the entire simulation.

#### Unknown block IDs in REST endpoints
Both `GET /api/blocks/{id}` and `POST /api/dispatch/{id}` call `simulator.get_block(id)` and check for `None`. If `None`:
```python
raise HTTPException(status_code=404, detail={"error": f"Block '{id}' not found"})
```
FastAPI serialises the `detail` dict to JSON automatically.

#### Malformed dispatch request bodies
The `POST /api/dispatch/{id}` endpoint ignores the request body entirely — the block ID comes from the URL path only. FastAPI will not raise a validation error for an unexpected or missing body because the endpoint declares no `Body` parameter.

#### WebSocket send failures
The broadcast loop iterates `connected_clients` and calls `ws.send_json()`. Any `WebSocketDisconnect`, `ConnectionClosedOK`, or other send-time exception is caught per-client:
```python
for ws in list(connected_clients):
    try:
        await ws.send_json(payload)
    except Exception:
        connected_clients.discard(ws)
```
This ensures one failed client never blocks others.

#### Missing `FRONTEND_URL` environment variable
On startup, the application checks `os.getenv("FRONTEND_URL")`. If absent, logs a `WARNING` and sets `allow_origins=["*"]`.

### Frontend Error Handling

#### WebSocket connection failure and reconnection
The `useWebSocket` hook manages a state machine with five states: `connecting`, `connected`, `reconnecting`, `error`. On `onerror` or `onclose`:
1. Compute `delay = Math.min(INITIAL_DELAY * 2 ** retryCount, MAX_DELAY)`.
2. `setTimeout(() => reconnect(), delay)` — retry up to `MAX_RETRIES`.
3. After 5 failures: set `connectionStatus = "error"`, call `onConnectionError()` if provided.

#### Missing environment variables
`src/main.tsx` (or a dedicated `validateEnv.ts` utility) checks for each of `VITE_MAPBOX_TOKEN`, `VITE_API_URL`, and `VITE_WS_URL` at startup and logs a `console.warn` for any that are absent or empty.

#### Hero background image failure
The `Hero` component sets `background-color: #0f172a` unconditionally via Tailwind. The `hero-bg.jpg` image is layered on top using a CSS `background-image`. If the image fails to load, the browser simply falls back to the solid dark background color.

#### Dispatch API failures
The `BlockDetails` component wraps the `fetch` call in `try/catch`. On any non-2xx response or network error:
- `isDispatching` is set back to `false` (re-enables the button).
- An error toast is displayed for 3 seconds.

#### Font loading failure
The Tailwind `fontFamily` configuration specifies system-font fallbacks:
```
font-display: ['Playfair Display', 'Georgia', 'serif']
font-body:    ['Inter', 'system-ui', 'sans-serif']
```
If the web fonts fail to load, the browser automatically uses the next font in the stack.

---

## Testing Strategy

### Overview

The testing strategy uses a dual approach: **property-based tests** for universal correctness invariants (Properties 1–13 above) and **example-based unit/integration tests** for specific scenarios, edge cases, and infrastructure verification.

### Backend Testing

#### Technology
- **Property-based testing**: [Hypothesis](https://hypothesis.readthedocs.io/) (Python)
  - Minimum 100 iterations per property (`@settings(max_examples=100)`)
  - Each test tagged with a comment: `# Feature: hearth-platform, Property N: <property_text>`
- **Example-based testing**: `pytest` with `httpx.AsyncClient` for REST endpoint tests and `websockets` library for WebSocket integration tests

#### Property Tests (`backend/tests/test_properties.py`)

**Property 1 — Score Formula Correctness:**
```python
@given(
    evictions=floats(0, 100), wait_days=floats(0, 100),
    calls=floats(0, 100), encampment=floats(0, 100)
)
@settings(max_examples=100)
def test_score_formula_correctness(evictions, wait_days, calls, encampment):
    # Feature: hearth-platform, Property 1: Score Formula Correctness
    expected = max(0.0, min(100.0, 0.4*evictions + 0.3*wait_days + 0.2*calls + 0.1*encampment))
    result = RiskEngine().calculate(evictions, wait_days, calls, encampment)
    assert abs(result - expected) < 1e-9
```

**Property 2 — Score Bounds:**
```python
@given(...)
def test_score_bounds(evictions, wait_days, calls, encampment):
    # Feature: hearth-platform, Property 2: Score Bounds Invariant
    result = RiskEngine().calculate(evictions, wait_days, calls, encampment)
    assert 0.0 <= result <= 100.0
```

**Property 3 — Determinism:**
```python
@given(...)
def test_score_determinism(...):
    # Feature: hearth-platform, Property 3: Score Determinism
    engine = RiskEngine()
    assert engine.calculate(*args) == engine.calculate(*args)
```

**Property 4 — Threshold Consistency:**
```python
@given(score=floats(0, 100))
def test_level_threshold_consistency(score):
    # Feature: hearth-platform, Property 4: Level–Score Threshold Consistency
    level = RiskEngine().get_level(score)
    if score >= 80:   assert level == 'critical'
    elif score >= 60: assert level == 'high'
    elif score >= 40: assert level == 'medium'
    elif score >= 20: assert level == 'low'
    else:             assert level == 'safe'
```

**Property 5 — Simulator Factor Bounds:**
```python
@given(ticks=integers(1, 50))
def test_simulator_factor_bounds(ticks):
    # Feature: hearth-platform, Property 5: Simulator Factor Bounds Preservation
    sim = CitySimulator()
    for _ in range(ticks):
        sim.update()
    for block in sim.get_all_blocks():
        assert 0 <= block['risk_score'] <= 100
        assert 0 <= block['evictions'] <= 100
        assert 0 <= block['wait_days'] <= 365
        assert 0 <= block['calls'] <= 200
        assert 0 <= block['encampment'] <= 10
```

**Property 6 — Error Isolation:**
Uses a mock `RiskEngine` that raises for a randomly-chosen block ID; verifies others update normally.

**Property 12 — Backoff Formula:**
```python
@given(n=integers(1, 5))
def test_exponential_backoff(n):
    # Feature: hearth-platform, Property 12: Exponential Backoff Delay Formula
    expected = min(1000 * (2 ** (n - 1)), 30000)
    assert compute_backoff_delay(n) == expected
```

#### Integration / Example Tests (`backend/tests/test_api.py`)

| Test | Type | Validates |
|---|---|---|
| `test_get_all_blocks_returns_16` | Integration | Req 3.1 |
| `test_get_block_by_valid_id` (16 IDs) | Example | Req 3.2 |
| `test_get_block_unknown_id_404` | Property (Hypothesis) | Req 3.3 |
| `test_dispatch_valid_id_reduces_factors` | Property (Hypothesis) | Req 3.4 |
| `test_dispatch_unknown_id_404` | Property (Hypothesis) | Req 3.5 |
| `test_cors_headers_present` | Integration | Req 3.6 |
| `test_dispatch_malformed_body_still_200` | Example | Req 3.7 |
| `test_websocket_broadcast_payload` | Integration | Req 4.1, 4.2 |
| `test_websocket_disconnection_isolation` | Integration | Req 4.3 |
| `test_color_map_all_five_levels` | Example | Req 1.4 |
| `test_simulator_16_blocks_correct_names` | Example | Req 2.1 |
| `test_simulator_coordinates_in_sf_bounds` | Example | Req 2.2 |

### Frontend Testing

#### Technology
- **Property-based testing**: [fast-check](https://fast-check.dev/) (TypeScript/JavaScript)
  - Minimum 100 iterations per property (`{ numRuns: 100 }`)
  - Each test tagged: `// Feature: hearth-platform, Property N: <property_text>`
- **Example-based testing**: Vitest + React Testing Library for component tests
- **E2E** (optional): Playwright for scroll animation smoke tests

#### Property Tests (`frontend/src/__tests__/properties.test.ts`)

**Property 7 — Fetch Block by Valid ID:**
```typescript
// Feature: hearth-platform, Property 7: Fetch Block by Valid ID Returns Correct Block
fc.assert(fc.asyncProperty(
  fc.constantFrom(...VALID_BLOCK_IDS),
  async (id) => {
    const res = await fetch(`${API_URL}/api/blocks/${id}`);
    const data = await res.json();
    expect(res.status).toBe(200);
    expect(data.id).toBe(id);
  }
), { numRuns: 16 });
```

**Property 8 — Unknown ID Returns 404:**
```typescript
// Feature: hearth-platform, Property 8: Unknown Block ID Returns 404
fc.assert(fc.asyncProperty(
  fc.string().filter(s => !VALID_BLOCK_IDS.includes(s)),
  async (id) => {
    const res = await fetch(`${API_URL}/api/blocks/${id}`);
    const data = await res.json();
    expect(res.status).toBe(404);
    expect(data.error).toBeTruthy();
  }
));
```

**Property 12 — Backoff Formula (pure function test):**
```typescript
// Feature: hearth-platform, Property 12: Exponential Backoff Delay Formula
fc.assert(fc.property(
  fc.integer({ min: 1, max: 5 }),
  (n) => {
    const expected = Math.min(1000 * Math.pow(2, n - 1), 30000);
    expect(computeBackoffDelay(n)).toBe(expected);
  }
));
```

**Property 13 — Formatter Correctness:**
```typescript
// Feature: hearth-platform, Property 13: Number Formatter Correctness
fc.assert(fc.property(
  fc.integer({ min: 0, max: 10_000_000 }),
  (n) => {
    const formatted = formatNumber(n);
    // No commas in wrong places; currency prefix for currency formatter
    expect(formatted).toMatch(/^\d{1,3}(,\d{3})*$/);
  }
));
```

#### Unit / Example Tests (`frontend/src/__tests__/components.test.tsx`)

| Test | Type | Validates |
|---|---|---|
| `Navbar renders links` | Example | Req 5, 12 |
| `Hero renders headline and CTA` | Example | Req 5.2, 5.3, 5.4 |
| `Hero fallback background color` | Example | Req 5.9 |
| `CostCalculator ratio display` | Example | Req 9.3 |
| `StatsSection initial values` | Example | Req 10.1 |
| `SuccessStories renders 4 cards` | Example | Req 11.1 |
| `useWebSocket sets error after 5 retries` | Example | Req 4.6 |
| `BlockDetails disables button during dispatch` | Example | Req 8.3 |
| `BlockDetails shows success toast on 200` | Example | Req 8.5 |
| `BlockDetails shows error toast on failure` | Example | Req 8.8 |
| `env validation logs warnings for missing vars` | Example | Req 14.4 |

### Test Configuration Notes

- Hypothesis settings: `@settings(max_examples=100, deadline=None)` — deadline disabled because the simulator update involves I/O-adjacent operations.
- fast-check settings: `{ numRuns: 100, seed: deterministic seed in CI }`.
- All property tests run in CI on every push.
- Integration tests against the live API are gated behind a `TEST_ENV=integration` flag to avoid hitting the deployed service in unit test runs.
