# Business Central ↔ Flow integration

## The dividing line

Business Central keeps everything **up to and including release**. Flow owns everything
**after release** — and posts the results back. Nothing is re-keyed at the boundary.

```
  BUSINESS CENTRAL  (system of record for demand + planning)
  ─ Forecast ─ Sales orders ─ Production orders (from BOMs)
                                   Planned → Firm Planned → Released
                                                              │
                                          RELEASED  ──────────┤  ← Flow picks up here
                                                              │
  FLOW  (system of record for execution)                     ▼
  ─ Kitting (the production BOM) ─ Routing execution ─ Sign-offs ─ Labor time ─ Finished
        │                 │                    │
        ▼                 ▼                    ▼
   Consumption       Output journal      Status → Finished     ← posted back to BC
     journal        (qty + run time)
```

Production orders are built from a **production BOM** (components + routing) inside BC — never
"by product" in Flow. Flow reads the released order, its BOM, and its routing; it does not create
or plan them.

## Why "Released" is the handoff

In BC a production order can be **Simulated → Planned → Firm Planned → Released → Finished**.
Released is the *only* status where the shop floor can register **consumption** and **output** and
where flushing occurs. That is exactly Flow's job, so Released is the natural pickup point:

- **Planned / Firm Planned** — planner's territory, dates still moving. Flow shows them read-only.
- **Released** — BC hands off. Flow kits, runs, signs off, and posts back.
- **Finished** — Flow closes it and flips the BC status.

## What Flow pulls from BC

Base URL: `https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{environment}/api/v2.0/companies({companyId})/`

| Data | Source | Notes |
|---|---|---|
| Released production orders | **Custom API page** over `Production Order` (5405), filtered `Status = Released` | Header: no., item, quantity, due date, source sales order. |
| Production BOM lines (the kit) | **Custom API page** over `Prod. Order Component` (5407) | Item, quantity-per, expected qty, bin. Becomes the kitting list. |
| Routing operations | **Custom API page** over `Prod. Order Routing Line` (5409) | Operation no., work/machine center, setup + run time. Becomes the traveler. |
| Sales orders (demand context) | Standard `salesOrders` + `salesOrderLines` | For the demand view + linking an order back to its customer. |
| Demand forecast | **Custom API page** over `Production Forecast Entry` | Not in the standard API. Sales Item / Component / Both. |
| Items / inventory | Standard `items` | On-hand for the demand view's net-requirement display. |

> The manufacturing tables (5405/5407/5409) are **not** in the standard API — they require custom API
> pages (an AL extension exposing read pages under an `APIGroup`). Sales orders, items, and customers
> are standard v2.0.

## What Flow posts back to BC

This is core to the design, not a later phase — the whole point is that manipulating the order in
Flow keeps BC correct.

| Flow action | Posted to BC |
|---|---|
| **Kitting complete** — BOM issued to the floor | **Consumption journal** (item journal, entry type Consumption) against the prod. order components |
| **Operation signed off** | **Output journal** — output quantity + **run time** for that routing line, tagged with the operator |
| **Order finished** | Status change **Released → Finished** on the production order |
| Scrap reported | Scrap quantity on the output posting |

Posting uses custom **unbound API actions** (AL codeunits exposed as APIs) that wrap
`Prod. Order Journal` / item-journal posting, because the standard API doesn't post these journals.
Each post is idempotent (keyed on Flow's operation + a client token) and retried with backoff; the
UI's "Posted to Business Central" feed shows the running log.

## Sign-offs & labor time (Flow-native)

BC has no first-class "operator sign-off with labor by person" concept on the routing line — this is
the gap Flow fills and the reason execution lives here:

- Each operation is **assigned to an operator**; a run timer captures actual labor minutes.
- **Sign-off** stamps the operation with who completed it (QA operations stamp the inspector) and when.
- Labor rolls up per operator per order and is what the **output journal** carries back as run time,
  so BC's cost/capacity numbers reflect what actually happened on the floor.

## Auth & sync

- **OAuth2 client-credentials** via Microsoft Entra ID; token scope
  `https://api.businesscentral.dynamics.com/.default`; all calls scoped to tenant → environment →
  company. Secrets in env vars (`fly secrets set`), never in the repo.
- **Inbound:** webhook/subscription on the production-order API page for near-real-time "released"
  events, plus a `lastModifiedDateTime` delta poll (2–5 min) as a fallback, and a nightly full
  reconcile for cancellations/reopens.
- **Outbound:** post consumption/output/status as the operator acts; queue + retry so a BC hiccup
  never blocks the floor.

## Prototype mapping

`production-orders/prototype.html` models this exactly:

- `FAMILIES[*].bom` / `.routing` stand in for `Prod. Order Component` / `Prod. Order Routing Line`.
- Orders enter at **Firm Planned/Released** and Flow drives kitting → floor → finished.
- Every kit/sign-off/finish calls `wbPost(...)`, the stand-in for the BC write-back actions; the
  Command center and Floor "Posted to Business Central" panels render that log.

Swapping mocks for a real feed replaces the seed data + `wbPost` with authenticated GET/POST calls —
the stage machine, kitting, sign-off, and labor logic are unchanged.
