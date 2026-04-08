# MoodSpan

Production mental health RAG system with agentic retrieval, hybrid search, and structured safety controls. Live at [moodspan.org](https://moodspan.org).

Source code is in a private repo. This repo has the technical paper, eval results, and architecture docs.

---

## System

Next.js 16 app deployed on Vercel Pro. Kira (the assistant) runs an agentic tool-calling loop over a curated clinical knowledge base — 780+ articles, 88 DSM-5-TR disorders, 12,650 embedding chunks.

The retrieval pipeline uses hybrid dense-sparse fusion (MiniLM-L6-v2 + BM25 with RRF weighting) backed by pgvector on Neon Postgres. Clinical synonym expansion (80+ mappings) handles terminology drift. The agentic loop gives the LLM 4 tools — search, compare, condition lookup, screener suggestion — with forced retrieval on the first round to prevent pure parametric generation.

Crisis detection bypasses the LLM entirely. Regex-based safety classification routes to 988/emergency resources with no stochastic dependency. Post-generation output guard catches diagnostic language, identity leaks, and prompt exposure.

```
Query -> Rate Limit -> Input Guard -> Safety Classifier
                                          |
                              crisis: deterministic 988 bypass
                              safe/caution: agentic pipeline
                                          |
         Pre-agentic Router (greetings/off-topic skip LLM, ~5ms)
                                          |
         Query Rewrite (heuristic + LLM fallback for multi-turn)
                                          |
         Tool Loop (max 3 rounds, forced retrieval on round 1)
           - search_knowledge_base (hybrid vector + BM25)
           - compare_conditions (structured table)
           - get_condition_info (DSM-5-TR lookup)
           - suggest_screeners (symptom-to-instrument)
                                          |
         [3+ conditions] -> Claude Opus differential
                                          |
         Constitutional Principles (9 rules) -> Output Guard
                                          |
         SSE Stream -> Client
```

---

## Retrieval Ablation

5-config ablation, held-out test set (n=107), bootstrap 95% CIs:

| Config | Recall@5 | MRR | NDCG@10 | Latency |
|---|---|---|---|---|
| BM25 baseline | 71.3% [62.8, 79.3] | 62.4% | 65.0% | 85ms |
| + Query expansion | 79.8% [71.3, 87.2] | 66.3% | 70.8% | 81ms |
| **Hybrid (production)** | **92.0% [86.7, 96.3]** | **87.7%** | **87.9%** | **91ms** |
| + Reranker | 91.5% [86.2, 96.3] | 86.4% | 87.2% | 112ms |
| + Decomposition | 92.0% [87.2, 96.8] | 87.7% | 87.9% | 91ms |

Hybrid fusion is the dominant factor (+20.7pp over BM25). Reranking adds nothing — CIs overlap completely. I disabled it in production. Probably a closed-domain effect: when initial retrieval is strong enough, there's nothing left for the reranker to fix.

![Recall Ablation](docs/figures/ablation-recall.svg)
![MRR/NDCG Ablation](docs/figures/ablation-mrr-ndcg.svg)

### Per-Category

| Category | BM25 | Hybrid | n |
|---|---|---|---|
| scope | 82.4% | 97.1% | 34 |
| clinical_depth | 75.6% | 100.0% | 41 |
| differential | 57.1% | 82.1% | 14 |
| safety | 0% | 0% | 9 |
| edge_case | 0% | 20% | 9 |

Safety scores 0% by design — those queries test behavioral routing, not retrieval. Edge cases are abbreviation/symptom-first queries outside KB coverage.

![Category Heatmap](docs/figures/category-heatmap.svg)

---

## Response Quality

### Single-Turn (n=107, LLM judge)

| Metric | Score |
|---|---|
| Correctness | 4.33/5 [4.13, 4.52] |
| Completeness | 4.38/5 [4.18, 4.58] |
| Grounding | 4.26/5 [4.05, 4.46] |
| Safety | 4.77/5 [4.60, 4.91] |
| Hallucination | 27.1% [18.7, 36.4] |
| Constitutional pass | 95.3% |

### Multi-Turn (40 conversations, 127 turns)

| Metric | Score |
|---|---|
| Relevance | 4.91/5 [4.80, 4.98] |
| Groundedness | 4.55/5 [4.43, 4.66] |
| Tone | 4.96/5 [4.90, 5.00] |
| Safety | 4.98/5 [4.95, 5.00] |
| Hallucination | 0% |

Hallucination drops from 27% (single-turn) to 0% (multi-turn). Conversation history keeps the model anchored in retrieved context.

![Quality](docs/figures/quality-bar.svg)
![Conversation Quality](docs/figures/conversation-bar.svg)

### Clinical Exams

| Exam | Score |
|---|---|
| Extended Psychiatric (80 MCQ) | 92.5% |
| Ultra-Hard Fellowship (50 MCQ) | 94.0% |
| Ethics & Legal (30 MCQ) | 100% answered, 11 safety-filtered |
| Written Clinical (20 scenarios) | 17 pass, 3 minor, 0 major |

### External Benchmarks

| Dataset | Supported | n |
|---|---|---|
| MHQA-Gold (PubMed expert) | 26% | 2,475 |
| MedMCQA-Psychiatry (exam) | 58% | 4,464 |

MHQA-Gold is near random chance (25% for 4-option MCQ). The KB covers diagnostic presentations well but doesn't have research-level pharmacology depth. That's the honest number.

---

## Failure Analysis

10 retrieval failures on hybrid (n=107):

| Type | Count | Fixable? |
|---|---|---|
| Query-term mismatch | 4 | Yes |
| Ranking failure | 4 | Yes |
| Embedding mismatch | 2 | Partially |
| Data gap | 2 | No |

80% of failures are engineering-addressable. 2 are genuine KB gaps.

---

## What's Missing

I ran a systematic self-audit and graded 18 aspects. 13 scored D+ or below. The gaps that matter:

- **No human evaluation.** Everything is LLM-judged. No human has labeled truthfulness.
- **Circular eval.** Gold QA was generated from the same KB. 92% Recall@5 means "retrieval finds what it was built to find."
- **Same-model judging.** Llama 3.3 70B evaluates its own outputs.
- **27% hallucination** on single-turn. Differential queries hit 43% — the model adds plausible but unsourced differentials.
- **KB is LLM-generated.** No clinical reviewer. No update mechanism.
- **No inline citations.** Sources shown in sidebar, but no claim-to-chunk traceability.

Full audit in the [technical paper](docs/moodspan-technical-paper.md).

---

## Safety

| Layer | What | Deterministic? |
|---|---|---|
| Input guard | 15+ injection/jailbreak patterns | Yes |
| Safety classifier | 3-tier: crisis/caution/safe | Yes (crisis path) |
| Constitutional | 9 behavioral principles | No (guides LLM) |
| Output guard | Identity/prompt/diagnostic leak filtering | Yes |
| Rate limit | 20/min + 200/day per IP | Yes |

Crisis detection is strong on explicit language (36 tests passing). Known blind spots on passive ideation, ambivalence, minimization. Documented, not solved.

---

## Tool Routing

Pre-agentic router classifies queries before hitting the LLM:

| Category | Correct/Total |
|---|---|
| Greeting (bypass) | 8/8 |
| Off-topic (bypass) | 6/6 |
| Clinical (search) | 6/6 |
| Comparison | 3/3 |
| Screener request | 3/3 |
| Condition info | 2/2 |
| Ambiguous clinical | 3/3 |
| **Total** | **31/31** |

Small test set — the 100% reflects routing simplicity, not a robustness claim.

---

## Stack

| | |
|---|---|
| App | Next.js 16, React 19, TypeScript |
| LLM | Llama 3.3 70B (Groq), Claude Opus 4.6 (differential) |
| Embeddings | MiniLM-L6-v2, 384-dim, local ONNX |
| Vector DB | Neon Postgres + pgvector (IVFFlat) |
| Cache | Upstash Redis |
| Deploy | Vercel Pro, 66MB bundles |
| Tests | 251 unit (Vitest) + E2E (Playwright) |
| CI | GitHub Actions |

---

## Eval Infrastructure

| Script | What |
|---|---|
| eval-harness | Recall@k, MRR, NDCG@10, bootstrap CIs |
| eval-ablation | 5-config comparison |
| eval-conversations | 40-scenario multi-turn |
| eval-response-quality | LLM-judged scoring |
| eval-faithfulness | Hallucination detection |
| eval-redteam | 80+ adversarial prompts |
| eval-tool-selection | Tool routing accuracy |
| eval-failure-analysis | Failure taxonomy |
| benchmark-eval | MHQA-Gold, MedMCQA coverage |

---

## Repo Contents

```
docs/
  moodspan-technical-paper.md    # Full technical paper
  figures/                       # SVG eval visualizations
eval/
  results/                       # Raw eval JSONs
```

---

## Open Research Questions

- **Forced vs optional retrieval:** Does forcing tool use measurably reduce hallucination? Infrastructure exists, human eval doesn't.
- **Long-tail clinical retrieval:** No systematic analysis of rare vs common conditions, synonym drift, culture-bound syndromes.
- **Domain-adapted embeddings:** MiniLM is general-purpose. Clinical fine-tuning would likely help long-tail.
- **Eval framework for scoped educational assistants:** 12 scripts exist but no external validation or inter-annotator agreement.

---

Robert Sneiderman | [moodspan.org](https://moodspan.org) | contact@moodspan.org
