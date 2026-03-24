# Monitoring workflow

Track specific companies' pricing changes over time: set up watchlists, check for recent changes, and pull detailed before/after diffs.

## Use cases

- "Add these competitors to my pricing watchlist"
- "What pricing changes happened this week?"
- "Has [Company] changed their pricing recently?"
- "Show me exactly what changed in [Company]'s pricing last quarter"

---

## Setting up a watchlist

### Step 1: Resolve slugs for named companies

For each company the user wants to track:

```
search_companies(query="{company name}")
```

Run in parallel for all companies. Extract the slug from each result. If a company isn't found in PricingSaaS, offer to submit their pricing page (see below).

### Step 2: Add to watchlist

```
add_to_watchlist(slugs=["{slug1}", "{slug2}", "{slug3}", ...])
```

Free. Confirm which companies were successfully added. For any that failed (not found), note them separately.

### Step 3: Handle missing companies

For companies not found in PricingSaaS:

```
WebSearch(query="{Company Name} pricing page URL")
add_page(url="{pricing page URL}")
```

Tell the user: "I've submitted {Company}'s pricing page for tracking. Data will be available in ~15 minutes and I can add them to the watchlist after that."

---

## Checking for pricing changes

### Quick check — what changed recently

```
get_pricing_news()
```

Free. Returns recent pricing changes across all tracked companies. Filter to the user's companies of interest if a specific set was named.

Present as a change log:

```
**Pricing changes — [date range]**

{Company A} — {what changed, e.g., "raised Pro plan from $49 to $59/user/mo"}  
→ pricingsaas.com/pulse/companies/{slug}

{Company B} — {what changed}
→ pricingsaas.com/pulse/companies/{slug}

No changes detected for: {Company C}, {Company D}
```

### Detailed change data — weekly or monthly

**Warn the user before calling:** this costs 1 credit per company (company scope) or 2 credits (global scope).

For watchlist companies, call `fetch_diffs` once per company using `scope="company"` — run all calls in parallel:

```
fetch_diffs(scope="company", slug="{slug_1}", period="latest", period_type="weeks")
fetch_diffs(scope="company", slug="{slug_2}", period="latest", period_type="weeks")
# ... one call per watchlist company with changes
```

Or for a specific period:

```
fetch_diffs(scope="company", slug="{slug}", period="{YYYY-MM}", period_type="months")
```

Or for global market changes (not just watchlist) — costs 2 credits:

```
fetch_diffs(scope="global", period="latest", period_type="weeks")
```

### Visual before/after screenshots — one call per change

**Warn the user before calling:** this costs 1 credit per `get_diff_highlight` call.

Do NOT make one broad call per period. Instead, call `get_diff_highlight` once per individual change detected in that period, running all calls in parallel. This maximises the chance of getting screenshot images back for each change event.

**How to determine the queries:**

From the `get_company_history` output, the period entry lists each change type (e.g., "Feature Changed: 1, Capacity Increased: 1, Addon Added: 1"). Map each change to a targeted query string describing that specific change in plain language.

**Example — Notion 2026W10 (3 changes):**

```
# Run all three in parallel:
get_diff_highlight(slug="notion.", period="2026W10", query="guest seats external guest limit unlimited")
get_diff_highlight(slug="notion.", period="2026W10", query="Notion Agent AI agent rename")
get_diff_highlight(slug="notion.", period="2026W10", query="addon added new add-on")
```

**After collecting results:**

- If an image URL is returned in any result: embed it in the output using markdown `![Change screenshot]({image_url})`
- If no image is returned but text match is present: use the text match as the change description
- If both are missing: fall back to the browser MCP (see below)

**Browser MCP fallback (when no images returned):**

If `get_diff_highlight` returns "No highlight images were returned" for all calls in a period, use the browser MCP to capture the diff page directly:

```
browser_navigate(url="https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}")
browser_take_screenshot()
```

This gets a full-page screenshot of the PricingSaaS diff page showing all changes visually. Include the screenshot in the output and in any monday doc created.

---

## Checking history for a specific company

### Free preview — available periods

```
get_company_history(slug="{slug}", discovery_only=true)
```

Returns a list of available diff periods at zero cost. Present these to the user before pulling full history.

Example output:
```
**{Company} — available pricing history**
- 2024-Q4: changes detected
- 2024-Q3: changes detected
- 2024-Q2: no changes
- 2024-Q1: changes detected

Full history costs 1 credit per diff ({N} diffs = {N} credits). Want me to pull all of them?
```

### Full history

After user confirms:

```
get_company_history(slug="{slug}")
```

Present changes chronologically, most recent first:

```
**{Company} — pricing history**

**{Date}** — {what changed}
e.g., "Pro plan: $49 → $59/user/mo (+20%). Business plan renamed to Enterprise."

**{Date}** — {what changed}
e.g., "Added Starter plan at $19/user/mo. Free plan removed."

[Link to full profile: pricingsaas.com/pulse/companies/{slug}]
```

---

## Enrichment: go deeper on detected changes

After identifying pricing changes for a period, automatically run the following enrichment in parallel — no user prompt needed. See [enrichment.md](enrichment.md) for full instructions.

### Wayback Machine — confirm the visual change

For each company with detected changes, fetch the live pricing page and a historical Wayback snapshot to cross-reference what changed:

```
# Live page (always works):
WebFetch(url="https://{domain}/pricing")

# Wayback availability check (lightweight — less likely to be blocked than CDX):
WebFetch(url="https://archive.org/wayback/available?url={domain}/pricing")

# If a recent snapshot timestamp is returned, fetch it:
WebFetch(url="https://web.archive.org/web/{snapshot-timestamp}/{domain}/pricing")
```

Diff the two versions to surface price point wording, plan name changes, and copy rewrites. If the availability API is also blocked, rely on the live page fetch + PricingSaaS diff text alone.

**Note:** The Wayback CDX bulk API (used for frequency analysis) is blocked for most large SaaS companies. Do not use it here — use the lightweight availability API or direct snapshot fetch instead.

### Product changelog — was it announced?

```
WebSearch(query='"{Company}" changelog OR release notes pricing plan 2025 OR 2026')
```

If the company has a public changelog, check whether the pricing change was announced and what language they used to frame it.

### Job postings — leading indicators

```
WebSearch(query='"{Company}" pricing OR monetization OR "revenue operations" job 2026 site:linkedin.com OR site:greenhouse.io')
```

Any open monetization or RevOps roles signal another change may be incoming within the next 1–3 quarters.

### Cross-company pattern synthesis

After analyzing changes across multiple companies in a session:

1. Call `search_pricing_knowledge(query="{change type} pricing trend")` for any common change pattern detected
2. Check whether other companies in the watchlist made the same type of move in the same period (e.g., multiple companies moving to usage-based in the same quarter is a market signal)
3. Surface this in the output as "Market pattern: {description}" when it's present

### Freemium / trial changes — auto-detect

While processing each diff period, check whether any change types match the freemium/trial pattern: `Capacity Decreased`, `Capacity Increased`, `Feature Removed`, `Feature Added`, `Plan Removed`, `Plan Added`, `Pricing Metric Changed`. If detected on a plan that appears to be a free, trial, or starter tier, automatically run [freemium-trial-tracker.md](freemium-trial-tracker.md) Step 4 (classification) and include the result in the output.

This requires no extra tool calls if you already have the diff data — just classify the change type against the freemium pattern table.

### Sentiment (offer after delivering changes)

After the main change summary, ask once:

> "Want me to check what customers and operators are saying publicly about any of these changes — Reddit, HN, G2?"

If yes, run [sentiment-research.md](sentiment-research.md) for each company with significant changes.

---

## Final step: log to monday

After delivering the monitoring output, follow [monday-logging.md](monday-logging.md) to log results. Create one item per company that had changes (not one item for the whole session). Run `create_item` calls in parallel.

- Item name: `{Company} — {Period}` (e.g., "Clay — 2026W12")
- Change Type: map the most significant change type (see status label table in monday-logging.md)
- Summary: what specifically changed — old value → new value where known
- PricingSaaS link: `https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}` — use the exact period string from the diff call (e.g., `2026W12`, `2025Q1`). Strip the trailing dot from the slug (`clay` not `clay.`)
- Workflow: `monitoring`

Companies with no changes in the period: do not log them.

---

## Output standards

- Always state the credit cost before any paid tool call and wait for confirmation
- Lead every change summary with company name + what changed (not background context)
- Price changes: always show old price → new price + % delta (e.g., "$49 → $59/user/mo, +20%")
- Structural changes (new plan, removed tier, renamed plan): describe the before and after state clearly
- If no changes detected: say so explicitly — "No pricing changes detected for these companies in the selected period"

### Per-change presentation format

When `get_diff_highlight` results are collected, present each change as its own numbered section. Mirror the PricingSaaS diff page structure exactly — this is also the format used verbatim in the monday doc.

**Opening:** synthesise a single title sentence naming the two or three most significant changes (no extra credit spend). Example: "Notion launches Custom Agents add-on, opens guest seats to unlimited".

```
### {Company} — {Period}
#### {Synthesised title}

**Source:** [View on PricingSaaS](https://pricingsaas.com/pulse/companies/{slug}/diffs/{period})

---

**1. {Change type} | {Category}**
{Full text match from get_diff_highlight, including bracketed detail}
{![Change screenshot]({image_url}) if image returned by get_diff_highlight}
{If Cloudinary snapshots available (from visual-diff.md probe):}
  Before ({YYYYMMDD_before}): ![...]({cloudinary_before})
  After ({YYYYMMDD_after}): ![...]({cloudinary_after})
  [Open side-by-side comparison →]({compare-viewer URL})
{If no images at all:}
  [View before/after on PricingSaaS →]({diff URL}) *(1 credit when available)*

---

**2. {Change type} | {Category}**
{Text match}
{Images — same pattern}

---

**{N}. {Change type} | {Category}**
{Text match}
{Images}
```

**Category** is always one of: `Pricing`, `Packaging`, or `Product` — taken from the PricingSaaS event data.

**If credits = 0:** use the change type labels from `discovery_only` as section headings, show Cloudinary full-page snapshots where available, and show the diff link placeholder for all image sections.
