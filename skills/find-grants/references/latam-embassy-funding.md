# US Embassy funding pages — Latin America

For `is_latam_based` orgs: each US embassy runs its own small-grants/public-affairs program, and **each country's site has a genuinely different URL structure** — there is no reliable `{country-code}.usembassy.gov/funding-opportunities/` template to guess from (confirmed by testing: that exact path worked for Mexico, but 404'd for Colombia, Brazil, and Argentina, which each use a different path).

## Verified URLs (checked directly, not guessed)

- **Mexico**: `mx.usembassy.gov/funding-opportunities/`
- **Colombia**: `co.usembassy.gov/opportunity-fund-program-colombia-2025-26/` and `co.usembassy.gov/education/missions-public-affairs-section-small-grants-program-u-s-embassy-in-colombia/`. Note: as of the last check, a secondary source reported "no funding opportunities at this time" as of December 2023 — **re-verify current status before presenting this as open**, don't assume it's reopened.
- **Brazil**: `br.usembassy.gov/embassy-consulates/grants-corner/` (a general "Grants Corner" hub, not a single fixed program page — check this section for whatever's currently posted)
- **Argentina**: `ar.usembassy.gov/education-culture/programs/public-affairs-section-small-grants-program/`

## For any country not listed above

Don't guess the URL structure. Instead: `WebSearch "<country-code>.usembassy.gov" grants OR "funding opportunities" OR "small grants"` (e.g. `"pe.usembassy.gov" grants` for Peru) to find that country's actual page, then confirm it resolves before citing it. This is slower than a lookup table, but a wrong guessed URL is worse than an extra search.

## Why this matters

Search engines rank secondary aggregator sites (fundsforNGOs, grantedai, and similar) above the embassy's own primary page for most of these programs — which is exactly how an unconfirmed "$500K FY26 nationwide" claim showed up in testing that couldn't be verified against the real `simpler.grants.gov` listing. Going to the embassy's own page directly, once you have the right URL, is more reliable than trusting what ranks first in search results.
