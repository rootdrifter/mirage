# Security Implications — Reference

Translates MIRAGE's findings ([results.md](results.md)) into operational meaning for SOC analysts,
threat intelligence, and model-deployment decisions, and sets out what productionising the research
would require. Reference document — it reasons from the engagement's reported results, introducing
no new statistics.

---

## 1. Implications for SOC analysts (tier-1 triage)

### 1.1 The current tier-1 problem

Tier-1 phishing triage is high-volume and binary-leaning: an alert fires (user report, gateway
flag), the analyst decides phish / not-phish and escalates. Two chronic pains: **alert fatigue**
(volume) and **shallow rationale** (a verdict without a defensible *why*, which slows escalation and
weakens user feedback).

### 1.2 How causal reasoning could change it

A model that reasons over a validated causal structure (not just classifies) could give tier-1:

- **Verdict + mechanism, not just verdict.** "Malicious — Authority (spoofed helpdesk) → Deception,
  amplified by Obfuscation (double redirect)" is triage-ready: it tells the analyst *what lever* the
  attacker pulled, which speeds escalation and feeds targeted user awareness messaging.
- **Construct-tagged enrichment** on every alert (which of the five constructs are present, and the
  inferred chain), turning a binary queue into a prioritisable, explainable one.
- **Resistance to paraphrase.** Because causal edges survive surface perturbation (results.md §6),
  a causal layer is less likely than a token classifier to be evaded by reworded lures — reducing
  the false-negative tail that hurts most.

### 1.3 Workflow integration (realistic)

- **Human-in-the-loop assist, not autonomous verdicts.** Use the model to *enrich and explain*,
  analyst confirms — consistent with the finding that even the best model is excellent but not
  infallible at depth.
- **Construct tags → playbooks.** Map dominant construct(s) to response playbooks (e.g.
  Authority/BEC → finance verification path; Obfuscation-heavy link → detonation/sandbox).
- **Feed disagreements back** (model rationale vs analyst verdict) as labelled data for tuning.

---

## 2. Implications for threat intelligence

- **Causal models of attacker tradecraft.** Modelling which *psychological* levers a campaign pulls
  — and through what chain — adds a layer above IOC/TTP reporting. Two campaigns with different
  infrastructure may share an Authority → Deception spine; clustering on **causal signature** can
  link activity that IOC-matching misses.
- **Campaign profiling and attribution support.** Persistent construct-usage patterns (e.g. a group
  that reliably leads with Urgency + Obfuscation) become a behavioural fingerprint that survives
  infrastructure churn — complementing, not replacing, technical attribution.
- **Predictive framing for defenders.** Knowing the *mechanism* a sector is being hit with informs
  proactive user training and control placement (e.g. enforce out-of-band verification where
  Authority/BEC dominates).
- **Honest limit:** this enriches human-authored TI products; it does not replace analyst judgement,
  and causal-signature clustering needs validation before it drives attribution claims.

---

## 3. The 94.2% vs 53.0% gap — operational meaning

GPT-4 reproduced **94.2%** of validated DAG edges; DeepSeek-67B only **53.0%** (results.md §3). If
these were deployed in a security pipeline:

- **53% alignment ≈ near-coin-flip on causal structure.** A model that reproduces only half the
  validated edges, **under-weights obfuscation, and inverts trust** (treating exploitable trust cues
  as protective, results.md §6) would systematically **misexplain** alerts — and worse, mislead a
  trusting analyst with confident-but-wrong rationale. In triage, *confidently wrong* is more
  dangerous than *uncertain*.
- **94% is strong but not a blank cheque.** Even GPT-4 shows the awareness-vs-depth gap and loses
  depth under cross-channel drift (results.md §6, §8). It is suitable as an **assistive** layer with
  human confirmation, not for autonomous blocking decisions.
- **Model choice is a security control.** The spread shows that *which* model you deploy materially
  changes detection quality and failure modes. A cheaper/open model is not a drop-in substitute here;
  the deployment decision must weigh the alignment gap, not just cost or latency.
- **Self-hosting trade-off.** DeepSeek-67B is self-hostable (the engagement ran it on RunPod H100s) —
  attractive for data-sovereignty/privacy in a SOC — but its weaker causal fidelity is the price.
  The right answer may be a **hybrid**: self-hosted model for bulk/low-risk enrichment, strongest
  model reserved for ambiguous/high-impact cases. Data-handling rules (don't send sensitive content
  to third-party APIs) may force self-hosting regardless, which makes closing the open-model gap a
  priority (§4).

---

## 4. Future research directions — toward productionisation

What would have to happen to turn MIRAGE into a real detection capability:

1. **Scale and diversify the ground truth.** The validated DAG rests on the modelled feature set and
   corpora (datasets.md); a production system needs DAGs validated across more sources, languages,
   and current campaigns, refreshed as tradecraft evolves.
2. **Live, adversarial evaluation.** Move from 36 controlled prompts to continuous testing against
   real, **adversarially paraphrased** lures — the condition weaker models fail. Measure drift over
   time, not a one-shot score.
3. **Automate the scoring/grading.** Human ICC(2,1) = 0.98 scoring (llm-evaluation.md) does not scale;
   a validated automated grader (itself audited for bias) is needed for continuous monitoring.
4. **Close the open-model gap.** Fine-tune / distil causal-reasoning capability into self-hostable
   models so privacy-constrained SOCs aren't forced onto the weak end of the 53%–94% range.
5. **Integrate, don't isolate.** Wire the causal layer behind existing controls (gateway, SIEM,
   SOAR) as enrichment; define escalation thresholds; keep humans in the loop.
6. **Counterfactual robustness as a release gate.** Treat performance on the **inverse/adversarial**
   batches (construct-absence, reversed-direction) — not headline accuracy — as the bar a model must
   clear before deployment, since that is where correlational models fail.
7. **Adversarial-input hardening.** Anticipate that attackers will craft lures to fool the causal
   layer itself; red-team the detector.

---

## 5. Bottom line

MIRAGE's security value is a reframing: **explainable, mechanism-level reasoning** about *why* a
message persuades, robust to surface change — directly useful for triage rationale and TI campaign
profiling. The results also deliver a deployment warning: causal fidelity varies enormously by model
(53%→94%), the best models are assistive-not-autonomous, and **construct recognition is necessary but
insufficient** for reliable defence. Productionisation hinges on adversarial robustness and closing
the open-model gap, not on chasing a higher headline accuracy number.

> No statistics beyond those reported in the existing MIRAGE documents are introduced here. Forward
> claims (campaign clustering, hybrid deployment) are reasoned implications, flagged as requiring
> validation before operational reliance.
