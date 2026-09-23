# V1 — three modules, three devs

One dev per module. Each owns its slice end to end (engine, API and UI),
so nobody is blocked waiting for someone else's layer.

```
   ┌────────────────────────────────────────────────────────────────┐
   │  B  ·  INSTRUCTION LOOP                                         │
   │  command bar · Telegram · LLM stages 1/2/4 · undo · masters      │
   └──────────────────┬──────────────────────────┬──────────────────┘
                      │ solve(resolved)          │ plan:updated
                      ▼                          ▼
   ┌──────────────────────────────┐   ┌────────────────────────────┐
   │  A  ·  PLAN ENGINE            │   │  C  ·  MAP & EVIDENCE       │
   │  matrix · insertion · CVRPTW  │◄──┤  drag → solve(manual)       │
   │  models · db · seed           │   │  MapLibre · visits · coverage│
   └──────────────────────────────┘   └────────────────────────────┘
                      └──────── PlanDiff ────────►

   A is called by both. A calls nobody. No LLM ever enters A.
```

---

## A · Plan engine

> The hard maths and the shared foundation. No UI, no LLM, no network.

```
  OWNS      models.py · db.py · scripts/seed_db.py
            data/matrix.json  (OSRM run once, committed)
            services/insertion.py   ← daily exceptions
            services/zoning.py      ← periodic CVRPTW
            services/routing.py     ← TSP within a zone
            GET /zones/{date} · POST /zones/generate

  EMITS     PlanDiff · Zone payload

  TESTS     insertion is pure and deterministic — same input, same output,
            no clock, no randomness. This module should be the only one
            with real unit tests.
```

**Ships day 1:** `models.py` + seed. B and C are blocked on nothing else. Frozen after, changed only by a PR all three review.

---

## B · Instruction loop

> The hypothesis. Small in code, highest in risk.

```
  OWNS      services/ai_pipeline.py   stages 1, 2, 4
            POST /instructions        orchestration + undo
            GET  /dashboard/instructions
            services/telegram.py      (WhatsApp swaps in later)
            UI: command bar · diff panel · clarification · undo
            UI: rep console · outlet console · goals

  CALLS     A.solve(resolved_instruction) → PlanDiff
  EMITS     window event `plan:updated` carrying the PlanDiff

  RULE      stage 4 runs AFTER the event is emitted. Never await the
            summary before telling C to redraw.
```

Masters and goals sit here because B's pipeline is thin in code volume — this is what balances the three.

---

## C · Map & evidence

> The wow moment and the deliverable number.

```
  OWNS      UI: MapLibre — zone polygons, route polylines, numbered stops,
                 pin states, ETA per stop, legend
            UI: live redraw animation  ← the demo turns on this
            UI: drag-to-reassign       → calls A, same path as an instruction
            UI: rep visit screen (responsive web)
            POST /visits · GET /dashboard/coverage
            UI: coverage dashboard — self-reported vs GPS-verified

  LISTENS   `plan:updated` → animate to the new PlanDiff
  CALLS     A.solve(manual_move) on drag
```

---

## Frozen day 1 — the three contracts

Agree these before anyone writes code. All three devs then build against
`fixtures/`, not against each other.

### 1 · PlanDiff — A → B, A → C

```json
{
  "instruction_id": 42,
  "mode": "insertion",
  "reps": [
    { "rep_id": 2, "name": "Amit",
      "added":   [ {"outlet_id": 17, "name": "Balaji Kirana", "class": "A", "position": 3} ],
      "removed": [],
      "route":   [11, 4, 17, 9],
      "minutes_before": 402, "minutes_after": 463,
      "km_before": 18.2,     "km_after": 29.4 }
  ],
  "deferred": [
    { "outlet_id": 88, "name": "Sai General", "class": "C",
      "defer_to": "2026-09-25", "reason": "no feasible slot" }
  ],
  "unchanged_rep_ids": [5],
  "summary": null
}
```

`summary` is null when C receives it. B fills it in after the map has redrawn.
`deferred` is never empty-by-omission — an empty list means nothing was dropped, and the UI must say so.

### 2 · Zone payload — A → C

```
  zone_id · rep_id · valid_date · stops[ {outlet_id, visit_order, eta_minutes} ]
```

### 3 · Redraw event — B → C

```
  window.dispatchEvent(new CustomEvent('plan:updated', { detail: <PlanDiff> }))
```

---

## Integration checkpoints

```
  ①  CONTRACTS FROZEN     fixtures/plan_diff.sample.json committed
      day 1               all three now work in parallel

  ②  FIRST REAL REDRAW    B's bar → A's insertion → C's map, seeded data
      as early as possible  ◄── STOP THE LINE
                          if this isn't convincing, the plan changes

  ③  DEMO DRY RUN         wayzy.md §10, start to finish, on a laptop
```

---

## Rules

- **Nobody waits.** Fixtures exist so all three can build from hour one.
- **A never imports B or C.** If the engine needs to know where a request came from, the boundary is wrong.
- **One shared file:** `models.py`. Changes to it are a three-way PR, not a push.
- **Checkpoint ② is the project.** Everything else is scaffolding around it.
