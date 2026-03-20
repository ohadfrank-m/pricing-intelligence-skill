# Company research workflow

Deep-dive on a specific company's pricing strategy: current plans, pricing model, packaging logic, and historical changes.

## Step 1: Resolve the company slug

```
search_companies(query="{company name}")
```

If multiple results return, pick the best match by name similarity and employee count context. If genuinely ambiguous, show the top 3 options and ask the user to confirm.

## Step 2: Pull current pricing

```
get_company_details(slug="{slug}")
```

Extract and structure:
- All plans (name, monthly price, annual price, pricing metric)
- Freemium / free trial availability
- Add-ons and usage-based components
- Employee count (company size proxy)
- logo_url (for any report header)

## Step 3: Check pricing history (free preview)

```
get_company_history(slug="{slug}", discovery_only=true)
```

This returns a list of available diff periods at zero cost. Present the available periods to the user before pulling full history.

**If the user wants full history:** warn that each diff costs 1 credit, then confirm before calling:

```
get_company_history(slug="{slug}")
```

**If the user wants visual screenshots of a specific period:**

Cost: 1 credit per `get_diff_highlight` call. State the total cost (1 credit × number of changes in the period) and confirm before calling.

Call `get_diff_highlight` once per individual change detected in that period — not once per period. Run all calls in parallel. Use targeted query strings derived from the change types listed in the `discovery_only` output.

**Example — a period with 3 changes (price increase, plan renamed, restructure):**

```
# Run in parallel:
get_diff_highlight(slug="{slug}", period="{period}", query="price increase plan pricing")
get_diff_highlight(slug="{slug}", period="{period}", query="plan renamed new name")
get_diff_highlight(slug="{slug}", period="{period}", query="pricing restructure structure")
```

For each result:
- If an image URL is returned: embed it as `![Change screenshot]({image_url})`
- If no image: use the text match as the change description

**After identifying any high-signal period (with or without credits):** run [visual-diff.md](visual-diff.md) to capture before/after screenshots and embed them in the monday doc. High-signal change types: `Price Increased`, `Price Decreased`, `Plan Added`, `Plan Removed`, `Plan Renamed`, `Discount Removed`, `Discount Added`.

---

### Credit-zero fallback (automatic — no user prompt needed)

**Trigger:** `get_status()` returns 0 credits OR the user has not confirmed spending credits.

Do NOT skip history analysis. Instead, run the credit-zero fallback protocol automatically — full instructions in [enrichment.md](enrichment.md) Method 6. It reconstructs the same information using Wayback Machine snapshots, third-party trackers, community posts, and changelogs. No user prompt needed.

## Step 4: Enrich with supplementary sources

After pulling PricingSaaS data, run enrichment in parallel to add depth the MCP alone can't provide. See [enrichment.md](enrichment.md) for full instructions on each method.

### Wayback Machine (always run)

Pull pricing page snapshots to compare how the page looked before vs. after a detected change period. Use the Availability API — the CDX bulk API is blocked (403) for most large SaaS companies:

```
# Find the most recent available snapshot:
WebFetch(url="https://archive.org/wayback/available?url={domain}/pricing")

# Find a snapshot at a specific point in time (e.g., before a known change period):
WebFetch(url="https://archive.org/wayback/available?url={domain}/pricing&timestamp={YYYYMMDD}")
```

From each response, extract the `closest.url` field, then fetch that snapshot directly:

```
WebFetch(url="{closest.url}")
```

Fetch the two closest snapshots around any detected change period and diff the text content. This surfaces copy changes, removed plan names, and price point rewrites that screenshot APIs may miss.

### Product changelog (always run)

Check whether the company publicly documents what changed and when:

```
WebFetch(url="{domain}/changelog")  # or /whats-new, /releases, /blog/category/product
WebSearch(query='"{Company}" changelog pricing plan tier 2025 OR 2026')
```

Look for entries that gate features by plan or reference pricing restructure.

### Earnings call mining (public companies only)

If the company is publicly traded:

```
WebSearch(query='"{Company}" earnings call transcript pricing 2025 OR 2026')
WebSearch(query='"{Company}" "pricing strategy" investor relations 2026')
```

Extract management commentary on pricing rationale and analyst questions about price sensitivity.

### Job postings (run when monitoring is also needed)

```
WebSearch(query='"{Company}" "pricing" OR "monetization" OR "revenue operations" job posting 2026')
```

Flag any open monetization/pricing/RevOps roles as a leading indicator of upcoming changes.

### Strategy context

```
search_pricing_knowledge(query="{company name} pricing")
search_pricing_knowledge(query="{company's category} pricing strategy")
```

Free. Run both in parallel.

## Step 5: Sentiment check (offer to user)

After delivering the main analysis, ask once:

> "Want me to pull public sentiment on {Company}'s pricing — what customers and operators are saying on Reddit, HN, and G2?"

If yes, run [sentiment-research.md](sentiment-research.md) and append results to the output.

## Step 6: Check if missing from PricingSaaS

If `search_companies` returns no results for the target company:

1. Tell the user the company isn't tracked yet
2. Ask: "Want me to submit their pricing page for tracking? I'll need the URL."
3. If the user provides a URL or if you can identify the pricing page:

```
add_page(url="{pricing page URL}")
```

Data will be available in ~15 minutes. Offer to proceed with any publicly visible pricing in the meantime.

---

## Output format

Present findings as a structured analysis. Use this structure:

### [Company name] — pricing analysis

**Exec summary** *(always include at the top — 3 bullets, each one sentence)*
- **What we found:** {The single most important pricing fact — e.g., "Linear raised the entry paid tier 25% in 2024 via a rename-then-raise playbook, now at $10/user/mo."}
- **What it signals:** {Strategic implication — e.g., "Strong retention at paid cohort; they're testing price ceiling before pushing further."}
- **Recommended action:** {One specific thing monday.com should do or test — e.g., "Add Linear to the watchlist and run a pricing page teardown before next QBR."}

**Overview**
One sentence: what the company sells, what segment it targets, and its pricing model.

**Plan structure**

| Plan | Monthly | Annual | Model | Notes |
|------|---------|--------|-------|-------|
| {Plan A} | ${x}/mo | ${x}/mo | per user / flat / usage | |
| {Plan B} | ${x}/mo | ${x}/mo | | |

Include: link to `https://pricingsaas.com/pulse/companies/{slug}`

**Pricing model analysis**
- Value metric: what they charge for and why it makes sense (or doesn't)
- Tier differentiation: what actually changes between tiers (features vs. limits vs. support)
- Packaging logic: how they gate features and what it signals about their target buyer
- Freemium / trial: if applicable, what the conversion mechanism is

**Pricing history**
Summarize available periods from `discovery_only` scan. If full history was pulled, list key changes with dates and direction. If `get_diff_highlight` was called for specific periods, present each change as its own subsection:

```
#### {Period} — {N} changes

##### Change 1: {Change type}
{Text match from get_diff_highlight}
{![Screenshot]({image_url}) if image returned}
[View diff →](https://pricingsaas.com/pulse/companies/{slug}/diffs/{period})

##### Change 2: {Change type}
...
```

If no history available: "PricingSaaS shows no tracked history for this company yet."

**Enrichment findings**
Include a section for each enrichment method that returned meaningful data:

- **Wayback Machine:** Describe the earliest and latest snapshots found, and what changed between them. Link to both snapshot URLs.
- **Product changelog:** List any entries that reference pricing, plans, or packaging. Link to the changelog entry.
- **Earnings call:** Quote any management commentary on pricing rationale. Cite source + quarter.
- **Job postings:** Note any open monetization/pricing/RevOps roles and their implication.

Omit sections where no useful data was found — do not pad with "no signal found" filler.

**Strategic read**
2–3 sentences: what the pricing signals about the company's positioning, growth motion (PLG vs. SLG), and expansion strategy. Flag anything unusual or competitively significant.

**So what for monday.com**
Always include this section. Address each of the following — skip only if genuinely not applicable:

- **Pricing headroom:** Does this competitor's pricing suggest monday.com has room to raise prices, or is it pricing pressure in the other direction? Be specific (e.g., "monday.com Business at $16/seat sits at parity with {Company} — no headroom signal here, but {Company} raised 25% in 12 months, which suggests the market will accept it").
- **Positioning implication:** Does anything in their packaging or tier structure change how monday.com should frame itself in deals against this competitor? (e.g., "Their AI gating at the top tier means monday.com should lead with AI-included-by-default as a differentiator").
- **Experiment or test to consider:** Based on the findings, is there a pricing or packaging experiment monday.com should consider? Be specific — name the hypothesis, the metric to watch, and the tier it affects. (e.g., "Test moving SSO to Basic tier — {Company} offers it at mid-tier, and it may be a blocker for mid-market self-serve conversion").
- **Threat signal:** Is there anything in their pricing trajectory (recent raises, restructuring, AI add-on expansion) that creates risk for monday.com? (e.g., "If {Company} continues adding AI agents to free, monday.com's free tier comparison will face increasing pressure").

**What to do next**
Offer relevant follow-ups based on what was found:
- "Want me to generate a pricing battlecard — monday.com vs. {Company} — with objection handling and negotiation intelligence for sales?"
- "Want me to map the competitive landscape in {category} to see how {Company} compares against the full market?"
- "Want to add {Company} to your pricing watchlist to track future changes automatically?"
- "Want me to pull public sentiment on how customers are reacting to their pricing?"
- "Want the full pricing change history? That's {N} diffs at 1 credit each."
- {If company has a free tier and history shows Capacity/Feature changes:} "Want a full freemium change analysis — what their free tier looked like before vs. now and what the move signals?"
- {If researching a competitor:} "Want negotiation intelligence — what buyers actually pay, typical discount range, and how to handle pricing objections if they come up in a deal?"
- "Want an exec summary — 3 bullets ready to share with your manager or paste into a Slack message?"

## Post-research: offer battlecard

After delivering the main output, ask once:

> "Want me to generate a pricing battlecard — monday.com vs. {Company} — with plan comparison, objection handling, and negotiation intelligence? It goes directly to sales or product."

If yes, run [battlecard-generator.md](battlecard-generator.md). Pre-populate Steps 2 and 2b with the data already gathered in this research session — do not re-fetch what you already have.

---

## Final step: log to monday

After delivering the output above, follow [monday-logging.md](monday-logging.md) to log this result.

- Item name: `{Company} — Research`
- Change Type: `Company Research`
- Summary: 1–2 sentence distillation of the key pricing insight from this analysis
- PricingSaaS link: `https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}` if a specific period was pulled; `https://pricingsaas.com/pulse/companies/{slug}` if no period was used
- Workflow: `company-research`
