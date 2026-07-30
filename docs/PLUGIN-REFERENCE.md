# Hispanic & Latin American Nonprofit Grant Finder

A Claude Cowork plugin that matches a nonprofit's mission, location, sector, budget, and size against grant-making foundations **and open federal grant opportunities** — prioritizing funders/programs with a track record of supporting Hispanic-serving organizations. Serves **both** US-based orgs and orgs based in Latin America.

## Overview

Give the skill five facts about your organization and it returns a ranked shortlist of 5-10 grants worth pursuing — private foundation and federal opportunities together in one list — each with a type, description, eligibility notes, typical grant size, and application deadline.

**Two audiences, one skill:**
- **US-based orgs** serving Hispanic/Latino communities — the full pipeline applies: IRS 990, peer-organization discovery, federal grants.gov search, and optional Census demographic context for the grant narrative.
- **Orgs based in Latin America** — this plugin has no access to any country's own local/government funding database, so it surfaces **US-origin funding that reaches Latin America** instead: the Inter-American Foundation, State Department/embassy small-grants programs (searched per-country, not generically), and US private foundations with dedicated Latin America programs (Ford, Open Society, etc.). Peer-organization discovery (US-tax-filing-based) and Census demographic context don't apply and are skipped automatically.

## Components

- **Skill: `find-grants`** (`skills/find-grants/SKILL.md`) — the core matching logic: collects inputs, queries IRS Form 990 data, calls the `grants-gov` MCP connector for federal opportunities, falls back to web search when needed, ranks candidates, and formats the output table.
- **MCP server: `grants-gov`** (`.mcp.json`) — a hosted connector wrapping the real Simpler Grants API for structured federal-opportunity search. Server implementation and deployment instructions live in the separate `grant-finder-hispanic-server` project, not inside this plugin bundle (installers only need the URL in `.mcp.json`, not the server source).
- **Connector config: `connectors/irs-990.json`** — endpoint and field reference for querying IRS Form 990 filings via ProPublica's free Nonprofit Explorer API. No API key required.
- **Connector config: `connectors/grants-gov.json`** — documents the `grants-gov` MCP tool's parameters (primary method) plus the WebFetch-based public-page method it falls back to if the MCP server is unreachable.
- **Connector config: `connectors/census-acs.json`** — documents how to pull Hispanic/Latino population share and poverty rate for the org's city/county from the Census Bureau's ACS API, to support the grant narrative's statement of need. Includes a documented, tested workaround for a real bug where `WebFetch` fails on a specific Census query pattern.
- **Reference: `skills/find-grants/references/seed-funders.md`** — a starting list of funder families known for Hispanic-serving/Latin American giving, used to prime searches (not treated as ground truth — always verified against live data).

One skill, one MCP connector — no agents or hooks.

## Setup

**Private foundations (IRS 990):** nothing to configure — public API, no key.

**Federal grants (grants-gov):** requires the `grants-gov` MCP server to be deployed and `.mcp.json` pointed at it. See `grant-finder-hispanic-server/README.md` for full deployment steps (get a free Simpler Grants API key, deploy the included Docker container to a host of your choice, update `.mcp.json`). **This is an ongoing responsibility, not a one-time setup** — see "Who runs this" below. If the server is ever unreachable, the skill automatically falls back to the WebSearch/WebFetch method (no API key needed) and says so explicitly in its output, so results degrade rather than fail outright — but that fallback is less complete than the real API (see "Known limitations").

**Census demographic data (optional):** get a free API key at [api.census.gov/data/key_signup.html](https://api.census.gov/data/key_signup.html) (instant, email-based, no OAuth) and replace the placeholder in `connectors/census-acs.json`. Unlike `grants-gov`, this is a one-time setup with no ongoing hosting — but **each installer needs their own key**, since it's just pasted into a config file rather than living on a server. Don't ship your personal key if distributing this plugin to others.

### Who runs this

Federal-grant search depends on a server that has to keep running. That means:

- **Cost**: whatever hosting platform you deploy to (see the server README for options) — some have free tiers with tradeoffs (e.g. cold-start delays), none are guaranteed free forever.
- **Maintenance**: if the server goes down, the API key expires, or the host stops the service for non-payment, the skill silently degrades to its (less complete) fallback rather than breaking outright — but only you can actually fix the underlying problem.
- **No built-in alerting**: the server doesn't monitor or notify itself. Set up an external uptime check (e.g. a free UptimeRobot ping) if you want to know about outages before a user mentions one.

The private-foundation half of this plugin has none of these dependencies — it's the federal-grants half specifically that now requires this tradeoff, made deliberately in exchange for more accurate/current federal results than the web-search fallback alone.

## Usage

In a Cowork session, say something like:

> Find grants for my nonprofit. We're a workforce development org serving Hispanic immigrants in Houston, TX. Annual budget is about $400,000, and we have a staff of 8.

Or trigger it explicitly by providing the five inputs:

- `org_mission` — what the org does
- `location` — where it operates or who it serves
- `sector` — e.g. education, health, arts, workforce development, immigration services
- `annual_budget` — approximate annual operating budget
- `nonprofit_size` — grassroots, small, mid-size, or large

The skill will return a ranked table of funder matches with a short rationale for each, and will flag any deadline or grant-size figure it couldn't confirm against current sources.

## How matching works

The two data sources answer different questions, and the skill merges them by organization name before ranking:

- **IRS Form 990** (via ProPublica) answers *"is this a real, adequately-resourced funder?"* — revenue, assets, and (for private foundations only) total grants paid. It doesn't contain program names, eligibility rules, or deadlines, and it lags 12-24 months.
- **Web search** answers *"what's the actual current program, and is it open?"* — the named grant cycle, eligibility criteria, award size, and deadline.

In testing, web search did most of the candidate *discovery* for a specific cause+geography combination (990 keyword search alone struggled to surface niche/local funders), while 990 data was most useful for *validating* a candidate once named — confirming it's a substantial funder rather than a shell org. Expect that division of labor rather than 990 driving discovery end-to-end.

Two additional techniques strengthen discovery and filter out weak matches:

- **Peer-organization discovery** — since there's no direct "who funded this org" query available, the skill instead finds 2-4 similar nonprofits (same sector/NTEE code, comparable scale, nearby region) via the 990 search filters, then checks *those* orgs' own funders/annual-report pages for foundation names. A funder that backs multiple peers is a stronger signal than one found via a generic keyword hit.
- **Recipient turnover check** — before recommending a funder, the skill spot-checks whether their published grantee list actually changes year to year. A foundation that gives to the same handful of orgs every cycle is flagged as a weak prospect for a new applicant, even if it's large and well-resourced.

Federal grants are a third, independent track (Step 3.6): grants.gov opportunities have no relationship to IRS 990 data. The skill calls the `grants-gov` MCP connector first — a real, structured, authenticated search — and only falls back to `WebSearch`/`WebFetch` against public pages if that connector is unreachable, explicitly saying so in its output when that happens. Either way, results are folded into the same ranked list as the private-foundation candidates.

### Pass-through federal money

A lot of federal grant dollars don't go straight to a nonprofit — Congress funds a federal agency, which sends the money by formula to a state or local agency, which then runs its own competition to subaward it locally. When a grants.gov posting's eligible applicant types are entirely government bodies (no nonprofit/501(c)(3) code), this plugin labels it `Federal (pass-through only)` and then traces it one hop further (Step 3.7): a plain web search for the program name plus the org's state reliably named the actual administering agency in testing — no need for the heavier Assistance Listing Number (ALN) + SAM.gov/USAspending lookup that a fuller version would use as a fallback.

Tracing doesn't always end in a confirmed deadline, and the skill reports honestly whichever of these it actually finds rather than forcing a fake one: a live subgrant competition with a real deadline, a funding-opportunities page with only stale/past-year postings, or a named contact for partnership inquiries with no formal open competition at all. In testing (a CSBG posting and a Dept. of Labor dislocated-worker program, both for a Texas-based org), the result was a named contact in one case and a stale funding page in the other — genuinely useful next steps, just not guaranteed deadlines. Tracing stops after one hop; if the agency found further delegates locally (e.g. to a county-level subrecipient), that's named in the output rather than chased further.

### Community need context (Census data)

A fourth, optional section (Step 6) — distinct from funder matching entirely. It answers "how do we justify the need in our application," not "who might fund us." Given the org's location, it pulls real Hispanic/Latino population share and poverty rate from the Census Bureau's American Community Survey, to back a grant narrative's statement of need with a real, citable figure instead of a vague claim.

This surfaced a real, reproducible bug worth knowing about: `WebFetch` reliably returns **empty** for any Census API query combining a specific `for=` value with an `in=` clause (e.g. `for=county:201&in=state:39`) — even though the identical URL works fine in a browser or via `curl` with a normal User-Agent. The fix, confirmed working: use the `ucgid=` parameter (Census's single-parameter geography syntax) for the actual data fetch instead, reserving `for=`/`in=` only for wildcard name-lookup queries (which work fine). This is documented in `connectors/census-acs.json` so it doesn't get rediscovered the hard way. This section only applies to US-based orgs — Census has no data for other countries, so it's skipped entirely for Latin-America-based orgs.

### Latin-America-based orgs

Set by the org's `location` input — if it's a Latin American country/city rather than a US one, the skill switches several steps:

- **Skips** peer-organization discovery (Step 2.5) — 990 data is US-tax-filing-only, so the applicant itself has no "peers" to look up this way — and Census community context (Step 6) — no data source exists for non-US geography.
- **Retargets** IRS-990/web search (Steps 2-3.5) toward finding *US-based funders whose giving reaches Latin America*, using a dedicated seed list (`references/seed-funders.md`, Section B): Inter-American Foundation, State Dept/embassy programs, Ford Foundation, Open Society Foundations, and others — cross-checked per-country, since a foundation's "Latin America program" rarely covers the whole region.
- **Adds an eligibility caveat** to federal search: most `grants-gov` postings requiring `nonprofits_non_higher_education_with_501c3` need actual IRS 501(c)(3) status, which a foreign org won't have without a US fiscal sponsor. Embassy/IAF programs are flagged as the safer direct matches since they're built for foreign applicants specifically.
- **Verifies geographic eligibility explicitly** (Step 3.8) — a dedicated check, not an afterthought. Testing on a real sample org caught two false-positive-looking matches this way: a program restricted to six specific Mexican states (not the org's actual city), and IAF's own materials stating it favors certain regions within a country over uniform national coverage. Without this step both would have been presented as clean matches.
- **`references/latam-embassy-funding.md`** — verified (not guessed) embassy funding-page URLs for Mexico, Colombia, Brazil, and Argentina, plus how to find others. Built after confirming there's no reliable URL template across countries — three guessed URLs based on Mexico's pattern all 404'd before the real ones were found via search.

### What was investigated and couldn't be added (reported honestly, not silently dropped)

- **USAID** — searching for it directly via the `grants-gov` connector, under several phrasings, surfaces nothing attributable to the agency. This may reflect USAID's real 2025 restructuring rather than a search gap, but either way there's nothing to build on right now.
- **IAF's own country pages** (`iaf.gov/country/<name>/`) return HTTP 403 specifically to `WebFetch`, even though a browser-identified request reaches the same URL fine. Unlike the Census bug, this isn't a fixable query-parameter issue — it's a static page blocked outright. No workaround found.
- **ForeignAssistance.gov** — a real, official US government API exists for historical foreign-aid spending by country/agency, which could show which agencies have actually funded a given country before. Its docs page is JS-rendered, so the actual request format and auth requirements couldn't be confirmed through fetching. Flagged as a genuine lead for future investigation, not built, since it hasn't been validated to actually work.
- **Diaspora/hometown-association funding** (e.g. Mexican "clubes de oriundos" channeling US diaspora remittances, matched by Mexico's "3x1 Program for Migrants") — real money, but relationship-driven through community networks, not a searchable database. Documented as a lead worth knowing about, not something this skill can search for.

Why IDB (Inter-American Development Bank) isn't a connector here: IDB mostly does project financing negotiated through country offices and loans to governments — there's no browsable database of open competitions the way grants.gov has one, so it doesn't fit this plugin's search-and-match architecture regardless of engineering effort.

## Notes on file layout

- `.claude-plugin/plugin.json` is the manifest Cowork actually reads to install the plugin (name, version, description).
- `manifest.json` at the plugin root is a supplementary, human-readable capability summary (inputs/outputs/data sources) — useful documentation, not read by the Cowork loader itself.
- `.mcp.json` points at the deployed `grants-gov` MCP server — update this with your own deployed URL (see Setup). `mcp.config.dev.json` points at `localhost:8000` for local testing.
- `connectors/irs-990.json`, `connectors/grants-gov.json`, and `connectors/census-acs.json` are data-source configs the skill reads at runtime (via `${CLAUDE_PLUGIN_ROOT}`) — `irs-990.json` and `census-acs.json` document public APIs called directly with WebFetch; `grants-gov.json` documents the MCP tool (primary) and its WebFetch-based fallback.

## Known limitations (v0.7.0 — by design, kept simple for testing)

- Latin-America-based org support surfaces only US-origin funding — there's no connector for any country's own domestic/government grant programs (would require per-country scraping, language localization, and very different eligibility rules per the earlier design discussion).

- IRS 990 filings lag 12-24 months behind the current fiscal year — good for confirming a funder's size and historical focus, not for confirming an open application cycle. The skill always tries to verify current deadlines via web search before presenting them.
- ProPublica's `ntee[id]` search filter is currently broken (HTTP 500 in every combination tested) — peer-organization discovery works around this with a sector keyword + `state[id]` instead, checking the NTEE code shown on each result manually.
- The seed funder list is a starting point, not exhaustive — expect to expand it after testing against real organizations.
- The `grants-gov` MCP server is a real, ongoing hosting commitment (cost, uptime, API key upkeep) — see "Who runs this" above. Its fallback (WebSearch/WebFetch against public pages) still depends on search-engine indexing and can miss opportunities the real API would surface.
- Pass-through federal money is traced one hop down to the specific administering agency (Step 3.7), but not further — a deeper chain (e.g. down to a specific county-level subrecipient) isn't implemented, and tracing doesn't always end in a confirmed deadline (see "Pass-through federal money" above for what it realistically returns instead).
- The Census connector ships with a placeholder API key — each installer needs their own (free, instant) before Step 6 will produce anything; without one, that section is silently skipped rather than broken.
- No persistence: each run is a fresh search. A future version could cache prior results or track which grants a user has already applied to.
