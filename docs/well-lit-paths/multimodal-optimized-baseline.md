# Multimodal Optimized Baseline

Traditional HTTP requests are fast, uniform, and cheap. Standard round-robin request scheduling strategies balance this load well.

LLM requests break all three assumptions. Multimodal LLM requests (containing images, video, or audio) break them even further:

* **Context Inflation** — A single high-resolution image, audio clip, or video file drastically inflates the context window (often by thousands of tokens).
* **Heavy Prefill Cost** — Running vision/auditory encoders and prefilling thousands of tokens is highly resource-intensive.

The **llm-d Router** extends text-based prefix scheduling by tracking, hashing, and matching complex multimodal inputs (images, video, audio) across a distributed inference cluster. The goal is to intelligently direct incoming requests containing multimodal payloads to the specific backend worker that already holds the corresponding pre-computed key-value (KV) blocks in its memory.

> [!NOTE]
> This guide builds upon the text-based [Optimized Baseline](optimized-baseline.md) and demonstrates one approach to prefix- and load-aware routing for multimodal workloads. The llm-d Router supports other options as well, including session affinity and active request based routing, which make no assumptions about the router's ability to parse the request or probe the servers. See [configuration](../architecture/core/router/epp/configuration.md) for more details on the available scorers, or [precise prefix cache routing](precise-prefix-cache-routing.md) for KV-event-driven scoring.

---

## Deploy

See the [multimodal optimized baseline guide](../../guides/multimodal-optimized-baseline) for manifests and step-by-step deployment.

---

## Architecture & Scheduling

The llm-d Router schedules multimodal requests using prefix cache affinity and server load metrics.

> [!NOTE]
> For the high-level scheduling architecture flow and EPP load-balancing diagrams, see the [Optimized Baseline guide](optimized-baseline.md#architecture).


### Prefix-Aware Scheduling (Multimodal)

EPP maintains a view of each endpoints' prefix-cache state in memory. When a request arrives, it identifies which pod already holds the matching prefix in KV-cache and routes the request there. 

For multimodal workloads, EPP guesses the size of multimodal placeholders in the prompt and folds visual/auditory asset signatures (e.g., image asset hashes) directly into EPP's block-key hashing chain.

#### 1. Why Multimodal Prefix-Cache Aware Routing?
* **Massive Compute Reduction:** Reusing the KV cache skips vision encoder and token prefill execution, saving highly resource-intensive GPU computations.
* **Improved Time-To-First-Token (TTFT):** Routing requests with repeated visual contexts (e.g., shared reference templates, system prompt assets, or product photos) to the correct worker pod yields dramatic improvements in TTFT.

#### 2. Text Hashing vs. Multimodal Alignment Mechanics

To identify prefix matches without tokenizing prompts, EPP chunks incoming requests and hashes them sequentially:
* **Character-Based Blocks:** Text is segmented into static blocks of a configured character size, completely bypassing tokenization.
* **Chained Prefix Hashing:** Blocks are hashed sequentially. Each block's hash integrates the hash of its predecessor:
  $$Hash(Chunk_i) = Hash(Content_i + Hash(Chunk_{i-1}))$$
* **Historical Sequence Verification:** This chained structure guarantees prefix integrity: if a backend server matches Chunk $X$, it is mathematically guaranteed to hold matches for all preceding chunks ($0$ through $X-1$).

However, when multimodal assets are mixed into the text stream, this character-based model breaks down. Because EPP cannot natively predict how many tokens an image, video, or audio asset will yield, EPP's chunk boundaries become completely misaligned with the physical memory blocks allocated on the model server. This misalignment introduces significant error:
1. **Match Score Distortion:** The prefix match score—critical for scheduling decisions—becomes highly skewed.
2. **LRU State Drift:** The proxy's simulated LRU eviction queue drifts rapidly from the actual cache state of the backends, resulting in routing false positives.

To resolve this boundary misalignment, EPP mathematically estimates ("guesses") the virtual token footprint of each multimodal asset before performing character chunking. EPP uses two highly customizable **Token Estimation Strategies**:

##### A. Dimension-Based Approximation (e.g., Qwen-VL)
Estimate tokens based on image width and height:
$$\text{Tokens} = \frac{\text{Image Width} \times \text{Image Height}}{\text{Factor}} + 2$$

* The `Factor` parameter is configurable per-EPP:
  * For **Qwen 2.5 VL**: `factor = 784` (which is $28 \times 28$)
  * For **Qwen 3.5 VL**: `factor = 1024` (which is $32 \times 32$)

##### B. Configuration-Based Fixed Allocation (e.g., Gemma 4)
Directly use fixed values from user configuration matching the model's support levels:
* Gemma 4 supported values: 70, 140, 280 (default), 560, or 1120 tokens per image.



---

### Load-Aware Scheduling

EPP continuously probes each endpoints' metrics by scraping `/metrics` at a regular interval (50ms default). It scores endpoints on queue depth, running requests, and KV-cache utilization to schedule requests to the endpoint with the lowest load, avoiding hotspots caused by heterogeneous request patterns.



---

## Further Reading

* See [Optimized Baseline](optimized-baseline.md) for details on text-based scheduling and general load-balancing.
* See [EPP Architecture](../architecture/core/router/epp/README.md) for more details.
* See [KV-Cache Indexer](../advanced/kv-management/kv-indexer.md) for details on precise event-driven indexing.
