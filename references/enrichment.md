# Enrichment methods

Supplementary research techniques that add context beyond PricingSaaS data. Call these from any workflow to deepen the analysis. All methods use `WebSearch` or `WebFetch` — zero credit cost.

**WebFetch failure handling:** If any `WebFetch` call fails (timeout, domain blocked, 4xx/5xx), skip that source silently and continue with the remaining sources. Never block or stall a workflow on a single failed fetch. Document which sources returned useful data in the output.

---

## 1. Wayback Machine — pricing page history

Use when: you need to see how a company's pricing page looked at a specific point in time, or want to reconstruct changes that predate PricingSaaS tracking.

**Important:** The Wayback CDX bulk API (`/cdx/search/cdx`) is blocked (403) for most large SaaS companies. Use the lightweight Availability API to find snapshot timestamps, then fetch individual snapshots directly.

### Find available snapshots (use Availability API, not CDX)

```
WebFetch(url="https://archive.org/wayback/available?url={domain}/pricing")
```

This returns the most recent successful snapshot timestamp. To find earlier snapshots, try date-specific lookups:

```
# Check what was archived around a specific date:
WebFetch(url="https://archive.org/wayback/available?url={domain}/pricing&timestamp={YYYYMMDD}")
```

Try several dates (e.g., 6 months ago, 12 months ago, 18 months ago) to build a timeline of available snapshots.

### Fetch a specific snapshot

```
WebFetch(url="https://web.archive.org/web/{timestamp}/{domain}/pricing")
```

Example: `https://web.archive.org/web/20241001120000/clay.com/pricing` for Clay's pricing page as it appeared October 1 2024.

### Compare two snapshots

Fetch both timestamps, then diff the text content to identify:
- Plan names that appeared or disappeared
- Price values that changed
- Feature descriptions that were rewritten
- CTAs or trial offers that changed

### When Wayback is unavailable

If both the CDX API and availability API are blocked, compare the live pricing page (fetched fresh) against PricingSaaS diff text to reconstruct what changed, without needing archive images.

### Output format

```
## {Company} — pricing page history ({date range})

### Snapshot: {Date A}
{Key pricing elements: plans, prices, notable features/CTAs}

### Snapshot: {Date B}
{Key pricing elements}

### What changed
- {Specific element}: {before} → {after}
- {Specific element}: {before} → {after}

[View snapshot A on Wayback Machine]({url_A})
[View snapshot B on Wayback Machine]({url_B})
```

### When Wayback Machine has no data

Some companies block Wayback Machine crawling. If `cdx` returns empty or no `statuscode:200` results, note: "Wayback Machine has no archived snapshots for this pricing page" and fall back to PricingSaaS history or Google Cache:

```
WebSearch(query='cache:{domain}/pricing')
WebSearch(query='"{Company}" pricing "{plan name}" site:web.archive.org')
```

---

## 2. Job postings as leading indicator

Use when: monitoring a company for signals of upcoming pricing changes, or trying to understand their monetization direction.

### What to look for

Companies hire for pricing-related roles 3–6 months before major pricing changes. Key signals:

| Job title | What it signals |
|-----------|----------------|
| Pricing Analyst / Manager | Pricing review underway |
| Head of Monetization | Revenue model rethink |
| Revenue Operations | GTM motion shift, often tied to pricing |
| Growth PM / Monetization PM | PLG pricing experiment incoming |
| Enterprise Sales | Moving upmarket → pricing tier expansion |

### Search queries

```
WebSearch(query='"{Company}" "pricing" OR "monetization" job posting 2025 OR 2026')
WebSearch(query='site:linkedin.com/jobs "{Company}" pricing OR monetization OR "revenue operations"')
WebSearch(query='site:greenhouse.io OR site:lever.co OR site:ashbyhq.com "{Company}" pricing')
```

### Output

Note the role, posting date, and what it signals about timing and direction. A cluster of monetization/RevOps hires in Q4 2025 at a company that then restructured pricing in Q1 2026 is a pattern worth documenting.

---

## 3. Product changelog mining

Use when: tracking packaging changes that haven't yet appeared on the pricing page. Changelogs often announce new features being added to specific tiers before the pricing page is updated.

### How to find changelogs

```
WebSearch(query='"{Company}" changelog OR "what\'s new" OR "release notes" 2026')
WebFetch(url="{company changelog URL}")
```

Common changelog URLs:
- `{domain}/changelog`
- `{domain}/whats-new`
- `{domain}/releases`
- `{domain}/blog/category/product`
- `changelog.{domain}`

For companies with public changelogs (Notion, Figma, Linear, ClickUp, HubSpot), fetch the page directly and scan for:
- Feature gating language ("now available on Business and above")
- New add-ons or limits
- Trial or freemium changes
- Deprecations that might indicate packaging consolidation

### Output

List changelog entries relevant to pricing/packaging with dates and a note on whether the pricing page has been updated to reflect it yet (if not, that's a gap worth flagging).

---

## 4. Earnings call and investor report mining

Use when: researching a public company's pricing strategy rationale or anticipating future changes.

Public companies (HubSpot, Salesforce, Docusign, etc.) disclose pricing strategy in earnings calls and annual reports. This is the highest-quality signal for *why* changes happened and *what's coming*.

### Find transcripts

```
WebSearch(query='"{Company}" earnings call transcript 2025 OR 2026 pricing')
WebSearch(query='"{Company}" Q4 2025 OR Q1 2026 earnings "pricing" "ARPU" OR "revenue per seat" OR "monetization"')
WebFetch(url="https://seekingalpha.com/symbol/{TICKER}/earnings/transcripts")
```

Free transcript sources: Motley Fool, Seeking Alpha (limited free), The Motley Fool, company IR pages.

### What to extract

- Management commentary on pricing rationale ("we're seeing strong demand at the new price point...")
- Analyst pushback questions about pricing pressure or churn
- ARPU or NRR metrics disclosed alongside pricing changes
- Forward-looking pricing signals ("we expect to complete the migration to new pricing by Q2...")

### Output

```
## {Company} — earnings pricing mentions ({quarter})

**Source:** [{Earnings call title}]({url})

**Key quotes:**
> "{exact quote from management}" — {speaker title}, {date}

**Strategic signal:** {1-2 sentence interpretation of what this means for pricing direction}
```

---

## 5. Cross-company pattern synthesis

Use when: a company makes a pricing move and you want to understand whether it's an isolated decision or part of a broader market pattern.

### Approach

After identifying a specific change type (e.g., "Clay switched to dual-metric pricing"), search for other companies that made the same type of move:

```
WebSearch(query='SaaS "dual metric pricing" OR "usage + seat pricing" 2024 OR 2025 OR 2026')
WebSearch(query='B2B SaaS pricing restructure "actions and credits" OR "seats and usage" 2025')
search_pricing_knowledge(query="{change type} pricing strategy examples")
search_pricing_knowledge(query="{company category} pricing model trends")
```

Then cross-reference with `search_companies_advanced` in PricingSaaS to find companies with similar pricing attributes:

```
search_companies_advanced(has_license=true, price_min={range_floor}, price_max={range_ceiling})
```

### Output

```
## Market pattern: {Change type}

**Companies that made a similar move:**
| Company | When | What they did | Outcome (if known) |
|---------|------|---------------|-------------------|
| {Company A} | {Date} | {Description} | {Revenue impact, churn signal, etc.} |

**Pattern assessment:** {Is this an isolated move or a market-wide trend? What does the prevalence tell you about where the category is heading?}

**Implication for {original company}:** {What can be inferred about likely success or risk of their move given what others experienced?}
```

---

---

## 6. Credit-zero fallback — reconstructing pricing history without paid diffs

Use when: `get_status()` returns 0 credits and paid PricingSaaS tools (`get_company_history`, `get_diff_highlight`, `fetch_diffs`) cannot be called. This method reconstructs the same information from zero-cost sources. Run automatically — no user prompt needed.

### When to activate

Triggers whenever `get_status()` shows 0 remaining credits AND `discovery_only` has identified periods with high-signal change types: `Price Increased`, `Price Decreased`, `Plan Added`, `Plan Removed`, `Plan Renamed`, `Discount Removed`, `Discount Added`.

Skip periods that contain only `Feature Added` or `Feature Changed` — those rarely need diff reconstruction.

### Period-to-date mapping

Convert PricingSaaS period strings to snapshot target dates:

| Period format | Before snapshot date | After snapshot date |
|---------------|---------------------|---------------------|
| `{YYYY}Q1` | `{YYYY-1}1231` | `{YYYY}0401` |
| `{YYYY}Q2` | `{YYYY}0331` | `{YYYY}0701` |
| `{YYYY}Q3` | `{YYYY}0630` | `{YYYY}1001` |
| `{YYYY}Q4` | `{YYYY}0930` | `{YYYY+1}0101` |
| `{YYYY}M{MM}` | first day of prior month | first day of next month |
| `{YYYY}W{WW}` | 3 days before week start | 3 days after week end |

### Source waterfall (run all in parallel per high-signal period)

#### Source 1: Wayback Machine snapshots (highest fidelity)

```
# Find available snapshots bracketing the change:
WebFetch(url="https://archive.org/wayback/available?url={domain}/pricing&timestamp={before_YYYYMMDD}")
WebFetch(url="https://archive.org/wayback/available?url={domain}/pricing&timestamp={after_YYYYMMDD}")

# Fetch the actual pages using closest.url from each response:
WebFetch(url="{closest.url_before}")
WebFetch(url="{closest.url_after}")
```

Diff the fetched pages — scan for plan names, `$` price values, feature bullets per tier, and CTA text changes.

**Note:** The CDX bulk API (`/cdx/search/cdx`) is blocked for most large SaaS sites — always use the Availability API.

#### Source 2: Third-party price trackers

```
WebFetch(url="https://www.saaspricepulse.com/tools/{company-name}")
WebFetch(url="https://saascompare.io/pricing/{company-name}/")
WebFetch(url="https://getpulsesignal.com/pricing/{company-name}")
WebSearch(query='"{Company}" pricing history site:saascompare.io OR site:saaspricepulse.com OR site:getpulsesignal.com')
```

These sites often cache old plan names and prices alongside current ones — especially useful after a rename-and-raise.

#### Source 3: Community disclosure (customers post price increase emails/screenshots)

```
WebSearch(query='"{Company}" price increase {year} site:reddit.com OR site:news.ycombinator.com')
WebSearch(query='"{Company}" pricing change {year} "per user" OR "per month" OR "per seat"')
WebSearch(query='"{Company}" new pricing {month_range} {year} announcement OR email OR notification')
```

Reddit and HN threads frequently contain exact before/after prices quoted by customers who received the increase notification.

#### Source 4: Official announcement (changelog / blog)

```
WebFetch(url="{domain}/changelog")
WebFetch(url="{domain}/blog")
WebSearch(query='"{Company}" pricing {year} changelog OR "pricing update" OR "new plans" OR "pricing change"')
```

#### Source 5: Google Cache (recent changes only — within ~30 days)

```
WebSearch(query='cache:{domain}/pricing')
```

#### Source 6: Vendr marketplace

```
WebFetch(url="https://www.vendr.com/marketplace/{company-name}")
```

Vendr sometimes documents historical pricing in their negotiation sections.

### Output format

Label reconstructed diffs clearly so they are not confused with PricingSaaS-sourced data:

```markdown
#### {Period} — reconstructed diff (credit-free)

*PricingSaaS flagged {N} change(s): {change types}. Reconstructed from: {sources used}.*

| Element | Before | After | Source |
|---------|--------|-------|--------|
| {Plan name / price / feature} | {old value} | {new value} | Wayback / SaasPricePulse / Reddit / etc. |

[View before snapshot]({wayback_url_before})   [View after snapshot]({wayback_url_after})
[Unlock full diff on PricingSaaS](https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}) *(requires {N} credit — resets {reset_date})*
```

If no source returned useful data, always note explicitly:
> "Reconstruction attempted for {period} ({change types}) — Wayback Machine had no snapshots, no community posts found, third-party trackers showed current pricing only. Unlock with {N} credit when available."

Never silently omit a period. Always document what was attempted.

---

## When to call these methods

| Trigger | Method |
|---------|--------|
| "How did their pricing page look before?" | Wayback Machine (Method 1) |
| "Are they planning a pricing change?" | Job postings (Method 2) |
| "Did they add anything to a plan recently?" | Changelog mining (Method 3) |
| "Why did they raise prices?" (public co) | Earnings call (Method 4) |
| "Is this a trend or a one-off?" | Cross-company pattern synthesis (Method 5) |
| Credits = 0 and history has price-related changes | Credit-zero fallback (Method 6) — run automatically, no prompt |
| After any significant change detected via monitoring | All of the above, in parallel |
