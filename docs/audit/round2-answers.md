# Round 2 Honest Audit — 18 Hard Questions (April 2026)

Critical assessment of Kira's capabilities, gaps, and defensible claims.
Framing: "Kira is a vertical mental-health education copilot with hybrid retrieval, structured screeners, and scoped safety controls."

## Overall Classification
- Product quality: decent to strong early vertical copilot
- Agent sophistication: low to moderate (not an advanced agent — structured vertical RAG copilot with agent-like routing)
- Evaluation maturity: moderate
- Clinical/safety rigor: mixed
- Research novelty: currently weak unless claim is narrowed hard

## Q1: Defensible Claim
"On a curated mental-health education workload of ~780 articles covering 88 DSM-5 disorders, Kira provides retrieval-grounded explanatory answers with 92% Recall@5 on in-distribution queries, scoped safety controls that bypass the LLM entirely for crisis detection, and structured screener integration — but it does not enforce retrieval, verify groundedness, or perform autonomous clinical reasoning."

## Q2: True Baseline (Ablation Table)
| Config | Recall@5 | MRR | NDCG@10 | Latency |
|---|---|---|---|---|
| BM25 only, no expansion | 71.3% | 62.4% | 65.0% | 85ms |
| BM25 + query expansion | 79.8% | 66.3% | 70.8% | 81ms |
| Hybrid (vector+BM25) | 92.0% | 87.7% | 87.9% | 91ms |
| Hybrid + reranker | 91.5% | 87.0% | 87.3% | 112ms |
| Hybrid + decompose | 92.0% | 87.7% | 87.9% | 91ms |

**Missing:** Same model, no retrieval (pure parametric). Cannot claim "RAG adds X%" without this.

## Q3: Model vs Retrieval Attribution
Unknown. Estimated 70-80% of value is retrieval + curation. Model adds explanatory fluency, not clinical reasoning. No attribution table exists. Would need same retrieval + smaller model (Llama 8B) experiment.

## Q4: Production Tool-Call Rate
**Unknown.** tool_choice: "auto" — model decides. System prompt says "ALWAYS call search" but it's a soft instruction. No production analytics aggregated from [KIRA_EXPERIMENT] logs. "Grounded assistant" framing unsupported by production evidence until this is measured.

## Q5: Groundedness Conditional on Tool Use
**Table does not exist.** Need: sample 100+ production queries, label tool-used vs not, have reviewer label supported/partial/unsupported. Bet: no-tool bucket has dramatically higher unsupported rate.

## Q6: Benchmark Contamination
**Structural contamination, not prevented.**
- Gold QA (438 pairs) generated from same structured data as KB (closed-loop)
- 88% of test queries have expected_source_slugs mapping to KB entries
- No decontamination check for test↔KB overlap
- Same Groq/Llama model judges its own outputs (self-preference bias)
- Clinical exams: hand-authored but no KB overlap check
- External benchmarks more honest: MHQA-Gold 23.8% supported, MedMCQA-Psych 58%
- 92% Recall@5 = "retrieval pipeline finds what it was built to find" — not generalizable

## Q7: Human-Evaluated Truthfulness
**Zero. No human evaluation exists.** All eval is automated (retrieval metrics, LLM-judged quality). This is the single most important missing piece.

## Q8: Failure Denominator
Failure taxonomy exists but rates are not quantified from production:
- % ranking failure: observed, not quantified
- % embedding mismatch: observed for rare/specialized terms
- % tool omission: not aggregated
- % hallucinated fact: no verification pipeline
- % safety-format violation: output guard logs exist but not aggregated
- % crisis misroute: no false-negative tracking beyond test suite
- In-distribution: 8% fail retrieval (Recall@5=92%). Out-of-distribution: MHQA-Gold shows 76.2% not supported.

## Q9: Reranker Disabled — Why?
No significant improvement on 107 test queries (+21ms latency). But:
- Test set too easy and too small (mostly common conditions)
- No subgroup analysis (rare disorders, multi-hop, medication, synonym drift, culture-bound)
- Closed-domain effect: strong initial retrieval makes reranking less impactful
- Need long-tail subgroup analysis before concluding reranker has no value

## Q10: Long-Tail Retrieval
No systematic slices exist. Category breakdown:
- Scope (general): 97.1% (n=34) — easy
- Clinical depth: 100% (n=41) — suspiciously perfect (generated from KB)
- Differential: 82.1% (n=14) — harder
- Safety: 0% (n=9) — by design
- Edge case: 20% (n=9) — out of KB

Missing: common vs rare, short vs long query, layperson vs clinician, single vs cross-condition, treatment vs diagnosis vs screening vs medication slices.

## Q11: Knowledge Base Provenance
- ~780 articles: primarily LLM-generated (Claude Opus 4.6 via Batches API)
- 1 published source: "Nursing Mental Health and Community Concepts 2e" (open-access textbook, RAG-ingested)
- Structured data: Claude-generated (KB v3, non-PD disorders, DSM parsed)
- **No clinical reviewer has validated articles. No SME review.**
- **No update policy.** Content is static — no mechanism for outdated guidance, new DSM/ICD revisions, medication updates, conflicting recommendations.
- Most damaging admission for clinical credibility.

## Q12: Citation Model
**Weak.** Sources shown in UI sidebar (article titles + links). No inline citations, no quotes from chunks, no "According to [source]" attribution, no claim↔chunk mapping. System prompt explicitly forbids "Sources" sections. User cannot verify which claims came from which sources.

## Q13: Differential Diagnosis Card Risk
**Not studied.** Ranked conditions with high/moderate/low badges could be interpreted as diagnostic guidance. No usability testing, no user interpretation study, no UI disclaimer on the card itself. No pre-safety check before differential execution (crisis-adjacent queries with 3+ conditions would trigger). Significant product and review risk.

## Q14: Crisis Detection False Negatives
Good on explicit language (36 tests passing). Blind spots:
- Passive ideation ("wouldn't mind not waking up") — NOT DETECTED
- Minimization + signal ("I'm fine but sometimes I think about dying") — PARTIAL
- Ambivalence ("I don't know if I want to be alive") — NOT DETECTED
- Manic grandiosity with harm — NO PATTERNS
- Command hallucinations ("voices tell me to hurt myself") — PARTIAL
- Sarcasm ("Kill me now, homework is terrible") — FALSE POSITIVE
Architecture strength: crisis bypasses LLM entirely. Weakness: regex is sole gatekeeper.

## Q15: Why LLM Instead of Search-and-Template?
LLM adds moderate, not transformative value:
- Natural language synthesis of retrieved chunks for open-ended questions
- Multi-turn conversation carryover (10-message window)
- Cross-document comparative synthesis
- Flexible tool routing
A search-and-template system could achieve ~80% of value for common queries. LLM justified as "better UX layer on top of search" at low cost (~$0.001-0.003/query via Groq).

## Q16: Cost/Latency/Reliability Frontier
| Mode | Latency | Cost/query | Faithfulness |
|---|---|---|---|
| Pre-agentic router (greeting/off-topic) | ~5ms | $0 | N/A |
| BM25 only | ~85ms + 1-2s Groq | ~$0.001 | 71% Recall@5 |
| Current production (hybrid, auto tools) | ~91ms + 1-3s Groq | ~$0.002 | 92% Recall@5, unverified groundedness |
| With differential (Pro, Claude) | +2-5s | ~$0.05-0.15 | Higher (structured) |
No engineered operating point. No dynamic routing by difficulty/confidence/tier.

## Q17: Research Novelty
Nothing currently publishable. Potential angles (all require new experiments):
1. Groundedness collapse when tool use is optional (forced vs optional experiment)
2. Failure taxonomy for mental-health RAG (systematic production classification)
3. Rare-term retrieval failures in consumer mental-health QA
4. Eval framework for clinically scoped educational assistants

## Q18: Decisive Experiment
Force tool use for all clinical queries. Measure human-labeled groundedness with forced tools vs optional tools. If forced grounding cuts unsupported claims from X% to Y%, that's a real result. Highest signal-to-effort ratio.

## Summary Grades (Honest)
| Category | Grade |
|---|---|
| Defensible claim | B- (stated, not proven) |
| Controlled baselines | B (ablation exists, missing no-RAG) |
| Attribution | D (no model-vs-retrieval separation) |
| Tool-call enforcement | F (optional, unmeasured) |
| Groundedness verification | F (does not exist) |
| Benchmark contamination | D (structural, unaddressed) |
| Human evaluation | F (none) |
| Failure quantification | C- (taxonomy exists, rates unknown) |
| Reranker analysis | C (global only, no subgroup) |
| Long-tail retrieval | D (no systematic slices) |
| Knowledge provenance | D (LLM-generated, no review, static) |
| Citation model | D (sidebar only, no evidence linking) |
| Differential risk | D (not studied) |
| Crisis false-negatives | C+ (good explicit, blind indirect) |
| LLM justification | B- (moderate value, honestly stated) |
| Cost/latency frontier | C (observed, not engineered) |
| Research novelty | D+ (potential, nothing ready) |
| Decisive experiment | B- (identified, not run) |

## Priority Actions (from reviewer feedback)
1. Force retrieval/tool use for in-scope clinical queries
2. Add evidence-linked outputs (inline citations)
3. Run human-labeled supportedness study on production queries
4. Improve long-tail retrieval (domain-adapted embeddings or better lexical treatment)
5. Narrow research claim to something provable
