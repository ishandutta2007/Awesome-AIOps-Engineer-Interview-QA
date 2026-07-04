# 🧠 LLMOps: Prompting, RAG & Evaluation

[← Back to main README](../README.md)

---

### Q: What is LLMOps, and how does it differ from traditional MLOps in terms of what needs to be operationalized?

**Answer:**
LLMOps applies MLOps principles specifically to large language model applications, but addresses several concerns that don't exist (or exist very differently) for traditional predictive ML models: **prompt versioning and management** (prompts function like a critical, frequently-iterated "code" artifact that directly affects model behavior, requiring its own version control and testing), **evaluation of open-ended, non-deterministic text output** (traditional ML has clear accuracy/precision metrics; evaluating whether a generated response is "good" is far more subjective and requires different techniques like LLM-as-judge or human evaluation), **retrieval system operations** (for RAG-based applications, operating and monitoring the retrieval/vector search component alongside the generation component), and **token-based cost and latency management** (costs scale with input/output token counts in a way that's a genuinely new operational dimension compared to traditional model inference cost structures). Many core MLOps concepts (CI/CD, monitoring, versioning) still apply, but the specific implementation details and additional concerns are substantially different.

---

### Q: What is prompt versioning, and why does treating prompts as unversioned, ad-hoc strings scattered through application code create real operational risk?

**Answer:**
Prompt versioning treats prompts (and prompt templates) as first-class, version-controlled artifacts — tracked alongside metadata like which model they were validated against, their evaluation results, and their deployment history — rather than embedding raw prompt strings directly and informally within application code. Treating prompts as unversioned ad-hoc strings creates real risk because: **prompt changes directly and significantly affect output quality/behavior**, similar in impact to a model change, yet without versioning there's **no reliable way to know which prompt version was in use for a given historical output**, no easy rollback if a prompt "improvement" actually regresses quality in some cases, and no systematic way to **A/B test or evaluate** prompt changes rigorously before shipping them broadly — essentially reintroducing the same "how do we know what changed and whether it's actually better" problems that model versioning and CI/CD solved for traditional ML models, but for prompts specifically.

---

### Q: What is Retrieval-Augmented Generation (RAG), and what are the operational components an AI Ops engineer needs to monitor/maintain beyond just the LLM itself?

**Answer:**
RAG combines a retrieval system (searching a knowledge base/document corpus for relevant context) with an LLM that generates a response grounded in the retrieved content. Beyond the LLM inference component itself, an AI Ops engineer needs to operate: the **embedding pipeline** (converting documents and queries into vector representations, which needs monitoring for embedding model version consistency and processing failures), the **vector database/index** (covered in more depth in the next topic — its own availability, latency, and index freshness), the **document ingestion/chunking pipeline** (keeping the knowledge base current as source documents change, and monitoring chunking quality), and the **retrieval quality itself** (are the retrieved documents actually relevant to the query — a failure here silently degrades response quality even if the LLM and vector database are both technically "healthy" and running without errors).

---

### Q: What is the difference between evaluating a traditional ML classifier and evaluating an LLM's generated output, and what makes LLM evaluation meaningfully harder to automate?

**Answer:**
A traditional classifier's output can typically be compared against a single, objectively correct ground-truth label using well-defined metrics (accuracy, precision, F1). An LLM's generated output is **open-ended natural language**, where there's often **no single "correct" answer** — multiple genuinely good responses can differ substantially in wording while being equally valid, and quality dimensions like coherence, factual accuracy, tone, and helpfulness are inherently more **subjective and harder to reduce to a simple automated comparison**. This is why LLM evaluation increasingly relies on techniques like **LLM-as-judge** (using another strong LLM to score outputs against a rubric, as a scalable proxy for human judgment), curated **golden test sets with human-graded reference answers**, and **human evaluation** for high-stakes or nuanced quality dimensions — none of which fully replace the need for careful evaluation design, since automated proxies (including LLM-as-judge) can themselves have systematic biases that need to be understood and accounted for.

---

### Q: What is hallucination in the context of LLMs, and what operational/monitoring strategies can help detect or reduce it in a production RAG system?

**Answer:**
Hallucination refers to an LLM generating **plausible-sounding but factually incorrect or unsupported content** — a particularly significant operational risk since hallucinated output can look confident and well-formed, making it hard for users (and automated systems) to distinguish from accurate content. Mitigation/detection strategies specific to a **RAG system**: **grounding checks** that verify the generated response is actually well-supported by the retrieved source documents (e.g., using a secondary model or heuristic to check claim-by-claim consistency between generated output and retrieved context), **citation requirements** (prompting the model to cite specific retrieved sources for claims, making unsupported claims more visually/structurally apparent to reviewers), and **confidence/uncertainty signals** where available, combined with **sampling-based consistency checks** (generating multiple responses to the same query and flagging cases with high variance/disagreement as more likely to be unreliable). No single technique fully eliminates hallucination, which is why production systems often combine several of these as layered mitigations alongside appropriate user-facing framing about the system's limitations.

---

### Q: What is context window management, and why does an AI Ops engineer need to actively design for it rather than just "sending everything relevant" to the LLM?

**Answer:**
Every LLM has a maximum **context window** (the total amount of input + output tokens it can process in a single call), and exceeding it either causes an error or forces truncation. Beyond the hard technical limit, **larger contexts also increase latency and cost** (since cost and processing time scale with token count) and can sometimes **degrade output quality** (some models exhibit a "lost in the middle" effect, attending less effectively to information buried in the middle of a very long context versus the beginning/end). This means an AI Ops/LLMOps engineer needs to actively design **context construction strategies** — e.g., retrieving and including only the most relevant chunks (RAG's core purpose), summarizing/compressing conversation history for multi-turn applications rather than including the full raw history indefinitely, and setting explicit token budgets across different context components (system prompt, retrieved context, conversation history) — rather than naively assuming "just include everything that might be relevant" is a viable default strategy at any meaningful scale or conversation length.

---

### Q: What is prompt injection, and why is it considered one of the most significant security concerns specific to LLM-powered applications (especially those with tool-use/agentic capabilities)?

**Answer:**
Prompt injection occurs when an attacker crafts input (either directly in a user prompt, or indirectly embedded in content the LLM processes, like a webpage or document it's asked to summarize) designed to **override or manipulate the LLM's intended instructions/behavior**, causing it to ignore its original system prompt/guardrails and instead follow the attacker's injected instructions. This is especially significant for **agentic LLM applications with tool-use capabilities** (e.g., an LLM that can browse the web, execute code, or send emails on a user's behalf) because a successful prompt injection in that context isn't just about generating undesired *text* — it could potentially cause the LLM agent to **take unintended, harmful real-world actions** through its available tools (e.g., an indirect prompt injection hidden in a webpage the agent is summarizing could instruct it to exfiltrate sensitive data via an available email tool). Mitigations remain an active area of development, including input/output filtering, strict tool permission scoping (least privilege for what actions an agent can actually take), and architectural patterns that reduce the LLM's ability to blindly trust untrusted content it processes.

---

### Q: What is the difference between fine-tuning an LLM and using Retrieval-Augmented Generation (RAG), and from an operational standpoint, why might a team choose RAG even when fine-tuning could theoretically achieve similar quality?

**Answer:**
**Fine-tuning** updates the model's weights on task/domain-specific data, baking that knowledge/behavior directly into the model. **RAG** keeps the base model's weights unchanged and instead supplies relevant, current information at inference time via retrieval. From an **operational standpoint**, teams often prefer RAG because: **updating the knowledge base is far cheaper and faster** than retraining/fine-tuning a model every time underlying information changes (critical for frequently-changing information like documentation, policies, or current events), RAG provides **built-in citability/traceability** (you can point to exactly which retrieved document supported a given response, which fine-tuned "baked-in" knowledge can't provide), and RAG **avoids the operational overhead of managing and versioning many fine-tuned model variants** for different knowledge domains, instead managing a more easily updatable knowledge base alongside a single shared base model. Fine-tuning remains valuable for adjusting a model's *behavior/style/format* rather than its factual knowledge, and the two techniques are frequently combined rather than treated as strictly either/or.

---

### Q: What is an LLM gateway/proxy layer, and what operational capabilities does it typically provide for an organization using multiple LLM providers or models?

**Answer:**
An LLM gateway sits between an organization's applications and one or more underlying LLM providers (OpenAI, Anthropic, open-source self-hosted models, etc.), providing a unified interface and centralized operational control. Typical capabilities: **provider abstraction/fallback** (routing requests across multiple providers/models, with automatic failover if one provider has an outage), **centralized cost tracking and rate limiting** (monitoring and capping spend/usage across many internal teams/applications sharing underlying LLM access), **caching** (avoiding redundant, costly calls for identical or semantically similar repeated queries), **centralized logging/observability** for all LLM calls across the organization (supporting both debugging and evaluation), and **consistent guardrail/safety filtering** applied uniformly regardless of which underlying model/provider actually serves a given request — this centralization is a common pattern specifically because managing these operational concerns separately, per application/team, doesn't scale well as LLM usage spreads across an organization.

---

### Q: What is semantic caching for LLM applications, and how does it differ from traditional exact-match caching, and what's the risk of an overly aggressive semantic cache?

**Answer:**
Traditional caching returns a cached result only for an **exact match** of a previous request. **Semantic caching** uses embedding similarity to identify and return a cached response for a **semantically similar (not necessarily identical) previous query**, significantly increasing cache hit rates for LLM applications where users phrase conceptually identical questions in many different ways — providing meaningful cost and latency savings by avoiding a full LLM call for queries that are "close enough" to something already answered. The risk of an **overly aggressive semantic cache** (too loose a similarity threshold) is returning a cached response that's actually **not sufficiently relevant/accurate for the specific nuance of the new query** — e.g., two questions that are superficially similar in phrasing/topic but have a subtly different, important distinction (different date ranges, different specific product versions) could incorrectly be treated as cache-equivalent, silently serving a wrong or outdated answer — tuning the similarity threshold and validating cache-hit quality is a genuine, non-trivial operational tuning problem, not a "set it and forget it" configuration.

---

### Q: What is an LLM evaluation harness/pipeline, and what should it automatically check before a new prompt version, fine-tuned model, or RAG configuration change is promoted to production?

**Answer:**
An evaluation harness runs a candidate LLM configuration (new prompt, new model version, new RAG retrieval setup) against a **curated set of test queries/scenarios** and scores the results against defined criteria, automating what would otherwise be slow, inconsistent manual spot-checking. A robust pre-promotion evaluation pipeline should check: **quality/correctness** against a golden set of representative queries with known-good expected characteristics (via LLM-as-judge scoring, exact/fuzzy matching for factual queries, or human-graded reference comparison), **regression testing against previously-identified failure cases** (ensuring a past, specifically-fixed problem doesn't silently reappear), **safety/guardrail compliance** (testing against known adversarial/edge-case prompts to ensure safety behaviors haven't regressed), and **latency/cost benchmarks** (a prompt or configuration change that improves quality slightly but doubles token usage/cost might not be an acceptable tradeoff without an explicit, informed decision) — treating LLM configuration changes with the same rigor as a code or model change, rather than shipping prompt tweaks based on a few manual spot-checks.

---

### Q: How would you design a monitoring strategy specifically for a production RAG chatbot to detect when retrieval quality has degraded (e.g., due to a stale or poorly-maintained knowledge base), separately from monitoring the generation quality of the LLM itself?

**Answer:**
Monitor the **retrieval component's outputs directly**, independent of what the LLM ultimately generates from them: track the **retrieval relevance score/similarity distribution** for served queries over time (a downward trend suggests the knowledge base isn't matching well against real user queries, perhaps because it's gone stale relative to what users are actually asking about), track the **rate of queries returning zero or very low-confidence retrieval results** (a strong signal of a knowledge base coverage gap), and periodically run **known-answer retrieval test queries** against the live index (verifying that specific, previously-validated documents are still being correctly retrieved for their corresponding known queries, catching index corruption or unexpected re-indexing issues). Separating retrieval-quality monitoring from generation-quality monitoring matters because it lets the team **correctly attribute** a degraded end-to-end user experience to the right underlying cause (a knowledge base/retrieval problem needing content updates, versus an LLM/prompt problem needing a prompt or model change) rather than only seeing an ambiguous aggregate "chatbot answers got worse" signal with no clear diagnostic direction.

---
