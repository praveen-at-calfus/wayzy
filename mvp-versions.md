# Wayzy — version plan

The whole build, cut into three. Each version has one job and one gate.
Nothing moves to the next section until its gate is passed.

```
   V1  DEMO                V2  PILOT                V3  PRODUCT
   ────────                ─────────                ──────────
   prove the loop   ───►   survive real data ───►   make it sellable

   seeded Aundh            3–5 distributors         multi-tenant
   1 territory, 4 reps     real outlet masters      integrations
   Telegram                WhatsApp                 native app
   local / free tier       hosted, real auth        second vertical

   GATE                    GATE                     GATE
   the 7 criteria          a real coverage-         design partners
   in wayzy.md §13         recovery number          convert to paid
```

---

# V1 — DEMO

**Job:** a manager describes a disruption in plain language, the plan redraws live, on data a Pune distributor recognises as their own.

### Ships

```
  DATA        seed script · Aundh · 4 reps · ~160 outlets
              data/matrix.json (OSRM, build-time)

  ENGINE      marginal insertion    → today's exceptions
              full CVRPTW           → periodic PJP regeneration
              stages 1–4 pipeline   → LLM at 1 and 4 only

  UI          MapLibre: zones, routes, pin states, live redraw
              command bar + diff + clarification + 1-tap undo
              rep + outlet consoles · goals · coverage dashboard
              rep visit screen (responsive web)

  CHANNEL     Telegram bot
```

### Does not ship

| | Why |
|---|---|
| WhatsApp Cloud API | Meta verification unbounded — started day 1, lands when it lands |
| Multi-distributor tenancy | one territory is the whole demo |
| Auth beyond a shared password | nothing real is at stake yet |
| Offline sync, native app, Tally, multilingual | `wayzy.md` §3 deferred list |

### Gate

- [ ] All 7 success criteria, `wayzy.md` §13
- [ ] Redraw under 5s, warm cache, on a laptop with hotel wifi
- [ ] **"Ask vs instruct" validated** with a live competitor demo + one working ASM (§2.1, §14.8) — *before* this goes in any pitch

---

# V2 — PILOT

**Job:** find out what real data and real reps do to it. The V1 demo is a controlled environment; this is not.

### Ships

```
  CHANNEL     WhatsApp Cloud API (Telegram stays as fallback)

  DATA        bulk import that survives a real outlet master
              geocode repair + "walk and pin" flow
              multi-distributor tenancy · real auth · audit log

  ENGINE      insertion tuning against actual absence days
              deferred-list behaviour under real backlog pressure

  UI          offline-tolerant rep screen (queue + retry, not full sync)
              manager review of the periodic rebuild
```

### Does not ship

Native app · Tally/Marg · multilingual · order-booker/delivery-van split · second vertical.

### Gate

- [ ] 3–5 distributors, 4–8 weeks, real data (`wayzy.md` §14.1)
- [ ] **A real coverage-recovery number** from real absence days — the pitch is this number
- [ ] An ASM opens it unprompted, for several consecutive weeks, without being asked to

### Expect to learn

```
  geocode quality          →  how much of the master is unusable
  insertion acceptance     →  do managers apply it, or override it
  deferral tolerance       →  is the deferred list read, or ignored
  auto-commit vs review    →  settles the mvp-v1.md §8 open question
```

---

# V3 — PRODUCT

**Job:** everything V2 proved is needed but was deliberately skipped.

### Ships

```
  1  Tally / Marg read-only connector — party master + outstanding dues
  2  React Native offline-first app, replacing the web rep screen
  3  Multilingual / voice instruction input
  4  Order-booker vs delivery-van role modelling
  5  Add-on mode — API integration for BeatRoute / FieldAssist / Salesforce
  6  Live-traffic refinement on affected legs (wayzy.md §6.1.1, step 2)
  7  Second vertical — NBFC field collections
```

Order is not fixed. V2's learnings re-rank this list; do not commit to it in advance.

### Gate

- [ ] Design partners convert to paid
- [ ] Item 5 decided: **standalone product or add-on?** — the answer changes the company, not just the roadmap

---

## Where each doc applies

```
  wayzy.md       scope · data model · demo script      V1
  mvp-v1.md      build decisions · dev split           V1
  mvp-versions.md (this)                               V1 → V3
```
