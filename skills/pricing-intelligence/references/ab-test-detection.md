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
