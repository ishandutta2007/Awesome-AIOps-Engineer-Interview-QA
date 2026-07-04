# 🚀 Model Deployment & Serving Infrastructure

[← Back to main README](../README.md)

---

### Q: What are the main deployment patterns for rolling out a new model version (blue-green, canary, shadow), and when would you choose each?

**Answer:**
**Blue-green deployment** runs the new model version ("green") fully in parallel with the current version ("blue"), then switches all traffic over at once — enabling instant rollback (switch back to blue) but requiring double the serving capacity during the transition and offering no gradual validation with real traffic before the full cutover. **Canary deployment** gradually routes a small percentage of live traffic to the new version, monitoring closely, and progressively increasing that percentage if metrics look healthy — this limits the blast radius of a problematic new model to a small fraction of users while still validating against real production traffic. **Shadow deployment** (covered in the fundamentals topic) sends traffic to the new model without acting on its output at all, for pure observation with zero user-facing risk. In practice, teams often combine these — shadow first to catch gross issues safely, then canary to validate real-world impact incrementally, then a full blue-green-style cutover once confidence is high.

---

### Q: What is the difference between model serving via a REST API and via a batch prediction job, and what determines which architecture a given use case needs?

**Answer:**
A **REST API serving architecture** exposes the model behind an HTTP endpoint that responds to individual prediction requests in real time, appropriate when predictions are needed **on-demand, at the moment of a user/system interaction** (e.g., a real-time recommendation, a fraud check during checkout). A **batch prediction job** processes a large volume of inputs on a schedule, writing results to storage for later consumption, appropriate when predictions can be **precomputed ahead of time** for a bounded, known set of entities (e.g., nightly churn scores for all current users) and low-latency, on-demand response isn't required. The determining factor is usually: does the use case need a fresh prediction for an input that couldn't have been known in advance (favoring real-time serving), or can the space of needed predictions be enumerated and computed proactively on a schedule (favoring cheaper, simpler batch processing)?

---

### Q: What is model quantization, and how does it help with production inference performance/cost, and what's the tradeoff?

**Answer:**
Quantization reduces the numerical precision used to represent a model's weights/activations (e.g., converting 32-bit floating point to 8-bit integers), significantly **reducing model size and memory footprint** and often substantially **speeding up inference** (particularly on hardware with optimized low-precision compute paths), which directly reduces serving cost (smaller/cheaper instances, higher throughput per instance, potentially fitting larger models onto available hardware). The tradeoff is a potential **small loss in model accuracy/quality**, since reduced numerical precision inherently loses some information — the magnitude of this quality loss depends heavily on the specific model architecture and quantization technique (post-training quantization vs. quantization-aware training, which retrains the model to be more robust to the reduced precision) — production teams typically validate quantized model quality carefully against acceptance thresholds before deploying, since the acceptable tradeoff point is very use-case dependent.

---

### Q: What is model distillation, and why might an AI Ops team push for a distilled model in production even if the larger "teacher" model has better raw accuracy?

**Answer:**
Model distillation trains a smaller "student" model to mimic the behavior/outputs of a larger, more capable "teacher" model, aiming to capture much of the teacher's performance in a substantially smaller, cheaper-to-run package. An AI Ops team might push for the distilled version in production because the **operational costs and latency of serving the larger teacher model at scale** can be prohibitive — a model with slightly lower raw accuracy but a fraction of the inference cost and latency can represent a much better overall tradeoff for a high-volume production system, especially when the accuracy gap doesn't meaningfully change the business outcome for the specific task, while the cost/latency difference very much does affect infrastructure spend and user experience at scale.

---

### Q: What is dynamic batching in the context of model serving, and how does it improve GPU utilization for inference workloads?

**Answer:**
Dynamic batching accumulates multiple incoming inference requests that arrive within a short time window into a **single batch**, processed together through the model in one forward pass, rather than processing each request individually one at a time. This significantly improves GPU utilization because GPUs achieve much higher computational efficiency processing larger batches (better parallelism, amortizing fixed per-call overhead across more work) than processing many tiny individual requests sequentially — serving frameworks like NVIDIA Triton or vLLM implement sophisticated dynamic/continuous batching specifically to maximize throughput under real-world traffic patterns where requests arrive somewhat unpredictably, balancing the throughput gains of larger batches against the added **latency** of waiting briefly to accumulate a batch (a tunable tradeoff, since waiting too long to build a bigger batch could violate latency SLAs for individual requests).

---

### Q: What is a model serving framework (e.g., TensorFlow Serving, TorchServe, NVIDIA Triton, vLLM), and what capabilities do these frameworks provide beyond just "loading a model and running predict()"?

**Answer:**
These frameworks provide production-grade serving infrastructure beyond a bare model.predict() call: **standardized APIs** (REST/gRPC) so client applications don't need custom integration code per model framework, **multi-model serving** (hosting many models/versions efficiently on shared hardware), **dynamic batching** (as above) for throughput optimization, **hardware acceleration integration** (optimized execution on GPUs/specialized inference chips), **built-in monitoring/metrics** (request latency, throughput, error rates exposed for observability tooling), and often **automatic model version management** (routing traffic to specific versions, supporting canary/shadow patterns natively). Using a mature serving framework rather than a custom-built serving wrapper saves significant engineering effort re-implementing these common, well-understood production serving concerns.

---

### Q: What is the difference between horizontal and vertical scaling for a model serving deployment, and which is generally preferred for handling variable inference traffic?

**Answer:**
**Vertical scaling** increases the resources (CPU/GPU/memory) of a single serving instance. **Horizontal scaling** increases the **number of serving instances/replicas** running behind a load balancer. For variable inference traffic, **horizontal scaling is generally preferred** because it allows **fine-grained, incremental capacity adjustment** matching traffic fluctuations (adding/removing replicas as load changes, often via autoscaling based on request queue depth or GPU utilization), provides **fault tolerance** (one instance failing doesn't take down the whole service, since other replicas continue serving), and avoids hitting a hard ceiling on the largest available single-machine hardware configuration. Vertical scaling still matters for choosing the right base instance size/GPU type per replica (especially for large models that simply don't fit on smaller hardware), but the primary mechanism for handling variable production traffic load is typically horizontal autoscaling.

---

### Q: What is a model rollback strategy, and what specific capabilities does your deployment infrastructure need to support a fast, safe rollback if a newly deployed model causes problems in production?

**Answer:**
A rollback strategy is the plan/mechanism for quickly reverting to a previously known-good model version if a newly deployed version causes degraded performance, errors, or unacceptable business impact. Required infrastructure capabilities: the **previous model version must remain readily available** (not deleted/archived inaccessibly the moment a new version deploys), the **deployment mechanism must support near-instant traffic routing changes** (e.g., switching a load balancer's target group, or updating a version pointer, rather than requiring a lengthy redeployment/rebuild process to revert), and **automated or well-defined manual triggers** for when a rollback should occur (e.g., error rate or latency crossing a threshold, or a business metric degrading beyond an acceptable bound during a canary phase) — a rollback capability that exists in theory but takes 30 minutes of manual, error-prone process to actually execute during a live incident provides much less real protection than an automated, pre-tested rollback path that can execute in seconds.

---

### Q: What is warm-up/cold-start latency for a model serving instance, and why does it matter particularly for autoscaling and serverless inference deployments?

**Answer:**
Cold-start latency is the delay before a newly-launched serving instance is ready to handle requests at full performance — often including time to load the model weights into memory (which can be substantial for very large models), initialize hardware acceleration contexts (e.g., CUDA initialization), and potentially "warm up" internal caches/JIT-compiled execution paths that only reach peak efficiency after processing some initial requests. This matters significantly for **autoscaling** (a newly spun-up replica responding to a traffic spike isn't immediately useful if it takes 60+ seconds to become ready, potentially missing the window when it was actually needed) and **serverless inference** (platforms that scale down to zero between requests can suffer particularly severe cold-start penalties on the next request, directly hurting latency-sensitive use cases) — mitigation strategies include maintaining a minimum "warm" replica pool even at low/no traffic, pre-loading/caching model weights on fast storage, and choosing serving architectures explicitly optimized for fast startup for latency-critical services.

---

### Q: What is a feature/inference skew test, and how would you build an automated check into a deployment pipeline to catch training-serving skew before it reaches production?

**Answer:**
A feature/inference skew test compares the **actual feature values computed by the production serving pipeline** against the **feature values computed by the training pipeline for the same underlying raw inputs**, flagging discrepancies before they cause silent model quality degradation in production. A practical automated check: as part of the deployment pipeline, run a **held-out set of known raw inputs through both the training feature computation code and the production serving feature computation code**, then assert the resulting feature vectors match within an acceptable numerical tolerance — failing the deployment/promotion if a meaningful mismatch is detected. This is a good example of the kind of automated safety net an AI Ops engineer would build specifically to catch training-serving skew (a notoriously silent failure mode) systematically and early, rather than relying on it eventually surfacing as a mysterious model quality drop weeks after a change ships.

---

### Q: What is the difference between latency and throughput as serving performance metrics, and describe a realistic scenario where optimizing purely for throughput could hurt the actual user experience.

**Answer:**
**Latency** measures the time for a single request to be processed and return a response (often reported as p50/p95/p99 percentiles, not just an average, since tail latency matters a lot for user experience). **Throughput** measures the total volume of requests a system can process per unit time. Optimizing purely for throughput can hurt user experience via **dynamic batching with an overly aggressive batch-accumulation window** — waiting longer to build bigger batches maximizes total throughput/GPU efficiency, but individual requests that happen to arrive right after a batch was just dispatched now wait longer for the *next* batch window to close, increasing tail latency for those unlucky requests even while aggregate throughput numbers look great — this is a classic real tradeoff an AI Ops engineer has to tune deliberately, since a system can have excellent throughput on paper while still delivering an unacceptably slow experience for a meaningful fraction of actual users, depending on p95/p99 latency rather than just the average.

---

### Q: What is A/B testing infrastructure for models, and what specific technical capabilities does it require beyond what's needed for a simple canary rollout?

**Answer:**
While a canary rollout is primarily concerned with **safely, gradually shifting traffic** to validate a new version isn't broken, true **A/B testing infrastructure** additionally needs: **consistent user/request bucketing** (the same user should reliably see the same model version across their session/repeated visits, not be randomly re-assigned on every request, which would confound the comparison), **statistically rigorous experiment tracking** (logging which model version served which prediction, tied to eventual outcome/label data, so a proper statistical comparison of business/model metrics between arms can be computed later), and **support for running multiple concurrent experiments** without interference (e.g., ensuring a model A/B test and an unrelated pricing A/B test don't accidentally confound each other's results if they overlap on the same user population). This is meaningfully more infrastructure than canary deployment alone requires, since canary rollout's goal is operational safety during a rollout, while A/B testing's goal is a statistically valid comparison of business impact between two genuinely coexisting variants over a sustained period.

---
