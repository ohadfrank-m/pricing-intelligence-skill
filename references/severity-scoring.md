# Severity scoring framework

Score every detected pricing change by its competitive impact on monday.com. The score determines alert urgency, digest prioritization, and experiment generation priority.

---

## Prerequisites

Severity scoring requires monday.com's own pricing as a reference. On first use, check `config.monday_pricing` in the knowledge base (`Claude-Workspace/pricing-intelligence/knowledge-base.json`).

If `config.monday_pricing.plans` is empty or `last_updated` is older than 90 days:

> "To score competitive changes against monday.com, I need your current plan structure. Share your pricing page URL or list your plans with prices — I'll store this for future sessions."

After receiving the data, write it to `config.monday_pricing` per the schema in [knowledge-base.md](knowledge-base.md).

---

## Scoring dimensions

Every detected change is scored on 4 dimensions. Each dimension scores 0–3 points. Maximum total: 12.

### Dimension 1: Price proximity to monday.com (0–3)

Does this change move the competitor's price closer to or further from monday.com's equivalent tier?

**How to evaluate:**

1. Identify the competitor's changed plan/tier
2. Match it to the closest monday.com tier by segment (use `config.monday_pricing.plans` and the `tier` field: free, smb, midmarket, enterprise)
3. Calculate the price delta before and after the change

| Scenario | Score |
|----------|-------|
| Change moves competitor's price to within 10% of monday.com's equivalent tier (undercut or parity) | 3 |
| Change moves competitor's price to within 25% of monday.com's equivalent tier | 2 |
| Change moves competitor's price closer to monday.com but still >25% gap | 1 |
| Change moves price further from monday.com, or no comparable tier exists | 0 |

**Special cases:**
- Competitor introduces a new tier that directly overlaps with a monday.com tier in price + features → score 3
- Competitor removes a plan, forcing users toward a tier that competes with monday.com → score 2
- Enterprise "contact us" pricing with no disclosed price → score 0 (unscoreable on price)

### Dimension 2: Segment overlap (0–3)

Does this company compete for the same buyer as monday.com?

| Scenario | Score |
|----------|-------|
| Direct competitor in work management / project management for same segment | 3 |
| Adjacent category (CRM, collaboration, docs) that competes in deals | 2 |
| Same vertical (B2B SaaS tools) but different use case, occasional competitive overlap | 1 |
| No competitive overlap (different vertical, different buyer entirely) | 0 |

**Shortcut:** If the company is in the knowledge base, use `severity_baseline.segment_overlap` for a cached score. Update the cache if the company's positioning has changed.

**Default segment overlap scores for common categories:**

| Category | Default score |
|----------|--------------|
| Work management (Asana, ClickUp, Wrike, Smartsheet) | 3 |
| Project management (Jira, Linear, Shortcut, Height) | 2 |
| CRM (HubSpot, Salesforce, Pipedrive) | 2 (monday CRM competes directly) |
| Collaboration / docs (Notion, Coda, Confluence) | 2 |
| Customer support (Zendesk, Intercom, Front) | 1 |
| Data / BI (Tableau, Looker, Metabase) | 0 |
| Sales engagement (Outreach, Salesloft) | 1 |

### Dimension 3: Change magnitude (0–3)

How structurally significant is the pricing move?

| Change type | Score |
|-------------|-------|
| Pricing model restructure (e.g., per-seat → usage-based, dual-metric introduction) | 3 |
| Price change >20% on any plan | 3 |
| Plan removed or added | 3 |
| Free tier removed or introduced | 3 |
| Price change 10–20% | 2 |
| Plan renamed with accompanying price adjustment | 2 |
| Add-on introduced at >$10/seat/mo | 2 |
| Price change <10% | 1 |
| Feature moved between tiers | 1 |
| Add-on introduced at <$10/seat/mo | 1 |
| Feature added or changed within existing tier | 0 |
| Capacity change on non-core metric | 0 |

### Dimension 4: Strategic signal (0–3)

Does this change signal a GTM motion shift or competitive positioning move?

| Signal | Score |
|--------|-------|
| Freemium introduced in a previously paid-only product | 3 |
| AI pricing model change (new AI add-on, per-resolution pricing, AI credits introduced) | 3 |
| Direct competitive positioning change (mentions monday.com or "work management" in pricing page copy) | 3 |
| Reverse trial introduced | 3 |
| GTM motion shift visible (PLG → SLG or vice versa, indicated by CTA/trial changes) | 2 |
| Pricing page A/B test detected coinciding with the change | 2 |
| Multi-year / volume discount structure changed | 1 |
| Feature gating changed without price impact | 1 |
| Routine annual price adjustment consistent with prior years | 0 |
| Cosmetic or copy-only change | 0 |

---

## Severity tiers

| Tier | Score range | Response | Alert action |
|------|------------|----------|-------------|
| **P0** | 10–12 | Direct competitive threat requiring immediate awareness | Alert immediately per user preference |
| **P1** | 7–9 | Significant move worth knowing today | Include in daily batch alert |
| **P2** | 4–6 | Noteworthy, relevant for strategic context | Include in weekly digest only |
| **P3** | 0–3 | Noise — log for pattern detection | Log to KB, no alert |

---

## Scoring output format

For every scored change, append the severity assessment to the change description:

```
**Severity: P{tier} ({score}/12)**
- Price proximity: {score}/3 — {1-sentence rationale}
- Segment overlap: {score}/3 — {1-sentence rationale}
- Change magnitude: {score}/3 — {1-sentence rationale}
- Strategic signal: {score}/3 — {1-sentence rationale}
```

---

## Scoring examples

### Example 1: P0 — Asana undercuts monday.com Business

**Change:** Asana drops Premium from $13.49/seat to $10.99/seat (annual).

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Price proximity | 3 | Asana Premium at $10.99 is now 8% below monday.com Standard at $12 — direct undercut |
| Segment overlap | 3 | Direct work management competitor, same mid-market buyer |
| Change magnitude | 2 | 18.5% price decrease |
| Strategic signal | 2 | Signals aggressive self-serve acquisition push |
| **Total** | **10** | **P0 — alert immediately** |

### Example 2: P2 — Notion adds AI add-on

**Change:** Notion introduces "Notion AI" as a $10/seat/mo add-on for Plus and above.

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Price proximity | 0 | Notion Plus base price unchanged; add-on doesn't affect tier comparison |
| Segment overlap | 2 | Adjacent category, competes in collaboration deals |
| Change magnitude | 1 | Add-on introduced at $10/seat |
| Strategic signal | 3 | AI pricing model change — sets a market reference for AI add-on pricing |
| **Total** | **6** | **P2 — include in weekly digest** |

### Example 3: P3 — Tableau changes feature gating

**Change:** Tableau moves "Ask Data" feature from Enterprise to Professional tier.

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Price proximity | 0 | Different category, no tier comparison applicable |
| Segment overlap | 0 | No competitive overlap with monday.com |
| Change magnitude | 1 | Feature moved between tiers |
| Strategic signal | 0 | Routine packaging adjustment |
| **Total** | **1** | **P3 — log only** |

---

## Integration points

This framework is called by:

- **[proactive-monitoring.md](proactive-monitoring.md):** Scores every change detected in the daily monitoring sweep
- **[monitoring.md](monitoring.md):** Scores changes in manual monitoring checks
- **[weekly-digest.md](weekly-digest.md):** Uses severity scores to sort and prioritize the digest
- **[hypothesis-engine.md](hypothesis-engine.md):** Uses severity tier to set `competitive_urgency` on generated experiments
