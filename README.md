# Grant Finder for Hispanic-Serving & Latin American Nonprofits

A Claude plugin that matches a nonprofit against real grant opportunities — private foundations, open federal grants, and (for US orgs) Census-backed community data — then ranks them, and **refuses to guess** when it can't verify a detail.

**→ [Live walkthrough](https://claude.ai/code/artifact/6cb6fedd-31f9-419f-b4a6-e9652d58593f)**  ·  **[Hosted MCP server (companion repo)](https://github.com/sofiagoble/grants-gov-server)**

---

## What it is

Grant prospecting is slow, and the commercial databases that speed it up cost money most small nonprofits don't have — while still missing the Latino-serving funders and local community-foundation funds that never surface in a generic index.

Describe your organization in plain language, and the plugin searches real public data — IRS tax filings, the federal `grants.gov` register, and the live web — then returns a ranked shortlist with eligibility, grant size, and a deadline it has actually checked.

One conversation serves two kinds of organization, branching on the org's location:

- **US-based** nonprofits serving Hispanic/Latino communities → the full domestic pipeline.
- **Latin-America-based** organizations → US-origin funding that reaches their country (Inter-American Foundation, State Department embassy programs, US foundations with Latin America portfolios).

## How it works

Three data sources, each good at one thing, merged by organization name into a single ranked list:

| Source | Answers | How it's reached |
|---|---|---|
| **IRS Form 990** (ProPublica API) | Is this a real, adequately-resourced funder? | Direct API call, no key |
| **Live web search** | Is it open now, and what's the deadline? | Claude's built-in search |
| **Federal grants** (Simpler Grants API) | What's posted on grants.gov right now? | A [hosted MCP server](https://github.com/sofiagoble/grants-gov-server) I built + deployed |

The skill logic lives in [`skills/find-grants/SKILL.md`](skills/find-grants/SKILL.md); each data source's quirks and rules are documented in [`connectors/`](connectors/).

## The engineering, briefly

The interesting part wasn't the happy path — it was handling public government data sources that misbehave. A sample, all written into the connector configs so they wouldn't have to be rediscovered:

- **ProPublica returns HTTP 404 for a zero-result query** (not an empty list), and its `ntee[id]` sector filter is **completely broken (HTTP 500)** — worked around with keyword+state search and manual classification.
- **grants.gov's search pages are JavaScript shells** that return only page chrome to a fetch — which is *why* the dedicated API server exists.
- **A valid Census key was rejected as "invalid"** — the real cause was a User-Agent bot-block; a separate query shape returned empty and was fixed with the API's `ucgid` geography syntax.
- **The deployed server rejected every request** (`421 Invalid Host header`) — traced through the MCP SDK source to a DNS-rebinding guard that only trusts localhost, then disabled for the hosted case.
- **A tempting "$500K program"** that showed up across aggregator sites **failed primary-source verification** and was excluded rather than reported.

It's also honest about what *couldn't* be done (USAID returns nothing searchable, the Inter-American Foundation's pages block automated reads) — documented as findings rather than quietly dropped.

## Install

Requires the Claude desktop app on a paid plan.

1. Build the installable plugin from this repo: zip the contents so `.claude-plugin/plugin.json` sits at the archive root, and name it `hispanic-latam-grants.plugin`.
2. In **Claude Desktop → Customize → Plugins**, upload that file.
3. Describe your nonprofit in a chat and read back the ranked shortlist.

Federal search runs against the hosted server; if it's ever unreachable the plugin falls back to a public-page search and says so in its answer. The optional Census section needs a free [Census API key](https://api.census.gov/data/key_signup.html) pasted into [`connectors/census-acs.json`](connectors/census-acs.json) (ships with a placeholder — no key is included in this repo).

## Repository layout

```
.claude-plugin/plugin.json   Plugin manifest (what Claude reads to install)
manifest.json                Human-readable capability summary
.mcp.json                    Points at the hosted federal-grants server
skills/find-grants/          The skill logic + reference data
connectors/                  Per-source configs, quirks, and documented bugs
docs/PLUGIN-REFERENCE.md     Full operational detail, setup, and limitations
```

## Built with

Claude plugin/skill authoring · Model Context Protocol server (Python, FastMCP) · Docker · Render · REST/GraphQL integration with the Simpler Grants, ProPublica Nonprofit Explorer, and US Census ACS APIs.

## Honest limitations

Deliberately scoped for a testable v1: 990 data lags 12–24 months, the federal server is a real hosting commitment (free-tier cold starts), and Latin America support surfaces only US-origin funding — there's no connector for any country's own domestic grant programs. Full detail in [`docs/PLUGIN-REFERENCE.md`](docs/PLUGIN-REFERENCE.md).
