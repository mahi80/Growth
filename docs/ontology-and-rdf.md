# Ontology and RDF for captured data

> **Status:** draft. Seed at [`ontology/growth-seed.json`](../ontology/growth-seed.json).
> **Companions:** README §4 (data model), §6 (vertical packs), §9 phase 0.
> **Decision:** adopt [GraphBuilder](https://github.com/mahi80/GraphBuilder) as the ontology and
> export layer. Do not adopt its document pipeline.

---

## 1. Why an ontology, and why now

The Postgres schema in README §4 is the system of record. It is good at writes, ranking and
point-in-time queries, and bad at two things we need:

1. **Explaining a lead as a graph.** "Why is this company on the list" is a subgraph — company →
   signal → evidence → source, plus dependency → counterparty. In SQL that is four joins and a
   hand-built serializer; in RDF it is a neighborhood query.
2. **Interchange.** Buyers, data partners and lawyers ask for the evidence trail in a standard
   shape. OWL + SHACL + PROV-O answers that without a bespoke export format.

Doing this after 50k companies means retrofitting provenance into a system that did not model it.
Doing it now costs about a week.

---

## 2. What we take from GraphBuilder

| Module | Use |
| --- | --- |
| `ontology/model.py` | Versioned ontology: PascalCase classes, camelCase typed properties, relations with a `view` |
| `ontology/export.py` | `to_owl()` → Turtle, `to_shacl()` → validation, `instances_to_rdf()` with PROV-O |
| `structured/r2rml.py` | Postgres → W3C R2RML → materialize to RDF, or virtual SPARQL via Ontop |
| `ontology/catalog.json` | `schema-org` and `fibo-parties` packs for IRI alignment |
| `ontology/seeds/bfsi.json` | Template for our seed format |
| `export/okf` | Markdown bundle, one page per entity — the verification brief format |
| `graph/*` | Memory / Gremlin / Neptune; **deferred** until traversal outgrows a recursive CTE |

## 3. What we do not take

- **`pipeline/`, `induce`, `autobuild`.** Document-centric: PDFs and DITA in, LLM-extracted
  candidates out. Our facts arrive typed from collectors. Routing them through an LLM extractor
  would add cost, latency and a new failure mode for zero benefit.
- **`api_rag`, `api_structured`'s OpenMetadata path, `monitor`.** Their retrieval and monitoring
  stack duplicates what Growth builds.
- **The graph database, for now.** Two- and three-hop contagion is a recursive CTE on 150k rows.
  Revisit only if README §9 phase 2 shows traversal latency.

---

## 4. Class model

22 classes, single inheritance, validated against `Ontology.model_validate` (22 classes, 38 relations).

```
Thing
└── Entity                        displayName, reviewState, validFrom, validTo
    ├── Organization              legalName, domain, hqCity, hqCountry, foundedYear, headcount, headcountAsOf
    │   ├── Company               stage, status, lastRoundUsd, lastRoundAt, universeTier,
    │   │                         businessModel, endMarket, primarySector
    │   └── Investor              fundVintage, investorType
    ├── Person                    fullName, roleTitle, departureDate
    ├── Sector                    sectorName, taxonomy
    ├── Source                    sourceName, baseUrl, licenceClass, refreshCadence, lastFetchedAt
    ├── ExternalIdentifier        idScheme, idValue, issuer
    ├── Dependency                dependencyType, criticality, discoveredVia, discoveredAt
    ├── Permission                permissionScheme, permissionStatus, expiresAt
    │   ├── Licence               licenceNumber
    │   └── Certification
    ├── Signal                    signalClass, signalType, polarity, strength, leadTimeDays,
    │                             verticalPack, detectedAt
    ├── Score                     modelVersion, propensity, dealFit, computedAt
    ├── Transaction               announcedAt, closedAt, amountUsd
    │   ├── FundingRound          roundType
    │   └── Deal                  dealType, processStage
    └── Observation               observedAt, sourceUrl, extractionMethod, confidence, summary
        ├── Event                 eventKind
        ├── Evidence              quotedText
        └── Assertion             assertionField, assertionValue
```

Three design notes:

- **`Organization` and `Person` carry schema.org IRIs.** Subclassing `schema:Organization` gives
  interop with the largest open vocabulary at zero import cost. Remove the `iri` fields if legal
  prefers an isolated namespace.
- **`Entity.validFrom` / `validTo` sit on the base class.** Every entity is bitemporal, not just
  licences. It is the cheapest place to put it and the hardest thing to add later.
- **No property name is defined on two unrelated classes.** GraphBuilder writes one `rdfs:domain`
  per property definition, so a duplicated name silently creates an OWL intersection. Shared
  properties live on the nearest common parent — that is why `expiresAt` is on `Permission`, not on
  `Licence` and `Certification` separately.

## 5. Relations and `view`

38 relations, all with declared inverses except `competesWith` and `counterparty`. The `view` field
is the part worth using deliberately:

| View | Relations | Why |
| --- | --- | --- |
| `temporal` | `hasRound`, `hasEvent`, `acquiredBy`, `employs`, `dealTarget`, `dealAcquirer` | Valid only inside a window; drives point-in-time queries |
| `causal` | `hasSignal`, `derivedFrom`, `hasScore` | Explains why a lead surfaced |
| `entity` | `hasDependency`, `counterparty`, `hasLicence`, `hasExternalId`, `hasInvestor` | Structural facts about the company |
| `semantic` | `fromSource`, `operatesIn` | Taxonomy and provenance links |

`counterparty` is the load-bearing one: `Dependency -counterparty-> Company` is the contagion edge
from README §6.5, so a counterparty's distress is reachable from every dependent company in one hop.

---

## 6. Postgres → RDF

GraphBuilder's mapper is **one subject per row**, and it does not expand JSONB or split rows. Three
views bridge that, and two rules follow from it.

| Postgres | Ontology | Rule |
| --- | --- | --- |
| `company` | `Company` | PK → `{base}/company/{id}` |
| `company_external_id` | `ExternalIdentifier` | `hasExternalId` |
| `fact` | `Assertion` | reified, never dropped — provenance lives here |
| `event` | `Event` | `eventKind` discriminates |
| `signal` | `Signal` | `verticalPack` carries the §6 pack |
| `dependency` | `Dependency` | `counterparty` → `Company` |
| `company_score` | `Score` | one node per model version per run |
| `verification` | `Entity.reviewState` | actor and time via PROV |
| `company_alias` | — | **gap:** no `AlternativeName` class yet; add one when ER merges ship |

```sql
-- Reified assertions: preserves method, confidence and URL per field.
create or replace view v_fact_assertion as
select
  md5(company_id::text || '|' || field || '|' || source_url || '|' || observed_at::text) as assertion_id,
  company_id,
  field                  as assertion_field,
  value_json->>'value'   as assertion_value,   -- confirm the JSONB shape before mapping
  source_url,
  observed_at,
  method                 as extraction_method,
  confidence
from fact;

-- Latest value per field, for datatype properties.
create or replace view v_fact_latest as
select distinct on (company_id, field) *
from fact
order by company_id, field, observed_at desc;

-- Pivot for direct datatype properties; one column per ontology property.
create or replace view v_company_profile as
select
  c.id as company_id,
  c.domain,
  c.name,
  c.founded_year,
  c.headcount,
  max((f.value_json->>'value')::int)  filter (where f.field = 'headcount')     as headcount_fact,
  max((f.value_json->>'value')::date) filter (where f.field = 'last_round_at') as last_round_at
from company c
left join v_fact_latest f on f.company_id = c.id
group by c.id;
```

```sql
-- Signal evidence arrays become rows so each evidence link survives the mapping.
create or replace view v_signal_evidence as
select
  s.id                        as signal_id,
  e.value->>'type'            as evidence_type,
  e.value->>'id'              as evidence_ref
from signal s
cross join lateral jsonb_array_elements(coalesce(s.evidence, '[]'::jsonb)) as e(value);
```

**Rule 1 — row polymorphism is a property, not a class.** `Event.eventKind` and `Deal.dealType`
carry the subtype. If a query needs a real subclass, create `v_event_layoff` and map it separately;
do not expect the mapper to infer it.

**Rule 2 — arrays and JSONB need views.** `sector text[]`, `signal.evidence`, and any multi-valued
field must be expanded before mapping, or the data silently disappears from the RDF.

---

## 7. Provenance

GraphBuilder emits `prov:wasDerivedFrom`, `prov:value` and `prov:hadPrimarySource` for node sources.
We extend that with the columns README §4 already requires:

| Growth column | RDF |
| --- | --- |
| `fact.source_url` | `:fromSource` → `:Source`; `prov:hadPrimarySource` |
| `fact.observed_at` | `prov:atTime` on the `Assertion` |
| `fact.method` | `prov:wasGeneratedBy` → `:Activity` with the method as label |
| `fact.confidence` | `:confidence` on the `Assertion` |
| `signal.evidence` | `:derivedFrom` → `Assertion` / `Event` |
| `verification.verified_by` | `prov:wasAttributedTo` |
| `verification.verified_at` | `prov:endedAtTime` on the review activity |

The rule from README §8 carries over: a served lead must resolve to field → value → URL →
timestamp → method. In RDF that is a path, and SHACL can enforce that the path exists.

---

## 8. Bitemporality and as-of export

GraphBuilder has no temporal query and every export is current-state. That is acceptable because
**Postgres remains the backtest source**; the RDF copy is for explanation and interchange.

Two clocks must stay separate:

- `Entity.validFrom` / `validTo` — when the fact was true in the world.
- `Observation.observedAt` — when we saw it.

For a point-in-time export, materialise a snapshot rather than filtering at query time:

```sql
create materialized view v_company_asof_2025_12_31 as
select * from company where updated_at <= '2025-12-31';
```

Never run a phase-0 backtest against the RDF export. It carries today's state and would leak the
future into the feature table.

---

## 9. Validation with SHACL

Generate shapes with `shacl_turtle(ontology)` and add these constraints in `shapes/growth-extra.ttl`:

- `Signal` — at least one `derivedFrom`; exactly one `signalOf`.
- `Company` — exactly one `domain`.
- `Assertion` — `sourceUrl` and `observedAt` required.
- `Permission` — at most one `expiresAt`.
- `Score` — `modelVersion` and `computedAt` required.

CI gate:

```bash
pyshacl -s out/growth.shacl.ttl -e shapes/growth-extra.ttl -df out/data.nt
```

A signal without evidence is the failure mode that makes the whole product untrustworthy, so this
check belongs in CI from the first commit, not in a later cleanup.

---

## 10. Export surfaces

| Surface | Call | Consumer |
| --- | --- | --- |
| OWL schema | `GET /workspaces/growth/ontology/export?format=owl` | partners, ontology review |
| SHACL shapes | `.../ontology/export?format=shacl` | CI, validators |
| Instance RDF | `GET /workspaces/growth/graph/export?format=ttl&limit=…` | graph consumers |
| R2RML + ontology files | `structured export <id> -o out/` | Ontop, Morph-KGC, external ETL |
| N-Triples | `structured load <id> --nt out/data.nt` | Neptune bulk loader, archival |
| OKF bundle | `GET /workspaces/growth/export/okf` | analyst verification queue |

The OKF bundle is the sleeper: one Markdown page per company with relations and sources is already
the "why this lead" brief, without building a UI.

---

## 11. Known gaps

| Gap | Impact | Mitigation |
| --- | --- | --- |
| No bitemporality | Cannot backtest from RDF | Keep Postgres authoritative; snapshot views per export |
| Single inheritance | `Company` cannot also be a `LegalEntity` | Model one parent; record the other as an IRI alignment |
| One class per table | Row subtypes are not classes | `kind` property + views per subtype where needed |
| No SPARQL endpoint | Extra service to operate | Ontop (virtual) or Morph-KGC (materialized) |
| `instances_to_rdf` writes untyped literals | Loses XSD types | Prefer the R2RML/Morph-KGC path for data |
| Property name uniqueness | Duplicate names create OWL intersections | Define shared properties once on the common parent |
| JSONB/array columns | Silent data loss in mapping | Expand in views before mapping |
| `company_alias` unmodelled | ER history not exportable | Add `AlternativeName` when merges ship |

---

## 12. Integration plan (week one)

| Day | Work |
| --- | --- |
| 1 | Load the seed (`POST /ontology/import`, mode `merge`); export OWL + SHACL; pySHACL passes on zero instances |
| 2 | Create the three views; `structured introspect` against a read-only Postgres user; fix class/property names |
| 3 | `structured export` → R2RML; run Morph-KGC over 5,000 companies; diff triple count against row count |
| 4 | Run SHACL on real data; fix violations; add the extra shapes to CI |
| 5 | Export an OKF bundle for 100 companies; put it in front of an analyst |

```bash
# ontology
curl -X POST "$GB/api/workspaces/growth/ontology/import" -F "files=@ontology/growth-seed.json" -F "mode=merge"
curl "$GB/api/workspaces/growth/ontology/export?format=owl"   -o out/growth.owl.ttl
curl "$GB/api/workspaces/growth/ontology/export?format=shacl" -o out/growth.shacl.ttl

# data
python -m graphbuilder.cli structured introspect --ws growth --name growthdb --source growth
python -m graphbuilder.cli structured export <id> --ws growth -o out/
python -m graphbuilder.cli structured load   <id> --ws growth --nt out/data.nt
python -m graphbuilder.cli export --ws growth --format okf -o out/growth-okf.zip
```

Use a read-only Postgres role. GraphBuilder never stores the password, but the mapping needs schema
access and there is no reason for it to have write access to the system of record.

---

## 13. Licensing

- GraphBuilder: Apache-2.0 — safe to depend on, vendor, or fork.
- FIBO packs: MIT.
- schema.org: **CC BY-SA 3.0**. The seed references two schema.org IRIs but imports no schema.org
  terms. Confirm before shipping an ontology bundle to a customer, since share-alike may attach to a
  derived vocabulary.
- Our namespace (`https://growthpal.local/onto#`) is a placeholder; change it before any external
  distribution.

---

## 14. Open questions

- [ ] Keep the schema.org IRI alignment on `Organization` and `Person`, or isolate the namespace for v1?
- [ ] One workspace for all verticals, or one per pack? A single workspace keeps entity resolution and cross-pack scoring in one graph; separate workspaces keep ontology changes independent.
- [ ] Who approves ontology version bumps — the data owner, or the engineer changing the mapping?
- [ ] Is a SPARQL endpoint a product requirement, or is file export enough for buyers?
- [ ] Does `AlternativeName` belong in the ontology, or does alias merging stay internal to Postgres?
