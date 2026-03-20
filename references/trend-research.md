# Trend research workflow

Map the pricing landscape of a SaaS category or industry: who's in the space, how they price, what models dominate, and where the market is moving.

## Step 1: Discover companies in the space

**If given a category or industry**, run 2–3 keyword variations in parallel to maximize coverage (goal: 10–30 companies):

```
search_companies(query="{primary category term}")
search_companies(query="{alternate phrasing}")
search_companies(query="{industry + 'software' or 'platform'}")
```

Then use attribute filters to catch what text search missed:

```
search_companies_advanced(
  has_license=true,
  price_min={estimated floor},
  price_max={estimated ceiling}
)
```

**If given a seed company** ("who competes with X"):

```
get_company_details(slug="{seed slug}")  # extract category, price range, attributes
```

Then find peers using extracted attributes:

```
search_companies(query="{seed's product description keywords}")
search_companies_advanced(
  has_freemium={match seed},
  price_min={seed price * 0.3},
  price_max={seed price * 5}
)
```

Deduplicate all results across searches.

## Step 2: Pull pricing details

Fetch details for all discovered companies in batches of 5–6 (run in parallel):

```
get_company_details(slug="{slug}")  # repeat for each company
```

From each response, extract:
- Plan names and prices (monthly + annual)
- Pricing metric (per user, per seat, flat, usage-based)
- Freemium / trial availability
- Add-ons
- Employee count
- logo_url

## Step 3: Pull recent market moves

```
get_pricing_news()
```

Filter results to companies in this category. Note any that changed pricing recently — this is signal for where the market is heading.

## Step 4: Pull strategy frameworks (optional but recommended)

```
search_pricing_knowledge(query="{category} pricing strategy")
search_pricing_knowledge(query="{category} packaging best practices")
```

Run in parallel. Use findings to contextualize patterns in step 5.

## Step 5: Enrich with supplementary sources

Run these in parallel with Step 4 to add depth. See [enrichment.md](enrichment.md) for full instructions.

### Wayback Machine — detect legacy pricing patterns

For the 3–5 most relevant companies in the landscape, pull Wayback Machine snapshots to see how their pricing page has evolved over the past 12–24 months:

```
WebFetch(url="https://web.archive.org/cdx/search/cdx?url={domain}/pricing&output=json&limit=10&fl=timestamp,statuscode&filter=statuscode:200&collapse=timestamp:6")
```

Use the earliest and latest snapshots to characterize the trajectory (e.g., "moved from flat to per-seat", "added enterprise tier", "removed freemium").

### Job postings as leading indicator

For any company that appears to be in transition (recent price changes, new plan structures), check for open monetization/pricing roles:

```
WebSearch(query='"{Company}" pricing OR monetization OR "revenue operations" job 2026')
```

Cluster findings: companies actively hiring for pricing roles are likely to restructure soon — note this as a forward-looking signal in the report.

### Cross-company pattern synthesis

After collecting data, use `search_pricing_knowledge` to benchmark detected patterns against established frameworks:

```
search_pricing_knowledge(query="{dominant model in this category} pricing")
search_pricing_knowledge(query="{unusual pattern observed} SaaS pricing")
```

Then look for market-wide signals in PricingSaaS:

```
get_pricing_news()  # already called in Step 3 — filter to companies in this category
```

Flag any cluster of companies making the same type of move in the same quarter as a market-level signal (e.g., "5 of 12 companies added usage-based components since Q3 2025").

## Step 6: Fill coverage gaps

After pulling details, identify well-known competitors not found in PricingSaaS. For each:

```
# Search for their pricing page
WebSearch(query="{Company Name} pricing page URL")

# If a public pricing page exists, submit it
add_page(url="{pricing page URL}")
```

Run web searches in parallel (batches of 4–6). Note submitted companies in the report — their data will be available in ~15 minutes. Skip companies with no public pricing page (enterprise "contact us" only).

## Step 7: Structure the landscape

Group companies by market layer — use price bands, feature depth, and employee count as signals:

**Tier examples:**
- SMB / self-serve: typically lower price points, self-service signup, per-seat or flat
- Mid-market: broader feature sets, higher per-seat prices, often hybrid PLG+SLG
- Enterprise: custom pricing or high-end tiers, SSO/security features, annual contracts

**Identify pricing model patterns:**
- What model dominates? (per user, flat, usage-based)
- Who is an outlier and why?
- Where do prices cluster? (e.g., $10–20/user, $50–100/user, $200+ enterprise)
- Who has freemium in a mostly paid market, or vice versa?
- Who is moving to AI-based pricing or usage-based in a per-seat market?

## Step 8: Generate landscape report

Generate a self-contained HTML report using the PricingSaaS monochrome template. The full CSS template lives at:
`https://raw.githubusercontent.com/pricingsaas/pricingsaas-claude-skills/main/plugins/pricingsaas/skills/pulse-scan/references/report-structure.md`

**Read that file and copy the CSS verbatim.** Do NOT write custom styles, import Google Fonts, use colored headers, add box-shadows, or deviate from the monochrome design system.

### Report sections

1. **Header** — category title + PricingSaaS branding (+ seed company logo if applicable)
2. **Stats row** — companies found · pricing models · price range · freemium count
3. **Executive summary** — 1–2 paragraphs: category overview, how landscape breaks down
4. **Market layers** — one table per tier with: Company · Plan · Monthly · Annual · Model · PricingSaaS link
5. **Pricing patterns** — insight boxes for key observations (model dominance, freemium, AI, price clusters)
6. **Bar chart** — vertical price comparison across companies (tallest bar = 100%, others scaled proportionally)
7. **Data sources** — company grid with all companies, linked to pricingsaas.com profiles
8. **Footer**

Write to `/tmp/{category}-pricing-landscape.html`, then upload:

```
upload_report(filename="{category}-pricing-landscape.html", file_path="/tmp/{category}-pricing-landscape.html")
```

Execute the returned curl command to complete the upload:

```bash
curl -X PUT -H "Content-Type: text/html" --data-binary @"/tmp/{category}-pricing-landscape.html" "{presigned-url}"
```

The tool response includes the final public URL.

## Step 9: Deliver conversational summary

Structure the response as:

```
**{Category} pricing landscape**
Full report: {upload_report URL}

---

{Category overview — how many companies, how the landscape breaks down}

**Market layers**
{For each tier: companies, price range, dominant model}

**Key patterns**
- {Pattern 1}
- {Pattern 2}
- {Pattern 3}

**Recent market moves** (from get_pricing_news)
{Any notable changes in this category}

**Forward-looking signals**
{Job postings analysis: which companies are hiring for pricing/monetization roles and what it suggests}
{Wayback Machine findings: any notable trajectory observations from historical snapshots}

---

Full report: {upload_report URL}
```

Always present the report link at top and bottom. Include `https://pricingsaas.com/pulse/companies/{slug}` links for every company named in the summary.

After delivering the summary, offer once:
> "Want me to check what customers are saying about pricing in this space — Reddit, G2, HN sentiment across the top players?"

If yes, run [sentiment-research.md](sentiment-research.md) for the top 3–5 most significant companies in the landscape.

## Step 10: Log to monday

After delivering the summary, follow [monday-logging.md](monday-logging.md) to log the results. For landscape scans, create one item representing the scan (not one per company).

- Item name: `{Category} — Landscape`
- Change Type: `Landscape Scan`
- Summary: 1–2 sentences on the dominant pricing model and key market pattern
- PricingSaaS link: leave blank (landscape scans have no single company diff URL)
- Workflow: `trend-research`

## Step 11: Recommend next steps

After the summary, offer 3 tailored follow-ups using specific company names and findings:

1. **Company deep-dive** — "Want a full breakdown of how {top competitor} structures their pricing? I can pull their plan details, packaging logic, history, and Wayback Machine page evolution."
2. **Monitor the market** — "Want to track when any of these {N} companies changes pricing? I can add them all to your watchlist now."
3. **Sentiment sweep** — "Want to check what customers and operators are saying about pricing in this space across Reddit, G2, and HN?"
