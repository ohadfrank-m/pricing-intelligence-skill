# Proactive monitoring workflow

Daily automated monitoring that detects pricing changes, scores them by competitive severity, and delivers alerts via Slack. Replaces the pull-based weekly digest model with push-based intelligence that surfaces what matters when it matters.

## When to run

- User asks: "run daily monitor", "check for critical pricing changes", "set up pricing alerts", "run proactive monitoring"
- Scheduled via Slack reminder, calendar event, monday.com recurring item, or Claude scheduled task
- Can run alongside or instead of the weekly digest — the digest still provides the synthesis layer, but P0/P1 changes no longer wait for it

---

## Step 1: Load context

### 1a: Read the knowledge base

```
Read(path="Claude-Workspace/pricing-intelligence/knowledge-base.json")
```

Extract:
- `config.monday_pricing` — needed for severity scoring
- `config.slack` — channel and alert mode preferences
- `companies` — for historical context on changes
- `patterns` — to detect pattern reinforcement

If the KB file does not exist, create it with the empty schema from [knowledge-base.md](knowledge-base.md).

### 1b: Check monday.com pricing reference

If `config.monday_pricing.plans` is empty or `config.monday_pricing.last_updated` is older than 90 days, run the monday.com pricing setup from [severity-scoring.md](severity-scoring.md) before proceeding.

### 1c: Check Slack configuration

If `config.slack.channel_id` is null, run the Slack setup flow (Step 5 below) before proceeding with the first monitoring run.

---

## Step 2: Pull changes

Run all in parallel:

```
get_pricing_news()
get_watchlist()
```

From `get_pricing_news()`, extract all changes detected since the last monitoring run. If this is the first run, use the last 7 days as the window.

Cross-reference against the watchlist to separate:
- **Watchlist changes** — companies the user is actively tracking
- **Market changes** — other companies with detected changes (lower priority, but P0-worthy changes still surface)

---

## Step 3: Score every change

For each detected change, run the severity scoring framework from [severity-scoring.md](severity-scoring.md).

**For watchlist companies:** Score all 4 dimensions using the full framework. If the company has a KB entry, use cached `severity_baseline.segment_overlap` for Dimension 2.

**For non-watchlist companies:** Quick-score using Dimensions 2 (segment overlap) and 3 (change magnitude) only. If the quick-score is >= 4 (suggesting possible P0/P1), run the full 4-dimension scoring.

### Enrich P0/P1 changes

For any change scoring P0 or P1, immediately pull additional context:

```
get_company_details(slug="{slug}")
```

Compare the fresh pricing data against the KB entry (if one exists) to identify exactly what changed. Also compare against `config.monday_pricing` to calculate the precise price delta relative to monday.com.

---

## Step 4: Classify and route

After scoring all changes, sort into delivery buckets:

| Severity | Action |
|----------|--------|
| P0 (10–12) | Deliver Slack alert immediately (Step 6) |
| P1 (7–9) | Deliver Slack alert in daily batch (Step 6) |
| P2 (4–6) | Queue for weekly digest — write to KB only |
| P3 (0–3) | Log to KB silently |

---

## Step 5: Slack configuration (first run only)

On the first run, or if `config.slack` is unconfigured, ask the user:

> "I'll send pricing alerts via Slack. A few setup questions:"
>
> 1. **Where should alerts go?** Share a channel name (e.g., #pricing-intel) or say "DM me"
> 2. **Alert mode:**
>    - **Auto-send** — I send alerts directly (fastest)
>    - **Draft** — I create drafts for you to review before sending
>    - **Hybrid** — Auto-send P0 (critical), draft P1 (significant)

Use `slack_search_channels(query="{channel name}")` to resolve the channel ID. For DMs, the user's Slack user ID is used as the channel_id.

Store the configuration:

```json
{
  "channel_id": "C12345",
  "alert_mode": "hybrid",
  "last_alert": null
}
```

Write to `config.slack` in the KB.

---

## Step 6: Deliver alerts

### P0 alert format

```
*P0 ALERT: {Company} — {1-line change summary}*

{Company} {specific change description with old → new values}.

*monday.com impact:*
• Price gap: {monday.com tier} at ${monday_price} vs. {Company} {tier} now at ${new_price} ({delta})
• {1 sentence on positioning / deal win rate implication}

*Recommended action:* {1 specific action}

<{pricingsaas_url}|View on PricingSaaS> | <{monday_doc_url}|Full research>
```

### P1 alert format

```
*P1: {Company} — {1-line change summary}*

{2-sentence description of the change and its competitive significance.}

*Severity: {score}/12* — {top scoring dimension rationale}

<{pricingsaas_url}|View on PricingSaaS>
```

### Daily batch format (when multiple P1 changes)

```
*Daily pricing monitor — {date}*
{N} changes scored, {M} requiring attention.

*P0 (critical):*
• {Company}: {change} — *{score}/12*

*P1 (significant):*
• {Company}: {change} — *{score}/12*
• {Company}: {change} — *{score}/12*

*P2 (queued for weekly digest):*
{count} lower-priority changes logged.

Run "show weekly digest" for the full picture.
```

### Delivery mechanics

Based on `config.slack.alert_mode`:

**Auto-send:**
```
slack_send_message(channel_id="{config.slack.channel_id}", message="{alert text}")
```

**Draft:**
```
slack_send_message_draft(channel_id="{config.slack.channel_id}", message="{alert text}")
```

After creating a draft, tell the user: "Slack draft created for {N} pricing alerts — review and send from your Drafts in Slack."

**Hybrid:**
- P0 → `slack_send_message` (auto-send)
- P1 → `slack_send_message_draft` (draft)

---

## Step 7: Log to knowledge base

After delivering alerts, update the KB:

1. For every company with a detected change (any severity), upsert the company entry per [knowledge-base.md](knowledge-base.md)
2. Update `config.slack.last_alert` with the current timestamp
3. Run cross-company pattern detection
4. If the hypothesis engine is configured, run [hypothesis-engine.md](hypothesis-engine.md) for any P0/P1 change that has a "So what for monday.com" implication

---

## Step 8: Log to monday

Follow [monday-logging.md](monday-logging.md) for any P0 or P1 change. P2/P3 changes are logged to the KB only (not to the monday board) to avoid noise.

---

## Step 9: Output summary

After all alerts are delivered and logging is complete, output a brief summary to the chat:

```
## Daily pricing monitor — {date}

**Changes detected:** {total count}
**Alerts delivered:** {P0 count} P0 · {P1 count} P1

{If P0 or P1 exist:}
### Critical changes
{List each P0/P1 with company, change, score, and delivery status}

### Queued for weekly digest
{P2 count} changes logged — run "show weekly digest" for details.

### No changes detected for
{List watchlist companies with no activity}

---

Knowledge base updated. {pattern_count} new patterns detected.
```

If zero changes are detected across all sources, say: "No pricing changes detected today across your watchlist. Market quiet."

---

## Scheduled execution setup

After the first successful run, offer to set up automatic scheduling:

> "Want this to run automatically every morning? Options:"
>
> 1. **Slack reminder:** `/remind me to "Run daily pricing monitor" every weekday at 8am`
> 2. **Claude scheduled task:** Fully automatic, no manual trigger
> 3. **monday.com recurring item:** Creates a daily recurring task on your Pricing Intelligence board

### Claude scheduled task (if available)

```
create_scheduled_task(
  name="Daily pricing monitor",
  schedule="every weekday at 8:00am",
  prompt="Run daily pricing monitor"
)
```

### Recommended cadence

- **Daily** (weekdays) for teams actively in competitive deals or during known competitor pricing review cycles
- **Monday + Thursday** for steady-state monitoring (catches most changes without noise)
- **Weekly** (Monday only) if the weekly digest already covers the need — use proactive monitoring only for P0 escalation

---

## Edge cases

**No watchlist set up:** Ask: "Your watchlist is empty. Which companies should I monitor? Or run 'set up category watchlist' to add a full competitive set."

**Slack MCP not connected:** Skip Slack delivery. Output alerts in chat only. Tell the user: "Slack integration not connected — alerts delivered in chat. Connect the Slack MCP to enable push alerts."

**Credits at 0:** The monitoring workflow itself (`get_pricing_news`, `get_watchlist`) is free. Severity scoring uses only KB data and `config.monday_pricing` — no credits needed. Only `get_company_details` for P0/P1 enrichment is an API call, and it's also free. The daily monitor runs at zero credit cost.

**Multiple P0 alerts in one run:** Deliver each P0 as a separate Slack message (not batched). P0 means "read this now" — batching dilutes urgency.

**Duplicate alerts:** Before delivering, check `config.slack.last_alert` and the KB's `pricing_history_summary` for the company. If this exact change (same period, same change type) was already alerted, skip it. Do not re-alert on the same change across runs.
