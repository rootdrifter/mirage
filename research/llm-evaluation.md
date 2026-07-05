# LLM Evaluation Framework

This document details the structured LLM-interrogation pathway: how the 36 prompts are organised,
how each response is scored, and how scoring reliability is computed. It expands the summary in
[methodology.md](methodology.md) and the scores in [results.md](results.md).

The validated, DoWhy-refuted hybrid DAG is the ground truth. Each model's task is to reconstruct
and reason over that causal structure; scores measure how closely its reasoning matches the
verified graph, not whether it can label a message "phishing".

## Prompt batches

The 36 structured JSON prompts are organised into **seven prompt batches**. The batches are a
finer-grained decomposition of the five reasoning categories reported in
[methodology.md](methodology.md) — they sum to the same 36 prompts.

| # | Prompt batch | What it tests | Reasoning category (methodology.md) | Count |
|---|--------------|---------------|-------------------------------------|-------|
| 1 | Probability inference | Estimate the likelihood a construct drives the malicious outcome | Probability | 6 |
| 2 | Conditional reasoning | Reason about P(outcome \| construct present/absent); conditional dependence | Conditional | 6 |
| 3 | Construct absence | Reason about what happens when a known causal construct is removed | Inverse Reasoning | 3 |
| 4 | Reversed-direction causality | Detect and reject an inverted cause→effect arrow | Inverse Reasoning | 3 |
| 5 | Direct edge testing | Confirm/deny a single construct→outcome edge present in the DAG | Fixed | 6 |
| 6 | Causal chains | Reconstruct multi-hop mediated chains (e.g. Urgency → Deception → Phishing) | Fixed | 6 |
| 7 | Construct ranking | Rank constructs by causal influence on the outcome | Impact Ranking | 6 |
| | **Total** | | | **36** |

Batches 3–4 are the **inverse / adversarial** conditions (construct removed, or causal arrow
reversed) — the hardest tests, designed to separate genuine causal reasoning from pattern recall.
Batches 5–6 are the 12 "Fixed" prompts: direct edge recognition plus multi-step chain inference.

## Scoring dimensions

Each response is scored on **five equally weighted dimensions** (wᵢ = 0.2), giving a composite
`S_LLM = Σ(wᵢ × dᵢ)` on a 0–5 scale. The dimension labels below are the rubric's analytic names;
the equivalent short labels used in [results.md](results.md) are given for cross-reference.

| Dimension | Short label (results.md) | What it measures |
|-----------|--------------------------|------------------|
| Construct awareness | Awareness | Is the relevant construct explicitly named as causally involved? |
| Causal depth | Depth | Does the explanation reach a psychological mechanism, beyond correlation? |
| Probabilistic / structural logic | Structure | Is the causal/probabilistic chain coherent and acyclic, with sound likelihood reasoning? |
| Directionality | Directionality | Do cause→effect arrows stay correct, including under reversed-direction prompts? |
| Cross-dataset generalisation | Generalisability | Does the reasoning hold when features drift or the channel changes (email → SMS → voice)? |

The dimensions are deliberately matched to the batches: probability/conditional batches exercise
**probabilistic logic**; construct-absence and reversed-direction batches exercise
**directionality** and **causal depth**; cross-channel prompts exercise **generalisation**. A
model can score highly on awareness while failing depth — the persistent awareness/depth gap is
the central finding in [results.md](results.md).

## Composite score

```
S_LLM = Σ (w_i × d_i),   w_i = 0.2 for all five dimensions,   d_i ∈ [0, 5]
```

Reported alongside `S_LLM` are an **Alignment** score (/20) and a **Fidelity** score (/60)
aggregating edge-level agreement with the validated DAG, and a **DAG alignment %** — the fraction
of expert-validated graph edges the model reproduces (GPT-4 94.2%, Claude 3 Sonnet 85.7%).

## Inter-rater reliability — ICC(2,1)

Scoring used multiple human raters, so reliability is quantified before the scores are trusted.

- **Model:** two-way random-effects, single rater/measurement, absolute agreement —
  **ICC(2,1)** in the Shrout and Fleiss (1979) convention.
- **Design:** two rating waves × four raters × five scoring dimensions.
- **Tooling:** `pingouin.intraclass_corr`.

Why ICC(2,1): raters are treated as a random sample (Type 2), and reliability of a **single**
rater's score is what we report (the "1" in 2,1), under absolute-agreement rather than mere
consistency — appropriate when scores are used at face value across raters.

Conceptually, ICC(2,1) partitions variance into between-target (genuine signal across the scored
responses), between-rater, and residual error:

```
ICC(2,1) = (MS_R − MS_E) / ( MS_R + (k − 1)·MS_E + (k/n)·(MS_C − MS_E) )
```

where `MS_R` = between-targets mean square, `MS_C` = between-raters mean square, `MS_E` = residual
mean square, `k` = number of raters, `n` = number of targets.

**Result: ICC(2,1) = 0.98 (95% CI ≈ 0.94–0.99)** — "almost perfect" agreement. Per-dimension ICC
ranged from **0.89 (Directionality)** — the hardest dimension to score consistently, as expected
for the adversarial reversed-direction prompts — to **0.97 (Structure)**. The high overall ICC
means the model rankings in [results.md](results.md) reflect model behaviour, not rater noise.

## Execution controls (determinism)

| Model | Access | Sampling |
|-------|--------|----------|
| GPT-4 (0613) | Secured desktop environment | Lowest-entropy default preset |
| Claude 3 Sonnet | Web interface | "Precise" preset |
| Gemini 2.5 Pro | Web interface | Default deterministic preset |
| DeepSeek-67B | 2 × RunPod H100 SXM (80 GB VRAM, 16 vCPU, 125 GB RAM) | temperature 0.70, top-p 0.95, max tokens 1024 |

Fixing sampling parameters keeps responses reproducible so that score differences reflect model
capability rather than decoding randomness.
