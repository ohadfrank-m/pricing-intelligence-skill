---
name: pricing-intelligence
description: >-
  Research and monitor SaaS pricing strategies using live PricingSaaS data, public sentiment, and
  supplementary intelligence sources. Use when the user wants to: research how a specific company
  prices (triggers: "how does X price", "pricing strategy of X", "research X pricing", "break down
  X's pricing", "what does X charge", "X pricing model", "X packaging", "how did X's pricing page
  change"); research pricing trends in a category or industry (triggers: "pricing trends in", "how
  does the market price", "competitive pricing landscape", "who competes with X and how do they
  price", "category pricing", "pricing in [industry]", "map the market"); monitor and track pricing
  changes (triggers: "monitor pricing", "track pricing changes", "what changed in pricing", "pricing
  news", "watchlist", "add to watchlist", "who changed pricing", "pricing updates"); analyze
  customer and market sentiment about pricing (triggers: "what do people think about X's pricing",
  "pricing sentiment", "customer reactions to X", "how is the market reacting", "pricing
  controversy"); tear down a competitor's pricing page (triggers: "tear down X's pricing page",
  "analyze how X presents pricing", "pricing page teardown", "how does X's page work"); build a
  competitive battlecard (triggers: "pricing battlecard for X", "we're losing deals to X on price",
  "competitive pricing comparison", "how do we handle X pricing objections"); run a weekly pricing
  digest (triggers: "weekly pricing digest", "what changed in pricing this week", "pricing brief",
  "weekly pricing update"); or detect A/B testing on a competitor's pricing page (triggers: "is X
  testing their pricing page", "pricing page experiments at X", "is X A/B testing pricing").
  track changes to free tiers and trial structures (triggers: "has X changed their free tier",
  "did X restrict their freemium", "track trial changes in {category}", "freemium changes");
  research real deal economics and negotiation leverage (triggers: "what do people actually pay for
  X", "typical discount for X", "how to negotiate X pricing", "real cost of X", "negotiation
  intelligence for X"); set up a category-level watchlist to track all competitors in a space at
  once (triggers: "track {category} pricing", "add all {category} competitors to watchlist", "monitor
  the {category} market", "set up category watchlist"); or schedule and automate the weekly digest
  (triggers: "schedule my pricing digest", "automate pricing monitoring", "how do I get this every
  week"). Every company research output includes a 'So what for monday.com' section and an exec
  summary ready to share with Orly or paste into Slack. Requires PricingSaaS MCP connected. See
  setup section in this skill.
---

# Pricing intelligence

Research specific company pricing strategies, map industry pricing landscapes, and monitor pricing changes — powered by live PricingSaaS MCP data.

## Prerequisites: PricingSaaS MCP

The PricingSaaS MCP (`https://mcp.pricingsaas.com`) is already registered in `~/.cursor/mcp.json`.

If the connection fails, the server may require an API key. To add one:
1. Get your key at [pricingsaas.com](https://pricingsaas.com)
2. Add it to `~/.cursor/mcp.json` under the `pricingsaas` entry: `"headers": { "authorization": "Bearer YOUR_API_KEY" }`

Verify connectivity by calling `get_status()` before proceeding. If it fails, prompt the user to check their API key.

---

## Routing

Identify the user's primary intent and route to the appropriate workflow:

| Intent | Signals | Workflow |
|--------|---------|----------|
| Single company deep-dive | company name + "price", "strategy", "packaging", "tiers", "model" | [company-research.md](references/company-research.md) |
| Category / industry landscape | "industry", "category", "landscape", "trends", "market", "competitors", "who competes" | [trend-research.md](references/trend-research.md) |
| Category watchlist setup | "track the {category} space", "monitor pricing in {category}", "add all {category} competitors", "set up watchlist for {category}", "watch the whole {category} market" | [category-watchlist.md](references/category-watchlist.md) |
| Track pricing changes | "monitor", "watchlist", "what changed", "pricing news", "updates", "alerts" | [monitoring.md](references/monitoring.md) |
| Sentiment / market reaction | "what do people think", "sentiment", "customer reactions", "controversy", "how are people reacting" | [sentiment-research.md](references/sentiment-research.md) |
| Pricing page teardown | "tear down", "analyze pricing page", "how does X present pricing", "pricing page teardown", "pricing psychology" | [pricing-page-teardown.md](references/pricing-page-teardown.md) |
| Competitive battlecard | "battlecard", "losing deals to X", "pricing objections", "compete with X on price", "vs X pricing" | [battlecard-generator.md](references/battlecard-generator.md) |
| Weekly digest | "weekly digest", "weekly pricing", "what changed this week", "pricing brief", "weekly update", "run my digest", "schedule digest", "automate pricing monitoring" | [weekly-digest.md](references/weekly-digest.md) |
| A/B test detection | "testing their pricing page", "A/B testing pricing", "pricing page experiments", "is X changing their page" | [ab-test-detection.md](references/ab-test-detection.md) |
| Freemium / trial tracker | "changed their free tier", "restricting freemium", "trial change", "free plan limits", "did X remove freemium" | [freemium-trial-tracker.md](references/freemium-trial-tracker.md) |
| Negotiation intelligence | "what do people actually pay", "typical discount", "how to negotiate X", "real cost", "negotiation leverage", "what can I get off" | [negotiation-intelligence.md](references/negotiation-intelligence.md) |

Enrichment methods (Wayback Machine, job postings, changelog mining, earnings calls, cross-company patterns) are embedded in the core workflows and called automatically when relevant. They are also documented in [enrichment.md](references/enrichment.md) for direct reference.

If intent is ambiguous, ask one clarifying question: "Are you researching a specific company, mapping a market, tracking changes, building a battlecard, or something else?"

---

## MCP tool reference

All tools are free unless marked with credit cost.

| Tool | Cost | Purpose |
|------|------|---------|
| `get_status()` | Free | Check account and connection status |
| `search_companies(query)` | Free | Find companies by name or keyword |
| `search_companies_advanced(filters)` | Free | Attribute-based discovery |
| `get_company_details(slug)` | Free | Full pricing breakdown for a company |
| `get_company_history(slug, discovery_only=true)` | Free | Preview available pricing history periods |
| `get_company_history(slug)` | **1 credit/diff** | Full pricing change history |
| `get_diff_highlight(slug, period, query)` | **1 credit** | Visual before/after screenshot of a price change |
| `add_to_watchlist(slugs=[...])` | Free | Add companies to monitoring watchlist |
| `get_pricing_news()` | Free | Recent pricing changes across all tracked companies |
| `fetch_diffs(scope, period, period_type)` | **2 credits** | Detailed change data for watchlist or global scope |
| `search_pricing_knowledge(query)` | Free | Pricing strategy frameworks and best practices |
| `add_page(url)` | Free | Submit a company's pricing page for tracking |
| `upload_report(filename, file_path)` + curl | Free | Generate and upload shareable HTML landscape report |

**Credit cost rule:** Always state the credit cost and get explicit user confirmation before calling any paid tool.

**Credit-zero fallback:** When `get_status()` returns 0 credits, do NOT skip pricing history. Automatically run the credit-zero fallback protocol ([enrichment.md](references/enrichment.md) Method 6) for any period flagged as a high-signal change — reconstructing before/after values from Wayback Machine, third-party trackers, community posts, and changelogs. Label all reconstructed output clearly. No user prompt needed to activate this — it runs silently whenever credits are depleted.

---

## Monday.com logging

Every workflow automatically logs its output to the **Pricing Intelligence** board on monday.com after delivering results to the user.

Full setup and item creation logic: [monday-logging.md](references/monday-logging.md)

The monday.com MCP (`plugin-monday.com-monday`) is already connected. Tools used: `search`, `create_board`, `create_column`, `get_board_info`, `create_item`.

---

## Output standards

- Lead with the key insight, not methodology
- Include `https://pricingsaas.com/pulse/companies/{slug}` links for every company
- Metrics always include units and context (e.g., "$49/user/mo", "3 tiers", "+$20 since Q3 2024")
- Company plan tables: Company · Plan name · Monthly price · Annual price · Pricing model
- When enrichment sources are used, cite them with URLs — don't present WebSearch or Wayback Machine findings as PricingSaaS data
- No filler, no padding, no restating the question before answering
- **Every company research output must include:** (1) a 3-bullet exec summary at the top, (2) a "So what for monday.com" section covering pricing headroom, positioning implications, experiment opportunities, and threat signals
- After every company research, offer to generate a pricing battlecard (monday.com vs. the researched company) before closing
