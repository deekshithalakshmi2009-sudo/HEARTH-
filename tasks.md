# Implementation Plan: Hearth Homelessness Prevention Platform

## Overview

Build the Hearth full-stack platform in three stages: (1) backend core (RiskEngine → CitySimulator → FastAPI app), (2) frontend foundation (config, types, utils, hooks), then (3) frontend components wired together in App. Documentation and deployment configs can be written in parallel. Property-based tests are embedded as optional sub-tasks within each relevant implementation task.

---

## Tasks

- [x] 1. Backend: RiskEngine
  - [x] 1.1 Implement `RiskEngine` class in `hearth/backend/risk_engine.py`
    - Define weight constants: `WEIGHT_EVICTION=0.40`, `WEIGHT_WAITLIST=0.30`, `WEIGHT_INCIDENTS=0.20`, `WEIGHT_ENCAMPMENT=0.10`
    - Implement `calculate(eviction_rate, wait_days, calls, encampment) -> float` — weighted sum clamped to [0, 100]
    - Implement `get_level(score) -> str` — threshold table: ≥80 → `critical`, ≥60 → `high`, ≥40 → `medium`, ≥20 → `low`, else `safe`
    - Implement `get_color(level) -> str` — map each level to its hex: `critical=#dc2626`, `high=#f97316`, `medium=#eab308`, `low=#84cc16`, `safe=#22c55e`
    - Implement `get_action(level) -> str` — human-readable recommended action per level
    - Implement `get_cost_savings() -> dict` — return `{'prevention': 3500, 'emergency': 45000, 'savings': 41500}`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

  - [ ]* 1.2 Write property tests for `RiskEngine` in `hearth/backend/tests/test_properties.py`
    - **Property 1: Score Formula Correctness** — `@given(floats(0,100) × 4)` assert `abs(result - expected) < 1e-9` where `expected = clamp(0.4*e + 0.3*w + 0.2*c + 0.1*enc, 0, 100)` — `# Feature: hearth-platform, Property 1`
    - **Property 2: Score Bounds Invariant** — same generators, assert `0.0 <= result <= 100.0` — `# Feature: hearth-platform, Property 2`
    - **Property 3: Score Determinism** — same generators, call `calculate()` twice, assert equality — `# Feature: hearth-platform, Property 3`
    - **Property 4: Level–Score Threshold Consistency** — `@given(floats(0,100))`, assert level matches threshold table including boundary values — `# Feature: hearth-platform, Property 4`
    - Use `@settings(max_examples=100, deadline=None)` on all tests
    - _Requirements: 1.1, 1.2, 1.3, 1.5_

- [x] 2. Backend: CitySimulator
  - [x] 2.1 Implement `CitySimulator` class in `hearth/backend/simulator.py`
    - In `__init__`, seed all 16 blocks with initial random factor values using the name/coordinate table from the design
    - Each block dict must contain: `id`, `name`, `lat`, `lng`, `evictions` [0,100], `wait_days` [0,365], `calls` [0,200], `encampment` [0,10], `population`, `income`, `risk_score`, `risk_level`
    - Implement `get_block(block_id) -> dict | None`
    - Implement `get_all_blocks() -> list[dict]` returning a shallow copy
    - Implement `update()` — random-walk tick: add U(-10,+10) per factor, clamp to valid ranges, call `RiskEngine.calculate()`; on exception log at ERROR level and retain previous `risk_score`/`risk_level`; continue to next block
    - Implement `dispatch_outreach(block_id) -> dict | None` — reduce each numeric factor by 15% (floor 0), recalculate, return updated block or None if unknown
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6_

  - [ ]* 2.2 Write property tests for `CitySimulator` in `hearth/backend/tests/test_properties.py`
    - **Property 5: Simulator Factor Bounds Preservation** — `@given(integers(1,50))`, run N ticks, assert all blocks have `risk_score` in [0,100] and each factor in its valid range — `# Feature: hearth-platform, Property 5`
    - **Property 6: Simulator Error Isolation** — mock `RiskEngine.calculate` to raise for a specific block ID, run one tick, assert that block retains its old `risk_score` and all other blocks' scores differ from their pre-tick values (probabilistically) — `# Feature: hearth-platform, Property 6`
    - Use `@settings(max_examples=100, deadline=None)`
    - _Requirements: 2.3, 2.5, 2.6_

- [ ] 3. Backend: FastAPI application
  - [ ] 3.1 Implement FastAPI app in `hearth/backend/main.py`
    - Import `CitySimulator`, `asyncio`, CORS middleware; define Pydantic models `BlockSchema`, `BroadcastPayload`, `DispatchResponse`, `ErrorResponse`
    - Implement `lifespan` context manager: create `CitySimulator`, attach to `app.state`, launch `broadcast_loop` task, cancel on shutdown
    - Add `CORSMiddleware`; read allowed origin from `FRONTEND_URL` env var; fall back to `"*"` with a `WARNING` log if absent
    - Implement `GET /api/blocks` → returns all 16 blocks as `list[BlockSchema]`
    - Implement `GET /api/blocks/{id}` → returns single `BlockSchema` or raises `HTTPException(404, {"error": "..."})`
    - Implement `POST /api/dispatch/{id}` → calls `dispatch_outreach`, returns `DispatchResponse` or `HTTPException(404)`; ignores request body
    - Implement WebSocket endpoint `GET /ws/live`: add client to `connected_clients`, receive loop handles `ping` → reply `pong`, remove client on disconnect
    - Implement `broadcast_loop(simulator)` coroutine: `sleep(3)` → `simulator.update()` → build payload → iterate clients, send JSON, silently discard on any send error
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 4.1, 4.2, 4.3, 13.2, 13.3_

  - [ ]* 3.2 Write property and integration tests for API in `hearth/backend/tests/test_properties.py` and `test_api.py`
    - **Property 8: Unknown Block ID Returns 404** (Hypothesis `text()` filter) for both `GET /api/blocks/{id}` and `POST /api/dispatch/{id}`; assert HTTP 404 and non-empty `error` field — `# Feature: hearth-platform, Property 8`
    - **Property 9: Dispatch Reduces Risk Factors** (Hypothesis `sampled_from(VALID_IDS)`) — GET block state before dispatch, POST dispatch, assert each factor ≤ pre-dispatch value — `# Feature: hearth-platform, Property 9`
    - Integration: `test_get_all_blocks_returns_16`, `test_get_block_by_valid_id` (all 16 IDs), `test_dispatch_malformed_body_still_200`, `test_cors_headers_present`, `test_websocket_broadcast_payload`, `test_websocket_disconnection_isolation`
    - Use `httpx.AsyncClient` for REST tests; use `websockets` library for WebSocket tests
    - _Requirements: 3.2, 3.3, 3.4, 3.5, 4.2, 4.3_

- [ ] 4. Backend: dependencies and deployment config
  - [ ] 4.1 Create `hearth/backend/requirements.txt`
    - Pin exact versions: `fastapi==0.104.1`, `uvicorn==0.24.0`, `websockets==12.0`, `python-multipart==0.0.6`, `hypothesis` (latest stable), `httpx`, `pytest`, `pytest-asyncio`
    - _Requirements: 13.1_

  - [ ] 4.2 Create `hearth/backend/render.yaml`
    - Define service with non-empty `name`, `runtime: python`, `buildCommand: pip install -r requirements.txt`, `startCommand: uvicorn main:app --host 0.0.0.0 --port $PORT`
    - Include `envVars` block with `FRONTEND_URL` placeholder
    - _Requirements: 13.1_

- [ ] 5. Checkpoint — Backend complete
  - Ensure `pytest hearth/backend/tests/` passes with no failures. Confirm all 16 blocks are returned by `GET /api/blocks`. Ask the user if questions arise before proceeding to frontend.

- [x] 6. Frontend: project scaffolding and configuration
  - [x] 6.1 Create `hearth/frontend/package.json`
    - Pin exact versions: `react@18`, `react-dom@18`, `framer-motion`, `mapbox-gl`, `canvas-confetti`, `tailwindcss@3`, `vite`, `@vitejs/plugin-react`, `typescript`, `vitest`, `fast-check`, `@testing-library/react`, `@testing-library/jest-dom`
    - _Requirements: 14.1_

  - [x] 6.2 Create `hearth/frontend/vite.config.js`
    - Configure `@vitejs/plugin-react`
    - Add `server.proxy` rule: `/api` → `http://localhost:8000`, `/ws` → `ws://localhost:8000` with `ws: true`
    - _Requirements: 14.2_

  - [x] 6.3 Create `hearth/frontend/tailwind.config.js`
    - Extend theme with `colors`: `primary` (amber), `secondary` (navy `#0f172a`), `cream`, `risk` object with all 5 level hex colors
    - Extend `fontFamily`: `display: ['Playfair Display', 'Georgia', 'serif']`, `body: ['Inter', 'system-ui', 'sans-serif']`
    - Extend `borderRadius`: `4xl: '2rem'`
    - Extend `animation`: `pulse-slow`, `marker-pulse`
    - Extend `keyframes`: `markerPulse` (scale 1→1.4 at 50%, opacity 1→0.6)
    - _Requirements: 12.1, 12.2, 12.3, 7.4_

  - [x] 6.4 Create `hearth/frontend/postcss.config.js` — standard Tailwind + autoprefixer setup
    - _Requirements: 12.1_

  - [x] 6.5 Create `hearth/frontend/vercel.json`
    - Add rewrite rule: `source: "/(.*)"`, `destination: "/index.html"` to support SPA client-side routing
    - _Requirements: 14.3_

- [ ] 7. Frontend: types, utilities, and motion variants
  - [x] 7.1 Create `hearth/frontend/src/types/index.ts`
    - Export `Block` interface with all 12 fields (`id`, `name`, `lat`, `lng`, `evictions`, `wait_days`, `calls`, `encampment`, `population`, `income`, `risk_score`, `risk_level`)
    - Export `type RiskLevel = 'critical' | 'high' | 'medium' | 'low' | 'safe'`
    - Export `BroadcastPayload` interface (`timestamp: string`, `blocks: Block[]`)
    - Export `RISK_COLORS: Record<RiskLevel, string>` constant with all 5 hex values
    - _Requirements: 7.2, 1.4_

  - [x] 7.2 Create `hearth/frontend/src/utils/riskColors.ts`
    - Export helper that returns the hex color for a given `RiskLevel`, re-using `RISK_COLORS` from types
    - _Requirements: 7.2, 1.4_

  - [x] 7.3 Create `hearth/frontend/src/utils/formatters.ts`
    - Export `formatNumber(n: number): string` — whole numbers with comma separators for values ≥1,000 (e.g., `1950000` → `"1,950,000"`)
    - Export `formatCurrency(n: number): string` — dollar-prefixed with comma separators, no space (e.g., `1950000` → `"$1,950,000"`)
    - _Requirements: 10.4_

  - [ ]* 7.4 Write property test for formatters in `hearth/frontend/src/__tests__/properties.test.ts`
    - **Property 13: Number Formatter Correctness** — `fc.integer({ min: 0, max: 10_000_000 })`, assert `formatNumber` output matches `/^\d{1,3}(,\d{3})*$/` and `formatCurrency` output matches `/^\$\d{1,3}(,\d{3})*$/` — `// Feature: hearth-platform, Property 13`
    - Use `{ numRuns: 100 }`
    - _Requirements: 10.4_

  - [x] 7.5 Create `hearth/frontend/src/utils/motionVariants.ts`
    - Export `fadeInUp`, `slideUp`, `staggerChildren`, `cardReveal` Framer Motion variant objects as defined in the design
    - _Requirements: 6.3, 6.5, 11.2, 11.3_

- [x] 8. Frontend: global styles and entry point
  - [x] 8.1 Create `hearth/frontend/src/index.css`
    - Add Tailwind `@tailwind base/components/utilities` directives
    - Import Google Fonts: Playfair Display, Inter (via `@import url(...)`)
    - Define `.glass` utility class: `backdrop-blur-md bg-white/10 border border-white/20`
    - Define `.glass-card` extending `.glass` with `rounded-3xl` and padding
    - Define `.reveal` class for scroll-triggered opacity transitions
    - _Requirements: 12.1, 12.2, 12.3_

  - [x] 8.2 Create `hearth/frontend/src/main.jsx`
    - Validate all three env vars at startup: `VITE_MAPBOX_TOKEN`, `VITE_API_URL`, `VITE_WS_URL`; `console.warn` with variable name for each that is absent or empty
    - Render `<App />` into `#root`
    - _Requirements: 14.4_

- [ ] 9. Frontend: hooks
  - [ ] 9.1 Implement `useWebSocket` hook in `hearth/frontend/src/hooks/useWebSocket.js`
    - Accept `{ url, onMessage, onConnectionError }` options
    - Manage connection state machine: `connecting → connected → reconnecting → error`
    - On `onmessage`: parse JSON, ignore `{type:'pong'}` messages, call `onMessage(data)` for broadcast payloads
    - Implement heartbeat: `setInterval` every 30s to send `{type:'ping'}`; if no pong within 5s, close socket and trigger reconnect
    - Implement exponential backoff: `delay = Math.min(1000 * 2 ** retryCount, 30000)`, up to 5 retries; after 5 failures set status to `"error"` and call `onConnectionError()`
    - Export `{ connectionStatus }`
    - Expose `computeBackoffDelay(n)` as a named export for testing
    - Clean up socket and intervals in `useEffect` teardown
    - _Requirements: 4.4, 4.5, 4.6_

  - [ ]* 9.2 Write property test for backoff formula in `hearth/frontend/src/__tests__/properties.test.ts`
    - **Property 12: Exponential Backoff Delay Formula** — `fc.integer({ min: 1, max: 5 })`, assert `computeBackoffDelay(n) === Math.min(1000 * Math.pow(2, n-1), 30000)` — `// Feature: hearth-platform, Property 12`
    - Use `{ numRuns: 100 }`
    - _Requirements: 4.4_

  - [ ] 9.3 Implement `useCountUp` hook in `hearth/frontend/src/hooks/useCountUp.js`
    - Accept `{ target, duration, trigger }` where `trigger` is a boolean that starts the animation
    - Use `requestAnimationFrame` with ease-out interpolation to count from 0 to `target` over `duration` ms
    - Include a `useRef<boolean>` `once` guard — never re-animate after first completion
    - Return current animated value
    - _Requirements: 9.2, 10.2_

- [ ] 10. Frontend: Navbar, Hero, and ScrollReveal components
  - [ ] 10.1 Implement `hearth/frontend/src/components/Navbar.jsx`
    - Render nav links (logo + "See the Live Map" anchor)
    - Use Framer Motion `motion.nav` + `useScroll`; interpolate `backgroundColor` from `rgba(0,0,0,0)` to `rgba(15,23,42,0.95)` over first 100px of scroll
    - On page load, nav must be fully transparent (Req 5.7)
    - _Requirements: 5.7, 6.2_

  - [ ] 10.2 Implement `hearth/frontend/src/components/Hero.jsx`
    - Full-viewport `<section>` (h-screen) with `background-color: #0f172a` always applied; `background-image: url('/hero-bg.jpg')` layered on top with `bg-cover bg-center`
    - Dark gradient overlay: `linear-gradient(rgba(0,0,0,0.6), rgba(0,0,0,0.6))`
    - Headline: "Every family deserves a second roof." using `font-display`
    - Subheadline: "Hearth predicts homelessness 4 weeks early. Before the eviction. Before the street. Before it's too late."
    - Stat line: "653,000 people experienced homelessness last year. 1 in 5 was a child."
    - CTA button "See the Live Map": pill-shaped, `animate-pulse-slow`, `onClick` smooth-scrolls to `#map-section`
    - Bounce arrow at bottom of viewport: `animate-bounce`, `onClick` smooth-scrolls to next section
    - Hero opacity tied to `useScroll` via `useTransform(scrollY, [0, vh], [1, 0])`
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.8, 5.9, 5.10_

  - [ ] 10.3 Implement `hearth/frontend/src/components/ScrollReveal.jsx`
    - Wrap children in `motion.div` with `initial={{ opacity: 0, y: 30 }}`, `whileInView={{ opacity: 1, y: 0 }}`, `viewport={{ once: true, amount: 0.2 }}`, `transition={{ duration: 0.5, ease: 'easeOut' }}`
    - `once: true` ensures no re-trigger on scroll back (Req 6.6)
    - _Requirements: 6.3, 6.5, 6.6_

- [ ] 11. Frontend: Map component
  - [ ] 11.1 Implement `hearth/frontend/src/components/Map.jsx`
    - Initialize `mapboxgl.Map` in `useEffect` (run once); use `VITE_MAPBOX_TOKEN`; center `[37.7749, -122.4194]`; zoom 11
    - After map `load` event: add GeoJSON `FeatureCollection` source `"blocks"` from initial block data; add circle layer `"blocks-layer"` with data-driven `circle-color` using `["match", ["get", "risk_level"], ...]` paint expression mapping all 5 levels to hex colors
    - On WebSocket update: call `map.getSource("blocks").setData(newGeoJSON)` to update markers in-place within 500ms
    - Critical pulse: maintain `Set<string>` of critical block IDs; for each critical marker render an absolutely-positioned overlay `<div>` with `animation: markerPulse 1.5s ease-in-out infinite`; remove when level changes away from `critical`
    - Hover popup: `map.on("mouseenter", "blocks-layer", ...)` → `new mapboxgl.Popup(...)` with block name + score; `map.on("mouseleave", ...)` → `popup.remove()`
    - Click selection: `map.on("click", "blocks-layer", ...)` → call `onSelectBlock(feature.properties)`
    - Accept `blocks` prop (from WS feed) and `onSelectBlock` callback
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6_

- [ ] 12. Frontend: BlockDetails (Sidebar) component
  - [ ] 12.1 Implement `hearth/frontend/src/components/BlockDetails.jsx`
    - Glassmorphism card: `backdrop-blur-md bg-white/10 border border-white/20 rounded-3xl` (`.glass-card`)
    - Display block name, numeric `risk_score` (1 decimal), and `risk_level` badge colored with `RISK_COLORS`
    - Four animated progress bars (Framer Motion `motion.div` width `0% → N%` on mount/update) for: `evictions` (max 100), `wait_days` (max 365), `calls` (max 200), `encampment` (max 10)
    - "Dispatch Outreach Team" button: pill-shaped (`rounded-full`); `isDispatching` state disables button and shows `animate-spin` spinner during in-flight call
    - On click: `POST /api/dispatch/{block.id}` via `fetch`; on success: show success toast for 3s, call `onDispatchSuccess()` callback; on error: show error toast for 3s, re-enable button
    - Toasts: absolute-positioned `motion.div` with auto-dismiss `setTimeout`
    - Hover on button: scale up + elevated drop shadow within 300ms (Req 12.4)
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.8, 12.3, 12.4_

  - [ ] 12.2 Create `hearth/frontend/src/components/Sidebar.jsx`
    - Thin wrapper that renders `<BlockDetails block={selectedBlock} onDispatchSuccess={onDispatchSuccess} />` or an empty/placeholder state when no block is selected
    - Apply responsive width: 35% on ≥1024px, full width on smaller viewports
    - _Requirements: 6.4, 12.5_

- [ ] 13. Frontend: StatsSection and CostCalculator
  - [ ] 13.1 Implement `hearth/frontend/src/components/StatsSection.jsx`
    - Three metric cards with labels and initial values: "Families Helped" (47), "Money Saved" ($1,950,000), "Blocks Monitored" (16, static)
    - Use `useCountUp` hook for "Families Helped" and "Money Saved"; triggered once when section enters viewport via `ScrollReveal`
    - Accept `familiesHelped` and `moneySaved` props from App; increment animations on prop change (500ms ease-out)
    - Format values using `formatNumber` and `formatCurrency` from utils
    - _Requirements: 10.1, 10.2, 10.3, 10.4_

  - [ ] 13.2 Implement `hearth/frontend/src/components/CostCalculator.jsx`
    - Two labeled bars: prevention ($3,500) at `(3500/45000)*100 ≈ 7.78%` width; emergency ($45,000) at 100% width; proportional sizing satisfies Req 9.1
    - Use `useCountUp` for dollar amounts (0→target over 1.5–2s); `once` guard prevents re-trigger
    - Display savings callout: "Emergency response costs ~12.9× more than prevention ($45,000 ÷ $3,500)"
    - Accept `totalSaved` prop; animate increment by $41,500 per dispatch over 500ms
    - Wrap in `ScrollReveal` so animation fires at 20% viewport entry
    - _Requirements: 9.1, 9.2, 9.3, 9.4_

- [ ] 14. Frontend: SuccessStories and Footer
  - [ ] 14.1 Implement `hearth/frontend/src/components/SuccessStories.jsx`
    - Render exactly 4 story cards with the content from Req 11.1:
      1. "James and Maya — Outreach prevented eviction, family remains housed"
      2. "Willow Avenue — 3 families connected to rental assistance"
      3. "Riverside Block — Shelter waitlist reduced by 45 days after intervention"
      4. "Oak Grove — Mental health services connected, risk dropped from 82 to 24"
    - Use Framer Motion `variants` with `staggerChildren: 0.1` and `cardReveal` variant for each card (fade in + slide up 20px, 400ms ease-out)
    - Wrap section in `ScrollReveal`; `once: true` on the stagger container
    - _Requirements: 11.1, 11.2, 11.3_

  - [ ] 14.2 Implement `hearth/frontend/src/components/Footer.jsx`
    - Display copyright, platform name "Hearth", tagline "Predict. Prevent. Protect."
    - Apply `font-body` and dark/navy background to match design system
    - _Requirements: 12.1, 12.2_

- [ ] 15. Frontend: App root component
  - [ ] 15.1 Implement `hearth/frontend/src/App.jsx`
    - Declare top-level state: `selectedBlock`, `familiesHelped` (init 47), `moneySaved` (init 1,950,000), `totalSaved` (init 0)
    - Call `useWebSocket({ url: VITE_WS_URL, onMessage: handleWsMessage })` to receive live block updates; store latest `blocks` array in state
    - Implement `handleWsMessage(payload)` — update `blocks` state from `payload.blocks`; if `selectedBlock` is set, update it from the new data
    - Implement `handleDispatchSuccess()` — increment `familiesHelped` by 1, `moneySaved` by 3,500, `totalSaved` by 41,500; call `triggerConfetti()`
    - Implement `triggerConfetti()` — import and call `canvas-confetti` with a celebratory config
    - Compose layout: `<Navbar>` + `<Hero>` + `<section id="map-section">` containing `<Map>` (65% width, ≥1024px) and `<Sidebar>` (35%) + `<StatsSection>` + `<CostCalculator>` + `<SuccessStories>` + `<Footer>`
    - Pass `selectedBlock`, `onSelectBlock`, `onDispatchSuccess`, `familiesHelped`, `moneySaved`, `totalSaved` as props to relevant children
    - Apply responsive layout classes: single-column ≤767px, two-column 768–1199px, 65/35 split ≥1200px
    - _Requirements: 6.4, 7.6, 8.6, 8.7, 9.4, 10.3, 12.5_

- [ ] 16. Frontend: property tests (WS and dispatch)
  - [ ]* 16.1 Write property tests in `hearth/frontend/src/__tests__/properties.test.ts`
    - **Property 7: Fetch Block by Valid ID Returns Correct Block** — `fc.constantFrom(...VALID_BLOCK_IDS)`, async property: GET block, assert HTTP 200 + `data.id === id` + required fields present — `// Feature: hearth-platform, Property 7` (`{ numRuns: 16 }`)
    - **Property 8: Unknown Block ID Returns 404 (frontend)** — `fc.string().filter(s => !VALID_BLOCK_IDS.includes(s))`, assert HTTP 404 + non-empty `error` field — `// Feature: hearth-platform, Property 8`
    - **Property 10: Broadcast Payload Completeness** — mock WS server sends N payloads; assert each has `blocks.length === 16`, valid `timestamp` string, each block has `id`, `risk_score` in [0,100], `risk_level` in valid set — `// Feature: hearth-platform, Property 10` (`{ numRuns: 100 }`)
    - **Property 11: WebSocket Disconnection Isolation** — spin up mock WS server with N≥2 clients; disconnect a subset; assert remaining clients still receive broadcasts — `// Feature: hearth-platform, Property 11`
    - _Requirements: 3.2, 3.3, 4.2, 4.3_

- [ ] 17. Checkpoint — Frontend feature-complete
  - Run `vitest --run` in `hearth/frontend`; all tests should pass. Manually verify the Vite dev server proxies to the backend. Ask the user if questions arise before proceeding.

- [x] 18. Docs and environment files
  - [x] 18.1 Create `hearth/.env.example`
    - Include exactly three entries with inline comments and placeholder values:
      ```
      VITE_MAPBOX_TOKEN=pk.your_token_here   # Mapbox public token for map rendering
      VITE_API_URL=http://localhost:8000      # Backend REST API base URL
      VITE_WS_URL=ws://localhost:8000         # Backend WebSocket base URL
      ```
    - _Requirements: 14.1_

  - [x] 18.2 Create `hearth/README.md`
    - Numbered setup steps covering: clone repo, install Python deps (`pip install -r backend/requirements.txt`), install Node deps (`cd frontend && npm install`), copy `.env.example` to `.env` and fill in values, place `hero-bg.jpg` at `frontend/public/hero-bg.jpg`, start backend (`uvicorn main:app --reload`), start frontend (`npm run dev`)
    - Each step includes the exact command or file path
    - _Requirements: 15.1_

  - [x] 18.3 Create `hearth/PITCH.md`
    - 400–500 words divided into three named sections:
      - **Problem**: homelessness crisis scale, current reactive cost model ($45k emergency vs $3.5k prevention)
      - **Solution**: Hearth's predictive approach, risk engine, real-time monitoring, dispatch workflow
      - **Impact**: quantified metrics (16 blocks, 47 families helped, $1.95M saved), call to action
    - _Requirements: 15.2_

  - [x] 18.4 Create `hearth/SUBMISSION.md`
    - Checklist verifying presence of: `backend/main.py`, `backend/risk_engine.py`, `backend/simulator.py`, `backend/requirements.txt`, `backend/render.yaml`, all frontend component files, `.env.example`, `vercel.json`, `README.md`, `PITCH.md`, `frontend/public/hero-bg.jpg` (placeholder note)
    - Each item independently verifiable by file tree inspection
    - _Requirements: 15.3_

- [ ] 19. Final checkpoint — All done
  - Run `pytest hearth/backend/tests/` and `vitest --run` in `hearth/frontend`; confirm all property tests pass and no regressions. Ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP build
- Each task references specific requirements for full traceability
- Property tests are embedded as sub-tasks immediately after the implementation they validate — catching regressions early
- Tasks 18.1–18.4 (docs) are independent of implementation and can be worked on in parallel with tasks 6–16
- The Mapbox token (`VITE_MAPBOX_TOKEN`) must be provided before the Map component can render; use a free Mapbox account
- The `hero-bg.jpg` is developer-supplied and not generated by code — a placeholder or stock photo is acceptable for the submission

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "6.1", "6.2", "6.3", "6.4", "6.5", "18.1", "18.2", "18.3", "18.4"] },
    { "id": 1, "tasks": ["1.2", "2.1", "7.1", "7.2", "7.3", "7.5", "8.1", "8.2"] },
    { "id": 2, "tasks": ["2.2", "3.1", "7.4", "9.1", "9.3"] },
    { "id": 3, "tasks": ["2.2", "3.2", "4.1", "4.2", "9.2", "10.1", "10.2", "10.3"] },
    { "id": 4, "tasks": ["11.1", "12.1", "13.1", "13.2", "14.1", "14.2"] },
    { "id": 5, "tasks": ["12.2", "15.1"] },
    { "id": 6, "tasks": ["16.1"] }
  ]
}
```
