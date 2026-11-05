# Agent control plane

Where LangGraph-style agents fit in the pipeline in [README §3](../README.md#3-architecture), what
they are allowed to write, and the maths for keeping them off the hot path.

## 1. Thesis: two planes

The pipeline has a **data plane** and a **control plane**. They have different requirements and
should not share a runtime.

| | Data plane | Control plane |
| --- | --- | --- |
| Work | fetch, parse, hash, extract, resolve, score | decide what to fetch, repair broken sources, investigate a company, propose ontology changes |
| Volume | 50k–150k companies, millions of documents | thousands of runs at most |
| Property needed | determinism, idempotency, replayability | adaptivity, tool use, judgement |
| Runtime | queue workers + batched model calls | LangGraph graphs with checkpoints and human interrupts |
| Failure mode | retry the shard | return `partial` with evidence, never guess |

Agents are the control plane. They decide, propose and escalate. Deterministic workers do the
throughput. A general "planner" that owns the whole pipeline is the wrong abstraction: it cannot be
replayed, cannot be backtested against the Phase 0 set, and turns a cost model into a distribution.

**Hard rule:** an agent never writes to `fact`, `ontology`, `source` or `company` directly. It writes
to `proposed_*` staging tables. A deterministic promoter validates, and a human approves.

## 2. Stage ownership

| Pipeline stage | Owner | Agent role |
| --- | --- | --- |
| SOURCE REGISTRY | Hybrid | `source_discovery` proposes rows; templates and probes do the bulk |
| INGEST | Deterministic | `collector_repair` on yield drop or parse failure |
| CLEAN / PARSE | Deterministic | none — an LLM rewrite breaks quote offsets |
| DELTA | Deterministic | none |
| EXTRACT | Deterministic, agent escalation | `extraction_escalation` for low confidence / rejected / malformed |
| RESOLVE | Deterministic, human review | agent-assisted match candidates for ambiguous cases |
| STORE | Deterministic | none |
| SIGNALS + SCORE | Deterministic | none |
| SERVE | Hybrid | `company_investigation` + verification for the weekly shortlist |
| FEEDBACK | Deterministic log | agent mines failure clusters for eval sets |
| ONTOLOGY | Human-owned | `ontology_proposal` proposes; CI and a human approve |

## 3. The five graphs

Each graph is small, typed, and triggered by an event. There is no meta-planner.

### 3.1 `source_discovery`

Trigger: new company, or profile older than 90 days. Purpose: find the official surfaces the
deterministic collectors will poll.

```python
class SourceCandidate(TypedDict):
    kind: Literal["ats", "github", "statuspage", "changelog", "trust",
                  "marketplace", "sos", "pricing", "docs"]
    url: str
    evidence_url: str          # where the link was found
    confidence: float
    discovered_by: Literal["template", "probe", "agent"]

class SourceDiscoveryState(TypedDict):
    company_id: str
    name: str
    domain: str
    tried: Annotated[list[str], operator.add]
    candidates: Annotated[list[SourceCandidate], operator.add]
    tokens_used: int
    calls_used: int
    status: Literal["running", "complete", "partial", "failed"]
```

Nodes: `load_company → template_scan → probe → classify → [agent_probe] → validate → stage`.
`agent_probe` is a conditional edge reached only when the first three stages leave a required source
kind unresolved. `validate` checks HTTP reachability, robots/ToS from the registry, and that the
candidate is the company's own property (not a competitor's or a directory listing).

Exit: staged rows with evidence, or `partial` listing the kinds it could not find. Precision is
measured on a 100-company hand-labelled set, per kind.

### 3.2 `collector_repair`

Trigger: `expected_yield` falls below threshold, parse-failure rate spikes, or `last_nonzero_at` is
stale. Purpose: turn a broken selector into a reviewed change.

```python
class RepairState(TypedDict):
    source_id: str
    domain: str
    kind: str
    last_good_fixture: str          # object-store key
    current_fixture: str
    expected_fields: list[str]
    attempts: int
    proposal: dict | None           # selector / parser / schema
    test_result: dict | None
    approval: Literal["pending", "approved", "rejected"]
```

Nodes: `load_fixtures → diff_structure → propose_patch → test_on_fixtures → open_pr →
await_approval` (LangGraph `interrupt`). The proposal is a diff against the collector, with the
fixture that proves it. A human merges; the agent never edits a live collector.

Exit: a PR that passes the fixture suite plus the source's last 30 days of stored responses, or a
`blocked` verdict that quarantines the source instead of silently returning zero.

### 3.3 `extraction_escalation`

Trigger: grounding validator rejection, low confidence, or a parse failure. Purpose: resolve the
tail without weakening the gate.

```python
class EscalationState(TypedDict):
    doc_id: str
    source_url: str
    text_ref: str                   # object-store key, never inline text
    schema_version: str
    reason: Literal["low_confidence", "rejected", "parse_failed"]
    attempts: int
    assertions: list[dict]
    unknowns: list[str]
    validator_result: dict | None
    tokens_used: int
```

Nodes: `load_doc → read → [search_similar] → extract → validate → [retry_once] → stage`.
Tools are read-only: fetch through the source-registry gate, read the stored document, query the
graph for prior facts, run the validator. A second failure produces `unknowns`, not a retry loop.

Exit: an assertion set that passes the same validator as the deterministic extractor, or an explicit
`unknown`. The escalation path is never allowed to skip grounding.

### 3.4 `company_investigation`

Trigger: the weekly shortlist (~300 accounts). Purpose: produce the why-now brief from cited facts,
not from model memory.

```python
class InvestigationState(TypedDict):
    company_id: str
    question: str
    evidence: Annotated[list[dict], operator.add]   # {claim, quote, url, observed_at}
    conflicts: Annotated[list[dict], operator.add]
    draft: str | None
    tokens_used: int
    calls_used: int
```

Nodes: `load_signals → gather_evidence → reconcile → draft → verify_claims → stage`.
`verify_claims` re-checks every quote against the stored document and rejects unmatched claims. The
brief is staged, not sent.

Exit: every claim has at least one verbatim span; conflicts are listed in the brief rather than
smoothed over; a human reviews before outreach.

### 3.5 `ontology_proposal`

Trigger: weekly cluster of unmapped fields or entity mentions above a threshold. Purpose: keep the
ontology current without letting it drift under the extractor.

```python
class OntologyProposalState(TypedDict):
    cluster_ref: str
    ontology_version: str
    proposal: dict | None           # classes, properties, relations, parents
    validation: dict | None         # Pydantic + cycle + SHACL results
    migration_note: str
    approval: Literal["pending", "approved", "rejected"]
```

Nodes: `cluster_unmapped → search_graph → propose → validate → write_migration_note → open_pr →
await_approval`. Validation runs the GraphBuilder model checks from
[ontology-and-rdf.md](ontology-and-rdf.md) §4: name regexes, duplicate detection, unknown
parents/domains/ranges, cycles, and the SHACL export.

Exit: a versioned PR with examples and a migration note (which extractor contract version, whether a
backfill is needed). The agent may add classes and properties; renaming or deleting requires a
separate human-authored change.

## 4. Harness requirements

LangGraph gives you the graph, the checkpointer and interrupts. Everything below is yours.

- **Durable state.** Postgres checkpointer, `thread_id` = deterministic key
  (`source_discovery:{company_id}:{run_date}`). That makes a re-run idempotent and gives you replay.
- **Budgets in code, not prompts.** Wrap every model/tool call in a guard that decrements a token and
  call budget. On exhaustion, return `partial` with what was verified. A prompt-level instruction is
  not a budget.
- **Tool allowlist.** No shell, no arbitrary browser. `fetch` consults the source registry and honours
  robots/ToS and per-domain concurrency; the agent cannot bypass the gate by calling a raw HTTP tool.
- **Read/write separation.** Read tools hit Postgres and object storage. Write tools only append to
  `proposed_*`. A deterministic promoter validates and a human approves; the approval is recorded
  with the proposal hash.
- **PII handling.** SoS, recorder and enforcement sources contain personal names and addresses.
  Redact before the agent context; keep the raw record in storage with its provenance. Do not let
  an agent infer officer intent or personal circumstances.
- **Traces and evals per graph.** OTel spans per node and tool; a golden set per graph; metrics for
  success rate, precision, cost per success and p95 wall time. Rerun the eval on every tool or prompt
  change, and every model change.
- **Model routing.** Call the SLM gateway from
  [infra-extraction-tier.md](infra-extraction-tier.md) by alias, not by provider SDK: cheap model for
  classify/validate nodes, stronger model for planning and investigation, frontier fallback. The
  router's policy then owns cost and residency.
- **Concurrency.** Cap agent runs per domain and globally. An agent retry storm against one host is
  a ToS problem, not just a cost problem.

## 5. Cascade maths

Do not run an agent per company. Run a cascade:

| Step | Coverage | Cost per company | Volume at 50k |
| --- | --- | --- | --- |
| 1. Path/slug templates (`boards.greenhouse.io/{slug}`, `jobs.lever.co/{slug}`, `{slug}.ashbyhq.com`, `github.com/{org}`, `/trust`, `/changelog`) | ~75% | ~10 HTTP probes, <$0.001 | 37,500 |
| 2. Probes + classifier on remaining candidates | ~15% | ~30 probes + 1 cheap model call | 7,500 |
| 3. Agent tail | ~10% | ~15 tool calls, ~30k in / 3k out tokens | 5,000 |

Agent cost at the tail:

- 4B on the owned GPU: effectively fixed capacity, under $0.01 per run.
- Frontier at ~$0.15/$0.60 per 1M tokens: ~$0.006 per run → **~$30 one-off for 50k companies**.
- Frontier at ~$3/$15 per 1M tokens: ~$0.135 per run → **~$680 one-off for 50k companies**.

Running the agent on every company instead: 50k × $0.135 ≈ **$6,800**, ten times the cascade, for
the same output on the 90% the templates already resolve. Refresh is ~10% of the universe monthly,
so steady state is 10× cheaper again.

Investigation is affordable because volume is bounded: 300 briefs/week × ~20 tool calls × ~20k in /
2k out is ~$30/week at frontier pricing, or near zero on the owned GPU. The budget that matters is
attention, not tokens.

The cascade also fixes the evaluation problem: you can measure template hit rate and probe precision
separately from agent precision, so an agent regression does not hide inside a blended number.

## 6. Anti-patterns

1. **One meta-planner.** It cannot be replayed, and every failure becomes a prompt-debugging session.
   Bounded graphs with explicit nodes fail loudly at a known edge.
2. **An LLM cleaning stage.** Rewriting text invalidates character offsets and therefore the grounding
   validator. Cleaning stays deterministic (trafilatura/readability, PDF text extraction, table parsers).
3. **Agents in the per-document hot path.** 10–50× the cost of batched extraction, worse latency, and
   no per-field eval. Escalate only the tail.
4. **Agent writes to production tables.** Staging plus promoter plus approval, or the ontology and
   the fact table become unreviewable.
5. **Unbounded retries.** Two attempts, then `unknown`. A loop that keeps trying is a budget bug with
   a narrative.
6. **Ontology auto-evolution.** Changing the classes invalidates historical labels and extractor
   versions. Propose, migrate, version, approve.

## 7. Phase placement

| Phase | Agents | Why |
| --- | --- | --- |
| 0 (README) | none | Build the extraction contract, golden sets and eval harness they will be judged by |
| 1 (README) | `source_discovery`, `collector_repair` | Pay for themselves immediately on 5k companies; source maintenance is the biggest hidden cost |
| 2 (README) | `extraction_escalation` | Only after the deterministic extractor and validator exist and the batch layer is in place |
| 3 (README) | `company_investigation`, verification | The shortlist is where open-ended research has direct revenue value |
| 4–5 (README) | `ontology_proposal`; agent traces become adapter training data | Clusters are only real once the graph has volume; traces are the best label source |

## 8. Open questions

- Whether LangGraph Platform is worth it versus a self-hosted Postgres checkpointer plus a small
  trigger service. Start self-hosted; the graph code is portable.
- Whether `source_discovery` should be per company or per domain cluster (shared ATS vendor, shared
  parent) — clustering could cut agent runs further.
- Whether `collector_repair` should open PRs or only stage patches. PRs are more honest; staging is
  less noisy. Decide once source count is stable.
- Which graph owns conflict resolution when two sources disagree on a fact. Proposal: deterministic
  precedence by source class, agent only annotates the conflict for the reviewer.

## 9. References

- Pipeline and stage ownership: [README §3](../README.md#3-architecture), [README §9](../README.md#9-build-order).
- Model serving and routing: [infra-extraction-tier.md](infra-extraction-tier.md).
- Ontology validation and SHACL: [ontology-and-rdf.md](ontology-and-rdf.md).
- Source registry and yield monitoring: [README §8](../README.md#8-crawling-policy-and-legal-guardrails).
