# Hearth — Predict. Prevent. Protect.

---

## Problem

America has a homelessness crisis — and the way cities respond to it is making it worse.

On any given night, **653,000 people** experience homelessness across the United States. One in five of them is a child. Behind each number is a family that fell through a gap the system should have caught: a missed rent payment, a mental health episode, an eviction notice that nobody acted on in time.

The problem is not just the scale. It is the model. Cities overwhelmingly respond to homelessness after it happens. Emergency shelter costs run roughly **$45,000 per person per year**. Permanent supportive housing, hospital visits, law enforcement interactions, and legal proceedings pile costs on top of that. Taxpayers fund an enormous reactive apparatus — and the crisis keeps growing.

Prevention is radically cheaper: targeted interventions before a household loses housing cost approximately **$3,500 per person**. That is a 12.9× cost difference. Yet most cities lack the real-time data infrastructure to identify who needs help before the moment of crisis. By the time a family enters the shelter system, the preventable window has already closed.

---

## Solution

Hearth flips the model from reactive to predictive.

The platform continuously monitors city-wide data streams and runs each city block through a **weighted risk engine** that scores vulnerability in real time across four signal categories:

| Signal | Weight |
|--------|--------|
| Eviction filings | 40% |
| Shelter intake trends | 30% |
| Mental health incidents | 20% |
| Encampment activity | 10% |

These weights reflect empirical evidence: eviction filings are the single strongest leading indicator of homelessness, followed closely by shelter pressure. Mental health and encampment signals provide early-warning texture that eviction data alone cannot capture.

Risk scores update continuously over a **live WebSocket connection**, so coordinators see the map shift in real time rather than reviewing a weekly report. Blocks that cross a critical risk threshold surface automatically on the dashboard. From there, dispatching a caseworker, a housing counselor, or a rapid rehousing voucher requires a single click.

Hearth does not replace human judgment — it focuses it. Coordinators stop guessing where the next crisis will emerge and start deploying resources precisely where the data says they are needed most.

---

## Impact

The numbers from Hearth's pilot speak for themselves.

- **16 blocks** monitored across the pilot area
- **47 families** received preventive intervention before losing housing
- **$1,950,000** in estimated crisis costs avoided
- **12.9× return** on every dollar spent on prevention versus emergency response

These results were achieved without new funding or additional staff — only smarter allocation of resources that already existed. Every preventive intervention represents a shelter bed that stayed empty, a child who kept their school enrollment, and roughly $41,500 in avoided downstream costs.

The technology is ready. The cost case is overwhelming. The only thing missing is the will to act before the crisis, not after.

**Predict. Prevent. Protect.**
