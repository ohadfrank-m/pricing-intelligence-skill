---
name: pricing-intelligence
description: >-
  Research and monitor SaaS pricing strategies using live PricingSaaS data, public sentiment, and
  supplementary intelligence sources. Triggers: "how does X price", "pricing strategy of X",
  "research X pricing", "pricing trends in [industry]", "competitive pricing landscape",
  "who competes with X", "monitor pricing", "track pricing changes", "what changed this week",
  "pricing watchlist", "what do people think about X's pricing", "pricing sentiment",
  "tear down X's pricing page", "pricing page teardown", "pricing battlecard for X",
  "losing deals to X on price", "weekly pricing digest", "pricing brief",
  "has X changed their free tier", "freemium changes", "what do people actually pay for X",
  "how to negotiate X pricing", "is X A/B testing their pricing page",
  "set up category watchlist", "schedule pricing digest", "automate pricing monitoring".
  Every company research output includes a 'So what for monday.com' section.
  Requires PricingSaaS MCP. See setup section in this skill.
---

# Pricing intelligence

Research specific company pricing strategies, map industry pricing landscapes, and monitor pricing changes — powered by live PricingSaaS MCP data.

## Prerequisites: PricingSaaS MCP

This skill requires the PricingSaaS MCP server (`https://mcp.pricingsaas.com`) to be connected in your AI environment.

**Cursor:** Add to `~/.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "pricingsaas": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://mcp.pricingsaas.com"]
    }
  }
}
```

**Claude / other environments:** Follow your environment's MCP configuration instructions to add `https://mcp.pricingsaas.com` as a remote MCP server.

**API key:** If the connection fails, the server may require an API key. Get one at [pricingsaas.com](https://pricingsaas.com) and pass it as a Bearer token in the Authorization header.

Verify connectivity by calling `get_status()` before proceeding. If it fails entirely (not just 0 credits), inform the user the PricingSaaS connection is unavailable — see the connection fallback rule in the MCP tool reference section below.

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
| Proactive monitoring | "run daily monitor", "check for critical pricing changes", "set up pricing alerts", "run proactive monitoring" | [proactive-monitoring.md](references/proactive-monitoring.md) |
| Knowledge base query | "show me what I know about X", "what's in my pricing knowledge base", "KB summary", "cross-company patterns", "what patterns have I seen" | [knowledge-base.md](references/knowledge-base.md) |
| Experiment backlog | "show experiment backlog", "what experiments should I run", "quarterly experiment review", "review experiment backlog", "mark experiment as launched", "sync experiments to monday" | [hypothesis-engine.md](references/hypothesis-engine.md) |

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
| `get_watchlist()` | Free | List all companies currently on the watchlist |
| `get_pricing_news()` | Free | Recent pricing changes across all tracked companies |
| `fetch_diffs(scope, period, period_type)` | **2 credits** | Detailed change data for watchlist or global scope |
| `search_pricing_knowledge(query)` | Free | Pricing strategy frameworks and best practices |
| `add_page(url)` | Free | Submit a company's pricing page for tracking |
| `upload_report(filename, file_path)` + curl | Free | Generate and upload shareable HTML landscape report |

**Credit cost rule:** Always state the credit cost and get explicit user confirmation before calling any paid tool.

**Credit-zero fallback:** When `get_status()` returns 0 credits, do NOT skip pricing history. Automatically run the credit-zero fallback protocol ([enrichment.md](references/enrichment.md) Method 6) for any period flagged as a high-signal change — reconstructing before/after values from Wayback Machine, third-party trackers, community posts, and changelogs. Label all reconstructed output clearly. No user prompt needed to activate this — it runs silently whenever credits are depleted.

**Connection failure fallback:** If `get_status()` fails entirely (timeout, unreachable, auth error — not just 0 credits), inform the user: "The PricingSaaS connection is currently unavailable. I can still run enrichment-only research using Wayback Machine snapshots, web signals, and community sources — this covers most of what the skill does without live MCP data. Want me to proceed?" If yes, route directly to [enrichment.md](references/enrichment.md) for all applicable methods.

---

## Monday.com logging

Every workflow logs its output to the user's chosen monday.com board after delivering results. The board is selected at the start of the first logging call in a session — see [monday-logging.md](references/monday-logging.md) for the full flow.

Full setup and item creation logic: [monday-logging.md](references/monday-logging.md)

Requires a monday.com MCP connection with access to: `search`, `create_board`, `create_column`, `get_board_info`, `create_item`, `create_doc`, `create_update`. If not connected, logging is skipped silently — all other skill functionality works without it.

---

## Knowledge base

Every research session persists its findings to a local knowledge base at `Claude-Workspace/pricing-intelligence/knowledge-base.json`. This enables cross-session intelligence: prior research is loaded at session start, cross-company patterns are detected automatically, and the experiment backlog accumulates over time.

**Session start:** Before routing to any workflow, check if the knowledge base file exists. If yes, read it. For company-research and battlecard workflows, look up the target company and surface prior research context in the output preamble.

**Session end:** After every workflow's monday.com logging step, upsert the session's findings into the knowledge base. This happens automatically via [knowledge-base.md](references/knowledge-base.md).

Full schema, read/write logic, cross-company pattern detection, and monday.com board sync: [knowledge-base.md](references/knowledge-base.md)

---

## Severity scoring

Every detected pricing change is scored on 4 dimensions (price proximity to monday.com, segment overlap, change magnitude, strategic signal) for a total of 0–12 points, mapped to severity tiers P0–P3. This score drives alert urgency in proactive monitoring, digest prioritization, and experiment generation urgency.

**Credit cost:** Zero. Severity scoring uses only knowledge base data and the monday.com pricing reference — no MCP credits consumed.

Full framework with scoring tables and examples: [severity-scoring.md](references/severity-scoring.md)

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
