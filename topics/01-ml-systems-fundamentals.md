# 🧩 ML/LLM Systems Fundamentals & Lifecycle

[← Back to main README](../README.md)

---

### Q: What is MLOps, and how does it differ from traditional DevOps?

**Answer:**
MLOps applies DevOps principles (automation, CI/CD, monitoring, versioning) to the **machine learning lifecycle**, but addresses concerns unique to ML that traditional DevOps doesn't handle: **data versioning** (not just code versioning, since model behavior depends heavily on training data), **model versioning and lineage** (tracking which code, data, and hyperparameters produced a given model artifact), **continuous training/retraining** (models degrade over time as real-world data drifts, requiring an ongoing retraining loop that traditional software doesn't need), and **model-specific testing/validation** (accuracy/fairness/performance metrics on held-out data, not just pass/fail unit tests). The core insight is that in ML systems, **both code and data are first-class, versioned inputs** to the deployed artifact, which fundamentally changes what "continuous integration" and "continuous deployment" need to mean.

---

### Q: Describe the end-to-end ML system lifecycle, from data to a deployed, monitored model.

**Answer:**
Typical stages: (1) **Data collection & validation** — ingesting and validating raw data meets quality/schema expectations. (2) **Feature engineering** — transforming raw data into model-ready features, ideally via a reusable feature pipeline. (3) **Model training & experimentation** — training candidate models, tracked via an experiment tracking system. (4) **Model evaluation & validation** — testing against held-out data, checking for acceptable performance and fairness before promotion. (5) **Model registry & versioning** — storing the approved model artifact with metadata/lineage. (6) **Deployment/serving** — exposing the model via a serving infrastructure (batch or real-time). (7) **Monitoring** — tracking prediction quality, data drift, and system health in production. (8) **Retraining/feedback loop** — using monitoring signals and new labeled data to trigger retraining, closing the loop back to step 1. An AI Ops engineer typically owns the infrastructure and automation connecting these stages, not necessarily the model development itself.

---

### Q: What is the difference between an ML Engineer, an MLOps/AI Ops Engineer, and a Data Scientist, in terms of typical responsibilities?

**Answer:**
A **Data Scientist** typically focuses on exploratory analysis, model development, and feature/algorithm selection — the "what model and why." An **ML Engineer** focuses on turning a data scientist's model into robust, production-ready code — often owning the boundary between experimentation and production. An **MLOps/AI Ops Engineer** focuses on the **infrastructure, automation, and operational reliability** enabling the entire lifecycle at scale — CI/CD pipelines, serving infrastructure, monitoring systems, cost/resource management — often supporting many models/teams rather than being deeply embedded in any single model's development. In practice these roles overlap significantly and blur at smaller organizations, but larger organizations increasingly separate them, with the AI Ops role resembling a platform/SRE function specifically for ML/AI workloads.

---

### Q: What is model versioning, and why is it insufficient to version only the model's code without also versioning training data and hyperparameters?

**Answer:**
Model versioning tracks distinct iterations of a trained model artifact so any specific version can be identified, reproduced, or rolled back to. Versioning only the code is insufficient because a model's actual behavior is a function of **code + training data + hyperparameters + the specific training run's environment** (library versions, random seeds) — the same code trained on different data (or even the same data at a different point in time, if the dataset is mutable) produces a genuinely different model artifact. Full reproducibility and proper debugging require tracking all of these together (often via an experiment tracking tool like MLflow or Weights & Biases) — without this, a team investigating "why did the model's behavior change" has no reliable way to determine whether it was a code change, a data change, or a hyperparameter change that caused it.

---

### Q: What is a model registry, and what problem does it solve in an ML deployment pipeline?

**Answer:**
A model registry is a centralized system for storing, versioning, and managing the lifecycle stage (e.g., staging, production, archived) of trained model artifacts, along with their associated metadata (training metrics, lineage, approval status). It solves the problem of **ad-hoc, untracked model deployment** — without a registry, teams often end up with model files scattered across different storage locations, unclear about which specific version is currently live in production, no formal approval/promotion gate between "a data scientist trained this" and "this is now serving production traffic," and no clean mechanism to **roll back** to a previous known-good version if a newly deployed model causes issues. A registry provides a single source of truth for "what model version is running where," which is foundational for both reliable deployment automation and incident response.

---

### Q: What is the difference between online (real-time) inference and batch/offline inference, and what infrastructure implications does each have?

**Answer:**
**Online inference** serves predictions for individual requests in real time as they arrive (e.g., a fraud check at transaction time), requiring a **low-latency serving infrastructure** (model loaded in memory, fast feature retrieval, often behind a load balancer/autoscaling group) available continuously. **Batch inference** scores a large set of examples on a schedule (e.g., nightly churn scores for all users), which can use less latency-sensitive, more cost-efficient infrastructure (e.g., spot instances, larger batch jobs run during off-peak hours) since there's no strict per-request response time requirement. An AI Ops engineer needs to design fundamentally different infrastructure for each — online serving demands availability/latency SLAs and autoscaling, while batch inference infrastructure is optimized more for throughput and cost-efficiency over a scheduled window.

---

### Q: What is "training-serving skew," and why is it one of the most common and hardest-to-debug production ML issues?

**Answer:**
Training-serving skew occurs when the features/data available and computed **at serving time differ subtly from how they were computed during training** — e.g., a feature computed via a batch SQL query during training but recomputed via different application logic in the real-time serving path, producing slightly different values for what's conceptually "the same" feature. This is hard to debug because the model often still **runs successfully and produces predictions** without any error — it just silently underperforms in ways that aren't immediately obvious, since there's no crash or exception to point to the root cause. This is precisely the problem a **feature store** is designed to solve, by ensuring the exact same feature computation logic and (ideally) the same underlying pipeline is used for both training and serving.

---

### Q: What is model decay/degradation, and why does even a well-trained, well-validated model's performance typically decline over time in production without any change to the model itself?

**Answer:**
Model decay occurs because the **real-world data distribution the model encounters after deployment gradually shifts away from the distribution it was trained on** — user behavior changes, new products/categories emerge, external conditions change (seasonality, economic shifts, a competitor's action) — none of which require any change to the model itself to cause its predictions to become progressively less accurate/relevant. This is why production ML systems need **continuous monitoring for drift** and a **retraining cadence/trigger strategy** (scheduled or drift-triggered), rather than treating a validated, deployed model as a "finished," static artifact the way a traditional software feature might be treated once released and tested.

---

### Q: What is the difference between concept drift and data drift, and why does distinguishing between them matter for choosing a mitigation strategy?

**Answer:**
**Data drift (covariate shift)** occurs when the **distribution of input features** changes over time, while the underlying relationship between features and the target remains the same — e.g., user demographics shifting due to a new marketing campaign reaching a different audience. **Concept drift** occurs when the **relationship between features and the target itself changes** — e.g., what predicts customer churn changes after a pricing model change, even if the input feature distributions look similar. This distinction matters because data drift can sometimes be addressed by **retraining on more recent data** with the same modeling approach, while genuine concept drift may require **rethinking the feature set or model approach entirely**, since the fundamental patterns the model needs to learn have actually changed, not just the population it's being applied to.

---

### Q: What is a "shadow deployment" (or "dark launch"), and how does it let a team validate a new model before it affects real users?

**Answer:**
A shadow deployment runs a new model **alongside** the current production model on live traffic, but the new model's predictions are only **logged, not actually acted upon or shown to users** — the production model's predictions remain the ones that drive real behavior. This lets a team validate the new model's real-world behavior (latency, prediction distribution, edge-case handling on genuine production traffic patterns) with **zero user-facing risk**, before committing to a proper A/B test or full rollout that would actually let the new model's decisions affect real users/outcomes — it's a way to catch operational surprises (unexpected latency, crashes on unusual real inputs, wildly different prediction distributions than expected from offline validation) before any of them could cause real business impact.

---

### Q: What is the difference between reproducibility and determinism in the context of ML pipelines, and why can a "reproducible" pipeline still produce a slightly different model on reruns?

**Answer:**
**Determinism** means a given process, run with identical inputs and identical configuration, always produces **bit-for-bit identical output**. **Reproducibility** is a somewhat broader, more practically-oriented goal — being able to **recreate a model with materially equivalent behavior/performance** given the same recorded code, data, and configuration, even if not every internal computation is perfectly bit-identical. A pipeline can be "reproducible" in this practical sense while still not being fully deterministic due to sources like **non-deterministic GPU operations** (certain parallel floating-point operations can produce tiny numerical differences depending on execution order/hardware), **unseeded randomness** in data shuffling or initialization if seeds aren't explicitly fixed, or **library/dependency version drift** between runs if the environment isn't fully pinned — which is why rigorous MLOps practice explicitly pins library versions, sets random seeds, and containerizes the training environment, treating true determinism as a deliberate, engineered property rather than an automatic given.

---

### Q: What is technical debt in ML systems, and why did the influential "Machine Learning: The High-Interest Credit Card of Technical Debt" framing argue that ML systems accumulate technical debt in ways beyond typical software?

**Answer:**
Beyond standard software technical debt (messy code, inadequate tests), ML systems accumulate debt through issues specific to the ML lifecycle: **entanglement** (changing one feature or hyperparameter can silently shift the behavior/importance of seemingly unrelated features — the "CACE principle," Changing Anything Changes Everything, since ML models learn complex interdependencies that aren't visible or modular the way well-separated software components are), **hidden feedback loops** (a model's own predictions influence future training data in ways that aren't obvious — e.g., a recommendation model shaping what users click on, which then becomes the training data for the next model version, creating a self-reinforcing loop), **glue code and pipeline jungles** (ML systems often accumulate large amounts of brittle, hard-to-maintain data transformation/plumbing code connecting generic ML libraries to specific business systems), and **undeclared consumers** (other systems silently depending on a model's output without the model owner's awareness, making it risky to change the model without breaking something unexpected downstream). Recognizing these ML-specific debt patterns is a big part of why disciplined MLOps practice (testing, monitoring, clear ownership/contracts between systems) matters more in ML than it might seem to at first glance.

---
