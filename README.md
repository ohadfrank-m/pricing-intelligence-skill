# Pricing intelligence — Cursor Agent Skill

A Cursor agent skill for researching and monitoring SaaS pricing strategies using live [PricingSaaS](https://pricingsaas.com) data, public sentiment, and supplementary intelligence sources.

## What it does

| Intent | What you get |
|--------|-------------|
| Single company deep-dive | Full pricing breakdown, tier structure, recent changes, and competitive implications |
| Category / industry landscape | Pricing map across all major players in a space |
| Track pricing changes | Watchlist monitoring with visual before/after diffs |
| Sentiment analysis | How the market and customers are reacting to a company's pricing |
| Pricing page teardown | Psychology, structure, and conversion mechanics of a competitor's page |
| Competitive battlecard | Objection handling, negotiation leverage, plan-by-plan comparison |
| Weekly digest | What changed in pricing this week across your watchlist |
| A/B test detection | Whether a competitor is actively testing their pricing page |
| Freemium / trial tracker | Changes to free tiers, trial structures, and plan limits |
| Negotiation intelligence | What buyers actually pay, typical discounts, how to negotiate |

Every company research output includes a "So what" section covering pricing headroom, positioning implications, experiment opportunities, and threat signals.

## Prerequisites

### 1. PricingSaaS MCP

This skill is powered by the [PricingSaaS MCP](https://mcp.pricingsaas.com).

Add it to your `~/.cursor/mcp.json`:

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

If the server requires an API key, get one at [pricingsaas.com](https://pricingsaas.com) and add it:

```json
"headers": { "authorization": "Bearer YOUR_API_KEY" }
```

### 2. monday.com MCP (optional)

The skill automatically logs all outputs to a "Pricing Intelligence" board on monday.com. This requires the monday.com MCP to be connected (`plugin-monday.com-monday`). If it's not connected, logging is skipped silently — all other functionality works without it.

## Installation

1. Clone this repo into your Cursor skills folder:

```bash
git clone https://github.com/ohadfrank-m/pricing-intelligence-skill ~/.cursor/skills/pricing-intelligence
```

2. In Cursor, go to **Settings → Features → Agent Skills** and verify the skill appears, or add the path manually.

3. Verify connectivity by asking Cursor: `"check pricing intelligence status"` — the skill will call `get_status()` on the PricingSaaS MCP and confirm the connection.

## Usage

Just talk to Cursor naturally. Example triggers:

- `"How does Notion price?"`
- `"Map the project management pricing landscape"`
- `"What changed in pricing this week?"`
- `"Tear down HubSpot's pricing page"`
- `"Build a battlecard — us vs. Asana"`
- `"What do people actually pay for Salesforce?"`
- `"Has Figma changed their free tier recently?"`
- `"Is Intercom A/B testing their pricing page?"`

## File structure

```
SKILL.md                          # Main entrypoint — routing logic and MCP tool reference
references/
  company-research.md             # Single company deep-dive workflow
  trend-research.md               # Category / industry landscape workflow
  monitoring.md                   # Watchlist and pricing change tracking
  battlecard-generator.md         # Competitive battlecard generation
  weekly-digest.md                # Weekly pricing digest
  ab-test-detection.md            # Pricing page A/B test detection
  freemium-trial-tracker.md       # Free tier and trial structure changes
  negotiation-intelligence.md     # Real deal economics and negotiation leverage
  pricing-page-teardown.md        # Pricing page psychology and structure analysis
  sentiment-research.md           # Customer and market sentiment analysis
  category-watchlist.md           # Category-level watchlist setup
  enrichment.md                   # Supplementary research methods (Wayback, job postings, etc.)
  monday-logging.md               # monday.com board logging workflow
  visual-diff.md                  # Visual before/after diff generation
```

## Credits

Built on [PricingSaaS](https://pricingsaas.com). Some tools consume credits (pricing history diffs, visual diffs) — the skill always states the cost and asks for confirmation before spending credits.
