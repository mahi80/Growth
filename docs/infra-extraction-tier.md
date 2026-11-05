# Infra: the extraction tier

Assessment of [`mahi80/SLM-FIne-Tuned-Inference`](https://github.com/mahi80/SLM-FIne-Tuned-Inference)
against the pipeline in [README §3](../README.md#3-architecture). Written 2026-11-05, against the
repo at `main` (pushed 2026-11-05).

## 1. Verdict

**It fits the EXTRACT stage and none of the other stages.** It is a serving fabric for a narrow
fine-tuned model: gateway, policy router, vLLM on one GPU, QLoRA adapters, a golden-set release
gate, signed releases and cluster overlays. It is not an ingestion, storage, resolution or signal
platform, and it should not be adopted as one.

| Question | Answer |
| --- | --- |
| Can it serve our extraction model? | Yes — this is the best-fit piece of the repo |
| Does it crawl, parse, dedupe or resolve? | No |
| Does it store facts or compute signals? | No |
| Should we deploy the Kubernetes surface in phase 0/1? | No |
| Is the platform or the adapter reusable? | The platform; the adapter is support-ticket triage |

The useful framing: this repo is the **runtime for the model**, not the runtime for the product.
Adopt it where the pipeline says "schema-constrained LLM (tiered: small model first, escalate on
low confidence)", and keep building everything else.

## 2. What the reference repo actually contains

| Component | What it does | Status it claims |
| --- | --- | --- |
| GPU tier | vLLM serving Qwen3-4B-Instruct-2507 AWQ, multi-LoRA | Measured on an RTX 4080 laptop |
| Adapters | Two QLoRA adapters for support-ticket triage, hot-loaded | Measured; v2 passes the gate |
| Gateway | OpenAI-compatible `/v1/chat/completions`, bearer auth, per-tenant quotas, streaming, hash-chained audit | Measured; deliberately one replica |
| Router | Admits candidates on tenant, region, data class, context and breaker state, then scores; atomic policy reload; sticky canary by tenant | Measured |
| CPU / cloud tiers | Calibrated simulators, every response labelled `x-simulated: true` | **Modelled** |
| Eval gate | 24-item golden set, strict JSON + schema floor + per-field exact match + drift caps | Measured |
| Releases | Signed manifest pinning model revision, adapter digests, image digests, policy hash, gate verdict | Verified |
| Deployment | Compose locally; vanilla K8s observed on kind; OpenShift and air-gap static validation only | Mixed |

Measured numbers worth keeping in mind: **125 tok/s** at concurrency 1 and **1,123 tok/s** at
concurrency 12 on the 4B model; loading both adapters costs **15–25% of aggregate throughput**;
owned-GPU cost is **$0.117 per 1M output tokens** at 35% utilisation, with break-even against the
cloud profile at ~56k requests/day.

Two limits that matter for us:

- The **CPU and cloud tiers are simulators**. Do not capacity-plan or price against them.
- The **task contract is triage**, not extraction. `finetune/prompt.py` fixes five categories, a
  priority rubric and a 15-word summary. The platform is task-agnostic; the adapter is not.

## 3. Stage map

Coverage is roughly **1.5 of 9** stages. That is not a criticism — it is a precise fit.

| Pipeline stage (README §3) | Covered | What to do |
| --- | --- | --- |
| SOURCE REGISTRY | No | Build. Postgres table, robots/ToS gate, `expected_yield`, `last_nonzero_at`. The repo has a model **release** registry, not a source registry — different problem. |
| INGEST | No | Build. Crawlee/Scrapy + Playwright, per-domain politeness, conditional GET, object store. |
| DELTA | No | Build. Section-level hashing against the company's own baseline. |
| **EXTRACT** | **Yes** | Reuse gateway, router, vLLM, QLoRA loop, eval gate. Write a new contract and a batch layer. |
| RESOLVE | No | Build. Domain key, fuzzy name/city/founder, alias table, review queue. |
| STORE | No | Build. Postgres + provenance + embeddings; RDF export per [ontology-and-rdf.md](ontology-and-rdf.md). |
| SIGNALS + SCORE | No | Build. Detectors and ranker; the repo has no feature or scoring code. |
| SERVE | No | Build. Weekly list, why-now brief, verification queue. |
| FEEDBACK | No | Build. Verification log. The repo's audit log is an inference audit, not a label store. |
| EVAL (cross-cutting) | Partly | Reuse the gate **pattern**; replace exact-label scoring with span-grounded field scoring (§4.4). |

## 4. How to wire it into EXTRACT

### 4.1 Shape

```
                 ┌──────────────────────────────────────────────┐
  doc ready  ───▶│ extract queue (idempotency key, retries, DLQ)│
                 └───────────────────┬──────────────────────────┘
                                     │  POST /v1/chat/completions
                          ┌──────────▼───────────┐
                          │ gateway (alias:      │
                          │ extract-v1)          │
                          └──────────┬───────────┘
                                     │ policy route
              ┌──────────────────────┼───────────────────────┐
              ▼                      ▼                       ▼
      GPU vLLM adapter        CPU / batch tier        frontier backend
      (clean filings)         (statuspage, changelog) (messy HTML, low conf)
              └──────────────────────┼───────────────────────┘
                                     ▼
                          ┌──────────────────────┐
                          │ grounding validator  │  quote must exist in source
                          └──────────┬───────────┘
                                     ▼
                          Postgres facts + object-store raw output
```

The gateway already supports external backends (ADR-0009), so the "frontier fallback" is a policy
change, not new code. That is the main architectural win: extraction stays behind one API and the
model is swappable.

### 4.2 The contract

Do not reuse the triage contract. Write `extract-v1` around the ontology, with evidence attached to
every assertion:

```json
{
  "class": "FundingRound",
  "subject": {"name": "Acme", "domain": "acme.com"},
  "assertions": [
    {
      "field": "lastRoundUsd",
      "value": 12000000,
      "datatype": "integer",
      "quote": "raised $12 million in a Series A",
      "charStart": 1841,
      "charEnd": 1878
    }
  ],
  "relations": [
    {
      "name": "hasInvestor",
      "target": {"name": "Northline Capital"},
      "quote": "led by Northline Capital",
      "charStart": 1883,
      "charEnd": 1908
    }
  ],
  "unknowns": ["lastRoundAt"]
}
```

Rules that belong in the contract, not in post-processing:

- **Null over guess.** A field not stated in the document goes in `unknowns`, never in `assertions`.
- **Every assertion carries a verbatim quote and offsets.** No quote, no fact.
- **No free-text prose.** One JSON object, schema-validated.
- **Contract version is part of the adapter release**, exactly as `triage-v1` is in the reference repo.

### 4.3 Grounding validator

Cheap, deterministic, and non-negotiable:

1. Normalise whitespace and Unicode in source and quote.
2. Re-find the quote in the source; if absent, reject the assertion (do not "repair" it).
3. Re-parse `value` against `datatype`; reject on mismatch.
4. Check the class and relation names against the active ontology version.
5. Write the fact with `source_url`, `observed_at`, `extraction_method`, `confidence`.

A rejected assertion is a signal about the adapter, not a row to clean later. Track the rejection
rate per adapter version and per document class; it is the first metric that moves when a release
regresses.

### 4.4 Eval gate changes

The reference gate is right in shape and wrong in metric for us. Replace:

| Reference | Ours |
| --- | --- |
| Strict JSON | Strict JSON, plus schema version |
| Exact field match | Per-field precision / recall, per class and per pack |
| 24 golden tickets | 200–500 golden documents, stratified by source and class |
| Drift cap on exact match | Drift cap on field F1 **and** grounding rate |
| — | **Grounding rate = 100%** (a hard gate, not a target) |

Add a hard floor on **hallucinated assertions per 1,000 tokens**. On a review sample, an assertion
that is schema-valid but not in the source is worse than a missing one, because it propagates into
the dependency graph and the brief.

### 4.5 Routing by document class

| Document class | Tier | Why |
| --- | --- | --- |
| SEC filings, Form D, 8-K, S-4 | GPU adapter | Stable layout, typed fields, high volume |
| ATS JSON, package registries, statuspage | CPU / batch | Structured input, extraction is mostly parsing |
| Pricing, docs, changelog diffs | GPU adapter | Semi-structured, needs judgement |
| Messy marketing pages, news, PDFs | Frontier fallback | Ambiguity and layout variance; cost is worth it |
| Everything below the confidence floor | Frontier escalation | Never let a 4B model guess a relation |

Set a **cost ceiling per page** in policy, not in application code. README §9 phase 2 targets
≤$3 per 1,000 pages; the router is where that ceiling is enforced.

### 4.6 The batch layer the repo does not have

The gateway exposes one request/response endpoint. Extraction is a batch workload. Add:

- A queue (Celery/Arq/Temporal) with **idempotency key** `sha256(doc_id | extractor_version | schema_version)`.
- Object storage for raw input and raw output, so a bad adapter can be re-run without re-crawling.
- Retries with backoff, a dead-letter queue, and resumable checkpoints per shard.
- A worker that calls vLLM directly for bulk runs, or multiple gateway replicas with shared
  counters — the reference gateway is deliberately single-replica and keeps quotas, breakers and the
  audit chain in process.

## 5. Phase split

These follow README §9. Do not start a phase before the previous exit is met.

### Phase 0 — API only, no GPU (README phase 0)

Purpose: prove the contract and produce the labels the adapter will need.

- Write `extract-v1` and the grounding validator.
- Assemble 200–500 documents tied to the phase-0 exit and control companies, stratified across
  filings, ATS, pricing/docs and news.
- Extract with a frontier model; hand-correct a stratified sample.
- Measure per-field F1, grounding rate, cost per document, tokens per document.

**Exit:** schema validity ≥99.5%, grounding 100% by the validator, field F1 measured per class, cost
per 1,000 documents recorded. **No SLM in this phase.** A GPU here would optimise a contract that is
still changing.

### Phase 1 — one adapter, low-risk fields (README phase 1, first 5k companies)

Purpose: take the cheapest extraction off the frontier bill without touching relations.

- Label 1–2k documents from the phase-0 corpus plus phase-1 production.
- Fine-tune a QLoRA adapter on the reference `train_qlora.py` / `dataset.py` loop.
- Serve it as alias `extract-v1` with the frontier backend as fallback.
- Move only high-frequency, low-ambiguity fields: dates, amounts, headcounts, licence numbers,
  URLs, release tags. Keep relations and free text on the frontier model.
- Shadow for two weeks: run both, diff, review disagreements.

**Exit:** adapter within 3 F1 points of the frontier baseline on the moved fields, grounding 100%,
zero unreviewed relation extraction on the adapter, canary by queue/tenant with instant rollback.

### Phase 2 — batch, routing and cost ceiling (README phase 2, 50k companies)

Purpose: make the tier economical at volume and safe to change.

- Add the queue, object store, idempotency, DLQ and checkpoints (§4.6).
- Route by document class (§4.5); enforce the per-page cost ceiling in policy.
- On every ontology change: bump the contract version, re-run the golden set, retrain or re-gate,
  and ship through the signed release manifest.
- Split adapters by task if quality demands it — extraction, relation typing, classification — and
  keep canary/rollback per alias.
- Keep the gated fields on the frontier model until the adapter proves itself per pack.

**Exit:** 50k companies at ≤$3 per 1,000 pages, zero ungrounded facts in the store, per-tier cost
reported, and an adapter rollback exercised at least once in staging.

### After phase 2 — deferred, not planned

- Multi-adapter routing per vertical pack, once packs have their own labels.
- On-prem / air-gap deployment, **only if a buyer requires it**. The reference OpenShift and air-gap
  paths are statically validated, not observed; treat them as a starting point, not a proven path.
- Distillation from the feedback log (README phase 5). The reference stack is a good home for it.

## 6. Failure modes to design against

- **A 4B model inventing a relation.** One fabricated `sponsor_bank` edge re-rates every company
  that depends on it. Grounding validator or no adapter.
- **Silent truncation on long documents.** The reference policy advertises 8,192 tokens per
  candidate. Chunk with overlap and merge assertions; never truncate a filing and call it done.
- **Uncapped fallback.** If the frontier fallback fires on 30% of pages, the cost model is wrong.
  Measure the escalation rate per document class weekly.
- **Adapter / ontology drift.** Pin contract version, ontology version, adapter digest and policy
  hash together, as the reference release manifest does.
- **Planning on simulated tiers.** CPU and cloud throughput in the reference repo are modelled. Buy
  or benchmark real capacity before committing to an SLO.
- **Treating the audit chain as compliance.** It is tamper evidence, not prevention, and the head
  hash needs an external append-only sink. Buyers will still ask for provenance per fact, which the
  grounding validator and `fact` table provide — not the inference log.

## 7. Build vs borrow

| Item | Decision |
| --- | --- |
| Prompt-as-contract, versioned with the adapter | Borrow the pattern |
| QLoRA dataset generator and trainer | Borrow, replace the task generator |
| Golden-set gate, drift caps | Borrow, replace the scoring |
| Policy router, canary, breakers | Borrow |
| Signed release manifest | Borrow |
| Queue, parser, grounding validator, store, eval slices | Build |
| K8s / OpenShift / air-gap / GitOps / Kyverno | Defer until a buyer requires on-prem |

## 8. Open questions

- Base size: the reference uses 4B; README phase 5 proposes 7–8B. Decide on field F1 per dollar, not
  on parameter count, and only after phase-1 labels exist.
- One general extraction adapter with the schema in the prompt, or one adapter per ontology class?
  Start general; split only where a class shows a persistent deficit.
- Where the SLM stops: it should never be the final authority on entity resolution or on a relation
  that changes a score. Those stay deterministic or frontier-reviewed.
- Whether GrowthPal's buyers will demand on-prem inference. If yes, the reference OpenShift/air-gap
  work becomes relevant earlier; if no, keep it deferred.

## 9. References

- Reference repo: `docs/adr/ADR-0003-adapters.md`, `ADR-0004-policy-router.md`,
  `ADR-0005-benchmarks-eval-cost.md`, `ADR-0009-external-backends.md`.
- Our pipeline: [README §3](../README.md#3-architecture) and [README §9](../README.md#9-build-order).
- Our ontology and provenance: [ontology-and-rdf.md](ontology-and-rdf.md).
