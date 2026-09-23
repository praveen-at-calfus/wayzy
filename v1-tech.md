# V1 — technical description

---

## 1 · FRONTEND

```
  React 18 + Vite  ·  Tailwind  ·  MapLibre GL JS
```

| Piece | What it is |
|---|---|
| **Map view** | MapLibre GL + OpenFreeMap vector tiles. Zone polygons (convex hull per rep), route polylines with numbered stops, pin states (pending / done / skipped / rescheduled), ETA per stop. WebGL so the redraw animates smoothly. |
| **Command bar** | Text input → `POST /instructions`. Renders the returned diff, or an inline clarification prompt if a name didn't resolve. One-tap undo. |
| **Drag-to-reassign** | Dragging a pin between zones hits the same re-solve endpoint as a typed instruction — one code path, not two. |
| **Consoles** | Rep list/detail, outlet list/detail, goals. Plain CRUD tables + forms. |
| **Coverage dashboard** | Self-reported vs GPS-verified completion, plus the overdue-outlet backlog. |
| **Rep screen** | Responsive web, no native app. Today's stops in order, check in, pick an outcome, optional note. |

State: local React state + fetch. No Redux, no query library — V1 isn't big enough to need one.

---

## 2 · BACKEND

```
  FastAPI  ·  SQLModel  ·  SQLite  ·  uv
```

**Services**

| File | Does |
|---|---|
| `matrix.py` | Loads `data/matrix.json` — the pre-built N×N OSRM duration/distance matrix. Read-only at runtime. |
| `insertion.py` | **Daily path.** Marginal insertion: orphaned outlets are the only free variables, every other route is frozen. Pure Python, deterministic, sub-second. Emits `PlanDiff`. |
| `zoning.py` | **Periodic path.** Multi-depot CVRPTW via OR-Tools, capped by each rep's `daily_working_minutes`. Runs weekly, manager reviews before it goes live. |
| `routing.py` | TSP within a zone, over the same cached matrix. |
| `ai_pipeline.py` | 4 stages — (1) extract intent, LLM, structured output, temp 0 · (2) resolve IDs, plain Python · (3) call the engine, no LLM · (4) summarise the diff, LLM, streamed after the response. |
| `visits.py` | Visit logging. A `RESCHEDULE` outcome becomes a system-generated instruction through the same path. |
| `telegram.py` | Webhook → same `process_instruction()`. Pure transport. |

**Data model** — `Outlet · Rep · Manager · Zone · ZoneOutlet · Visit · Goal · Instruction`. No product, inventory or financial entities. The system never touches money or stock.

**Key endpoint** — `POST /instructions`: raw text in, `PlanDiff` + summary out. Everything else is CRUD around it.

---

## 3 · OTHERS

| | |
|---|---|
| **OSRM** | Runs once at build time in Docker (`extract → partition → customize → routed`), one `/table` call, output committed as `data/matrix.json`. **Not a runtime service.** Re-run only when outlets or rep start points change. |
| **Map data** | Geofabrik OSM extract, clipped to a Pune bounding box with `osmium extract`. Free. |
| **LLM** | Provider SDK directly with Pydantic structured output — no LangChain. Two calls per instruction (~1s each). The one paid dependency; a few dollars for the whole POC. |
| **Messaging** | Telegram Bot API in V1 — instant, no verification. WhatsApp Cloud API swaps into the same handler once Meta Business verification clears; start that on day 1. |
| **Seed data** | Aundh, Pune. 4 reps, ~160 outlets, A/B/C mix ≈ 20/50/30, jittered around real coordinates. Vikram seeded at 420 working minutes so a shorter day visibly produces a smaller zone. |
| **Hosting** | Local for the demo; Render/Railway/Fly free tier for a shareable link. SQLite re-seeds on boot — free-tier disk is ephemeral. |
| **Auth** | Shared password. Nothing real is at stake in V1. |

**Latency budget** — 5s submit → redrawn map. Stage 1 ~0.8s, stage 2 ~0s, stage 3 ~0.2s, map live at ~1s. Stage 4 streams in after and never blocks the redraw.
