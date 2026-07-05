# mirage

Dissertation-level research evaluating whether frontier large language models can move beyond
pattern-matching to reason causally about the psychological mechanisms that drive social
engineering success. The central question: can an LLM not just flag a phishing message, but
explain *why* it works?

---

## Overview

Social-engineering attacks — phishing, smishing, vishing — succeed because they exploit
cognitive heuristics rather than software vulnerabilities. Existing detection systems model
e-mails as unordered feature collections, flagging lexical anomalies and blacklist matches
while ignoring the causal drivers — urgency, authority, trust — that persuade recipients to
comply. Verizon's 2024 DBIR finds the human element present in 68% of confirmed breaches.

This research fuses two analytical traditions that are rarely combined:

1. **Causal graph discovery** — automated construction of Directed Acyclic Graphs (DAGs) from
   real phishing data using an ensemble of four algorithms, validated with DoWhy refutation testing.
2. **Structured LLM interrogation** — systematic probing of four frontier models against those
   same graphs to measure how closely their explanations replicate verified causal structure.

The dual-pathway design produces a concrete benchmark: not whether models detect phishing, but
whether their reasoning about *why* a message is persuasive matches statistically inferred
ground truth. This distinction matters for audit-ready, explainable security tooling.

---

## Threat Model and Objectives

**What attackers do:** Social-engineering campaigns exploit psychological constructs — urgency,
authority, trust — to suppress deliberation and elevate click-through rates. Campaigns rotate
semantics (substituting "final notice" for "urgent") to defeat static feature weights, forcing
detectors to chase tokens rather than the underlying manipulation mechanism.

**The defensive gap:** Current ML filters over-fit correlational patterns. A detector that learns
"URGENT" correlates with phishing will fail when an attacker switches to "time-sensitive" or moves
to voice. Causal reasoning — identifying *what causes* compliance, not what co-occurs with it —
produces defences that survive adversarial paraphrase.

**Research objectives:**

- O1: Generate hybrid DAGs capturing causal relationships among five behavioural constructs across
  phishing, smishing, and vishing datasets.
- O2: Benchmark GPT-4, Claude 3 Sonnet, Gemini 2.5 Pro, and DeepSeek-67B against those DAGs using
  36 structured prompts aligned to the five constructs.
- O3: Quantify model strengths, weaknesses, and failure modes in causal reasoning.
- O4: Identify where current LLMs are fit for causally-informed detection pipelines and where
  fine-tuning or hybrid graph-LLM ensembles remain necessary.

---

## Methodology

### Datasets

Three datasets were evaluated, each mapped to the same five-construct schema so that differences
in scores reflect reasoning skill rather than schema mismatch:

| Channel           | Raw rows | Cleaned rows | Malicious % | Features |
|-------------------|----------|--------------|-------------|----------|
| E-mail phishing   | 88,647   | 59,788       | 31.15%      | 10       |
| SMS smishing      | 67,008   | 67,008       | 39.07%      | 10       |
| Synthetic vishing | 60,000   | 60,000       | 30.00%      | 10       |

The phishing corpus (Vrbančič, Fister and Podgorelec, 2020) provides the primary benchmark.
The smishing set (Salman, Ikram and Kaafar, 2024) covers evasion-optimised SMS. The vishing
simulation was synthesised using a Conditional Variational Auto-Encoder (CVAE), preserving the
latent causal scaffold while stripping channel-specific surface artefacts.

All records are anonymised. No model was asked to generate live phishing content — prompts
referenced behavioural constructs only.

### Feature-to-Construct Mapping

Technical indicators were translated into five behavioural constructs grounded in social-engineering
psychology:

| Construct    | Representative technical features                              |
|--------------|----------------------------------------------------------------|
| Obfuscation  | Shortened URLs, quantity redirects, quantity resolved IPs      |
| Trust        | URL in Google Index, TLS-SSL certified, domain in Google Index |
| Authority    | Email address embedded within URL                              |
| Deception    | Time of domain activation (domain age)                         |
| Urgency      | Website response time                                          |

Construct scores were computed as the normalised mean of contributing feature values
(`C_j = (1/|S_j|) Σ f_i m_ij`), with z-score and sum variants available for sensitivity checks.

### Graph Pipeline — Hybrid DAG Construction

Four causal-discovery algorithms attacked each dataset independently:

- **GES** (Greedy Equivalence Search) — score-based, handles high-dimensional sparse data
- **PC-Algorithm** (Peter–Clark) — constraint-based, conditional-independence testing (α = 0.05)
- **Bayesian Networks** — probabilistic, reasons under noisy conditions
- **DeepNOTEARS** — gradient-descent structure learning, captures non-linear relationships (L1 = 0.01)

Graphs were merged into two hybrid ensembles: **GES ∪ BN** and **PC ∪ DeepNOTEARS**. Bidirectional
and semantically absurd edges were pruned; behaviourally grounded conflicting edges were retained.
All intermediate graph iterations were stored for rollback and comparison.

Cross-dataset DAG robustness results:

| Channel | GES-BN | PC-DNT | DoWhy result | Notes                  |
|---------|--------|--------|--------------|------------------------|
| Phish   | Pass   | Pass   | Success      | Full consensus         |
| Vish    | Pass   | Pass   | Success      | Edge divergence        |
| Smish   | Pass   | Fail   | Partial      | NOTEARS acyclicity fail|

### DoWhy Validation

Every construct→outcome edge entered the DoWhy four-stage pipeline: stating assumptions,
identifying the estimand, estimating with a binomial GLM, and refuting via robustness checks.
Refutation methods included n = 500 Monte-Carlo placebo permutations, random-common-cause
injection, and bootstrapped subset tests. Minimum empirical p across placebo runs: p ≈ 0.002.
All five constructs passed:

| Construct    | β (log-odds) | Placebo p | Rand-CC p | Subset p | Verdict |
|--------------|--------------|-----------|-----------|----------|---------|
| Obfuscation  |  0.116       | 0.002     | 0.92      | 0.92     | Pass    |
| Trust        |  0.273       | 0.002     | 0.84      | 0.92     | Pass    |
| Urgency      | −0.067       | 0.002     | 1.00      | 0.96     | Pass    |
| Deception    | −0.177       | 0.002     | 0.68      | 0.95     | Pass    |
| Authority    | −0.441       | 0.002     | 0.94      | 0.98     | Pass    |

The validated hybrid DAGs served as ground truth for all subsequent LLM scoring.

### Prominent causal chains in the validated DAG

```
Urgency    → Deception → Phishing
Trust      → Obfuscation → Phishing
Authority  → Deception → Phishing
```

Deception emerged as a convergent mediator across all discovery methods. Obfuscation acted
as a technical amplifier — enabling rather than initiating manipulation.

### LLM Pipeline — Structured Prompt Evaluation

Four frontier models were selected to span instruction-tuning styles and parameter budgets:
**GPT-4**, **Claude 3 Sonnet**, **Gemini 2.5 Pro**, **DeepSeek-67B**.

Models received 36 structured JSON prompts distributed across five reasoning categories:

| Prompt category  | Instances | Proportion |
|------------------|-----------|------------|
| Probability      | 6         | 16.7%      |
| Conditional      | 6         | 16.7%      |
| Impact Ranking   | 6         | 16.7%      |
| Inverse Reasoning| 6         | 16.7%      |
| Fixed Prompts    | 12        | 33.3%      |

Fixed prompts targeted direct edge recognition, multi-step causal chain inference, and construct
prioritisation. Inverse prompts tested whether models could reason under reversed or absent
constructs — the hardest adversarial condition.

Each response was scored across five equally weighted dimensions:

- **Awareness** — construct explicitly named as causally relevant
- **Depth** — explanation moves beyond correlation to a psychological mechanism
- **Structure** — causal chain is coherent and acyclic
- **Directionality** — arrows stay correct when prompts reverse cause/effect
- **Generalisability** — reasoning holds when features drift or channel changes

Composite score: `S_LLM = Σ(w_i × d_i)`, equal weights (w_i = 0.2).

DeepSeek-67B was evaluated on two RunPod H100 SXM GPU pods (80 GB VRAM, 16 vCPU, 125 GB RAM)
at fixed sampling parameters (temperature 0.70, top-p 0.95, max tokens 1024). All other models
were queried via their respective web interfaces with deterministic/low-temperature presets.

### Inter-rater Reliability

Scoring reliability was assessed with a two-way random intraclass correlation ICC(2,1), computed
across two rating waves, four raters, and five scoring dimensions:

**ICC(2,1) = 0.98 (95% CI ≈ 0.94–0.99)** — classified as "almost perfect" (Shrout and Fleiss, 1979).

Individual dimension ICC values ranged from 0.89 (Directionality) to 0.97 (Structure).

---

## Results and Findings

### DAG Alignment by Model

| Model           | Alignment (/20) | Fidelity (/60) | DAG alignment % | S_LLM (/5) |
|-----------------|-----------------|----------------|-----------------|------------|
| GPT-4           | 18.5            | 58.0           | **94.2%**       | **4.60**   |
| Claude 3 Sonnet | 16.0            | 56.5           | **85.7%**       | **4.14**   |
| Gemini 2.5 Pro  | 14.0            | 45.5           | 72.3%           | 3.45       |
| DeepSeek-67B    | 10.0            | 34.5           | 53.0%           | 2.44       |

GPT-4 achieved the highest composite score (4.60/5), reproducing 94.2% of expert-validated graph
edges and maintaining coherence under counter-factual prompts. Claude 3 Sonnet followed at 4.14/5.

### Dimensional breakdown (S_LLM components)

| Model           | Awareness | Depth | Structure | Directionality | Generalisability |
|-----------------|-----------|-------|-----------|----------------|-----------------|
| GPT-4           | 5.00      | 4.67  | 4.58      | 4.50           | 4.25            |
| Claude 3 Sonnet | 4.83      | 4.58  | 3.92      | 4.00           | 3.38            |
| Gemini 2.5 Pro  | 4.00      | 3.58  | 3.67      | 3.50           | 2.50            |
| DeepSeek-67B    | 3.08      | 2.67  | 2.33      | 2.50           | 1.62            |

### Construct-level interpretation fidelity

| Construct    | GPT-4 (A/D) | Claude (A/D) | Gemini (A/D) | DeepSeek (A/D) |
|--------------|-------------|--------------|--------------|----------------|
| Deception    | 5.0 / 5.0   | 5.0 / 5.0    | 5.0 / 4.5    | 4.0 / 3.5      |
| Urgency      | 5.0 / 5.0   | 5.0 / 5.0    | 4.5 / 4.0    | 3.0 / 3.0      |
| Obfuscation  | 5.0 / 4.5   | 5.0 / 5.0    | 3.5 / 3.5    | 3.5 / 2.5      |
| Authority    | 5.0 / 4.5   | 5.0 / 4.5    | 3.5 / 3.5    | 2.5 / 2.5      |
| Trust        | 5.0 / 4.5   | 4.5 / 4.0    | 4.0 / 3.0    | 2.5 / 2.5      |
| Causality    | 5.0 / 4.5   | 4.5 / 4.0    | 3.5 / 3.0    | 3.0 / 2.0      |

*A = Awareness (0–5), D = Depth (0–5)*

### Model-specific failure modes

**GPT-4** produced the most robust and internally consistent causal structures, accurately
identifying mediating nodes and demonstrating deep awareness of construct interactions.
Superior performance in chain construction, ranking tasks, and inverse reasoning. Only model
to consistently attempt genuine causal explanation rather than correlational description.

**Claude 3 Sonnet** performed strongly on deception and urgency, with moderate success in
multi-step chains. Occasionally lacked deeper abstraction in comparative tasks but maintained
high logical consistency. Faltered when reasoning required linking constructs into longer causal
sequences.

**Gemini 2.5 Pro** displayed consistent construct recognition with reduced interpretive depth.
Responses were frequently descriptive rather than inferential — particularly in prioritisation
and causal explanation tasks. Struggled with inverse-direction and missing-construct prompts.

**DeepSeek-67B** demonstrated basic awareness but underperformed in reasoning depth. Misranked
trust and urgency; treated trust cues as protective rather than exploitable; failed to map key
indirect causal chains. Responses became less stable under adversarial prompt twists despite
its parameter scale.

### Cross-dataset stress tests

Core causal edges — Urgency → Deception and Trust → Authority — survived perturbation across
e-mail and voice datasets, supporting Pearl's (2009) criterion that genuine causal relations
should survive surface perturbations.

Top models lost up to 0.42 depth points when features drifted to the synthetic vishing set,
revealing fragility in real-world cross-channel transfer. Misclassifications clustered around
low-depth constructs (trust, obfuscation), reinforcing that causal depth and alignment are
partially linked.

### Central finding

Current frontier LLMs can detect surface patterns in social engineering and reproduce coarse
causal structure, but consistently struggle with multi-hop causal chains and inverse reasoning.
GPT-4 approaches human-level causal fluency in controlled conditions; models at the lower end
of the benchmark revert to correlation-driven explanations that would fail under adversarial
paraphrase. The gap between awareness and depth scores — visible across all models — indicates
that construct recognition is a necessary but insufficient condition for reliable causal defence.

---

## Adversarial Evasion — Why Causal Detection Resists It

This is the practical pay-off of the whole project, stated plainly enough for a non-specialist.

A **correlational** detector — essentially every mainstream phishing filter — learns from the
*surface* of today's phishing: the word "URGENT", a particular sender format, a link shape, a subject
line. It works until the attacker changes those surface features, and the attacker *can*. An adversary
with access to the detector (or a copy of it) sends variation after variation, watches which ones score
below the block threshold, and keeps the changes that get through. This is a cheap hill-climbing attack.
Swap "URGENT" for "time-sensitive", restructure the sender line, reshape the URL — the message still
works on the human, but the filter no longer recognises it. The surface is exactly what the attacker
controls, so a detector that depends on the surface is one the attacker can tune their way past.

A **causal** detector is built differently. Instead of learning *how phishing looks*, it learns *what
makes phishing succeed*: the underlying mechanisms — urgency, authority, trust, deception — that cause a
recipient to comply. One sentence makes it robust:

> The surface features can be changed freely; the causal structure cannot be changed without defeating
> the attack itself.

If the attacker strips out the urgency, the authority, and the deception to evade a causal detector,
they have removed the very levers that make the recipient act — so there is no successful phishing
message left to catch. The evasion that beats a correlational filter (paraphrase the surface) is
precisely the evasion a causal detector is immune to, because it was never looking at the surface. This
is Pearl's criterion in operational form: a genuine causal relation survives surface perturbation — and
in the cross-channel stress tests the core edges (Urgency→Deception, Trust→Authority) **did** survive a
move from e-mail all the way to synthetic voice.

The research question is therefore not "is causal detection a nice idea" but "can today's LLMs actually
do the causal reasoning it requires." The benchmark answers honestly: **GPT-4 reproduced 94.2%** of the
validated causal structure and held up under reversed prompts; the weakest model (**DeepSeek-67B,
53.0%**) fell back to correlational description — which means, deployed as a detector, it would be
evadable in exactly the way above. Causal robustness is only as strong as the reasoner that implements
it, and measuring that gap is the contribution.

---

## Dataset Scale — What 88,647 Records Buys

The primary corpus is **88,647 raw phishing e-mails** (59,788 after cleaning), alongside 67,008
smishing and 60,000 synthetic vishing records. Scale is not vanity here; it is what makes the
statistical claims defensible.

- **Stable estimates and tight intervals.** With tens of thousands of records per channel, the
  construct→outcome effect sizes (the log-odds β in the DoWhy table) are precise enough to be trusted
  *and* refuted. The reliability ceiling shows it: ICC(2,1) = 0.98 with a 95% CI of ≈ 0.94–0.99 — a
  narrow interval that only large, consistent samples produce.
- **Power to detect small and inverse effects.** Several constructs act through *negative* or indirect
  log-odds (Authority β = −0.441, Urgency β = −0.067). Effects that subtle vanish into the noise of a
  few hundred samples; at this scale they are stable enough that all five constructs pass placebo
  testing at p ≈ 0.002 (n = 500 Monte-Carlo permutations each).
- **Robust structure discovery.** Causal-discovery algorithms (GES, PC, Bayesian Networks, DeepNOTEARS)
  are data-hungry — sparse data yields unstable, contradictory graphs. The cross-method consensus that
  lets edges be *merged* into a validated hybrid DAG only emerges from a large, representative sample.
- **A meaningful held-out test.** Scale is what let the synthetic-vishing cross-channel stress test
  mean something — the causal edges were shown to survive a channel shift rather than being an artefact
  of one small dataset.

**What a smaller dataset would have missed.** With a few hundred e-mails the pipeline would likely have
surfaced the one or two loudest correlations ("URGENT" ≈ phishing) and stopped — the exact brittle,
correlational signal this project set out to move beyond. The subtle mediators (Deception as the
convergent mediator; Obfuscation as a technical *amplifier*, not an initiator), the negative-coefficient
constructs, and the cross-channel robustness all need the statistical power that 88,647 records provide.
Scale is what turns "a plausible story about phishing" into "a refuted, reproducible causal model."

*(Honesty note: 88,647 is the raw corpus; 59,788 records remain after cleaning. Both are reported so the
scale claim is not overstated.)*

---

## Limitations and Future Work

Intellectual honesty strengthens the result; these are the genuine limits.

- **Human-rated scoring.** The S_LLM dimensions were rated against a fixed rubric by multiple raters.
  ICC(2,1) = 0.98 shows the rating was highly consistent, but rubric-based scoring of "depth" and
  "generalisability" retains an irreducible element of judgement.
- **Model versioning / reproducibility.** Three of the four models (GPT-4, Claude 3 Sonnet, Gemini 2.5
  Pro) were queried through their web interfaces, not pinned API snapshots; only DeepSeek-67B ran on
  controlled hardware. Frontier models change underneath their names, so the scores are a **snapshot of
  that generation**, not a permanently reproducible constant.
- **Construct→feature mapping is a modelling choice.** Proxies such as *Urgency ≈ website response time*
  or *Deception ≈ domain age* are defensible operationalisations, not ground truth; a different mapping
  could shift edge weights.
- **Synthetic vishing.** The voice channel was CVAE-generated to preserve the latent causal scaffold
  without processing real personal voice data (a deliberate privacy choice). It is therefore a model of
  vishing structure, not captured real-world vishing, and the −0.42 depth drop there may partly reflect
  that.
- **One weaker graph.** The smishing DAG was only partially validated (PC ∪ DeepNOTEARS failed the
  NOTEARS acyclicity check), so cross-channel claims rest most firmly on the e-mail and voice graphs.
- **Probe-set size.** 36 structured prompts across four models is a focused probe, not an exhaustive
  one; a larger prompt bank and more models would tighten the conclusions.
- **Feasibility, not deployment.** This work *benchmarks* whether LLMs can do the causal reasoning a
  robust detector would need (objective O4). It does **not** ship a live causal-informed detector;
  building and red-teaming one is future work.

**A follow-up study should:** pin model versions via fixed API snapshots; expand the prompt bank and
model set; validate the construct→feature mappings against human-labelled ground truth; test on real
(not synthetic) vishing; and build an actual causal-informed detector to **measure** evasion resistance
empirically — turning the argument above from a sound deduction into a red-teamed result.

---

## Skills Demonstrated

| Skill area                  | Evidence                                                                        |
|-----------------------------|---------------------------------------------------------------------------------|
| Causal inference            | Hybrid DAG construction via GES, PC, Bayesian Networks, DeepNOTEARS             |
| Statistical validation      | DoWhy four-stage refutation pipeline; n=500 Monte-Carlo placebo tests           |
| ML security research        | Applied causal methods to 88,647-record real-world phishing corpus              |
| LLM evaluation              | Designed and executed 36-prompt structured evaluation across four frontier models|
| Quantitative analysis       | SLLM composite scoring, ICC inter-rater reliability, re-weighting sensitivity   |
| Threat modelling            | Mapped technical indicators to psychological manipulation constructs             |
| Adversarial robustness      | Articulated why causal structure resists the surface paraphrase that defeats correlational filters |
| Cross-channel generalisation| Tested robustness across e-mail, SMS, and voice attack surfaces                 |
| Experimental design         | Dual-pathway framework isolating graph discovery from LLM interpretive fidelity |
| Research communication      | Dissertation-level writeup; structured rubrics with defined scoring dimensions  |
| Python / data engineering   | YAML-driven modular pipeline; reproducible HPC execution; DAG version control   |

---

*Part of the [rootdrifter](https://github.com/rootdrifter) security portfolio — built and maintained by a security-cleared candidate. UK-issued clearance held now, not pending vetting: deployable to cleared work from day one.*
