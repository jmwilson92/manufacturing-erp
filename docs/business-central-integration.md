# Business Central → Production integration

## Principle: one system of record per concern

| Concern | System of record | Why |
|---|---|---|
| **Demand** — sales orders, forecasts, customers, items | **Business Central** | Sales/finance already live here; don't fork the truth. |
| **Supply / execution** — production orders, routing, picks, floor progress, capacity | **This app** | BC's production module is the part we're replacing. |

Demand flows **in** from BC and drives planning. Finished-good completions and (optionally)
material consumption flow **back** to BC to keep inventory honest. We never ask a planner to
re-key a sales order.

## What we pull from BC, and from where

All endpoints are the standard **API v2.0** unless noted. Base URL:

```
https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{environment}/api/v2.0/companies({companyId})/
```

| Data | Endpoint | Notes |
|---|---|---|
| Sales orders | `GET /salesOrders?$expand=salesOrderLines` | Open demand. Header + lines in one call. |
| Sales order lines | `salesOrderLines` (expanded above) | `lineObjectNumber` = item no., `quantity`, `shippedQuantity`, header `requestedDeliveryDate`. |
| Items | `GET /items` | Item master, base UoM, item category. |
| Item inventory | `GET /items({id})?$expand=...` / itemLedger | On-hand for net-requirement calc. |
| Customers | `GET /customers` | Names shown on demand feed. |
| **Demand forecast** | **Custom API page** over `Production Forecast Entry` (table 99000852) | Not in the standard API. Publish a read API page (`APIGroup=planning`), or use the Sales & Inventory Forecast / ML Forecasting extension for predicted demand. Forecast type = `Sales Item` / `Component` / `Both`. |

### Field mapping (sales order line → planning)

| BC property | Used as |
|---|---|
| `salesOrder.number` | Demand source reference (shown on planned PO) |
| `salesOrder.customerName` | Customer on demand feed |
| `salesOrder.requestedDeliveryDate` | Need-by date → backward schedule |
| `salesOrderLine.lineObjectNumber` | Item / part number |
| `salesOrderLine.quantity − shippedQuantity` | Open demand quantity |
| Forecast entry `quantity` / `forecastDate` | Forecast demand in the planning bucket |

## Authentication

OAuth2 **client-credentials** via Microsoft Entra ID (Azure AD):

1. Register an Entra app; grant the BC API app role `API.ReadWrite.All` (or read-only for pull-only).
2. Token from `https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token`, scope
   `https://api.businesscentral.dynamics.com/.default`.
3. All calls scoped to a **company** GUID within a tenant + environment.

Store tenant/client id/secret as env vars (`fly secrets set` for the deployed app); never in the repo.

## Sync strategy

- **Near-real-time:** register API **webhooks/subscriptions** on `salesOrders` so BC pushes change
  notifications → we fetch the delta. Best for "new sales order shows up on the board in seconds."
- **Delta polling fallback:** `$filter=lastModifiedDateTime gt {lastSyncUtc}` on a 2–5 min timer.
  Idempotent upsert keyed on BC `id`. Survives missed webhooks.
- **Full reconcile:** nightly full pull to catch deletes/cancellations.

Persist a `bc_sync_state` row per entity (last cursor + timestamp). The UI's "Last sync" and
"Sync now" surface this.

## Planning bridge (the new bit)

For each item, per planning bucket:

```
grossDemand   = Σ(open sales-order qty) + Σ(forecast qty)   // forecast consumed by actual SOs
netRequirement = max(0, grossDemand − onHand − onOrder)     // onOrder = open production orders here
```

`netRequirement > 0` → suggested production order. Planner clicks **Plan PO**, which creates a
routed production order pre-linked to the originating BC sales order(s)/forecast. From there the
order runs the existing lifecycle (route → pick → sign-off → floor → close).

> This mirrors BC's planning worksheet / MRP, but the output lands in *our* execution workflow
> instead of BC production orders.

## Write-back to BC (phase 2)

| Event here | Write to BC |
|---|---|
| Production order completed | `itemJournal` output posting → increments finished-good inventory |
| Material issued at pick | `itemJournal` consumption posting (optional; or keep components in BC) |
| Order status change | Custom field / status API on the linked sales order for visibility |

Write-back is opt-in per environment so a pilot can run **read-only** first.

## Prototype ↔ real feed

In `production-orders/prototype.html`, `BC_SALESORDERS`, `BC_FORECAST`, and `ON_HAND` use the exact
BC property names above. Swapping the mocks for a real feed is a 1:1 replacement of those three
constants with the authenticated GET responses — the planning math and UI don't change.
