# rubic-widget-skill

Agent skill generated from the Cryptorubic/docs repository (Rubic SDK Lite, API v2, MCP server, monetization docs) with [book-to-skill](https://github.com/virgiliojr94/book-to-skill).

> **Content is synthesized summaries of the official Rubic documentation, not the docs verbatim.** Chapter files restructure the material into frameworks, decision rules, worked examples, and anti-patterns.

## What's inside

- **Rubic SDK Lite** (`@cryptorubic/sdk-lite`): quoteBest / quoteAll / quoteDepositTrades → swap / swapBest / swapDepositTrade → waitForStatus; allowance, checkApprove, claim, celerRefund; getChains / getProviders / healthcheck
- **REST API v2** (`api-v2.rubic.exchange`): quoteBest / quoteAll / swap / info endpoints, request/response shapes
- **MCP server** (`@cryptorubic/mcp`): tool catalog, config for every MCP host, security model, error envelope
- **Monetization**: fee models (percentage / fixed / mixed), integrator whitelisting, deposit-provider fee sharing
- **Statuses, claims & refunds**: terminal vs pending statuses, READY_TO_CLAIM (Arbitrum Bridge), Celer refunds
- **Error handling**: 6-digit error-code decoder, RubicApiError, quote-vs-swap amount drift guard (±0.5%)
- **Chains & providers**: 70+ chains, 340+ DEXs/bridges, provider blacklist semantics

## File inventory

```
SKILL.md                     core flow, decision rules, quickstart, indexes
chapters/ch01..ch08          deep-dives per topic
glossary.md                  ~50 terms with definitions
patterns.md                  11 integration patterns (retry, approval, deposit, claim, ...)
cheatsheet.md                decision tables, error decoder, status tree, thresholds
```

## Install

```bash
# Hermes (this machine or any Hermes install)
hermes skills install https://github.com/Luci666-Onyx/rubic-widget-skill --skill rubic-widget
```

For other Agent Skills hosts (Claude Code, Copilot CLI, Amp, Codex):

```bash
npx skills add https://github.com/Luci666-Onyx/rubic-widget-skill --skill rubic-widget
```

Or manually: clone the repo and copy `SKILL.md` + `chapters/` + `glossary.md` + `patterns.md` + `cheatsheet.md` into your host's skills directory as `rubic-widget/`.

## Usage

```
Ask: rubic-widget                       → loads core flow + decision rules
Ask: rubic-widget about waitForStatus   → reads the status chapter
Ask: rubic-widget for ch07              → error handling deep-dive
```

## Updating

The skill is generated from the live Rubic docs — re-run `book-to-skill` against https://github.com/Cryptorubic/docs to refresh. Chain/provider lists change constantly; the skill itself tells the agent to query `getChains()` / `getProviders()` at runtime rather than trusting memorized tables.
