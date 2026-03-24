# Curation Protocol — Draft Plan

The implementation that we've so far has been based on the media-buy protocol. The problem is it is meant for buying ads and not Inventory. This is the initial plan for our proposal of the Curation/Deals Protocol for AdCP.

Curation is fundamentally different. The agent needs create inventory packages from audience segments and activate them as Deals on SSPs (Magnite, Pubmatic and many more).

The idea is to create the curation protocol as sibling to https://docs.adcontextprotocol.org/docs/media-buy .

| | Media Buy | Curation |
|---|---|---|
| Seller | Publisher (owns inventory) | Curator (Pubx in this case) |
| Execution | Publisher's ad server | SSP platform |
| Core entity | Product (inventory slot) | Segment (audience/context def) |
| Transaction | Campaign / media buy | Deal on SSP |
| Pricing | CPM/CPC/CPA | Not sure. We need to discuss this |

---

## Flow

**3 steps**: Discover segments → Select segments → Activate deal

```
Buyer Agent ──(Curation Protocol)──> Curation Agent ──(SSP API)──> SSP Platform
                                          │                            │
                                     Catalog Service               Deal Creation
                                     (segments, rules)
```

---

## Tasks

Initially I've kept the tasks/tool in a similar standard to media buy as both of the protocols are similar in nature:

| Phase | Curation Task | Media Buy Equivalent | Purpose |
|-------|--------------|---------------------|---------|
| Discovery | `get_segments` | `get_products` | Find inventory |
| Discovery | `list_platforms` | `list_creative_formats` | List available SSP platforms |
| Execution | `create_deal` | `create_media_buy` | Activate |
| Execution | `update_deal` | `update_media_buy` | Modify (PATCH) |
| Monitoring | `get_deals` | `get_media_buys` | List/status |
| Monitoring | `get_deal_delivery` | `get_media_buy_delivery` | Delivery metrics |
| Monitoring | `provide_performance_feedback` | `provide_performance_feedback` | Optimization loop |

**MUST haves**: `get_segments`, `list_platforms`, `create_deal`, `get_deals`

**GOOD to haves**: `update_deal`, `get_deal_delivery`, `provide_performance_feedback`

---

## Task details

### get_segments

Three buying modes (similar to media-buy's `brief` / `wholesale` / `refine`):

| Mode | Input | Use case |
|------|-------|----------|
| `brief` | Natural language description | "Premium sports video in US for auto brand" |
| `catalog` | Structured filters | Browse by country, device, ad type, etc. |
| `refine` | Previous segments + instructions | "More like seg_123, drop seg_456, premium only" |

**Key filters**: countries, devices, ad_format_types, platforms, categories, min_daily_impressions, budget_range, date range

**Returns**:
- `segments[]` — each with: segment_id, name, description, targeting, pricing_guidance, supported_platforms, supported_ad_format_types, status, brief_relevance
- `proposals[]` (optional) — curator-suggested (pubx_suggested) deal packages with budget allocations across segments

### list_platforms

Returns supported SSPs with their capabilities:
- platform_id, name
- supported_deal_types (`PMP`, `Curated`)
- supported_ad_format_types (`Banner`, `Video`, `Native`, `Audio`)
- supported_dsps (DSP integrations available on that SSP) eg: Magnite supports DV360 ?!?! **NOT SURE ON THIS, NEED MORE FEEDBACK**

### create_deal

Two input modes (mutually exclusive — provide one OR the other, not both):
- **Direct**: provide `segments[]` (selected segment IDs)
- **Proposal shortcut**: provide `proposal_id` + `total_budget` (from get_segments proposal)

**Required fields**: buyer_ref (idempotency key — dedup on buyer_ref + account, safe retries without duplicate SSP deals), platform (SSP), dsps[] (including buyer's `seat_id` per DSP), pricing (type + cpm + currency), start_time, end_time

**Unique/New in curation**:
- `platform` — which SSP to activate on
- `dsps[]` — which DSPs receive the deal

**Request side**: buyer provides `dsps[].seat_id` (their identity on the SSP for that DSP)
**Response side**: curator confirms activation with `ssp_deal_id`

### update_deal

PATCH semantics — only send fields to change. Supports:
- Dates, pricing **NOT SURE ON THIS, NEED MORE FEEDBACK**
- `paused: true/false` for pause/resume

### get_deals

List and filter deals by: deal_id, buyer_ref, status, platform with pagination.

### get_deal_delivery

Delivery metrics for a deal: status, impressions, spend, bids, wins, win_rate, avg_cpm. Optional daily breakdown.

### provide_performance_feedback (Not sure if we should keep this)

Buyer sends DSP-side metrics (impressions, clicks, CTR, conversions, viewability) to help curator optimize/update the deal further? **NOT SURE ON THIS, NEED MORE FEEDBACK**

---

## Core entities

### Segment (Or Package, whatever we prefer)
- Atomic unit of curated inventory
- Defines targetable audience/context through rules (geo, contextual, behavioral, device)
- **Discovered** through the protocol, not created. Curator manages the catalog internally.
- Has: targeting, estimated_scale, pricing_guidance, supported_platforms

### Deal
- Transactional entity. Represents a Deal on an SSP
- Created from one or more segments
- Lifecycle: `pending_activation` → `active` → `paused` / `completed` / `failed`
- **Distinct identifiers**:
  - `deal.id` — protocol handle, used in all protocol calls (update_deal, get_deals, etc.). This would be sale_id for us based on the sales service.
  - `deal.platform.ssp_deal_id` — SSP's deal ID, generated by the SSP on activation
  - `deal.dsps[].seat_id` — Buyer's identity on the SSP for that DSP Platform (provided by buyer in request)

### SSP
- Where deals activate (Magnite, Pubmatic, etc.)
- Each has specific deal types, ad types, DSP integrations ?!? **NOT SURE ON THIS, NEED MORE FEEDBACK**

---

## Deal status lifecycle

```
pending_approval → pending_activation → active → paused / completed / failed / rejected
```

---

## Key design decisions

- **Union responses**: errors OR data, never both (same as media-buy)
- **Async**: `completed` (immediate), `working` (<120s), `submitted` (hours/days, HITL)
- **Transport**: MCP preferred, A2A supported
- **Capability declaration**: Declared in `get_adcp_capabilities`. Something like `deal_creation`
- **Extensions**: NEEDS MORE RESEARCH

---

## Error codes

| Code | When |
|------|------|
| `SEGMENT_NOT_FOUND` | Referenced segment doesn't exist |
| `DEAL_NOT_FOUND` | Referenced deal doesn't exist |
| `PLATFORM_NOT_SUPPORTED` | SSP not supported for this segment |
| `DSP_NOT_SUPPORTED` | DSP not available on this platform |
| `INVALID_PRICING` | Pricing invalid for platform/deal type |
| `INVALID_DATES` | Bad date range |
| `INVALID_STATE` | Operation not allowed for current deal status |
| `BUDGET_EXCEEDED` | Exceeds authorized budget |
| `ACTIVATION_FAILED` | SSP activation error |

---

## Example flow (brief-based)

```
1. get_segments  { buying_mode: "brief", brief: "Premium sports video in US for auto brand, Q2" }
2. ← segments[] + proposals[]
3. create_deal   { proposal_id: "prop_001", total_budget: $40k, platform: "magnite",
                   dsps: [{ id: 2249, name: "DV360", seat_id: "756032" }] }
4. ← id: "deal_abc", platform.ssp_deal_id: "MG-756032", dsps: [{ seat_id: "756032", status: "active" }]
5. Deal is live — DV360 seat 756032 can bid on Magnite deal MG-756032
```

---

## Open questions

### 1. Multi-platform deals
Should one `create_deal` call activate on multiple SSPs?
- **Option A**: Single platform per call, buyer makes multiple calls
- **Option B**: Accept `platforms[]`, return per-platform results
- Tradeoff: simplicity vs round trips

### 2. Segment/Deal lifecycle notifications
Should curators (seller agent) proactively notify when segments/deal updates mid-flight? How:
- **Option A**: Buyers poll via `get_deals`
- **Option B**: Webhook for segment & deal availability changes

### 3. $$ Transparency
How is the curator making money on this and should the curator (seller agent) be open about it in the pricing options (like including margin details and so on)

---

## Schemas we need to create

| Schema | Description |
|--------|-------------|
| `curation/get-segments-{request,response}.json` | Segment discovery |
| `curation/list-platforms-{request,response}.json` | Platform capabilities |
| `curation/create-deal-{request,response}.json` | Deal creation |
| `curation/update-deal-{request,response}.json` | Deal modification |
| `curation/get-deals-{request,response}.json` | Deal listing |
| `curation/get-deal-delivery-{request,response}.json` | Delivery metrics |
| `curation/provide-performance-feedback-{request,response}.json` | Performance feedback |
| `core/segment.json` | Segment entity |
| `core/deal-pricing.json` | Pricing model |
