# AI/ML Delivery Playbook

**A practitioner's reference methodology for taking AI/ML systems from concept to
governed production — with a bias toward regulated, human-in-the-loop environments.**

> This is my working POV, distilled from systems I have built: a closed-loop agent
> operations platform, a metadata-governed AI guardrail layer validated against a live
> catalog, a fully on-prem LLM agent, and regulated statistical modeling for public
> health. It is opinionated on purpose — a playbook that hedges every choice is not a
> playbook.

---

## 0. First principles

1. **The model is the small part.** The system around it — data, evaluation,
   governance, observability, human oversight — is where production value and
   production risk actually live.
2. **A confident wrong answer is the enemy.** Every design choice is measured against
   whether it makes the system more honest about what it does and does not know.
3. **Governance is a property you build in, not a bolt-on.** If traceability,
   evaluation, and human oversight aren't in the reference architecture, they won't
   survive delivery pressure.
4. **Optimize for reversibility.** Standardize the expensive-to-change layers
   (governance, evaluation, data contracts) early; stay deliberately swappable on the
   fast-commoditizing layers (the model itself).
5. **Depth over duration.** Judge an approach by shipped systems and demonstrated
   failure modes survived, not by how long a tool has existed.

---

## 1. Reference architecture: invariants vs substrate

Separate what never changes from what flexes per engagement.

**Invariant layers (fixed across projects):**
- **Feature / context layer** with online-offline consistency — train-serve skew is
  the most common silent production failure.
- **Model (or artifact) registry** — the system of record for what is deployed, its
  version, and its validation status. Nothing serves that isn't registered.
- **Evaluation harness** — every model/prompt/retriever passes it before promotion.
- **Observability + evidence layer** — inputs, outputs, drift signals, and a
  reconstructable record of every consequential decision.

**Substrate (flexes on data gravity + TCO + governance surface):**
- Data-resident / warehouse-native when the data gravity and governance already live
  there (minimizes egress and copy-governance).
- Cloud-hosted / containerized when custom training, multi-model serving, or
  low-latency patterns don't fit the warehouse.
- On-prem / local model serving when data residency or restriction demands it.

> Principle: a reference architecture that dictates *tools* ages badly; one that
> dictates *guarantees* (consistency, traceability, evaluation-before-promotion)
> travels across engagements.
>
> Proven in: a platform designed for graceful degradation, where every external
> integration is optional and governance rules live in reviewable config, not code —
> so a new engagement swaps the substrate without touching the core.

---

## 2. The delivery lifecycle (with a gate at every stage)

| Stage | Key decision | The gate before advancing |
|-------|-------------|---------------------------|
| **Frame** | Prediction problem or decision problem? Who acts on the output? | A written estimand/target tied to a real decision — not "improve X." |
| **Ingest** | Structured, unstructured, documents? Batch or incremental? | Data-quality gates that stop bad data BEFORE it reaches modeling. |
| **Model** | Classical, deep, or LLM/RAG? (see §4) | Reproducible pipeline, versioned, not notebook-only. |
| **Evaluate** | Offline metrics + representative test set + subgroups | Meets pre-registered thresholds incl. disaggregated subgroup perf. |
| **Deploy** | Shadow → canary → full; API + container | Shadow results match expectations; rollback path exists. |
| **Operate** | Monitors registered at deploy; drift + cost + latency | Every alert has a defined response (retrain/recalibrate/rollback). |
| **Govern** | Audit trail, evidence log, human oversight | A reviewer can reconstruct WHY any output happened. |

---

## 3. The evaluation ladder (promotion, not a single score)

"Is the model good?" is the wrong question. "Is it good enough to advance to the next
level of exposure?" is the right one.

1. **Offline** — versioned, representative test set; **disaggregated subgroup metrics**
   (subgroup failure blocks promotion even if the aggregate passes).
2. **Shadow** — score live traffic without acting; compare to incumbent and to realized
   outcomes. Catches train-serve skew and distribution shift offline eval can't.
3. **Canary / A-B** — pre-registered success metric AND a guardrail metric (measure the
   lift, watch for the thing that could break).

For GenAI/RAG the ladder is identical; the metrics change:
- **Retrieval quality** (recall@k / precision@k vs labeled query→doc pairs)
- **Grounding** (fraction of answer claims supported by retrieved context = the
  hallucination measure)
- **Answer quality** (human or LLM-as-judge against a rubric)
- **Abstention rate** (the system MUST be able to say "not enough to answer")

---

## 4. Decision frameworks (the reusable "when to reach for what")

**Classical vs LLM.** Structured prediction, ranking, anomaly detection → classical
(more accurate, cheaper, SHAP-explainable, deterministic/auditable). Unstructured
retrieval, natural-language interface, document grounding → LLM/RAG. Mature systems are
**hybrid**; the skill is knowing which layer owns which job. You cannot put a stochastic
text generator in front of an auditor asking "why was this scored high."

**Build vs buy the model.** The model layer is commoditizing fastest — stay
model-agnostic and invest in the platform underneath. Own the governance and
evaluation; rent the frontier model.

**Substrate choice.** Follow data gravity and total cost of ownership, not tool
preference. Document the decision as a reusable pattern so the next engagement doesn't
relitigate it.

**RAG design.** Chunk with overlap + per-chunk metadata for filtering → embed → vector
store → semantic retrieval → **re-rank** (raw similarity surfaces plausible-but-wrong) →
generate strictly from context with per-claim citation → abstain when retrieval is empty.

---

## 5. Responsible AI & model risk (map, don't reinvent)

Three frameworks answer different questions; use them together.

- **SR 11-7** (model risk management): effective challenge via validation *independent
  of development*, conceptual soundness, ongoing monitoring, outcomes analysis, model
  inventory + documentation.
- **NIST AI RMF**: the operating structure — **Govern, Map, Measure, Manage**.
- **EU AI Act**: risk-tiered obligations; classify each system, then meet the high-risk
  duties (risk management, data governance, technical docs, logging/traceability, human
  oversight, accuracy/robustness).

> Architect move: build these into the reference architecture — inventory, validation
> gate, evidence logs, human-oversight mechanism present by default — so each engagement
> *inherits* compliance instead of reinventing it.
>
> Proven in: independent-QC statistical validation (a second analyst reproduces key
> results by different code = SR 11-7 "effective challenge") and a governance layer that
> writes a self-contained evidence record for every automated decision, including how
> stale the underlying data was at decision time.

---

## 6. Human-in-the-loop patterns

The AI supports an accountable human decision; it is never the decision.

- **Review** — surface outputs WITH their evidence; the reviewer sees what the model saw.
- **Escalate** — route by confidence × risk; senior human attention goes to hard/high-
  impact cases, not uniformly.
- **Override** — the human can always overturn, and the override is captured as labeled
  feedback.
- **Feedback** — overrides and corrections flow back into the eval set and retraining.

> Proven in: an agent platform where state-changing actions pause for explicit human
> approval while read-only actions run freely — the permission boundary *is* the
> human-in-the-loop control.

---

## 7. Observability & operations

- **Data drift** — input distribution vs training (PSI-style); the earliest warning,
  because inputs shift before performance visibly degrades.
- **Concept drift** — rolling performance on realized outcomes; the X→y relationship can
  change under stable inputs.
- **Operational** — cost, latency, error; for LLM systems add retrieval quality and
  abstention rate over time.
- **Rule** — an alert with no defined response is noise. The monitor and the runbook
  ship together, registered at deploy and tied to the registry entry.

---

## 8. Anti-patterns (the failure modes I design against)

- **Notebook-to-prod** — a model with no tests, container, API, or CI/CD is a prototype,
  not a system.
- **Aggregate-metric blindness** — a strong headline number hiding subgroup failure.
- **Explanation-as-validation** — SHAP tells you *what* the model uses, not whether it
  *should*; use it to find leakage and proxies, not to certify fairness.
- **Leakage** — a feature available at training but not at prediction time; audit top
  features that look "too good."
- **Silent partial failure** — a pipeline that half-runs and ships wrong numbers is worse
  than one that crashes loudly.
- **Governance as afterthought** — traceability retrofitted after a regulator asks is the
  most expensive way to build it.

---

## 9. Pre-flight checklists

**Before modeling:** estimand written · decision + consumer identified · data-quality
gates defined · representative test set (incl. subgroups + edge cases) versioned.

**Before deploy:** registry entry + validation status · evaluation ladder passed ·
shadow results reviewed · rollback path · monitors + runbooks registered.

**Before handoff:** confidence/limitations/failure-modes documented in operational terms
· evidence log reconstructable · human-oversight mechanism live · policy in reviewable
config.

---

*Author's note: this playbook reflects how I build and reason about AI systems. The
patterns are grounded in systems I have shipped; the frameworks (SR 11-7, NIST AI RMF,
EU AI Act) I apply from regulated modeling practice. It is a living document — every
engagement sharpens it.*
