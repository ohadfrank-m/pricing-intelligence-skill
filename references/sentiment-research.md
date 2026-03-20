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

### LinkedIn — articles and operator commentary

LinkedIn surfaces thoughtful practitioner reactions — founders, growth leads, and product operators who write long-form takes on pricing moves.

**Articles (LinkedIn Pulse):**
```
WebSearch(query='"{Company}" pricing site:linkedin.com/pulse')
WebSearch(query='"{Company}" "price increase" OR "pricing strategy" site:linkedin.com/pulse 2025 OR 2026')
```

If a specific article URL is returned in results, fetch it:
```
WebFetch(url="{linkedin_pulse_article_url}")
```

**Posts and general discussion:**
```
WebSearch(query='"{Company}" pricing change 2025 OR 2026 site:linkedin.com')
WebSearch(query='"{Company}" price increase reaction operators founders site:linkedin.com')
```

**Thought leader search** — pricing practitioners with large followings often write about notable pricing moves:
```
WebSearch(query='"{Company}" pricing (Kyle Poyar OR Patrick Campbell OR "Lenny Rachitsky" OR OpenView OR Bessemer OR ProfitWell OR "Madhavan Ramanujam")')
WebSearch(query='"{Company}" pricing "Growth Unhinged" OR "monetization" site:linkedin.com')
```

### X / Twitter — real-time reactions

X is the highest-velocity signal source — backlash, praise, and viral pricing drama surface here first, often within hours of an announcement.

**Broad post search:**
```
WebSearch(query='"{Company}" pricing site:x.com OR site:twitter.com')
WebSearch(query='"{Company}" "price increase" OR "too expensive" OR "pricing change" site:x.com')
WebSearch(query='"{Company}" pricing (founder OR SaaS OR "product-led" OR "per seat") site:x.com')
```

**Prominent SaaS voices** — these accounts regularly comment on notable pricing moves:
```
WebSearch(query='"{Company}" pricing from:KylePoyar OR from:ProfitWell OR from:jasonlk OR from:hitenshah OR from:lennysan site:x.com')
WebSearch(query='"{Company}" pricing site:x.com (VC OR "seed" OR "Series" OR "growth" OR "PLG")')
```

**Nitter fallback** — if direct WebSearch against x.com returns sparse results, try fetching via nitter (public Twitter mirror):
```
WebFetch(url="https://nitter.net/search?q={Company}+pricing&f=tweets")
WebFetch(url="https://nitter.net/search?q={Company}+%22price+increase%22&f=tweets")
```

If nitter is unreachable, skip silently and note "X coverage: not retrievable via current tooling."

---

### Earnings calls (public companies only)

For publicly traded companies (HubSpot, Salesforce, Canva post-IPO, etc.):

```
WebSearch(query='"{Company}" pricing earnings call transcript 2025 OR 2026')
WebSearch(query='"{Company}" "pricing strategy" OR "price increase" investor relations 2026')
WebFetch(url="https://seekingalpha.com/symbol/{TICKER}/earnings/transcripts")
```

Look for: management commentary on pricing rationale, analyst questions about pricing pressure, customer reaction mentions, churn attribution to pricing.

---

### Publications — editorial and analyst coverage

Publication coverage signals that a pricing move has crossed from "product news" to "industry event." Run all three tiers in parallel.

**Tier 1 — tech and business press** (high reach, broad audience):
```
WebSearch(query='"{Company}" pricing site:techcrunch.com OR site:wsj.com OR site:bloomberg.com OR site:forbes.com OR site:businessinsider.com')
WebSearch(query='"{Company}" "price increase" OR "pricing strategy" OR "pricing change" news 2025 OR 2026')
WebSearch(query='"{Company}" pricing site:cnbc.com OR site:theverge.com OR site:venturebeat.com')
```

If a specific article is returned, fetch it for the full text:
```
WebFetch(url="{article_url}")
```

**Tier 2 — SaaS-specific press and newsletters** (practitioner audience, high signal-to-noise):
```
WebSearch(query='"{Company}" pricing site:saastr.com OR site:openviewpartners.com OR site:a16z.com OR site:bvp.com')
WebSearch(query='"{Company}" pricing "Growth Unhinged" OR "Kyle Poyar" OR "Lenny" OR "ProfitWell" OR "Price Intelligently"')
WebSearch(query='"{Company}" pricing site:substack.com (SaaS OR pricing OR monetization)')
```

Known newsletter/blog URLs to try directly when the company is highly relevant:
```
WebFetch(url="https://kylepoyar.substack.com/search?query={Company}")
WebFetch(url="https://www.lennysnewsletter.com/search?q={Company}+pricing")
```

**Tier 3 — analyst and research coverage** (enterprise buyer signal):
```
WebSearch(query='"{Company}" pricing site:gartner.com OR site:forrester.com')
WebSearch(query='"{Company}" pricing analyst report OR "magic quadrant" OR "wave report" 2025 OR 2026')
WebSearch(query='"{Company}" pricing commentary site:g2.com/categories OR site:peerspot.com')
```

**Publication coverage scoring:** Note how many publications covered the pricing move and at what tier. Tier 1 coverage = the change was significant enough for mainstream business attention. Tier 2 coverage = SaaS community is paying attention. Tier 3 coverage = enterprise procurement teams and analysts are tracking it.

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

**Source weighting by signal type:**

| Source | Signal type | Weight |
|--------|------------|--------|
| X / Twitter | Real-time velocity — first to surface backlash or praise, often within hours | Highest for recency |
| Reddit / HN | Depth and authenticity — longer-form user reactions, less PR-filtered | Highest for candor |
| LinkedIn | Practitioner reaction — founders, PMs, growth leads giving thoughtful takes | High for operator sentiment |
| G2 / Capterra | End-user and buyer reaction — structured, often tied to renewal decisions | High for buyer sentiment |
| Publications (Tier 1) | Mainstream business signal — coverage means the move was large enough to matter broadly | Signals magnitude |
| Publications (Tier 2) | SaaS community signal — practitioner press is covering it as an industry pattern | Signals strategic interest |
| Publications (Tier 3) | Analyst / enterprise signal — procurement teams and analysts are tracking it | Signals enterprise sales impact |
| Earnings calls | Official management framing — how the company explains the pricing decision | Signals intent and confidence |

When sources conflict (e.g., G2 reviews are negative but X shows acceptance), surface both readings and note the discrepancy — the audience segments are different.

---

## Step 3: Output format

```
## {Company} — pricing sentiment

**Overall tone:** {Positive / Mixed / Negative / No signal}
**Sources checked:** Reddit · HN · G2 · Capterra · LinkedIn · X/Twitter · Publications · {other}
**Signal strength:** {Strong (multiple sources, high volume) / Moderate / Weak (sparse discussion)}

### Key themes
- {Theme 1}: {description with evidence}
- {Theme 2}: {description with evidence}

### Representative quotes
> "{quote}" — [Reddit/{subreddit}]({url}), {date}
> "{quote}" — [G2 review]({url}), {date}
> "{quote}" — [LinkedIn / {author name}]({url}), {date}
> "{quote}" — [X / @{handle}]({url}), {date}
> "{quote}" — [{Publication name}]({url}), {date}

### What this means
{2-3 sentences: strategic interpretation. Is the market absorbing this? Are customers at risk of churning? Does the reaction suggest the pricing change was well-executed or clumsy?}

### Sources
**Community:** [{Reddit/HN thread title}]({url}) · [{G2 review}]({url})
**LinkedIn:** [{Author name — article/post title}]({url})
**X/Twitter:** [{@handle — post summary}]({url})
**Publications:** [{Publication — article title}]({url}) · [{Newsletter — post title}]({url})
**Earnings / analyst:** [{Company earnings call Q{N} {year}}]({url})
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
- Weight Reddit and HN highest for candor and depth. Weight X/Twitter highest for recency and velocity. Weight LinkedIn for practitioner/operator reaction. Weight G2 and Capterra for structured buyer reaction. Weight publications for magnitude signal.
- A "no reaction" finding is itself useful signal — it means the change was either too small, too gradual, or targeted a segment that doesn't discuss pricing publicly.
- Date-filter all searches to the last 6 months unless the user asks for historical sentiment.
- X/Twitter and nitter may be unreachable in some environments — if both fail, note it once and move on. Do not block the workflow.
- LinkedIn pulse articles are often the highest-quality practitioner takes — prioritize fetching full text when a relevant article URL is found.
