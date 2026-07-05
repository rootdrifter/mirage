# Causal Inference Background — Reference

Theory reference for the methods used in MIRAGE (see [methodology.md](methodology.md),
[results.md](results.md), [llm-evaluation.md](llm-evaluation.md)). Explains DAG theory, the four
discovery algorithms, the DoWhy refutation framework, and ICC(2,1). Background/theory document —
it cites the engagement's reported figures but does not introduce new ones.

---

## 1. DAG theory

### 1.1 Formal definition

A **Directed Acyclic Graph (DAG)** `G = (V, E)` has vertices `V` (here, the five behavioural
constructs plus the phishing outcome) and directed edges `E ⊆ V × V`, with **no directed cycle**.
An edge `X → Y` encodes a direct causal dependency of `Y` on `X` relative to the other modelled
variables. Acyclicity is what lets the joint distribution factorise (§1.3) and what makes "cause
precedes effect" coherent — it is why the pipeline's post-processing **removes bidirectional and
cyclic edges** (methodology.md §3) and why DeepNOTEARS' acyclicity constraint can *fail* on a
channel (the smishing NOTEARS acyclicity failure in results.md §1).

### 1.2 The causal Markov condition

In a causal DAG, each variable is **independent of its non-descendants given its parents**:

```
X ⫫ nondescendants(X) | parents(X)
```

This is the bridge from graph structure to testable statistical independence. It licenses the
factorisation `P(V) = Π_i P(X_i | parents(X_i))` — the basis of Bayesian-network parameterisation
(§2.3).

### 1.3 d-separation

**d-separation** is the graphical criterion that reads conditional independencies off the DAG. A
path between `X` and `Y` is *blocked* by a conditioning set `Z` if it contains:

- a **chain** `A → B → C` or **fork** `A ← B → C` with the middle node `B ∈ Z` (conditioning blocks
  it), or
- a **collider** `A → B ← C` where `B ∉ Z` **and** no descendant of `B` is in `Z` (a collider blocks
  by default; conditioning on it *opens* the path — the classic collider/selection-bias trap).

If every path between `X` and `Y` is blocked given `Z`, then `X ⫫ Y | Z`. d-separation is what
**constraint-based** discovery (PC, §2.2) exploits: it runs conditional-independence tests and keeps
only the graph structures consistent with the observed independencies.

### 1.4 Faithfulness assumption

**Faithfulness** asserts that the *only* conditional independencies in the data are those implied by
d-separation in the true DAG — i.e. there are no "accidental" independencies arising from causal
effects that exactly cancel. Discovery algorithms assume it so that "independence in the data" can
be read as "d-separation in the graph." Where faithfulness is violated (near-cancelling paths,
deterministic relations, small samples), edges can be missed or spurious — one reason MIRAGE
**validates** discovered edges with DoWhy (§3) rather than trusting discovery output alone.

### 1.5 Markov equivalence and CPDAGs

Several DAGs can encode the *same* set of conditional independencies — a **Markov equivalence
class**, represented by a CPDAG (some edges directed, some undirected). Observational data alone
generally identifies only the equivalence class, not every arrow direction. This is why MIRAGE's
**reversed-direction** and **construct-absence** prompts (llm-evaluation.md batches 3–4) are the
hard tests: distinguishing `X → Y` from `Y → X` is exactly what observational structure-learning
struggles to pin down, so a model that gets direction right is doing more than echoing correlations.

---

## 2. The four discovery algorithms

### 2.1 GES — Greedy Equivalence Search (score-based)

- **How it works:** searches over **Markov equivalence classes** (CPDAG space), not individual DAGs.
  Two phases: a **forward** phase greedily adds the edge that most improves a decomposable score
  (e.g. BIC), then a **backward** phase greedily removes edges until no removal improves the score.
- **Assumptions:** a decomposable, consistent score; large-sample consistency under the usual causal
  assumptions; faithfulness.
- **Complexity:** greedy and score-decomposable, so far cheaper than exhaustive DAG search, but
  worst-case still grows quickly with the number of variables — tractable here because the construct
  space is small (5 constructs + outcome).
- **Role in MIRAGE:** the score-based half of the **GES ∪ BN** ensemble.

### 2.2 PC (Peter–Clark) — constraint-based

- **How it works:** start from a fully connected undirected graph; **remove** edges whose endpoints
  are found conditionally independent (growing the conditioning set), yielding the skeleton; then
  orient **colliders** (v-structures) and propagate orientations with Meek's rules to a CPDAG.
- **Constraint- vs score-based:** PC tests **conditional independence** (constraints) rather than
  optimising a global **score**. Constraint-based methods are sensitive to CI-test errors
  (especially at small n or with the wrong test); score-based methods (GES) are sensitive to score
  misspecification. Pairing them (MIRAGE uses PC in the **PC ∪ DeepNOTEARS** ensemble) hedges the two
  failure modes.
- **CI testing:** MIRAGE fixes the significance level at **α = 0.05** (methodology.md §3). The choice
  of CI test must match the data type (e.g. Fisher-z for Gaussian, χ²/G² for discrete); α trades
  false edges (too high) against missed edges (too low).
- **Assumptions:** faithfulness, correct CI test, causal sufficiency (no hidden common causes) —
  the last is exactly what the "add unobserved common cause" refuter stress-tests (§3).

### 2.3 Bayesian Networks — structure vs parameter learning

A Bayesian network is a DAG **plus** conditional probability parameters. Two distinct learning
problems:

- **Structure learning:** find the graph (edges) — score-based search (e.g. hill-climbing on BIC/BDeu)
  or constraint-based, conceptually overlapping with GES/PC.
- **Parameter learning:** *given* the structure, estimate `P(X_i | parents(X_i))` — by maximum
  likelihood or Bayesian (e.g. Dirichlet-prior) estimation.

In MIRAGE the BN contributes the probabilistic-graph view in the **GES ∪ BN** ensemble; structure
learning gives the edges, parameter learning gives the strengths used to reason about the
constructs.

### 2.4 DeepNOTEARS — continuous-optimisation structure learning

- **The NOTEARS insight:** classic structure learning is a *combinatorial* search over DAGs (the
  acyclicity constraint is discrete). NOTEARS reformulates it as a **continuous optimisation**:
  represent the graph as a weighted adjacency matrix `W` and minimise a data-fit loss subject to a
  smooth **acyclicity constraint** `h(W) = tr(e^{W∘W}) − d = 0`, which equals zero **iff** `W` is
  acyclic. This turns discovery into gradient-based optimisation with an augmented-Lagrangian solver.
- **Why "Deep" handles non-linearity:** linear NOTEARS assumes linear structural equations. The deep
  variant replaces the linear map with **neural networks** (e.g. MLPs) for each variable's mechanism,
  so non-linear cause→effect relationships are captured while keeping the same differentiable
  acyclicity machinery. MIRAGE freezes its **L1 = 0.01** sparsity penalty (methodology.md §3) to keep
  the learned graph sparse/interpretable.
- **Failure mode observed:** the continuous acyclicity constraint is approximate and can fail to
  converge to a clean DAG on some data — precisely the **NOTEARS acyclicity failure on smishing**
  (results.md §1), which is why that channel was only a *partial* pass and why DeepNOTEARS is paired
  with PC rather than used alone.

### 2.5 Why an ensemble

No single discoverer is robust across assumptions: GES (score), PC (constraints), BN (probabilistic),
DeepNOTEARS (continuous/non-linear) fail differently. Merging into **GES ∪ BN** and **PC ∪
DeepNOTEARS** and keeping only edges that survive **DoWhy refutation** (§3) is a deliberate
triangulation — consensus edges (e.g. Deception as a convergent mediator, results.md §2) are far more
credible than any one method's output.

---

## 3. DoWhy refutation framework

DoWhy frames causal inference in four steps (methodology.md §4): **model** assumptions as a graph,
**identify** the estimand, **estimate** it (MIRAGE uses a binomial GLM), and **refute**. Refutation
does not *prove* an effect is causal — it tries to **break** the estimate; an effect that resists
breaking is more trustworthy. The refuters:

| Refuter | What it does | What *passing* means |
|---------|--------------|----------------------|
| **Placebo treatment** | Replace the real treatment/construct with a random ("placebo") variable and re-estimate | The estimated effect should collapse to ~0. A **low placebo p** (MIRAGE: empirical p ≈ 0.002, n = 500 permutations) means the real effect is not reproducible by noise — *pass* |
| **Random common cause** | Add an independent random variable as an extra "confounder" and re-estimate | A genuine effect should be **stable** (estimate barely moves). MIRAGE reports high Rand-CC p-values (0.68–1.00), i.e. no significant change — *pass* |
| **Data subset (bootstrap/subset)** | Re-estimate on random subsets of the data | A robust effect is **stable across subsets** (not driven by a few rows). High subset p (0.92–0.98) — *pass* |
| **Add unobserved common cause** | Simulate a *hidden* confounder of given strength and see how much the estimate shifts | Tests sensitivity to the **causal-sufficiency** assumption (the one PC/discovery cannot test from data). The effect should not vanish under plausible unobserved confounding |

MIRAGE applied placebo, random-common-cause, and subset refuters to every construct→outcome edge;
**only edges surviving all refuters enter the validated DAG used as LLM ground truth**
(methodology.md §4, results.md §2). The "add unobserved common cause" refuter is named in the build
prompt as part of the framework; **confirm whether it was run in this engagement** — results.md
reports placebo / random-common-cause / subset explicitly. *(Flagged for verification rather than
assumed.)*

> Interpretation caveat: refutation is a **robustness** check, not a proof of causation. Surviving
> all refuters means "no easy way to break this estimate was found under the stated assumptions,"
> which is the honest claim MIRAGE makes.

---

## 4. ICC(2,1) — intraclass correlation

### 4.1 Statistical meaning

ICC measures **rater agreement** for continuous scores by partitioning variance into between-target
(true signal), between-rater, and residual error. The Shrout & Fleiss (1979) notation `ICC(model,
form)`:

- **Model 2 ("2"):** **two-way random effects** — both the scored responses *and* the raters are
  random samples from larger populations (so the reliability estimate generalises to other raters).
- **Form 1 ("1"):** reliability of a **single** rater's score (not the mean of k raters), under
  **absolute agreement** (raters must give the *same* value, not merely rank-correlated values).

### 4.2 Why it was appropriate here

MIRAGE's scores are used **at face value** by individual raters across a two-wave × four-rater ×
five-dimension design (llm-evaluation.md). That calls for:

- **Type 2** because the raters are a sample, and we want the result to generalise beyond these four
  individuals;
- **single-rater form (",1")** because any one rater's score is taken as the measurement; and
- **absolute agreement** (not consistency) because the *magnitude* of the score matters, not just the
  ordering — a model's 4.6 must mean the same thing regardless of who scored it.

A consistency-only or mean-of-raters (k) variant would overstate reliability for how the scores are
actually used.

### 4.3 The estimator and interpreting 0.98

```
ICC(2,1) = (MS_R − MS_E) /
           ( MS_R + (k−1)·MS_E + (k/n)·(MS_C − MS_E) )
```

(`MS_R` between-targets, `MS_C` between-raters, `MS_E` residual; `k` raters, `n` targets.) Common
benchmarks (Koo & Li 2016; Cicchetti 1994): `<0.5` poor, `0.5–0.75` moderate, `0.75–0.9` good,
`>0.9` excellent. MIRAGE's **ICC(2,1) = 0.98 (95% CI ≈ 0.94–0.99)** is "almost perfect," meaning
essentially all score variance is genuine between-response signal rather than rater disagreement —
so the **model rankings reflect model behaviour, not rater noise**. The per-dimension spread (0.89
Directionality → 0.97 Structure) is itself informative: Directionality, exercised by the adversarial
reversed-direction prompts, is the hardest to score consistently, exactly as expected.

> Caveat: a high ICC certifies *rater agreement*, not *rubric validity*. Agreement that the rubric is
> being applied consistently is necessary but not sufficient for the rubric to measure "causal
> reasoning" correctly — a separate construct-validity question.

---

## 5. Cross-references and open items

- Discovery/validation pipeline detail: [methodology.md](methodology.md) §3–4.
- Reported edge statistics and verdicts: [results.md](results.md) §2.
- Scoring/ICC design: [llm-evaluation.md](llm-evaluation.md).
- [ ] Verify whether the "add unobserved common cause" refuter was executed (§3).
- [ ] Confirm the CI test used by PC for these (binary/continuous) features and α handling (§2.2).

> No statistics beyond those already reported in the engagement documents are introduced here.
> Citations (Shrout & Fleiss 1979; Koo & Li 2016; Cicchetti 1994; NOTEARS/Zheng et al. 2018) are
> standard references — confirm exact details before formal citation.
