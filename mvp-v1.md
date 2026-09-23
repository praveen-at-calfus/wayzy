# Wayzy — MVP v1 build decisions

Amends `wayzy.md`. Scope (§3), data model (§5) and demo script (§10) are unchanged.
This doc overrides **how we build it** only.

---

## 1. Decisions

| # | Area | Spec said | v1 does | Why |
|---|---|---|---|---|
| 1 | Replan engine | Full CVRPTW + stability penalty | **Marginal insertion** | Deterministic, <1s, no tuning |
| 2 | OSRM | Runtime service | **Build-time only** | Nothing to crash mid-demo |
| 3 | Maps | Leaflet | **MapLibre GL** | WebGL — the redraw animation |
| 4 | LLM glue | LangChain | **Provider SDK direct** | 2 calls ≠ a framework |
| 5 | Messaging | WhatsApp Cloud API | **Telegram first**, WA verification in parallel | Meta approval is unbounded |
| 6 | DB | SQLite | SQLite + **re-seed on boot** | Free-tier disk is ephemeral |
| 7 | Model | OpenAI, assumed | **Bench 2 models × 20 Hinglish instructions** | Decide with data |
| 8 | Commit UX | Instruct → commit | **Instruct → commit → 1-tap undo** | Survives the first bad plan |

---

## 2. Two solve modes — never confuse them

```
                       ┌──────────────────────────────────┐
  §9.3  weekly /       │  FULL CVRPTW  (OR-Tools)          │
        monthly   ────►│  all zones · all reps             │
        PJP cycle      │  slow · manager reviews → go live │
                       └──────────────────────────────────┘

                       ┌──────────────────────────────────┐
  §9.5  disruption ───►│  MARGINAL INSERTION               │
        today only     │  orphans only · others frozen     │
                       │  <1s · auto-commit + undo         │
                       └──────────────────────────────────┘
```

---

## 3. Marginal insertion — the rule

```
free            orphaned outlets only
frozen          every other rep's existing stop sequence
op              insert stop S into rep R's route at position i

cost(S,R,i)  =  d(prev, S) + d(S, next) − d(prev, next)      ← added travel only
reject if    =  total_time(R) > R.daily_working_minutes
order        =  highest-priority stop first (A-class, overdue, promised)
pick         =  lowest-cost feasible (R, i)
leftover     →  DEFERRED LIST + a date.  Always rendered. Never silent.
```

> Every insertion is someone's deletion. Showing the deferred list is what
> makes the output trustworthy — it is not an edge case, it is the feature.

---

## 4. OSRM: build-time, not runtime

```
   BUILD TIME  (once, on a laptop)          RUNTIME  (demo / pilot)
   ──────────────────────────────           ───────────────────────
   maharashtra.osm.pbf                      FastAPI
        │  osmium extract → Pune bbox       SQLite
        ▼                                   matrix.json   ◄── read from disk
   osrm-extract → partition → customize     LLM API
        │
        ▼
   osrm-routed :5000
        │  ONE /table call, N×N
        ▼
   data/matrix.json  ──────────────────────►  committed to repo
        │
        └── container stopped. Not a demo dependency.
```

Recompute only when the outlet/rep coordinate set changes → re-run, re-commit.

---

## 5. Instruction pipeline + latency budget

```
 submit
   │
   ├─ Stage 1   extract intent          LLM          ~0.8s
   ├─ Stage 2   resolve IDs             pure Python   ~0.0s   ← rejects bad names
   ├─ Stage 3   marginal insertion      deterministic ~0.2s   ← LLM never here
   │
  1.0s ══════  MAP REDRAWS  ══════════════════════════════  ◄ the demo moment
   │
   └─ Stage 4   diff → plain English    LLM          ~1.0s   ← streams in after
```

Budget is 5s (§13.2). Map is live at ~1s because the diff is fully known
before Stage 4 is called — **never block the redraw on the sentence.**

---

## 6. Stack

| Layer | v1 | Changed? |
|---|---|---|
| Backend | FastAPI + SQLModel + uv | — |
| DB | SQLite (re-seed on boot) | 6 |
| Solver | OR-Tools (periodic) + plain Python insertion (daily) | **1** |
| Travel time | OSRM → `matrix.json` | **2** |
| LLM | Provider SDK, Pydantic structured output | **4** |
| Frontend | React + Vite + Tailwind | — |
| Map | MapLibre GL + OpenFreeMap vector tiles | **3** |
| Messaging | Telegram Bot API → WhatsApp Cloud API | **5** |

---

## 7. Build order (dependency, not schedule)

```
  models + seed + matrix.json
          │
          ▼
  map renders 4 real zones ─────────────────┐
          │                                  │
          ▼                                  │
  marginal insertion + POST /instructions    │  ← the hypothesis
          │                                  │
          ▼                                  │
  command bar → live redraw + undo ◄─────────┘
          │
          ▼
  visit logging → coverage dashboard          ← the deliverable number
          │
          ▼
  Telegram/WhatsApp · goals · rep+outlet consoles
```

Stop-the-line rule: if the redraw isn't convincing, nothing downstream matters.

---

## 8. Open, decide before external pitch

- [ ] Auto-commit vs review — validate with a real ASM (`wayzy.md` §14.8)
- [ ] Model choice — run the Hinglish bench
- [ ] WhatsApp Business verification — **start day 1**, it gates nothing else
- [ ] Geocode quality on the seeded Aundh set — garbage lat-longs break §3
