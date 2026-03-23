# Pricing intelligence — AI Agent Skill

An agent skill for researching and monitoring SaaS pricing strategies using live [PricingSaaS](https://pricingsaas.com) data, public sentiment, websearch, and additional intelligence sources.

Works in **Cursor**, **Claude** (via Claude Code or Cowork), and any AI environment that supports MCP and markdown-based skills.

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

**Claude Code** — add to your MCP config (typically `~/.claude/mcp.json` or via `claude mcp add`):

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

**Claude Cowork / other environments** — follow your environment's MCP setup instructions to add `https://mcp.pricingsaas.com` as a remote MCP server.

If the server requires an API key, get one at [pricingsaas.com](https://pricingsaas.com) and pass it as a Bearer token in the Authorization header:

```json
"headers": { "authorization": "Bearer YOUR_API_KEY" }
```

### 2. monday.com MCP (optional)

The skill automatically logs all outputs to a "Pricing Intelligence" board on monday.com. Requires a monday.com MCP connection with access to: `search`, `create_board`, `create_column`, `get_board_info`, `create_item`, `create_doc`, `create_update`. If not connected, logging is skipped silently — all other functionality works without it.

## Installation

### Claude Code

Clone the repo into your project or home directory:

```bash
git clone https://github.com/ohadfrank-m/pricing-intelligence-skill
```

Then register the skill using `--add-dir` so Claude Code discovers it automatically:

```bash
claude --add-dir /path/to/pricing-intelligence-skill
```

Claude Code picks up the skill from `.claude/skills/pricing-intelligence/SKILL.md` via the included symlink. You can also copy or symlink the `skills/pricing-intelligence/` directory into your own project's `.claude/skills/` if you prefer project-level installation:

```bash
# From your project root:
mkdir -p .claude/skills
ln -s /path/to/pricing-intelligence-skill/skills/pricing-intelligence .claude/skills/pricing-intelligence
```

The skill auto-triggers when you ask pricing questions, or invoke it directly with `/pricing-intelligence`.

### Cursor

```bash
git clone https://github.com/ohadfrank-m/pricing-intelligence-skill
```

Then in Cursor, go to **Settings → Features → Agent Skills → Add path** and point to the `skills/pricing-intelligence/` subdirectory inside the cloned repo.

### Claude Cowork

Install directly as a plugin from the Cowork plugin browser, or clone the repo and install manually — the `.claude-plugin/plugin.json` and `.mcp.json` at the root are picked up automatically.

### Other AI environments

Clone or download this repo. The skill entrypoint is `skills/pricing-intelligence/SKILL.md`. Follow your environment's instructions for registering a skill folder, pointing to `skills/pricing-intelligence/`.

### Verify

After installing, ask your AI assistant: `"check pricing intelligence status"` — the skill will call `get_status()` on the PricingSaaS MCP and confirm the connection.

## Usage

Just talk to your AI assistant naturally. Example triggers:

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
.claude/
  skills/
    pricing-intelligence/         -> ../../skills/pricing-intelligence (symlink for Claude Code)
.claude-plugin/
  plugin.json                     # Cowork plugin manifest
.mcp.json                         # Bundled PricingSaaS MCP config (auto-loaded in Cowork)
.gitignore
README.md
skills/
  pricing-intelligence/
    SKILL.md                      # Main entrypoint — routing logic and MCP tool reference
    references/
      company-research.md         # Single company deep-dive workflow
      trend-research.md           # Category / industry landscape workflow
      monitoring.md               # Watchlist and pricing change tracking
      battlecard-generator.md     # Competitive battlecard generation
      weekly-digest.md            # Weekly pricing digest
      ab-test-detection.md        # Pricing page A/B test detection
      freemium-trial-tracker.md   # Free tier and trial structure changes
      negotiation-intelligence.md # Real deal economics and negotiation leverage
      pricing-page-teardown.md    # Pricing page psychology and structure analysis
      sentiment-research.md       # Customer and market sentiment analysis
      category-watchlist.md       # Category-level watchlist setup
      enrichment.md               # Supplementary research methods (Wayback, job postings, etc.)
      monday-logging.md           # monday.com board logging workflow
      visual-diff.md              # Visual before/after diff generation
      hypothesis-engine.md        # Experiment hypothesis generation and backlog
      proactive-monitoring.md     # Automated daily monitoring with severity-based alerts
      severity-scoring.md         # 4-dimension competitive impact scoring framework
      knowledge-base.md           # Persistent cross-session pricing knowledge base
```
