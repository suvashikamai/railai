<div align="center">

<img src="docs/screenshot-splash.png" alt="RailAI" width="560" />

# RailAI

**AI-Powered Automatic Block Planning & Asset Availability Platform for Indian Railways**

Problem statement **PSC26027** — *AI-Powered Automatic Block Planning to Maximize Asset Availability for Train Operations on Indian Railways*

</div>

---

> ### ⚠️ What this is, and what it is not
>
> RailAI is a **decision-support and simulation platform**. It does **not** control signalling, points,
> interlocking or train movements, and it is not connected to any live railway system. Every
> recommendation it produces is placed in front of a human controller, who authorises or rejects it —
> nothing changes the plan without a recorded authorisation.
>
> All operational data in this repository is **simulated**. Train numbers and station names resemble
> real Indian Railways services because that makes the demonstration legible; the timings, delays,
> consists and asset conditions are synthetic.
>
> Production deployment would require integration through approved Indian Railways signalling, control
> and safety systems, after the validation, certification and authorisation those systems demand.

---

## The problem, and what RailAI actually does about it

A section controller has to decide, continuously, which train occupies which block section and when —
around delays arriving from neighbouring divisions, engineering possessions booked days earlier, and
assets that fail without warning. Every one of those decisions trades one service's punctuality against
another's, and against the maintenance window that keeps the asset alive.

RailAI is **not a train-tracking app**. It re-plans the corridor.

```
DISRUPTION → DETECTION → RE-PLAN → RANKED OPTIONS → HUMAN AUTHORISATION → UPDATED PLAN → MEASURED RESULT
```

Given the timetable, priorities, block topology, possessions, failures and speed restrictions, it
allocates every service an exclusive, headway-separated occupancy of each section on its route, detects
every point where a train would have to stand, and searches over the three levers a controller actually
has — **route it differently, regulate it, or move the possession** — for the plan with the lowest
operating cost.

---

## Quick start

```bash
git clone <your-repo-url> railai
cd railai
npm install
npm run dev          # http://localhost:5173
```

```bash
npm run build        # production build into dist/
npm run preview      # serve the production build
npm run test:engine  # planner invariants + a worked failure scenario, in the terminal
```

No backend, no database, no API keys. The optimiser runs in the browser. `npm run build` produces a
static `dist/` that deploys to Vercel, Netlify, GitHub Pages or any static host with no configuration.

---

## What is in the box

| Screen | What it does |
|---|---|
| **Splash** | Three-second branded opening sequence |
| **Sign-in** | Five roles with different authority — only Control Room and Divisional Operations Manager can authorise a plan |
| **Control Dashboard** | KPIs, live corridor map, plan quality index with its full breakdown, live alerts, and the demo controls |
| **Live Network** | Block-by-block state, section inspector, live train positions, platform demand |
| **AI Block Planner** | Path allocation for every service, leg-by-leg regulation record, detected conflicts, **ranked resolution options**, block-occupancy chart |
| **Asset Availability** | Condition, 30-day failure probability, and a transparent breakdown of *why* each health index reads what it does |
| **Maintenance Planner** | Two-stage least-disruptive possession window search, costed by full re-plan |
| **What-If Simulator** | Inject failures, delays, speed restrictions, emergency possessions or cancellations; compare unmanaged outcome against the optimiser's recovery |
| **Control Copilot** | Questions answered from the live plan, every answer citing its source |
| **Analytics** | Punctuality, utilisation, priority-band performance, seven-day trend, and AI-vs-unaided-desk comparison |
| **Alerts** | Conflicts, predicted failures, congestion — plus the authorisation audit trail |

---

## How the planner works

### The network model

A compressed but structurally honest model of the New Delhi – Lucknow corridor: **10 stations, 22 block
sections, 25 services, 16 tracked assets**, served by two competing routes plus a goods avoiding line and
the Unnao chord.

```
                    HPU ──── MB ──── BE ────────────┐
                   ╱ (single line)                   ╲
        NDLS ── GZB                                   LKO
                   ╲                                 ╱  ╲
                    ALJN ── TDL ── ETW ── CNB ───────┘    (Unnao chord)
                                ╲______╱
                              (avoiding line)
```

Each edge is an **absolute block section aggregated to inter-station granularity**. Real signalling
divides these into many short absolute blocks; the constraint structure is identical at either
granularity, so the planner ports directly onto real block data.

Sections carry a signalling regime, and the scheduler respects the difference:

- **Absolute block** (single line, avoiding lines, the chord — capacity 1): one movement at a time.
  A following move waits for the section to clear plus headway; an opposing move additionally needs the
  line-clear exchange time.
- **Automatic block** (the double-line main — capacity 2–3): trains may follow each other through the
  section. What is forbidden is entering closer than headway behind the train ahead, closing up on it at
  the exit (catching or overtaking inside the section), and exceeding the number of absolute blocks the
  section holds.

### The scheduler

`src/engine/scheduler.js` — a **priority list scheduler** with exact block-occupancy constraints.
Services are ordered by operating priority, then booked departure; each is walked leg by leg, taking the
earliest conflict-free slot on each section. Where a service would have to stand, a conflict is recorded
with what was in the way and for how long.

**Hard constraints (never violated, never traded):**

- exclusive occupancy of a section, or a permitted following move under automatic block
- minimum headway between successive movements
- full separation for opposing moves on single line
- possessions and failed assets block the section absolutely
- no movement is planned into a booked possession
- a hold longer than 120 minutes is refused — the planner will not silently invent a two-hour detention.
  It marks the service **unpathed** and raises an alert, because that is a decision for a human
  (cancel, short-terminate, or extend the possession), not for software.

**Objective (what the optimiser minimises):**

```
cost = 1.0 × priority-weighted delay      // a Rajdhani minute costs 6× a goods minute
     + 0.3 × total delay                  // so low-priority delay still counts
     + 3.0 × minutes a possession must move
     + 2.0 × regulation events            // each one is a manual controller action
     + 400 × services left without a path
```

### The optimiser

`optimise()` runs a local search over the levers a controller actually has, in four passes:

0. **No-path repair** — a service with nowhere to go matters more than any amount of delay. Try every
   alternative path, then try moving whichever movable possession is in the way.
1. **Reroute** the worst-delayed services onto alternative paths with the same endpoints.
2. **Regulate** the services that are *blocking* others, to change the crossing order.
3. **Move** movable possessions off contended windows.

Every candidate is **fully re-scheduled** and accepted only if the total cost strictly improves. The
returned plan is therefore never worse than the input, and the search log records what each accepted move
did to the objective.

**Nothing here is random.** The same scenario in produces the same plan out — a controller must be able
to reproduce and audit any recommendation the system makes.

### Ranked conflict resolution

For any detected conflict, `resolutionOptions()` builds every resolution a controller could authorise —
regulate the waiting service, reverse the crossing order, route either service a different way, move the
possession — **re-plans the entire corridor under each**, and ranks them by resulting cost. Each option
carries the measured network delay, worst single delay, punctuality and conflict count, plus a note on
what authorising it would actually require operationally. Statutory possessions are shown as unavailable
rather than silently omitted.

### Predictive maintenance

`src/engine/predictive.js`. Asset health is a published, auditable decay model, not a black box:

```
health = 100 − (wear index × days since attention × k)   // time
              − (usage index × movements today)          // usage, capped
              − (years in service × k)                   // age, capped

P(failure within 30 days) = logistic((health − 45) / 9)
```

The movement count comes from the **live block plan**, so an asset's condition responds to how hard the
network is actually working it. The UI shows the three contributions separately, so any health figure can
be traced to its causes. In production the decay terms are replaced by models fitted on the asset failure
register and condition-monitoring telemetry; nothing above the model changes.

**Least-disruptive window search** runs in two deliberate stages: a cheap sweep slides the possession
across the permitted window in 15-minute steps, scoring each position by the priority-weighted movements
it would interrupt; then the best few candidates are **exactly evaluated** by re-planning the whole
corridor with the possession booked. The reported delay cost is measured, not estimated from gap size.

### The Copilot — stated plainly

**It is not a language model.** It is a deterministic intent parser over the live operational state.
Every sentence it returns is assembled from numbers the planner computed this session, and every answer
names its source, so a controller can verify any statement on the relevant screen. The interface
(question in, grounded answer + sources out) is the same shape an LLM-backed copilot would expose, which
leaves a clean seam for attaching one later using these same accessors for retrieval.

---

## Reading the numbers honestly

**Read priority-weighted delay before raw total delay.** A first-come-first-served desk can post a
*lower* raw delay total simply by letting whichever service turned up first take the section — including
a goods rake ahead of a Rajdhani. The optimiser deliberately pushes delay down the priority order, so its
raw total can be higher while every figure that matters operationally improves.

This is why the comparison panels lead with weighted delay, punctuality and worst-single-delay, and why
the app says so on screen rather than burying it.

The **plan quality index** (0–100) is a *readability* figure, not the objective. It is a fixed, published
weighting — 45% punctuality, 20% average delay, 20% asset availability, 15% freedom from regulation — so
a controller can see why a plan scores what it does. The optimiser minimises `planCost`, not the index.

The **passenger-minutes** figure assumes 60 passengers per coach on coaching services and none on
freight. That is an assumption, not a measurement, and the UI labels it as one. Replace it with
reservation and UTS load factors before quoting it anywhere that matters.

---

## The demonstration

1. Open the **Control Dashboard**. 25 services planned, the corridor running, the plan quality index and
   its breakdown visible.
2. Press **⚡ Simulate asset failure** — OHE section OHE-B11 fails, taking the Kanpur–Lucknow main line
   out for 70 minutes. This is the section the asset model flagged as the highest failure risk on the
   register, and it is the only main-line path into Lucknow.
3. Watch the map, the KPIs and the alerts update. Delay climbs; services stack behind the failed section.
4. Press **▶ Run AI optimisation**. The optimiser evaluates candidate plans and returns a recommendation:
   several services onto alternative paths (including the Unnao chord), a few regulated to change the
   crossing order, a possession moved.
5. Read the **measured effect** panel — every figure is the result of a real re-plan, not an estimate.
6. Press **✓ Authorise & apply**. The plan changes, and the decision is recorded in the authorisation
   trail on the Alerts screen.
7. Open **Analytics** to see the AI plan against an unaided first-come-first-served desk.

For the conflict-by-conflict story, go to the **AI Block Planner**, press **Resolve** on any conflict, and
read the ranked options — each one a full re-plan with its own measured consequences.

For a specific hypothesis, use the **What-If Simulator**: it runs two complete re-plans side by side —
what happens if nothing is done, and what the optimiser can recover.

---

## Verification

`npm run test:engine` asserts the invariants that matter operationally, not just that the code runs:

```
possessions and failures are exclusive
headway respected at section entry
no catching or overtaking inside a section
opposing moves on single line fully separated
section capacity never exceeded
each train traverses its route in order
no movement inside a booked possession
every service pathed within the operating day
optimiser cost is non-increasing
optimised plan beats the unaided baseline
planner is deterministic
```

plus a worked failure scenario, the ranked resolution options it produces, asset condition ranges, the
maintenance window search, alert generation and the Copilot's answers.

---

## Architecture

```
src/
├── data/           the modelled world — pure data, no logic
│   ├── network.js     stations, 22 block sections, routes, timing constants
│   ├── trains.js      25 services, priority classes, weights
│   └── assets.js      16 assets, booked possessions, maintenance requests
├── engine/         all the intelligence — pure functions, no React
│   ├── scheduler.js   constraint model, list scheduler, optimiser, resolutions
│   ├── predictive.js  asset condition, possession window search
│   ├── analytics.js   metrics, alerts, plan comparison, trend series
│   └── copilot.js     deterministic query engine over plan state
├── store/          React state — scenario in, plan out
├── components/     shell, logo, map, Gantt, UI primitives
├── pages/          the nine screens
└── styles/         one stylesheet, design tokens at the top
```

The engine has **no React dependency**. It is a set of pure functions from a serialisable scenario to a
costed plan, which is why `npm run test:engine` can exercise it in plain Node.

### Replacing the solver

The constraint set implemented in `schedule()` — exclusive/following occupancy, headway, opposing
clearance, immovable possessions — is exactly the constraint set of a job-shop no-overlap CP model. The
list scheduler is the standard fast heuristic for that model, and **everything outside it treats the
scheduler as a black box that maps a scenario to a costed plan**.

To move to a production-grade solver, implement the same signature with OR-Tools CP-SAT (`NoOverlap` on
block intervals, `AddNoOverlap2D` for capacity, optional intervals for route choice) or a MILP, and
return the same plan shape. The UI, the conflict resolver, the simulator, the maintenance planner and the
copilot need no changes.

The natural production topology is React frontend → FastAPI service → CP-SAT optimiser → PostgreSQL, with
the simulated dataset replaced by TMS/COA feeds, the asset register and condition-monitoring telemetry.
The scenario object in `baseScenario()` is already the API contract for that boundary.

---

## Design

Dark control-room theme: deep navy ground, Indian Railways saffron and green from the mark, amber/red/violet
for section states following signalling convention. Numbers are set in a monospace face with tabular
figures so columns align and values do not jitter as the clock runs. The mark is drawn as vector — the
tricolour arch cradling a semi-high-speed trainset — so it stays crisp from a 16px favicon to the
full-screen splash. `docs/logo-reference.jpg` is the reference image the mark was designed from.

Desktop-first, as a control-room interface should be, and responsive down to tablet and phone where the
navigation collapses to an icon rail.

---

## Licence

MIT — see [LICENSE](LICENSE).

Built as a prototype for Smart India Hackathon problem statement PSC26027.
