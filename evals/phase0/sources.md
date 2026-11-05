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
interesting it looks.

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
| HHS OCR breach portal | | | | | low | free | |
| GitHub API | | | | | low | free | |
| GH Archive | | | | | low | free | |
| npm registry / downloads | | | | | low | free | |
| PyPI stats | | | | | low | free | |
| ATS boards (GH / Lever / Ashby) | | | | | low | free | |
| Wayback CDX | | | | | low | free | |
| Common Crawl | | | | | medium | free | |
| crt.sh | | | | | low | free | |
| USPTO (fees, assignments, TSDR) | | | | | low | free | |
| openFDA / ClinicalTrials.gov | | | | | low | free | |
| CMS PFS | | | | | low | free | |
| FedRAMP marketplace | | | | | low | free | |
| SAM.gov / FPDS | | | | | low | free | |
| FCC equipment authorization | | | | | low | free | |
| NMLS Consumer Access | | | | | **high — counsel** | free | |

## Notes for week 1

- Run EDGAR 8-K 2.01 and Crunchbase over the **same** 50 deals first; that coverage delta decides the
  licence question.
- Verify history depth with an actual query, not a changelog. Two entries are easy to get wrong:
  EDGAR full-text (2001) and GH Archive (2011).
- Record every source that required a workaround (rate limit, missing key, broken pagination) in
  `../phase0-us-data-sources.md` §13, not here.
