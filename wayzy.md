# Project Brief: Conversational Field-Sales Planning Engine (Proof of Concept)

## 1. Purpose of this document

This is the technical and functional brief for building a **Proof of Concept (POC)** — not a production product. The POC exists to prove exactly one thing convincingly: *a manager can describe a real-world disruption in plain language, watch the day's plan redraw itself instantly, and see a measurable before/after improvement in coverage.* Every feature in this brief is scoped against that single goal. Nothing here is meant to be a complete SFA platform.

---

## 2. Problem being demonstrated

Mid-market FMCG distributors in India run field sales on a fixed, repeating route (a "beat" or PJP — Permanent Journey Plan) — deliberately so. A retailer who knows their salesman comes every Tuesday holds their order for Tuesday; the routine itself has value, and the standing plan is usually fixed for 3–6 months on purpose. **This product does not propose rewriting that standing rhythm.** It targets the specific, recurring exception days that break it: a rep falls sick, a shop is shut, an urgent priority comes in — and today, fixing that exception is still a manual scramble.

A second, quieter problem sits underneath the first: **most companies don't actually know their real coverage number.** Visit adherence is typically self-reported, and self-reported adherence tends to come back looking near-perfect almost everywhere — including in territories where coverage is visibly falling on the ground. GPS-verified visit logging replaces that fiction with a real number, independent of anything AI-related.

### 2.1 What the competitive landscape actually looks like (corrected)

An earlier version of this brief claimed no competitor could do any of this and cited specific adherence/revenue-loss percentages. Neither claim held up under scrutiny, and this section replaces them with what's actually defensible:

- **BeatRoute already ships a multilingual conversational AI copilot** (English, Hindi, Bahasa) plus a Scheduling AI Agent and a Route Optimization AI Agent. **FieldAssist already markets dynamic route optimization** that adjusts for traffic, weather, urgent orders, and rep availability. **Botree already lets managers reassign outlets when someone's unavailable.** The "storm hits, redistribute the day" scenario is not a gap in the market — several vendors already describe it in their own marketing.
- **What is not shown in any public competitor material**: a manager issuing a natural-language instruction that *directly writes and commits* a new plan across the team. What exists instead is **recommend-and-review** — a button produces a ranked suggestion, a copilot answers questions and sends nudges, but a human still reviews and applies the change through the app's normal UI. The gap is specifically between **asking** (get a suggestion) and **instructing** (say what you want, it happens). That gap is real but narrow — plausibly closeable by an incumbent's engineering team in a quarter, since the conversational surface, scheduling engine, and data model already exist at BeatRoute specifically.
- **The 55%/85%-style adherence figures previously used here are not well-sourced and are dropped.** The defensible version of the coverage-data problem is qualitative, not a specific percentage: self-reported numbers are unreliable, and GPS-verified logging is a real, separable source of value regardless of the AI story.

Given this, the honest positioning is: **a narrower "instruct, not just recommend" gap in a genuinely crowded category, plus a verified-coverage-data angle that stands on its own.** This is a smaller, more defensible claim than the original pitch, and it should be validated directly — a live BeatRoute demo and a conversation with a working FMCG area sales manager would settle, definitively, whether a natural-language write-command already exists somewhere it hasn't been publicly documented.

The POC demonstrates the fix for that specific, narrow, exception-day gap — not a claim that the broader category is unserved.

---

## 3. What the POC must prove (must-have scope)

| # | Capability | Why it's in scope |
|---|---|---|
| 1 | **Conversational replanning core loop** | This is the hypothesis itself. A manager types (or WhatsApps) an instruction ("Rep X is out, redistribute his beat") and the system re-clusters zones and re-sequences routes instantly. |
| 2 | **Visit outcome logging** (visited/skipped + simple outcome tag) | Without this, no coverage number can be produced at all. |
| 3 | **Before/after coverage dashboard** | This is the actual deliverable — the pitch is a number, not a feature list. |
| 4 | **Real (or realistically modeled) outlet data for a real beat** | Synthetic/toy data doesn't convince a distributor who knows their own market. |
| 5 | **WhatsApp channel for instructions and visit logging** | Field research confirmed orders and instructions already flow through WhatsApp informally today — meeting managers and reps where they already are is a real adoption lever, not a nice-to-have, and it's a thin layer on top of the same AI pipeline (Section 6.4), not a separate build. |
| 6 | **Rep and outlet management console** (Section 8.2, 8.3) | A manager needs to see and manage who works for them and who they sell to as base functionality — this is standard in every SFA/DMS platform and everything else (zones, visits, goals) depends on this data existing and being editable. |
| 7 | **Quarterly/monthly goal setting and tracking** (Section 8.4) | Matches the "Goal-Driven SFA" pattern that's now standard across the category (BeatRoute, FieldAssist) — a manager needs to set and track targets, not just watch a live map. |

### Explicitly deferred (not built in this POC)

- Tally/Marg read-only connector
- Multilingual/voice conversational input (though most LLMs handle Hindi/regional text reasonably out of the box — worth testing informally, not engineering around)
- Order-booker vs. delivery-van role split
- Proactive staleness alerts, polished onboarding, production-grade offline sync
- Any write access to financial/inventory data — this system **never** touches money or stock

---

## 4. Technical architecture

```
┌──────────────────────┐  ┌─────────────┐        ┌──────────────────────────┐
│   Frontend (React)     │  │  WhatsApp    │◄──────►│   Backend (FastAPI)       │
│  Manager Dashboard      │  │  Cloud API   │  REST/ │                            │
│  + Rep Visit Screen      │  └─────────────┘ Webhook│  ┌──────────────────────┐ │
└──────────────────────┘         │            │  │ Distance Matrix (OSRM) │ │
            │                     │            │  │ — built once, cached   │ │
            └─────────────────────┴───────────►│  ├──────────────────────┤ │
                                                  │  │ Zoning + Routing        │ │
                                                  │  │ (OR-Tools, rep-aware)   │ │
                                                  │  ├──────────────────────┤ │
                                                  │  │ AI Pipeline              │ │
                                                  │  │ (LangChain + OpenAI)     │ │
                                                  │  ├──────────────────────┤ │
                                                  │  │ Feedback Loop            │ │
                                                  │  └──────────────────────┘ │
                                                  │                            │
                                                  │   SQLite / PostgreSQL     │
                                                  └──────────────────────────┘
```

### 4.1 Tech stack — everything free to use

| Layer | Choice | Why | Cost |
|---|---|---|---|
| Backend framework | **FastAPI** (Python) | Fast to build, auto-generated docs, async-ready | Free |
| Package management | **uv** | Fast, modern, reproducible Python env management | Free |
| Database | **SQLite** (POC) → PostgreSQL + PostGIS (later) | Zero-setup for a POC; upgrade path exists | Free |
| Routing/clustering math | **Google OR-Tools** | Open-source VRP/clustering solver — does the actual zone math, consuming real travel-time data (below) rather than straight-line distance | Free |
| Travel-time matrix | **Self-hosted OSRM (Open Source Routing Machine)** on a free OpenStreetMap extract (via Geofabrik) — the full N×N driving-time/distance matrix is computed **once** via OSRM's Table API and cached, not recalculated per instruction | Real road-network travel times, not straight-line approximations — this materially changes which outlets get clustered together and in what order they're visited | Free (self-hosted, no per-request billing) |
| Conversational layer (LLM) | **OpenAI (`gpt-4o-mini` for parsing/classification, `gpt-4o` for natural-language diff summaries if quality needs it)**, orchestrated through **LangChain** | Reliable structured-output support, mature tool-calling, and LangChain gives us composable chains/parsers instead of hand-rolled prompt-and-parse code | **Paid — the one non-free dependency** |
| Frontend framework | **React + Vite** | Fast dev loop, huge ecosystem | Free |
| Styling | **Tailwind CSS** | Fast to build a clean-looking dashboard without custom CSS overhead | Free |
| Maps | **Leaflet.js + OpenStreetMap tiles + react-leaflet** | Avoids Google Maps billing account requirement entirely, and supports the custom polylines/polygons/drag interactions this POC needs | Free |
| Hosting (demo) | Run locally, or free tier of Render/Railway/Fly.io for a shareable demo link | Free tier sufficient for a POC | Free |

**Honest note on cost**: everything except the LLM layer is genuinely free. OpenAI is a paid API — for a POC's usage volume (a few dozen instructions during demos and a short pilot), `gpt-4o-mini` costs a few dollars total, not a meaningful budget line, but it's not literally "$0" like the rest of the stack. If a hard zero-cost constraint is non-negotiable, the fallback is swapping `ChatOpenAI` for a LangChain-compatible local model via Ollama (`langchain-ollama`) — the pipeline design below doesn't change, only the model binding does.

---

## 5. Data model

Deliberately minimal — no product, inventory, or financial entities.

```python
# models.py (using SQLModel — combines Pydantic validation + SQLAlchemy ORM)

from sqlmodel import SQLModel, Field
from typing import Optional
from datetime import date, datetime
from enum import Enum

class OutletClass(str, Enum):
    A = "A"
    B = "B"
    C = "C"

class VisitFrequency(str, Enum):
    TWICE_WEEKLY = "2x_week"
    WEEKLY = "weekly"
    FORTNIGHTLY = "fortnightly"
    MONTHLY = "monthly"

class RepAvailability(str, Enum):
    WORKING = "working"
    ON_LEAVE = "on_leave"

class VisitOutcome(str, Enum):
    ORDER_LIKELY = "order_likely"
    NO_ORDER = "no_order"
    CLOSED = "closed"
    RESCHEDULE = "reschedule"
    SKIPPED = "skipped"

class Outlet(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    address: str
    lat: float
    lng: float
    outlet_class: OutletClass
    visit_frequency: VisitFrequency
    visit_duration_minutes: int = 15  # avg time spent per visit — drives the time-budget zoning math (Section 6.2)
    contact_name: Optional[str] = None
    contact_phone: Optional[str] = None
    notes: Optional[str] = None
    assigned_rep_id: Optional[int] = Field(default=None, foreign_key="rep.id")
    onboarded_at: datetime = Field(default_factory=datetime.utcnow)
    last_visited_at: Optional[datetime] = None
    active: bool = True

class Rep(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    phone: str
    territory_name: Optional[str] = None
    availability: RepAvailability = RepAvailability.WORKING
    daily_working_minutes: int = 480  # e.g. 8-hour day — the real constraint zoning solves within, not a raw outlet count
    start_lat: float
    start_lng: float
    hired_at: datetime = Field(default_factory=datetime.utcnow)
    active: bool = True

class Manager(SQLModel, table=True):
    """
    The human who issues instructions. Needed because the WhatsApp webhook
    (Section 6.6) resolves an inbound sender's phone number to either a
    Manager (→ treat the message as a plan instruction) or a Rep (→ treat it
    as a visit log). Without this, an unknown number could issue commands.
    """
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    phone: str = Field(index=True, unique=True)  # E.164, matched against the WhatsApp webhook sender
    active: bool = True
    created_at: datetime = Field(default_factory=datetime.utcnow)

class GoalType(str, Enum):
    VISIT_COUNT = "visit_count"
    NEW_OUTLETS_ADDED = "new_outlets_added"
    COVERAGE_PERCENTAGE = "coverage_percentage"
    CUSTOM = "custom"  # e.g. a revenue or order-count figure the manager tracks manually — we don't compute or touch the underlying financial data, only the target/progress framing

class Goal(SQLModel, table=True):
    """
    Quarterly/monthly target-setting, matching how BeatRoute/FieldAssist/Bizom
    frame this ("Goal-Driven SFA") — a manager sets a target per rep or per
    territory and tracks progress against it. Scoped to execution metrics we
    can measure ourselves (visits, coverage, new outlets); a CUSTOM goal lets
    a manager track a number from elsewhere (e.g. revenue) without us ever
    computing or storing financial data ourselves.
    """
    id: Optional[int] = Field(default=None, primary_key=True)
    rep_id: Optional[int] = Field(default=None, foreign_key="rep.id")  # null = team/territory-wide goal
    goal_type: GoalType
    label: str  # e.g. "Q3 outlet coverage", "New outlets — North zone"
    target_value: float
    current_value: float = 0.0  # auto-updated for visit_count/coverage/new_outlets; manually logged for CUSTOM
    period_start: date
    period_end: date
    created_at: datetime = Field(default_factory=datetime.utcnow)

class Zone(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    rep_id: int = Field(foreign_key="rep.id")
    valid_date: date
    generation_method: str = "auto"  # "auto" or "manual_override"
    created_at: datetime = Field(default_factory=datetime.utcnow)

class ZoneOutlet(SQLModel, table=True):
    """Join table: which outlets belong to which zone, in what order"""
    id: Optional[int] = Field(default=None, primary_key=True)
    zone_id: int = Field(foreign_key="zone.id")
    outlet_id: int = Field(foreign_key="outlet.id")
    visit_order: int

class Visit(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    outlet_id: int = Field(foreign_key="outlet.id")
    rep_id: int = Field(foreign_key="rep.id")
    zone_id: int = Field(foreign_key="zone.id")
    checked_in_at: Optional[datetime] = None
    outcome: Optional[VisitOutcome] = None
    notes: Optional[str] = None

class Instruction(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    raw_text: str
    parsed_intent: str  # JSON string of structured LLM output
    affected_zone_ids: str  # JSON list
    issued_at: datetime = Field(default_factory=datetime.utcnow)
    applied: bool = False

class DistanceMatrixCache(SQLModel, table=True):
    """
    Stores the last computed OSRM N×N matrix so it isn't recalculated on
    every re-plan. Keyed by a hash of the active outlet+rep coordinate set —
    only recomputed when that set actually changes (an outlet added/closed,
    a rep added), not on every conversational instruction.
    """
    id: Optional[int] = Field(default=None, primary_key=True)
    coordinate_set_hash: str = Field(index=True)
    point_ids_json: str          # ordered list of outlet/rep IDs matching matrix rows/cols
    duration_matrix_json: str    # N×N seconds, as JSON
    distance_matrix_json: str    # N×N meters, as JSON
    computed_at: datetime = Field(default_factory=datetime.utcnow)
```

---

## 6. Core services — implementation notes

### 6.1 Distance matrix service (`services/matrix.py`) — the real routing data, built once

This replaces straight-line/haversine distance with actual road-network travel times from OpenStreetMap data, computed once per stable outlet-set and cached — not recalculated on every instruction.

**Setup, one-time**:
1. Download a regional OSM extract (e.g., Maharashtra or a Pune-city cut) from [Geofabrik](https://download.geofabrik.de/) — free, open data
2. Run OSRM's preprocessing pipeline once on that extract (`osrm-extract` → `osrm-partition` → `osrm-customize`), producing a routable graph
3. Serve it locally via `osrm-routed` (official Docker image, no external calls, no billing account)

**Build-once matrix logic**:

```python
import hashlib
import httpx

OSRM_BASE_URL = "http://localhost:5000"  # self-hosted OSRM instance

def coordinate_set_hash(points: list[tuple[float, float]]) -> str:
    """A stable fingerprint of the active outlet+rep coordinate set."""
    key = "|".join(f"{lat:.6f},{lng:.6f}" for lat, lng in sorted(points))
    return hashlib.sha256(key.encode()).hexdigest()

def get_or_build_matrix(points: list[tuple[float, float]], point_ids: list[int], db_session):
    """
    Returns the cached N×N duration/distance matrix if the coordinate set
    hasn't changed; otherwise calls OSRM's Table API once and caches the result.
    """
    fingerprint = coordinate_set_hash(points)
    cached = db_session.query(DistanceMatrixCache).filter_by(
        coordinate_set_hash=fingerprint
    ).first()
    if cached:
        return json.loads(cached.duration_matrix_json), json.loads(cached.distance_matrix_json)

    # OSRM Table API expects "lng,lat;lng,lat;..." and returns full N×N matrices in one call
    coords_param = ";".join(f"{lng},{lat}" for lat, lng in points)
    response = httpx.get(
        f"{OSRM_BASE_URL}/table/v1/driving/{coords_param}",
        params={"annotations": "duration,distance"},
        timeout=30,
    )
    data = response.json()
    duration_matrix = data["durations"]  # seconds, N×N
    distance_matrix = data["distances"]  # meters, N×N

    db_session.add(DistanceMatrixCache(
        coordinate_set_hash=fingerprint,
        point_ids_json=json.dumps(point_ids),
        duration_matrix_json=json.dumps(duration_matrix),
        distance_matrix_json=json.dumps(distance_matrix),
    ))
    db_session.commit()
    return duration_matrix, distance_matrix
```

**Why build it once rather than per-request**: outlets don't move, and rep start locations rarely change day to day. Recomputing the full matrix on every conversational instruction would add latency (an OSRM Table call for ~50 points is fast, but not free, and there's no reason to pay that cost repeatedly for the same coordinate set). The cache is invalidated only when the underlying point set actually changes — a new outlet added, one marked inactive, or a rep's start location updated — which is exactly the "calculate the N×N matrix once" approach requested. All downstream services (zoning, routing) read from this cached matrix rather than computing distances themselves.

### 6.1.1 Honest note on "real-time" and traffic

Worth being precise here, because self-hosted OSRM on OpenStreetMap data and *live* traffic are two different things:

- **What self-hosted OSRM gives you**: real road-network topology and realistic average travel times based on road type and speed limits — a large, genuine improvement over haversine, but **not live congestion data**. It won't know that a specific road is jammed right now.
- **What it doesn't give you for free**: actual live traffic requires a commercial provider — Google's Distance Matrix API with `traffic_model`, Mapbox's traffic-aware Directions API, TomTom's Traffic API, or HERE — all of which charge per request past a free tier, which breaks the "everything free" constraint if used for the full N×N matrix.

**Recommended design — a hybrid, used deliberately, not everywhere**:
1. **Baseline (all zone generation, all routine re-plans)**: the cached, self-hosted OSRM matrix from above — free, fast, good enough for clustering and sequencing decisions.
2. **Selective live refinement (only at the moment a disruption instruction is being applied)**: when `process_instruction()` (Section 6.4) is about to commit a re-plan, optionally call a traffic-aware provider **only for the handful of origin-destination legs actually affected** by the change — not the full matrix. This keeps API usage small and bounded (a few requests per instruction, not thousands) while giving the ETA shown to the manager a live-traffic-adjusted number at exactly the moment it matters most.
3. **If a hard zero-cost constraint applies**, skip step 2 entirely for the POC — the self-hosted OSRM baseline is a legitimate, realistic, and free way to demonstrate the product, and live traffic becomes a clearly-labeled post-POC enhancement rather than something silently promised and not delivered.

```python
# services/matrix.py — selective live-traffic refinement, used only on active re-plans

def refine_affected_legs_with_traffic(affected_pairs: list[tuple[Point, Point]]) -> dict:
    """
    Called only for the small set of origin-destination pairs touched by a
    disruption instruction — never the full matrix. Falls back silently to
    the cached OSRM value if the traffic provider is unavailable or unconfigured,
    so this is additive, not a hard dependency.
    """
    if not TRAFFIC_PROVIDER_CONFIGURED:
        return {}  # baseline OSRM matrix values are used as-is
    refined = {}
    for origin, dest in affected_pairs:
        refined[(origin, dest)] = traffic_provider.get_duration(origin, dest, live=True)
    return refined
```

### 6.2 Zoning service (`services/zoning.py`) — sized by real time budget, not a raw outlet count

**Zones are generated with respect to the available reps, not independently.** This is not "cluster outlets geographically, then assign clusters to reps" — outlet-to-rep assignment happens *as part of* the same optimization, because each rep's real time budget has to bound the clusters as they form, and the redistribution feature (the core of this product) depends on zones being reshaped around whichever reps are actually available on a given day, not reshuffled from a fixed, rep-independent partition.

**"How much one person can do in a day" is a time budget, not an outlet count.** A zone's size isn't a static number like "40 outlets" — it's whatever combination of travel time plus time spent at each stop fits inside a rep's `daily_working_minutes`. A zone of tightly clustered C-class outlets (short visits, short drives) can hold far more stops than a spread-out zone of A-class outlets (longer visits, longer drives) — the solver has to reason about this, not a fixed per-rep cap.

The correct formulation is a **multi-depot Capacitated Vehicle Routing Problem with Time Windows (CVRPTW)**: each available rep is a "vehicle" with its own start location and a **time capacity** (`daily_working_minutes`), not an outlet-count capacity. Each outlet is a demand node whose "cost" to visit is its own `visit_duration_minutes` plus the travel time to reach it. OR-Tools' routing library supports this natively through its **cumulative dimension** mechanism (`AddDimension`), where the dimension being accumulated is *time* (travel + service time) and the per-vehicle capacity is each rep's working-minutes budget — this is the standard, industry-correct way to model "how much can one person realistically do today," and it's exactly how professional route-optimization tools (including the ones inside BeatRoute, FieldAssist, and Delta Sales App) frame the same problem.

**Stability constraint**: re-solving from scratch every day risks reshuffling zone boundaries with no real trigger, which erodes rep/manager trust. The solver should include a soft penalty for moving an outlet away from its *previously assigned* rep, so the default behavior is "keep yesterday's zones stable" and boundaries only shift meaningfully when something real changes it — a rep going `on_leave`, a working-hours change, or an explicit instruction.

```python
from ortools.constraint_solver import routing_enums_pb2, pywrapcp

def generate_zones(outlets: list[Outlet], reps: list[Rep], db_session, previous_zones=None) -> list[Zone]:
    available_reps = [r for r in reps if r.availability == "working" and r.active]
    points = [(o.lat, o.lng) for o in outlets] + [(r.start_lat, r.start_lng) for r in available_reps]
    point_ids = [o.id for o in outlets] + [r.id for r in available_reps]

    duration_matrix, _ = get_or_build_matrix(points, point_ids, db_session)
    service_times = [o.visit_duration_minutes * 60 for o in outlets] + [0] * len(available_reps)  # seconds

    manager = pywrapcp.RoutingIndexManager(
        len(points), len(available_reps),
        [len(outlets) + i for i in range(len(available_reps))],  # each rep's own start node
    )
    model = pywrapcp.RoutingModel(manager)

    def time_callback(from_index, to_index):
        from_node, to_node = manager.IndexToNode(from_index), manager.IndexToNode(to_index)
        return duration_matrix[from_node][to_node] + service_times[to_node]

    transit_callback_index = model.RegisterTransitCallback(time_callback)
    model.SetArcCostEvaluatorOfAllVehicles(transit_callback_index)

    # This is the actual "how much can one person do in a day" constraint —
    # a cumulative TIME dimension, capped per-vehicle at that rep's real working minutes.
    model.AddDimension(
        transit_callback_index,
        slack_max=0,
        vehicle_capacities=[r.daily_working_minutes * 60 for r in available_reps],
        fix_start_cumul_to_zero=True,
        name="WorkingTime",
    )

    if previous_zones:
        apply_stability_penalty(model, manager, previous_zones, point_ids)

    search_params = pywrapcp.DefaultRoutingSearchParameters()
    search_params.first_solution_strategy = routing_enums_pb2.FirstSolutionStrategy.PATH_CHEAPEST_ARC
    solution = model.SolveWithParameters(search_params)

    return build_zones_from_solution(solution, model, manager, outlets, available_reps)
```

### 6.3 Route sequencing service (`services/routing.py`)

Within a zone, orders stops starting from the rep's `start_lat/start_lng`, solving a TSP over the **same cached OSRM duration matrix** (OR-Tools' routing module, not a fresh distance calculation), with outlet class as a tie-break weight (A-class outlets sequenced earlier where reasonable). Because the matrix is already built and cached from Section 6.1, this step is just matrix lookups and a solve — no additional OSRM calls.

### 6.4 AI pipeline — LangChain + OpenAI (`services/ai_pipeline.py`)

This is the differentiated core of the product, so it gets its own detailed design rather than a single prompt-and-parse function. The pipeline has **four stages**, each a separate, testable LangChain component — deliberately not one giant prompt, because a single-shot prompt is harder to debug, harder to guard against hallucinated rep/outlet IDs, and harder to extend later.

```
Manager types instruction
        │
        ▼
┌─────────────────────────┐
│ Stage 1: Intent          │   LangChain structured-output chain
│ Classification & Entity  │   (ChatOpenAI + Pydantic schema)
│ Extraction                │
└───────────┬───────────────┘
            │  structured PlanInstruction object
            ▼
┌─────────────────────────┐
│ Stage 2: Grounding /      │   Pure Python validation — NOT the LLM.
│ Guardrail Check           │   Confirms every rep/outlet name the LLM
│                            │   extracted actually resolves to a real
│                            │   ID in the database. Rejects or asks
│                            │   for clarification if not.
└───────────┬───────────────┘
            │  validated, ID-resolved instruction
            ▼
┌─────────────────────────┐
│ Stage 3: Plan Re-solve     │   Deterministic OR-Tools re-run
│                            │   (zoning.py + routing.py) — NOT the LLM.
│                            │   This is why the routing math stays
│                            │   trustworthy: the LLM never invents
│                            │   a route, it only sets constraints.
└───────────┬───────────────┘
            │  old plan + new plan (diff)
            ▼
┌─────────────────────────┐
│ Stage 4: Diff              │   LangChain chain (ChatOpenAI) —
│ Summarization              │   turns the structured diff into the
│                            │   plain-English sentence the manager
│                            │   sees ("Amit picked up 15 outlets...")
└─────────────────────────┘
```

**Why split it this way**: the LLM's job is strictly *understanding intent* (Stage 1) and *explaining outcomes in plain language* (Stage 4). The actual planning math never touches the LLM (Stage 3) — this is what makes the system trustworthy to a manager who needs to believe the plan is sound, not just plausible-sounding. Stage 2 exists specifically to catch the most common LLM failure mode: hallucinating or mis-resolving a name ("Suresh" matching the wrong rep, or an outlet name that doesn't exist).

```python
# services/ai_pipeline.py

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field
from typing import Literal, Optional

# ---- Stage 1: structured schema the LLM must fill in ----

class PlanInstruction(BaseModel):
    action: Literal["redistribute", "reprioritize", "reschedule", "query"] = Field(
        description="The type of change the manager wants"
    )
    source_rep_name: Optional[str] = Field(
        default=None, description="The rep whose outlets are being reassigned, if any"
    )
    target_rep_names: list[str] = Field(
        default_factory=list, description="Reps who should receive reassigned outlets"
    )
    outlet_class_filter: Optional[Literal["A", "B", "C"]] = Field(
        default=None, description="If the instruction prioritizes a specific outlet class"
    )
    reasoning_note: Optional[str] = Field(
        default=None, description="Any stated reason, e.g. 'scheme launches today'"
    )
    date_shift: Optional[str] = Field(
        default=None, description="If outlets should move to a different date, e.g. 'next week'"
    )

# ---- Stage 1 chain: intent classification + entity extraction ----

intent_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
structured_llm = intent_llm.with_structured_output(PlanInstruction)

intent_prompt = ChatPromptTemplate.from_messages([
    ("system", (
        "You parse a field-sales manager's plain-language instruction into a "
        "structured plan change. Only extract what is explicitly stated — "
        "never invent rep or outlet names that weren't mentioned."
    )),
    ("human", "{raw_text}"),
])

intent_chain = intent_prompt | structured_llm


def extract_instruction(raw_text: str) -> PlanInstruction:
    return intent_chain.invoke({"raw_text": raw_text})


# ---- Stage 2: grounding / guardrail check (plain Python, no LLM) ----

def resolve_and_validate(instruction: PlanInstruction, db_session) -> "ResolvedInstruction":
    """
    Confirms every name the LLM extracted actually matches a real Rep/Outlet.
    Raises a clarification-needed exception rather than silently guessing
    if a name is ambiguous or not found.
    """
    source_rep = lookup_rep_by_name(db_session, instruction.source_rep_name)
    if instruction.source_rep_name and source_rep is None:
        raise NeedsClarification(f"No rep matching '{instruction.source_rep_name}' found.")

    target_reps = [lookup_rep_by_name(db_session, name) for name in instruction.target_rep_names]
    if any(r is None for r in target_reps):
        raise NeedsClarification("One or more target reps couldn't be matched.")

    return ResolvedInstruction(
        action=instruction.action,
        source_rep_id=source_rep.id if source_rep else None,
        target_rep_ids=[r.id for r in target_reps],
        outlet_class_filter=instruction.outlet_class_filter,
        date_shift=instruction.date_shift,
    )


# ---- Stage 3: deterministic re-solve (calls zoning.py / routing.py, no LLM) ----

def apply_resolved_instruction(resolved: "ResolvedInstruction", current_zones) -> "PlanDiff":
    old_zones = snapshot(current_zones)
    new_zones = zoning.reassign_and_resequence(resolved, current_zones)  # OR-Tools
    return build_diff(old_zones, new_zones)


# ---- Stage 4: diff summarization back into plain English ----

summary_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.3)

summary_prompt = ChatPromptTemplate.from_messages([
    ("system", (
        "You explain a field-sales plan change to a manager in one or two "
        "short sentences. Be specific with numbers. No preamble."
    )),
    ("human", "Plan diff data: {diff_json}"),
])

summary_chain = summary_prompt | summary_llm


def summarize_diff(diff: "PlanDiff") -> str:
    return summary_chain.invoke({"diff_json": diff.model_dump_json()}).content


# ---- Orchestration: the full pipeline, called by the /instructions endpoint ----

def process_instruction(raw_text: str, db_session, current_zones) -> dict:
    try:
        extracted = extract_instruction(raw_text)
        resolved = resolve_and_validate(extracted, db_session)
        diff = apply_resolved_instruction(resolved, current_zones)
        summary = summarize_diff(diff)
        return {"success": True, "diff": diff.model_dump(), "summary": summary}
    except NeedsClarification as e:
        return {"success": False, "clarification_needed": str(e)}
```

**Guardrails worth calling out explicitly**:
- `temperature=0` on the extraction chain — we want deterministic, literal parsing, not creative interpretation of an instruction that controls a real business operation
- The grounding step (Stage 2) is intentionally plain Python, not another LLM call — validating IDs against the database is a correctness problem, not a language-understanding problem, and should never be delegated to a model that can hallucinate
- If a rep or outlet name doesn't resolve cleanly, the pipeline returns a clarification request rather than guessing — this matters more here than in a typical chatbot, because a wrong guess silently reassigns real sales visits
- The routing/clustering math (Stage 3) is 100% deterministic OR-Tools — the LLM is never in the loop for the actual planning decision, only for translating human intent into constraints and translating results back into human language

### 6.5 Feedback loop (`services/visits.py`)

When a visit is logged with `outcome=RESCHEDULE`, this is treated internally as a system-generated instruction — it creates a future constraint that pulls the outlet from near-term zones and resurfaces it automatically, routed through the same `apply_instruction` logic rather than a separate code path.

### 6.6 WhatsApp integration (`services/whatsapp.py`)

This is deliberately a thin layer, not a parallel system — WhatsApp is just another **source of raw text** feeding the same AI pipeline (Section 6.4), and another **channel to deliver** the same plan and diff summaries. No new planning logic is introduced here.

**Two flows:**

1. **Manager instructions via WhatsApp.** A manager messages the distributor's WhatsApp Business number directly — "Suresh is off today, redistribute his beat, prioritize A-class" — exactly as they would type into the dashboard's command bar. A webhook receives it, resolves the sender's phone number to a known manager, and passes the message text straight into `process_instruction()` (Section 6.4). The plain-English diff summary that function already produces is sent back as the WhatsApp reply — no separate message-formatting logic needed.
2. **Rep visit logging via WhatsApp.** Each morning, a rep's sequenced route is sent as a WhatsApp message. After a visit, the rep replies — either tapping a quick-reply button (WhatsApp's native interactive message buttons: *Order likely / No order / Reschedule*) or typing free text ("shop wants to skip this month"), which is parsed through the same intent-extraction chain used for instructions, since a logged outcome is treated internally as a system-generated instruction (Section 6.5).

```python
# services/whatsapp.py — simplified webhook handler

from fastapi import APIRouter, Request

router = APIRouter()

@router.post("/webhooks/whatsapp")
async def handle_whatsapp_message(request: Request, db_session=Depends(get_session)):
    payload = await request.json()
    sender_phone = extract_sender_phone(payload)
    message_text = extract_message_text(payload)

    manager = lookup_manager_by_phone(db_session, sender_phone)
    rep = lookup_rep_by_phone(db_session, sender_phone)

    if manager:
        # Same pipeline as the dashboard command bar — Section 6.4
        result = process_instruction(message_text, db_session, current_zones=get_today_zones(db_session))
        reply_text = result.get("summary") or f"Couldn't apply that: {result.get('clarification_needed')}"
        send_whatsapp_message(sender_phone, reply_text)

    elif rep:
        outcome = parse_visit_outcome(message_text)  # reuses the same structured-output approach as Section 6.3
        log_visit(db_session, rep_id=rep.id, outcome=outcome)
        send_whatsapp_message(sender_phone, "Logged, thanks.")
```

**Provider choice**: Meta's WhatsApp Cloud API directly (rather than a paid reseller like Twilio/Gupshup) — it has a genuine free tier for a meaningful volume of business-initiated conversations, which fits a POC's usage well, though like the OpenAI dependency, it's worth re-checking current free-tier limits at build time rather than assuming they're unlimited.

**Why this stays architecturally simple**: because Stage 1 and Stage 4 of the AI pipeline (Section 6.4) already turn raw text into structured intent and structured results back into plain English, WhatsApp doesn't need its own parsing or response-generation logic — it's purely a transport layer bolted onto an interface the pipeline already exposes.

---

## 7. API endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/outlets/bulk` | Import outlet list (JSON/CSV) |
| GET | `/outlets` | List outlets, filterable by zone/class/status/rep |
| POST | `/outlets` | Add a single outlet (manager-entered, not bulk import) |
| GET | `/outlets/{id}` | Outlet detail — profile, full visit history, current assigned rep |
| PATCH | `/outlets/{id}` | Edit outlet details (class, frequency, contact info, active status) |
| GET | `/reps` | List all reps with current status/zone/territory |
| POST | `/reps` | Add a new rep |
| GET | `/reps/{id}` | Rep detail — profile, current zone, visit history, goal progress |
| PATCH | `/reps/{id}` | Edit rep details (availability, working hours, territory, active status) |
| POST | `/zones/generate` | Trigger zone generation for a given date |
| GET | `/zones/{date}` | Get today's zones with sequenced stops |
| POST | `/instructions` | **Core endpoint** — submit plain-language instruction, get back the re-solved plan + diff summary |
| POST | `/visits` | Log a visit outcome |
| GET | `/goals` | List goals, filterable by rep/territory/period |
| POST | `/goals` | Create a quarterly/monthly goal (rep-specific or team-wide) |
| GET | `/goals/{id}` | Goal detail with current progress |
| GET | `/dashboard/coverage` | Coverage view — self-reported vs. GPS-verified visit status, backlog list |
| GET | `/dashboard/instructions` | Instruction history/audit log |

---

## 8. Frontend — screens

This mirrors how real SFA/DMS platforms (FieldAssist, Bizom, BeatRoute) structure their manager console: master-data management (reps, outlets) sits alongside planning and live tracking, not bolted on separately — a manager's day involves moving between "who works for me," "who do I sell to," and "what's happening right now" constantly.

### 8.1 Map view and route-planning interactions

The map is the centerpiece of the planning view, not a decorative panel — it needs to support real inspection and manual override, not just display pins.

- **Zone rendering**: each rep's zone is drawn as a semi-transparent colored polygon (convex hull of their assigned outlets), so the "zone" concept is visible as a shape, not just a color grouping of dots
- **Route polyline**: within a selected rep's zone, draw the actual sequenced route as a connected line between stops, with small numbered markers (1, 2, 3…) showing visit order — this is what makes "route planning" visible, not just "which outlets belong to whom"
- **Outlet pin states**: pending (gray outline), completed (filled green), skipped (filled red), rescheduled (filled amber) — status is visible on the map in real time as visits get logged
- **Click a rep card → highlight that rep's route**: dims other zones, brings their polyline and numbered stops to the foreground, shows ETA per stop (read directly from the cached OSRM duration matrix, Section 6.1 — real road travel time, refined with live traffic on affected legs per Section 6.1.1 where configured)
- **Click an outlet pin → detail popup**: name, class, last-visited date, assigned rep, visit history for that outlet — links through to the full Outlet detail screen (Section 8.3)
- **Manual override via drag**: a manager can drag an outlet pin from one rep's zone into another's — this triggers a re-sequence call for both affected zones immediately, exactly like a conversational instruction would, so manual and conversational edits go through the same underlying re-solve logic rather than being two separate code paths
- **Live redraw animation**: when an instruction is applied (via command bar or drag), outlet pins animate color/position transitions rather than a hard page refresh — this is the visual "wow" moment the demo depends on, so it's worth the small extra frontend effort
- **Legend**: rep colors, outlet class markers, visit-status markers, all visible at a glance

- **Command bar**: primary, prominent text input for instructions — with the resolved diff shown immediately after submission ("Amit picked up 15 outlets from Suresh's zone, including 4 A-class"), and a clarification prompt shown inline if the AI pipeline's guardrail (Section 6.4, Stage 2) couldn't confidently resolve a name
- **Rep cards**: per-rep outlet count, completed vs. pending, remaining time-budget for the day, clicking one drives the map highlight described above
- **Coverage dashboard**: self-reported vs. GPS-verified visit status over the period, list of outlets falling behind their visit-frequency rule
- **Instruction history**: chronological log of what changed and why, including manual drag-based overrides alongside typed instructions

### 8.2 Rep management console

Standard "user master" screen, matching how FieldAssist/Bizom/BeatRoute structure rep administration:

- **Rep list**: table view — name, phone, territory, current availability, active zone, today's completed/pending count. Sortable/filterable, with search.
- **Add/edit rep**: name, phone, territory name, start location (pin-drop or address), `daily_working_minutes`, active/inactive toggle.
- **Rep detail page**: profile info, a mini-map of their current zone, full visit history (chronological, with outcomes), goal progress (Section 8.4) for any goals assigned to them, and an availability toggle (`working`/`on_leave`) that — when flipped — is the manual-UI equivalent of typing "Rep X is out today" into the command bar, routed through the exact same re-plan logic.

### 8.3 Outlet / client management console

The "party master" / customer master screen — every SFA and DMS platform has one, since it's the base data everything else (beats, visits, goals) is built on:

- **Outlet list**: table view — name, address, class, assigned rep, visit frequency, last visited date, active/inactive. Sortable/filterable/searchable, with a "days since last visit vs. frequency rule" column so overdue outlets are visible at a glance without opening the coverage dashboard separately.
- **Add/edit outlet**: name, address (with the pin-drop fallback described in earlier product discussion, for informal addresses), class, visit frequency, `visit_duration_minutes`, contact name/phone, notes, active/inactive toggle.
- **Outlet detail page**: full visit history with outcomes and notes, a small map showing its location and current zone, and its assigned rep — a manager can reassign an outlet directly from here (equivalent to a drag on the main map).

### 8.4 Goal / target management

Matches the "Goal-Driven SFA" framing that's become standard across BeatRoute and similar platforms — a manager sets a target and tracks progress against it, without this system ever touching financial/order data directly:

- **Goal list**: active goals by rep or team-wide, period (monthly/quarterly), target vs. current progress, shown as a simple progress bar.
- **Create goal**: pick a goal type (visit count, new outlets added, coverage percentage, or a custom label for something tracked manually, like a revenue figure sourced from elsewhere), a target value, a period, and optionally a specific rep or "team-wide."
- **Progress tracking**: visit-count, new-outlet, and coverage-percentage goals update automatically as visits get logged and outlets get added — no manual data entry required. Custom goals are updated by the manager directly, since we deliberately don't compute or store the underlying financial numbers ourselves.
- **Goal detail view**: a simple trend line of progress over the period, visible per rep if it's a team-wide goal broken down by contribution.

### 8.5 Rep screen (can be a responsive web view for POC — no native app needed)
- Today's outlet list in visit order, on a simple map
- Tap to check in, pick an outcome (order likely / no order / closed / reschedule), optional text note
- Notification banner if the manager's instruction changed today's route

---

## 9. Core workflows — how this is actually used, end to end

This is the operational sequence, matching how FMCG distributor field operations actually run day to day, week to week, and quarter to quarter.

### 9.1 Rep onboarding (one-time, per new hire)
1. Manager adds the rep via the Rep management console (Section 8.2) — name, phone, territory, working hours, start location.
2. Rep is included in the next zone generation run — since zoning is rep-aware (Section 6.2) and stability-biased, a new rep triggers a real re-solve (there's no "previous zone" to stay stable against), while existing reps' zones only shift as much as needed to free up capacity for the newcomer.
3. Rep receives their first day's route via the dashboard and/or WhatsApp (Section 6.6).

### 9.2 Outlet onboarding (ongoing, as the outlet universe changes)
1. Manager adds a new outlet individually (Section 8.3) or via bulk CSV import for an initial setup.
2. Address is geocoded; if it fails (common for informal addresses), it's queued for the "walk and pin" flow — the assigned rep confirms the exact location on their first visit.
3. The outlet enters the distance-matrix cache and next zone-generation run as a new point — the stability bias (Section 6.2) means it gets assigned to whichever rep can absorb it with least disruption to existing zones, not a random reshuffle.

### 9.3 Standing plan generation (periodic, e.g. weekly/monthly — mirrors a real PJP cycle)
1. Manager triggers (or the system schedules) a full zone regeneration.
2. The time-budget CVRPTW solve (Section 6.2) runs across all active outlets and available reps.
3. Manager reviews the proposed zones on the map before they go live — this is a deliberate checkpoint, since the standing plan is meant to be stable for months (Section 2), not something that should change without a human glancing at it first.

### 9.4 Daily execution (every working day)
1. Each rep receives their sequenced route for the day (dashboard + WhatsApp).
2. Rep travels the route, checking in (GPS-verified where possible) and logging an outcome at each stop.
3. Each logged outcome updates the coverage dashboard in real time and, if it's a `RESCHEDULE`, feeds back into future planning automatically (Section 6.5).

### 9.5 Disruption / exception handling (as needed — the core differentiated loop)
1. A disruption occurs: a rep is unavailable, an urgent priority arises, weather or a local event makes part of a zone impractical for the day.
2. Manager describes it in plain language — via the dashboard command bar or WhatsApp — or performs the equivalent manual action (toggling a rep's availability, dragging an outlet on the map).
3. The AI pipeline (Section 6.4) resolves the instruction, re-solves only the affected portion of the plan (today's assignment, not the standing PJP), and returns a plain-English summary of what changed.
4. This is explicitly scoped as an **exception to today's plan**, not a rewrite of the standing beat (Section 2) — tomorrow, the system reverts to the stable, previously-generated zones unless the disruption persists (e.g., a rep is on extended leave, which would trigger a real regeneration per 9.3).

### 9.6 Goal setting and tracking (quarterly/monthly, per the Goal-Driven SFA pattern)
1. Manager creates a goal (Section 8.4) — e.g., "90% coverage of A-class outlets this quarter" or "10 new outlets onboarded in North zone this month."
2. Visit-count, new-outlet, and coverage-percentage goals update automatically as the daily execution loop (9.4) and outlet onboarding (9.2) generate real activity.
3. Manager reviews goal progress on a regular cadence (the Goal detail view, Section 8.4), identifying reps or territories falling behind before the period ends, not after.

### 9.7 Weekly/quarterly review (manager-facing, matches how FMCG sales reviews actually run)
1. Manager opens the coverage dashboard — sees GPS-verified visit completion, not self-reported numbers (Section 2.1), across the period.
2. Cross-references against active goals (9.6) and the outlet backlog list (outlets overdue against their frequency rule).
3. Decisions from this review — reassigning underperforming territory, onboarding new outlets, adjusting a rep's working hours — feed back into 9.1–9.3 for the next cycle.

---

## 10. Demo scenario — simulating a real industry workflow

This is the script to run live, and the dataset to seed for it.

### The setup
- **Location**: Aundh, Pune (realistic mid-market FMCG distributor territory)
- **4 reps**: Suresh, Amit, Deepak, Vikram
- **~45 outlets** in Suresh's zone (kirana stores, general stores), mix of A/B/C class, seeded with realistic Aundh-area coordinates
- **Amit, Deepak, Vikram** each already have their own ~35–40 outlet zones nearby (so the demo shows real multi-rep redistribution, not an empty map filling up)

### Seed data shape (`data/seed_aundh_beat.json`)

```json
{
  "reps": [
    {"name": "Suresh", "phone": "9800000001", "daily_working_minutes": 480, "start_lat": 18.5610, "start_lng": 73.8070},
    {"name": "Amit", "phone": "9800000002", "daily_working_minutes": 480, "start_lat": 18.5636, "start_lng": 73.8095},
    {"name": "Deepak", "phone": "9800000003", "daily_working_minutes": 480, "start_lat": 18.5583, "start_lng": 73.8112},
    {"name": "Vikram", "phone": "9800000004", "daily_working_minutes": 420, "start_lat": 18.5648, "start_lng": 73.8048}
  ],
  "outlets": [
    {"name": "Shree Ganesh General Store", "address": "Aundh Main Rd", "lat": 18.5615, "lng": 73.8082, "outlet_class": "A", "visit_frequency": "2x_week", "assigned_rep": "Suresh"},
    {"name": "Balaji Kirana", "address": "DP Road, Aundh", "lat": 18.5622, "lng": 73.8071, "outlet_class": "B", "visit_frequency": "weekly", "assigned_rep": "Suresh"}
    // ... ~43 more, generated programmatically with realistic jitter around Aundh's coordinates
  ]
}
```

**Note**: reps are seeded with `daily_working_minutes` (the real solver constraint, Section 6.2), not an outlet-count `capacity` — zone size is an emergent result of the time budget, never an input. Vikram is deliberately seeded at 420 rather than 480 so the demo can show a shorter working day producing a visibly smaller zone (success criterion 6, Section 13).

A seed script (`scripts/seed_db.py`) should generate the remaining outlets programmatically — realistic Indian shop-name patterns, jittered coordinates within a ~2km radius of the given center points, and a plausible class distribution (roughly 20% A, 50% B, 30% C, matching real-world beat composition).

### The live demo script

1. **Show today's plan, as-is.** Map loads with all 4 zones visible, color-coded. Suresh's 45 outlets are visibly his own color, unassigned to anyone else.
2. **Narrate the trigger**: "Suresh just called in sick. A new scheme launches today and needs to hit A-class outlets first."
3. **Manager marks Suresh as `on_leave`** via a simple toggle (simulating the real first step — before, this information had nowhere structured to go).
4. **Type the instruction** into the command bar: *"Suresh is off today, split his 45 outlets across Amit, Deepak and Vikram — prioritize A-class outlets first since the scheme launches today."*
5. **Submit — the map redraws live.** Suresh's outlets are absorbed into the three other zones, sequenced with A-class outlets pulled toward the front of each rep's route. A plain-English diff appears: "Amit +16 outlets (5 A-class), Deepak +14 outlets (4 A-class), Vikram +15 outlets (4 A-class)."
6. **Simulate the day happening**: for the demo, either pre-load some visits as already logged (to show the outcome flow), or manually log 2–3 visits live through the rep screen to show the check-in → outcome → dashboard update loop.
7. **Show the coverage dashboard**: this is the closing beat — not a fabricated adherence percentage, but the honest version of the value: a comparison between "self-reported" visit status (which tends to look near-perfect regardless of what actually happened) and the **GPS-verified** completed/skipped/rescheduled count from this session — making the real-coverage-data point visually, alongside the redistribution the audience just watched happen.

### What this scenario proves, and to whom

- To a **distributor/manager**: this is their own Monday morning, not an abstract feature tour
- To an **investor**: this is the "signal-driven vs. instruction-driven" gap made visible in under 2 minutes, live, not on a slide

---

## 11. Project structure

```
field-sales-poc/
├── pyproject.toml          # uv-managed dependencies
├── uv.lock
├── backend/
│   ├── main.py             # FastAPI app entrypoint
│   ├── models.py           # SQLModel data model (Section 5)
│   ├── db.py                # DB session/engine setup
│   ├── services/
│   │   ├── matrix.py        # OSRM distance/duration matrix, cached (Section 6.1)
│   │   ├── zoning.py
│   │   ├── routing.py
│   │   ├── ai_pipeline.py   # LangChain + OpenAI (Section 6.4)
│   │   ├── visits.py
│   │   └── whatsapp.py      # WhatsApp Cloud API webhook + message routing (Section 6.6)
│   ├── api/
│   │   ├── outlets.py
│   │   ├── reps.py
│   │   ├── zones.py
│   │   ├── instructions.py
│   │   ├── visits.py
│   │   └── dashboard.py
│   └── tests/
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── App.jsx
│   │   ├── components/
│   │   │   ├── MapView.jsx
│   │   │   ├── CommandBar.jsx
│   │   │   ├── RepCards.jsx
│   │   │   ├── CoverageDashboard.jsx
│   │   │   └── InstructionHistory.jsx
│   │   └── api/client.js
│   └── vite.config.js
├── data/
│   ├── seed_aundh_beat.json
│   └── osrm/                # downloaded .osm.pbf extract + preprocessed .osrm files
├── scripts/
│   └── seed_db.py
└── README.md
```

---

## 12. Setup instructions (using uv)

```bash
# --- One-time: set up OSRM for real road-network routing ---
mkdir -p data/osrm && cd data/osrm

# Download a regional extract (example: Maharashtra) from Geofabrik — free, open data
wget https://download.geofabrik.de/asia/india/maharashtra-latest.osm.pbf

# Preprocess once, using OSRM's official Docker image (no local OSRM install needed)
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-extract -p /opt/car.lua /data/maharashtra-latest.osm.pbf
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-partition /data/maharashtra-latest.osrm
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-customize /data/maharashtra-latest.osrm

# Serve it locally — this is what services/matrix.py calls
docker run -t -i -p 5000:5000 -v "${PWD}:/data" osrm/osrm-backend osrm-routed --algorithm mld /data/maharashtra-latest.osrm

cd ../..

# --- Backend setup ---
cd field-sales-poc
uv init backend
cd backend
uv add fastapi uvicorn sqlmodel ortools httpx langchain langchain-openai pydantic

# Set your OpenAI API key (the one paid dependency in this stack)
export OPENAI_API_KEY="sk-..."

# Seed the demo data (this also triggers the first OSRM Table API call,
# building and caching the N×N matrix for the seeded outlet set)
uv run scripts/seed_db.py

# Start the backend
uv run uvicorn main:app --reload

# Frontend setup (separate terminal)
cd ../frontend
npm create vite@latest . -- --template react
npm install tailwindcss leaflet react-leaflet
# react-leaflet powers the zone polygons, route polylines, and drag-to-reassign
# interactions described in Section 8
npm run dev
```

**Note on picking the extract size**: a full-state extract (e.g., all of Maharashtra) is more than needed for a single-city pilot and makes the preprocessing step slower. For a POC scoped to one city (Pune/Aundh, per the demo scenario), it's worth clipping the extract down first with `osmium extract` (also free, part of the `osmium-tool` package) to just the relevant bounding box — cuts preprocessing time from minutes to seconds.

---

## 13. Success criteria for this POC

The POC is considered successful if it can, live, in front of a real distributor or investor:

1. Show a real (or realistically modeled) multi-rep beat, visually, on a map
2. Take a plain-language disruption instruction — typed in the dashboard **or sent via WhatsApp** — and redraw the plan in **under 5 seconds**, measured from submit to redrawn map.

   That budget is tight and has to be designed for, not hoped for. It covers two LLM round-trips (Stage 1 extraction, Stage 4 summarisation — roughly 1–2s combined) plus the re-solve, leaving the solver ~2s. Three things make it hold: the OSRM matrix must be **warm in cache before the demo starts** (a cold `/table` build for ~160 points during a live demo will blow the budget on its own); the solver must be given an explicit `time_limit` in its search parameters so it returns the best solution found rather than running to optimality; and Stage 4's summary can be **streamed after the map has already redrawn**, since the diff is fully known before the LLM is called — the manager sees the new plan first and the sentence a moment later. If the re-solve is scoped to marginal insertion rather than a full CVRPTW re-solve, this becomes comfortable rather than tight.
3. Produce a plain-English explanation of what changed and why, delivered back through whichever channel the instruction came from
4. Log at least a handful of visit outcomes through the rep screen or WhatsApp reply
5. Display a coverage view that honestly distinguishes self-reported status from GPS-verified status — not a single invented adherence percentage
6. **Demonstrate the manager console**: add/edit a rep, add/edit an outlet, and show a zone visibly resize when a rep's working hours or an outlet's visit duration changes — proving zones are driven by real time budgets, not a fixed count
7. **Show at least one goal being created and its progress updating automatically** as visits are logged during the demo

Anything beyond this — polish, integrations, multilingual support — belongs on the post-POC roadmap, not in this build.

---

## 14. Post-POC roadmap (for reference, not for this build)

1. Real design-partner pilot (3–5 distributors, 4–8 weeks, real data)
2. Read-only Tally/Marg connector (party master + outstanding dues)
3. Offline-first mobile app (React Native) replacing the responsive web rep screen
4. Multilingual/voice conversational input
5. Order-booker vs. delivery-van role modeling
6. Add-on/API-integrated mode for BeatRoute/FieldAssist/Salesforce customers
7. Second vertical (NBFC field collections is the leading candidate)
8. Direct validation of the "ask vs. instruct" gap (Section 2.1) against a live BeatRoute demo and a working FMCG area sales manager — before this claim goes into any external pitch material
9. Full live-traffic provider integration (Section 6.1.1) beyond the selective-refinement approach, if usage volume and budget justify it
