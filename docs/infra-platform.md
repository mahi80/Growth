# Infrastructure and storage

Decision record for the runtime that carries the pipeline in [README §3](../README.md#3-architecture).
Scope is phase 0–4; each component names the measurement that would make us revisit it.

## 1. Decision

**One Postgres cluster is the system of record, the vector store and the agent checkpoint store.**
`pgvector` handles embeddings. Cosmos DB is rejected as the primary store. Object storage holds raw
bytes. A queue table moves work. Nothing else is introduced before a measured trigger says so.

The reason is the query shape, not a preference for SQL. The product is joins — `fact → event →
signal → score → verification` — and the backtest is a full-cohort scan. Both are the workload a
document store is worst at and a relational database is built for.

## 2. Components

| Concern | Choice | Why | Revisit trigger |
| --- | --- | --- | --- |
| System of record | Postgres 16 | joins, constraints, PITR, one migration path | write throughput or table size beyond a single primary |
| Vectors | `pgvector` HNSW, `halfvec` | embeddings live beside the rows they describe; no dual-write | §5 triggers |
| Lexical search | Postgres FTS first | enough for company and document search at 50k–150k | ranking quality or facet latency fails the phase-4 eval |
| Fuzzy match | `pg_trgm` + blocking keys | entity resolution is a join, not a separate service | duplicate rate > 2% after tuning |
| Object store | S3 or R2 | raw HTML, fixtures, extracted text, eval artefacts | egress cost or residency |
| Queue | Postgres job table (`SKIP LOCKED`), `pgmq` if needed | transactional with the work it schedules | sustained backlog or > ~1M jobs/day |
| Agent state | LangGraph Postgres checkpointer, separate schema | same durability and backup as the facts it writes near | never — this is the point |
| Model serving | SLM gateway behind a router ([infra-extraction-tier.md](infra-extraction-tier.md)) | cost and residency policy in one place | phase-2 cost data |
| Observability | OpenTelemetry → collector → traces + metrics | per-node agent spans and pipeline SLOs | vendor choice only |
| Deployment | docker-compose locally; one VM + managed Postgres in phase 1 | the crawler is the only heavy process | phase 2 worker count or availability target |

## 3. Schema layout

One database, separate schemas so grants and backups can diverge later:

| Schema | Holds | Notes |
| --- | --- | --- |
| `core` | company, alias, external id, dependency, fact, event, signal, score, verification | the §4 model; append-only for fact and event |
| `src` | source registry, fetch log, fixtures pointer, yield counters | `last_nonzero_at` drives alerts |
| `stage` | `proposed_*` from agents | promoter validates, human approves, then writes `core` in one transaction |
| `agent` | LangGraph checkpoints and run metadata | `thread_id` is deterministic; replayable |
| `eval` | golden sets, eval runs, metric rows, backcast cohorts | read-only for production workers |
| `audit` | promotions, approval hashes, schema and ontology versions | who changed what, from which proposal |

Rules that matter more than the layout:

- Every `fact` and `event` row carries `source_url`, `source_id`, `observed_at` and `method`.
  A row without those is a bug, not a convenience.
- `company_score` is append-only across `model_version`; the backtest depends on replaying old scores.
- Backtests and backcasts read from a replica once one exists. They are scans; they must not compete
  with serving.
- Migrations are forward-only and reviewed. The ontology version and extractor contract version are
  recorded with the migration that changes them.

## 4. Why not Cosmos DB

Cosmos DB is a good globally distributed document store. It is the wrong primary store for this
product, for four reasons that are specific rather than generic:

1. **The joins are the product.** A why-now brief is a traversal from company to events to facts to
   sources. In a document model that traversal happens in application code, or is duplicated into
   denormalised documents that drift. The RDF export and the ontology checks assume a graph of
   typed rows.
2. **Backtests are scans.** Phase 0 and every quarterly backcast read a full cohort over a fixed
   window and compute rates. Request-unit pricing makes repeated large scans the expensive operation,
   and the workload is inherently repetitive.
3. **Entity resolution needs constraints.** Unique domains, alias uniqueness and merge history are
   enforced by the database today. Cosmos has no foreign keys, and its unique keys are scoped to a
   partition, so duplicate prevention becomes an application invariant that fails under concurrency.
4. **Operational surface.** Local development, migrations, PITR and SQL tooling are mature for
   Postgres. A second paradigm buys global write latency we do not need for a weekly shortlist.

Cosmos DB's vector index is not the objection — it works. The objection is that vector search is a
phase-4 retrieval feature, while joins, constraints and scans are load-bearing from phase 0. If the
company later mandates Azure, the answer is Postgres Flexible Server with `pgvector`, not Cosmos.

## 5. pgvector: where it fits and when it stops fitting

Embeddings are **not** in the detection path. Detectors read typed facts and events; retrieval reads
embeddings at phase 4. That keeps the vector index off the hot path and makes its sizing a phase-4
problem.

Sizing rule of thumb at 768 dimensions stored as `halfvec` (~1.5 KB per chunk):

| Corpus | Chunks (est.) | Index size (est.) | Fits |
| --- | --- | --- | --- |
| 50k companies × ~20 docs | ~5M | ~10–12 GB with HNSW overhead | one 32 GB primary |
| 150k companies | ~15M | ~30 GB | dedicated instance or partition |
| > 20M | — | tens of GB, and the index stops fitting beside the working set | split (§ triggers) |

Triggers to move retrieval out of Postgres, in order of preference:

1. **Tune first:** lower dimensions, `halfvec`, binary quantization with rerank, partial indexes per
   pack, and partition by company hash.
2. **Add `pgvectorscale`** (StreamingDiskANN) if the index no longer fits in memory but the data
   still belongs beside the facts.
3. **Add Qdrant** only if filtered ANN at high selectivity or p95 retrieval > 300 ms at target recall
   survives step 1. Qdrant is a retrieval service, not a system of record; facts stay in Postgres.

Re-embedding is a migration: store `embedding_model` and `chunk_version` on every vector row, and
treat a model change as a backfill with its own eval, not an in-place update.

## 6. Environments

| Environment | Shape | Purpose |
| --- | --- | --- |
| Local | compose: Postgres + pgvector, MinIO, api, worker | deterministic fixtures; no network crawls by default |
| CI | ephemeral Postgres, `evals/harness/` runner | golden sets, migration check, regression gate |
| Staging | same images, anonymised subset, object store bucket | collector and agent changes before production |
| Production | VM(s) + managed Postgres with PITR, read replica for eval | phase 1: one region, US data in the US |

- Backups: 30-day PITR on the primary; object store versioning for fixtures and raw HTML.
- Secrets: secret manager or environment injection. No token, key or DSN in the repo or in prompts.
- Residency: US company data stays in a US region. Licensed people data follows its licence, which
  may be stricter than the company data.
- Access: agents get a role that can read `core`/`src` and write only `stage`. Production workers get
  no DDL rights; migrations run as a separate role.

## 7. Failure modes

| Failure | Behaviour |
| --- | --- |
| Postgres primary down | workers stop claiming jobs; queue holds; agents fail closed, deterministic pipeline resumes on recovery |
| Checkpointer unavailable | agent graphs refuse to start; no partial `proposed_*` rows |
| Object store down | collectors stop before writing facts; registry marks the source stale |
| Vector index rebuild | retrieval degrades to lexical; scoring and serving continue |
| Queue backlog | scale workers; alert on age of oldest job, not depth alone |
| Replica lag | backtests wait; they never read the primary when a replica is configured |

The principle is the same as the agent control plane: fail closed, return `partial` with evidence,
never guess. A stale shortlist entry is suppressed by the freshness SLO rather than served with a
confident label.

## 8. Phase placement

| Phase | Infrastructure work |
| --- | --- |
| 0 | none beyond local Postgres and CSVs; the point is to avoid building this |
| 1 | compose → one VM + managed Postgres; object store; queue table; OTel; nightly eval job |
| 2 | read replica for eval; partition `event`/`fact` by month; pooler; per-domain concurrency in the queue |
| 3 | immutable score history for backtests; verification queue UI backing tables |
| 4 | pgvector + hybrid retrieval; golden retrieval set in CI |
| 5 | adapter registry; no new storage tier |

Kubernetes, a separate vector database and a streaming platform are all phase-2-or-later decisions
with named triggers. None of them is a phase-1 prerequisite.

## 9. Open questions

- Managed Postgres provider and region: RDS vs Cloud SQL vs Neon — decide on PITR, residency and
  egress cost before phase 1.
- Queue: stay on a Postgres job table, or adopt SQS once workers run in more than one region.
- Lexical search: is Postgres FTS enough, or does the phase-4 eval justify `pg_search`/OpenSearch?
- Vector hosting at 150k companies: dedicated instance vs partitioning vs `pgvectorscale`.
