# Weekly pricing digest workflow

A single command produces a structured pricing brief covering everything that moved in the past 7 days across your watchlist and the broader market. Replaces manual monitoring that would otherwise take 30–60 minutes per week.

## When to run

- User asks: "run my weekly pricing digest", "what changed in pricing this week", "give me my pricing brief", "weekly pricing update"
- Can be run any day — "this week" always means the last 7 days from today

---

## Step 1: Pull watchlist changes

```
get_pricing_news()
```

Free. Returns recent changes across all tracked companies. Filter the output to:
1. Companies explicitly on the watchlist (use `get_watchlist()` if unsure which companies are tracked)
2. Any other changes that appear significant (large companies, unusual change types)

Note the full raw output — you'll need it for classification in Step 3.

---

## Step 2: Enrich with web signals

Run all in parallel — these catch things the MCP misses (announcements, soft launches, community reactions):

```
WebSearch(query='SaaS pricing change OR "price increase" OR "new pricing" OR "pricing update" 2026 last week')
WebSearch(query='{watchlist company 1} OR {watchlist company 2} OR {watchlist company 3} pricing change site:reddit.com OR site:news.ycombinator.com last week')
WebSearch(query='SaaS pricing "announced" OR "launching" OR "effective" {current month} {year}')
```

**Required: A/B test detection for all watchlist companies.** This is free — no credits required. Run Steps 1 and 2 of [ab-test-detection.md](ab-test-detection.md) in parallel for every watchlist company. Do not skip this step regardless of credit balance.

```
# Fetch all pricing pages in parallel (reuse content if already fetched above):
WebFetch(url="https://{company1-domain}/pricing")
WebFetch(url="https://{company2-domain}/pricing")
WebFetch(url="https://{company3-domain}/pricing")
...

# BuiltWith lookups in parallel:
WebSearch(query='site:builtwith.com "{company1-domain}"')
WebSearch(query='site:builtwith.com "{company2-domain}"')
WebSearch(query='site:builtwith.com "{company3-domain}"')
...
```

Scan each page for A/B testing tool signatures (Optimizely, Statsig, VWO, LaunchDarkly, etc.). Any company with a detected tool gets flagged as "testing infrastructure active" in the digest.

---

## Step 3: Classify changes

For each change detected (from MCP + web signals), assign a category:

| Category | What it means | Examples |
|----------|--------------|---------|
| **Price move** | Actual price point changed | "$49 → $59/seat", "Free tier removed" |
| **Structure change** | Plan added, removed, or renamed | "New Starter tier added", "Pro renamed to Business" |
| **Add-on expansion** | New paid add-on launched | "AI Copilot add-on at $30/seat" |
| **Packaging shift** | Feature moved between tiers | "SSO moved from Enterprise to Professional" |
| **AI pricing signal** | New AI-related monetization move | "Per-resolution pricing for AI agents" |
| **Soft signal** | Hiring, changelog, community discussion | "Pricing Strategy Principal hire at Zendesk" |
| **Testing signal** | A/B testing tool detected in page source or BuiltWith | "Figma: Statsig detected on pricing page — experiment infrastructure active" |

---

## Step 3b: Get images for top 1–2 changes (free, before composing the doc)

Before writing the digest, run [visual-diff.md](visual-diff.md) Steps 1a–1d (Cloudinary probe) for the top 1–2 highest-signal watchlist changes. This is free and must always run. If the probe finds snapshots, embed the images in the digest doc. If no snapshots are found, omit the image section silently — no placeholder, no credit message.

---

## Step 4: Compose the digest

Output the digest in this exact structure. Keep it scannable — executives and PMs should be able to read this in 3 minutes.

```markdown
# Pricing digest — week of {Monday date} to {Sunday date}

**{N} changes detected** across {M} companies | Generated {today's date}

---

## Must-know this week

{1–3 highest-signal changes — the ones that genuinely matter. Lead with the "so what", not the "what". Example: "Zendesk moved QA and WFM into a single $75/agent add-on bundle — effective simplification of their stack story, and a potential price increase for customers who only bought one."}

---

## Changes by company

{For each company with a detected change — ordered by significance, most important first}

### {Company}
**Change:** {What specifically changed}  
**Direction:** {Price up / Price down / Structure change / Add-on / AI signal}  
**Impact:** {Who is affected and how — buyers, existing customers, or competitors}  
**Source:** [PricingSaaS]({diff URL if available}) · [{other source}]({url})

---

### {Company 2}
...

---

## Soft signals to watch

{Changes that aren't confirmed pricing moves but are worth tracking — job postings, changelog entries, community discussions}

- **{Company}:** {Signal} — {what it suggests and when to expect a move}
- **{Company}:** {Signal}

---

## Testing signals (A/B activity)

{Any companies where a testing tool was detected in the pricing page source or via BuiltWith}

- **{Company}:** {Tool name} detected on pricing page — experiment infrastructure active. {If experiment name visible: "Experiment: '{name}'"} Watch for a pricing page change or PricingSaaS diff within 2–4 weeks.

No new tests indicated. {if none detected across all watchlist companies}

---

## No changes detected

{List watchlist companies with no activity this week — confirmation that nothing moved, not just an omission}

{Company A} · {Company B} · {Company C}

---

## Recommended actions

{2–3 specific, actionable recommendations based on this week's findings. Not generic advice.}

1. {Action}: {Rationale tied to a specific change detected}
2. {Action}: {Rationale}
3. {Action}: {Rationale}

---

*Sources: PricingSaaS MCP · Wayback Machine CDX · WebSearch. Watchlist: {list of tracked companies}.*
```

---

## Step 5: Log to monday

Follow [monday-logging.md](monday-logging.md):

- Item name: `Weekly Digest — {week of date, e.g., "Mar 17–23 2026"}`
- Summary: Must-know summary from the digest — the 1–2 most significant changes
- PricingSaaS link: Link to the most significant change's diff URL, or leave blank if no diffs
- Workflow: `weekly-digest`

Create a monday doc with the full digest content attached to the item. Name it: `Pricing Digest — {week of date}`.

---

## Handling edge cases

**No changes detected at all:** Still create the digest. The "No changes" section confirms the watchlist was checked. A quiet week is itself useful signal — say so: "No changes this week. Market appears stable — this is unusual for this category and may indicate a batch move is building."

**Too many changes (5+):** Limit the main "Changes by company" section to the 5 most significant. Move lower-signal changes to a "Also this week" subsection.

**Watchlist is empty:** Ask: "Your watchlist appears empty. Which companies should I track for the weekly digest?" Then run [monitoring.md](monitoring.md) to add them before proceeding.

---

## Setting up a regular digest cadence

The weekly digest is designed to run on a fixed cadence with zero prompting friction. Use this setup when the user asks to "automate", "schedule", or "make this happen every week":

### Recommended cadence: Monday morning

Monday morning is the optimal trigger — it captures any changes from the prior week before the week's planning begins, and feeds directly into weekly syncs or exec reviews.

### Option 1: Slack reminder (lowest friction)

Set a recurring Slack reminder — share this with the user:

```
/remind me to "Run my weekly pricing digest" every Monday at 9:00am
```

When the reminder fires, open your AI assistant and say "run my weekly pricing digest" — the skill routes automatically.

### Option 2: Calendar event

Add a recurring calendar event every Monday at 9am titled "Run pricing digest". When it fires, open your AI assistant and trigger the digest.

### Option 3: monday.com recurring item

Create a recurring monday.com item to prompt the weekly run:

1. On the Pricing Intelligence board, create an item: `Weekly Digest — recurring`
2. Set recurrence: every Monday
3. Assign to self
4. When the item fires, trigger the digest in your AI assistant

### Option 4: Claude Cowork — fully automatic (no manual trigger needed)

If running in Claude Cowork, use `create_scheduled_task` to schedule the digest to run every Monday at 9am automatically — no manual trigger needed.

```
create_scheduled_task(
  name="Weekly pricing digest",
  schedule="every Monday at 9:00am",
  prompt="Run my weekly pricing digest"
)
```

This is the lowest-friction option — the digest runs and delivers results without the user needing to open a chat or type anything.

### What to tell the user after first digest

After the first digest runs, say:

> "To get this every week without prompting: set a Slack reminder — `/remind me to 'Run pricing digest' every Monday at 9am`. Takes 30 seconds to set up and replaces 30–60 minutes of manual monitoring."

### Digest distribution

After generating the digest, offer:

> "Want me to format this as a Slack message you can paste directly into your team channel?"

If yes, reformat the digest output as a compact Slack-native message:
- Replace markdown headers with bold text (`*Must-know this week*`)
- Collapse each company change to one line: `• {Company}: {change} — {impact}`
- Keep "Recommended actions" as 2–3 bullets
- Add at the top: `*Pricing digest — week of {date}* | {N} changes across {M} companies`
- Total length: under 20 lines so it reads clean in Slack without scrolling
