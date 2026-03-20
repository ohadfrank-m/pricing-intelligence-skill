# Sentiment research workflow

Surfaces how customers, operators, and the market are *reacting* to a company's pricing — not just what changed. This is the signal that PricingSaaS alone can't provide.

## When to run

- User asks: "what do people think about X's pricing", "how are customers reacting to X's price increase", "pricing sentiment for X", "is X's pricing controversial"
- After pulling a pricing change via monitoring or company-research, offer to run sentiment as a follow-up
- Any time a significant change is detected (price increase, restructure, plan removal)

## Step 1: Build the search queries

Run all searches in parallel. Use the company name and any known pricing change details (price points, plan names, timing) to make queries specific.

### Reddit

```
WebSearch(query='"{Company}" pricing site:reddit.com')
WebSearch(query='"{Company}" "price increase" OR "too expensive" OR "pricing change" site:reddit.com')
WebSearch(query='"{Company}" pricing r/SaaS OR r/entrepreneur OR r/{company_subreddit}')
```

### Hacker News

```
WebSearch(query='"{Company}" pricing site:news.ycombinator.com')
WebSearch(query='site:news.ycombinator.com "{Company}" pricing')
```

Or fetch HN search directly:

```
WebFetch(url="https://hn.algolia.com/api/v1/search?query={Company}+pricing&tags=story&numericFilters=created_at_i>1735689600")
```

(The timestamp filters to posts after Jan 1 2026 — adjust as needed)

### G2 / Capterra / Trustpilot

```
WebSearch(query='"{Company}" pricing reviews site:g2.com')
WebSearch(query='"{Company}" "pricing" OR "expensive" OR "value for money" site:capterra.com')
WebSearch(query='"{Company}" pricing site:trustpilot.com')
```

For G2, also try fetching the pricing-filtered review page directly:

```
WebFetch(url="https://www.g2.com/products/{company-slug}/reviews?filters[review_answers][73][]=1")
```

### LinkedIn / operator commentary

```
WebSearch(query='"{Company}" pricing change 2025 OR 2026 site:linkedin.com')
WebSearch(query='"{Company}" price increase reaction operators founders')
```

### Earnings calls (public companies only)

For publicly traded companies (HubSpot, Salesforce, Canva post-IPO, etc.):

```
WebSearch(query='"{Company}" pricing earnings call transcript 2025 OR 2026')
WebSearch(query='"{Company}" "pricing strategy" OR "price increase" investor relations 2026')
WebFetch(url="https://seekingalpha.com/symbol/{TICKER}/earnings/transcripts")
```

Look for: management commentary on pricing rationale, analyst questions about pricing pressure, customer reaction mentions, churn attribution to pricing.

---

## Step 2: Synthesize findings

After collecting search results, categorize the sentiment signal:

**Tone** — Overall positive / mixed / negative / no signal

**Key themes** (pick whichever apply):
- "Price shock" — customers surprised by increase magnitude
- "Value questioned" — sentiment that pricing no longer matches value delivered
- "Switching intent" — mentions of evaluating alternatives post-change
- "Acceptance" — change absorbed with minimal friction
- "Praise" — customers calling pricing fair or generous
- "No reaction" — change happened quietly, no detectable public response

**Representative quotes** — 2–4 direct quotes from sources with URL citation

**Timing signal** — When did the sentiment spike? Immediately post-change, or delayed (delayed often means churn is still materializing)?

---

## Step 3: Output format

```
## {Company} — pricing sentiment

**Overall tone:** {Positive / Mixed / Negative / No signal}
**Sources checked:** Reddit · HN · G2 · Capterra · {other}
**Signal strength:** {Strong (multiple sources, high volume) / Moderate / Weak (sparse discussion)}

### Key themes
- {Theme 1}: {description with evidence}
- {Theme 2}: {description with evidence}

### Representative quotes
> "{quote}" — [Reddit/{subreddit}]({url}), {date}
> "{quote}" — [G2 review]({url}), {date}

### What this means
{2-3 sentences: strategic interpretation. Is the market absorbing this? Are customers at risk of churning? Does the reaction suggest the pricing change was well-executed or clumsy?}

### Sources
- [{Source title}]({url})
- [{Source title}]({url})
```

---

## Step 4: Log to monday

After delivering sentiment output, log to the Pricing Intelligence board per [monday-logging.md](monday-logging.md):

- Item name: `{Company} — Sentiment`
- Summary: overall tone + key theme in 1–2 sentences
- PricingSaaS link: `https://pricingsaas.com/companies/{slug}`
- Workflow: `sentiment-research`

---

## Important notes

- Not all companies will have detectable public sentiment — niche B2B tools often have near-zero public discussion. Say so explicitly rather than filling with noise.
- Weight Reddit and HN higher than G2 for operator/founder reactions. Weight G2 and Capterra higher for end-user/buyer reactions.
- A "no reaction" finding is itself useful signal — it means the change was either too small, too gradual, or targeted a segment that doesn't discuss pricing publicly.
- Date-filter all searches to the last 6 months unless the user asks for historical sentiment.
