# 💰 GPU Infrastructure, Scaling & Cost Optimization

[← Back to main README](../README.md)

---

### Q: What is the difference between GPU utilization and GPU memory utilization as monitoring metrics, and why can a training/inference job show low compute utilization while still being memory-bound?

**Answer:**
**GPU compute utilization** measures how busy the GPU's processing cores are. **GPU memory utilization** measures how much of the GPU's VRAM is occupied. A job can show **low compute utilization while being memory-bound** when, for example, the model/batch is so large that most available VRAM is consumed just holding weights/activations, leaving little room to run a batch size large enough to keep the compute cores fully occupied — or when the workload is bottlenecked by **data loading/transfer speed** (CPU-to-GPU transfer, or slow storage I/O feeding training data) rather than actual computation, leaving the GPU cores frequently idle waiting for data despite memory being heavily utilized. Diagnosing which specific resource is the actual bottleneck (compute, memory, I/O, data loading) is essential before applying the right optimization — increasing batch size doesn't help an I/O-bound job, and optimizing data loading doesn't help a genuinely compute-bound job.

---

### Q: What is the difference between training-time GPU infrastructure needs and inference-time GPU infrastructure needs, and why might an organization use entirely different GPU types/configurations for each?

**Answer:**
**Training** typically requires **large amounts of high-bandwidth memory and strong multi-GPU interconnect** (for distributed training across many GPUs, requiring fast communication for gradient synchronization) — often justifying more expensive, higher-end GPUs (e.g., NVIDIA H100s) run in large, tightly-networked clusters for a bounded training period. **Inference** at scale often benefits more from **cost-efficient throughput and low latency for a single (or modest batch of) forward pass(es)**, and depending on the model size, can often run effectively on smaller, cheaper GPU instances, or even benefit significantly from specialized inference-optimized hardware/quantization — and inference infrastructure typically needs to run **continuously, at scale, matching variable real-time traffic** rather than training's bounded, scheduled job pattern. This is why many organizations deliberately provision and manage separate GPU fleets/configurations for training versus inference workloads, since optimizing for one doesn't naturally optimize for the other.

---

### Q: What is the difference between data parallelism and model parallelism in distributed training, and when does a model require model parallelism rather than just data parallelism?

**Answer:**
**Data parallelism** replicates the **entire model** across multiple GPUs/nodes, with each replica processing a different subset (shard) of the training batch in parallel, then synchronizing gradients across replicas — this works well as long as the model itself fits comfortably in a single GPU's memory. **Model parallelism** splits the **model itself** across multiple GPUs (e.g., different layers on different GPUs, or splitting individual layers via tensor parallelism) — this becomes necessary when a model is **too large to fit in a single GPU's memory at all**, regardless of batch size, which is increasingly common for very large language models. In practice, large-scale LLM training typically uses **a combination of both** (and additional techniques like pipeline parallelism) simultaneously, since neither approach alone is sufficient at the largest current model scales — an AI Ops engineer supporting large model training needs to understand these tradeoffs to correctly provision cluster topology (GPU count, interconnect bandwidth requirements) for a given model size and training approach.

---

### Q: What is spot/preemptible instance usage for ML workloads, and what specific engineering practices are needed to safely use much cheaper spot instances for training without losing significant progress when they're preempted?

**Answer:**
Spot/preemptible instances offer substantial cost savings (often 60-90% cheaper) compared to on-demand instances, in exchange for the cloud provider being able to **reclaim the instance with little notice** when the underlying capacity is needed elsewhere. Safely using spot instances for training requires: **frequent checkpointing** (saving model/optimizer state regularly enough that a preemption only loses a small, acceptable amount of recent progress, not hours of training), **automatic job resumption logic** (the training pipeline should automatically detect a preemption, acquire new spot capacity, and resume from the latest checkpoint without requiring manual intervention), and **graceful preemption handling** (many cloud providers give a short warning period, e.g., 30 seconds to 2 minutes, before actually reclaiming the instance — the training job should ideally use this window to trigger an immediate, final checkpoint save rather than being abruptly killed mid-step). Without these practices in place, spot instances can actually end up costing more in wasted, unrecoverable compute time than the on-demand savings are worth.

---

### Q: What is the concept of "right-sizing" GPU instances for inference, and describe a systematic approach to determining the appropriate instance type/size for a given model rather than guessing or defaulting to the largest available option.

**Answer:**
Right-sizing means selecting the smallest/cheapest instance configuration that still meets the required performance (latency, throughput) SLAs for a given workload, rather than over-provisioning "just to be safe" (which directly wastes cost) or under-provisioning (which causes SLA violations). A systematic approach: **benchmark the actual model** on a range of candidate instance types/sizes, measuring real latency and throughput at realistic batch sizes and concurrency levels representative of actual production traffic patterns (not just a single-request synthetic benchmark, which often doesn't reflect real-world batched serving performance), **calculate the cost-per-inference or cost-per-token** for each candidate configuration at the throughput level needed, and **factor in headroom for traffic variability** (peak vs. average load) when deciding on both the base instance size and the autoscaling strategy around it — treating right-sizing as an ongoing, re-evaluated practice (since traffic patterns and available hardware/pricing options change over time) rather than a one-time decision made at initial launch.

---

### Q: What is tensor/mixed-precision training, and how does it reduce both memory usage and training time without significantly harming model quality?

**Answer:**
Mixed-precision training uses **lower-precision numerical formats** (like FP16 or BF16) for most computations during training, instead of full FP32 precision, while selectively keeping certain numerically sensitive operations (like gradient accumulation in some implementations) in higher precision to maintain training stability. This reduces **memory usage** (lower-precision numbers take less space, allowing larger batch sizes or bigger models to fit in the same GPU memory) and **speeds up training** (modern GPUs have specialized hardware — e.g., NVIDIA Tensor Cores — that perform lower-precision matrix operations significantly faster than full FP32 operations) — the careful mixed-precision approach (rather than using low precision everywhere naively) is specifically designed to capture most of these performance/memory benefits while avoiding the training instability (e.g., gradient underflow) that naive full low-precision training can otherwise cause.

---

### Q: What is autoscaling for GPU-based inference workloads, and why is it typically more challenging to implement effectively than autoscaling for typical stateless CPU-based web services?

**Answer:**
GPU-based inference autoscaling faces several challenges beyond typical CPU-based web service autoscaling: **slower cold-start/scale-up time** (loading large model weights into GPU memory and initializing hardware acceleration contexts takes meaningfully longer than starting a typical lightweight web service container, making rapid response to sudden traffic spikes harder), **coarser-grained, more expensive scaling units** (a GPU instance is a much larger, more expensive unit of capacity to add/remove compared to a small CPU container, making overly reactive/thrashing autoscaling decisions costlier both in wasted spend and in the disruption of frequent scale events), and **GPU availability constraints** (in periods of high demand, the desired GPU instance type may simply not be available to provision immediately, unlike the generally more elastic availability of standard CPU compute) — this combination of factors means GPU inference autoscaling strategies often need to be more conservative/predictive (e.g., maintaining a buffer of warm capacity, or using traffic forecasting to pre-scale ahead of known demand patterns) rather than relying purely on fast, reactive scaling the way many CPU-based web services can.

---

### Q: What is the concept of "cost attribution" for shared ML/GPU infrastructure, and why does an AI Ops team need robust cost attribution/chargeback mechanisms as an organization scales its number of ML teams/projects sharing common infrastructure?

**Answer:**
Cost attribution tracks and allocates the actual infrastructure cost incurred by **specific teams, projects, or models** using shared GPU/ML infrastructure, rather than treating the entire infrastructure spend as one large, undifferentiated cost center. This becomes increasingly important at scale because without it, there's **no accountability or visibility into which specific workloads are actually driving cost** — making it very difficult to identify optimization opportunities (e.g., "which specific model/team's inefficient batch job is responsible for this cost spike"), to make informed prioritization decisions about infrastructure investment, or to hold individual teams accountable for the cost efficiency of their own workloads. Implementing this typically requires **tagging/labeling infrastructure resources** by team/project/model at provisioning time, combined with cost monitoring tooling that can aggregate and report spend along these dimensions — a capability that needs to be deliberately built into the infrastructure provisioning process from early on, since retrofitting proper cost attribution onto already-running, unlabeled shared infrastructure is significantly harder after the fact.

---

### Q: What is model/GPU sharing (e.g., NVIDIA MIG — Multi-Instance GPU, or time-slicing), and what problem does it solve for organizations running many smaller models that individually don't require a full GPU's capacity?

**Answer:**
GPU sharing technologies let a **single physical GPU be partitioned or time-shared among multiple separate workloads**, rather than requiring each workload to have exclusive access to an entire GPU even if it only needs a small fraction of its compute/memory capacity. This solves a significant **cost efficiency problem** for organizations running many smaller models (e.g., many lightweight models each serving relatively low individual traffic) — without GPU sharing, each such model would need its own dedicated (and often substantially underutilized) full GPU instance, representing significant wasted spend on unused capacity. **NVIDIA MIG** provides **hardware-level partitioning** (each partition gets guaranteed, isolated compute/memory resources, providing stronger performance isolation between co-located workloads) while **time-slicing** shares the GPU sequentially across workloads (simpler to configure but with less strict performance isolation, since workloads compete for the same underlying resources over time) — the right choice depends on how strictly isolated/predictable each workload's performance needs to be versus prioritizing simpler configuration and potentially higher aggregate utilization.

---

### Q: How would you approach capacity planning for an anticipated large increase in inference traffic (e.g., a planned product launch expected to significantly increase usage), given the longer lead times often involved in provisioning additional GPU capacity compared to standard CPU infrastructure?

**Answer:**
Given that **GPU capacity, especially high-demand GPU types, often has meaningfully longer procurement/provisioning lead times** than standard cloud CPU instances (sometimes requiring advance reservations or commitments with cloud providers well ahead of need), capacity planning for a known future demand increase requires: **forecasting expected load** based on the best available business projections (even if imperfect, an informed estimate is far better than reactive scrambling), **securing capacity commitments/reservations well in advance** of the anticipated launch date rather than assuming on-demand capacity will simply be available exactly when needed, **load testing the actual serving infrastructure** at the projected scale ahead of time (not just assuming linear scaling from current traffic will hold, since bottlenecks often emerge non-linearly at higher scale — e.g., in shared dependencies like a feature store or database that individual replicas all rely on), and **building in a realistic buffer/margin** above the point-estimate forecast, since demand forecasts for new product launches are often meaningfully uncertain in either direction, and running out of committed capacity during a high-visibility launch is a significantly worse outcome than having modestly over-provisioned capacity that gets scaled back down shortly afterward.

---

### Q: What is the tradeoff between using a fully managed inference platform (e.g., a cloud provider's managed model endpoint service) versus self-managing GPU infrastructure and a serving framework directly, from a cost and operational-effort standpoint?

**Answer:**
A **fully managed inference platform** typically charges a **premium over raw underlying compute cost** in exchange for handling much of the operational burden (autoscaling, patching, monitoring infrastructure, availability management) automatically — reducing the AI Ops team's operational effort and time-to-deployment, which can be the right tradeoff especially for smaller teams or when engineering time is the more scarce/valuable resource relative to the direct infrastructure cost premium. **Self-managing GPU infrastructure** directly (provisioning raw GPU instances and running an open-source serving framework like Triton or vLLM yourself) typically offers **lower raw compute cost and more fine-grained control** over performance tuning, batching behavior, and cost optimization strategies (like spot instances or custom GPU-sharing configurations) — but requires the team to build and maintain significantly more of the operational tooling and expertise themselves. The right choice often depends on **scale** (self-managed infrastructure's cost advantage typically grows more compelling at larger scale, where the managed platform's percentage premium translates to larger absolute dollar amounts) and the **team's specific operational expertise/capacity** to take on that additional responsibility reliably.

---
