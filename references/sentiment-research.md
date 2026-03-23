# Sentiment research workflow

Surfaces how customers, operators, and the market are *reacting* to a company's pricing — not just what changed. This is the signal that PricingSaaS alone can't provide. The goal is deep conviction: enough cross-source evidence to confidently assess whether a pricing move landed well, badly, or went unnoticed.

## When to run

- User asks: "what do people think about X's pricing", "how are customers reacting to X's price increase", "pricing sentiment for X", "is X's pricing controversial"
- After pulling a pricing change via monitoring or company-research, offer to run sentiment as a follow-up
- Any time a significant change is detected (price increase, restructure, plan removal)

## Step 1: Build the search queries

Run all source tiers in parallel. Use the company name and any known pricing change details (price points, plan names, timing) to make queries specific.

### Reddit — depth and authenticity

Reddit surfaces the most candid user reactions. Search multiple subreddits and query patterns to maximize coverage.

**Core searches:**
```
WebSearch(query='"{Company}" pricing site:reddit.com')
WebSearch(query='"{Company}" "price increase" OR "too expensive" OR "pricing change" site:reddit.com')
WebSearch(query='"{Company}" pricing r/SaaS OR r/entrepreneur OR r/{company_subreddit}')
```

**Deep community mining — extend reach to operational and buyer subreddits:**
```
WebSearch(query='"{Company}" pricing OR "switched from" OR "alternative to" site:reddit.com/r/startups OR site:reddit.com/r/smallbusiness OR site:reddit.com/r/sysadmin')
WebSearch(query='"{Company}" pricing OR cost OR expensive site:reddit.com/r/projectmanagement OR site:reddit.com/r/ProductManagement OR site:reddit.com/r/devops')
WebSearch(query='"{Company}" "not worth" OR "overpriced" OR "good value" OR "best price" site:reddit.com')
```

**Engagement capture:** When a Reddit result is found, note the upvote count and comment count visible in the search snippet. These feed into the conviction score (Step 2b).

### Hacker News — technical and founder sentiment

```
WebSearch(query='"{Company}" pricing site:news.ycombinator.com')
WebSearch(query='site:news.ycombinator.com "{Company}" pricing')
```

Or fetch HN search directly:

```
WebFetch(url="https://hn.algolia.com/api/v1/search?query={Company}+pricing&tags=story&numericFilters=created_at_i>1735689600")
```

(The timestamp filters to posts after Jan 1 2026 — adjust as needed)

**HN comment depth:** If the Algolia API returns results, check `num_comments` and `points`. Threads with 100+ points or 50+ comments are high-signal — fetch the full thread for representative quotes:

```
WebFetch(url="https://hn.algolia.com/api/v1/items/{objectID}")
```

### G2 / Capterra / Trustpilot / PeerSpot — structured buyer reviews

```
WebSearch(query='"{Company}" pricing reviews site:g2.com')
WebSearch(query='"{Company}" "pricing" OR "expensive" OR "value for money" site:capterra.com')
WebSearch(query='"{Company}" pricing site:trustpilot.com')
WebSearch(query='"{Company}" pricing review site:peerspot.com')
```

For G2, also try fetching the pricing-filtered review page directly:

```
WebFetch(url="https://www.g2.com/products/{company-slug}/reviews?filters[review_answers][73][]=1")
```

**G2/Capterra review mining — look for:**
- Star ratings specifically on "Value for Money" or "Pricing" dimensions
- Reviews that mention price increases, hidden costs, or billing surprises
- Reviewer titles and company sizes (enterprise buyer vs. freelancer = different signal)
- Recency — reviews from the last 90 days are most relevant after a pricing change

### LinkedIn — operator and practitioner commentary

LinkedIn surfaces thoughtful practitioner reactions — founders, growth leads, product operators, and pricing specialists who write long-form takes on pricing moves.

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

**Extended thought leader search — SaaS pricing and growth experts:**

The following practitioners regularly comment on notable pricing moves. Search for each by name — a hit from any of these carries high signal weight because their audience is large and engaged.

```
WebSearch(query='"{Company}" pricing ("Kyle Poyar" OR "Patrick Campbell" OR "Lenny Rachitsky" OR "Madhavan Ramanujam" OR "Dan Hockenmaier")')
WebSearch(query='"{Company}" pricing ("Elena Verna" OR "Leah Tharin" OR "Emily Kramer" OR "Adam Fishman" OR "Casey Winters")')
WebSearch(query='"{Company}" pricing ("Tomasz Tunguz" OR "David Sacks" OR "Jason Lemkin" OR "Hiten Shah" OR "Andrew Chen")')
WebSearch(query='"{Company}" pricing ("Claire Suellentrop" OR "Asia Orangio" OR "Rob Snyder" OR "Jeff Epstein" OR "Naomi Ionita")')
```

**LinkedIn newsletter searches — these are the highest-quality practitioner analyses:**
```
WebSearch(query='"{Company}" pricing site:linkedin.com/newsletters')
WebSearch(query='"{Company}" pricing "Growth Unhinged" OR "Lenny\'s Newsletter" OR "The Product Compass" OR "Elena\'s Growth Scoop"')
```

### X / Twitter — real-time velocity and virality

X is the highest-velocity signal source — backlash, praise, and viral pricing drama surface here first, often within hours of an announcement.

**Broad post search:**
```
WebSearch(query='"{Company}" pricing site:x.com OR site:twitter.com')
WebSearch(query='"{Company}" "price increase" OR "too expensive" OR "pricing change" site:x.com')
WebSearch(query='"{Company}" pricing (founder OR SaaS OR "product-led" OR "per seat") site:x.com')
```

**Prominent SaaS voices on X — extended roster:**
```
WebSearch(query='"{Company}" pricing from:KylePoyar OR from:Patticus OR from:jasonlk OR from:hitenshah OR from:lennysan site:x.com')
WebSearch(query='"{Company}" pricing from:ElenaVerna OR from:LeahTharin OR from:EmilyKramer OR from:CaseyWinters site:x.com')
WebSearch(query='"{Company}" pricing from:ttunguz OR from:DavidSacks OR from:andrewchen OR from:fishmanaf site:x.com')
WebSearch(query='"{Company}" pricing from:bridgetkromhout OR from:aabordes OR from:AdamFishman site:x.com')
```

**Viral thread detection — pricing drama often goes viral on X:**
```
WebSearch(query='"{Company}" pricing thread OR "hot take" OR "unpopular opinion" site:x.com')
WebSearch(query='"{Company}" pricing site:x.com (VC OR "seed" OR "Series" OR "growth" OR "PLG")')
```

**Nitter fallback** — if direct WebSearch against x.com returns sparse results, try fetching via nitter (public Twitter mirror):
```
WebFetch(url="https://nitter.net/search?q={Company}+pricing&f=tweets")
WebFetch(url="https://nitter.net/search?q={Company}+%22price+increase%22&f=tweets")
```

If nitter is unreachable, skip silently and note "X coverage: not retrievable via current tooling."

### YouTube and podcasts — long-form expert analysis

YouTube and podcast episodes often contain the deepest analysis of pricing moves — 10-30 minute breakdowns by respected operators and analysts. These are underweighted by most competitive intelligence tools but carry extremely high conviction signal.

**YouTube:**
```
WebSearch(query='"{Company}" pricing review OR analysis OR breakdown site:youtube.com')
WebSearch(query='"{Company}" pricing strategy OR "price increase" OR "pricing change" site:youtube.com 2025 OR 2026')
WebSearch(query='"{Company}" pricing (SaaStr OR "Product Led" OR "Growth" OR "SaaS") site:youtube.com')
```

Known SaaS pricing YouTube channels to search directly:
```
WebSearch(query='"{Company}" pricing site:youtube.com/c/SaaStr OR site:youtube.com/@lennyspodcast OR site:youtube.com/@a16z')
WebSearch(query='"{Company}" pricing site:youtube.com/@ProductLedPodcast OR site:youtube.com/@growthhackers')
```

**Podcasts (via transcript search and show notes):**
```
WebSearch(query='"{Company}" pricing podcast transcript 2025 OR 2026')
WebSearch(query='"{Company}" pricing site:podcasts.apple.com OR site:open.spotify.com')
WebSearch(query='"{Company}" pricing ("Lenny\'s Podcast" OR "SaaStr" OR "The Twenty Minute VC" OR "Growth Stage" OR "Pricing Page Teardown")')
```

**Podcast transcript search engines:**
```
WebFetch(url="https://www.listennotes.com/search/?q=%22{Company}%22+pricing&type=episode&sort_by_date=1")
```

When a relevant video or episode is found, note: title, host/channel, view/listen count (if visible), and date. A YouTube video with 50K+ views discussing the pricing change is a strong signal of market attention.

### Substack and independent blogs — practitioner deep-dives

Substacks are where the most nuanced pricing analyses live — longer than LinkedIn posts, more analytical than X threads, and written by people with skin in the game.

**Substack search:**
```
WebSearch(query='"{Company}" pricing site:substack.com')
WebSearch(query='"{Company}" "pricing strategy" OR "price increase" OR "pricing model" site:substack.com 2025 OR 2026')
```

**Known pricing and growth Substacks to search directly:**
```
WebFetch(url="https://kylepoyar.substack.com/search?query={Company}")
WebFetch(url="https://www.lennysnewsletter.com/search?q={Company}+pricing")
WebSearch(query='"{Company}" pricing site:every.to OR site:generalist.com OR site:platformer.news')
WebSearch(query='"{Company}" pricing site:stratechery.com OR site:readthegeneralist.com')
```

**Independent SaaS blogs:**
```
WebSearch(query='"{Company}" pricing site:saastr.com OR site:openviewpartners.com OR site:a16z.com OR site:bvp.com')
WebSearch(query='"{Company}" pricing site:tomtunguz.com OR site:bothsidesofthetable.com OR site:avc.com')
WebSearch(query='"{Company}" pricing site:productled.com OR site:priceintelligently.com OR site:reforge.com')
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

### 2a: Theme classification

**Tone** — Overall positive / mixed / negative / no signal

**Key themes** (pick whichever apply):
- "Price shock" — customers surprised by increase magnitude
- "Value questioned" — sentiment that pricing no longer matches value delivered
- "Switching intent" — mentions of evaluating alternatives post-change
- "Acceptance" — change absorbed with minimal friction
- "Praise" — customers calling pricing fair or generous
- "Competitive comparison" — users publicly comparing this company's pricing to alternatives (high signal for monday.com positioning)
- "Feature-price disconnect" — users citing specific features that don't justify the price tier they're gated behind
- "No reaction" — change happened quietly, no detectable public response

**Timing signal** — When did the sentiment spike? Immediately post-change, or delayed (delayed often means churn is still materializing)?

### 2b: Conviction scoring

Every sentiment finding gets a conviction score (1–5) based on three dimensions. This replaces the subjective "Strong / Moderate / Weak" with a repeatable rubric.

#### Volume score (0–2 points)

How much discussion exists across all sources?

| Volume | Score |
|--------|-------|
| 0–2 mentions across all sources | 0 |
| 3–8 mentions across 2+ sources | 1 |
| 9+ mentions across 3+ sources | 2 |

#### Engagement depth score (0–2 points)

How engaged are people with the discussion?

| Engagement signal | Score |
|-------------------|-------|
| Low engagement: <10 upvotes/likes, <5 comments, <1K views on videos | 0 |
| Medium engagement: 10–100 upvotes/likes, 5–30 comments, 1K–20K views | 1 |
| High engagement: 100+ upvotes/likes, 30+ comments, 20K+ views, or viral thread (100+ retweets) | 2 |

#### Source authority score (0–1 point)

Did high-authority voices weigh in?

| Authority signal | Score |
|------------------|-------|
| Only anonymous users, unknown reviewers, or low-follower accounts | 0 |
| Named expert (from the thought leader list), verified buyer review (enterprise title), publication coverage (any tier), or podcast/YouTube with 10K+ followers | 1 |

#### Total conviction score

| Score | Label | Interpretation |
|-------|-------|---------------|
| 0–1 | **Anecdotal** | Sparse signal, insufficient for strategic conclusions. Note the finding but flag low confidence. |
| 2–3 | **Directional** | Enough signal to form a hypothesis. Multiple sources agree but sample is limited. |
| 4–5 | **High conviction** | Strong cross-source convergence. Multiple audiences, high engagement, expert validation. Treat as reliable market signal. |

### 2c: Engagement-weighted quote selection

Do not simply list the first quotes found. Rank quotes by engagement and authority, then select the top 4–6:

**Quote ranking criteria (in priority order):**
1. Expert attribution — named thought leader, verified enterprise buyer, or publication author
2. Engagement — highest upvotes, likes, comments, or retweets
3. Specificity — quotes with exact numbers, plan names, or dollar amounts outrank vague complaints
4. Recency — newer quotes outrank older ones for the same sentiment
5. Source diversity — ensure at least 3 different source types are represented in the final quote selection

### 2d: Source weighting by signal type

| Source | Signal type | Weight |
|--------|------------|--------|
| X / Twitter | Real-time velocity — first to surface backlash or praise, often within hours | Highest for recency |
| Reddit / HN | Depth and authenticity — longer-form user reactions, less PR-filtered | Highest for candor |
| LinkedIn | Practitioner reaction — founders, PMs, growth leads giving thoughtful takes | High for operator sentiment |
| G2 / Capterra / PeerSpot | End-user and buyer reaction — structured, often tied to renewal decisions | High for buyer sentiment |
| YouTube / Podcasts | Long-form expert analysis — deepest reasoning, highest conviction per piece | Highest for depth of analysis |
| Substacks / blogs | Practitioner deep-dives — nuanced and analytical | High for strategic interpretation |
| Publications (Tier 1) | Mainstream business signal — coverage means the move was large enough to matter broadly | Signals magnitude |
| Publications (Tier 2) | SaaS community signal — practitioner press is covering it as an industry pattern | Signals strategic interest |
| Publications (Tier 3) | Analyst / enterprise signal — procurement teams and analysts are tracking it | Signals enterprise sales impact |
| Earnings calls | Official management framing — how the company explains the pricing decision | Signals intent and confidence |

When sources conflict (e.g., G2 reviews are negative but X shows acceptance), surface both readings and note the discrepancy — the audience segments are different. Source conflicts are themselves a finding.

---

## Step 3: Output format

```
## {Company} — pricing sentiment

**Overall tone:** {Positive / Mixed / Negative / No signal}
**Conviction score:** {N}/5 ({label}) — Volume: {N}/2 · Engagement: {N}/2 · Authority: {N}/1
**Sources checked:** Reddit · HN · G2 · Capterra · PeerSpot · LinkedIn · X/Twitter · YouTube · Podcasts · Substacks · Publications · Earnings calls
**Sources with signal:** {list only sources that returned relevant results}

### Key themes
- {Theme 1}: {description with evidence and engagement metrics}
- {Theme 2}: {description with evidence and engagement metrics}

### Expert and influencer takes

{This section surfaces named experts and high-authority voices who weighed in. Only include if authority score = 1.}

| Expert | Platform | Take | Engagement |
|--------|----------|------|------------|
| {Name} | LinkedIn / X / YouTube / Substack | {1-sentence summary of their position} | {followers / likes / views} |
| {Name} | {platform} | {summary} | {engagement} |

{If no expert/influencer coverage: omit this section entirely.}

### Top quotes (engagement-weighted)

> "{quote}" — [Reddit/{subreddit}]({url}), {date} · {upvotes} upvotes, {comments} comments
> "{quote}" — [G2 review]({url}), {date} · {reviewer title at company}
> "{quote}" — [LinkedIn / {author name}]({url}), {date} · {reactions} reactions
> "{quote}" — [X / @{handle}]({url}), {date} · {retweets} RTs, {likes} likes
> "{quote}" — [{Publication name}]({url}), {date}
> "{quote}" — [YouTube / {channel}]({url}), {date} · {views} views

### Long-form analysis

{If YouTube, podcast, or Substack deep-dives were found, summarize the key arguments here — these carry the highest conviction per piece.}

- **[{Title}]({url})** by {author/channel} ({platform}, {date}, {views/listens}): {2-3 sentence summary of their analysis and conclusion}
- **[{Title}]({url})** by {author/channel}: {summary}

{If none found: omit this section.}

### What this means
{2-3 sentences: strategic interpretation. Is the market absorbing this? Are customers at risk of churning? Does the reaction suggest the pricing change was well-executed or clumsy?}

**Conviction assessment:** {Based on the score, how much should monday.com weight this signal? A score of 4-5 means "act on this." A score of 2-3 means "monitor but don't overreact." A score of 0-1 means "insufficient data to draw conclusions."}

### Source map

**Community:** [{Reddit/HN thread title}]({url}) · [{G2 review}]({url})
**LinkedIn:** [{Author name — article/post title}]({url})
**X/Twitter:** [{@handle — post summary}]({url})
**YouTube:** [{Channel — video title}]({url})
**Podcasts:** [{Show — episode title}]({url})
**Substacks/Blogs:** [{Author — post title}]({url})
**Publications:** [{Publication — article title}]({url}) · [{Newsletter — post title}]({url})
**Earnings / analyst:** [{Company earnings call Q{N} {year}}]({url})
```

---

## Step 4: Log to monday

After delivering sentiment output, log to the Pricing Intelligence board per [monday-logging.md](monday-logging.md):

- Item name: `{Company} — Sentiment`
- Summary: overall tone + conviction score + key theme in 1–2 sentences
- PricingSaaS link: `https://pricingsaas.com/companies/{slug}`
- Workflow: `sentiment-research`

---

## Important notes

- Not all companies will have detectable public sentiment — niche B2B tools often have near-zero public discussion. Say so explicitly rather than filling with noise. A conviction score of 0–1 makes this clear.
- Weight Reddit and HN highest for candor and depth. Weight X/Twitter highest for recency and velocity. Weight LinkedIn for practitioner/operator reaction. Weight G2 and Capterra for structured buyer reaction. Weight YouTube and podcasts highest for depth of analysis per piece.
- A "no reaction" finding is itself useful signal — it means the change was either too small, too gradual, or targeted a segment that doesn't discuss pricing publicly.
- Date-filter all searches to the last 6 months unless the user asks for historical sentiment.
- X/Twitter and nitter may be unreachable in some environments — if both fail, note it once and move on. Do not block the workflow.
- LinkedIn pulse articles are often the highest-quality practitioner takes — prioritize fetching full text when a relevant article URL is found.
- YouTube videos and podcast episodes with 10K+ views/listens carry more conviction weight than any single social media post — prioritize surfacing these when found.
- The conviction score is designed to prevent over-indexing on a single angry Reddit thread or one positive tweet. Require cross-source convergence before labeling anything as "high conviction."
- When selecting quotes, always include engagement metrics (upvotes, likes, views, comments) alongside the quote text — this lets the reader assess credibility without trusting the AI's judgment alone.
