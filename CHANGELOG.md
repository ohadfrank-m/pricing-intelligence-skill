# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

## [0.1.0] — 2026-03-20

### Added
- Initial release of the pricing intelligence skill
- 10 core workflows: company research, trend/landscape research, monitoring, pricing page teardown, competitive battlecard, weekly digest, A/B test detection, freemium/trial tracker, negotiation intelligence, sentiment analysis
- Sentiment research: LinkedIn articles and posts, X/Twitter posts (with nitter fallback), and 3-tier publication coverage (tech press, SaaS-specific, analyst firms)
- Source weighting guidance in sentiment synthesis
- PricingSaaS MCP tool reference with credit cost annotations and `get_watchlist()` support
- Credit-zero fallback — reconstructs pricing history from Wayback Machine, third-party trackers, and community sources when credits are depleted
- Connection failure fallback — offers enrichment-only research when PricingSaaS is unreachable
- monday.com board logging — auto-logs all outputs to the Pricing Intelligence board with visual diffs
- Weekly digest scheduling options: Slack reminder, calendar event, monday.com recurring item, and Claude Cowork `create_scheduled_task` (fully automatic)
- Category-level watchlist setup for bulk competitor monitoring
- Enrichment methods: Wayback Machine, job postings, changelog mining, earnings call analysis, cross-company pattern detection
- WebFetch graceful failure handling — skips blocked or unreachable sources silently
- Cowork-compatible plugin format: `.claude-plugin/plugin.json`, `.mcp.json`, `skills/` directory structure

[0.1.0]: https://github.com/ohadfrank-m/pricing-intelligence-skill/releases/tag/v0.1.0
