# Methodology — Dual-Pathway Causal Evaluation Framework

The framework runs two pipelines that deliberately converge at a **causal-alignment layer**:
statistically inferred causal edges (graph pathway) are compared against the causal structure
that frontier LLMs articulate (LLM pathway). The central question is not *whether* a model
detects phishing, but whether its reasoning about *why* a message persuades matches verified
causal ground truth.

## 1. Dataset engineering pipeline

Three datasets are cleaned and mapped to a shared five-construct schema so that score
differences reflect reasoning skill rather than schema mismatch. Full dataset detail is in
[datasets.md](datasets.md).

| Stage | Action |
|-------|--------|
| Acquisition | Phishing (real corpus), smishing (engineered rows), vishing (CVAE-synthesised) |
| Cleaning | Per-channel deduplication and normalisation |
| Construct mapping | Each technical feature assigned to a behavioural construct via YAML maps (`phishing.yaml`, `smishing.yaml`, `vishing.yaml`) |
| Reuse | YAML-driven config supports new threat domains without code change |

The vishing corpus is generated with a Conditional Variational Auto-Encoder (CVAE) that
preserves the latent causal scaffold while removing channel-specific surface cues — enabling a
fair cross-channel comparison with no privacy risk.

## 2. Construct mapping — five behavioural constructs

Technical indicators are translated into five psychology-grounded constructs:

| Construct | Representative technical features |
|-----------|-----------------------------------|
| Obfuscation | Shortened URLs, redirect count, resolved-IP count |
| Trust | URL in Google index, TLS/SSL certified, domain in Google index |
| Authority | Email address embedded within URL |
| Deception | Time of domain activation (domain age) |
| Urgency | Website response time |

Construct scores are the normalised mean of contributing features
(`C_j = (1/|S_j|) Σ f_i m_ij`), with z-score and sum variants retained for sensitivity checks.

## 3. DAG construction — four algorithms, two ensembles

Four causal-discovery algorithms attack each dataset independently, then merge into two hybrid
ensembles. Hyper-parameters are frozen in `dag_config.yaml` with channel overrides.

| Algorithm | Type | Key parameter |
|-----------|------|---------------|
| GES (Greedy Equivalence Search) | Score-based | — |
| PC (Peter–Clark) | Constraint-based | α = 0.05 |
| Bayesian Networks | Probabilistic | — |
| DeepNOTEARS | Gradient-descent structure learning | L1 = 0.01 |

- Ensembles: **GES ∪ BN** and **PC ∪ DeepNOTEARS**.
- Post-processing removes bidirectional and cyclic edges; behaviourally grounded conflicting
  edges are retained. All intermediate graphs are stored for rollback.
- Discovery runs from `causal_inference_framework.ipynb`, branching on dataset tags and skipping
  feature modules when columns are absent.

## 4. DoWhy validation methodology

Every construct→outcome edge passes through the DoWhy four-stage pipeline:

1. **State assumptions** — encode the causal graph.
2. **Identify the estimand** — derive the target causal quantity.
3. **Estimate** — binomial GLM.
4. **Refute** — robustness checks.

Refutation methods:

- **Placebo treatment** — n = 500 Monte-Carlo permutations; minimum empirical p ≈ 0.002.
- **Random common cause** — inject a random confounder; estimate should be stable.
- **Bootstrapped subset** — re-estimate on data subsets.

Only edges surviving all refuters enter the validated DAG used as LLM ground truth.

## 5. Prompt engineering approach

Four frontier models — **GPT-4, Claude 3 Sonnet, Gemini 2.5 Pro, DeepSeek-67B** — receive **36
structured JSON prompts** distributed across five reasoning categories (Probability,
Conditional, Impact Ranking, Inverse Reasoning, and 12 Fixed prompts). Each response is scored
across five equal-weight dimensions (Awareness, Depth, Structure, Directionality,
Generalisability); composite `S_LLM = Σ(w_i × d_i)`, w_i = 0.2.

**Execution controls (determinism):**

| Model | Access | Sampling |
|-------|--------|----------|
| GPT-4 (0613) | Secured desktop environment | Lowest-entropy default preset |
| Claude 3 Sonnet | Web interface | "Precise" preset |
| Gemini 2.5 Pro | Web interface | Default deterministic preset |
| DeepSeek-67B | Two RunPod H100 SXM GPU pods (80 GB VRAM, 16 vCPU, 125 GB RAM) | temperature 0.70, top-p 0.95, max tokens 1024 (`run_deepseek.py`) |

## 6. Inter-rater reliability

Scoring reliability uses a two-way random intraclass correlation **ICC(2,1)**, computed with
`pingouin.intraclass_corr` across two rating waves, four raters, and five dimensions. See
[results.md](results.md) for the reported value.
