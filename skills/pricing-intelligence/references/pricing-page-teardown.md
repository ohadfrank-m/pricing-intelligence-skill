# Pricing page teardown workflow

Deconstruct *how* a company communicates pricing — not just what they charge. The teardown surfaces psychological tactics, GTM signals, and structural choices that reveal strategic intent. This is a 2–3 hour manual task compressed into minutes.

## When to run

- User asks: "tear down X's pricing page", "analyze how X communicates pricing", "what pricing psychology does X use", "how does X's pricing page work"
- As an optional add-on at the end of company-research: "Want a full teardown of how they present their pricing?"
- When researching a competitor before a product or pricing review

---

## Step 1: Fetch the live pricing page

```
WebFetch(url="https://{domain}/pricing")
```

If that 404s, try common variants:
```
WebFetch(url="https://{domain}/plans")
WebFetch(url="https://{domain}/pricing/plans")
WebSearch(query='"{Company}" pricing page site:{domain}')
```

Capture the full page text. If the page is heavily JS-rendered and WebFetch returns minimal content, note this and supplement with a WebSearch for cached/described versions.

---

## Step 2: Run the teardown framework

Analyze the fetched content against each dimension. Apply all dimensions, even if the finding is "not present" — absence is itself a signal.

### Dimension 1: Hero plan anchoring

Identify which plan is positioned as the default or recommended choice.

Look for:
- "Most popular", "Best value", "Recommended" badge
- Visual highlighting (different background, border, color)
- Annual billing pre-selected vs. monthly
- Middle plan positioned as hero (decoy framing)

**Signal:** The hero plan reveals the segment the company is optimizing conversion for. If it's the highest tier, they're pushing enterprise. If it's mid-tier, they're self-serve/PLG.

### Dimension 2: Pricing metric clarity

Assess how clearly the "what you pay for" is communicated.

Look for:
- Is the unit of pricing stated explicitly? (per seat, per user, per month, per resolution)
- Is usage-based pricing explained or obscured?
- Are limits (storage, API calls, seats) visible without clicking?
- Is the "per user" vs. "per account" distinction clear?

**Signal:** Deliberately obscured metrics (no unit shown, "contact us" for real price) indicate SLG-heavy motion or pricing complexity being used as a sales forcing function.

### Dimension 3: Decoy pricing

Check whether any plan exists primarily to make another look reasonable.

Classic patterns:
- Three plans where middle is clearly the intended purchase and bottom is stripped-down bait
- Enterprise plan with no price listed (anchors perception of Suite Professional as "reasonable")
- A plan priced only slightly below the hero that makes hero look like a bargain

**Signal:** Strong decoy presence = mature pricing team applying behavioral economics. Absence = earlier-stage or less sophisticated pricing.

### Dimension 4: Annual vs. monthly framing

How is the billing frequency choice presented?

Look for:
- Default display: annual or monthly?
- Discount shown on toggle switch or hidden?
- Percentage savings called out explicitly (e.g., "save 20%")?
- Annual price shown per-month (lower number) or per-year (higher, scarier number)?

**Signal:** Defaulting to annual + showing per-month price = maximizing conversion by minimizing sticker shock. Showing per-year = either confidence in willingness to pay or less sophisticated page design.

### Dimension 5: Feature gating transparency

Assess how clearly plan differentiation is communicated.

Look for:
- Feature comparison table: present, partial, or absent?
- "Everything in [lower plan] plus..." structure
- Features listed as included vs. listed with limits vs. hidden until clicked
- Add-ons called out separately from core plans
- "Contact sales" gates (what tier do they hide pricing for?)

**Signal:** Long feature lists with checkmarks = PLG, self-serve decision. Vague "advanced features" language + "contact sales" = SLG, deal-desk-dependent.

### Dimension 6: CTA strategy

Capture the call-to-action text and placement for each plan.

Common patterns and what they signal:

| CTA | Signal |
|-----|--------|
| "Get started free" | PLG, self-serve, low friction |
| "Start free trial" | Trial-led, conversion-optimized |
| "Buy now" | E-commerce motion, low ACV |
| "Start for free" + credit card required | Conversion-optimized but friction-aware |
| "Contact sales" | SLG, enterprise motion, high ACV |
| "Request a demo" | SLG, longer sales cycle |
| Mixed ("Get started" on lower, "Contact sales" on enterprise) | Hybrid PLG+SLG |

Note whether CTAs differ between plans and where the self-serve / sales handoff line sits.

### Dimension 7: Social proof placement

Where does trust-building content appear relative to the price reveal?

Look for:
- Logos above the fold (before prices shown) — reduces sticker shock
- Logos below the fold (after prices) — less sophisticated or different intent
- Testimonials next to specific plans (associating proof with conversion point)
- G2/analyst badges near pricing
- Customer count or ARR claims near pricing ("Trusted by 100,000+ companies")

### Dimension 8: Loss aversion and urgency tactics

Check for pressure/scarcity signals:

- Countdown timers or limited offers
- "Prices increase on [date]" banners
- "You're saving X% vs. monthly" callouts
- Free trial expiry prominently shown
- "Cancel anytime" reassurance (reducing commitment anxiety)

### Dimension 9: Free tier / trial structure

If applicable:

- Free tier limits: what specifically is capped?
- Trial length and whether credit card is required
- Is the free → paid upgrade path visible from the pricing page?
- Is the free tier listed alongside paid plans or on a separate page?

**Signal:** Free tier that's hidden or de-emphasized = PLG company in the process of monetizing their free base. Prominent free tier = still acquisition-focused.

### Dimension 10: AI / new technology pricing signal

Specific to 2025–2026 context:

- Is AI priced as a separate add-on, bundled into higher tiers, or included everywhere?
- Is there "per-resolution" or "per-interaction" pricing visible?
- Is AI mentioned on the pricing page at all, or kept off-page?

---

## Step 3: Output format

```
## {Company} — pricing page teardown

**Page URL:** {url}  
**Fetched:** {date}  
**GTM motion (inferred):** {PLG / SLG / Hybrid PLG+SLG}

---

### Hero plan
{Which plan, what signals mark it as hero, what segment it targets}

### Pricing metric
{What they charge for, how clearly it's stated, any deliberate obscuring}

### Decoy pricing
{Whether present, which plan serves as decoy, effectiveness}

### Annual/monthly framing
{Default display, how savings are shown, psychological framing used}

### Feature gating
{Transparency level, comparison table presence, what's hidden vs. shown}

### CTA strategy
{CTAs per plan, where the self-serve/sales handoff line is}

### Social proof
{Placement, type, density relative to price reveal}

### Loss aversion / urgency
{Tactics present, assessment of aggression level}

### Free tier / trial
{Structure, limits, visibility, upgrade path clarity}

### AI pricing signal
{How AI is positioned on the page, add-on vs. bundled vs. absent}

---

### Overall assessment

**Sophistication level:** {Low / Medium / High / Very High}  
Rate based on: decoy presence, behavioral tactics used, CTA optimization, feature table quality.

**What this reveals strategically**  
{2–3 sentences: what the page design tells you about their conversion priorities, target buyer, GTM motion, and pricing team maturity}

**What to borrow / what to avoid**  
{1–2 specific tactical observations relevant to your own pricing page}
```

---

## Step 4: Compare against Wayback Machine

After the live teardown, check one historical snapshot to surface what changed:

```
WebFetch(url="https://web.archive.org/cdx/search/cdx?url={domain}/pricing&output=json&limit=5&fl=timestamp,statuscode&filter=statuscode:200&collapse=timestamp:6")
```

Fetch the earliest available snapshot and note any structural differences vs. today. A company that added "Most Popular" badges, switched to annual-default, or restructured their CTA copy since their early days has an active, iterating pricing team.

---

## Step 5: Log to monday

After delivering the teardown, log per [monday-logging.md](monday-logging.md):

- Item name: `{Company} — Page Teardown`
- Summary: GTM motion inferred + single most significant tactical observation
- PricingSaaS link: `https://pricingsaas.com/companies/{slug}`
- Workflow: `pricing-page-teardown`
