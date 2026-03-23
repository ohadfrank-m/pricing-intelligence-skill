# A/B test detection workflow

Detect when a competitor is actively testing their pricing page. The primary method is scanning the live page source for A/B testing tool signatures — this works for every company regardless of archive access. Archive frequency analysis (Wayback CDX) is used as a secondary signal when available, but the CDX API is frequently blocked (403) for large SaaS companies, so it is never the first or only approach.

## When to run

- User asks: "is X testing their pricing page", "is X A/B testing pricing", "check for pricing page experiments at X", "which competitors are actively changing their pricing page"
- Automatically as part of company-research — the pricing page fetch from the teardown step already gives you the content needed for Step 1 below at zero extra cost
- As part of the weekly digest (run for top 3–5 watchlist companies in parallel)
- As a follow-up after monitoring detects a change: "Want me to check if they're actively testing the page presentation around this change?"

---

## Signal hierarchy

Run signals in this order. Stop when you have enough confidence for a clear classification.

| Priority | Signal | Reliability | Always available? |
|----------|--------|-------------|-------------------|
| 1 | A/B testing tool signatures in live page source | Very High | Yes — page fetch works for all companies |
| 2 | BuiltWith technology stack lookup | High | Yes — public lookup |
| 3 | Wayback Availability API (single-call, lightweight) | Medium | Usually (lighter than CDX) |
| 4 | CRO / Growth Engineering job postings | Medium (leading indicator) | Yes — WebSearch |
| 5 | Wayback CDX API (frequency analysis) | High when it works | No — blocked for most large SaaS |

---

## Step 1: Fetch the pricing page and scan for testing tool signatures

If the pricing page was already fetched as part of company-research or teardown, reuse that content. Otherwise:

```
WebFetch(url="https://{domain}/pricing")
```

Once you have the page content, scan the full text (including any visible script tags, data attributes, or embedded JSON) for these signatures. Search case-insensitively.

### A/B testing tools

| Signature to look for | Tool | What it means |
|-----------------------|------|---------------|
| `optimizely` | Optimizely | Enterprise-grade A/B testing; if present, tests are almost certainly running |
| `_vwo_` or `vwo.com` | Visual Website Optimizer | Popular mid-market testing tool |
| `abtasty` | AB Tasty | Common in European SaaS |
| `statsig` | Statsig | Very common in PLG/product-led companies (Notion, Figma, etc.) |
| `launchdarkly` | LaunchDarkly | Feature flags often used to gate pricing variants |
| `split.io` or `splitio` | Split.io | Feature flag + experiment platform |
| `amplitude-experiment` or `@amplitude/experiment` | Amplitude Experiments | Used by companies already on Amplitude analytics |
| `growthbook` | GrowthBook | Open-source; common in engineering-led companies |
| `posthog` | PostHog | Open-source; popular with startups |
| `kameleoon` | Kameleoon | Enterprise testing, common in France/EU |
| `google-optimize` or `gtag.*optimize` | Google Optimize (deprecated) | Occasionally still running on older setups |
| `monetate` | Monetate | Enterprise personalization + testing |

### Feature flag / experimentation platforms (also count)

Feature flags are how engineering teams run pricing page variants — they're functionally equivalent to A/B tests. If present, a pricing test could be running without a dedicated A/B tool.

| Signature | Platform |
|-----------|----------|
| `launchdarkly` | LaunchDarkly |
| `unleash` | Unleash |
| `flagsmith` | Flagsmith |
| `statsig` | Statsig (also appears above) |
| `configcat` | ConfigCat |

### What to do with the findings

**Tool found:** Report which tool, note that experiment infrastructure is active, and check whether any experiment names or variant IDs are exposed in the page source. Optimizely and VWO in particular often expose experiment metadata in JavaScript variables (`window.optimizely`, `window._vwo_exp`) — if visible, read it to determine what's being tested.

**No tool found:** Either they're not testing, or they use a fully server-side experimentation system (no client-side fingerprint). Move to Step 2.

---

## Step 2: BuiltWith technology stack lookup

Run this in parallel with Step 1 — it confirms what tools are installed on the domain even if the page source is heavily minified or the scripts are loaded asynchronously (making them invisible in a basic WebFetch).

```
WebSearch(query='site:builtwith.com "{domain}"')
WebFetch(url="https://builtwith.com/{domain}")
```

Look for: Optimizely, VWO, Statsig, LaunchDarkly, AB Tasty, or any other experimentation platform in the results.

**If the BuiltWith fetch is blocked:** Try the technology overlay:
```
WebSearch(query='"{domain}" Optimizely OR VWO OR Statsig OR LaunchDarkly OR "AB testing" technology stack')
```

---

## Step 3: Wayback Availability API (lightweight archive check)

Unlike the CDX API (which is frequently blocked), the Availability API is a single lightweight call:

```
WebFetch(url="https://archive.org/wayback/available?url={domain}/pricing")
```

This returns the most recent successful snapshot timestamp. Use it to:
- Confirm the page is being archived at all
- Compare the snapshot date to the PricingSaaS last-change date — if they're close, the archive caught a recent change

If this also returns an error, skip to Step 4. Do not treat a CDX/availability failure as a signal — it's an access issue, not evidence of anything about the company.

---

## Step 4: CRO and Growth Engineering job postings

A company running pricing page experiments needs people to run them. Search for active roles:

```
WebSearch(query='"{Company}" "growth engineer" OR "CRO" OR "conversion rate" OR "experimentation" job 2025 OR 2026 site:linkedin.com OR site:greenhouse.io OR site:lever.co')
```

**Signal:** An open Growth Engineer or Experimentation Platform role = active testing program. A recently closed one = they just staffed up and experiments are likely running now.

---

## Step 5: PricingSaaS history cross-reference

```
get_company_history(slug="{slug}", discovery_only=true)
```

Cross-reference the PricingSaaS diff history with any signals found above to distinguish test type:

| Testing tool found | PricingSaaS diffs in same window | Interpretation |
|--------------------|----------------------------------|----------------|
| Yes | No | **Conversion optimization test** — testing copy, CTA, layout, framing. Prices unchanged. |
| Yes | Yes | **Pricing launch iteration** — launched a price change and testing page presentation around it. |
| No | No recent changes | **No active testing detected** — page appears stable. |
| No | Yes | **Real pricing change** — structural update, not a test. |

---

## Step 6: Wayback CDX (frequency analysis — when available)

Only attempt this if Steps 1–4 gave ambiguous results and you want to validate. The CDX API is blocked for most large SaaS companies (returns 403).

```
WebFetch(url="https://web.archive.org/cdx/search/cdx?url={domain}/pricing&output=json&limit=50&fl=timestamp,statuscode&filter=statuscode:200&from={90-days-ago-YYYYMMDD}&to={today-YYYYMMDD}")
```

If it works, parse the response: count snapshots per 30-day window and look for dense clusters (5+ snapshots in a single week = active test signal).

**Scoring model (if CDX returns data):**

| 30-day snapshot count | Signal level |
|-----------------------|-------------|
| 0–1 | None |
| 2–3 | Low |
| 4–6 | Medium |
| 7–10 | High |
| 11+ | Very High |

Clustering matters more than raw count: 5 snapshots spread over 90 days = low signal. 5 snapshots in one week = high signal.

If CDX returns 403: note it, skip this step entirely. Do not treat the 403 as a finding.

---

## Step 7: Output format

```
## {Company} — pricing page A/B test detection

**Pricing page URL:** {url}
**Detection date:** {today}

### Testing tool scan

**Tools detected:** {list, or "None found"}
**Experiment metadata visible:** {Yes — {details} / No}

{If tool found:}
> {Tool name} is active on this page. {Company} has live experiment infrastructure running on their pricing page.
> {If experiment names/IDs visible: "Experiment detected: '{name}' — variants: {list}"}

### Technology stack (BuiltWith)

{What BuiltWith shows for experimentation/testing tools}

### Archive signal

**Wayback Availability API:** {Most recent snapshot date, or "blocked"}
**CDX frequency (if available):** {snapshot count / 30-day breakdown, or "CDX blocked — not used"}

### PricingSaaS cross-reference

{Recent diffs from discovery_only scan — periods and change types}
{Interpretation: does archive/test activity coincide with a real pricing change, or is it pure conversion testing?}

### Job posting signal

{Any open CRO / Growth Engineering / Experimentation roles — date posted and what they imply}

---

### Classification

**Signal level:** {None / Low / Medium / High / Confirmed}

**Likely activity:** {One of:}

- **No active testing** — No testing tools found, no CRO hiring, no unusual archive activity. Page is stable.
- **Infrastructure present, test status unknown** — Testing tool found but no experiment metadata visible. They have the capability; unclear if pricing page is the active test subject.
- **Active conversion test** — Testing tool found + no PricingSaaS price changes in window. Testing copy, CTA, framing, or layout. **Expect a page change or stabilization within 4–8 weeks.**
- **Active pricing launch iteration** — Testing tool found + coincides with PricingSaaS diff. They launched a pricing change and are optimizing the page around it.
- **Confirmed A/B test** — Tool found AND experiment metadata visible in page source. Highest confidence — you can read what's being tested.

### Recommended action

{One of:}
- "No action needed — page is stable, no testing infrastructure detected."
- "Run a teardown now to capture the current page state before a variant wins: 'tear down {Company}'s pricing page.'"
- "Re-check in 3 weeks — if a variant wins, the page will stabilize and PricingSaaS will likely record a diff."
- "Check experiment: '{name}' — {what it tells you about what hypothesis they're testing}."
```

---

## Step 7b: Multi-variant page rendering detection

The standard `WebFetch` returns only one version of the pricing page. Real A/B tests serve different content to different visitors. This step attempts to catch the test in action by fetching the page multiple times with varied conditions and diffing the responses.

### Method 1: Multiple fetches with cache-busting

Fetch the pricing page 3 times in parallel with different query parameters to bypass any CDN caching:

```
WebFetch(url="https://{domain}/pricing?_v=1&t={timestamp_1}")
WebFetch(url="https://{domain}/pricing?_v=2&t={timestamp_2}")
WebFetch(url="https://{domain}/pricing?_v=3&t={timestamp_3}")
```

Use different timestamps (seconds apart) for each fetch. Compare the three responses for differences in:
- Plan names or tier labels
- Price values (monthly, annual)
- CTA button text ("Start free trial" vs. "Get started" vs. "Try for free")
- Feature list content or ordering
- Hero plan badge ("Most popular", "Best value", "Recommended")
- Annual/monthly toggle default state
- Social proof elements (logo bar, customer count, testimonial)

### Method 2: Browser MCP with cookie/state variation

If Method 1 returns identical content (most server-side tests use cookies, not query params), use the browser MCP for deeper detection:

```
browser_navigate(url="https://{domain}/pricing")
browser_snapshot()
```

Then clear cookies and navigate again:

```
browser_navigate(url="https://{domain}/pricing")
browser_snapshot()
```

Compare the two snapshots. Client-side A/B tools (Optimizely, VWO) often assign variants via cookies — clearing cookies between visits may surface a different variant.

### Diff analysis

If any differences are found between fetches:

```
### Live A/B test detected — variant comparison

| Element | Variant A | Variant B | Variant C |
|---------|-----------|-----------|-----------|
| {element} | {value} | {value} | {value} |
| {element} | {value} | {value} | — |

**Variants detected:** {count}
**Test scope:** {Copy only / Pricing values / Plan structure / Full page redesign}
**Confidence:** Confirmed — multiple page versions served to different requests
```

If all fetches return identical content: note "Multi-variant check: no content differences detected across {N} fetches. Either no active test, or test is segmented by geo/account/cookie that cannot be varied via fetch."

---

## Step 7c: Experiment hypothesis reverse-engineering

When testing infrastructure is detected (Step 1) or a live A/B test is confirmed (Step 7b), systematically decode *what the competitor is testing* by combining tool metadata with pricing page teardown analysis.

### Extract experiment metadata

If experiment names or variant IDs are visible in the page source (common with Optimizely and VWO):

**Optimizely:**
```
Look for: window.optimizely, optimizely.get('state'), experiment IDs in data-optly-* attributes
Extract: experiment name, variant names, audience targeting rules
```

**VWO:**
```
Look for: window._vwo_exp, _vwo_exp_ids, VWO campaign data in page scripts
Extract: campaign name, goal name (reveals what metric they're optimizing for)
```

**Statsig:**
```
Look for: statsig, experiment gate names in network requests or embedded config
Extract: gate name (often descriptive, e.g., "pricing_page_v2", "annual_default_test")
```

**LaunchDarkly:**
```
Look for: ldclient, feature flag keys in page source or network requests
Extract: flag key names (e.g., "show-enterprise-tier", "pricing-cta-variant")
```

### Map metadata to hypothesis

Combine any visible experiment name/ID with the pricing page teardown dimensions from [pricing-page-teardown.md](pricing-page-teardown.md) to infer the test hypothesis:

| Experiment signal | Likely hypothesis being tested |
|-------------------|-------------------------------|
| CTA text varies between fetches | Conversion rate optimization — testing which action language drives more signups |
| Hero plan badge changes | Tier conversion — testing which plan to push buyers toward |
| Price values differ | Price elasticity test — actively testing willingness to pay at different price points |
| Annual/monthly default toggles | Revenue optimization — testing whether annual-default increases contract value |
| Feature list reordered | Value perception — testing which features drive purchase decisions |
| Plan count differs (e.g., 3 vs. 4 tiers) | Packaging test — testing whether a simpler or more granular tier structure converts better |
| Free tier visible in one variant, hidden in another | Freemium test — testing whether hiding free tier increases paid conversion |
| Enterprise "Contact Sales" vs. visible price | Sales motion test — testing self-serve enterprise vs. sales-led |

### Output format for hypothesis

```
### Experiment hypothesis (reverse-engineered)

**Experiment name:** {name if visible, or "Not visible"}
**Likely hypothesis:** "{If they [change being tested], then [metric] will [direction] because [reasoning]}"
**Test scope:** {element being varied}
**What this tells us:** {1-2 sentences on what this reveals about their pricing strategy direction — e.g., "They're testing whether they can push more buyers to the Business tier, suggesting their current conversion is heavily weighted toward the lower tier."}
**monday.com implication:** {1 sentence — does this test, if successful, create a competitive positioning issue for monday.com?}
```

---

## Step 8: Compare two page versions (if testing is confirmed)

If the signal is Medium or above, offer to fetch an earlier version of the page from Wayback to show what already changed:

```
# Most recent available snapshot (from Availability API result)
WebFetch(url="https://web.archive.org/web/{recent-timestamp}/{domain}/pricing")

# Earlier snapshot for comparison (try 30–60 days prior)
WebFetch(url="https://web.archive.org/web/{earlier-timestamp}/{domain}/pricing")
```

Diff the two versions and present as a before/after table:

| Element | Before | After |
|---------|--------|-------|
| Hero plan badge | — | "Most popular" added |
| Business plan CTA | "Start free trial" | "Get started" |
| Annual toggle default | Monthly | Annual |
| Feature list order | Security listed last | Security listed first |

If Wayback snapshots are unavailable, offer the pricing page teardown as an alternative: "I can run a full teardown of the current page to capture its state before it changes."

---

## Step 9: Log to monday

Follow [monday-logging.md](monday-logging.md):

- Item name: `{Company} — A/B Test Signal`
- Summary: Tool detected + classification + one-line on what's likely being tested
- PricingSaaS link: `https://pricingsaas.com/companies/{slug}`
- Workflow: `ab-test-detection`

Only log if signal level is Medium or above. Low/None signals don't need a monday entry.

---

## Step 10: Pricing page change velocity tracking

After every scan (single company or watchlist sweep), persist the current state to the knowledge base to build a change-over-time record. This turns one-off detection into a longitudinal signal — you can see *how often* a competitor is iterating on their pricing page, which is itself a strategic signal.

### Persist current scan

After completing all detection steps, write the following to the knowledge base entry for this company (see [knowledge-base.md](knowledge-base.md)):

```json
{
  "ab_test_status": {
    "last_scanned": "ISO date",
    "testing_tools_detected": ["Optimizely", "Statsig"],
    "active_experiments": [
      {
        "detected_date": "ISO date",
        "experiment_name": "pricing_cta_v3",
        "scope": "CTA text",
        "variants_detected": 2,
        "hypothesis": "Testing whether 'Start free trial' vs 'Get started' affects signup rate",
        "still_active": true
      }
    ],
    "page_snapshots": [
      {
        "date": "ISO date",
        "plan_count": 4,
        "price_points": {"Individual": 0, "Basic": 12, "Standard": 17, "Pro": 28},
        "cta_text": "Get started",
        "hero_plan": "Standard",
        "annual_default": true,
        "hash": "{content_hash_of_key_pricing_elements}"
      }
    ]
  }
}
```

### Velocity analysis

When the knowledge base already has prior scan data for this company, compute the velocity metrics:

```
### Pricing page change velocity — {Company}

**Scans on record:** {count} over {time span}
**Changes detected:** {count of snapshots where hash differs from previous}
**Change frequency:** {changes per month}

| Date | What changed | Experiment active? |
|------|-------------|-------------------|
| {date} | {specific change} | {Yes — experiment name / No} |
| {date} | {specific change} | {Yes / No} |

**Velocity classification:**
```

| Frequency | Classification | Signal |
|-----------|---------------|--------|
| 0 changes in 3+ months | **Static** | Not iterating on pricing — either confident or neglecting it |
| 1 change per quarter | **Normal** | Standard iteration cadence |
| 2+ changes per quarter | **Active** | Actively optimizing pricing — treat as high-priority competitor |
| 4+ changes per quarter | **Rapid** | Aggressive pricing experimentation — likely has dedicated pricing/growth team |

**monday.com implication:** Companies classified as "Active" or "Rapid" should be scanned more frequently and flagged for the proactive monitoring workflow.

### Cross-company velocity comparison

If the knowledge base has velocity data for 3+ companies, include a comparative view:

```
### Competitive pricing iteration velocity

| Company | Scans | Changes | Frequency | Classification |
|---------|-------|---------|-----------|---------------|
| {Company A} | {n} | {n} | {n/mo} | Rapid |
| {Company B} | {n} | {n} | {n/mo} | Normal |
| {Company C} | {n} | {n} | {n/mo} | Static |

**Pattern:** {1 sentence — e.g., "Direct competitors are iterating 2-3x faster than adjacent players, suggesting the competitive pricing space is actively contested."}
```

---

## Running for multiple companies (watchlist sweep)

**Steps 1 and 2 are entirely free — no MCP credits required. These must always run as part of the weekly digest regardless of credit balance.**

When called as part of the weekly digest, run all page fetches and BuiltWith lookups in parallel:

```
# Fetch all pricing pages simultaneously:
WebFetch(url="https://{company1-domain}/pricing")
WebFetch(url="https://{company2-domain}/pricing")
WebFetch(url="https://{company3-domain}/pricing")

# BuiltWith lookups simultaneously:
WebSearch(query='site:builtwith.com "{company1-domain}"')
WebSearch(query='site:builtwith.com "{company2-domain}"')
```

Return a summary table first, then full detail only for Medium+ signals:

| Company | Testing tool found | Experiment visible | Classification | Action |
|---------|-------------------|-------------------|----------------|--------|
| {Company A} | None | — | None | — |
| {Company B} | Statsig | No | Infrastructure present | Teardown recommended |
| {Company C} | Optimizely | Yes — "pricing_plan_v3" | Confirmed A/B test | Read experiment details |
