# Results — Causal Alignment Findings

All scores derive from the validated hybrid DAGs (DoWhy-refuted) used as ground truth, and from
the 36-prompt structured evaluation of four frontier LLMs. Methodology is documented in
[methodology.md](methodology.md).

## 1. Cross-dataset DAG robustness summary

| Channel | GES-BN | PC-DNT | DoWhy result | Notes |
|---------|--------|--------|--------------|-------|
| Phishing | Pass | Pass | Success | Full consensus |
| Vishing | Pass | Pass | Success | Edge divergence |
| Smishing | Pass | Fail | Partial | NOTEARS acyclicity failure |

## 2. DoWhy edge validation (phishing)

Every construct→outcome edge passed placebo, random-common-cause, and subset refuters. Minimum
empirical placebo p ≈ 0.002.

| Construct | β (log-odds) | Placebo p | Rand-CC p | Subset p | Verdict |
|-----------|--------------|-----------|-----------|----------|---------|
| Obfuscation | 0.116 | 0.002 | 0.92 | 0.92 | Pass |
| Trust | 0.273 | 0.002 | 0.84 | 0.92 | Pass |
| Urgency | −0.067 | 0.002 | 1.00 | 0.96 | Pass |
| Deception | −0.177 | 0.002 | 0.68 | 0.95 | Pass |
| Authority | −0.441 | 0.002 | 0.94 | 0.98 | Pass |

Prominent validated causal chains:

```
Urgency   → Deception   → Phishing
Trust     → Obfuscation → Phishing
Authority → Deception   → Phishing
```

Deception emerged as a convergent mediator across all discovery methods; obfuscation acted as a
technical amplifier rather than an initiator.

## 3. LLM alignment scores (side by side)

| Model | Alignment (/20) | Fidelity (/60) | DAG alignment % | S_LLM (/5) |
|-------|-----------------|----------------|-----------------|------------|
| GPT-4 | 18.5 | 58.0 | **94.2%** | **4.60** |
| Claude 3 Sonnet | 16.0 | 56.5 | **85.7%** | **4.14** |
| Gemini 2.5 Pro | 14.0 | 45.5 | 72.3% | 3.45 |
| DeepSeek-67B | 10.0 | 34.5 | 53.0% | 2.44 |

## 4. Dimensional breakdown (S_LLM components)

| Model | Awareness | Depth | Structure | Directionality | Generalisability |
|-------|-----------|-------|-----------|----------------|------------------|
| GPT-4 | 5.00 | 4.67 | 4.58 | 4.50 | 4.25 |
| Claude 3 Sonnet | 4.83 | 4.58 | 3.92 | 4.00 | 3.38 |
| Gemini 2.5 Pro | 4.00 | 3.58 | 3.67 | 3.50 | 2.50 |
| DeepSeek-67B | 3.08 | 2.67 | 2.33 | 2.50 | 1.62 |

## 5. Construct-level interpretation fidelity

*A = Awareness (0–5), D = Depth (0–5)*

| Construct | GPT-4 (A/D) | Claude (A/D) | Gemini (A/D) | DeepSeek (A/D) |
|-----------|-------------|--------------|--------------|----------------|
| Deception | 5.0 / 5.0 | 5.0 / 5.0 | 5.0 / 4.5 | 4.0 / 3.5 |
| Urgency | 5.0 / 5.0 | 5.0 / 5.0 | 4.5 / 4.0 | 3.0 / 3.0 |
| Obfuscation | 5.0 / 4.5 | 5.0 / 5.0 | 3.5 / 3.5 | 3.5 / 2.5 |
| Authority | 5.0 / 4.5 | 5.0 / 4.5 | 3.5 / 3.5 | 2.5 / 2.5 |
| Trust | 5.0 / 4.5 | 4.5 / 4.0 | 4.0 / 3.0 | 2.5 / 2.5 |
| Causality | 5.0 / 4.5 | 4.5 / 4.0 | 3.5 / 3.0 | 3.0 / 2.0 |

## 6. Model stability under cross-channel stress

Core causal edges (Urgency → Deception, Trust → Authority) survived perturbation across e-mail
and voice datasets, supporting Pearl's criterion that genuine causal relations should survive
surface perturbation. Top models lost up to **0.42 depth points** when features drifted to the
synthetic vishing set; misclassifications clustered around the low-depth constructs (trust,
obfuscation).

DeepSeek-67B systematically under-weighted obfuscation and mis-ranked trust (treating trust cues
as protective rather than exploitable).

## 7. Inter-rater reliability

**ICC(2,1) = 0.98** (95% CI ≈ 0.94–0.99) — "almost perfect" (Shrout and Fleiss, 1979).
Individual-dimension ICC ranged from 0.89 (Directionality) to 0.97 (Structure).

## 8. Central finding

Current frontier LLMs detect surface patterns and reproduce coarse causal structure, but
consistently struggle with multi-hop causal chains and inverse reasoning. GPT-4 approaches
human-level causal fluency in controlled conditions; lower-ranked models revert to correlational
explanation that would fail under adversarial paraphrase. The persistent gap between **awareness**
and **depth** scores across all models indicates that construct recognition is necessary but
insufficient for reliable causal defence.
