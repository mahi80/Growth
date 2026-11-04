# Growth — Lead Intelligence for GrowthPal

A build plan for a Fable-class company-intelligence engine: a continuously refreshed
company graph, a typed signal layer, and a small fine-tuned model on top of it —
purpose-built to surface off-market M&A targets for GrowthPal's corp-dev buyers.

> **Status:** plan / pre-code. The repo is the working home for this build.
> **Owner:** @mahi80

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

Two consequences for us:

1. **The scraped corpus is the asset.** The model is a thin, replaceable layer. Budget
   engineering time accordingly.
2. **Do not start with fine-tuning.** Fine-tuning before we have ~10k labelled examples
   from real analyst usage is the classic way to burn a quarter and ship nothing.

### What GrowthPal needs out of it

GrowthPal sells *off-market acquisition targets* to corporate development teams — leads
are companies with a buy-side or sell-side signal (funding, hiring velocity, leadership
change, product launch, ownership shift), not generic company descriptions. Every
priority below is set by "does this help us rank targets and explain why".

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

We reproduce the **shape**, not the exact internals. Cloud, DB and frontend choices are
ours to make (Section 6).

---

## 3. Architecture

```
                 ┌──────────────────────────────────────────────┐
                 │  SOURCE REGISTRY (Postgres)                  │
                 │  domain · robots/ToS · cadence · licence     │
                 └───────────────┬──────────────────────────────┘
                                 │
        ┌────────────────────────▼─────────────────────────┐
        │  INGEST  Crawlee / Scrapy + Playwright           │
        │  per-domain politeness · conditional GET · queue │
        │  raw HTML + headers → object store (S3/R2)       │
        └────────────────────────┬─────────────────────────┘
                                 │ content hash per section
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
   │  STORE   Postgres (entities, edges, facts, provenance)       │
   │          pgvector/Qdrant (embeddings) · OpenSearch (facets)  │
   └─────────────────────────────┬────────────────────────────────┘
                                 │
        ┌────────────────────────▼─────────────────────────┐
        │  SIGNALS + SCORE  rule signals → learned ranker  │
        │  (funding, hiring, leadership, launches, M&A)    │
        └────────────────────────┬─────────────────────────┘
                                 │
        ┌────────────────────────▼─────────────────────────┐
        │  SERVE  hybrid search + reranker + cited answer  │
        │  API / internal UI / M&A copilot                 │
        └───────────────┬──────────────────────────────────┘
                        │ accept · reject · shortlist
        ┌───────────────▼──────────────────────────────────┐
        │  FEEDBACK LOG → eval set → ranker → (later) LoRA │
        └──────────────────────────────────────────────────┘
```

---

## 4. Data model (first cut)

Keep facts as **typed rows with provenance**, never only as chunks. This is what makes
"why did you show me this lead" answerable in one click — a hard requirement for buyers.

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
  sector        text[],
  status        text,          -- active | acquired | closed | unknown
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

-- derived, recomputed nightly
create table signal (
  id          bigserial primary key,
  company_id  uuid references company(id),
  type        text,            -- funding | hiring_surge | leadership_change
                               -- | product_launch | ownership_change | distress
  strength    numeric,
  evidence    jsonb,           -- fact ids + urls
  detected_at timestamptz
);
```

Edge table for person↔company, investor↔company, company↔company (competitor,
acquired_by). Start as Postgres edges; only move to a graph DB when traversals hurt.

---

## 5. Crawling policy and legal guardrails

Non-negotiable, because GrowthPal's buyers are regulated acquirers and will ask about
provenance.

- **Source registry is the gate.** A domain is crawled only after robots.txt is parsed,
  the ToS note is recorded, and a licence/provenance class is set
  (`open` | `api` | `restricted` | `blocked`).
- **Politeness by default:** 1–2 concurrent requests per domain, identifiable User-Agent
  with a contact URL, exponential backoff, hard daily caps.
- **No LinkedIn or similar.** Scraping it at scale breaches ToS and creates real exposure.
  Use licensed feeds or official APIs for people data.
- **Prefer official sources where they exist:** MCA (India), Companies House (UK), SEC
  EDGAR, job-board APIs, Crunchbase, funding APIs. Cheaper *and* defensible.
- **Personal data is a separate lane.** Founder/exec contact data is personal data under
  India's DPDP Act, Singapore's PDPA and GDPR. Store the minimum, record lawful basis and
  purpose, honour deletion requests, and keep a suppression list checked at query time,
  not just at ingest.
- **Retention:** raw HTML 90 days, extracted facts indefinitely, personal fields on a
  review cycle.
- **Audit trail:** every served lead can produce (field → value → source URL → timestamp
  → extraction method). Build this in phase 1; retrofitting it is miserable.

---

## 6. Build order

Deliberately front-loaded on data quality. Each phase has an exit criterion; do not start
the next one before it's met.

### Phase 0 — Foundations (week 1)
Repo skeleton, docker-compose (Postgres + Redis + object store), schema migrations, source
registry table, logging/metrics baseline.
**Exit:** one domain crawls end-to-end into a `fact` row with full provenance.

### Phase 1 — 50 sites, done properly (weeks 1–2)
Crawlee worker, readability + chunking, schema-constrained extraction for ~12 fields
(name, domain, HQ, founded, headcount band, sector, description, execs, investors,
funding events, products, careers page). SQLAlchemy models. Manual review of 200 extracted
rows.
**Exit:** ≥90% field accuracy on the reviewed sample; a human can trace any value to a URL.

### Phase 2 — Scale to 2k sites (weeks 3–6)
Queue-based scheduler, per-domain concurrency, conditional GET (ETag/Last-Modified),
section-level content hashing, incremental re-extraction, dead-source detection, tiered
model escalation with a cost ceiling per page. Entity resolution v1 (domain key + fuzzy
name/city/founder match, merge confidence, review queue). Internal search over companies.
**Exit:** 2k domains live at ≤$3 per 1,000 pages; duplicate rate under 2%; freshness lag
p95 under 7 days on tracked fields.

### Phase 3 — Signals and scoring (weeks 7–10)
Signal detectors (funding, hiring surge, leadership change, ownership change, product
launch). Rule-based score first, then a gradient-boosted ranker on it. Feedback capture
(accept/reject/shortlist) wired into the UI from day one of this phase. "Why this lead"
explanation generated from evidence facts, not from the LLM's imagination.
**Exit:** precision@10 beats the rule baseline by a measurable margin on a held-out set of
companies GrowthPal analysts have already sourced.

### Phase 4 — Retrieval quality (weeks 9–12, overlapping)
Hybrid retrieval (BM25 + dense), reranker (cross-encoder or hosted), citation-preserving
answer generation. Golden eval set of 200–500 companies with ground truth fields.
**Exit:** eval suite runs in CI; no prompt/model change ships without beating incumbent.

### Phase 5 — The weight loop (only after ~10k labels)
Distil best outputs + human-reviewed labels into a LoRA on a 7–8B base (Qwen / Llama /
Gemma class). DPO on preference pairs from the feedback log. 10–20% replay mix to avoid
catastrophic forgetting. Per-task adapters so rollback is a config change. Shadow traffic
before cutover.
**Exit:** adapter beats the prompt-only baseline on extraction F1 *and* ranking, at lower
cost per lead.

---

## 7. Eval harness (build in phase 1, not phase 4)

Metrics that matter:

| Metric | Target |
| --- | --- |
| Extraction field F1 | ≥0.90 on golden set |
| Entity duplicate rate | <2% |
| Freshness lag (p95, tracked fields) | <7 days |
| Lead precision@10 | beats analyst baseline |
| Recall vs human-sourced list | ≥0.70 |
| Cost per qualified lead | tracked weekly |
| Grounded-claim rate (answer cites real facts) | ≥0.98 |

Regression gate: nightly eval job, CI block on drop beyond tolerance.

---

## 8. Cost model (order of magnitude)

- Crawl + small-model extraction: **$0.5–3 per 1,000 pages** (proxy egress usually dominates).
- Embeddings: negligible at this corpus size.
- 7B LoRA fine-tune: cheap. **Labels and eval work are the real cost.**
- Big-model escalation on ~10% of pages is the main variable to watch; cap it per domain.

---

## 9. Risks

| Risk | Mitigation |
| --- | --- |
| Scraper drift as sites redesign | Fixture-based extraction tests per source; alert on zero-yield pages |
| Entity duplication poisoning rankings | Conservative merges + review queue + domain as key |
| Legal/ToS exposure | Registry gate, no LinkedIn, official APIs first, DPDP/PDPA/GDPR lane |
| "Continuous learning" turns into unshippable research | No weight updates before phase 5; freshness comes from data |
| Hallucinated lead rationales | Explanations generated only from cited facts |
| Enterprise buyers distrusting data | Provenance on every field, auditable per lead |

---

## 10. Open decisions

- [ ] Source mix for v1: company sites only, or add directories / job boards / news?
      (Changes crawler and entity-resolution design.)
- [ ] Hosted inference vs self-hosted small models for extraction — decide by phase 2 cost data.
- [ ] Vector store: pgvector (simpler) vs Qdrant (scale headroom).
- [ ] Who owns the analyst review queue in GrowthPal, and at what cadence?

---

## Repo layout (proposed)

```
growth/
├── README.md
├── docs/            # this plan, ADRs, source policy
├── src/growth/
│   ├── registry/    # source registry, robots/ToS gate
│   ├── crawl/       # queue, fetchers, change detection
│   ├── extract/     # readability, chunking, schema-constrained LLM
│   ├── resolve/     # entity resolution, merge rules
│   ├── signals/     # detectors, scoring
│   ├── serve/       # API, retrieval, reranker
│   └── db/          # models, migrations
├── evals/           # golden sets, nightly eval
├── infra/           # docker-compose, deploy
└── tests/
```
