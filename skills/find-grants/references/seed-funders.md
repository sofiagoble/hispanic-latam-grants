# Seed funder list — Hispanic-serving & Latin American focus

Starting points for search queries, not a final or exhaustive list. Verify each one's current focus areas, eligibility, and deadlines via the IRS 990 connector and web-search fallback before including it in output — funder priorities, program names, and staff turn over.

## Section A: US-based orgs serving Hispanic/Latino communities domestically

Use these as seed terms for the IRS 990 search (organization name lookups) and as anchors for web-search fallback queries, especially when the nonprofit's sector or region isn't well covered by 990 search results alone.

- Hispanic Federation
- Hispanics in Philanthropy
- NALCAB (National Association for Latino Community Asset Builders)
- Latino Community Foundation
- W.K. Kellogg Foundation (Latino-focused initiatives)
- California Community Foundation (Latino-serving funds)
- The Chicago Community Trust (Latino-focused funds)
- Ms. Foundation for Women (Latina-led organizing grants)
- Robert Wood Johnson Foundation (health equity grants reaching Hispanic-serving orgs)
- Comcast NBCUniversal / Verizon / other corporate foundations with Hispanic Heritage Month grant cycles

## Section B: US funders active in Latin America (for `is_latam_based` orgs)

Use these when the applicant org is based *in* a Latin American country, not the US. These are US-origin funders whose giving reaches into the region — always confirm which specific countries a given program currently covers, since "Latin America program" rarely means all of Latin America.

- **Inter-American Foundation (IAF)** — small US federal agency, sole mandate is funding grassroots orgs in Latin America/Caribbean. Check `grants-gov` for open IAF opportunities. `iaf.gov`'s own country pages return HTTP 403 to `WebFetch` specifically (confirmed bug, not fixable the way the Census one was) — rely on web-search summaries of IAF's country-specific coverage instead, and note that IAF's own materials say it favors certain regions within a country (e.g. central/southern Mexico) over uniform national coverage — verify this per country before assuming the org's specific city is covered.
- **US Embassy / State Department public diplomacy small grants** — country-specific, run through "U.S. Mission to `<country>`". See `references/latam-embassy-funding.md` for verified funding-page URLs (Mexico, Colombia, Brazil, Argentina) and how to find others — don't guess the URL structure, it varies by country.
- **Ford Foundation** — dedicated Latin America and Caribbean regional office
- **Open Society Foundations** — Latin America program
- **W.K. Kellogg Foundation** — has historically funded work in Mexico specifically, alongside its US Latino-focused giving
- **Hispanics in Philanthropy** — despite the name, also regrants into Latin America, not just US Hispanic-serving orgs

**Not included, tested and found not to work:** USAID — searching for it directly (by name or spelled out) via `search_opportunities` surfaces nothing attributable to the agency under any phrasing tried. This may reflect USAID's 2025 restructuring rather than a search gap.

**Not included, structural mismatch:** Inter-American Development Bank (IDB) — mostly does project financing negotiated through country offices and loans to governments, not a browsable database of open competitions, so it doesn't fit this skill's search-and-match approach regardless of effort.

**Worth knowing about but not a connector:** diaspora/hometown-association funding channels — e.g. Mexican "clubes de oriundos" (hometown associations) channeling US-based diaspora remittances into community development back home, sometimes matched by Mexico's government "3x1 Program for Migrants." This is real money reaching Latin American communities, but it's relationship- and community-network-driven, not a searchable application process — mention it as a lead worth pursuing through personal networks if relevant to the org's community, not something this skill can search for directly.

Note the overlap: some funders (Ford, Kellogg, Hispanics in Philanthropy) appear in both sections because they fund both domestic US Hispanic-serving work and Latin America programs — check which specific program/office is relevant to the applicant, not just the parent foundation's name.
