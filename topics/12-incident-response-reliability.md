# 🚨 Incident Response & Reliability (SRE for ML/LLM Systems)

[← Back to main README](../README.md)

---

### Q: What is the difference between a traditional software outage and an "ML incident" (e.g., silent model quality degradation), and why does the latter often require fundamentally different detection and response processes?

**Answer:**
A traditional software outage typically has a clear, binary signal — the service is down, returning errors, or clearly non-functional, usually detectable immediately via standard uptime/error-rate monitoring. An **ML incident** (e.g., a model silently producing systematically worse-quality predictions due to drift, a pipeline bug, or a bad deployment) often produces **no errors or downtime at all** — the system continues responding successfully with a 200 status code and a syntactically valid prediction, but the *quality* of that prediction has silently degraded, which standard infrastructure monitoring has no visibility into whatsoever. This means ML incident detection requires the **model-quality-specific monitoring layers** discussed earlier (drift detection, prediction distribution monitoring, delayed ground-truth tracking) as a *necessary addition* to standard infrastructure monitoring — an AI Ops team that only monitors traditional uptime/error-rate metrics can have a severely degraded model running in "healthy" production for a long time with zero automated alerting.

---

### Q: Walk through how you'd triage a report that "the model's predictions seem off" with no other specific detail — what's your systematic investigation process?

**Answer:**
1. **Quantify "off"** — get specific examples of problematic predictions from the reporter if possible, and check whether aggregate monitoring dashboards (accuracy proxies, prediction distribution, drift metrics) show any correlated anomaly around the relevant timeframe.
2. **Check recent changes** — review the deployment/change log for the relevant model, pipeline, or upstream data sources around when the issue reportedly started; a strong correlation with a recent change is often the fastest path to root cause.
3. **Check for pipeline/infrastructure issues** — verify the correct model version is actually being served (a surprisingly common root cause is an unintended rollback/rollforward, or a caching issue serving a stale model), and check feature pipeline health/data quality for the relevant time window.
4. **Reproduce with a specific example** — take one or more of the reported problematic cases and trace them through the full pipeline (feature values, model version, exact output) to understand exactly what happened for that specific case, rather than relying purely on aggregate statistics.
5. **Classify the root cause category** (genuine drift requiring retraining, a pipeline/data bug requiring a code fix, a deployment/infrastructure issue requiring a rollback) to determine the appropriate remediation path, using the diagnostic framework discussed in the monitoring section for distinguishing these cases.

---

### Q: What is a "runbook" for an ML/LLM production system, and what specific scenarios should an AI Ops team have documented runbooks for, beyond generic infrastructure incident runbooks?

**Answer:**
A runbook documents the step-by-step process for diagnosing and responding to a specific, anticipated failure scenario, reducing reliance on any single person's memory/expertise during a stressful, time-pressured incident. Beyond generic infrastructure runbooks (service down, high error rate), ML/LLM-specific scenarios warranting dedicated runbooks include: **"model quality has degraded" investigation and rollback procedure** (the specific steps to check drift metrics, verify deployed model version, and execute a rollback if warranted), **"retraining pipeline failed" recovery procedure**, **"vector database/retrieval quality has degraded" investigation** (for RAG systems), **"LLM provider outage/degradation" fallback procedure** (if using a third-party LLM API, what's the documented fallback — a different provider, a cached/degraded response mode, or a clear user-facing error), and **"a safety/guardrail incident has occurred"** (as discussed in the security/governance section) — having these documented and periodically tested/reviewed (not just written once and forgotten) significantly speeds up and de-risks response during an actual live incident.

---

### Q: What is the concept of an "error budget" (from SRE practice), and how would you adapt this concept specifically for a model-quality SLO rather than a traditional availability SLO?

**Answer:**
An error budget represents the acceptable amount of unreliability (the gap between 100% and the target SLO) a team can "spend" over a given period before needing to prioritize reliability work over new feature development — e.g., a 99.9% availability SLO implies an error budget of about 43 minutes of downtime per month. Adapting this for a **model-quality SLO** (as discussed in the monitoring section — e.g., "accuracy won't degrade more than 3 percentage points from baseline"): the "budget" becomes the **allowable magnitude and duration of quality degradation** before it's treated as an SLO violation requiring an incident response and prioritized remediation (retraining, rollback) — and similar to traditional error budgets, a model that's "spending" its quality error budget rapidly (frequent or sustained periods of degraded performance) should trigger the team to **prioritize model reliability/retraining infrastructure investment** over shipping new model features/experiments, applying the same underlying SRE philosophy of using the error budget as an objective signal for balancing reliability investment against feature velocity.

---

### Q: What is a "blast radius" consideration specific to a shared model serving platform hosting many different models/teams, and how would you design the infrastructure to limit the blast radius of a single problematic model deployment?

**Answer:**
Blast radius refers to the scope of impact a single failure/incident can cause. On a **shared multi-model serving platform**, a poorly-designed architecture could allow a single problematic model (e.g., one causing memory leaks, excessive resource consumption, or crashes) to **degrade or take down the shared infrastructure serving entirely unrelated models**, if resources aren't properly isolated. Mitigations: **resource isolation** (ensuring each model's serving instances have properly enforced, isolated resource limits — CPU, memory, GPU — as discussed in the Kubernetes section, so one model's resource exhaustion can't starve others sharing the same underlying infrastructure), **independent scaling and deployment** per model (avoiding a shared, monolithic deployment unit where updating/scaling one model requires touching infrastructure shared with others), and **circuit breakers/bulkheads** at the platform level (if one model's serving endpoint becomes unhealthy, ensuring this doesn't cascade into broader platform-level failures affecting the routing/serving of other, healthy models) — designing explicitly for blast radius containment is a core responsibility when operating shared ML infrastructure at the platform level, rather than each model team needing to independently reason about this risk.

---

### Q: What is chaos engineering, and how might it be specifically applied to test the resilience of an ML/LLM production system beyond typical infrastructure chaos testing (e.g., randomly killing pods)?

**Answer:**
Chaos engineering deliberately introduces controlled failures into a system to proactively validate its resilience/recovery behavior, rather than discovering failure modes for the first time during a real incident. Beyond typical infrastructure chaos testing, ML/LLM-specific chaos scenarios worth testing include: **simulating a corrupted/malformed model artifact** being deployed (verifying the deployment pipeline's validation gates actually catch it before it reaches production, rather than assuming they would), **simulating an upstream feature pipeline outage or a specific feature silently returning null/default values** (verifying the serving system degrades gracefully per its intended design, rather than failing in some untested, unexpected way), **simulating an LLM provider's API becoming slow or unavailable** (verifying fallback/retry logic and circuit breakers actually function as intended under a real, sustained provider degradation rather than only a brief, easily-retried blip), and **simulating a vector database/retrieval system returning unexpectedly poor/irrelevant results** (verifying the overall RAG system's behavior/guardrails under genuinely degraded retrieval quality, not just complete retrieval failure). These ML-specific chaos scenarios test failure modes that generic infrastructure chaos testing (which typically focuses on compute/network failures) wouldn't naturally cover.

---

### Q: What is the difference between Mean Time to Detect (MTTD) and Mean Time to Resolve (MTTR) as reliability metrics, and why might MTTD be a particularly important metric to focus on improving specifically for ML quality incidents (as opposed to typical infrastructure incidents)?

**Answer:**
**MTTD** measures the average time between when a problem actually begins and when it's detected/flagged. **MTTR** measures the average time between detection and full resolution. For typical infrastructure incidents (a service crashing), MTTD is often quite short by default, since the failure is usually immediately and obviously visible (an error rate spike, a health check failing) — the bigger opportunity for improvement is often in MTTR (faster diagnosis and remediation). For **ML quality incidents**, given the "silent degradation" nature discussed earlier (no errors, just gradually worse predictions), **MTTD is often the much larger and more improvable component** of total incident duration — a model could be quietly underperforming for days or weeks before anyone notices via delayed ground-truth data or a business metric investigation, meaning investment in **better, faster proxy-signal monitoring** (drift detection, prediction distribution monitoring — precisely to shorten this detection gap without waiting for slow ground-truth feedback) often provides a bigger overall reliability improvement for ML systems than investing purely in faster remediation processes once a problem is already known.

---

### Q: What is an on-call rotation structure appropriate for an AI Ops team, and what specific escalation/expertise considerations arise for ML incidents that might not fit a standard, generalist on-call engineer's typical skill set?

**Answer:**
A standard on-call rotation assumes the on-call engineer can reasonably diagnose and resolve (or appropriately escalate) most incidents using generalist skills plus good runbooks. ML incidents introduce a wrinkle: a **generalist on-call engineer may correctly identify "the model's predictions look wrong" but lack the specific ML domain expertise to determine whether this reflects a genuine, urgent problem requiring immediate rollback, versus expected/acceptable statistical variation** — this ambiguity is different from a typical clear-cut infrastructure failure. Practical approaches: **well-designed runbooks with explicit, pre-agreed thresholds** for what specifically constitutes an actionable anomaly (removing ambiguous judgment calls from the on-call engineer wherever possible), a **secondary/escalation on-call rotation specifically staffed by ML engineers or data scientists** for genuinely ambiguous cases the primary on-call can't confidently resolve using the runbook alone, and ensuring the **primary on-call AI Ops engineer has clear, fast rollback authority/mechanism** even without deep ML expertise (since "roll back to the previous known-good model version" is often a safe, reasonable default action even without fully understanding the root cause first, buying time for deeper investigation without leaving a potentially degraded model serving live traffic in the meantime).

---

### Q: What is a post-incident review specifically for an ML system incident, and what ML-specific questions should it address beyond a standard "what happened, why, and how do we prevent recurrence" software post-mortem?

**Answer:**
Beyond the standard post-mortem structure, an ML-specific post-incident review should also address: **was this detected by automated monitoring, or only discovered through a manual report/business metric investigation** (if the latter, this directly points to a monitoring/observability gap needing investment, per the MTTD discussion above), **did existing evaluation/validation gates fail to catch this before deployment, or was this a genuinely new failure mode not covered by existing evaluation** (distinguishing "our safety net had a gap we can now close" from "this was a fundamentally new scenario, requiring new test coverage to be added going forward"), and **what is the appropriate long-term fix** — is this addressed by improving data validation, adding a new monitoring signal, adjusting the retraining cadence, or does it reveal a more fundamental limitation in the current modeling approach requiring a larger redesign. These ML-specific questions ensure the retrospective produces concrete improvements to the **ML-specific safety nets** (evaluation gates, monitoring signals, data validation) — not just the general infrastructure/process improvements a standard software post-mortem would naturally surface.

---

### Q: How would you design a disaster recovery plan specifically for the "model + supporting infrastructure" system as a whole (not just the underlying compute infrastructure), given that recovering compute doesn't automatically mean recovering full ML system functionality?

**Answer:**
A complete ML system DR plan needs to address recovery of several distinct components beyond generic compute/infrastructure recovery: the **model registry and specific model artifacts** (ensuring the exact production model version, not just "a" model, can be reliably restored — tying back to the model versioning/registry practices discussed earlier), the **feature store/pipeline** (both the online store's current data and the ability to reconstruct/backfill it if lost, and the offline store's historical training data), and for RAG/LLM systems specifically, the **vector database index** (as discussed in that section, ideally recoverable from snapshots rather than requiring a full, slow re-embedding from source documents as the only recovery path). A DR plan that only addresses "can we bring compute infrastructure back online" without explicitly addressing these ML-specific data/artifact recovery components would likely restore a **technically running but functionally broken or severely degraded system** (e.g., serving infrastructure that's up but has no model to load, or a RAG system with an empty/stale knowledge base) — the DR testing/validation process should specifically verify full functional recovery, not just infrastructure/compute availability.

---
