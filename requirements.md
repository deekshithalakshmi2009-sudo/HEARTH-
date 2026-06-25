# Requirements Document

## Introduction

Hearth is a full-stack homelessness prevention platform with the tagline "Predict. Prevent. Protect." The system uses real-time data simulation and a predictive risk engine to identify city blocks at risk of homelessness-related crises up to 4 weeks in advance. A React/Vite frontend provides a cinematic, scroll-driven experience with a live Mapbox map, enabling city outreach coordinators and policymakers to dispatch aid before crises escalate. The backend is a Python FastAPI service with WebSocket support that continuously simulates and broadcasts risk scores for 16 San Francisco city blocks.

## Glossary

- **Hearth**: The full-stack homelessness prevention platform described in this document.
- **RiskEngine**: The backend module that computes a composite risk score (0–100) for each city block based on weighted input factors.
- **CitySimulator**: The backend module that maintains state for 16 simulated SF city blocks and updates their risk factors via random walk every 3 seconds.
- **Block**: A named geographic area (e.g., "Downtown", "Riverside") representing a unit of risk monitoring within the platform.
- **Risk Score**: A numeric value from 0 to 100 representing the likelihood of a homelessness-related crisis on a given Block, computed by the RiskEngine.
- **Risk Level**: A categorical label derived from a Risk Score — one of: `critical` (80–100), `high` (60–79), `medium` (40–59), `low` (20–39), or `safe` (0–19).
- **Outreach Dispatch**: An action taken by a coordinator to send an outreach team to a Block, triggered via the platform UI.
- **WebSocket Feed**: The real-time data channel (`/ws/live`) over which the backend broadcasts updated Block states every 3 seconds.
- **Glassmorphism**: A UI design style using frosted-glass appearance: `backdrop-filter: blur(12px)`, semi-transparent background (`rgba(255,255,255,0.1)`), and a 1px semi-transparent border.
- **Hero Section**: The full-viewport landing section displayed on page load before the user scrolls.
- **Cinematic Reveal**: A scroll-triggered animation sequence in which the Hero Section fades out and the Map Section slides up into view.
- **ScrollReveal**: A React component or utility that triggers entrance animations when elements enter the viewport during scroll.
- **CostCalculator**: A UI component displaying the cost comparison between homelessness prevention ($3,500) and emergency response ($45,000).
- **Mapbox**: The third-party mapping library (Mapbox GL JS) used to render the live risk map.
- **Framer Motion**: The React animation library used for declarative scroll and interaction animations.
- **canvas-confetti**: A JavaScript library used to render a confetti burst animation on successful Outreach Dispatch.
- **Exponential Backoff**: A reconnection strategy in which retry delay doubles with each failed attempt.

---

## Requirements

### Requirement 1: Risk Score Computation

**User Story:** As a city outreach coordinator, I want each city block to have an up-to-date risk score, so that I can prioritize where to send resources.

#### Acceptance Criteria

1. THE RiskEngine SHALL compute a Risk Score for each Block using the following weights: eviction rate 40%, shelter waitlist length 30%, mental health incidents 20%, and encampment count 10%, where the weighted contributions are summed and clamped to [0, 100].
2. THE RiskEngine SHALL produce a Risk Score in the range [0, 100] inclusive for every valid set of input factors, including boundary inputs (all factors at minimum and all factors at maximum).
3. WHEN the RiskEngine computes a Risk Score, THE RiskEngine SHALL assign a Risk Level according to the following thresholds: `critical` for scores 80–100, `high` for 60–79, `medium` for 40–59, `low` for 20–39, and `safe` for 0–19. Boundary values (e.g., exactly 80, exactly 60) SHALL be assigned to the higher severity level.
4. THE RiskEngine SHALL map each Risk Level to a unique display color: `critical` → `#dc2626`, `high` → `#f97316`, `medium` → `#eab308`, `low` → `#84cc16`, `safe` → `#22c55e`. All five mappings SHALL be present and no two levels SHALL share a color.
5. FOR ALL valid input factor sets, calling `calculate()` twice with the same inputs SHALL produce the same Risk Score (deterministic property). For any Risk Score s, `get_level(s)` followed by any threshold check SHALL be consistent with the threshold table in criterion 3.

---

### Requirement 2: City Block Simulation

**User Story:** As a platform operator, I want the system to continuously simulate risk conditions across 16 city blocks, so that the platform can demonstrate real-time monitoring without requiring live city data feeds.

#### Acceptance Criteria

1. THE CitySimulator SHALL maintain state for exactly 16 Blocks with the following names: Downtown, Northside, Southside, East End, West Park, Riverside, Hilltop, Lakeside, Maplewood, Cedar Heights, Oak Grove, Brookline, Fairmont, Kingsley, Pleasant Hill, and Sunnydale.
2. THE CitySimulator SHALL assign each Block a geographic coordinate within the San Francisco latitude range [37.70, 37.82] and longitude range [-122.52, -122.36]. No two Blocks SHALL share identical coordinates.
3. WHEN the simulation tick fires, THE CitySimulator SHALL update each Block's numeric risk factors (eviction rate, shelter waitlist days, mental health incident count) by a random walk step bounded to [-10, +10] per factor, then clamp each factor to its valid range, and produce a new Risk Score by invoking the RiskEngine.
4. WHILE the CitySimulator is running, THE CitySimulator SHALL fire a simulation tick at intervals of approximately 3 seconds, measured as elapsed time since the previous tick completion.
5. WHILE the CitySimulator is running, THE CitySimulator SHALL clamp all Block Risk Scores to [0, 100] after each tick, such that no Block ever reports a Risk Score outside that range.
6. IF the RiskEngine raises an exception during a tick, THEN THE CitySimulator SHALL log the error, retain the Block's previous Risk Score, and continue processing the remaining Blocks in that tick.

---

### Requirement 3: REST API

**User Story:** As a frontend developer, I want a REST API to fetch block data, so that I can populate the map and sidebar on initial page load.

#### Acceptance Criteria

1. THE Backend SHALL expose a `GET /api/blocks` endpoint that returns the current state of all 16 Blocks as a JSON array, where each element contains the Block's id, name, coordinates, risk factors, Risk Score, and Risk Level.
2. THE Backend SHALL expose a `GET /api/blocks/{id}` endpoint that returns the current state of the single Block identified by `{id}` as a JSON object with the same fields as the array elements in criterion 1.
3. IF a `GET /api/blocks/{id}` request specifies a Block ID that does not exist in the CitySimulator, THEN THE Backend SHALL return an HTTP 404 response with a JSON body containing an `error` field whose value indicates the Block ID was not found.
4. THE Backend SHALL expose a `POST /api/dispatch/{id}` endpoint that triggers an Outreach Dispatch for the Block identified by `{id}` and returns an HTTP 200 response containing the Block's updated state (id, Risk Score, Risk Level, and all risk factor values) after the dispatch is applied.
5. IF a `POST /api/dispatch/{id}` request specifies a Block ID that does not exist in the CitySimulator, THEN THE Backend SHALL return an HTTP 404 response with a JSON body containing an `error` field whose value indicates the Block ID was not found.
6. THE Backend SHALL include CORS response headers on all `GET /api/*` and `POST /api/*` endpoints that permit cross-origin requests, including the `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods` (at minimum GET and POST), and `Access-Control-Allow-Headers` headers.
7. IF a `POST /api/dispatch/{id}` request body is malformed or contains unexpected fields, THEN THE Backend SHALL still process the dispatch using only the Block ID from the URL path and return an HTTP 200 response as described in criterion 4.

---

### Requirement 4: Real-Time WebSocket Feed

**User Story:** As a coordinator viewing the live map, I want block risk scores to update in real time without refreshing the page, so that I can react to emerging crises immediately.

#### Acceptance Criteria

1. THE Backend SHALL expose a WebSocket endpoint at `/ws/live` that accepts client connections and adds each connected client to an active connection pool.
2. WHILE a WebSocket client is connected, THE Backend SHALL broadcast a JSON payload every 3 seconds containing, for each of the 16 Blocks, at minimum: the Block identifier, current Risk Score, current Risk Level, and a UTC timestamp of the broadcast.
3. WHEN a WebSocket client disconnects, THE Backend SHALL remove the client from the active connection pool without raising an unhandled exception and without interrupting broadcast delivery to remaining connected clients.
4. IF a WebSocket connection fails or closes unexpectedly, THEN THE useWebSocket hook SHALL attempt to reconnect using Exponential Backoff starting with an initial delay of 1 second, doubling on each subsequent attempt, capped at a maximum delay of 30 seconds, up to a maximum of 5 retry attempts.
5. WHILE a WebSocket connection is active, THE useWebSocket hook SHALL send a heartbeat ping message to the backend every 30 seconds. IF no acknowledgment is received within 5 seconds of a ping, THEN THE useWebSocket hook SHALL treat the connection as failed and initiate the reconnection sequence described in criterion 4.
6. IF all 5 reconnection attempts are exhausted without a successful connection, THEN THE useWebSocket hook SHALL set its exported `connectionStatus` field to `"error"` and invoke an `onConnectionError` callback if one was provided by the consuming component.

---

### Requirement 5: Hero Section and Page Load Experience

**User Story:** As a first-time visitor, I want a visually compelling full-screen landing experience, so that I immediately understand the platform's purpose and urgency.

#### Acceptance Criteria

1. WHEN the page loads, THE Hero Section SHALL occupy exactly 100% of the viewport height and display a full-cover background image overlaid with a dark gradient whose opacity is 0.6, ensuring the image is visible but text is legible.
2. WHEN the page loads, THE Hero Section SHALL display the headline "Every family deserves a second roof." using the primary display typeface configured for the application.
3. WHEN the page loads, THE Hero Section SHALL display the subheadline "Hearth predicts homelessness 4 weeks early. Before the eviction. Before the street. Before it's too late." using the body typeface, rendered below the headline with a visible separation.
4. WHEN the page loads, THE Hero Section SHALL display a call-to-action button labeled "See the Live Map" that cycles through a repeating pulse animation with a period of 2 seconds, making the button visually distinct from static elements.
5. WHEN the page loads, THE Hero Section SHALL display the text "653,000 people experienced homelessness last year. 1 in 5 was a child." in a visually subdued style distinct from the headline and subheadline.
6. WHEN the page loads, THE Hero Section SHALL display a downward-pointing arrow at the bottom of the viewport that animates with a bouncing motion, pausing briefly at the bottom of each bounce, to indicate scrollable content below.
7. WHEN the page loads, THE Navbar SHALL be fully transparent so that the hero background image is visible through the navigation bar area.
8. WHEN the user clicks the "See the Live Map" CTA button, THE page SHALL scroll smoothly to the Map Section.
9. IF the hero background image fails to load, THEN THE Hero Section SHALL display a solid dark fallback background color so that the headline and subheadline remain legible.
10. WHEN the user clicks or activates the scroll indicator arrow, THE page SHALL scroll smoothly to the first section below the Hero Section.

---

### Requirement 6: Scroll-Driven Cinematic Reveal

**User Story:** As a visitor scrolling down the page, I want smooth, cinematic transitions between sections, so that the platform feels premium and engaging.

#### Acceptance Criteria

1. WHEN the user scrolls past the Hero Section, THE Hero Section SHALL fade out via an opacity transition completing within 300–500ms using an ease-out curve, such that it is no longer visible once the Map Section is fully in view.
2. WHEN the user scrolls past the Hero Section, THE Navbar SHALL transition from a fully transparent background to a solid dark background within 300ms using an ease-out curve, making the navigation links clearly legible against the background.
3. WHEN at least 20% of the Map Section enters the viewport, THE Map Section SHALL animate into view by sliding upward from an offset of 30px and fading in from opacity 0 to 1, completing within 500ms using an ease-out curve.
4. WHILE the viewport width is ≥1024px, THE Map Section SHALL occupy 65% of the available horizontal content width and THE Sidebar SHALL occupy the remaining 35%.
5. WHEN at least 20% of the StatsSection or SuccessStories section enters the viewport, THE ScrollReveal component SHALL trigger that section's entrance animation within one animation frame.
6. WHEN a section's entrance animation has completed, THE ScrollReveal component SHALL NOT re-trigger that animation if the user scrolls away and then scrolls back to the same section.

---

### Requirement 7: Live Risk Map

**User Story:** As a coordinator, I want to view all 16 blocks as color-coded markers on an interactive map, so that I can visually identify high-risk areas at a glance.

#### Acceptance Criteria

1. THE Map component SHALL render a Mapbox GL JS map centered on San Francisco at coordinates (37.7749° N, 122.4194° W) at an initial zoom level of 11, and SHALL display all 16 Block markers at their assigned coordinates on initial render.
2. THE Map component SHALL display each Block as a circle marker whose fill color corresponds to the Block's current Risk Level color (`critical` → red, `high` → orange, `medium` → yellow, `low` → light green, `safe` → green), and SHALL render all 16 markers simultaneously.
3. WHEN a WebSocket update payload is received, THE Map component SHALL update all affected marker colors to reflect their new Risk Level colors within 500ms of receiving the payload.
4. WHEN a Block marker's Risk Level becomes `critical`, THE Map component SHALL apply a repeating pulsing animation to that marker with a period of 1.5 seconds. WHEN the marker's Risk Level changes away from `critical`, THE pulse animation SHALL stop.
5. WHEN the user's pointer enters a Block marker, THE Map component SHALL display a popup containing the Block's name and its current numeric Risk Score. WHEN the pointer leaves the marker, THE popup SHALL close.
6. WHEN the user clicks a Block marker, THE Map component SHALL mark that Block as selected (visually distinguishing it from unselected markers) and SHALL invoke the `onSelectBlock` callback with the selected Block's data, causing the Sidebar to update.

---

### Requirement 8: Block Detail Sidebar

**User Story:** As a coordinator, I want to see detailed risk factor information for a selected block in a sidebar, so that I can understand the underlying causes of a high risk score.

#### Acceptance Criteria

1. WHEN a Block is selected, THE Sidebar SHALL display the Block's name, numeric Risk Score, and Risk Level label in a frosted-glass card with a blurred translucent background.
2. WHEN a Block is selected, THE Sidebar SHALL display four animated progress bars representing eviction rate, shelter waitlist days, mental health incident count, and encampment presence, where each bar fills from 0 to a value proportional to the factor's current reading on a 0–100 scale.
3. WHEN a Block is selected, THE Sidebar SHALL display a "Dispatch Outreach Team" button. WHILE an Outreach Dispatch API call is in flight, THE button SHALL be disabled and display a loading indicator to prevent duplicate submissions.
4. WHEN the user clicks the "Dispatch Outreach Team" button and no dispatch call is already in flight, THE Sidebar SHALL call the dispatch API for the selected Block's ID.
5. WHEN the Outreach Dispatch API call returns a success response, THE Sidebar SHALL display a success toast notification that remains visible for 3 seconds.
6. WHEN the Outreach Dispatch API call returns a success response, THE App SHALL trigger a confetti burst animation.
7. WHEN the Outreach Dispatch API call returns a success response, THE StatsSection "Families Helped" counter SHALL increment by 1.
8. IF the Outreach Dispatch API call returns an error response or times out, THEN THE Sidebar SHALL display an error toast notification describing the failure, and THE "Dispatch Outreach Team" button SHALL return to its enabled state.

---

### Requirement 9: Cost Calculator

**User Story:** As a policymaker, I want to see the financial impact of prevention vs. emergency response, so that I can justify investment in the Hearth platform.

#### Acceptance Criteria

1. THE CostCalculator component SHALL display two labeled bars: one representing a prevention cost of $3,500 and one representing an emergency response cost of $45,000. The bar lengths SHALL be proportional to their respective costs, so the emergency bar is visually approximately 12.9× longer than the prevention bar.
2. WHEN the CostCalculator enters the viewport (at least 20% visible), THE CostCalculator SHALL animate the numeric dollar amounts from $0 to their target values over a duration of 1.5–2 seconds using an ease-out curve. This animation SHALL fire only once per page load.
3. THE CostCalculator SHALL display a labeled savings callout stating that emergency response costs approximately 12.9× more than prevention, calculated as $45,000 ÷ $3,500, presented as a readable ratio to the user.
4. WHEN a dispatch is recorded (Requirement 8, criterion 6), THE CostCalculator SHALL update a running "Total Saved" counter by adding $41,500 (the difference between emergency and prevention cost) to the previous total, animating the increment over 500ms.

---

### Requirement 10: Statistics Section

**User Story:** As a visitor, I want to see key platform metrics, so that I understand the real-world impact of Hearth.

#### Acceptance Criteria

1. THE StatsSection SHALL display exactly three metric cards with the following labels and initial values: "Families Helped" starting at 47, "Money Saved" starting at $1,950,000, and "Blocks Monitored" fixed at 16.
2. WHEN the StatsSection enters the viewport, THE StatsSection SHALL animate each of the two numeric counters ("Families Helped" and "Money Saved") from 0 to their initial target values over 1–2 seconds. This animation SHALL fire once per page load and SHALL NOT re-trigger on subsequent viewport re-entries.
3. WHEN an Outreach Dispatch confirmation is received, THE StatsSection SHALL increment the "Families Helped" counter by 1 and add $3,500 to the "Money Saved" counter, animating each increment over 500ms.
4. THE StatsSection SHALL format "Families Helped" and "Blocks Monitored" as whole numbers with comma separators for values ≥1,000, and SHALL format "Money Saved" as a dollar-prefixed value with comma separators (e.g., $1,950,000).

---

### Requirement 11: Success Stories

**User Story:** As a visitor, I want to read real-feeling success stories, so that I can connect emotionally with the platform's mission.

#### Acceptance Criteria

1. THE SuccessStories component SHALL display exactly 4 story cards with the following content: (1) "James and Maya — Outreach prevented eviction, family remains housed", (2) "Willow Avenue — 3 families connected to rental assistance", (3) "Riverside Block — Shelter waitlist reduced by 45 days after intervention", (4) "Oak Grove — Mental health services connected, risk dropped from 82 to 24". Each card SHALL display a title and a one-sentence outcome description.
2. WHEN the SuccessStories section enters the viewport, each story card SHALL fade in and slide upward from 20px below its final position, completing within 400ms using an ease-out curve.
3. WHEN the SuccessStories section enters the viewport, THE ScrollReveal component SHALL stagger the 4 card entrance animations with a delay of 100ms between each consecutive card, so cards appear sequentially rather than simultaneously.

---

### Requirement 12: Visual Design System

**User Story:** As a product stakeholder, I want the platform's visual quality to match or exceed industry-leading products (Vercel, Linear, Playbook), so that Hearth communicates credibility and urgency.

#### Acceptance Criteria

1. THE App SHALL apply the primary display typeface to all h1–h6 heading elements and the body typeface to all paragraph, list item, label, and input text elements throughout the application.
2. THE App SHALL use a cream-toned background color for the main content areas, a dark navy color for primary text, and a warm amber color for interactive accents such as CTA buttons and highlights, as defined in the application's design token configuration.
3. THE App SHALL render all card components with a frosted-glass visual appearance: a blurred, translucent background allowing underlying content to show through, a semi-transparent border, and rounded corners with a radius of 24px.
4. THE App SHALL render all primary action buttons with a fully pill-shaped border (radius ≥ 40px). WHEN the user hovers over a primary button, THE button SHALL visibly scale up slightly and display an elevated drop shadow, completing the transition within 300ms.
5. THE App SHALL adapt its layout across three breakpoints: a single-column stacked layout for viewport widths ≤767px (mobile), a two-column stacked map/sidebar layout for widths 768–1199px (tablet), and the 65/35 map/sidebar split layout for widths ≥1200px (desktop).
6. IF the primary display typeface or body typeface fails to load from the external font provider, THEN THE App SHALL render text using a system-native serif fallback for headings and a system-native sans-serif fallback for body text, maintaining legibility.

---

### Requirement 13: Backend Deployment Configuration

**User Story:** As a developer deploying the platform, I want the backend to include a render.yaml deployment manifest, so that it can be deployed to Render with minimal configuration.

#### Acceptance Criteria

1. THE Backend SHALL include a `render.yaml` file that defines the following fields with non-empty string values: service name, runtime environment (Python), build command (pip install), and start command (uvicorn).
2. WHEN the backend process starts, THE Backend SHALL read the allowed CORS origin from the `FRONTEND_URL` environment variable and apply it to all CORS response headers. This allows the deployed frontend URL to be configured without modifying source code.
3. IF the `FRONTEND_URL` environment variable is absent at process startup, THEN THE Backend SHALL log a warning and fall back to allowing all origins (`*`), so the application remains functional during local development without requiring the variable to be set.

---

### Requirement 14: Frontend Environment Configuration

**User Story:** As a developer setting up the project locally, I want a documented environment variable template, so that I can configure the platform without reading source code.

#### Acceptance Criteria

1. THE Frontend project SHALL include a `.env.example` file containing exactly three entries — `VITE_MAPBOX_TOKEN`, `VITE_WS_URL`, and `VITE_API_URL` — where each entry includes an inline comment describing its purpose and a placeholder example value, so a developer can copy the file and fill in real values without reading source code.
2. WHEN a `.env` file is present with `VITE_MAPBOX_TOKEN` set to a non-empty string, `VITE_API_URL` set to a valid `http` or `https` URL, and `VITE_WS_URL` set to a valid `ws` or `wss` URL, THE Frontend SHALL connect to the Mapbox map and WebSocket feed using those values without any additional code changes.
3. THE Frontend project SHALL include a `vercel.json` configuration file that rewrites all non-asset URL paths to `index.html`, so that direct navigation to any application route (e.g., refreshing the page) returns the app shell instead of a 404 response.
4. IF any of `VITE_MAPBOX_TOKEN`, `VITE_WS_URL`, or `VITE_API_URL` is absent or empty at build time, THEN THE Frontend SHALL log a console warning identifying the missing variable by name, so developers are alerted to incomplete configuration before the app is served.

---

### Requirement 15: Project Documentation

**User Story:** As a developer onboarding to the project, I want complete setup and pitch documentation, so that I can run the project and present it to stakeholders without additional research.

#### Acceptance Criteria

1. THE Project SHALL include a `README.md` containing numbered setup steps that cover, in order: cloning the repository, installing backend Python dependencies, installing frontend Node dependencies, copying `.env.example` to `.env` and populating the required variables, placing the hero background image at `frontend/public/hero-bg.jpg`, starting the backend server, and starting the frontend dev server. Each step SHALL include the exact command or file path required.
2. THE Project SHALL include a `PITCH.md` containing a presentation script of 400–500 words divided into three named sections: "Problem" (the homelessness crisis and current cost model), "Solution" (the Hearth platform and its predictive approach), and "Impact" (quantified metrics and a closing call to action).
3. THE Project SHALL include a `SUBMISSION.md` containing a checklist that confirms the presence of the following deliverables: backend `main.py`, `risk_engine.py`, `simulator.py`, `requirements.txt`, and `render.yaml`; frontend component files; `.env.example`; `vercel.json`; `README.md`; `PITCH.md`; and a placeholder for the hero background image. Each checklist item SHALL be independently verifiable by inspecting the repository file tree.
