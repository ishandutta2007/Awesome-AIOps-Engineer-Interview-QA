# 📦 Containers, Kubernetes & Orchestration for ML

[← Back to main README](../README.md)

---

### Q: Why has containerization (Docker) become close to a default standard practice for packaging ML training and serving workloads?

**Answer:**
ML workloads have notoriously complex, sensitive dependency requirements — specific versions of CUDA drivers, deep learning frameworks, and numerous Python libraries that must all be precisely compatible with each other and with the underlying model code, and even small version mismatches can cause subtle numerical differences or outright failures. Containerization packages the **entire runtime environment** (OS-level dependencies, CUDA/driver compatibility layers, Python environment, application code) into a single, portable, reproducible artifact — ensuring the exact same environment that worked during development/training reliably reproduces in any other environment (a different team member's machine, a CI pipeline, or production serving infrastructure) without the "works on my machine" class of problems that are especially prevalent and painful given ML's particularly fragile dependency stacks.

---

### Q: What is the difference between a standard Kubernetes deployment and using a Kubernetes-native ML orchestration layer like Kubeflow or KServe, and what specific ML-related gaps do these frameworks fill?

**Answer:**
A standard Kubernetes `Deployment` provides generic container orchestration — scheduling, scaling, and self-healing for any containerized workload, with no inherent awareness of ML-specific concepts. **Kubeflow** and similar ML-focused frameworks build on top of Kubernetes to add **ML-specific abstractions and workflows**: multi-step pipeline orchestration designed around ML lifecycle stages, native support for distributed training job patterns (e.g., coordinating multiple worker/parameter-server pods for a distributed training job, which requires specific coordination logic beyond a simple deployment), integrated experiment tracking and model registry concepts, and (in the case of **KServe** specifically) standardized, production-grade model serving abstractions (canary rollouts, autoscaling tuned for inference workload patterns, standardized inference protocols) — essentially, these frameworks provide the ML-specific "batteries" that plain Kubernetes deliberately leaves out as a general-purpose orchestration platform.

---

### Q: What is a Kubernetes resource request versus a resource limit for a GPU-using pod, and why is correctly configuring both particularly important (and sometimes tricky) for GPU workloads specifically?

**Answer:**
A **resource request** specifies the minimum resources Kubernetes guarantees a pod when scheduling it onto a node. A **resource limit** caps the maximum a pod can consume. This is particularly important — and different — for **GPU resources** compared to CPU/memory because **GPUs generally cannot be fractionally shared/oversubscribed by default the way CPU can** (without specific technologies like MIG or time-slicing, discussed in the GPU infrastructure section) — a pod requesting a GPU typically gets **exclusive access to an entire physical GPU** regardless of whether it actually needs the full capacity, meaning GPU resource requests directly and immediately translate into how many separate workloads can be scheduled per GPU-equipped node, with essentially no natural "bursting" or oversubscription safety net the way slightly-over-requesting CPU/memory resources often has in practice — misconfigured GPU resource requests can lead to either significant underutilization (over-requesting) or scheduling failures (under-provisioning available GPU-equipped nodes for the actual demand).

---

### Q: What is a Kubernetes Horizontal Pod Autoscaler (HPA), and why might standard CPU/memory-based autoscaling metrics be insufficient for autoscaling a GPU-based model serving deployment effectively?

**Answer:**
HPA automatically adjusts the number of running pod replicas based on observed metrics, commonly CPU or memory utilization by default. For **GPU-based inference serving**, CPU/memory utilization is often a **poor proxy for actual serving capacity/load** — a GPU-serving pod might show low CPU utilization (since the CPU is mostly just handling request routing/preprocessing while the GPU does the heavy computational lifting) even while the **GPU itself is fully saturated** and struggling to keep up with incoming request volume, meaning CPU-based autoscaling would fail to scale up when actually needed. Effective GPU workload autoscaling typically requires **custom metrics** more directly reflective of actual serving load — e.g., **GPU utilization itself** (via custom metrics adapters exposing GPU telemetry to Kubernetes), **request queue depth/length**, or **inference latency percentiles** — configured as the actual scaling trigger metric instead of, or in addition to, the default CPU/memory-based metrics.

---

### Q: What is a Kubernetes node affinity/taint-and-toleration setup, and why would an AI Ops engineer configure this specifically to ensure GPU workloads are scheduled only onto GPU-equipped nodes?

**Answer:**
**Node affinity** lets a pod specify a preference or requirement for being scheduled onto nodes with specific labels (e.g., a specific GPU type). **Taints and tolerations** work in the complementary direction — a node can be "tainted" to repel pods by default unless they explicitly "tolerate" that taint, commonly used to **reserve expensive GPU nodes exclusively for GPU-requiring workloads**, preventing regular CPU-only workloads from accidentally being scheduled onto (and wasting capacity on) expensive GPU-equipped nodes that should be reserved for workloads that actually need that hardware. Configuring both together (labeling/tainting GPU nodes, and having GPU-requiring pods declare the corresponding affinity/toleration) ensures the Kubernetes scheduler makes **cost-aware, hardware-aware placement decisions** automatically, rather than relying on manual coordination or accidentally allowing expensive GPU capacity to be consumed by workloads that didn't actually need it.

---

### Q: What is a StatefulSet in Kubernetes, and why might it be more appropriate than a standard Deployment for certain ML infrastructure components (e.g., a distributed training job's worker pods)?

**Answer:**
A standard `Deployment` treats its pod replicas as **interchangeable and stateless** — any replica can be killed and replaced by an identical new one without any special ordering or identity requirements. A `StatefulSet` provides **stable, unique network identities and stable persistent storage** per pod, along with **ordered, predictable deployment/scaling/termination** — properties needed for workloads where each replica has a distinct role or needs consistent identity across restarts. This matters for certain ML infrastructure like **distributed training jobs** where individual worker pods might need a **stable, known identity/rank** (e.g., "worker-0," "worker-1") for correct coordination logic (like all-reduce communication patterns in distributed training, which often rely on each worker knowing its specific rank/position), or for a **stateful component like a database or certain distributed storage systems** underlying an ML pipeline, where losing a pod's specific identity/storage association would break the system's consistency guarantees.

---

### Q: What is the difference between a Kubernetes Job and a CronJob, and how would each be used in a typical ML pipeline running on Kubernetes?

**Answer:**
A **Job** runs a pod (or set of pods) to completion for a **one-off task**, ensuring it runs to successful completion (with configurable retry behavior on failure) rather than running indefinitely like a typical long-lived service deployment — appropriate for a **single training run** or a one-time batch processing task. A **CronJob** creates Jobs automatically on a **recurring schedule** (using standard cron syntax) — appropriate for **regularly scheduled retraining pipelines**, periodic batch inference jobs, or scheduled data validation/monitoring checks that need to run on a fixed cadence (e.g., nightly) without manual intervention to trigger each run. Understanding this distinction matters for correctly modeling a given ML pipeline stage's actual execution pattern — a training pipeline invoked ad-hoc by a data scientist might use a `Job`, while the same pipeline set up to auto-retrain weekly would be wrapped in a `CronJob`.

---

### Q: What is a sidecar container pattern, and give an example of how it might be used in an ML serving pod (e.g., alongside the main model-serving container).

**Answer:**
A sidecar is a **secondary container running alongside the main application container within the same pod**, sharing the pod's network namespace and often shared storage volumes, used to handle auxiliary responsibilities without embedding that logic directly into the main application container. In an ML serving context, a sidecar might handle: **metrics/logging collection and forwarding** (scraping and shipping the model server's metrics/logs to a centralized observability platform, without the model-serving container needing to implement that logic itself), **model artifact synchronization** (a sidecar that periodically checks for and downloads a newer model version from the model registry, updating a shared volume the main serving container reads from, enabling model updates without needing a full pod restart), or a **service mesh proxy** (handling network-level concerns like mTLS, traffic routing, and observability transparently for the main container). This pattern keeps the core model-serving container's responsibility narrowly focused on inference itself, delegating cross-cutting operational concerns to purpose-built, reusable sidecar components.

---

### Q: What is resource quota management in a multi-tenant Kubernetes cluster shared by multiple ML teams, and why is this a particularly important (and sometimes contentious) concern for shared GPU clusters specifically?

**Answer:**
Resource quotas set limits on the total resources (CPU, memory, and critically, GPU count) a given namespace/team can consume within a shared cluster, preventing any single team/workload from monopolizing the cluster's finite resources at the expense of others. This is a particularly important — and sometimes contentious — concern for **shared GPU clusters** specifically because GPUs are typically a **much scarcer, more expensive, and less elastically available resource** than CPU/memory in most cloud environments, meaning conflicts over GPU quota allocation directly translate into real friction between teams competing for a genuinely limited, high-value resource (unlike CPU, where it's often comparatively easy/cheap to simply provision more capacity to avoid the conflict entirely). Effective GPU quota management typically requires not just static per-team limits, but often more sophisticated mechanisms like **priority classes/preemption** (allowing higher-priority workloads to reclaim GPU capacity from lower-priority ones when contention occurs) and **fair-share scheduling policies**, combined with organizational processes for negotiating and periodically revisiting quota allocations as different teams' needs evolve.

---

### Q: What is a readiness probe versus a liveness probe in Kubernetes, and why is correctly configuring the readiness probe particularly important for a model-serving pod that has a meaningful cold-start/model-loading time?

**Answer:**
A **liveness probe** determines if a container is still running correctly; if it fails, Kubernetes restarts the container. A **readiness probe** determines if a container is currently ready to receive traffic; if it fails, Kubernetes temporarily removes the pod from the service's load-balancing pool without restarting it. For a model-serving pod with a **meaningful model-loading/warm-up time** (as discussed in the deployment section), correctly configuring the readiness probe to only report "ready" **after the model has actually finished loading into memory and is genuinely prepared to serve requests** is critical — without this, Kubernetes might start routing live traffic to a newly-started pod immediately upon container startup, well before the model is actually loaded and ready, causing failed or extremely slow requests during that startup window. Getting this wrong (e.g., a readiness probe that just checks "is the web server process running" rather than "has the model finished loading") is a common, easy-to-overlook cause of intermittent errors specifically during pod restarts/scaling events for ML serving workloads.

---

### Q: How would you design a Kubernetes-based CI/CD deployment strategy for a model serving service to support canary rollouts (as discussed in the deployment topic), using Kubernetes-native or service-mesh capabilities?

**Answer:**
A common approach uses **traffic-splitting capabilities** provided either by a service mesh (like Istio) or an ingress controller supporting weighted routing, combined with running the new model version as a **separate deployment/pod set** alongside the existing production version. The traffic-splitting layer is configured to route a small, controlled percentage of live traffic to the new version's pods while the majority continues to the stable version, with this percentage gradually increased based on automated canary analysis (comparing error rates, latency, and ideally model-specific quality signals between the two versions, as discussed in the deployment topic) — Kubernetes-native tools like **Flagger** or **Argo Rollouts** specifically automate this entire progressive-delivery workflow (traffic shifting, automated metric-based promotion/rollback decisions) on top of the underlying Kubernetes/service-mesh traffic-splitting primitives, rather than requiring a team to hand-build and manually operate this canary orchestration logic themselves from scratch.

---
