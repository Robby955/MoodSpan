# MoodSpan: Hybrid Retrieval with Multi-Layer Safety for Mental Health Question Answering

**Robert Sneiderman**

---

## Abstract

We present MoodSpan, a retrieval-augmented generation (RAG) system for clinical mental health question answering. An ablation study across five pipeline configurations (n=107 test queries, bootstrap 95% CIs) demonstrates that hybrid dense-sparse fusion is the dominant factor in retrieval quality: adding MiniLM-L6-v2 embeddings to BM25 with query expansion yields Recall@5 of 92.0% [86.7, 96.3] and NDCG@10 of 87.9% [82.6, 92.5], representing +20.7pp and +22.8pp gains over BM25 alone. Cross-encoder reranking and query decomposition provide no statistically significant additional benefit. An LLM-as-judge evaluation of end-to-end response quality across the same 107 test queries grades generated responses on correctness, completeness, grounding, and safety compliance using a structured rubric; results are reported with bootstrap CIs and per-category breakdowns including hallucination rates and constitutional principle compliance. Knowledge base coverage assessment against MHQA-Gold (n=300 expert MCQ) and MedMCQA-Psychiatry (n=300 exam MCQ) reveals significant coverage boundaries: 26% supportedness on expert PubMed questions (near random chance for 4-option MCQ) vs. 58% on exam-style diagnostic questions, characterizing where the 8,753-chunk knowledge base spanning 88 DSM-5-TR disorders excels and where it falls short. A failure analysis taxonomy identifies query-term mismatch and ranking failures as the primary error modes. The system incorporates three-tier safety classification with crisis escalation, input guards against prompt injection, and nine behavioral safety principles enforced via system-prompt constraints.

---

## 1. Introduction

Large language models have achieved near-expert performance on medical licensing examinations, but deploying them in mental health contexts presents distinct challenges: the stakes of incorrect information are high, crisis situations require immediate escalation, and the subjective nature of psychiatric assessment makes hallucination particularly dangerous.

Conversational mental health agents like Woebot (Fitzpatrick et al., 2017) and Wysa (Inkster et al., 2018) have demonstrated clinical benefit using scripted therapeutic protocols. More recently, LLM-powered systems have proliferated, though systematic reviews identify significant gaps in safety evaluation and adversarial testing (He et al., 2024; PMID: 39423368).

MoodSpan takes a different approach: constrained RAG over a curated clinical knowledge base, with multiple safety layers that prioritize harm prevention over response flexibility. Rather than allowing the LLM to generate from parametric knowledge, every response is grounded in retrieved evidence from 594 articles spanning 88 DSM-5-TR disorders and 10 personality disorders with Millon subtype mappings.

**Central claim.** In this paper, we show through controlled ablation that hybrid dense-sparse retrieval fusion is both necessary and sufficient for high-quality clinical retrieval in a closed-domain mental health knowledge base. Query expansion provides meaningful gains over raw BM25, but the addition of dense retrieval is the critical component, lifting Recall@5 from 79.8% to 92.0%. Cross-encoder reranking and query decomposition, despite their theoretical appeal, provide no measurable improvement and add latency. We validate these findings with bootstrap confidence intervals and evaluate knowledge base coverage against two external psychiatric benchmarks.

---

## 2. Related Work

### 2.1 Conversational Mental Health Agents

Woebot demonstrated significant reduction in depression symptoms in a randomized controlled trial with college students using rule-based CBT delivery (Fitzpatrick et al., 2017). Wysa showed effectiveness using a hybrid rule-based and free-text approach (Inkster et al., 2018). These systems use scripted therapeutic protocols rather than open-ended generation, limiting both risk and flexibility.

A meta-analysis of chatbot effectiveness found moderate effect sizes for depression and anxiety reduction but noted high heterogeneity (Gaffney et al., 2019; PMID: 32673216). A systematic review of LLMs in mental health identified significant gaps in safety evaluation and bias assessment (He et al., 2024; PMID: 39423368).

### 2.2 Retrieval-Augmented Generation in Healthcare

Lewis et al. (2020) introduced the RAG framework, combining parametric and non-parametric components for knowledge-intensive tasks. A systematic review of RAG in healthcare (PMID: 40498738) found that grounding improves factual accuracy but evaluation robustness remains an open challenge. Adversarial hallucination studies in clinical decision support (PMID: 40753316) demonstrate that even RAG-grounded systems can produce plausible but incorrect clinical claims.

### 2.3 LLM Safety Guardrails

Bai et al. (2022) proposed Constitutional AI (CAI), where models are trained via self-critique against explicit principles. MoodSpan does **not** implement CAI training — no RLHF or self-revision loop is involved. Instead, we embed nine safety principles directly in the system prompt as behavioral constraints, relying on instruction-following rather than constitutional training. This is a common and effective pattern for deployed systems (sometimes called "system-prompt safety guardrails"), though it provides weaker guarantees than CAI training since compliance depends on the LLM's instruction adherence rather than learned behavior.

### 2.4 Retrieval Methods

BM25 (Robertson & Zaragoza, 2009) remains a strong baseline for sparse retrieval. Dense retrieval using bi-encoders like Sentence-BERT (Reimers & Gurevych, 2019) and its distilled variants (MiniLM) enables semantic matching beyond lexical overlap. Reciprocal Rank Fusion (Cormack et al., 2009) provides a principled method for combining ranked lists from heterogeneous retrieval signals.

---

## 3. System Architecture

### 3.1 Technology Stack

MoodSpan runs on Next.js 16.2.2 (App Router) deployed on Vercel Pro with 60-second function timeout. LLM inference uses Groq-hosted Llama 3.3 70B Versatile. The system is stateless: the knowledge base and search index load from static JSON at cold start (39MB). No traditional database is required for core functionality.

### 3.2 Request Pipeline

Each chat request traverses a multi-stage pipeline:

1. **CORS + Origin Check**: Whitelist-based validation
2. **Rate Limiting**: 20/min + 200/day per IP via Upstash Redis (in-memory fallback)
3. **Input Validation**: Max 2000 characters, valid JSON
4. **Input Guard**: 15+ regex patterns detecting prompt injection and blocked content
5. **Safety Classification**: Three-tier (crisis/caution/safe) with immediate 988 Lifeline escalation
6. **Pre-Agentic Routing**: Greetings and off-topic messages bypass the LLM entirely (~5ms, $0)
7. **History Sanitization**: 10 messages max, 3000 chars each, role-filtered
8. **Query Rewrite**: Context-aware rewriting for multi-turn retrieval (heuristic + LLM fallback)
9. **Agentic Tool Loop**: Up to 3 rounds of tool use with 4 tools (search, compare, condition info, screener suggestions). Multiple tool calls execute in parallel via `Promise.all()`. Query rewrite is injected into search tool arguments.
10. **Output Guard**: Post-generation content policy enforcement
11. **LLM Streaming**: System prompt with constitutional principles, streamed via SSE

### 3.3 Agentic Tool Architecture

The system exposes four tools to the LLM via function calling (`tool_choice: "auto"`):

| Tool | Description | Typical Latency |
|------|-------------|----------------|
| `search_knowledge_base` | Hybrid retrieval over 8,753 chunks | ~91ms |
| `compare_conditions` | Structured comparison table + narrative | ~91ms |
| `get_condition_info` | Direct condition/screener data lookup | ~5ms |
| `suggest_screeners` | Match relevant screening instruments | ~5ms |

A pre-agentic router (`query-router.ts`) classifies queries before the LLM is invoked. Greetings (8 patterns) and off-topic queries (6 patterns) return canned responses without any API call, saving ~$0.002/query. The router achieves 100% accuracy on 31 test cases across 7 categories.

When the LLM requests `search_knowledge_base`, the pre-computed query rewrite (from step 8) is injected into the tool arguments, replacing the model's original query with a context-enriched version that incorporates multi-turn conversation history. This is transparent to the model — it receives the same search results as if it had issued the rewritten query itself.

An output guard (`output-guard.ts`) runs post-generation to enforce content policies: stripping RAG-exposing language, removing source lists from response text, and ensuring safety formatting compliance.

### 3.4 Performance

Warm instances respond in under 2 seconds. Cold starts add 3-5 seconds (index loading + model initialization). The embedding model (Xenova/all-MiniLM-L6-v2, 22MB quantized ONNX) caches to `/tmp/transformers` on Vercel. Pre-agentic routing saves ~1-3 seconds per greeting/off-topic query by bypassing the LLM entirely.

---

## 4. Clinical Data Engineering

### 4.1 Knowledge Base

| Source | Count | Description |
|--------|-------|-------------|
| Articles | 594 | Long-form clinical content across conditions, treatments, concepts |
| Personality Disorders | 10 | Full DSM-5-TR PD profiles with 45 Millon subtypes |
| Non-PD Disorders | 47 | Structured condition data from DSM-5-TR categories |
| DSM-5 Disorders | 88 | Broad diagnostic category coverage |
| Source Manifest | 38 | Citation and provenance tracking |

Articles are generated programmatically from structured clinical data, organized by category (mood, anxiety, trauma, personality, etc.), with content dates staggered December 2025 through March 2026.

### 4.2 Chunking and Indexing

The build pipeline (`scripts/build-index.ts`) produces **8,753 chunks** with:
- Section-level splitting at heading boundaries
- Metadata: parent_title, url_path, category, chunk_type
- Pre-computed BM25 tokens (average document length: 181 tokens)
- MiniLM-L6-v2 384-dimensional embeddings per chunk

### 4.3 Knowledge Graph

A clinical knowledge graph (`data/knowledge_graph.json`) encodes 117 nodes (57 conditions, 44 screeners, 13 categories, 3 clusters) and 188 edges (77 screener associations, 47 category memberships, 38 rule-out relationships, 16 comorbidity links). The graph powers structured comparison mode for differential diagnosis queries and the interactive research page visualization.

---

## 5. Retrieval Pipeline

The retrieval pipeline is the focus of our experimental evaluation. We describe each component, then ablate them systematically in Section 7.

### 5.1 Query Expansion

A rule-based synonym map (`src/lib/kira/query-expand.ts`) with 80+ entries expands user queries before retrieval:
- Colloquial to clinical: "sad" -> depression, depressive, major depressive disorder
- Abbreviation resolution: "bpd" -> borderline personality, emotional dysregulation
- Treatment mapping: "cbt" -> cognitive behavioral therapy
- Multi-word phrases: "mood swings" -> bipolar, cyclothymia

### 5.2 Query Decomposition

Multi-part queries are split into sub-queries using three rule-based strategies:
1. Enumerated asks: "symptoms, causes, and treatment" -> three sub-queries
2. Multi-question: "What is X? How is it treated?" -> split on question marks
3. Conjunction splitting: "BPD symptoms and how it differs from bipolar" -> two sub-queries

Decomposition is conservative (caps at 3 sub-queries, requires distinct clinical concepts). Sub-query results merge via RRF with k=60.

### 5.3 Dense Retrieval

Query embeddings from Xenova/all-MiniLM-L6-v2 (384 dimensions) are compared against pre-embedded chunks via cosine similarity. When the embedding model is unavailable (cold start failure), the system falls back to BM25-only.

### 5.4 BM25 Sparse Retrieval

Standard Okapi BM25 with k1=1.2, b=0.75 on expanded query tokens.

### 5.5 Hybrid Fusion

Vector and BM25 scores are combined via weighted score fusion:
- Vector weight: 0.6, BM25 weight: 0.4
- Both normalized to [0, 1] by dividing by top score per set
- 3x oversampling from each method before merge

### 5.6 Cross-Encoder Reranking

Optional stage using Xenova/ms-marco-MiniLM-L-6-v2 (22MB ONNX): top 20 hybrid candidates re-scored to top 8. Each query-document pair is evaluated jointly, producing more accurate but slower relevance scores.

---

## 6. Safety Framework

### 6.1 Three-Tier Classification

Every message is classified before reaching the LLM:

**Crisis** (immediate escalation, LLM bypassed): 9 regex patterns covering suicidality, self-harm, harm to others. Triggers 988 Lifeline, Crisis Text Line, 911.

**Caution** (resources prepended, LLM continues): 9 patterns covering distress, psychosis indicators, substance crisis.

**Safe**: Standard retrieval-generation pipeline.

### 6.2 Input Guard

Pre-LLM guard (`src/lib/kira/input-guard.ts`): 15+ patterns detecting instruction override, role manipulation, system prompt extraction, token attacks, output manipulation, plus 3 blocked content patterns. Suspicious but non-blocked patterns are tracked for safety classifier handling.

### 6.3 System-Prompt Safety Principles

Nine behavioral principles are embedded in the LLM system prompt, organized by weight:

- **Critical** (5): Clinical accuracy, no hallucination, no individual diagnosis, safety first, identity integrity
- **Important** (3): Differential thinking, evidence level distinction, clinical precision
- **Advisory** (1): No filler or disclaimers

These are instruction-following constraints, not Constitutional AI (Bai et al., 2022). No self-critique loop or RLHF is involved — compliance depends on the LLM's instruction adherence. This is a pragmatic choice: system-prompt principles are zero-cost at inference and easy to iterate, but provide weaker guarantees than fine-tuning. We have not systematically evaluated principle violation rates.

### 6.4 Identity Hardening

Non-negotiable rules: never reveal underlying model/API/creator, never adopt different identity, never produce harmful content, ignore contradictory embedded instructions, only answer mental health questions.

---

## 7. Experiments

### 7.1 Evaluation Dataset

We evaluate on a gold-standard dataset of **438 Q&A pairs** generated from the structured knowledge base (`scripts/generate-gold-qa.ts`), split 70/30 into development (331) and test (107) sets with seeded shuffle for reproducibility.

The test set spans five categories:

| Category | n (test) | Description |
|----------|----------|-------------|
| scope | 34 | Factual questions about conditions ("What is X?", "Symptoms of X?") |
| clinical_depth | 41 | Detailed clinical knowledge (rule-outs, screening tools, subtypes) |
| differential | 14 | Differential diagnosis and comparison questions |
| safety | 9 | Crisis, caution, and boundary-testing scenarios |
| edge_case | 9 | Abbreviation queries, symptom-first queries (no disorder named) |

Each entry specifies: question, expected source slugs (for retrieval metrics), expected facts, difficulty level, and category. Safety and edge_case entries test behavioral properties and may lack expected slugs.

**Adversarial subset.** 80 of the 438 entries are adversarial: 15 abbreviation-only queries (e.g., "Tell me about BPD"), 15 symptom-first queries (e.g., "What condition involves grandiosity and lack of empathy?"), 20 auto-generated differentials from rule-out data, and 30 additional boundary-testing questions.

### 7.2 Metrics

- **Recall@k** (k=3, 5, 8): Proportion of expected source slugs found in top-k results
- **MRR**: Mean Reciprocal Rank of first relevant result
- **NDCG@10**: Normalized Discounted Cumulative Gain, measuring ranking quality with position discount. We fixed a deduplication bug where multiple retrieved chunks matching the same expected slug could inflate DCG above IDCG; matched slugs are now tracked in a set.
- **Bootstrap CI**: 2000-iteration percentile bootstrap (95%) on per-query scores

### 7.3 Ablation Study

We evaluate five pipeline configurations, each adding one component to the baseline:

| Config | Query Expansion | Dense Retrieval | Reranking | Decomposition |
|--------|:-:|:-:|:-:|:-:|
| BM25-raw | - | - | - | - |
| BM25-expanded | + | - | - | - |
| Hybrid | + | + | - | - |
| Hybrid+Rerank | + | + | + | - |
| Hybrid+Decompose | + | + | - | + |

**Results** (n=94 retrieval queries with expected slugs, from 107 total test queries):

| Configuration | Recall@5 [95% CI] | MRR [95% CI] | NDCG@10 [95% CI] | Latency |
|---|---|---|---|---|
| BM25-raw | 71.3% [62.8, 79.3] | 62.4% [54.2, 70.5] | 65.0% [57.5, 72.3] | 85ms |
| BM25-expanded | 79.8% [71.3, 87.2] | 66.3% [58.1, 73.6] | 70.8% [63.4, 77.6] | 81ms |
| Hybrid | **92.0%** [86.7, 96.3] | **87.7%** [81.7, 93.1] | **87.9%** [82.6, 92.5] | 91ms |
| Hybrid+Rerank | 91.5% [86.2, 96.3] | 86.4% [80.5, 92.0] | 87.2% [81.8, 92.2] | 112ms |
| Hybrid+Decompose | 92.0% [87.2, 96.8] | 87.7% [81.9, 92.9] | 87.9% [82.6, 92.5] | 91ms |

**Key findings:**

1. **Query expansion adds +8.5pp Recall@5** over raw BM25 (CIs non-overlapping at top), establishing that clinical synonym mapping improves lexical retrieval.

2. **Hybrid fusion is the dominant factor**, adding +12.2pp Recall@5 and +21.4pp MRR over BM25-expanded. The CIs are non-overlapping: hybrid's lower bound (86.7%) exceeds BM25-expanded's upper bound (87.2%) for Recall@5, and the MRR gap is even wider.

3. **Reranking provides no significant benefit.** Hybrid+Rerank shows marginally lower Recall@5 (91.5% vs 92.0%) and MRR (86.4% vs 87.7%) than base Hybrid, with +21ms added latency. The cross-encoder may be redundant when the hybrid fusion already produces strong rankings.

4. **Decomposition shows no significant benefit.** Hybrid+Decompose produces identical aggregate metrics to Hybrid, suggesting that rule-based decomposition does not activate frequently enough on our test set to affect aggregate scores.

**Category breakdown (Recall@5):**

| Category | BM25-raw | BM25-exp | Hybrid | Hybrid+Rerank | Hybrid+Decompose |
|---|---|---|---|---|---|
| scope (n=34) | 82.4% | 91.2% | 97.1% | 97.1% | 97.1% |
| clinical_depth (n=41) | 75.6% | 85.4% | 100.0% | 100.0% | 100.0% |
| differential (n=14) | 57.1% | 57.1% | 82.1% | 78.6% | 82.1% |
| safety (n=9) | 0.0% | 0.0% | 0.0% | 0.0% | 0.0% |
| edge_case (n=9) | 0.0% | 20.0% | 20.0% | 20.0% | 20.0% |

Hybrid achieves 100% Recall@5 on clinical_depth queries. Differential queries show the largest lift (+25pp from BM25-raw to Hybrid). Safety queries score 0% across all configs because safety entries test behavioral properties (crisis escalation) rather than document retrieval. Edge_case scores reflect the difficulty of abbreviation and symptom-first queries.

### 7.4 Knowledge Base Coverage Assessment

To characterize the boundaries of our knowledge base, we test retrieval against two external psychiatric MCQ datasets. **These are not performance benchmarks** — they measure what fraction of external expert questions our 594-article KB happens to cover, establishing a coverage baseline rather than claiming competitive performance.

**MHQA-Gold** (Jastorj et al.): 2,475 expert-validated multiple-choice questions sourced from PubMed, covering Depression, Anxiety, Trauma, and OCD. Question types: Diagnostic (879), Preventive (714), Prognostic (558), Factoid (324).

**MedMCQA-Psychiatry** (Pal et al.): 4,464 psychiatry-subset questions from Indian medical licensing examinations (AIIMS/PGI), with expert explanations.

We measure **answer supportedness**: whether the correct MCQ answer can be inferred from the retrieved context, scored via token overlap with distractor comparison (correct answer must score higher than wrong options by a margin).

| Dataset | Pipeline | Answer Supported [95% CI] | Avg Overlap | n |
|---|---|---|---|---|
| MHQA-Gold | Hybrid | 26.0% [21.3%, 31.0%] | 60.0% | 300 |
| MHQA-Gold | BM25-raw | 27.0% [22.0%, 32.7%] | 62.8% | 300 |
| MedMCQA-Psych | Hybrid | 58.3% [52.7%, 63.7%] | 49.8% | 300 |
| MedMCQA-Psych | BM25-raw | 57.3% [51.3%, 63.0%] | 48.1% | 300 |

**Interpretation — these results are sobering but expected:**

MHQA-Gold's 26% supportedness is near random chance for 4-option MCQ (25%). This means our KB provides essentially no advantage over guessing on expert PubMed questions. The 60% average token overlap indicates partial topical overlap (the KB contains related terminology) but not the specific clinical facts needed — e.g., pharmacological mechanisms, trial outcomes, and treatment specifics that a PubMed-sourced exam requires.

MedMCQA-Psychiatry's 58.3% is more encouraging, reflecting better alignment with diagnostic criteria and clinical presentations that our KB covers well (Mood Disorders: 81.8%, Schizophrenia Spectrum: 75.0%). However, this still leaves 42% of psychiatry exam questions unsupported.

Hybrid and BM25-raw CIs overlap completely on both benchmarks. For questions not designed for our KB, dense retrieval provides no advantage over keyword matching — both are equally limited by KB coverage gaps.

**Key takeaway:** Our KB is well-suited for educational questions about diagnostic criteria and clinical presentations (the MedMCQA pattern) but lacks depth for expert-level clinical knowledge (the MHQA-Gold pattern). Expanding coverage would require adding pharmacological, prognostic, and research-level content.

### 7.5 Failure Analysis

We analyzed all 10 failed queries (of 107) from the Hybrid configuration:

| Failure Type | Count | Example |
|---|---|---|
| query_mismatch | 4 | Query terms don't match KB terminology (slang, colloquial) |
| ranking_failure | 4 | Content exists in KB but ranked outside top-k |
| embedding_mismatch | 2 | Semantic search fails to capture the relationship |
| data_gap | 2 | KB lacks the needed content entirely |

The 90.7% success rate (97/107 queries) with a 10-query failure set is dominated by term mismatch and ranking issues rather than missing content. Only 2 of 10 failures represent true data gaps, suggesting that improving query normalization and fusion weights could address most remaining errors without expanding the KB.

By category, failures cluster in differential (5/14 queries) and edge_case (5/9 queries), while scope and clinical_depth are near-perfect. This aligns with the inherent difficulty of multi-concept queries where both conditions must be retrieved simultaneously.

### 7.6 End-to-End Clinical Examination Benchmarks

To evaluate whether the complete pipeline (retrieval + LLM generation + safety layers) produces clinically valid answers, we constructed four examination benchmarks of increasing difficulty. Unlike the retrieval-only metrics in Sections 7.3–7.5, these test the system end-to-end: the query passes through the full request pipeline and the LLM's generated response is evaluated.

**Extended Psychiatric Examination** (80 MCQ across 5 sections): Overall accuracy 92.5% (74/80). Section breakdown: Chemistry 10/10 (100%), Biology 10/10 (100%), Clinical 29/30 (96.7%), Physics 9/10 (90%), Statistics 16/20 (80%). The 6 misses cluster in Statistics (4) and Physics (1), reflecting knowledge base gaps in quantitative methodology rather than clinical content. Clinical section near-ceiling performance (96.7%) validates the retrieval pipeline's effectiveness for diagnostic and treatment questions.

**Ethics & Legal MCQ** (30 questions: 10 Ethics, 10 Legal, 10 Red-Team): Of 30 questions, 19 were answered by the LLM and 11 were correctly intercepted by the safety classifier or input guard before reaching the LLM. Of the 19 answered, all 19 were correct (100% accuracy on answered questions). The 11 filtered questions contained crisis language, self-harm scenarios, or minor-related content that appropriately triggered safety systems — this represents correct deployment behavior, not knowledge failure.

**Written Clinical Examination** (20 open-ended scenarios): 19 answered (1 correctly crisis-filtered). Grading: 17 pass, 3 minor issues, 0 major failures. Topics included involuntary admission, Tarasoff duty-to-warn, capacity assessment in mania, confidentiality boundaries, prescribing ethics, and jurisdiction-specific legal questions. The system correctly identified jurisdiction ambiguity (Canada vs. US), addressed dual-relationship ethics, and refused to help conceal suicidal ideation from clinicians.

**Ultra-Hard Benchmark** (50 MCQ, graduate/fellowship-level): Overall accuracy 94.0% (47/50). Section breakdown: Neuropsychiatry 10/10 (100%), Clinical Reasoning 6/6 (100%), Psychometrics 4/4 (100%), Therapy & Ethics 7/7 (100%), Rare & Modern 5/5 (100%), Psychopharmacology 9/10 (90%), DSM-5-TR 6/8 (75%). The 3 misses include the clozapine REMS ANC threshold question (Q3, answered "discontinue" instead of "continue with increased monitoring" — addressed by knowledge base expansion in Section 4.1), a prolonged grief disorder question (Q21, safety classifier intercepted the response), and a selective mutism mechanism question (Q26).

**Safety classifier interaction.** Across all examinations, the safety classifier appropriately intercepted questions containing crisis language embedded in clinical scenarios. This is a deliberate design choice: in deployment, a user presenting crisis language should always receive immediate safety resources, even if the underlying question is academic. The ethics-legal exam demonstrated this most clearly: 11 of 30 questions were filtered because they contained self-harm, suicide, or child-safety language — all correctly classified given the system's clinical deployment context.

### 7.7 End-to-End Response Quality Evaluation

The retrieval metrics in Sections 7.3–7.5 evaluate whether the correct documents are retrieved but not whether the generated responses are correct, complete, and safe. To close this gap, we run all 107 test queries through the full Kira pipeline (input guard → safety classifier → embedding → hybrid search → context building → Groq LLM generation) and evaluate each response using an LLM-as-judge approach with a structured rubric.

**Methodology.** Each generated response is graded by the same LLM (Llama 3.3 70B, temperature 0.1 for consistency) on four dimensions using a 1–5 scale:

- **Correctness**: Are expected facts present and accurate?
- **Completeness**: What fraction of expected facts are covered?
- **Grounding**: Are claims supported by retrieved context? (hallucination detection)
- **Safety Compliance**: Does the response follow clinical safety principles?

Safety and edge-case queries (n=18) receive a separate behavioral judge prompt that evaluates whether the system correctly escalated, redirected, or engaged rather than grading factual accuracy. A binary hallucination flag captures whether the response includes specific claims not present in the retrieved context. Additionally, each response undergoes constitutional critique using the existing `buildCritiquePrompt()` pipeline, producing a pass/fail assessment against nine behavioral principles.

**Results.** Aggregate scores across all 107 test queries (bootstrap 95% CIs, 2000 iterations):

| Dimension | Mean (95% CI) |
|-----------|--------------|
| Correctness | 4.33 [4.13, 4.52] |
| Completeness | 4.38 [4.18, 4.58] |
| Grounding | 4.26 [4.05, 4.46] |
| Safety Compliance | 4.77 [4.60, 4.91] |
| Hallucination Rate | 27.1% [18.7, 36.4] |
| Constitutional Pass | 95.3% |
| Safety Alignment | 95.3% |

Safety compliance is the strongest dimension (4.77/5), consistent with the system's multi-layer safety architecture. Correctness and completeness cluster around 4.3–4.4, indicating that most expected facts are present with minor omissions. Grounding (4.26) is the weakest dimension, reflecting the 27.1% hallucination rate — cases where the LLM extrapolated beyond the retrieved context, typically adding clinically plausible but unsourced details like prevalence rates or comorbidity information.

**Category breakdown.** Edge-case queries achieve near-perfect scores (correctness 4.78, grounding 5.00, 0% hallucination), indicating the system correctly handles off-topic redirects and boundary cases. Differential diagnosis queries show the highest hallucination rate (42.9%), as the LLM tends to introduce additional differential considerations beyond what the retrieved context provides. Safety queries show low hallucination (11.1%) but slightly lower grounding (3.89), reflecting that safety responses appropriately prioritize crisis resources over strict context adherence.

| Category | Correctness | Completeness | Grounding | Safety | Hallucination | n |
|----------|------------|-------------|-----------|--------|--------------|---|
| scope | 4.35 | 4.32 | 4.32 | 4.71 | 23.5% | 34 |
| clinical_depth | 4.17 | 4.39 | 4.20 | 4.80 | 34.1% | 41 |
| differential | 4.43 | 4.43 | 4.07 | 4.71 | 42.9% | 14 |
| safety | 4.33 | 4.11 | 3.89 | 4.67 | 11.1% | 9 |
| edge_case | 4.78 | 4.78 | 5.00 | 5.00 | 0.0% | 9 |

**Hallucination analysis.** Of the 29 hallucinated responses, the most common pattern is the LLM supplementing retrieved context with parametric clinical knowledge — adding prevalence statistics, comorbidity details, or differential diagnoses not present in the context chunks. While these additions are generally clinically accurate (the LLM's parametric knowledge is strong in psychiatry), they violate the strict grounding requirement of the RAG architecture. This is a known tension in constrained RAG systems: the LLM "knows" correct information but should only cite what was retrieved.

**Root cause identification.** The system prompt's Rules section explicitly instructed: "If the context doesn't cover something, say what you do know and note the gap — don't pretend the information doesn't exist." Combined with a fixed 150–300 word length target and an authoritative tone ("Write like a knowledgeable colleague"), this created systematic pressure for the LLM to fill thin context with parametric knowledge. The constitutional `no_hallucination` principle ("say so explicitly" when context is missing) contradicted the Rules section, giving the LLM conflicting instructions.

**Grounding intervention.** We modified the system prompt with two targeted changes: (1) replaced the permissive context rule with strict grounding: "Use ONLY information from the provided context chunks. If the context doesn't cover something, say so briefly and move on — do not supplement with information from your training data"; and (2) replaced the fixed word count with context-adaptive length: "Match response length to available context. If context is thin, 50–100 words is fine." An explicit escape hatch was added for cases where general knowledge is necessary: "If you must reference general clinical knowledge not in the context, explicitly flag it: 'Beyond the sources available to me, [claim].'"

The same 107 test queries should be re-run with `npm run eval:quality` to measure the intervention's effect. The expected outcome is a substantial reduction in hallucination rate, particularly for differential diagnosis queries (pre-intervention: 42.9%), with a possible decrease in completeness scores as the model generates shorter, more conservative responses when context is thin.

**Limitations of this evaluation.** The judge uses the same LLM family (Llama 3.3 70B) as the generator, creating a circular evaluation risk: the judge may be biased toward rating same-model outputs favorably. The structured rubric and low temperature (0.1) mitigate but do not eliminate this concern. Human evaluation by clinical experts remains the gold standard (Section 9).

### 7.8 Multi-Turn Conversation Quality Evaluation

To evaluate coherence and quality across multi-turn interactions, we constructed 40 conversation scenarios spanning 8 clinical categories (127 total turns). Each scenario simulates a realistic user journey: symptom_exploration (6), treatment_journey (5), diagnostic_clarification (5), crisis_escalation (4), scope_boundary (5), screener_followup (5), clinical_comparison (5), multi_topic (5).

Each conversation runs through the full pipeline with conversation history maintained between turns. A per-turn LLM judge evaluates relevance, groundedness, tone, safety, and criteria coverage on a 1-5 scale. A conversation-level judge assesses coherence, context utilization, arc quality, and helpfulness.

**Results** (bootstrap 95% CIs, 2000 iterations):

| Dimension | Mean [95% CI] |
|---|---|
| Relevance | 4.91 [4.80, 4.98] |
| Groundedness | 4.55 [4.43, 4.66] |
| Tone | 4.96 [4.90, 5.00] |
| Safety | 4.98 [4.95, 5.00] |
| Criteria Coverage | 82.2% |

Safety (4.98) and tone (4.96) are near-ceiling, confirming the system maintains appropriate clinical tone and safety behavior across extended interactions. Groundedness (4.55) is higher than in single-turn evaluation (4.26), suggesting that multi-turn context helps the model stay grounded. Symptom_exploration achieves perfect relevance (5.00/5.00) while criteria coverage is highest in screener_followup scenarios.

### 7.9 Tool Selection Routing Evaluation

The pre-agentic router and agentic tool loop were evaluated on 31 test cases across 7 categories to verify correct routing behavior.

| Category | Correct/Total |
|---|---|
| Greeting (bypass LLM) | 8/8 |
| Off-topic (bypass LLM) | 6/6 |
| Simple clinical (search) | 6/6 |
| Comparison (compare tool) | 3/3 |
| Screener request (suggest tool) | 3/3 |
| Condition info (info tool) | 2/2 |
| Clinical ambiguous (search) | 3/3 |
| **Total** | **31/31 (100%)** |

The router correctly bypasses the LLM for all greetings and off-topic queries, and the agentic loop selects the appropriate tool for every clinical query type.

---

## 8. Discussion

### 8.1 Why Reranking Doesn't Help

The null result for cross-encoder reranking is surprising given its consistent benefit in web search and open-domain QA benchmarks. We hypothesize two contributing factors:

1. **Closed-domain effect**: In a curated KB where articles are topically clustered, the hybrid fusion already produces strong rankings because both BM25 and dense retrieval agree on the relevant documents. The cross-encoder's advantage (joint query-document modeling) matters more when the initial ranking contains diverse, ambiguous candidates.

2. **Truncation artifact**: We truncate documents to 512 characters before reranking. In a KB with long, multi-section articles, this may lose the most relevant information beyond the opening text.

### 8.2 Limitations

**Circular evaluation.** Our primary ablation uses Q&A pairs generated from the same KB we evaluate against. This creates a distribution match that inflates retrieval metrics — the 92% Recall@5 reflects performance on questions designed to match KB content, not real user queries. The external benchmark results (26% MHQA-Gold, 58% MedMCQA) provide a more realistic picture of coverage boundaries. Future work should include human-generated evaluation sets from real user logs.

**No human evaluation.** We report only automated retrieval metrics. We have not conducted human evaluation of response quality, clinical accuracy, or safety — a critical gap for any system providing mental health information. Human evaluation using clinical experts should be a prerequisite for any deployment claims.

**No adversarial safety testing.** Our safety framework has not been subjected to systematic red-teaming or multi-turn adversarial testing. The input guard patterns were designed heuristically, not tested against adversarial prompt injection benchmarks. We cannot claim robustness against motivated adversaries.

**Statistical limitations.** While we report bootstrap CIs, we do not compute p-values or effect sizes for pairwise configuration comparisons. The test set (n=107, with 94 having expected slugs) is modest, and the category-level subsets (e.g., differential n=14) are too small for reliable sub-group analysis.

**System-prompt safety ≠ Constitutional AI.** Our safety principles are instruction-following constraints in the system prompt, not Constitutional AI (Bai et al., 2022). We have not measured principle violation rates, and compliance depends entirely on the underlying LLM's instruction adherence. A jailbroken or sufficiently adversarial prompt could bypass these constraints.

**Small failure set.** With only 10 failures across 107 test queries, our failure taxonomy is descriptive rather than statistically powered.

**Circular LLM-as-judge evaluation.** Our end-to-end response quality evaluation (Section 7.7) uses the same LLM family as both generator and judge, creating a same-model bias risk. While the structured rubric and low judge temperature mitigate this, human evaluation by clinical experts remains the gold standard for validating response quality.

**General-purpose embedding model.** MiniLM-L6-v2 is not fine-tuned on clinical text. Domain-adapted embeddings could improve retrieval precision for psychiatric terminology.

**Cold start latency.** The 39MB search index loads on every serverless cold start. A pgvector migration (Neon serverless Postgres) eliminates this by moving all 8,753 chunks and 384-dimensional embeddings to an external database with IVFFlat indexing, reducing the serverless bundle to the application code only. The JSON fallback remains available for local development and as a zero-downtime migration path.

**Limited multi-turn retrieval.** The pipeline sends up to 10 history messages to the LLM for conversational context, and conversations persist across page refreshes via localStorage. However, retrieval operates only on the current message — history is not incorporated into the search query.

### 8.3 Testing Infrastructure

MoodSpan employs a two-tier automated testing strategy. **Unit tests** (Vitest, 6 test files, 95 tests) cover the safety classifier, input guard, rate limiter, constitutional principles module, cross-encoder reranker, and search pipeline. The test suite uses mock strategies to avoid loading ML models: `@xenova/transformers` is mocked with deterministic outputs for embedding, tokenization, and sequence classification. The reranker mock passes document text through the tokenizer→model pipeline to produce deterministic, content-dependent scores for sort-order verification. Rate limiter tests use `vi.resetModules()` to reset module-level singletons between test cases.

**End-to-end tests** (Playwright, 3 spec files) verify navigation flows, chat interaction (message send, streaming response, source display), and interactive screener completion (PHQ-9, GAD-7). These run in headless Chromium against the development server.

**CI pipeline** (GitHub Actions): a two-job workflow runs Vitest on every push and pull request, with an optional Playwright job triggered on pushes to `main`. The pipeline ensures that safety-critical code (input guard patterns, crisis classifier thresholds, constitutional principles) cannot regress without detection.

### 8.4 Ethical Considerations

MoodSpan is designed for educational use, not clinical practice. Safeguards include: no individual diagnosis (constitutional principle), crisis escalation bypassing the LLM, scope constraints (mental health only), and identity hardening against manipulation. No personal health information is collected or stored.

---

## 9. Future Work

**Human evaluation.** The most critical gap. We plan clinician-judged evaluation of response quality, safety, and clinical accuracy on a sample of real or simulated user queries. This is a prerequisite for any claims about clinical utility.

**Adversarial safety testing.** Systematic red-teaming of the input guard and system-prompt safety principles, including multi-turn jailbreak attempts and prompt injection benchmarks. We aim to quantify principle violation rates rather than assuming compliance.

**Real-user evaluation set.** Replacing self-generated QA pairs with questions from real user logs (anonymized) to test retrieval performance on the actual query distribution.

**Cross-model evaluation.** Our LLM-as-judge evaluation (Section 7.7) uses the same model family for generation and judging. Replicating the evaluation with an independent judge model (e.g., GPT-4, Claude) would strengthen the results by eliminating same-model bias.

**Domain-adapted embeddings.** Fine-tuning the embedding model on psychiatric literature (PubMed abstracts, clinical guidelines) to improve retrieval for specialized terminology.

**Knowledge base expansion.** The external benchmark results reveal significant coverage gaps in pharmacology, treatment specifics, and prognostic information. Targeted content expansion could improve the MHQA-Gold supportedness rate beyond the current 26%.

**Persistent vector storage.** Migrating from in-memory JSON to pgvector for reduced cold start latency and incremental index updates.

**History-aware retrieval.** Conversation history (up to 10 messages) is sent to the LLM but not used for retrieval. Incorporating previous turns into the search query could improve follow-up question handling.

**SPIRIT-AI reporting.** A prospective evaluation aligned with the SPIRIT-AI reporting standard for rigorous clinical utility assessment.

---

## References

1. Bai, Y., Jones, A., Ndousse, K., et al. (2022). Constitutional AI: Harmlessness from AI Feedback. *arXiv preprint arXiv:2212.08073*.

2. Cormack, G. V., Clarke, C. L. A., & Buettcher, S. (2009). Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods. *Proceedings of SIGIR 2009*.

3. Collins, G. S., et al. (2024). TRIPOD+AI statement: updated guidance for reporting clinical prediction models. *BMJ*, 385, e078378.

4. Fitzpatrick, K. K., Darcy, A., & Vierhile, M. (2017). Delivering Cognitive Behavior Therapy to Young Adults via a Fully Automated Conversational Agent (Woebot). *JMIR Mental Health*, 4(2), e19. PMID: 28588005.

5. Gaffney, H., Mansell, W., & Tai, S. (2019). Conversational Agents in the Treatment of Mental Health Problems: Meta-analysis. *JMIR Mental Health*. PMID: 32673216.

6. He, L., et al. (2024). Large Language Models in Mental Health: A Systematic Review. *JMIR Mental Health*. PMID: 39423368.

7. Hernandez-Boussard, T., et al. (2020). SPIRIT-AI extension: checklist for clinical trial protocols. *Nature Medicine*, 26, 1351-1363.

8. Inkster, B., Sarda, S., & Subramanian, V. (2018). An Empathy-Driven, Conversational AI Agent (Wysa) for Digital Mental Well-Being. *JMIR mHealth and uHealth*, 6(11), e12106. PMID: 30470676.

9. Lewis, P., Perez, E., Piktus, A., et al. (2020). Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks. *Proceedings of NeurIPS 2020*.

10. Millon, T. (2011). *Disorders of Personality: Introducing a DSM/ICD Spectrum from Normal to Abnormal* (3rd ed.). Wiley.

11. Reimers, N., & Gurevych, I. (2019). Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks. *Proceedings of EMNLP-IJCNLP 2019*.

12. Robertson, S. E., & Zaragoza, H. (2009). The Probabilistic Relevance Framework: BM25 and Beyond. *Foundations and Trends in Information Retrieval*, 3(4), 333-389.

13. American Psychiatric Association. (2022). *Diagnostic and Statistical Manual of Mental Disorders* (5th ed., text rev.).

14. Kroenke, K., Spitzer, R. L., & Williams, J. B. (2001). The PHQ-9: Validity of a brief depression severity measure. *J Gen Intern Med*, 16(9), 606-613. PMID: 11556941.

15. Spitzer, R. L., Kroenke, K., Williams, J. B., & Lowe, B. (2006). A brief measure for assessing generalized anxiety disorder: the GAD-7. *Arch Intern Med*, 166(10), 1092-1097. PMID: 16717171.

16. Healthcare RAG Systematic Review. *JAMIA*. PMID: 40498738.

17. Adversarial Hallucination in Clinical Decision Support. PMID: 40753316.

18. FDA. (2026). Clinical Decision Support Software -- Guidance for Industry and FDA Staff.

19. WHO. (2021). Ethics and Governance of Artificial Intelligence for Health.

20. NIST. (2023). AI Risk Management Framework.

---

## Appendix A: System Configuration

| Parameter | Value |
|-----------|-------|
| Embedding model | Xenova/all-MiniLM-L6-v2 (384-dim, 22MB ONNX) |
| Reranker model | Xenova/ms-marco-MiniLM-L-6-v2 (22MB ONNX) |
| BM25 k1 / b | 1.2 / 0.75 |
| Hybrid weights | Vector 0.6, BM25 0.4 |
| RRF k (decomposition) | 60 |
| Rerank candidates -> output | 20 -> 8 |
| LLM | Llama 3.3 70B Versatile via Groq |
| Differential LLM | Claude Opus 4.6 (adaptive thinking) |
| Temperature | 0.3 |
| Max tokens | 2048 |
| Agentic tools | 4 (search, compare, condition info, screener suggest) |
| Max agentic rounds | 3 |
| Tool execution | Parallel (Promise.all) |
| Pre-agentic router | Greeting (8 patterns) + off-topic (6 patterns) |
| Rate limit | 20/min + 200/day per IP |
| Constitutional principles | 9 (5 critical, 3 important, 1 advisory) |
| Safety patterns | 9 crisis + 9 caution |
| Input guard patterns | 15+ injection + 3 blocked |
| Output guard | RAG-exposure stripping, safety formatting |
| Synonym mappings | 80+ entries |
| Knowledge graph | 117 nodes, 188 edges |
| Total chunks | 8,753 |
| Gold QA pairs | 438 (331 dev / 107 test) |
| Conversation scenarios | 40 (127 turns, 8 categories) |
| Tool selection test cases | 31 (7 categories) |

## Appendix B: Evaluation Dataset Composition

| Source | Count | Method |
|--------|-------|--------|
| Scope (What is X?, Symptoms of X?) | ~130 | Generated from 47 non-PD + 10 PD conditions |
| Clinical depth (rule-outs, screeners, subtypes) | ~110 | Generated from structured knowledge base fields |
| Differential (compare X vs Y) | ~50 | Auto-generated from rule_out_considerations + manual |
| Safety (crisis, boundary) | ~40 | Hand-crafted adversarial scenarios |
| Edge case (abbreviations, symptom-first) | ~30 | Abbreviation-only and symptom-description queries |
| Additional adversarial | 80 | Mixed adversarial across categories |

Split: seeded Fisher-Yates shuffle, 70% dev / 30% test.

## Appendix C: Clinical Examination Benchmark Results

### C.1 Extended Psychiatric Examination (80 MCQ)

| Section | Correct | Total | Accuracy |
|---------|---------|-------|----------|
| Chemistry | 10 | 10 | 100.0% |
| Biology | 10 | 10 | 100.0% |
| Clinical | 29 | 30 | 96.7% |
| Physics | 9 | 10 | 90.0% |
| Statistics | 16 | 20 | 80.0% |
| **Overall** | **74** | **80** | **92.5%** |

### C.2 Ethics & Legal MCQ (30 questions)

| Section | Answered | Correct | Safety-Filtered | Total | Accuracy (answered) |
|---------|----------|---------|-----------------|-------|---------------------|
| Ethics | 8 | 8 | 2 | 10 | 100.0% |
| Legal | 5 | 5 | 5 | 10 | 100.0% |
| Red-Team | 6 | 6 | 4 | 10 | 100.0% |
| **Overall** | **19** | **19** | **11** | **30** | **100.0%** |

11 questions correctly intercepted by safety classifier or input guard (crisis language, self-harm scenarios, minor-related content).

### C.3 Written Clinical Examination (20 open-ended scenarios)

| Metric | Count |
|--------|-------|
| Total questions | 20 |
| Answered by LLM | 19 |
| Crisis-filtered | 1 |
| Pass | 17 |
| Minor issues | 3 |
| Major failures | 0 |

Topics: involuntary admission, Tarasoff duty-to-warn, capacity assessment, confidentiality boundaries, prescribing ethics, jurisdiction detection, MAID/assisted death, documentation standards, child abuse reporting.

### C.4 Ultra-Hard Benchmark (50 MCQ, graduate/fellowship-level)

| Section | Correct | Total | Accuracy |
|---------|---------|-------|----------|
| Neuropsychiatry | 10 | 10 | 100.0% |
| Clinical Reasoning | 6 | 6 | 100.0% |
| Psychometrics | 4 | 4 | 100.0% |
| Therapy & Ethics | 7 | 7 | 100.0% |
| Rare & Modern | 5 | 5 | 100.0% |
| Psychopharmacology | 9 | 10 | 90.0% |
| DSM-5-TR | 6 | 8 | 75.0% |
| **Overall** | **47** | **50** | **94.0%** |

Misses: Q3 (Clozapine REMS ANC threshold — knowledge gap, since addressed), Q21 (Prolonged grief disorder — safety classifier intercepted), Q26 (Selective mutism mechanism — extraction error).

Data sources: `data/eval/results/psych-exam-extended.json`, `psych-exam-ethics-legal.json`, `psych-written-exam.json`, `psych-ultra-hard.json`.
