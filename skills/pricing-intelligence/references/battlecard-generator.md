# Battlecard generator workflow

Produce a sales- and product-ready pricing battlecard comparing your company against a specific competitor. Output is grounded in live PricingSaaS data + public intelligence, not opinion.

## When to run

- User asks: "build a pricing battlecard for X", "we're losing deals to X on price", "create a competitive battlecard vs. X", "how do we handle X pricing objections"
- Before a product pricing review where a specific competitor is referenced
- After a monitoring alert flags a competitor pricing change

---

## Step 1: Establish your company's pricing

Ask the user upfront (do not skip):

> "To build the battlecard, I need your current pricing. Share your plan names, prices, and key included features — or point me to your pricing page and I'll pull it."

If the user provides a URL:
```
WebFetch(url="{your pricing page URL}")
```

If the user provides a monday.com internal page or doc, read it directly.

Extract:
- Plan names and prices (monthly + annual)
- Pricing metric (per seat, usage, flat)
- Key features per tier
- Free tier / trial details
- Add-ons

Store as "Your Company" data for the comparison.

---

## Step 2: Pull competitor data

### Check knowledge base first

Before fetching fresh data, check the knowledge base for an existing entry on this competitor:

```
Read(path="Claude-Workspace/pricing-intelligence/knowledge-base.json")
```

If the competitor has a KB entry with `last_researched` within the last 7 days, reuse `current_pricing`, `pricing_history_summary`, and `strategic_notes` from the KB instead of re-fetching. Still run the live pricing page fetch and objection search (these are fast and may surface new data).

If the KB entry is older than 7 days or doesn't exist, run the full fetch:

```
search_companies(query="{competitor name}")
get_company_details(slug="{slug}")
get_company_history(slug="{slug}", discovery_only=true)
WebFetch(url="https://{competitor domain}/pricing")
WebSearch(query='"{Competitor}" pricing objections "too expensive" OR "vs {your company}" site:reddit.com OR site:g2.com')
```

Extract from PricingSaaS:
- All plan names, prices, and metrics
- Add-ons and their costs
- Any recent pricing changes (from history preview)

Extract from live page fetch:
- CTA strategy (self-serve vs. sales)
- Feature gating transparency
- Any promotion or trial offer visible

---

## Step 2b: Run negotiation intelligence

Run [negotiation-intelligence.md](negotiation-intelligence.md) Steps 1–4 for the competitor in parallel with Step 2 above. This populates the battlecard's negotiation section with real deal economics rather than guesses.

The negotiation workflow returns:
- Median contract value and typical discount range (Vendr)
- What buyers actually pay (G2 + community disclosures)
- Which pressure points move deals (timing, competitive leverage, volume)
- What is and isn't negotiable
- Hidden costs to surface before signing

Store this output as "Competitor negotiation intel" — it feeds directly into the battlecard's "How to handle 'they're cheaper'" and "Negotiation intelligence" sections below.

---

## Step 3: Match plans by segment

Map your plans to the competitor's plans by target buyer segment, not by name. Use price range + feature depth as the matching signal.

| Segment | Your Plan | Your Price | Competitor Plan | Competitor Price |
|---------|-----------|------------|-----------------|-----------------|
| SMB / self-serve | | | | |
| Mid-market | | | | |
| Enterprise | | | | |

If their plan structure doesn't map cleanly (e.g., they use usage-based, you use per-seat), note the structural difference and explain how to frame it to a buyer.

---

## Step 4: Build the battlecard

Output the following sections in order. This is the document that goes directly to sales/product — write it for that audience (direct, no hedging, no academic language).

---

```markdown
# Pricing battlecard: {Your Company} vs. {Competitor}
Generated: {date} | Source: PricingSaaS + live page data

---

## Quick verdict

{2 sentences: where you win on pricing, where you lose, and the single most important framing point}

---

## Plan comparison

| Segment | {Your Company} | {Competitor} | Delta | Who wins |
|---------|---------------|--------------|-------|----------|
| SMB | ${x}/mo | ${x}/mo | {+/- %} | {Us / Them / Depends} |
| Mid-market | ${x}/mo | ${x}/mo | | |
| Enterprise | ${x}/mo | ${x}/mo | | |

**Pricing model comparison**

| Dimension | {Your Company} | {Competitor} |
|-----------|---------------|--------------|
| Pricing metric | {per seat / usage / flat} | |
| Free tier | {Yes/No, limits} | |
| Trial | {length, CC required?} | |
| Annual discount | {%} | |
| Add-on complexity | {Low/Medium/High} | |

---

## Where we win

{List 3–5 specific, data-grounded points where your pricing is stronger. Be precise — "we include X at $Y per seat; they charge an additional $Z add-on for the same capability."}

- **{Point 1}:** {Specific advantage with numbers}
- **{Point 2}:** {Specific advantage with numbers}
- **{Point 3}:** {Specific advantage with numbers}

---

## Where they win

{Honest assessment — 2–3 points where the competitor has a genuine pricing advantage. Sales needs to know this to handle it, not be surprised by it.}

- **{Point 1}:** {Specific disadvantage with numbers}
- **{Point 2}:** {Specific disadvantage with numbers}

---

## How to handle "they're cheaper"

{The buyer who says this is usually comparing list prices without accounting for true cost. Response framework:}

**Step 1 — Reframe total cost**
{If competitor has add-on stacking or hidden fees: "Their {plan} at ${x} sounds lower, but a typical {segment} customer needs {add-on A} at ${y} and {add-on B} at ${z} — putting the real cost at ${total}, vs. our all-in ${your price}."}

**Step 2 — Anchor on value metric**
{If you have a better pricing model fit for the buyer: "They charge per {their metric}. You {buyer use case description}, which means..."}

**Step 3 — Highlight switching risk**
{If the competitor has a history of price increases or add-on expansion: "Their pricing changed {X times / by Y%} in the last {period}. {Reference specific change if available from PricingSaaS history.}"}

**Step 4 — Negotiate anchor**
{Based on public negotiation intelligence: "Their list price is ${x}, but typical deals close at {discount range}% off. Our pricing is {transparent/all-in/already discounted} — apples to apples, here's what the math looks like."}

---

## How to handle "we already use {Competitor}"

{If this is an existing customer scenario — they're comparing your expansion or an existing tool.}

{Migration cost framing, switching cost arguments, integration/workflow stickiness}

---

## Recent competitor pricing moves

{From PricingSaaS history + WebSearch: any changes in the last 12 months, direction of travel, and what it suggests about their pricing trajectory}

{If no tracked changes: "No pricing changes detected in PricingSaaS's tracking window. Last Wayback Machine snapshot shows pricing has been stable since {date}."}

---

## Negotiation intelligence

*Populated from [negotiation-intelligence.md](negotiation-intelligence.md) — Steps 1–4 run in Step 2b above.*

| Metric | Value |
|--------|-------|
| Median contract value | ${amount} (source: Vendr) |
| Typical discount off list | {range} |
| Best documented discount | {amount or %} |
| Most common contract length | {1-year / 2-year / mixed} |

**What moves the deal:**
- {Pressure point 1 — with source}
- {Pressure point 2}
- {Timing: best quarter to negotiate}

**What buyers actually report paying:**
- "{Quote or paraphrase}" — [{source}]({url})

**What they defend (don't expect movement):**
- {Line item 1}

**Hidden costs to surface before signing:**
- {Cost 1 — e.g., onboarding fee, true-up at renewal}

**Negotiation script:**
- *Opening:* "{Suggested opener using competitor mention or timing}"
- *If they won't move on seat price:* "{Alternative concession to ask for}"

*Sources: Vendr · G2 · Reddit · Community disclosures*

---

## Quick reference card (for live deals)

| Question | Answer |
|----------|--------|
| Their cheapest plan | ${x}/mo ({plan name}) |
| Their most common deal | {segment} at ${x}/mo |
| Their biggest weakness | {single phrase} |
| Our strongest counter | {single phrase} |
| Don't let them say... | "{objection}" |
| Our response | "{counter}" |
```

---

## Step 5: Offer follow-ups

After delivering the battlecard, offer:

1. "Want me to run a full pricing page teardown on {Competitor} to understand how they're presenting this to buyers?"
2. "Want me to add {Competitor} to your watchlist so you get alerted when they change pricing?"
3. "Want me to pull customer sentiment on {Competitor}'s pricing — what their customers say publicly on Reddit and G2?"

---

## Step 6: Log to monday

Follow [monday-logging.md](monday-logging.md):

- Item name: `{Competitor} — Battlecard`
- Summary: Quick verdict — where you win, where you lose, in 1–2 sentences
- PricingSaaS link: `https://pricingsaas.com/companies/{slug}`
- Workflow: `battlecard-generator`

Create a monday doc with the full battlecard content attached to the item. The battlecard should be usable directly from the monday doc without needing to return to chat.
