# X — Sell-Side Deal Intelligence 

A build plan for a company-intelligence engine that surfaces **technical startups preparing to
sell** — a continuously refreshed company graph, a typed signal layer whose polarity is
inverted for sell-side intent, and a small model that ranks accounts worth a phone call.

> **Status:** plan / pre-code. The repo is the working home for this build.
> **Owner:** @mahi80

## Walkthrough

A six-minute narrated tour of the plan below: the sell-side inversion, the nine-stage pipeline, the
signals and hypotheses that gate them, how people and headcount movement is observed, the validity
gate, the extraction tier and the agent control plane.

https://github.com/user-attachments/assets/a11a4d45-e953-4b00-a382-9b564c7d52f8

---

## 1. The thesis (read this before writing code)

A "Fable-type LLM that continuously learns" is **90% data pipeline and 10% model**.

What these products actually are:

| Component | What it really is |
| --- | --- |
| "Continuously learning" | A crawler that re-fetches changed pages, plus an incremental index. Not weight updates. |
| "Understands any company" | Entity resolution over a canonical company graph. |
| "Reasons across messy sources" | Hybrid retrieval + reranker, with citations back to source rows. |
| "Their own SLMs" | A 7–8B base + LoRA adapters for extraction and ranking, trained on labelled in-house data. |

Three consequences for us:

1. **The labelled deal history is the asset.** For buy-side, the scraped corpus was the asset.
   For sell-side, the scarce thing is *timing* — knowing which company is available next quarter.
   Keep the corpus, but budget engineering time against labels and lead time, not page count.
2. **Do not start with fine-tuning.** Fine-tuning before we have ~10k labelled examples
   from real analyst usage is the classic way to burn a quarter and ship nothing.
3. **Do not start with coverage.** The sell-side universe is 50k–150k companies, not 4M.
   Depth and polling frequency beat breadth; see §7.

### 1.1 Why sell-side is harder than buy-side

- **Availability is deliberately hidden.** A company running a process looks healthy — that is
  the point of the process. The easy-to-scrape surface is the surface built to say "we're fine."
- **The event is rare.** Roughly 1–3% of a 100k universe enters a process in a given year.
  At that base rate, precision is the binding constraint, not recall.
- **Ground truth arrives late and partial.** You usually learn a company was for sale after it
  was sold. Labels must be reconstructed retroactively (§9, phase 0).
- **Absence is a signal.** A hiring freeze, a stalled changelog and a lapsed certificate are
  detectable only as *deltas* against a company's own baseline.
- **Naming a company as distressed is a legal and reputational surface.** Distress scores stay
  internal, evidence-cited, never published (§8).

### 1.2 What X needs out of it

X product line sells off-market M&A targets. For the sell-side motion the lead is a
**technical startup with a credible reason to exit**, found *before* it engages a banker.
Every priority below is set by "does this shorten the time between a company becoming available
and an X rep being on the phone with it".

The deliverable is not a feed. It is a short account list — a few hundred per week — each with a
"why now" brief that cites the underlying facts.

### 1.3 The polarity inversion (most important design note)

Nearly every detector in a buy-side system flips sign. Do not delete the old detectors; make them
negatively-weighted features.

| Detector | Buy-side meaning | Sell-side meaning |
| --- | --- | --- |
| Priced round closed | Strong positive (validated, funded) | **Anti-signal** — just raised, won't sell |
| Hiring surge | Positive (growing) | **Anti-signal** — unless the roles are CFO / corp-dev / strategic finance |
| CEO or CTO departure | Neutral | **Positive** — founder disengagement |
| Layoffs | Ignored | **Positive**, but usually late (§5.1) |
| Bridge / unpriced round | Ignored | **Strongly positive** — equity market said no |
| Product launch | Positive | Weakly negative — momentum, not exit |
| Ownership change | Positive (investor interest) | **Strongly positive** — secondary sale or fund wind-down |
| Careers page going quiet | Noise | **Positive** — a freeze without an announcement |

---

## 2. Reference stack (what's publicly visible)

From the public job post + press coverage summarised in the brief:

- Python, PyTorch
- Own SLMs; LLaMA / Mistral / Gemma / Phi as base-model experience
- SFT, LoRA/QLoRA, RLHF/RLAIF, DPO/IPO/ORPO
- Advanced RAG, knowledge graphs, agents, multimodal search, cross-source matching
- Quantisation, distillation, distributed GPU training, cost/latency tuning
- ~4M company profiles; signals from filings, web traffic, job boards, funding, hiring, leadership changes
- Cloud / vector DB / graph DB / agent framework / frontend: not public

We reproduce the **shape**, not the exact internals. Note that the ~4M-profile corpus described
above is the *buy-side* build. We reuse its graph and extraction machinery, but the sell-side layer
adds an inverted signal layer and deliberately narrows the universe to companies worth a call.

---

## 3. Architecture

```
                 ┌──────────────────────────────────────────────┐
                 │  SOURCE REGISTRY (Postgres)                  │
                 │  domain · robots/ToS · cadence · licence     │
                 │  · expected yield · last_nonzero_at          │
                 └───────────────┬──────────────────────────────┘
                                 │
        ┌────────────────────────▼─────────────────────────┐
        │  INGEST  Crawlee / Scrapy + Playwright           │
        │  two lanes: high-signal (daily) · profile (wkly) │
        │  per-domain politeness · conditional GET · queue │
        │  raw HTML + headers → object store (S3/R2)       │
        └────────────────────────┬─────────────────────────┘
                                 │ content hash per section
        ┌────────────────────────▼─────────────────────────┐
        │  DELTA  compare against company's own baseline   │
        │  careers count · release cadence · pricing diff  │
        │  → typed change events (this is where sell-side  │
        │    signal actually lives)                        │
        └────────────────────────┬─────────────────────────┘
                                 │
        ┌────────────────────────▼─────────────────────────┐
        │  EXTRACT  readability → chunk → schema-constrained│
        │  LLM (tiered: small model first, escalate on     │
        │  low confidence) → typed fields + provenance     │
        └────────────────────────┬─────────────────────────┘
                                 │
        ┌────────────────────────▼─────────────────────────┐
        │  RESOLVE  domain key → fuzzy match → canonical   │
        │  company_id · alias table · human review queue   │
        └────────────────────────┬─────────────────────────┘
                                 │
   ┌─────────────────────────────▼────────────────────────────────┐
   │  STORE   Postgres (entities, edges, facts, events, provenance)│
   │          pgvector/Qdrant (embeddings) · OpenSearch (facets)  │
   └─────────────────────────────┬────────────────────────────────┘
                                 │
        ┌────────────────────────▼─────────────────────────┐
        │  SIGNALS + SCORE  detectors → features →         │
        │  propensity model (anti-signals negative)        │
        │  generic (runway, decay) + vertical packs (§6)   │
        └────────────────────────┬─────────────────────────┘
                                 │
        ┌────────────────────────▼─────────────────────────┐
        │  SERVE  weekly account list + "why now" brief    │
        │  verification queue · outreach handoff · API     │
        └───────────────┬──────────────────────────────────┘
                        │ verified / rejected / no-response / in-process
        ┌───────────────▼──────────────────────────────────┐
        │  FEEDBACK LOG → backtest labels → ranker         │
        │  → (later) LoRA                                  │
        └──────────────────────────────────────────────────┘
```

The DELTA stage is the sell-side addition. Buy-side scoring reads *state* ("this company has 80
engineers"); sell-side scoring reads *rate of change* ("this company had 80 engineers in March and
51 in September").

The EXTRACT tier's model strategy — what to reuse from the SLM serving fabric, the grounding rule,
and the split between frontier and fine-tuned extraction — is in
[docs/infra-extraction-tier.md](docs/infra-extraction-tier.md).

Agents sit **above** this pipeline, not inside it. They own source discovery, collector repair,
extraction escalation, company investigation and ontology proposals; they write only to staging
tables, and deterministic workers keep the throughput. Stage ownership, graph state schemas and the
template → probe → agent cascade are in
[docs/agent-control-plane.md](docs/agent-control-plane.md).

---

## 4. Data model (first cut)

Keep facts as **typed rows with provenance**, never only as chunks. This is what makes
"why did you show me this lead" answerable in one click — a hard requirement for the reps who
call on these accounts.

```sql
-- canonical entity
create table company (
  id            uuid primary key,
  domain        text unique,
  name          text not null,
  legal_name    text,
  hq_city       text,
  hq_country    text,
  founded_year  int,
  headcount     int,
  headcount_asof date,
  sector        text[],
  stage         text,          -- seed | series_a | series_b | growth | unknown
  last_round_at date,
  last_round_usd bigint,
  universe_tier smallint,      -- 1 = deep-coverage, 2 = monitored, 3 = graph-only
  status        text,          -- active | in_process | acquired | closed | unknown
  updated_at    timestamptz default now()
);

-- alias / merge history (entity resolution is here)
create table company_alias (
  company_id  uuid references company(id),
  alias       text,
  alias_type  text,            -- name | former_name | brand | misspelling
  confidence  numeric,
  unique (company_id, alias)
);

-- external registry handles: join key for every vertical source
create table company_external_id (
  company_id  uuid references company(id),
  scheme      text,            -- nmls | chpl | fedramp | github_org | crunchbase
  value       text,
  valid_from  date,
  valid_to    date,
  primary key (company_id, scheme, value)
);

-- what the company cannot replace; also the contagion edge
create table dependency (
  company_id              uuid references company(id),
  type                    text,   -- sponsor_bank | payer | ehr_listing | marketplace
                                  -- | cloud_provider | key_customer | oss_component
  counterparty_company_id uuid references company(id),
  counterparty_name       text,
  criticality             numeric, -- 0..1
  discovered_via          text,    -- nmls_lookup | analyst | extracted_page | api
  discovered_at           timestamptz
);
create index on dependency (counterparty_company_id);

-- every extracted field, with its receipts
create table fact (
  id            bigserial primary key,
  company_id    uuid references company(id),
  field         text,          -- headcount | revenue_band | ceo | investors ...
  value_json    jsonb,
  source_url    text not null,
  source_id     uuid,          -- → source registry row
  extracted_at  timestamptz not null,
  observed_at   timestamptz,   -- when the world was like this, not when we saw it
  method        text,          -- html_rule | llm:<model>@<prompt_ver> | official_api
  confidence    numeric
);
create index on fact (company_id, field, observed_at desc);

-- typed deltas: the sell-side substrate
create table event (
  id          bigserial primary key,
  company_id  uuid references company(id),
  kind        text,            -- careers_delta | release_cadence | priced_round
                               -- | bridge_round | director_change | charge_created
  payload     jsonb,           -- {before, after, window_days}
  source_url  text not null,
  observed_at timestamptz not null
);
create index on event (company_id, kind, observed_at desc);

-- derived, recomputed nightly
create table signal (
  id          bigserial primary key,
  company_id  uuid references company(id),
  class       text,            -- deadline | distress | process_leak
                               -- | explicit_intent | technical_decay
  type        text,            -- runway_pressure | bridge_round | down_round
                               -- | leadership_exit | hiring_freeze | layoff
                               -- | release_decay | charge_created
                               -- | corp_dev_hire | secondary_listing | sunset_notice
  pack        text,            -- generic | fintech_lending | devtools_oss
  polarity    smallint,        -- +1 sell-side positive, -1 anti-signal
  strength    numeric,
  lead_time_days int,          -- assumed, then fitted from backtests
  evidence    jsonb,           -- fact ids + event ids + urls
  detected_at timestamptz
);

-- nightly ranking output; keep every version for backtesting
create table company_score (
  company_id   uuid references company(id),
  model_version text,
  propensity   numeric,        -- P(process within 12 months)
  deal_fit     numeric,        -- depends on consumer; see §13
  components   jsonb,          -- per-class contribution, for the brief
  computed_at  timestamptz,
  primary key (company_id, model_version, computed_at)
);

-- human-in-the-loop; precision depends on this table existing
create table verification (
  company_id   uuid references company(id),
  state        text,           -- queued | researching | confirmed | rejected | stale
  verified_by  text,
  verified_at  timestamptz,
  notes        text
);
```

Edge table for person↔company, investor↔company, company↔company (competitor,
acquired_by). Start as Postgres edges; only move to a graph DB when traversals hurt.

The formal class model, R2RML mapping and export surfaces are in
[docs/ontology-and-rdf.md](docs/ontology-and-rdf.md); the draft seed is
[ontology/growth-seed.json](ontology/growth-seed.json).

---

## 5. The signal layer

Sell-side intent is observable in five classes. They differ enormously in lead time and precision,
which is why the scorer needs all five rather than optimising one. These are the **generic** layer;
§5.7 adds the second-order sources that make each class cheaper to detect, and §6 covers how
verticals instantiate them through counterparties and registries.

### 5.1 Signal classes

| Class | What it looks like | Lead time | Precision | Volume |
| --- | --- | --- | --- | --- |
| **Deadline math** | Runway under ~9 months; bridge instead of priced round; investor fund at year 8+ | 6–18 mo | medium — *predictive* | medium |
| **Distress** | Layoffs, hiring freeze, exec exodus, down round, late accounts | 3–9 mo | medium — often too late | low |
| **Process leakage** | First CFO or "Head of Corp Dev" at a 40-person company; data-room vendor; banker mandate; "strategic alternatives" language | 1–4 mo | **very high** | very low |
| **Explicit intent** | Secondary-market listing, sunset/shutdown post, acqui-hire announcement | 0–3 mo | near-certain | tiny |
| **Technical decay** | Release cadence collapse, contributor churn, archived repos, lapsed cert or domain, changelog stops | 3–12 mo | medium — **cheap** | high |

Two of these carry the product. **Deadline math** is computable from filings you can already reach
legally and is the only class with enough lead time to be actionable. **Technical decay** is unique
to your segment precisely because the targets are technical — no incumbent scores GitHub release
cadence as an M&A signal, and it costs almost nothing to compute.

Distress signals are the weakest of the five. By the time a layoff is public the company has often
already retained someone, so distress is a *confirmation* feature, not a prospecting one. Design
accordingly.

### 5.2 Where each class is observable

| Class | Sources | Access |
| --- | --- | --- |
| Deadline | SEC Form D; India PAS-3 + MCA charges (CHG-1); UK Companies House charges; venture-debt announcements; funding databases | filings, public APIs, licensed |
| Distress | US state WARN notices; Layoffs.fyi / TrueUp; ADT-3 auditor resignation; DIR-12 director changes; late/dormant accounts (UK); ROC strike-off notices (IN); NCLT/IBBI (IN) | public, mostly free |
| Process leak | Job posts for CFO / corp-dev / strategic finance / M&A; boutique bankers' transactions pages; data-room vendor mentions; news phrase search | public web, fragile |
| Explicit | Caplight, Notice.co, Hiive, EquityZen, Forge (secondary listings); founder posts; shutdown lists | mixed, some licensed |
| Technical decay | GitHub public API (releases, contributors, archived repos); npm/PyPI publish + download stats; Greenhouse/Lever/Ashby public job-board JSON; crt.sh certificate history; DNS; Wayback pricing/docs diffs | public APIs, stable |

Two practical notes:

- **ATS boards are the highest-value cheap source.** Greenhouse, Lever and Ashby all expose public
  JSON per company. Poll them weekly and you get role mix and headcount trend without any scraping
  of a protected page, and without LinkedIn.
- **Follow the bankers.** Scrape the transactions pages of 50–200 boutique tech M&A advisors. Each
  announced deal gives you both a label and the pre-deal signal trail. This is the cheapest path to
  fitted feature weights, and it should be a phase-0 task, not a nice-to-have.
- **People movement is a separate capture problem.** A departure is an absence, and absences need a
  prior roster; sources split into authoritative filings, curated pages and incidental traces, with
  very different reliability. Capture design, licences and the skip list:
  [docs/people-signals.md](docs/people-signals.md).

### 5.3 Derived features

The runway model is the single most valuable computed feature and needs no sensitive data:

```
runway_months ≈ (last_round_usd × retention + disclosed_debt) / monthly_burn
monthly_burn  ≈ headcount × fully_loaded_cost(geo, role_mix) × burn_multiple
```

Every input is an estimate with wide error bars. Use the output as a **rank feature with an
uncertainty band**, never as an assertion in a brief. Two refinements matter more than precision:
bridge rounds (unpriced) should reset the clock and cut the multiple, and fund vintage should raise
hazard for backers whose fund is past its harvest window.

### 5.4 Anti-signals

Keep these as explicit negative features, not omissions. A company that just closed a $30M Series B
is not a lead, and the model needs to be told so rather than being left to infer it from the absence
of positive evidence.

Priced round within 9 months · hiring growth above 20% QoQ in engineering · major product launch ·
new market entry · headcount above the fund's exit comfort range.

### 5.5 Hypotheses to test before building detectors

Cheap priors, all testable on the historical backtest set. If these do not hold, the thesis needs
rewriting before any pipeline exists.

| # | Hypothesis | Expected hazard ratio |
| --- | --- | --- |
| H1 | Bridge/unpriced round vs priced round → sale within 18 mo | 3–5× |
| H2 | Release cadence down >50% over 6 mo | ~2× |
| H3 | Founder or director departure within 12 mo | 2–3× |
| H4 | Corp-dev / CFO hire at a sub-100-person company | 10×+ (rare) |
| H5 | Estimated runway <9 months | ~4× |
| H6 | Two or more decay signals active simultaneously | 5×+ |

### 5.6 Scoring

Rules first, then a gradient-boosted ranker on the same features. The model's job is not to be
right about every company; it is to **compress 100k companies into 300 worth a call**. Report
per-class contribution in `company_score.components` so the brief can be generated from cited
facts rather than from model output.

### 5.7 Second-order sources: entity shell, money trail, contract clock

§5.2 lists what companies **publish**. The cheaper and much less crowded tier is what they **file**:
the legal shell that gets restructured before a sale, the payroll and tax trail that dates a runway,
and the contract dates that turn a dependency into a countdown. A fourth tier is derivative — rate
of change and *silence* across surfaces we already poll.

#### A. Entity shell — highest precision, lowest competition

| Source | What it leaks | Cadence | Access |
| --- | --- | --- | --- |
| State SoS entity search / bulk | new holding entity, Articles of Merger, name change, status `forfeited`/`suspended`, registered-agent change | daily–weekly | free, bulk varies by state |
| State annual reports (CA Statement of Information, FL, NV, WA) | officer and director changes without LinkedIn — the only public board list for most private companies | annual, per company | free |
| State UCC-1 / UCC-3 | new blanket lien (venture debt, a tighter lender), assignment to a new lender, **termination on payoff** | weekly | free search, paid bulk |
| County recorder | bulk-sale notices, assignment for benefit of creditors, tax liens, fictitious business names | weekly | free, per county |
| Legal-notice newspapers | bulk sale, ABC, dissolution, name change — often 30–90 days before any press | daily | free–cheap |
| USPTO assignment + security-interest records | IP pledged as collateral, or **assigned to the acquirer before announcement** | weekly | free API |
| USPTO TSDR | trademark abandoned or not renewed — a cheap technical-decay feature | monthly | free |
| PACER / CourtListener RECAP | bankruptcy, judgments, contract and IP suits; docket entries precede press by weeks | daily | free–$0.10/page |

#### B. Money trail — dates, not adjectives

| Source | What it leaks | Cadence | Access |
| --- | --- | --- | --- |
| DOL OFLC LCA / PERM disclosure | employer FEIN, job titles, worksites, **wage levels** — a quarterly headcount proxy and a wage-cut tell | quarterly | free bulk |
| DOL EFAST2 Form 5500 | 401(k) participant counts (headcount), **plan termination**, **final return** — a final 5500 is a winding-down tell | annual | free, full-text |
| Form D / D-A | rolling closes and extensions; a bridge that never prices | event | free |
| Form ADV (IAPD) | VC fund vintage, AUM, fund count — the investor-pressure feature from §5.3 | annual | free |
| FFIEC Call Reports + bank earnings | sponsor-bank health and partner-program disclosures | quarterly | free |
| FinCEN MSB registry; Visa/Mastercard service-provider lists | payments authority lapse or removal — cannot process, cannot onboard | monthly | free, manual |
| NPPES NPI registry | clinician count and deactivations for healthtech | monthly | free bulk |

The **EIN join key** matters more than any single row above: LCA and Form 5500 both carry employer
FEIN, and NPPES carries EIN. Harvesting EIN from DOL filings gives a private-company join key that
survives name changes and DBAs — far better than fuzzy name matching.

#### C. Contract clock — turning dependencies into dates

| Source | What it leaks | Cadence | Access |
| --- | --- | --- | --- |
| USAspending / FPDS | period-of-performance end and obligation trend per vendor — a dated revenue cliff | daily | free API |
| SAM.gov | entity registration lapse: the company can no longer be awarded or paid | weekly | free |
| Legistar / Granicus council agendas | contract approval, renewal delay or cancellation in a public agenda before any press | daily | free API per city |
| State procurement portals | incumbent contract expiry and recompete dates | weekly | free |
| CMS PFS/OPPS, FDA, NIH RePORTER, SBIR | code deletion, program expiry, grant end date, trial termination | monthly | free |
| FedRAMP, StateRAMP, trust centers, PCI AOC | authorization expiry — the product becomes unsellable to the segment | monthly | free |

#### D. Distribution and technical surfaces — daily, nearly free

| Source | What it leaks | Cadence | Access |
| --- | --- | --- | --- |
| AWS/Azure/GCP marketplaces, Salesforce AppExchange, HubSpot, Slack, Atlassian, Shopify, Epic Showroom | delisting or review freeze = distribution loss | weekly | free |
| Statuspage JSON, LaunchNotes/Headway/Canny changelogs, ReadMe/GitBook docs, trust center | cadence and incident rate; a changelog that stops is a cheap decay signal | daily | free |
| Meta Ad Library, Google Ads Transparency | paid demand stops while organic decays | weekly | free–medium |
| App Store / Play review RSS | review velocity and rating trend for consumer | weekly | free |
| Certificate Transparency (crt.sh) | **`dataroom.`, `diligence.`, `board.`, `investors.` subdomains** — a diligence environment stood up before a deal, plus cert expiry | daily | free |
| GitHub org, npm/PyPI | repo archive or **transfer** to another org, maintainer exit, package ownership change, deprecation | daily | free |
| deps.dev / Libraries.io, OpenSSF Scorecard | dependent counts and maintenance flags | weekly | free |
| RDAP/WHOIS, MX/NS, registrar; BGP/ASN | domain lapse, mail migration, infrastructure withdrawal | weekly | free |

The CT-log row is the cheapest process-leak detector we have: a certificate for
`dataroom.<company>.com` is public, dated, and almost never registered by the company's own
marketing.

#### 5.7.1 The derived feature that beats any single source

Surface heartbeat. For each of ~10 surfaces (releases, package publishes, ATS postings, statuspage
incidents, changelog entries, docs diffs, ad presence, app reviews, SoS/agent updates, CT certs)
store `last_seen_at` and a 12-week observation rate per company, then compute:

```
silence(s, w) = 1 - observations(s, last w weeks) / median(observations(s, trailing 52 weeks))
```

Aggregate into a `surface_vector` and require **two or more surfaces** plus at least one deadline
feature before the decay class fires. Silence alone is ambiguous — teams go quiet for launches, for
fundraising, and for death. Silence *with* a runway under 9 months is the product.

Two consequences for the build:

- **Start polling now.** Statuspages, ATS boards, CT logs and SoS filings have no historical
  archive. Anything not polled in Phase 0 cannot be backtested; it can only be validated
  prospectively. Keep raw responses from week 1, before labels exist — the cold-start archive.
- **Store `last_nonzero_at` per source**, not just per company (§8). A source that silently returns
  zero for a week is a broken collector, not 4,000 peaceful companies.

#### 5.7.2 Buyer appetite is the other half

A sell-side signal without a buyer is an alert, not a lead. The same public filings score the buy
side: a public acquirer's 8-K 1.01 (new or upsized revolver = dry powder), 10-K language about
acquisition strategy, goodwill impairment (a motivation to buy growth), and the sector's
announced-deal cadence. Map banker and law-firm deal flow to sectors, then weight a company's score
by whether an active, funded buyer exists for its dependency profile. Two otherwise identical
companies are not equally sellable.

#### 5.7.3 What I would not add

- **LinkedIn, X, Reddit, Glassdoor, Indeed** — ToS exposure, personal-data weight, and the phase-0
  evidence says the yield is worse than ATS boards. §8 already forbids LinkedIn.
- **Traffic-estimate vendors** (Similarweb, Sensor Tower) at seed stage — the estimates are too
  noisy below ~50k monthly visits to rank on, and per-domain cost is high.
- **Marketplaces as a recall source.** Axial, Acquire.com, Flippa and Empire Flippers are precision
  lanes with heavy selection bias toward small, bootstrapped, already-public sellers. Useful for
  explicit intent; never for base rates.
- **"Founder wants to sell" inferred from social posts** — high legal exposure, low precision, and
  exactly the claim that gets a product like this written up badly.
- **Broad news crawling.** GDELT plus 50 targeted RSS feeds cover the same ground for a fraction of
  the cost.

#### 5.7.4 Five additions for Phase 0

Add these to the backtest feature set, in this order: (1) LCA headcount and wage trend, (2) final
Form 5500 and plan termination, (3) entity-shell deltas (merger, new parent, forfeiture), (4) CT-log
data-room subdomains, (5) the surface heartbeat vector. Measure each as a standalone hazard ratio
before adding it to H6 — a source that cannot beat the base rate alone will not rescue the model in
combination.

---

## 6. Vertical packs and the dependency graph

The five classes in §5 are the **generic layer**: they apply to any company. But a healthtech
startup and a lender fail for different reasons, from different counterparties, on different
clocks. This section covers where those differences live and how we encode them without writing a
new pipeline per vertical.

### 6.1 The axis is dependency, not sector

Two fintechs differ from each other more than a fintech differs from a healthtech. One depends on a
sponsor bank, the other on a card BIN; a healthtech depends on a CMS code and an EHR listing. What
ends a company is **the thing it cannot replace**, so that is what we classify on.

Every company carries three facets, all multi-label:

- `business_model` — saas | marketplace | lending | payments | devtools | hardware | services
- `end_market` — healthcare | financial_services | education | logistics | public_sector | ...
- `dependencies[]` — the facet that predicts exits:
  `sponsor_bank:Sutton`, `ehr_listing:Epic`, `aws_marketplace`, `key_customer:<company_id>`,
  `oss_component:<repo>`

Assignment is automatic where a registry handle exists (NMLS, CHPL, FedRAMP, GitHub org — see
`company_external_id` in §4) and inferred from stack and role titles otherwise. A company belongs
to as many packs as its facets imply; "vertical" is a query, not a column.

### 6.2 Signal templates

Eight templates cover essentially all vertical signals. A pack instantiates a template by naming
the counterparty type and the registry.

| Template | Detects | Fintech lending | Healthtech | Devtools / OSS |
| --- | --- | --- | --- | --- |
| `counterparty_shock` | a critical partner is in trouble | sponsor-bank consent order, BIN sponsor exit | payer contract loss, EHR listing pulled | cloud marketplace delisting, model-provider deprecation |
| `license_lapse` | regulated permission lost | NMLS money-transmitter licence surrendered | ONC CHPL certification lapsed, state licensure | rare |
| `regulatory_obsoletion` | the product's legal basis disappears | state rate caps, true-lender rules, CFPB rule | CMS PFS/HCPCS code deletion, prior-auth rule | rare |
| `program_expiry` | the funding vehicle ends | warehouse line pulled, securitisation closed | demo/grant programme ends, trial terminated | SBIR or contract-vehicle expiry |
| `distribution_loss` | channel removed | card programme terminated | Epic/Cerner listing removed, FedRAMP ATO revoked | AppExchange or AWS Marketplace delisting |
| `metric_decay` | vertical KPI falling | delinquency rising, originations falling, licence count dropping | enrolment decline, EHR installs flat | release cadence, downloads, contributor retention |
| `talent_exodus` | the roles that precede an exit leave | BSA/compliance officer quits | CMO/CMIO quits | core maintainer leaves, repo archived |
| `partial_exit` | they already sold a piece | servicing rights transferred, loan book sold | product line divested | commercial repo relicensed to BSL/SSPL |

`partial_exit` is underrated. Servicing-rights transfers, divested product lines and licence
changes are public, and are the strongest single predictor of a full sale.

`talent_exodus` needs its own capture design — authoritative rosters, licensed workforce data, and
why single departures are noise: [docs/people-signals.md](docs/people-signals.md).

### 6.3 What each vertical contributes

| Vertical | Existential dependency | Public source | Strongest signal |
| --- | --- | --- | --- |
| Fintech lending | sponsor bank, MTL, warehouse line | NMLS Consumer Access, OCC/FDIC/CFPB enforcement, bank earnings calls, ABS trustee reports | sponsor-bank consent order — every client is distressed the same day |
| Healthtech | CMS codes, ONC certification, payer contracts | CMS PFS/HCPCS updates, CHPL, HHS OCR breach portal, ClinicalTrials.gov API | pivotal trial terminated, or certification lapsed |
| Devtools / OSS | distribution and maintainers | GitHub API, npm/PyPI, Hugging Face, licence diffs | commercial repo archived or relicensed; core maintainer gone |
| B2B SaaS | channel and logo concentration | ATS role mix, app marketplaces, pricing diffs, logo wall | enterprise plan removed, sales headcount down >30% |
| Consumer marketplace | paid acquisition and liquidity | Meta Ad Library, app-store rank, Similarweb | paid spend stops while traffic decays |
| Deep tech / hardware | certifications, contracts, patents | USPTO fee status, FCC/CE, FPDS, BIS entity list | patent maintenance fee lapsed |
| Crypto | treasury, exchange listings | on-chain balances, TVL, delisting notices | on-chain treasury runway |
| Edtech | accreditation, district contracts | accreditation registers, Title IV, SARA, state procurement | accreditation probation or contract non-renewal |

The pattern: the strongest sell-side signals are **counterparty events**, not company events. That
is what makes the vertical layer cheap — monitor a few hundred counterparties (banks, EHR vendors,
marketplaces, agencies) and propagate to thousands of companies.

### 6.4 Packs are configuration, not code

One YAML per vertical, versioned with fixtures, loaded into the same detector runner as the generic
signals.

```yaml
pack: fintech_lending
version: 2026-11
assign: { business_model: [lending, payments], requires_any: [sponsor_bank, nmls_license] }

dependencies:
  - type: sponsor_bank
    discover: [nmls_filings, company_blog]
    watch:    [occ_enforcement, fdic_enforcement, bank_earnings]
    criticality: 0.9
  - type: money_transmitter_license
    discover: [nmls_consumer_access]
    watch:    [nmls_status]
    criticality: 0.8

detectors:
  - id: sponsor_bank_consent_order
    template: counterparty_shock
    when: counterparty has enforcement_action within 180d
    strength: 0.9
  - id: mtl_surrender
    template: license_lapse
    when: license.status == surrendered
    strength: 1.0
  - id: servicing_rights_transfer
    template: partial_exit
    when: news matches "servicing rights"
    strength: 0.95

anti_signals:
  - priced_round within 270d
  - warehouse_line increase > 25%

calibration: { base_rate_12m: 0.04, shortlist_slots: 80 }
brief_template: "who holds the loan book, who is the backup servicer, what is the licence footprint"
```

`template` is shared logic; `when` is a small expression over typed rows; `brief_template` becomes
the rep's call sheet. The escape hatch is a Python function for the ~10% of detectors that need
one, registered under the same `id`.

### 6.5 Dependency propagation

The dependency edge turns the company graph into a contagion model. Counterparties are scored by
the same model, so pressure flows:

```
score(c) += Σ  criticality(d) × decay(age) × exit_pressure(counterparty(d))
```

- A hospital system announcing cuts re-rates every startup selling into it, with no pack at all.
- A sponsor-bank consent order flags every client on the day it is published.
- Two hops by default, three with a decay floor. Cap the contribution so one large counterparty
  cannot swamp the company's own signals.

This is the cheapest cross-vertical feature we have, because the graph and the scorer already exist.

### 6.6 Statistical rules that make packs work

- **Never pool raw scores across packs.** A 0.8 in fintech and a 0.8 in healthtech are different
  probabilities. Use within-pack percentiles, or a hierarchical model with a per-pack intercept and
  shared feature weights. With few labels per pack, partial pooling beats per-pack models.
- **Allocate shortlist slots by measured precision, not universe size.** Thompson sampling over
  packs on verified outcomes, so a noisy new pack cannot flood the weekly list.
- **Cold-start ladder per pack.** L0 generic templates only; L1 analyst seeds dependency edges for
  200 companies; L2 automated dependency discovery from registry handles; L3 calibrated pack model
  once ~40 labels exist.
- **Entity resolution widens with every pack.** NMLS IDs, CHPL cert IDs, FedRAMP IDs and CVE vendor
  names are aliases — hence `company_external_id` in §4.

### 6.7 Ship two packs, not ten

v1 is `fintech_lending` and `devtools_oss`. Fintech has the crispest public counterparty data and
high deal volume; devtools has the cheapest signals and is the best fit for "technical startup".
Healthtech is third — strongest signals, but slower-moving, which makes it a bad first calibration
target. Order after that by deal volume × data availability, not by how interesting the vertical is.

---

## 7. Universe and coverage

Define the universe explicitly before writing collectors. Proposed v1:

- VC-backed, technical (engineering is the product), 3–10 years old
- 10–200 engineers
- India first (MCA, EPFO, GST, NCLT are cheap and defensible), then US and UK
- Excludes: pre-revenue at seed, companies already owned by a strategic, public companies

That is roughly 50k–150k companies. Tier 1 (10–20k) gets daily polling of high-signal sources and
weekly profile refresh. Tier 2 gets weekly/monthly. Tier 3 is graph-only, used for context but never
scored into a shortlist.

This is the main architectural divergence from a buy-side build: the crawler is narrower, the
re-fetch interval is shorter, and the budget goes into polling frequency and source breadth per
company rather than raw page volume.

---

## 8. Crawling policy and legal guardrails

Non-negotiable, because X's buyers are regulated acquirers and will ask about provenance.

- **Source registry is the gate.** A domain is crawled only after robots.txt is parsed, the ToS
  note is recorded, and a licence/provenance class is set (`open` | `api` | `restricted` | `blocked`).
  Add `expected_yield` and `last_nonzero_at` per source so silent breakage is caught in a day.
- **Politeness by default:** 1–2 concurrent requests per domain, identifiable User-Agent with a
  contact URL, exponential backoff, hard daily caps.
- **No LinkedIn or similar.** Scraping it at scale breaches ToS and creates real exposure. Use
  licensed feeds or official APIs for people data. ATS boards and filings cover most of the need.
- **Prefer official sources where they exist:** MCA (India), Companies House (UK), SEC EDGAR,
  job-board APIs, Crunchbase, funding APIs. Cheaper *and* defensible.
- **Distress is internal-only.** Never publish, serve externally, or assert insolvency about a
  named company. Scores are internal ranking aids with cited evidence and a review state, and
  they must not be phrased as factual claims about a company's finances.
- **Personal data is a separate lane.** Founder/exec contact data *and* inferred founder intent
  are personal data under India's DPDP Act, Singapore's PDPA and GDPR. Store the minimum, record
  lawful basis and purpose, honour deletion requests, and keep a suppression list checked at query
  time, not just at ingest.
- **Secondary-market data is licensed data.** Listing data from Caplight, Hiive, Notice, EquityZen
  or Forge is usually contractually restricted. Do not mix it into a dataset whose redistribution
  terms you have not read.
- **Retention:** raw HTML 90 days, extracted facts indefinitely, personal fields on a review cycle.
- **Audit trail:** every served lead can produce (field → value → source URL → timestamp →
  extraction method). Build this in phase 1; retrofitting it is miserable.

---

## 9. Build order

Deliberately front-loaded on validation, because the base rate is low enough that a plausible-looking
pipeline can produce a year of nothing. Each phase has an exit criterion; do not start the next one
before it's met.

### Phase 0 — Prove the signal before building it (week 1)
Define the event precisely: `process_started` (banker retained / LOI signed, from X's own
history and banker announcements), with `acquired` and `acqui-hire` as weaker labels. Reconstruct
300–500 exits from 2019–2025 plus 2,000 matched controls. Compute the §5.5 hypotheses at
T−18, T−12, T−6 and T−3.
**Plan, endpoints and week-by-week:** [docs/phase0-us-data-sources.md](docs/phase0-us-data-sources.md)
(US-only).
**Pre-code review:** [docs/pre-code-gaps.md](docs/pre-code-gaps.md) — control design, label rights,
MNPI exposure, verification sampling and serving decisions that must close before labels are
assembled.
**Working artifacts:** [labels.csv](evals/phase0/labels.csv),
[sources.md](evals/phase0/sources.md), `features.parquet`, `lift.md`, `decision.md`.
**Exit:** a lift table showing at least three detector families with ≥2× hazard over base rate, a
median lead time above 9 months, and ≥70% of exits with three feature families reconstructable at
T−12. If this fails, rewrite the thesis, do not start phase 1.

### Phase 1 — 5k companies, done properly (weeks 2–5)
Universe definition and the first 5k tier-1 companies. Collectors for GitHub/ATS/filings/charges.
Readability + chunking, schema-constrained extraction for provenance-bearing fields. Runway model
v1. Six generic detectors from the classes that survived phase 0, plus the two v1 packs (§6.7) as
config with fixtures. Weekly hand-verified shortlist of 50–100 accounts with evidence briefs. Manual
review of 200 extracted rows.
**Exit:** ≥90% field accuracy on the reviewed sample; the shortlist beats a random-company baseline
by ≥5× when an analyst checks 100 of them; each pack shows lift on its own; a human can trace any
value to a URL.

### Phase 2 — 50k companies, process leakage (weeks 6–10)
Queue-based scheduler, per-domain concurrency, conditional GET (ETag/Last-Modified), section-level
content hashing, incremental re-extraction, dead-source detection, tiered model escalation with a
cost ceiling per page. Entity resolution v1 (domain key + fuzzy name/city/founder match, merge
confidence, review queue). Process-leakage detectors (corp-dev hires, banker pages, data-room
vendors). Automated dependency discovery from registry handles, and dependency propagation (§6.5).
Internal search over companies.
**Exit:** 50k companies live at ≤$3 per 1,000 pages; duplicate rate under 2%; freshness lag p95
under 24h on high-signal sources and 7 days on profile fields; precision reported separately per pack.
**Extraction tier:** [docs/infra-extraction-tier.md](docs/infra-extraction-tier.md) — stage map,
grounding rule, and the phase 0/1/2 split for moving fields from a frontier model to a fine-tuned
adapter behind the router.

### Phase 3 — Propensity and feedback (weeks 11–14)
Gradient-boosted ranker on the phase-1/2 features, trained on the backtest set and the growing
verification log. Anti-signals wired in as negative features. Per-pack intercepts and slot
allocation by measured precision (§6.6). Verification queue wired into the internal UI from day one
of this phase. "Why this lead" explanation generated from evidence facts, not from the LLM's
imagination.
**Exit:** precision@50 on analyst-verified accounts beats the rule baseline by a measurable margin;
top decile lift ≥5× base rate within each pack; every shortlisted account has at least two
independent evidence classes.

### Phase 4 — Retrieval quality (weeks 13–16, overlapping)
Hybrid retrieval (BM25 + dense), reranker (cross-encoder or hosted), citation-preserving answer
generation. Golden eval set of 200–500 companies with ground truth fields and deal outcomes.
**Exit:** eval suite runs in CI; no prompt/model change ships without beating incumbent.

### Phase 5 — The weight loop (only after ~10k labels)
Distil best outputs + human-reviewed labels into a LoRA on a 7–8B base (Qwen / Llama /
Gemma class). DPO on preference pairs from the feedback log. 10–20% replay mix to avoid
catastrophic forgetting. Per-task adapters so rollback is a config change. Shadow traffic
before cutover.
**Exit:** adapter beats the prompt-only baseline on extraction F1 *and* ranking, at lower
cost per lead.

---

## 10. Eval harness (build in phase 0, not phase 4)

Precision, not recall, is the binding constraint. At a 1–3% annual base rate, a 20% precision@50
already implies roughly a 10× lift; target accordingly and measure against the base rate, not against
zero.

| Metric | Target |
| --- | --- |
| Top-decile lift over base rate (backtest) | ≥5× |
| Precision@50 on analyst-verified accounts | ≥20% |
| Median lead time, first signal → process start | ≥9 months |
| Recall of known 2024–25 processes | ≥0.60 |
| Share of shortlisted accounts with ≥2 evidence classes | ≥80% |
| Extraction field F1 | ≥0.90 on golden set |
| Entity duplicate rate | <2% |
| Freshness lag (p95) | <24h high-signal, <7d profile |
| Cost per verified seller lead | tracked weekly |
| Grounded-claim rate (brief cites real facts) | ≥0.98 |

Regression gate: nightly eval job, CI block on drop beyond tolerance. Backtests are re-run whenever
the universe definition changes, since that silently changes the base rate.

---

## 11. Cost model (order of magnitude)

- Crawl + small-model extraction: **$0.5–3 per 1,000 pages**. Polling 50k companies weekly on
  changed pages only is a few thousand dollars a month, dominated by proxy egress.
- Embeddings: negligible at this corpus size.
- 7B LoRA fine-tune: cheap. **Labels and eval work are the real cost.**
- Licensed people and secondary-market feeds: the main line item. Price before committing to a
  class that depends on them (§5.1, explicit intent).
- Big-model escalation on ~10% of pages is the main variable to watch; cap it per domain.
- Human verification is a real recurring cost. Budget analyst hours per shortlisted account from
  phase 1; if a rep cannot verify 100 accounts a week, the shortlist is too large.

---

## 12. Risks

| Risk | Mitigation |
| --- | --- |
| Signals have no lift; base rate defeats the model | Phase 0 backtest is the gate; do not build past it without ≥2× hazard |
| False distress assertion about a named company | Internal-only scores, cited evidence, review state, no external publishing, no insolvency language |
| Coverage bias toward loud companies | Include quiet-decay features (§5.1) and ATS/filing sources that do not depend on PR |
| Detector drift as sites redesign | Fixture-based extraction tests per source; alert on zero-yield pages via `last_nonzero_at` |
| Entity duplication poisoning rankings | Conservative merges + review queue + domain as key |
| Legal/ToS exposure | Registry gate, no LinkedIn, official APIs first, DPDP/PDPA/GDPR lane, secondary-data licence check |
| Stale data creates confident wrong lists | Freshness SLOs per source class; suppress shortlist entries past their SLO |
| "Continuous learning" turns into unshippable research | No weight updates before phase 5; timing comes from polling frequency |
| Hallucinated lead rationales | Briefs generated only from cited facts |
| Enterprise buyers distrusting data | Provenance on every field, auditable per lead |

---

## 13. Open decisions

The pre-code review lists the decisions below with their blast radius and a week-one action:
[docs/pre-code-gaps.md](docs/pre-code-gaps.md).

- [ ] **Consumer fork:** are these leads sold to *acquirers* (find sellers before they run a
      process), or used by *X's own sell-side advisory* (source mandates)? The signal layer
      is shared; `deal_fit` optimises for target-acquirer match versus fee size and willingness to
      engage. This decides the ranker's second term and should be settled before phase 3.
- [ ] Geography for v1: India-only to keep the filings advantage, or India + US from the start?
      (Changes entity resolution and the source registry considerably.)
- [ ] Who seeds the first 200 dependency edges per pack — analyst, or automated discovery followed by
      review? This gates the L1→L2 cold-start step and is the main per-pack setup cost.
- [ ] Buy licensed people data and secondary-market listings, or leave the explicit-intent class
      out of v1? (Cheapest v1 is filings + technical decay only.)
- [ ] Who owns the verification queue in X, and at what cadence? Phase 1 assumes ~100
      accounts/week.
- [ ] Hosted inference vs self-hosted small models for extraction — decide by phase 2 cost data.
- [ ] Vector store: pgvector (simpler) vs Qdrant (scale headroom).

---

## Repo layout (proposed)

```
growth/
├── README.md
├── docs/            # this plan, ADRs, source policy, event + label dictionary
├── packs/           # vertical packs (YAML) + fixtures, one directory per pack
├── ontology/        # X seed + SHACL shapes for the RDF export
├── src/growth/
│   ├── registry/    # source registry, robots/ToS gate, yield monitor
│   ├── crawl/       # queue, fetchers, change detection
│   ├── delta/       # per-company baselines, change events
│   ├── extract/     # readability, chunking, schema-constrained LLM
│   ├── resolve/     # entity resolution, merge rules
│   ├── signals/     # runway, decay, process-leak, distress detectors
│   ├── score/       # propensity + deal_fit ranker
│   ├── serve/       # API, briefs, verification queue
│   └── db/          # models, migrations
├── evals/           # golden sets, backtest harness, nightly eval
│   └── phase0/      # labels.csv, sources.md, lift.md, decision.md
├── infra/           # docker-compose, deploy
└── tests/
```
