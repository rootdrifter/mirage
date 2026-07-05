# Social Engineering Background — Reference

Background on the five behavioural constructs MIRAGE models, their psychological basis, their
mapping to MITRE ATT&CK, the state of phishing detection, and the open problem of LLM causal
reasoning. Supports [methodology.md](methodology.md), [datasets.md](datasets.md), and
[results.md](results.md).

---

## 1. The five behavioural constructs

MIRAGE maps technical indicators to five psychology-grounded constructs (datasets.md). Each is a
*mechanism of persuasion* an attacker exploits — which is why the research scores whether models
reason about the **mechanism**, not just spot the surface cue.

### 1.1 Urgency

- **Mechanism:** time pressure suppresses **System-2** (deliberate) processing and pushes the target
  into **System-1** (fast, heuristic) responses (Kahneman). Under urgency people skip verification
  steps they would normally take.
- **Cialdini's *scarcity* principle:** things perceived as limited or time-bound are valued more and
  acted on faster ("offer expires in 1 hour," "account will be suspended today"). Scarcity +
  potential-loss framing (loss aversion) is a potent combination.
- **In the data:** proxied by signals such as **website response time** (datasets.md). In the
  validated DAG, **Urgency → Deception → Phishing** (results.md §2) — urgency *enables* deception
  rather than acting alone, the kind of multi-hop chain weaker models miss.

### 1.2 Authority

- **Mechanism:** people defer to perceived authority. **Milgram's** obedience studies showed
  ordinary people comply with instructions from an authority figure to a striking degree; Cialdini
  codifies **authority** as a compliance principle. Phishing impersonates banks, IT/helpdesk,
  executives (BEC), or government bodies to borrow that deference.
- **In the data:** proxied by **email address embedded within a URL** (a spoofed-legitimacy cue) and,
  in vishing, authority-based call scripts. In the validated DAG, **Authority → Deception → Phishing**
  (results.md §2).

### 1.3 Trust

- **Mechanism:** attackers hijack **established trust signals** — brand logos, a padlock/TLS, a
  familiar sender, a domain that *looks* indexed/reputable — so the target extends trust earned by
  the impersonated entity. Trust cues are *exploitable*, not protective, which is exactly the
  distinction DeepSeek-67B got wrong (it "treated trust cues as protective rather than exploitable,"
  results.md §6).
- **In the data:** **URL/domain in a search index, TLS/SSL certified** (datasets.md). Note the
  subtlety the research surfaces: a present TLS cert raises *perceived* trust but says nothing about
  legitimacy (phishing sites routinely use valid certs) — a model with real causal understanding
  should treat "has TLS" as a trust-*signal* the attacker exploits, not as evidence of safety.

### 1.4 Deception

- **Mechanism:** the core misrepresentation — making the malicious appear benign. Emerged as the
  **convergent mediator** across all discovery methods (results.md §2): urgency, authority, and trust
  largely act *through* deception to produce the malicious outcome.
- **Linguistic markers (detection):** research on deceptive text notes shifts in pronoun use,
  reduced first-person reference, increased negative-emotion and "certainty" terms, and lexical
  mismatch between claimed sender and style. Mismatch between the display name and the underlying
  address/domain is a structural deception marker.
- **In the data:** proxied by **domain age / time of domain activation** (newly registered domains
  are a strong deception indicator).

### 1.5 Obfuscation

- **Mechanism:** technical concealment to evade both human inspection and automated filters.
- **Methods:** URL shorteners and multiple redirects (hiding the true destination), homoglyph/IDN
  look-alike domains, `@`-in-URL and userinfo tricks, excessive subdomains, percent-encoding and
  path-normalisation variants (`/%2e/`), open redirects, and link-wrapping abuse. In MIRAGE,
  obfuscation acted as a **technical amplifier rather than an initiator** (results.md §2) — it
  strengthens an attack built on the human constructs, it doesn't start one.
- **In the data:** **shortened URLs, redirect count, resolved-IP count** (datasets.md).

---

## 2. Mapping to MITRE ATT&CK

The constructs are *psychological levers*; ATT&CK catalogues the *techniques* that pull them. The
mapping (Enterprise ATT&CK — verify technique IDs against the current matrix before formal citation):

| Construct | Representative ATT&CK technique(s) | Why it fits |
|-----------|-----------------------------------|-------------|
| Authority | **T1566 Phishing** and sub-techniques **.001 Spearphishing Attachment / .002 Link / .003 via Service**; **T1656 Impersonation** | Impersonating trusted/authoritative senders is the delivery vehicle |
| Urgency | **T1566** (lure content); **T1598 Phishing for Information** | Time-pressure framing inside the lure to force quick action |
| Trust | **T1656 Impersonation**; **T1583/T1584** (acquire/compromise infrastructure for look-alike domains); abuse of valid certs | Borrowing legitimate trust signals (brands, domains, certs) |
| Deception | Spans the **Initial Access** phishing techniques; **T1204 User Execution** (the deception induces the click/open) | The misrepresentation that converts contact into action |
| Obfuscation | **T1027 Obfuscated Files or Information**; **T1656**; URL/redirect concealment within **T1566.002** | Technical concealment of payload/destination |

Caveats: ATT&CK is technique-centric and primarily post-delivery; the *human* persuasion mechanism
sits "above" the technique. MIRAGE's contribution is reasoning about that human layer causally,
which ATT&CK does not model. Treat the mapping as a bridge, not an equivalence.

---

## 3. Phishing detection — state of the art and the gap

### 3.1 What current systems do (and achieve)

- **URL/feature classifiers:** lexical + host-based + reputation features into gradient-boosted
  trees / classical ML. Mature, fast, high headline accuracy on benchmark corpora — but
  feature-distribution-bound and brittle to novel/obfuscated URLs.
- **Content/NLP models:** transformer text classifiers over email/SMS bodies; strong at known
  patterns.
- **Reputation / blocklists / DMARC-DKIM-SPF:** infrastructure-level controls; effective against
  known-bad and spoofed senders, weak against new domains and look-alikes.
- **Visual / brand-detection:** screenshot/logo similarity to catch credential-harvest pages.

Reported benchmark accuracies are high but **dataset-specific and drift-prone**; headline numbers do
not transfer to adversarial, paraphrased, or cross-channel inputs. (No specific third-party accuracy
figure is asserted here — *verify before citing any.*)

### 3.2 The gap MIRAGE addresses

These systems answer **"is this phishing?"** (classification). They do **not** explain **"why does
this message persuade, through what causal chain?"** A correlational classifier can flag a message
yet attribute it to surface tokens that an attacker can trivially paraphrase away. MIRAGE reframes
the task as **causal reasoning over a validated DAG of persuasion mechanisms** — testing whether a
model can reconstruct multi-hop chains (Urgency → Deception → Phishing) and resist inverted/absent
constructs. Robustness to surface perturbation is the point: core edges survived the email→voice
shift (results.md §6), consistent with Pearl's criterion that genuine causal relations persist under
surface change, whereas correlational features do not.

---

## 4. LLM causal reasoning — the open problem

### 4.1 Current state

Whether LLMs perform **genuine causal reasoning** or sophisticated **pattern-matching over
causal-sounding text** is an open and actively debated research question. LLMs do well on tasks
resembling **causal recall** (reciting known cause→effect relations seen in training) but are far
weaker on **causal discovery/inference from novel structure**, **counterfactuals**, and
**direction** — distinguishing `X → Y` from `Y → X` when both are statistically plausible.

### 4.2 Why it is hard

- Observational text under-determines direction (Markov-equivalence problem, see
  [causal-inference-background.md](causal-inference-background.md) §1.5).
- Training rewards fluent plausibility, not mechanism correctness — a model can produce a confident
  correlational story that reads causal.
- Counterfactual/inverse reasoning ("what if the construct were absent / the arrow reversed") has no
  shortcut from surface co-occurrence.

### 4.3 What MIRAGE's results say

The pattern across models is diagnostic: a **persistent awareness-vs-depth gap** (all models
*recognise* constructs better than they *explain the mechanism*, results.md §8), and a steep
ranking — **GPT-4 94.2%** DAG alignment down to **DeepSeek-67B 53.0%** — with the gap widening
exactly on the **inverse/adversarial** batches. GPT-4 approaches human-level causal fluency *in
controlled conditions*; weaker models revert to correlational explanation that "would fail under
adversarial paraphrase." This supports the central thesis: **construct recognition is necessary but
insufficient for reliable causal defence**, and current LLMs detect surface patterns but struggle
with multi-hop chains and inverse reasoning. The operational consequences are drawn out in
[security-implications.md](security-implications.md).

---

## 5. Open items / to verify

- [ ] ATT&CK technique IDs/names against the current Enterprise matrix (§2).
- [ ] Any external phishing-detection accuracy figure before citing (§3.1).
- [ ] Exact references for Cialdini, Milgram, Kahneman, and deception-linguistics work if formally
      cited (§1).

> No engagement statistics beyond those in the existing MIRAGE documents are introduced. Psychology
> references (Cialdini's principles, Milgram, Kahneman dual-process) are standard; confirm specifics
> before formal citation.
