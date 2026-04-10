# Kira Self-Improvement Loop Architecture (April 2026)

## Core Principle
Automate the boring improvement loop. Keep human in the loop for claims, safety, and product decisions. Never let the agent self-modify live in production.

## 4 Planes
1. **Runtime plane** — Current chat product (classify → crisis guard → context → tools → generate → stream → log trace)
2. **Telemetry plane** — Structured events per turn (request, tool decisions, retrieval, output, safety, feedback)
3. **Eval plane** — Offline only. Replay saved queries against multiple configs, run graders/verifiers, compare candidates to baseline
4. **Promotion plane** — Canary deploy to small traffic slice, compare live metrics, auto-rollback or promote, human approval for sensitive changes

## Config Bundle Versioning
Every production run references one bundle ID containing versions of: system prompt, tool descriptions, routing rules, retrieval params, query rewrite, citation template, differential prompt, verifier prompt, safety policy.

```json
{
  "bundle_id": "kira-prod-2026-04-05.3",
  "generator_model": "llama-3.3-70b-versatile",
  "generator_prompt_version": "gen_v12",
  "tool_schema_version": "tools_v5",
  "retrieval_version": "retrieval_v9",
  "rewrite_version": "rewrite_v4",
  "citation_version": "citations_v2",
  "differential_version": "diff_v3",
  "verifier_version": "verify_v1",
  "safety_version": "safety_v6"
}
```

## Safe to Self-Iterate Offline (Lane A + B)
- Tool descriptions, query rewrite heuristics, retrieval fusion weights, reranker gating, chunk formatting, citation style, response templates, synonym expansion, rare-term lexical backoff, force-tool rules, BM25 params, top-k

## NOT Safe to Self-Iterate Without Human Approval (Lane C)
- Crisis classifier logic, safety policy, diagnostic/differential wording, KB article content, medical claims, user memory writes, anything that changes user risk posture

## Scoring Function
```
score =
  5.0 * supportedRate
+ 4.0 * toolUseCompliance
+ 4.0 * crisisRecall
+ 2.5 * citationCoverage
- 6.0 * unsupportedRate
- 8.0 * unsafeRate
- 1.0 * p95LatencySeconds
- 0.5 * costPerTurnUsd
```

## Rule-Based Failure Detectors (ship first)
1. Tool omission on in-scope clinical query (in_scope_clinical=true AND tool_called=false)
2. Unsupported number detector (%, prevalence, dose, cutoff with no citation anchor)
3. Diagnosis drift detector (strong diagnosis-like phrases)
4. Differential authority detector ("high likelihood"/ranking language)
5. Passive crisis detector miss-candidate (crisis-adjacent language in response but upstream regex didn't fire)

## Flywheel Loop
```
prod trace → flagged by detector → review queue
review queue → accepted as real failure → eval_cases
eval_cases → replayed nightly against candidates
top candidate → canary deploy → promote or rollback
```

## First 5 Automated Experiments
1. Forced tools vs auto tools (highest-signal)
2. Inline evidence citations vs sidebar-only sources
3. Rare-term lexical expansion pack (koro, dhat, MAOI, TCA, pharmacogenomics, culture-bound)
4. Selective reranker on long-tail queries only
5. Second-pass groundedness verifier (cheap model checks claims against chunks)

## Neon Schema (6 tables)
- `assistant_traces` — One row per user turn (session, bundle, model, tools, latency, cost, feedback)
- `assistant_trace_events` — One row per step (route, retrieve, tool_call, tool_result, generate, verify, guard)
- `failure_labels` — Human or model-assisted labels (supportedness, unsafe, diagnosis_drift, tool_omission, etc.)
- `eval_cases` — Permanent benchmark registry (source: prod_failure/synthetic/benchmark/manual)
- `candidate_bundles` — Proposed config variants with diff summary
- `eval_runs` — Scores candidate bundles against eval cases

## Build Order (10 days)
- Days 1-2: Tracing (assistant_traces + assistant_trace_events)
- Days 3-4: Review queue (rule-based detectors + /admin/failures page)
- Days 5-6: Eval registry (eval_cases + "promote to eval" button)
- Days 7-8: Forced-tools experiment (bundle flag for tool_mode, replay auto vs forced)
- Days 9-10: Verifier v1 (cheap verifier pass, measure supportedness improvement)

## Canary Deploy
- Deterministic hash of session_id, 5% traffic to canary
- Exclude crisis traffic from canary initially
- Compare: tool-call rate, verifier failure rate, unsupported proxy, citation coverage, thumbs up/down, latency, cost
- Promotion rule: unsupported down 20%+, no safety regression, p95 latency <20% increase, cost <30% increase
