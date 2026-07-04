# 🔎 Vector Databases & Retrieval Systems

[← Back to main README](../README.md)

---

### Q: What is a vector database, and how does it differ from a traditional relational database in terms of what query operation it's optimized for?

**Answer:**
A vector database stores high-dimensional numerical vectors (embeddings) and is optimized for **similarity search** — finding the vectors closest to a given query vector by some distance metric (cosine similarity, Euclidean distance) — rather than the exact-match/range-based lookups (via B-tree indexes) that relational databases are optimized for. This distinction matters operationally because similarity search over millions/billions of high-dimensional vectors via naive brute-force comparison is computationally expensive (**O(n)** per query against the full dataset), so vector databases implement specialized **Approximate Nearest Neighbor (ANN)** indexing structures (like HNSW or IVF) to make similarity search practically fast at scale — a capability a traditional relational database's indexing structures aren't designed to provide efficiently.

---

### Q: What is Approximate Nearest Neighbor (ANN) search, and why do production vector databases use approximate rather than exact nearest-neighbor search?

**Answer:**
Exact nearest-neighbor search guarantees finding the truly closest vectors to a query but requires comparing against every vector in the dataset (or using exact indexing structures like KD-trees, which degrade in effectiveness in very high dimensions — the "curse of dimensionality"), making it computationally impractical at the scale (millions to billions of vectors) many production systems operate at. **ANN algorithms** (like HNSW — Hierarchical Navigable Small World graphs, or IVF — Inverted File indexing) trade a small, usually acceptable amount of **recall accuracy** (occasionally missing the absolute closest match, returning a very close but not perfectly optimal result) for **dramatically faster query performance** — this tradeoff is almost always worthwhile in practice, since the marginal quality difference between the true nearest neighbor and a very close approximate match is rarely meaningful for downstream application quality, while the speed difference is often the difference between a production-viable system and one that's far too slow to use.

---

### Q: What operational tradeoffs would you consider when tuning an HNSW index's parameters (e.g., `ef_construction`, `M`) for a production vector search deployment?

**Answer:**
HNSW parameters trade off **index build time/memory usage** against **query speed and recall accuracy**. Higher `M` (number of connections per node in the graph) and higher `ef_construction` (search depth during index building) generally produce a **higher-quality index with better recall**, but at the cost of **significantly longer index build times and higher memory consumption**. At query time, a higher `ef_search` parameter improves recall at the cost of slower query latency. An AI Ops engineer tuning these for production needs to balance: **acceptable query latency** for the application's use case, **acceptable recall** (how much quality loss from approximate search is tolerable), **available memory** (larger, higher-quality indexes consume more RAM, directly affecting infrastructure cost), and **how frequently the index needs to be rebuilt** (since a slower-to-build but higher-quality index might be perfectly fine for a knowledge base that updates weekly, but impractical for one that needs near-real-time updates).

---

### Q: What is chunking strategy in a RAG pipeline, and why does chunk size/overlap significantly affect both retrieval quality and downstream generation quality?

**Answer:**
Chunking splits source documents into smaller segments before embedding and indexing them, since embedding an entire long document as a single vector loses too much granular detail for effective retrieval of specific relevant passages. **Chunk size** matters because too-small chunks may lack sufficient context to be meaningful/coherent on their own (a retrieved sentence fragment without surrounding context may confuse the generation step), while too-large chunks dilute the specific relevant information with a lot of irrelevant surrounding content (reducing retrieval precision, since the embedding represents an average of many different concepts within the chunk, and consuming more of the limited context window for less specifically relevant content). **Chunk overlap** (including some shared content between consecutive chunks) helps mitigate the risk of an important piece of information being awkwardly split right at a chunk boundary, at the cost of some storage/indexing redundancy. Tuning these parameters well typically requires empirical evaluation on representative queries/documents specific to the application, rather than relying on generic default values.

---

### Q: What is hybrid search (combining vector/semantic search with traditional keyword/BM25 search), and why might a production RAG system need both rather than relying purely on semantic vector search?

**Answer:**
Hybrid search combines **semantic/vector similarity search** (good at matching conceptually related content even without exact keyword overlap) with **traditional keyword-based search** like BM25 (good at precisely matching specific exact terms, codes, names, or acronyms that a semantic embedding might not represent distinctly enough to retrieve reliably). Pure vector search can sometimes underperform on queries containing **specific exact identifiers** (product SKUs, error codes, precise proper nouns) where an exact keyword match is actually the more reliable retrieval signal than semantic similarity — combining both (often via a weighted fusion of the two ranking scores) generally produces more robust retrieval across a wider variety of query types than either approach alone, which is why many production RAG systems have moved toward hybrid retrieval rather than relying solely on vector similarity.

---

### Q: What operational considerations arise when you need to re-embed and re-index an entire knowledge base after upgrading to a new embedding model, and why can't you simply mix old and new embeddings in the same index?

**Answer:**
Embeddings from **different embedding models (or even different versions of the same model) exist in different, generally incompatible vector spaces** — the numerical position/meaning of a vector produced by one embedding model has no consistent geometric relationship to a vector produced by a different model, so directly comparing similarity between an old-model embedding and a new-model query embedding (or vice versa) produces meaningless results. This means upgrading an embedding model requires **fully re-embedding and re-indexing the entire knowledge base** with the new model — a potentially significant operational undertaking for a very large corpus, requiring careful planning: computing the cost/time of full re-embedding, deciding whether to run old and new indexes **in parallel during a migration window** (querying both and comparing, or gradually cutting over) rather than an abrupt cutover, and validating retrieval quality on the new index before fully decommissioning the old one — this is meaningfully more involved than a typical software dependency upgrade precisely because of this vector-space incompatibility.

---

### Q: What is metadata filtering in vector search, and why is combining metadata filters with vector similarity search often essential for real-world RAG applications rather than pure unfiltered semantic search?

**Answer:**
Metadata filtering restricts vector similarity search to only consider vectors matching specific structured criteria (e.g., document date range, access permission level, document type/category) **alongside** the semantic similarity ranking, rather than searching the entire vector space unconstrained. This is often essential in real-world applications for reasons like: **access control** (a user should only retrieve content they're actually authorized to see, which pure semantic similarity has no inherent awareness of), **recency requirements** (retrieving only documents from within a relevant time window, since a semantically similar but outdated document could be actively misleading), and **precision improvements** (narrowing the search space to a relevant subset before ranking by similarity often produces more relevant top results than ranking the entire unfiltered corpus). Efficient metadata-filtered vector search (filtering *before* or *during* the ANN search rather than as a slow post-hoc filter on results) is a meaningful engineering capability that varies significantly in implementation quality/performance across different vector database products.

---

### Q: How would you monitor a production vector database's health and performance, and what metrics are specific to vector search (beyond typical database metrics like query latency and error rate)?

**Answer:**
Beyond standard database health metrics (query latency, error rate, resource utilization), vector-search-specific monitoring should track: **recall/quality metrics** (periodically running known-answer test queries and verifying expected documents are still retrieved with acceptable ranking, catching silent index corruption or degraded ANN parameter effectiveness), **index freshness/staleness** (how far behind the live index is from the latest source document updates, especially relevant for systems with frequently-changing knowledge bases and asynchronous re-indexing pipelines), **embedding pipeline health** (failures/errors in the embedding generation step, which could cause documents to silently fail to be indexed at all), and **query distribution analysis** (tracking whether real user queries are returning low-confidence/low-similarity results at a concerning rate, potentially indicating a knowledge base coverage gap relative to actual user needs).

---

### Q: What is the difference between storing embeddings in a dedicated vector database versus using a vector-search extension/feature within an existing general-purpose database (e.g., pgvector for PostgreSQL), and what factors would drive that architectural choice?

**Answer:**
A **dedicated vector database** (e.g., Pinecone, Weaviate, Milvus) is purpose-built for large-scale, high-performance vector search, typically offering more advanced/optimized ANN implementations, better horizontal scalability specifically for vector workloads, and vector-search-specific operational tooling. A **vector-search extension within an existing general-purpose database** (e.g., pgvector in Postgres) lets a team **avoid operating an entirely separate database system**, keeping vector data alongside existing relational data with the ability to join/query them together directly and leverage existing operational expertise/tooling already built around that database — appropriate for **moderate-scale use cases** where the operational simplicity of avoiding a new system outweighs the potentially lower raw performance/scalability ceiling compared to a dedicated vector database at very large scale. The right choice depends heavily on **expected scale** (millions vs. billions of vectors), **query performance requirements**, and **the value of tight integration with existing relational data** for the specific application.

---

### Q: What is index rebuilding/compaction for a vector database, and why might a production system need a strategy for periodic reindexing even without any embedding model changes?

**Answer:**
Beyond the full re-embedding scenario (triggered by an embedding model upgrade), vector indexes can also need periodic **rebuilding/compaction** simply due to the normal operational lifecycle of **inserts, updates, and deletes** accumulating over time — many ANN index structures (like HNSW graphs) degrade somewhat in search quality/efficiency as a large volume of individual insertions and deletions accumulate without a full rebuild, since incremental updates to these graph-based structures aren't always as clean/optimal as building the structure fresh from the complete, current dataset. A production system with a frequently-changing knowledge base needs a deliberate strategy for **when and how often to perform a full index rebuild** (balancing the computational cost/downtime risk of rebuilding against the gradual quality degradation of an aging, heavily-mutated index), which is an ongoing operational maintenance concern rather than a one-time setup decision.

---

### Q: How would you design a disaster recovery / backup strategy for a production vector database powering a business-critical RAG application, given that re-embedding an entire large corpus from scratch could take hours or days?

**Answer:**
Given that a full re-embedding/re-indexing from raw source documents could be prohibitively slow to use as your *only* recovery mechanism, a robust strategy should include: **regular snapshots/backups of the actual vector index itself** (not just the raw source documents), stored in durable, geographically-redundant storage, allowing much faster recovery (restoring a snapshot) than a full from-scratch re-embedding job. Additionally, maintain **redundant, replicated vector database infrastructure** (across availability zones or regions, depending on the required resilience level) so a single infrastructure failure doesn't require any recovery process at all in the common case. The **re-embedding-from-source path should still exist and be periodically tested** as a last-resort recovery mechanism (for scenarios like index corruption that also affects backups, or a need to recover from a point further back than available snapshots), but treating it as the *primary* recovery strategy — rather than a tested, fast-restoring backup/replication strategy — would likely violate any reasonable recovery time objective (RTO) for a business-critical system.

---
