# Hypothesis engine workflow

Convert competitive pricing signals into testable experiment hypotheses. Every "So what for monday.com" section generates a structured experiment spec with a metric, a tier, and an estimated ARR impact range. Over time, this builds a prioritized experiment backlog ranked by competitive urgency and business impact.

Experiments are stored in `knowledge-base.json` under the `experiments` array — see [knowledge-base.md](knowledge-base.md) for the schema.

---

## When to run

### Automatic (after every relevant workflow)

Run silently after any workflow that produces a "So what for monday.com" section:
- `company-research` — after delivering the main output
- `monitoring` — after scoring and logging changes
- `weekly-digest` — after composing the digest
- `battlecard-generator` — after building the battlecard
- `proactive-monitoring` — after delivering P0/P1 alerts

### User-triggered

- "Show my experiment backlog" / "what experiments should I run"
- "Review experiment backlog" / "quarterly experiment review"
- "Mark experiment {id} as launched/completed/cancelled"
- "Sync experiments to monday"

---

## Step 1: Extract actionable signals

Parse the "So what for monday.com" section from the calling workflow's output. Look for:

1. **"Experiment or test to consider" bullet** — this is the primary input. Extract the hypothesis, the metric, and the tier mentioned.
2. **"Positioning implication" bullet** — if it suggests a packaging change (e.g., "lead with AI-included-by-default"), translate to a testable experiment.
3. **"Threat signal" bullet** — if it identifies a competitive risk that could be mitigated by a pricing/packaging move, translate to a defensive experiment.
4. **"Pricing headroom" bullet** — if it identifies room to raise prices, translate to a price elasticity test.

Skip generating an experiment if the "So what" section contains only observational notes with no actionable implication (e.g., "No competitive overlap at this tier").

---

## Step 2: Map to experiment template

For each extracted signal, populate the experiment template:

```json
{
  "id": "exp-{YYYY-MM-DD}-{NNN}",
  "created": "{ISO date}",
  "source_company": "{slug}",
  "source_signal": "{1-sentence description of the competitive signal}",
  "source_workflow": "{workflow name}",
  "hypothesis": "{If we [action], then [metric] will [direction] by [range] because [reasoning]}",
  "test_design": {
    "mechanism": "{How to test — feature flag, A/B test, cohort rollout, etc.}",
    "metric": "{Primary metric to measure}",
    "secondary_metrics": ["{guardrail metrics}"],
    "tier_affected": "{Free / Basic / Standard / Pro / Enterprise}",
    "duration": "{recommended test duration}",
    "expected_impact_range": "{e.g., +5-8% upgrade rate}"
  },
  "competitive_urgency": "{critical / high / medium / low}",
  "estimated_arr_impact": "{dollar range}",
  "status": "backlog",
  "last_reviewed": null,
  "priority_score": null,
  "reinforcing_signals": [],
  "notes": ""
}
```

### Field generation rules

**id:** Format `exp-{YYYY-MM-DD}-{NNN}` where NNN is a zero-padded sequential number for that day. Read existing experiments to determine the next number.

**hypothesis:** Follow the structure: "If we [specific action on monday.com pricing/packaging], then [specific metric] will [increase/decrease] by [estimated range] because [reasoning based on competitive signal]." Never use vague language — every hypothesis must name the tier, the metric, and the expected direction.

**test_design.mechanism:** Default recommendations by experiment type:

| Experiment type | Recommended mechanism |
|----------------|----------------------|
| Feature gating change (move feature between tiers) | Feature flag — expose feature to test cohort at target tier |
| Price point change | Cohort test — new price for new signups only, measure conversion delta |
| Packaging restructure | A/B test on pricing page — show new structure to 50% of traffic |
| Free tier change | Reverse trial or limit change for new accounts, measure activation + conversion |
| Add-on introduction | Soft launch to subset of eligible accounts, measure attach rate |

**test_design.duration:** Default to 4 weeks for conversion experiments, 8 weeks for retention experiments, 2 weeks for pricing page A/B tests.

**competitive_urgency:** Mapped from the triggering change's severity score:

| Severity tier | Urgency |
|--------------|---------|
| P0 (10–12) | critical |
| P1 (7–9) | high |
| P2 (4–6) | medium |
| P3 (0–3) | low |

If no severity score is available (e.g., generated from a company-research "So what" section without monitoring), default to "medium."

---

## Step 3: Estimate ARR impact

Use a simple formula:

```
ARR impact = affected_accounts * conversion_delta * ARPU * 12
```

### Getting the inputs

**affected_accounts:** Use segment size estimates from `config.experiment_defaults` in the KB. On first use, if these are empty, ask:

> "To estimate experiment impact, I need approximate segment sizes for monday.com. These are stored once and reused:"
>
> - SMB self-serve accounts (Free + Basic + Standard): ___
> - Mid-market accounts (Pro): ___
> - Enterprise accounts: ___
> - Average revenue per account (ARPU): ___

Store the responses in `config.experiment_defaults`.

**conversion_delta:** Estimate based on the experiment type and competitive signal:

| Signal type | Typical delta range |
|-------------|-------------------|
| Feature moved down a tier (competitor did it successfully) | +3–8% upgrade rate |
| Price decrease matching competitor | +5–12% conversion rate |
| Free tier expansion | +10–20% signup volume, -2–5% conversion rate |
| AI feature inclusion at lower tier | +5–10% upgrade rate for AI-engaged users |
| Add-on introduction | 8–15% attach rate among eligible accounts |
| Pricing page optimization | +2–5% page-to-trial conversion |

These are default ranges. Adjust based on the specific competitive signal and market context.

**ARPU:** Use `config.experiment_defaults.average_arpu`. If unavailable, use $180/account/year as a conservative SaaS default and flag the assumption.

### Output format

Present the ARR impact as a range, not a point estimate:

```
Estimated ARR impact: $800K–$1.4M
(Based on: {N} mid-market accounts × {X}–{Y}% conversion lift × ${ARPU} ARPU × 12 months)
```

---

## Step 4: Deduplicate

Before appending the new experiment, check the existing `experiments` array in the KB:

1. Look for experiments with the same `tier_affected` AND similar `mechanism` (fuzzy match on keywords: "SSO", "AI", "free tier", "price", etc.)
2. If a match is found:
   - Do NOT create a duplicate
   - Instead, append the new `source_signal` to the existing experiment's `reinforcing_signals` array
   - Update `competitive_urgency` if the new signal has a higher urgency
   - Update `last_reviewed` to today's date
   - Surface in the output: "This reinforces existing experiment {id}: {hypothesis}. Now supported by {N} competitive signals."
3. If no match: append the new experiment to the array

---

## Step 5: Write to knowledge base

```
Read(path="Claude-Workspace/pricing-intelligence/knowledge-base.json")
# ... append or update experiment ...
Write(path="Claude-Workspace/pricing-intelligence/knowledge-base.json", contents="{updated JSON}")
```

---

## Step 6: Surface in output

After generating an experiment, add one line at the bottom of the calling workflow's output:

```
**Experiment logged:** "{hypothesis title}" — {urgency} urgency, est. {ARR impact range}. Run "show experiment backlog" to see the full queue.
```

Do not create a separate section or elaborate — one line is enough. The experiment is stored; the user can drill in later.

---

## Backlog management

### "Show my experiment backlog"

Read the `experiments` array from the KB. Calculate a priority score for each:

```
priority_score = urgency_weight * arr_weight * freshness_weight
```

Where:
- `urgency_weight`: critical=4, high=3, medium=2, low=1
- `arr_weight`: normalize estimated ARR impact to 1–4 scale (bottom quartile=1, top quartile=4)
- `freshness_weight`: created in last 30 days=4, 31–60 days=3, 61–90 days=2, >90 days=1

Present as a ranked table:

```
## Experiment backlog — {N} experiments

| Rank | Hypothesis | Tier | Urgency | Est. ARR | Age | Score | Status |
|------|-----------|------|---------|----------|-----|-------|--------|
| 1 | {title} | Pro | critical | $1.2M–$1.9M | 5d | 64 | backlog |
| 2 | {title} | Standard | high | $800K–$1.1M | 12d | 48 | backlog |
| ... | | | | | | | |

### Top 3 recommendations

1. **{Experiment title}** — {1-sentence rationale for why this should run next}
   Source: {source_company} {source_signal}
   Reinforced by: {count} additional signals

2. **{Experiment title}** — {rationale}
   Source: ...

3. **{Experiment title}** — {rationale}
   Source: ...
```

### "Review experiment backlog" / "quarterly experiment review"

Full review that validates and re-ranks the entire backlog:

1. **Freshness check:** Flag experiments older than 90 days as "stale — re-validate." For each stale experiment, check if the source company's pricing has changed since the experiment was created (compare against KB entry). If the competitive signal has been reversed (e.g., competitor rolled back the price change), mark the experiment as "obsolete."

2. **Reinforcement check:** Count how many experiments have been reinforced by multiple signals. Experiments with 3+ reinforcing signals are upgraded to the next urgency level (medium → high, high → critical).

3. **Re-rank:** Recalculate all priority scores and present the updated backlog.

4. **Output:**

```
## Quarterly experiment review — {date}

**Total experiments:** {N}
**Active (backlog):** {count}
**Stale (>90 days):** {count}
**Obsolete (signal reversed):** {count}
**Launched:** {count}
**Completed:** {count}

### Stale experiments requiring re-validation
| Experiment | Age | Original signal | Current status |
|-----------|-----|-----------------|----------------|
| {title} | {days}d | {signal} | {still valid / signal reversed / inconclusive} |

### Top 5 to run next (re-ranked)
{Same format as backlog view, but with rationale for each}

### Patterns in the backlog
{If multiple experiments cluster around the same tier or mechanism, surface it: "4 of 7 experiments target the Pro tier — this is the highest-leverage tier for pricing experiments right now."}
```

### "Mark experiment {id} as launched/completed/cancelled"

Update the experiment's `status` field in the KB. Valid statuses:
- `backlog` — not yet started
- `launched` — test is running
- `completed` — test finished (add outcome notes)
- `cancelled` — decided not to run
- `obsolete` — competitive signal no longer relevant

If marking as `completed`, ask: "What was the outcome? (metric result, decision made)"

Store the response in the `notes` field.

---

## Monday.com board sync (user-initiated)

Only runs when the user explicitly asks: "sync experiments to monday", "put my experiment backlog on a board", "create experiment board".

### Board setup

```
create_board(
  boardName="Pricing Experiments",
  boardKind="private",
  boardDescription="Experiment backlog generated from competitive pricing intelligence. Hypotheses, metrics, and estimated ARR impact."
)
```

Columns (create in parallel):

```
create_column(boardId={id}, columnType="long_text", columnTitle="Hypothesis")
create_column(boardId={id}, columnType="text", columnTitle="Source Signal")
create_column(boardId={id}, columnType="text", columnTitle="Metric")
create_column(boardId={id}, columnType="text", columnTitle="Tier")
create_column(boardId={id}, columnType="status", columnTitle="Urgency")
create_column(boardId={id}, columnType="text", columnTitle="Est. ARR Impact")
create_column(boardId={id}, columnType="status", columnTitle="Status")
create_column(boardId={id}, columnType="date", columnTitle="Created")
create_column(boardId={id}, columnType="numbers", columnTitle="Priority Score")
```

### Sync logic

For each experiment in the KB `experiments` array:

1. Search for existing item: `get_board_items_page(boardId={id}, searchTerm="{experiment id}")`
2. If found: update with `change_item_column_values`
3. If not found: `create_item` with all column values

### Urgency status mapping

- critical → "Critical" (red)
- high → "High" (orange)
- medium → "Medium" (yellow)
- low → "Low" (green)

### Status mapping

- backlog → "Backlog"
- launched → "Launched"
- completed → "Completed"
- cancelled → "Cancelled"
- obsolete → "Obsolete"

After sync: `Synced {N} experiments to [{board name}](https://monday.com/boards/{id}).`

---

## Examples

### Example 1: Feature gating experiment from company-research

**Source signal (from Figma research):** "Figma moved SSO from Organization ($75/editor/mo) to Professional ($45/editor/mo) — security features are moving downmarket."

**Generated experiment:**

```json
{
  "id": "exp-2026-03-22-001",
  "created": "2026-03-22",
  "source_company": "figma.",
  "source_signal": "SSO moved from Organization to Professional tier",
  "source_workflow": "company-research",
  "hypothesis": "If we move SSO from Enterprise to Pro, then mid-market self-serve conversion will increase by 5-8% because SSO is a procurement requirement that blocks self-serve adoption at 50+ seat companies",
  "test_design": {
    "mechanism": "Feature flag SSO access for Pro trial accounts in the 50-200 seat range",
    "metric": "Pro trial-to-paid upgrade rate for accounts with 50+ seats",
    "secondary_metrics": ["Enterprise downgrade rate", "Pro ARPU"],
    "tier_affected": "Pro",
    "duration": "4 weeks",
    "expected_impact_range": "+5-8% upgrade rate"
  },
  "competitive_urgency": "medium",
  "estimated_arr_impact": "$1.2M-$1.9M",
  "status": "backlog",
  "last_reviewed": null,
  "priority_score": null,
  "reinforcing_signals": [],
  "notes": ""
}
```

### Example 2: Pricing headroom experiment from monitoring

**Source signal (P1 alert):** "Asana raised Business tier from $24.99 to $27.99/seat — 12% increase with no visible churn signal after 6 weeks."

**Generated experiment:**

```json
{
  "id": "exp-2026-03-22-002",
  "created": "2026-03-22",
  "source_company": "asana.",
  "source_signal": "Asana raised Business tier 12% with no detected churn signal",
  "source_workflow": "monitoring",
  "hypothesis": "If we raise Pro from $19 to $21/seat (annual), then revenue per Pro account will increase ~10% with <2pp churn impact because the nearest competitor just raised prices without market backlash",
  "test_design": {
    "mechanism": "Cohort test — new price for new Pro signups, measure conversion and D30 retention vs. control",
    "metric": "Pro plan revenue per account",
    "secondary_metrics": ["Pro trial-to-paid conversion rate", "D30 retention"],
    "tier_affected": "Pro",
    "duration": "8 weeks",
    "expected_impact_range": "+10% revenue per account, <2pp churn"
  },
  "competitive_urgency": "high",
  "estimated_arr_impact": "$2.1M-$3.4M",
  "status": "backlog",
  "last_reviewed": null,
  "priority_score": null,
  "reinforcing_signals": [],
  "notes": ""
}
```

### Example 3: Deduplication — reinforcing signal

If a later monitoring session detects that ClickUp also raised their Business tier 15%, the engine finds the existing Asana-sourced experiment and reinforces it:

```json
{
  "id": "exp-2026-03-22-002",
  "reinforcing_signals": [
    {
      "date": "2026-04-05",
      "company": "clickup.",
      "signal": "ClickUp raised Business tier 15% — second work-management competitor raising mid-tier in Q1 2026"
    }
  ],
  "competitive_urgency": "critical"
}
```

Output: "This reinforces existing experiment exp-2026-03-22-002: Pro tier price increase test. Now supported by 2 competitive signals (Asana + ClickUp). Urgency upgraded to critical."
