---
name: find-grants
description: >
  Use when the user asks to "find grants for my nonprofit", "search for funding
  opportunities", "match grants to our mission", "who funds Hispanic-serving
  organizations", or "grants for Latin American nonprofits" — or provides a
  nonprofit's mission, location, sector, budget, and size and wants ranked
  matches. Searches private foundations (IRS Form 990 data), open federal grants
  (grants.gov), and the live web, prioritizing funders with a track record of
  supporting Hispanic-serving and Latin American nonprofits. Serves both US-based
  organizations and those based in Latin America — for the latter, surfacing
  US-origin funding (Inter-American Foundation, embassy programs, US foundations)
  that reaches the region. Optionally adds US Census demographic data to support
  a grant narrative.
metadata:
  version: "0.7.0"
---

Match a nonprofit against grant-making foundations and open federal grant opportunities, and produce one ranked shortlist.

## Step 1: Collect inputs

Require these five inputs before searching. If any are missing, ask the user directly rather than guessing:

- `org_mission` — short description of the organization's mission and programs
- `location` — city/state/country the org operates in or serves. Can be a US location, or a specific Latin American country/city (e.g. "Oaxaca, Mexico", "Bogotá, Colombia") — the org's own location determines which parts of this skill apply (see `is_latam_based` below).
- `sector` — e.g. education, health, arts, workforce development, immigration services
- `annual_budget` — approximate annual operating budget
- `nonprofit_size` — e.g. grassroots/volunteer-run, small (1-10 staff), mid-size (10-50 staff), large (50+ staff)

If `sector` or `location` is vague, ask a clarifying question rather than silently guessing an NTEE code or state filter for Step 2 — a wrong guess narrows out good candidates without anyone noticing.

**Determine `is_latam_based`:** is the org itself legally based/operating in a Latin American country, or is it a US-based org (regardless of who it serves)? This changes several steps below:

- **US-based org** (serving Hispanic/Latino communities domestically, anywhere in the US): run every step as written.
- **Latin-America-based org**: this skill has no access to that country's own local/government funding sources (no in-country database it can query) — it can only surface **US-origin funding that reaches Latin America** (US federal programs, US private foundations with LatAm giving programs). Say this limitation explicitly in the final output, not just apply it silently. Skip Step 2.5 (peer-organization discovery via 990 — the applicant itself won't appear in US tax filings, so there's no "peer" to look up this way) and Step 6 (Census — US geography only, not applicable). Steps 2-3.5 still apply, but retargeted: they're finding *US funders who give to Latin America*, not the applicant's own local peers/funders.

## Step 2: Query IRS Form 990 data (primary source)

Read `${CLAUDE_PLUGIN_ROOT}/connectors/irs-990.json` for the exact endpoints, parameters, and fields to use.

1. Use WebFetch against the ProPublica Nonprofit Explorer search endpoint with **one or two broad terms at a time** (e.g. "Hispanic" or "Latino", searched separately from a sector term like "workforce") — combining 3+ specific terms into one query tends to match zero organizations. The endpoint returns **HTTP 404 when a query matches zero organizations, not an empty list** — treat a 404 as "no results, broaden or simplify the query," not as a failed request.
2. For promising organizations returned by search, fetch the organization endpoint to pull their filing history: total revenue, total assets, and `grantsandsimilaramountspaid`. That grants-paid field is **only populated on 990-PF (private foundation) filings** — public charities and community/umbrella funders file a plain 990 and will show it empty. For those, use total revenue and total assets as the grant-making-capacity signal instead.
3. Note the filing year (`tax_prd_yr`) — 990 data lags 12-24 months, so treat it as evidence of a funder's size, capacity, and historical focus, not proof of a currently open cycle. If the most recent filing is more than ~3 years old, treat the funder as possibly defunct or inactive — verify via web search before including it, rather than assuming it's still grant-making.
4. `state[id]` filters to organizations **registered/headquartered** in that state — it does not find funders who give *to* that state. Most relevant regional and national funders are headquartered elsewhere but fund into the org's region, so don't use this filter to search for "funders serving my area"; search by name/keyword nationally instead, and confirm service area from the funder's own guidelines in Step 3.
5. Cross-check candidates against `${CLAUDE_PLUGIN_ROOT}/skills/find-grants/references/seed-funders.md`, a starting list of funder families known for Hispanic-serving/Latin American giving — use it to prime search queries, not as a final answer, since the list can go stale.

In practice, 990 keyword search is better at **validating and sizing a funder you already have a name for** than at discovering funders from scratch for a specific niche (a cause + geography combination) — expect Step 3 to surface most new candidate names, with this step confirming they're real and well-resourced.

**If `is_latam_based`:** this step still works, but for a different purpose — 990 data covers *US tax-exempt organizations*, so it can validate/size a *US-based* funder that gives to Latin America, not the applicant org itself (which won't appear in this data at all, since it isn't a US filer). Skip straight to Step 2.5's LatAm variant below rather than trying to find "peer organizations" here.

## Step 2.5: Peer-organization funder discovery

Direct keyword search (e.g. "Hispanic", "Latino") is a weak discovery method on its own — it tends to surface orgs that merely have that word in their name, not funders relevant to this specific mission and geography. Approximate a stronger signal instead:

1. **Do not use the `ntee[id]` filter** — it currently returns HTTP 500 in every combination (alone, with `state[id]`, even with a valid `q` term present). Instead, search `q=<sector keyword>&state[id]=<state>` (e.g. `q=workforce&state[id]=TX`) to find 2-4 **peer organizations**: nonprofits similar to this one — comparable scale, ideally serving Hispanic/Latino communities in a similar or nearby region. Each result still shows its NTEE code even though you can't filter by it — check that code manually to confirm sector relevance. These peers are not funders; they're organizations like the user's own.
2. For each peer, web-search `"<peer org name>" funders OR "annual report" OR "supported by"` — most nonprofits publish a funders/supporters page or an annual report listing the foundations that back them. Pull the foundation names mentioned. Expect this to come up empty for smaller/less web-visible peers — that's a normal outcome, not an error; move on to the next candidate rather than forcing a result.
3. A foundation that shows up for more than one peer is a stronger candidate than one found only via a generic keyword hit — prioritize it accordingly in Step 4.
4. Feed any new foundation names discovered this way back through Step 2's organization endpoint to confirm they're real, adequately-resourced grantmakers before treating them as candidates.

This works around the fact that ProPublica's public API has no direct "who funded organization X" query (unlike a full Schedule I/grants database) — going through a peer org's own public materials is the practical substitute.

**If `is_latam_based`, skip this whole step (1-4 above don't apply)** and instead:

1. Cross-check `${CLAUDE_PLUGIN_ROOT}/skills/find-grants/references/seed-funders.md` for the "US funders active in Latin America" section — known US foundations and federal programs with dedicated Latin America/Caribbean giving.
2. Web-search `"<foundation name>" grants "<org's country>"` for each seed candidate to confirm they're currently active in that specific country (a foundation's LatAm program may focus on a handful of countries, not the whole region).
3. Also web-search directly: `"<sector>" grant "<org's country>" foundation OR "development bank"` to surface additional US-origin funders not already on the seed list.

## Step 3: Fill gaps with web search (fallback)

Use web search when:

- 990 data doesn't surface enough candidates (fewer than 5 strong matches) — expect this often, since 990 keyword search is weak at open-ended discovery for a specific niche
- A candidate funder needs a current application deadline, RFP status, or grant-size range that 990 filings can't show
- The user's sector or region suggests a specific, well-known program (e.g. a corporate foundation's Hispanic Heritage Month grant cycle, or a local community foundation) worth checking directly

Query pattern: `"<funder name>" grant application deadline 2026 Hispanic OR Latino OR "Latin America"`. Always verify a funder's current status this way before including a specific deadline in the final output — don't present a stale or unconfirmed deadline as current.

Use exactly one of these four values for a candidate's deadline status — never guess a date or carry forward a past cycle's deadline as if it were current:

- A confirmed date (e.g. "July 31, 2026") when the funder's own site states one
- `Rolling` — no fixed deadline, accepts applications year-round
- `Invitation only` — funder states it doesn't accept unsolicited applications
- `Not stated` — web search found the funder but no deadline/cycle information

For any candidate that clears the mission/geography/size bar, also check **recipient turnover**: web-search the funder's grant announcements or "grants awarded" page for two different recent years. If it's essentially the same names both years, flag it in the output as unlikely to add a new grantee soon — a large, active funder can still be a weak prospect for this reason.

## Step 3.5: Merge candidates by organization name

Pool everything found in Steps 2 and 3 into one candidate list, merged **by organization name**:

- If a name appears in both 990 data and web search, combine them into one row: financial profile (revenue/assets/grants paid) from 990, program specifics (named grant cycle, eligibility, deadline, award size) from web search.
- If a candidate was found only via web search, look up its EIN with the 990 organization endpoint afterward (search by name first if the EIN isn't already known) to confirm it's a real, adequately-resourced funder before including it — a funder with no confirmable revenue/assets is a weak match regardless of how relevant its program sounds.
- If a candidate was found only via 990 search and web search turns up no named program, guidelines, or recent activity for it, treat it as a weak/stale match and prefer candidates with confirmed current activity.

## Step 3.6: Search federal grant opportunities (grants.gov)

**Primary method — the `grants-gov` MCP connector.** Call its `search_opportunities` tool (query/applicant_type/funding_category/opportunity_status/min_award_ceiling filters — see the tool's own description for exact parameter values) to find candidates, and `get_opportunity` for full detail on a specific one if needed. This is a real, structured, authenticated search — prefer it over the fallback below whenever it's reachable.

**Fallback method — if the MCP tool call errors, times out, or is otherwise unreachable:** don't fail the whole skill. Fall back to WebSearch + WebFetch against public grants.gov pages instead, and **say so explicitly in your final output** — a line like "Note: the live federal-grants connector was unreachable, so federal results below used the backup web-search method and may be less complete." This tells whoever is reading the results (and, if they mention it, whoever maintains the connector) that the hosted server needs attention, rather than the degradation happening silently.

Read `${CLAUDE_PLUGIN_ROOT}/connectors/grants-gov.json` for the primary tool's parameters and the fallback's URL pattern/status values.

**If `is_latam_based`:** also specifically search for the **Inter-American Foundation (IAF)** — a small US federal agency whose entire mandate is funding grassroots organizations in Latin America and the Caribbean, and **State Department "public diplomacy" small-grants programs**, which are run per-embassy for the org's specific country (e.g. "U.S. Mission to Mexico," "U.S. Mission to Colombia") rather than as one nationwide posting. Read `${CLAUDE_PLUGIN_ROOT}/skills/find-grants/references/latam-embassy-funding.md` for verified embassy funding-page URLs (Mexico, Colombia, Brazil, Argentina) and how to find others without guessing a URL pattern that likely won't work. **Eligibility caveat:** most `nonprofits_non_higher_education_with_501c3` applicant-type postings require actual IRS 501(c)(3) status, which a Latin-America-based org won't have unless it has a US fiscal sponsor or equivalency determination — don't present those as directly appliable without noting this. Embassy/IAF programs more commonly use an `other` or unrestricted applicant type specifically because they're designed for foreign organizations — those are the safer direct matches.

**Tested and confirmed not to work — don't retry these:**
- Searching literally for `"USAID"` or `"Agency for International Development"` via `search_opportunities` surfaces nothing attributable to that agency, under several phrasings. This may reflect USAID's real 2025 restructuring rather than a search gap — either way, don't present USAID as a source until this changes.
- `iaf.gov`'s own country pages (e.g. `iaf.gov/country/mexico/`) return HTTP 403 to WebFetch specifically (confirmed: a browser-identified `curl` request reaches the same URL fine) — this is a static informational page being blocked outright, not a fixable query-parameter issue like the Census bug. Get IAF's regional/state coverage for a country from web-search summaries instead, clearly caveated as secondary-source, not primary-confirmed.

Fallback steps (grants.gov's own search page is a JavaScript app that returns nothing useful to WebFetch directly, so discovery has to go through WebSearch first):

1. WebSearch a couple of phrasings, e.g. `site:simpler.grants.gov "<sector keyword>" nonprofit` and `"<sector keyword>" grants.gov opportunity 501(c)(3)`. Try more than one query — a single phrasing can easily miss relevant opportunities.
2. For each promising result, WebFetch the specific opportunity page — but **only if it's a `simpler.grants.gov/opportunity/<id>` URL**. Legacy `grants.gov/search-results-detail/<number>` links (and grants.gov's search page itself) are also a client-rendered JavaScript app and return nothing but page chrome to WebFetch, even though search engines index them. If a promising result is only a search-results-detail link or a third-party summary site, try a different search phrasing to find the `simpler.grants.gov/opportunity` page for the same program; if none turns up, present the opportunity by name/agency from the search snippet only, mark every other field `Not confirmed via direct fetch`, and don't invent a status, deadline, or amount.
3. From a confirmed `simpler.grants.gov/opportunity` page (or from the MCP tool's response), pull: title, issuing agency, current status, award ceiling/floor, post date, close date, eligible applicant types, and the application/more-info link. Keep only `Posted` (currently open) opportunities in the main output. Note a `Forecasted` one as "expected to open soon" rather than presenting it as actionable now, and drop `Closed`/`Archived` ones unless nothing else qualifies — if you do include one anyway, label it plainly as closed, not open.
4. Check eligible applicant types. If they're entirely government bodies (state/county/city/tribal agencies) with no nonprofit/501(c)(3) code, this org can't apply directly — the money passes through a state or local agency first, which then runs its own subgrant competition. Label these `Federal (pass-through only)` and continue to Step 3.7 to trace them down rather than leaving them as a bare "go figure it out" note.

## Step 3.7: Trace pass-through opportunities one hop down

For each opportunity labeled `Federal (pass-through only)` in Step 3.6, find the specific agency the org would actually need to approach — don't stop at "this passes through your state":

1. WebSearch `"<program name>" "<org's state>" administering agency` (or, if the eligible types named a specific local-level body like a workforce development board, Continuum of Care lead agency, or Area Agency on Aging, search for *that* body covering the org's specific city/county instead of the state generally). In testing, this plain search reliably surfaced the actual agency by name — e.g. "Texas Department of Housing and Community Affairs" for a CSBG posting, or the specific regional workforce board ("Workforce Solutions" via the Houston-Galveston Area Council) for a Dept. of Labor dislocated-worker program — without needing to look up the program's Assistance Listing Number (ALN) first.
2. WebFetch that agency's own site, specifically looking for a funding-opportunities/grants/NOFA page (search the agency's site for terms like "NOFA," "RFA," "subgrant," or "funding opportunities" if it's not obvious from the homepage).
3. Check dates carefully — a funding page can list opportunities that are clearly stale (e.g. labeled with a past year, no current cycle) rather than currently open. Report what you actually find plainly: a specific current opportunity with a deadline, a funding page with only past/stale postings, or a named contact for partnership/subcontracting inquiries with no formal open competition. All three are legitimate, useful outcomes — don't force a fake deadline to make the row look more complete than it is.
4. **Stop after this one hop.** If the agency found is itself another pass-through layer (e.g. it further delegates to county-level bodies), name that in the output rather than chasing it further — a deeper multi-hop chain is a possible future addition, not something this version attempts.
5. If step 1's search doesn't surface a plain agency name at all, this program's Assistance Listing Number (ALN, format `XX.XXX`) plus a search of `sam.gov/assistance-listings` or `usaspending.gov` for that ALN + the org's state can identify the prime recipient agency as a fallback — but try the direct search first, since it worked reliably without this extra step in testing.

## Step 3.8: Verify geographic eligibility precisely for `is_latam_based` candidates

Do this explicitly, as its own check, for every private-foundation or federal candidate before including it — this is the step that caught two false-positive-looking matches in testing (a program restricted to six specific Mexican states, and IAF favoring certain regions over others) that would otherwise have been presented as clean matches:

1. For any funder/program with a "Latin America" or country-level program, check whether it names specific states/regions/cities it actually covers — a country-level program name doesn't guarantee coverage of the org's specific city. IAF, for example, funds mainly in Mexico's central/southern states; it does not treat the whole country uniformly.
2. For embassy/State Department programs specifically, check for an explicit list of eligible states/regions in the program description — these are common (seen in testing: a technology-education program restricted to six named northern Mexican states) and are easy to miss if you only check the mission/eligible-applicant-type fields.
3. If geographic coverage isn't stated anywhere confirmable, say so plainly rather than assuming the org's city is included — "geography unconfirmed, worth a direct inquiry" is a legitimate, honest output, not a failure.

## Step 4: Rank matches

Score every candidate — private foundations and federal opportunities together — against the org's inputs:

1. **Mission alignment** — does the funder's/program's stated focus area overlap with `org_mission` and `sector`?
2. **Geographic eligibility** — does the funder fund in `location` (or nationally/regionally in a way that includes it)? For federal opportunities, does the org's location fall under the opportunity's eligible geography, if any is stated? If `is_latam_based`, confirm the funder/program's giving explicitly reaches the org's specific country — a US foundation's "Latin America program" doesn't necessarily cover every country in the region, and an IAF/embassy program is specific to one country by design.
3. **Size fit** — does the typical/ceiling grant size and the org's `annual_budget`/`nonprofit_size` make sense together (avoid matching a $2M-capacity foundation or an $80M federal program with grants too small to matter, or too large for a grassroots org to administer).
4. **Hispanic-serving / Latin American focus** — prioritize funders and programs with an explicit or demonstrated (via 990 grant history, peer-org funder mentions, or a federal program's stated priority population) focus on Hispanic-serving organizations or Latin American nonprofits over generalist ones, all else equal.
5. **Currently active** — deprioritize (don't necessarily drop) any private funder flagged in Step 2 as possibly defunct (stale filing) or in Step 3 as showing no recipient turnover, and any federal opportunity that's `Forecasted` rather than `Posted`.
6. **Directly appliable** — a federal opportunity labeled `Federal (pass-through only)` should rank below any directly-appliable match of similar strength even after Step 3.7's tracing, since it's still not something the org applies to on grants.gov itself.

Drop any candidate that fails geographic eligibility or is clearly mismatched on size — don't pad the list with poor fits to reach 5.

## Step 5: Output

Return 5-10 ranked matches (fewer only if genuinely not enough qualify) as a table, **one row per grant program, not per funder/agency** — an organization running several distinct programs gets a row for each:

| Rank | Type | Funder / Agency Name | Description | Eligibility | Typical Grant Size | Application Deadline |
|------|------|-----------------------|-------------|-------------|---------------------|-----------------------|

- **Type** — one of `Private Foundation` or `Federal`. Use `Federal (pass-through only)` for the case flagged in Step 3.6; put the specific agency name found in Step 3.7 in the Eligibility column (e.g. "Apply through Workforce Solutions Gulf Coast / H-GAC, not grants.gov directly"), not just "your state."
- **Application Deadline** must be one of: a confirmed date, `Rolling`, `Invitation only`, `Not stated`, `Expected to open soon` (a `Forecasted` federal opportunity), or — specifically for a traced pass-through row — `No confirmed open cycle; contact <name/office found in Step 3.7>` when tracing found only stale postings or a partnership contact rather than a live deadline. Never guess a date to fill this cell.

After the table, add one line per row noting why it's a good fit (mission/geography/size rationale), and call out plainly if a funder shows low recipient turnover, a stale 990 filing, or is pass-through-only — don't quietly drop these, since the user may still want to pursue them even without a clear direct opening. For a traced pass-through row, say explicitly what Step 3.7 found (a live subgrant deadline, a stale funding page, or a named contact) so the user knows exactly what kind of lead it is.

## Step 6: Community need context (optional, appended after the table)

**Skip this entire step if `is_latam_based`.** The Census Bureau only covers US geography — there is no equivalent data source wired up for Latin American countries in this version. Don't attempt it, and don't apologize for its absence at length; a brief "community-need context isn't available for non-US locations in this version" is enough if it comes up.

This is a distinct kind of output from the funder table above — it helps write the grant *narrative*, not find funders. Read `${CLAUDE_PLUGIN_ROOT}/connectors/census-acs.json` for the exact query workflow and its one important pitfall before using it.

1. Resolve the org's `location` to a Census geography (county or city/place) using a wildcard lookup — never construct a query combining a specific `for=` value with `in=` for the actual data fetch; always use `ucgid=` for that (see the connector's `critical_bug` note — this was confirmed broken in testing, not a hypothetical).
2. Fetch total population, Hispanic/Latino population, and poverty figures for that geography.
3. Compute the Hispanic/Latino population share and poverty rate yourself from the raw counts (the API returns totals, not percentages).
4. Append a short "Community Context" section after the funder table: 1-3 sentences citing these figures with the geography and ACS data year, e.g. "Harris County, TX is approximately 44% Hispanic/Latino (ACS 2023 5-year estimates), with a poverty rate of X% — useful context for a grant narrative's statement of need."
5. If the Census API key isn't configured (still the placeholder value) or the lookup fails, skip this section entirely rather than presenting a broken or fabricated stat — this step is optional and its absence shouldn't block the rest of the output.
