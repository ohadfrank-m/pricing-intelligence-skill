# Freemium / trial change tracker

Detect when a company changes their free tier limits, trial structure, or freemium-to-paid conversion mechanics. These moves are among the highest-signal pricing events in PLG companies — they directly affect acquisition volume, time-to-value, and conversion rates. They often don't appear as "price changes" in monitoring tools because the list price doesn't change, yet the monetization impact can be enormous.

**Examples of what this detects:**
- Free tier block/seat/feature limit cut (e.g., Notion: unlimited blocks → 1,000 for teams)
- Trial length change (30 days → 14 days, or gated → ungated)
- Credit card requirement added or removed from trial signup
- Reverse trial introduced (full paid features → free tier at trial end)
- Freemium plan removed entirely
- Feature moved from free tier to paid only
- New free tier added to a previously paid-only product

## When to run

- User asks: "has X changed their free tier", "did X change their trial", "is X restricting their freemium", "track freemium changes in {category}"
- Automatically as a pattern detector inside monitoring — any PricingSaaS diff containing "Capacity Decreased", "Feature Removed", or "Feature Added" on a plan named Free/Starter/Trial/Basic should trigger this workflow
- As an add-on to company-research when the company has a free tier

---

## Step 1: Establish the current free/trial structure

```
get_company_details(slug="{slug}")
```

From the response, extract:
- Whether a free tier exists (plan with $0 or "free" label)
- Freemium limits: seats, blocks, storage, API calls, automations, projects — whatever the value metric is
- Trial: length in days, credit card required (yes/no), features included vs. paid
- CTAs: "Start free", "Start free trial", "Try for free" — subtle wording differences matter

Also fetch the live pricing page to capture current language:
```
WebFetch(url="https://{domain}/pricing")
```

Look specifically for:
- How limits are communicated (prominently vs. buried in feature table)
- Whether the free tier is listed alongside paid plans or de-emphasized
- Any "upgrade" triggers or nudge copy visible on the page
- Trial-related messaging: "No credit card required", "Cancel anytime", countdown language

---

## Step 2: Check PricingSaaS history for free tier changes

```
get_company_history(slug="{slug}", discovery_only=true)
```

Flag any period containing these change types — they are the strongest indicators of free/trial changes:

| PricingSaaS change type | What it likely means |
|------------------------|----------------------|
| `Capacity Decreased` | Free tier limit cut (fewer seats, less storage, lower API quota) |
| `Capacity Increased` | Free tier became more generous — acquisition push |
| `Feature Removed` | Feature pulled from free tier, now paid only |
| `Feature Added` | New feature added to free tier — could be a retention/acquisition move |
| `Plan Removed` | Free tier eliminated entirely |
| `Plan Added` | New free tier introduced — major GTM shift |
| `Pricing Metric Changed` | How they count/meter the free tier changed |
| `Addon Changed` | Trial or freemium add-on structure changed |

If any of these appear in history, offer to pull the full diff (1 credit) to get the exact change details.

---

## Step 3: Enrich with public sources

Run all in parallel:

### Changelog check
```
WebSearch(query='"{Company}" free plan OR free tier OR trial change 2025 OR 2026')
WebFetch(url="https://{domain}/changelog")
```

Look for any announcement of limit changes, trial restructuring, or freemium policy updates.

### Community reaction check
```
WebSearch(query='"{Company}" "free plan" OR "free tier" changed OR limited OR removed site:reddit.com OR site:news.ycombinator.com 2025 OR 2026')
```

Community discussions often surface free tier changes days or weeks before they appear in official changelogs. User frustration about losing free access is a high-signal indicator.

### Wayback Machine — before/after the change
```
WebFetch(url="https://archive.org/wayback/available?url={domain}/pricing&timestamp={YYYYMMDD-of-change}")
WebFetch(url="https://web.archive.org/web/{snapshot-timestamp}/{domain}/pricing")
```

Fetch a snapshot from just before and just after the detected change period to show the exact limit values that changed.

---

## Step 4: Classify the move

Once you have the current structure and any detected changes, classify the strategic intent:

| Move type | Description | What it signals |
|-----------|-------------|-----------------|
| **Limit cut** | Free tier becomes less useful (fewer seats, less storage, fewer features) | Monetizing existing free base; growth has slowed or CAC is too high |
| **Limit expansion** | Free tier becomes more generous | Acquisition push; trying to grow top of funnel or compete with a freemium entrant |
| **Trial shortening** | Trial length reduced (e.g., 30 → 14 days) | Accelerating conversion urgency; product may not be delivering value fast enough |
| **Trial lengthening** | Trial length extended | Improving activation; reducing trial-to-paid friction |
| **CC requirement added** | Credit card now required to start trial | Filtering for higher-intent users; reducing support burden from low-quality trials |
| **CC requirement removed** | No longer requires credit card | Reducing signup friction; competing on ease of access |
| **Reverse trial introduced** | All users start with full paid features, then drop to free | Advanced PLG move; betting on habit formation during trial |
| **Free tier removed** | Product becomes fully paid | Freemium experiment ended; converting remaining free users to paid |
| **New free tier added** | Paid-only product introduces freemium | Major strategic pivot; entering PLG motion or defending against a freemium entrant |

---

## Step 5: Output format

```
## {Company} — freemium / trial change tracker

**Current structure ({date}):**
- Free tier: {Yes/No — plan name, key limits}
- Trial: {length} days | Credit card: {required/not required} | Features: {full paid / limited}
- Self-serve CTA: "{exact CTA text}"

---

### Changes detected

#### {Period} — {Change type from PricingSaaS}

**Before:** {what the free tier / trial looked like}
**After:** {what it looks like now}
**Source:** [PricingSaaS diff](https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}) · {changelog link if found}

{If community reaction found:}
**Community reaction:** {tone, key quotes, source links}

---

### Classification

**Move type:** {one of the types from Step 4}
**Strategic signal:** {2–3 sentences — why they made this move and what it implies for their growth motion, conversion rates, and competitive positioning}

### What to watch next

{Specific follow-on prediction: "If this is a limit cut to monetize the free base, expect a follow-up price increase on the Starter/Plus tier within 2 quarters." or "A reverse trial introduction often precedes a paid plan restructure — watch for plan changes in the next 60–90 days."}
```

---

## Step 5b: Free-to-paid conversion timeline and competitive benchmarking

Track how each company's free/trial structure has evolved over time and benchmark against competitors. This surfaces the trajectory — is the industry moving toward more restrictive free tiers, shorter trials, or more generous freemium?

### Timeline construction

Using data from the knowledge base and PricingSaaS history, build a chronological view of all free/trial changes for this company:

```
Read(path="Claude-Workspace/pricing-intelligence/knowledge-base.json")
```

Look up the company's `pricing_history_summary` and `ab_test_status` entries. Combine with PricingSaaS history (Step 2) to build the timeline.

```
### Free/trial evolution timeline — {Company}

| Date | Change | Before | After | Impact direction |
|------|--------|--------|-------|-----------------|
| {date} | {change type} | {previous state} | {new state} | {↑ more generous / ↓ more restrictive / → neutral} |
| {date} | {change type} | {previous} | {new} | {direction} |

**Overall trajectory:** {More restrictive / More generous / Oscillating / Stable}
**Trajectory signal:** {1 sentence interpretation — e.g., "Consistent tightening of free tier over 18 months signals transition from growth-mode to monetization-mode."}
```

### Cross-company benchmarking

When the knowledge base contains free/trial data for 3+ companies in the same category, generate a competitive benchmark:

```
### Free/trial competitive benchmark — {Category}

| Dimension | {Company A} | {Company B} | {Company C} | monday.com |
|-----------|------------|------------|------------|------------|
| Free tier exists | {Yes/No} | {Yes/No} | {Yes/No} | Yes |
| Free tier seats/users | {limit} | {limit} | {limit} | {limit} |
| Key free tier limits | {description} | {description} | {description} | {description} |
| Trial length | {days} | {days} | {days} | {days} |
| CC required | {Yes/No} | {Yes/No} | {Yes/No} | No |
| Reverse trial | {Yes/No} | {Yes/No} | {Yes/No} | {Yes/No} |
| Direction (last 12mo) | {more/less restrictive} | {direction} | {direction} | {direction} |

**Category pattern:** {1-2 sentences — "The work management category is converging on 14-day trials with CC required. monday.com's 14-day no-CC trial is now in the minority, which could be a competitive advantage or a conversion rate drag."}

**monday.com positioning:** {1-2 sentences — where monday.com sits vs. the benchmark, and whether the competitive landscape suggests any changes to monday.com's own free/trial structure.}
```

### Persist timeline data

After generating the timeline and benchmark, update the knowledge base entry for this company with the latest free/trial snapshot:

```json
{
  "freemium_trial": {
    "last_checked": "ISO date",
    "has_free_tier": true,
    "free_tier_limits": {"seats": 2, "boards": 3},
    "trial_length_days": 14,
    "cc_required": false,
    "reverse_trial": false,
    "trajectory": "more_restrictive",
    "timeline_entries": [
      {"date": "ISO date", "change": "description", "direction": "restrictive"}
    ]
  }
}
```

---

## Step 6: Pattern detection across multiple companies

When called for a category or watchlist sweep, run `get_company_details` for all companies in parallel and build a comparison table:

```
## Freemium / trial landscape — {Category}

| Company | Free tier? | Trial length | CC required? | Last change | Direction |
|---------|-----------|--------------|--------------|-------------|-----------|
| {A} | Yes (limited) | — | No | {date} | ↓ Limit cut |
| {B} | No | 14 days | Yes | — | Stable |
| {C} | Yes (generous) | — | No | {date} | ↑ Limit expanded |
| {D} | No | 30 days | No | {date} | ↓ Shortened to 14d |
```

**Pattern assessment:** {Is the category moving toward more or less generous freemium? Any company making a counter-move against the trend?}

---

## Step 7: Log to monday

Follow [monday-logging.md](monday-logging.md):

- Item name: `{Company} — Freemium Change`
- Summary: Move type + specific change (e.g., "Free tier guest limit cut from unlimited → 10. Monetizing existing free base.")
- PricingSaaS link: `https://pricingsaas.com/pulse/companies/{slug}/diffs/{period}` if a diff was pulled
- Workflow: `freemium-trial-tracker`

Only log when an actual change is confirmed — not when the current structure is simply documented with no change detected.
