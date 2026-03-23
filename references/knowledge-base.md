# Knowledge base workflow

Persistent pricing intelligence that accumulates across sessions. Every company researched, every change detected, every pattern identified — stored in a structured JSON file and available at the start of every future session.

## Storage location

```
Claude-Workspace/pricing-intelligence/knowledge-base.json
```

If the file does not exist, create it with the empty schema on first write. Never prompt the user about file creation — just create it silently.

---

## Schema

```json
{
  "meta": {
    "last_updated": "2026-03-22T09:00:00Z",
    "session_count": 0
  },
  "config": {
    "monday_pricing": {
      "last_updated": null,
      "plans": []
    },
    "slack": {
      "channel_id": null,
      "alert_mode": null,
      "last_alert": null
    },
    "experiment_defaults": {
      "smb_segment_size": null,
      "midmarket_segment_size": null,
      "enterprise_segment_size": null,
      "average_arpu": null
    }
  },
  "companies": {},
  "patterns": [],
  "experiments": []
}
```

### Company entry schema

Each key in `companies` is the PricingSaaS slug (e.g., `"notion."`):

```json
{
  "name": "Notion",
  "slug": "notion.",
  "last_researched": "2026-03-22T09:00:00Z",
  "research_count": 3,
  "category": "work-management",
  "domain": "notion.so",
  "employee_count": 2500,
  "current_pricing": {
    "snapshot_date": "2026-03-22",
    "plans": [
      {
        "name": "Free",
        "monthly": 0,
        "annual": 0,
        "model": "per user",
        "notes": "Up to 10 guests"
      },
      {
        "name": "Plus",
        "monthly": 12,
        "annual": 10,
        "model": "per user",
        "notes": ""
      }
    ],
    "free_tier": true,
    "trial": { "length_days": null, "cc_required": null },
    "pricing_model": "per-user",
    "add_ons": []
  },
  "pricing_history_summary": [
    {
      "period": "2026Q1",
      "change_types": ["Capacity Increased", "Addon Added"],
      "description": "Guest seats opened to unlimited; Custom Agents add-on launched"
    }
  ],
  "strategic_notes": "PLG-first, moving upmarket with AI add-ons. Reverse trial on Enterprise features.",
  "monday_implications": {
    "headroom": "Notion Plus at $10/seat sits $6 below monday.com Business at $16 — no upward pressure.",
    "positioning": "AI gated as add-on, not bundled — monday.com can differentiate on AI-included.",
    "threat": "If Notion bundles AI into Plus, free comparison pressure increases.",
    "experiment_ideas": ["Test AI feature inclusion at Pro tier"]
  },
  "ab_test_status": {
    "tools_detected": ["Statsig"],
    "classification": "Infrastructure present",
    "last_checked": "2026-03-22"
  },
  "sentiment_snapshot": {
    "tone": "Mixed",
    "date": "2026-03-22",
    "key_theme": "Value questioned after price increase"
  },
  "severity_baseline": {
    "segment_overlap": 2,
    "category_match": true
  }
}
```

### Pattern entry schema

```json
{
  "id": "pat-2026-03-22-001",
  "date_detected": "2026-03-22",
  "pattern": "AI add-on paywall",
  "description": "3 work-management tools moved AI features behind paid add-ons in Q1 2026",
  "companies": ["notion.", "figma.", "clickup."],
  "significance": "high",
  "monday_implication": "monday.com's AI-included positioning is now a differentiator, not table stakes"
}
```

---

## Reading the knowledge base

### When to read

At the **start of every session**, before routing to any workflow. This is triggered from `SKILL.md`.

### How to read

```
Read(path="Claude-Workspace/pricing-intelligence/knowledge-base.json")
```

If the file does not exist or is empty, proceed without KB context — the first workflow run will create it.

### What to do with the data

**For company-research and battlecard workflows:** Look up the target company in `companies`. If found:
- Surface in the output preamble: "Previously researched on {last_researched}. {research_count} prior sessions."
- Compare current pricing to `current_pricing.plans` — flag any differences as changes since last research
- Include relevant `monday_implications` from prior research as context
- Check `patterns` for any pattern involving this company

**For monitoring and weekly-digest:** Cross-reference detected changes against `companies` entries to add historical context. A price increase is more significant when the KB shows the company already raised prices in the prior quarter.

**For trend-research:** When pulling a landscape, check which companies already have KB entries. Skip re-fetching details for companies researched within the last 7 days — use cached data instead.

---

## Writing to the knowledge base

### When to write

After **every workflow** that produces findings about one or more companies. This is called from `monday-logging.md` as the final step.

### How to write

1. Read the current KB file
2. Upsert the company entry — merge new data with existing, preserving fields not updated in this session
3. Increment `meta.session_count` and update `meta.last_updated`
4. Run cross-company pattern detection (see below)
5. Write the updated JSON back to the file

```
Read(path="Claude-Workspace/pricing-intelligence/knowledge-base.json")
# ... merge data ...
Write(path="Claude-Workspace/pricing-intelligence/knowledge-base.json", contents="{updated JSON}")
```

### What to write per workflow

| Workflow | Fields to upsert |
|----------|-----------------|
| company-research | All fields: `current_pricing`, `pricing_history_summary`, `strategic_notes`, `monday_implications`, `ab_test_status`, `sentiment_snapshot` |
| monitoring | `pricing_history_summary` (append new period), `current_pricing` (if re-fetched) |
| weekly-digest | `pricing_history_summary` for each company with changes |
| trend-research | `current_pricing` and `category` for all discovered companies |
| battlecard-generator | `monday_implications` (updated competitive framing) |
| sentiment-research | `sentiment_snapshot` |
| ab-test-detection | `ab_test_status` |
| freemium-trial-tracker | `current_pricing.free_tier`, `current_pricing.trial` |
| pricing-page-teardown | `strategic_notes` (GTM motion, sophistication assessment) |
| negotiation-intelligence | append negotiation data to `strategic_notes` |
| category-watchlist | `category` and basic `current_pricing` for all added companies |

### Merge rules

- **Scalar fields** (name, category, domain): overwrite with latest
- **Array fields** (pricing_history_summary, plans): append new entries, do not duplicate existing periods
- **Object fields** (monday_implications, ab_test_status): merge keys, overwrite values
- **Timestamps** (last_researched, snapshot_date): always update to current session date
- **research_count**: increment by 1 per session that touches this company

---

## Cross-company pattern detection

Run automatically during every KB write. This is what turns isolated research sessions into compounding intelligence.

### Detection algorithm

After upserting a company entry, scan all `companies` entries for:

1. **Same-direction pricing moves in the same category:**
   - Filter companies by `category` matching the just-updated company
   - Check `pricing_history_summary` for entries within the last 180 days
   - If 3+ companies in the same category have the same `change_type` (e.g., "Price Increased"), flag as a pattern

2. **Price convergence toward monday.com:**
   - Compare each company's entry-tier price to monday.com's equivalent (from `config.monday_pricing`)
   - If 2+ competitors moved their entry price within 15% of monday.com's equivalent tier in the last 90 days, flag as "price convergence" pattern

3. **Model migration patterns:**
   - Check if multiple companies in the same category shifted pricing model (e.g., per-seat → usage-based) in the same period
   - 2+ companies making the same model shift = pattern

4. **Freemium expansion/contraction:**
   - Track `current_pricing.free_tier` changes across the category
   - If 2+ companies restricted or expanded their free tier in the same quarter, flag as pattern

### Pattern output

When a new pattern is detected, it is:
1. Appended to the `patterns` array in the KB
2. Surfaced in the current session's output as a highlighted section:

```
## Cross-company pattern detected

**Pattern:** {description}
**Companies:** {list with last-change dates}
**Significance:** {high/medium/low}
**What this means for monday.com:** {1-2 sentences}

This pattern was detected by comparing {N} companies researched across {M} sessions.
```

If the pattern already exists in the `patterns` array (same `pattern` string and overlapping `companies`), update the existing entry's `companies` list and `date_detected` rather than creating a duplicate.

---

## Querying the knowledge base

### "Show me what I know about {Company}"

1. Read the KB
2. Look up the company by name or slug
3. Present the full entry in a structured format:

```
## {Company} — knowledge base entry

**Last researched:** {date} ({research_count} sessions)
**Category:** {category}

### Current pricing
{Plan table from current_pricing.plans}

### Pricing history
{Timeline from pricing_history_summary}

### Strategic notes
{strategic_notes}

### Monday.com implications
- Headroom: {headroom}
- Positioning: {positioning}
- Threat: {threat}

### A/B testing
{ab_test_status}

### Community sentiment
{sentiment_snapshot}
```

### "What's in my knowledge base" / "KB summary"

Present a high-level summary:

```
## Pricing knowledge base — {meta.session_count} sessions

**Companies tracked:** {count}
**Patterns detected:** {count}
**Experiments in backlog:** {count}
**Last updated:** {meta.last_updated}

### Companies by category
| Category | Companies | Last updated |
|----------|-----------|-------------|
| {category} | {company list} | {most recent last_researched} |

### Active patterns
{List patterns with significance >= medium}

### Recent activity
{Last 5 companies researched with dates}
```

### "Cross-company patterns" / "What patterns have I seen"

Present all entries from the `patterns` array, sorted by date descending:

```
## Detected pricing patterns

### {Pattern 1 — most recent}
{description}
Companies: {list}
Detected: {date}
Significance: {level}
Monday.com implication: {text}

### {Pattern 2}
...
```

---

## Monday.com board sync (user-initiated)

Only runs when the user explicitly asks: "sync my knowledge base to monday", "put my pricing research on a board", "connect KB to monday.com".

### Board setup

Create a "Pricing Knowledge Base" board with columns:

```
create_board(boardName="Pricing Knowledge Base", boardKind="private", boardDescription="Accumulated pricing intelligence across all research sessions.")
```

Columns (create in parallel):

```
create_column(boardId={id}, columnType="text", columnTitle="Category")
create_column(boardId={id}, columnType="text", columnTitle="Pricing Model")
create_column(boardId={id}, columnType="numbers", columnTitle="Entry Price")
create_column(boardId={id}, columnType="text", columnTitle="Strategic Notes")
create_column(boardId={id}, columnType="date", columnTitle="Last Researched")
create_column(boardId={id}, columnType="link", columnTitle="PricingSaaS")
create_column(boardId={id}, columnType="status", columnTitle="Threat Level")
```

### Sync logic

For each company in `companies`:

1. Search for existing item by name: `get_board_items_page(boardId={id}, searchTerm="{company name}")`
2. If found: update with `change_item_column_values`
3. If not found: `create_item` with all column values populated from the KB entry

### Threat level mapping

Map `monday_implications.threat` to status labels:
- Contains "risk", "pressure", "threat" → "High"
- Contains "watch", "monitor" → "Medium"
- No threat language → "Low"

After sync, confirm: `Synced {N} companies to [{board name}](https://monday.com/boards/{id}).`

---

## Config management

### monday.com pricing reference

Stored in `config.monday_pricing`. On first access by severity scoring or hypothesis engine:

> "To score competitive changes against monday.com's pricing, I need your current plan structure. Share your pricing page URL or list your plans with prices."

After receiving the data, store:

```json
{
  "last_updated": "2026-03-22",
  "plans": [
    { "name": "Free", "monthly": 0, "annual": 0, "model": "per seat", "tier": "free" },
    { "name": "Basic", "monthly": 12, "annual": 9, "model": "per seat", "tier": "smb" },
    { "name": "Standard", "monthly": 17, "annual": 12, "model": "per seat", "tier": "smb" },
    { "name": "Pro", "monthly": 28, "annual": 19, "model": "per seat", "tier": "midmarket" },
    { "name": "Enterprise", "monthly": null, "annual": null, "model": "per seat", "tier": "enterprise" }
  ]
}
```

Refresh when the user says "update monday.com pricing" or when it's older than 90 days.

### Slack configuration

Stored in `config.slack`. Set up on first alert delivery — see [proactive-monitoring.md](proactive-monitoring.md) for the full setup flow.

### Experiment defaults

Stored in `config.experiment_defaults`. Set up on first experiment generation — see [hypothesis-engine.md](hypothesis-engine.md).
