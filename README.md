# Pricing Intelligence

An AI agent skill that researches and monitors SaaS pricing strategies using live [PricingSaaS](https://pricingsaas.com) data, public sentiment, and web intelligence. Works in Cursor and Claude Code.

Ask things like `"How does Notion price?"`, `"Build a battlecard — us vs. Asana"`, `"What changed in pricing this week?"`, or `"Tear down HubSpot's pricing page"` — the skill handles the rest.

| Capability | What you get |
|------------|-------------|
| Company deep-dive | Full pricing breakdown, tier structure, recent changes, competitive implications |
| Industry landscape | Pricing map across all major players in a category |
| Change monitoring | Watchlist with visual before/after diffs and weekly digests |
| Competitive battlecard | Plan-by-plan comparison, objection handling, negotiation leverage |
| Pricing page teardown | Psychology, structure, and conversion mechanics of a competitor's page |
| Sentiment analysis | How customers and the market are reacting to pricing changes |
| Freemium / trial tracker | Changes to free tiers, trial structures, and plan limits |
| Negotiation intelligence | What buyers actually pay, typical discounts, how to negotiate |

Every output includes a **"So what for monday.com"** section with pricing headroom, positioning implications, and experiment opportunities.

## Install

### Cursor

```bash
npx skills add ohadfrank-m/pricing-intelligence-skill
```

### Claude Code

```bash
claude plugins add ohadfrank-m/pricing-intelligence-skill
```

## Setup: connect PricingSaaS MCP

Each user needs to connect the [PricingSaaS MCP](https://mcp.pricingsaas.com) — the data source that powers the skill.

**Cursor** — add to `~/.cursor/mcp.json`:

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

**Claude Code:**

```bash
claude mcp add pricingsaas -- npx -y mcp-remote https://mcp.pricingsaas.com
```

On first use, a browser window will open for authentication. Verify the connection by asking: `"check pricing intelligence status"`.

## Optional: monday.com logging

The skill can log all research outputs to a "Pricing Intelligence" board on monday.com. Requires a monday.com MCP connection. If not connected, logging is skipped silently.
