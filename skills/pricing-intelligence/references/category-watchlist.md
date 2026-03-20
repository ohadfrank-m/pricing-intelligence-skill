# Category watchlist workflow

Track an entire competitive category as a bundle — add all relevant companies to the watchlist in one command, and get a category-level view whenever pricing moves in the space.

## When to run

- User asks: "track the {category} pricing landscape", "add all {category} competitors to my watchlist", "monitor pricing in {category}", "set up a watchlist for {category} tools", "watch competitors in {space}"
- After a company research session where the user wants ongoing monitoring of the full peer group
- When building a competitive pricing program across a category rather than tracking individual companies ad hoc

---

## Step 1: Identify companies in the category

Run both in parallel:

```
search_companies_advanced(filters={"category": "{category name}"})
WebSearch(query='"{category}" SaaS tools list competitors 2026 site:g2.com OR site:capterra.com OR site:getapp.com')
```

From the results, build a deduplicated list of the top 8–12 companies in the category. Prioritize by:
1. Direct competitors (same buyer, same use case)
2. Employee count as a proxy for market relevance (larger = more signal value)
3. PLG companies (they change pricing more frequently — higher monitoring value)

Present the list to the user before adding to watchlist:

> "Here are the {N} companies I'd add to your {category} watchlist. Confirm or remove any before I add them:"
>
> | Company | Slug | Why included |
> |---------|------|--------------|
> | {Company} | `{slug}` | {1-line reason} |
> | ... | | |

Wait for confirmation. Adjust if the user removes or adds companies.

---

## Step 2: Resolve slugs for all confirmed companies

For any company not already resolved from Step 1:

```
search_companies(query="{company name}")
```

Run all unresolved lookups in parallel.

---

## Step 3: Add the full category to watchlist

```
add_to_watchlist(slugs=["{slug1}", "{slug2}", "{slug3}", ...])
```

Single call with all slugs as an array. Free.

---

## Step 4: Pull a snapshot baseline

Run for all confirmed companies in parallel:

```
get_company_details(slug="{slug}")
get_company_history(slug="{slug}", discovery_only=true)
```

Build a category pricing table from the details responses:

| Company | Free tier | Entry paid | Mid tier | Enterprise | Pricing model |
|---------|-----------|------------|----------|------------|---------------|
| {Company A} | {Yes/$0} | ${x}/mo | ${x}/mo | Custom | Per user |
| {Company B} | No | ${x}/mo | — | Custom | Per seat |
| ... | | | | | |

Note which companies have the most active change history (highest diff count) — these are the highest-monitoring-value companies in the category.

---

## Step 5: Output the category watchlist summary

```markdown
# {Category} pricing watchlist — set up {date}

**{N} companies now tracked** | Weekly digest will surface any changes

---

## Category pricing snapshot

{Table from Step 4}

---

## Monitoring priority

**High signal** (active pricing history — watch closely):
- {Company}: {N} diffs tracked since {date}, most recent: {change type} in {period}

**Medium signal** (moderate activity):
- {Company}: {N} diffs

**Low signal / new tracking** (recently added — baseline being established):
- {Company}: Just added, no history yet

---

## Key patterns in this category

{2–3 sentences on what the snapshot reveals about the category's pricing dynamics — e.g., "The category is bifurcated: PLG-first tools (Linear, Height) lead with generous free tiers and per-seat paid; legacy players (Jira) are migrating toward usage-based AI add-ons. Entry paid tier ranges from $8 to $25/seat — a wide spread that signals different buyer archetypes are being targeted."}

---

## Next steps

- Run the weekly digest any time: "give me my pricing digest" or "what changed in {category} pricing"
- Add more companies: "add {Company} to my watchlist"
- Remove a company: "remove {Company} from my watchlist"
- Full research on a specific company: "deep-dive on {Company} pricing"
```

---

## Step 6: Log to monday

Follow [monday-logging.md](monday-logging.md):

- Item name: `{Category} — Category Watchlist`
- Change Type: `Company Research` (closest available)
- Summary: "{N} companies added to the {Category} category watchlist. Entry paid range: ${x}–${y}/seat. {Top observation from key patterns.}"
- PricingSaaS link: blank (category-level, no single diff URL)
- Workflow: `category-watchlist`

---

## Maintaining the category watchlist

**Adding companies later:**
> "Add {Company} to my {category} watchlist" → `add_to_watchlist(slugs=["{slug}"])`

**Removing companies:**
> "Remove {Company} from my watchlist" → `remove_from_watchlist(slugs=["{slug}"])`

**Getting a category refresh (without full weekly digest):**
> "What's the current {category} pricing landscape?" → Re-run Step 4 for all watchlist companies and regenerate the snapshot table.

---

## Pre-defined category bundles

Use these as starting points when the user names a category. Adjust based on what `search_companies_advanced` returns.

| Category | Default companies to include |
|----------|------------------------------|
| Dev project management | Linear, Jira, Asana, Shortcut, Height, Plane, ClickUp, Notion |
| CRM | HubSpot, Salesforce, Pipedrive, Attio, Close, Monday CRM |
| Customer support | Zendesk, Intercom, Front, Help Scout, Freshdesk, Kustomer |
| Data / BI | Tableau, Looker, Metabase, Mode, Sigma, ThoughtSpot |
| Work management | monday.com, Asana, ClickUp, Wrike, Smartsheet, Teamwork |
| AI writing / content | Notion AI, Jasper, Copy.ai, Writer, Grammarly Business |
| Sales engagement | Outreach, Salesloft, Apollo, Instantly, Lemlist |

For monday.com-specific competitive research, default to the **Work management** bundle.
