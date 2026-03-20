# Negotiation intelligence workflow

Surface the real deal economics for a SaaS company — what buyers actually pay vs. list price, which line items are negotiable, what pressure points move deals, and how to frame a competitive displacement. List prices are rarely what closes. This workflow closes that gap.

## When to run

- User asks: "what do people actually pay for X", "typical discount for X", "how to negotiate X pricing", "what's the real cost of X", "how much can we get off X's price"
- Automatically called at the end of battlecard-generator to populate the "Negotiation intelligence" section
- As a standalone lookup before a vendor renewal, competitive displacement, or pricing review
- When evaluating a competitor's true cost vs. your own in a deal

---

## Step 1: Pull Vendr market data

Vendr is the highest-quality source for actual SaaS deal economics — real contract data from thousands of buyer transactions.

```
WebFetch(url="https://www.vendr.com/marketplace/{company-slug}")
WebSearch(query='"{Company}" pricing vendr.com average cost discount')
```

From Vendr, extract:
- **Median contract value** (what the average buyer pays per year)
- **Typical discount range** (% off list)
- **Contract length norms** (1-year vs. multi-year prevalence)
- **Negotiated savings** (average $ saved per deal)
- **Price range** (floor to ceiling of real contracts)

Common Vendr URL patterns:
- `vendr.com/marketplace/notion`
- `vendr.com/marketplace/zendesk`
- `vendr.com/marketplace/hubspot`

If Vendr doesn't have a page for this company (common for smaller or newer companies), skip to Step 2.

---

## Step 2: Pull G2 and Capterra buyer intelligence

G2 and Capterra have pricing reviews where buyers disclose what they actually paid and what they negotiated.

```
WebSearch(query='"{Company}" pricing "what we paid" OR "negotiated" OR "discount" site:g2.com')
WebSearch(query='"{Company}" pricing review "contract" OR "annual" OR "per seat" site:capterra.com')
WebFetch(url="https://www.g2.com/products/{company-slug}/pricing")
```

Look for:
- Specific dollar amounts disclosed in reviews
- Comments about negotiability ("they came down 20% when we mentioned competitors")
- Pressure tactics that worked ("end of quarter saved us 15%")
- Line items that were waived ("they threw in onboarding for free")
- Hidden costs buyers discovered post-signature

---

## Step 3: Community intelligence (Reddit, HN, Slack communities)

Practitioners share real deal terms in public forums — especially when frustrated or proud of a good deal.

```
WebSearch(query='"{Company}" pricing negotiation discount "we got" OR "they offered" OR "came down" site:reddit.com')
WebSearch(query='"{Company}" "price increase" OR "renewal" OR "negotiate" site:reddit.com OR site:news.ycombinator.com 2025 OR 2026')
WebSearch(query='"{Company}" pricing "we pay" OR "our contract" OR "per seat" OR "annual cost" site:reddit.com')
```

Also search operations/procurement communities:
```
WebSearch(query='"{Company}" pricing negotiation site:reddit.com/r/sales OR site:reddit.com/r/procurement OR site:reddit.com/r/sysadmin')
```

Extract: disclosed contract values, successful negotiation tactics, what competitors were used as leverage, timing signals that created pressure.

---

## Step 4: Identify negotiation pressure points

Compile everything gathered into a structured pressure point map. For each point, note the source.

### Timing-based leverage

| Window | Why it works |
|--------|-------------|
| End of quarter (last 2 weeks of Q) | Sales reps have quota pressure; approval for larger discounts is easier |
| End of fiscal year | Maximum annual quota pressure; highest discount approval authority |
| Annual renewal window | Best time to renegotiate — moving cost is highest, but so is your leverage |
| Post-pricing-change | If they just raised prices, renewal is the moment to lock in old pricing |

### Competitive leverage

- Which competitors are commonly used as pricing leverage? (search: `"{Company}" vs "{Competitor}" pricing negotiation reddit`)
- Does mentioning a specific competitor reliably move the needle?
- Are there free alternatives or internal-build threats that create urgency?

### Volume and term leverage

- Multi-year commit: typical additional discount for 2-year vs. 1-year, 3-year vs. 2-year
- Seat volume: at what seat count do discounts typically jump? (e.g., enterprise tiers often have a hard break at 50, 100, 250 seats)
- Prepay discount: some companies offer additional % for annual upfront vs. invoiced

### What is typically negotiable vs. what isn't

| Usually negotiable | Usually not negotiable |
|-------------------|----------------------|
| Core seat/license price | AI add-on pricing (often defended) |
| Onboarding and professional services | Per-resolution / usage-based minimums |
| Number of included admin seats | Compliance / security add-ons |
| Support tier included | Data residency options |
| Contract length terms | SSO/SCIM (enterprise security features) |

---

## Step 5: Build the negotiation brief

```markdown
## {Company} — negotiation intelligence

**Source data:** Vendr · G2 · Reddit · Community forums
**Last updated:** {date}

---

### Real contract economics

| Metric | Value | Source |
|--------|-------|--------|
| Median annual contract | ${amount} | Vendr |
| Typical discount off list | {range, e.g., "12–22%"} | Vendr + G2 |
| Best documented discount | {amount or %}| {source} |
| Price range (floor to ceiling) | ${low}–${high}/year | Vendr |
| Most common contract length | {1-year / 2-year / mixed} | Vendr |

{If Vendr has no data: "Vendr does not have market data for {Company}. Estimates based on G2 reviews and community disclosures."}

---

### What buyers actually pay

{2–4 specific data points with sources. Concrete is better than general.}

- "{Quote or paraphrase from G2/Reddit/Vendr}" — [Source]({url})
- "We paid ${amount} for {N} seats on {Plan}, down from ${list} after negotiating end of quarter" — [Reddit/{sub}]({url})

---

### What moves the deal

**Most effective pressure points:**
1. **{Point 1}:** {description and why it works, with any data backing it}
2. **{Point 2}:** {description}
3. **{Point 3}:** {description}

**Competitive leverage:**
- Mentioning {Competitor A} as an alternative has been reported to unlock {discount range or concession}
- {Competitor B} is commonly positioned as the free/cheaper alternative — effective for SMB displacement

**Timing:**
- Best window: {end of Q/FY, specific months if company fiscal year is known}
- {Company}'s fiscal year ends {month} — their highest-pressure quarter is {Q}

---

### What they defend (don't expect movement here)

- {Line item 1}: {why it's defended}
- {Line item 2}: {why it's defended}

---

### Multi-year and volume discounts

| Scenario | Typical additional discount |
|----------|---------------------------|
| 2-year commit vs. 1-year | {range} |
| 3-year commit vs. 1-year | {range} |
| {Volume threshold} seats | {break point discount} |
| Annual prepay vs. invoiced | {range} |

---

### Hidden costs to surface before signing

{From G2 reviews and community: costs that don't appear on the pricing page but show up in contracts or post-signature}

- {Cost 1}: {description — e.g., "Implementation/onboarding fee: $2,000–$8,000 for enterprise plans"}
- {Cost 2}: {description — e.g., "True-up charges at annual renewal if seats were added mid-year"}
- {Cost 3}: {description}

---

### Negotiation script

**Opening:** "{Suggested opener that anchors the conversation to your leverage — competitor mention, renewal timing, volume commitment}"

**If they won't move on seat price:** "{Alternative concession to ask for — implementation credits, support tier upgrade, extra admin seats, extended trial for additional team}"

**If they invoke 'standard pricing':** "{Counter — reference specific community/Vendr data point that contradicts this}"
```

---

## Step 6: Log to monday

Follow [monday-logging.md](monday-logging.md):

- Item name: `{Company} — Negotiation Intel`
- Summary: Median contract value + typical discount range in 1–2 sentences
- PricingSaaS link: `https://pricingsaas.com/companies/{slug}`
- Workflow: `negotiation-intelligence`

---

## Data quality notes

Always cite sources and flag confidence level:

| Source | Confidence | Notes |
|--------|-----------|-------|
| Vendr (verified contract data) | High | Actual signed contracts; sample size varies |
| G2 pricing reviews | Medium | Buyer-disclosed; self-reported, may be rounded |
| Reddit / HN community | Medium-Low | Anecdotal but often most candid; date and context matter |
| No data available | — | Note explicitly: "No public negotiation data found for {Company}" |

If no meaningful data is found across all sources: "Insufficient public data to estimate real deal economics for {Company}. This is common for companies with fully sales-led pricing — deals are bespoke and not publicly discussed." Suggest asking in relevant Slack communities (e.g., SaaStr, RevGenius, Pavilion) as a next step.
