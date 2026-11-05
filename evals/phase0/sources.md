# Phase 0 — source scorecard (working sheet)

Fill one row per source after its first real pull. Score what the source **delivered**, not what its
documentation claims.

- **Labels covered** — share of the 300–500 exit set for which this source produced an independently
  verifiable data point.
- **Fields usable** — which `labels.csv` or feature columns arrived populated, e.g. `closed_at`,
  `consideration_usd`, `headcount`.
- **Refresh** — observed cadence, not advertised cadence.
- **History depth** — earliest date actually reachable, verified with one query.
- **ToS risk** — `low` (public record / official API) · `medium` (contract ambiguous) ·
  `high` (counsel sign-off required before pipeline).
- **Verdict** — `adopt` · `defer` · `drop` · `blocked`.

Drop rule: any source below 10% label coverage is out of the phase-1 build, regardless of how
interesting it looks. Exception for no-history sources: they cannot be scored on labels until a
12-month archive exists, so score them on prospective hits and keep them in the cold-start lane.

| Source | Labels covered | Fields usable | Refresh | History depth | ToS risk | Cost | Verdict |
| --- | --- | --- | --- | --- | --- | --- | --- |
| EDGAR 8-K 2.01 | | | | | low | free | |
| EDGAR 8-K 2.05 / 2.06 | | | | | low | free | |
| EDGAR S-4 / DEFM14A | | | | | low | free | |
| EDGAR full-text search | | | | | low | free | |
| Form D | | | | | low | free | |
| Crunchbase | | | | | medium | paid | |
| PitchBook | | | | | medium | paid | |
| GDELT | | | | | low | free | |
| Google News RSS | | | | | low | free | |
| Boutique banker pages | | | | | medium | free | |
| State WARN notices | | | | | low | free | |
| Layoffs.fyi | | | | | medium | free | |
| TrueUp | | | | | medium | free | |
| CFPB enforcement | | | | | low | free | |
| OCC / FDIC enforcement | | | | | low | free | |
| DOL EFAST2 Form 5500 (final return, plan termination) | | | | | low | free | |
| DOL OFLC LCA / PERM disclosure (FEIN, wage, worksite) | | | | | low | free | |
| Form ADV (IAPD) — fund vintage, AUM | | | | | low | free | |
| FFIEC Call Reports / bank earnings | | | | | low | free | |
| FinCEN MSB registry / card-network PF lists | | | | | low | free, manual | |
| NPPES NPI registry | | | | | medium — personal data | free | |
| HHS OCR breach portal | | | | | low | free | |
| State AG breach notifications | | | | | low | free | |
| OSHA / NLRB filings | | | | | low | free | |
| State SoS entity status (merger, status, agent) | | | | | low | free | |
| State annual reports (CA SI / FL / NV / WA) | | | | | medium — personal data | free | |
| State UCC-1 / UCC-3 | | | | | medium | free–paid | |
| County recorder (bulk sale, ABC, liens) | | | | | medium | free | |
| Legal-notice newspapers | | | | | low | free | |
| PACER / CourtListener RECAP | | | | | low–medium | free–paid | |
| State franchise-tax status | | | | | low | free | |
| GitHub API | | | | | low | free | |
| GH Archive | | | | | low | free | |
| npm registry / downloads | | | | | low | free | |
| PyPI stats | | | | | low | free | |
| ATS boards (GH / Lever / Ashby) | | | | | low | free | |
| Wayback CDX | | | | | low | free | |
| Common Crawl | | | | | medium | free | |
| crt.sh | | | | | low | free | |
| Statuspage JSON | | | | | low | free | |
| Changelog / docs feeds (LaunchNotes, Headway, ReadMe) | | | | | low | free | |
| Marketplace listings (AWS, Azure, GCP, AppExchange, HubSpot, Slack, Atlassian, Shopify, Epic) | | | | | medium | free | |
| Meta Ad Library / Google Ads Transparency | | | | | medium | free | |
| App Store review RSS | | | | | low | free | |
| RDAP / WHOIS, MX/NS | | | | | low | free | |
| deps.dev / Libraries.io / OpenSSF Scorecard | | | | | low | free | |
| USPTO (fees, assignments, TSDR) | | | | | low | free | |
| USPTO TTAB | | | | | low | free | |
| openFDA / ClinicalTrials.gov | | | | | low | free | |
| CMS PFS | | | | | low | free | |
| FedRAMP marketplace | | | | | low | free | |
| SAM.gov / FPDS | | | | | low | free | |
| USAspending (period of performance) | | | | | low | free | |
| Legistar / Granicus council agendas | | | | | low | free | |
| State procurement portals | | | | | low | free | |
| Federal Register API | | | | | low | free | |
| NIH RePORTER / SBIR | | | | | low | free | |
| FCC equipment authorization | | | | | low | free | |
| NMLS Consumer Access | | | | | **high — counsel** | free | |
| Axial | | | | | medium | membership | |
| Acquire.com / Flippa / Empire Flippers / DealStream | | | | | medium | free–paid | |
| VC portfolio pages (exited / removed) | | | | | low | free | |
| YC / Techstars directory status | | | | | low | free | |

## Derived, not a source

- **Surface heartbeat / silence vector** — computed per company from the rows above (`last_seen_at` and
  a 52-week baseline per surface). Not a source to pull; it is the feature that makes the cheap
  sources pay. Require ≥2 silent surfaces plus one deadline feature before the decay class fires.
- **EIN** — harvest from Form 5500 and LCA (employer FEIN); use as the join key across NPPES, SAM
  (UEI), and state filings. Better than name matching through renames and DBAs.

## No-history sources

Statuspage, ATS boards, CT logs, SoS filings, UCC, and marketplace listings have no usable archive.
They cannot be backtested, only validated prospectively — start polling in week 1 and keep raw
responses before labels exist.

Marketplace rows are **precision lanes**, not recall: heavy selection bias toward small, bootstrapped
sellers. Use them for explicit intent; never calibrate base rates from them.

## Notes for week 1

- Run EDGAR 8-K 2.01 and Crunchbase over the **same** 50 deals first; that coverage delta decides the
  licence question.
- Verify history depth with an actual query, not a changelog. Two entries are easy to get wrong:
  EDGAR full-text (2001) and GH Archive (2011).
- Check state SoS bulk availability early — the legal-shell signal is only cheap in the states that
  publish bulk files; per-query portals do not scale to 100k companies.
- Sample 50 CT-log hits for `dataroom.` / `diligence.` subdomains before trusting the process-leak
  detector; expect false positives from vendors that host on a customer subdomain.
- Record every source that required a workaround (rate limit, missing key, broken pagination) in
  `../phase0-us-data-sources.md` §13, not here.
