---
title: "How LLM Serving Scales to Millions of Users: The Memory and KV Cache Bottleneck"
description: "How LLM inference scales to millions of users: memory-bound decoding, KV cache sizing, PagedAttention, continuous batching, and OpenAI prompt caching."
keywords:
  - how llm inference works kv cache explained
  - how chatgpt serves millions of users
  - paged attention explained
  - continuous batching llm
  - prompt caching how it works
  - llm time to first token
  - speculative decoding math
date: 2026-10-31
last_verified: 2026-10-31
---

# How LLM Serving Scales to Millions of Users: The Memory and KV Cache Bottleneck

> [!IMPORTANT]
> **Status Update: Architecture, Model Lineup, and Disclosed Serving Specs (October 2026)**  
> OpenAI operates the world’s largest commercial LLM serving infrastructure, supporting over **250M to 300M weekly active users** and billions of daily API requests across Microsoft Azure and Oracle Cloud GPU clusters [1, 5].
> - **Model Lineup & Architecture Insights**: OpenAI’s commercial models include the flagship multimodal **`gpt-4o` / `gpt-4o-mini`** series, the **`o1` / `o3`** reasoning models (which generate hidden chain-of-thought tokens prior to output), and the open-weight **`gpt-oss`** series (`gpt-oss-120b` and `gpt-oss-20b`). Architectural disclosures from `gpt-oss` confirm that OpenAI utilizes **Grouped-Query Attention (GQA)** with 64 query heads and **8 Key-Value (KV) heads** ($d_k = 64$, 8:1 ratio), alternating dense and 128-token banded window attention layers, and learned attention sinks [2].
> - **Prompt Caching Rules & Pricing**: OpenAI enforces an automated prompt-caching threshold: prompts with prefix lengths $\ge \mathbf{1{,}024\text{ tokens}}$ automatically qualify for a **50% discount on input token pricing** (evaluated in 128-token increments), with typical in-memory retention of 5–10 minutes of idle time (up to 1 hour max, strictly isolated by organization boundary) [1].
> - **Streaming Transports Disclosed**: Standard text streaming in ChatGPT and the Chat Completions API uses **Server-Sent Events (SSE)** over HTTP/2 (`text/event-stream`), while the multimodal **Realtime API** supports **WebRTC** (browser clients) and **WebSockets** (telephony and server-to-server) [1].
> - **OpenAI Has Not Disclosed Its Proprietary Serving Engine**: OpenAI does not publish internal serving architecture papers. Core techniques such as **PagedAttention** (vLLM / UC Berkeley), **continuous batching** (Orca / OSDI 2022), and **prefill-decode disaggregation** (Splitwise / DistServe) are public industry breakthroughs that explain how high-scale LLM inference is solved mathematically [3, 4].

> [!NOTE]
> **TL;DR**
> - **The Decode Bottleneck**: LLM generation is fundamentally **memory-bandwidth-bound**, not compute-bound. At batch size $B=1$, generating a single token requires streaming all model weight tensors from High Bandwidth Memory (HBM) to on-chip SRAM to execute a simple vector-matrix multiplication, achieving less than 1% of the GPU's peak compute capacity [derived].
> - **The KV Cache Memory Wall**: Autoregressive decoding stores past Key and Value attention tensors in GPU memory to avoid recomputing history. For an 80-layer model using GQA, the KV cache consumes **327.68 KB per token**; at 128k context, a single user's KV cache requires **40.0 GB of VRAM**, consuming half of an 80 GB GPU [derived].
> - **Continuous Batching (Orca)**: Traditional static batching freezes an entire batch until the longest sequence completes. Continuous (iteration-level) batching swaps completed sequences out of the active batch at every single token step, multiplying server throughput by $2\times$ to $5\times$ [4].
> - **PagedAttention (vLLM)**: Traditional contiguous memory pre-allocation causes **60% to 80% memory waste** from internal and external fragmentation. Allocating KV cache in fixed-size paged blocks (e.g., 16 tokens/block) slashes memory waste to **$< 4\%$**, doubling concurrent batch capacity [3].
> - **Prompt Caching ROI**: Caching a 3,000-token system prompt prefix reduces Time-to-First-Token (TTFT) by **$\approx 3\times\text{ to }5\times$** while cutting input costs by **$46.9\%$**, requiring strict prefix matching because causal self-attention propagates changes to all downstream keys [derived].

In traditional web applications, scaling from 10,000 to 10 million users is an engineering challenge solved by adding stateless web containers behind a load balancer and caching hot rows in Redis. In Large Language Model serving, scaling breaks traditional distributed assumptions: models do not fit into standard CPU memory, requests cannot share state without memory fragmentation, and generating output requires streaming gigabytes of model weights from GPU memory for every single character emitted. Serving millions of concurrent ChatGPT sessions requires treating High Bandwidth Memory (HBM) as the scarcest commodity in computing.

---

## Production Scale and Operational Realities

Serving frontier LLMs across hundreds of millions of weekly active users imposes extreme hardware and latency boundaries.

| Metric / Dimension | Production Target / Observed Value | Source / Classification |
| :--- | :--- | :--- |
| **Weekly Active Users** | 250M – 300M+ active users | OpenAI Official Announcements [1, 5] |
| **Prompt Caching Min Prefix** | 1,024 tokens (128-token increments) | OpenAI Official Documentation [1] |
| **Prompt Caching Discount** | 50% discount on cached input tokens | OpenAI Pricing Specification [1] |
| **Cache Idle Retention** | 5 – 10 minutes (up to 1 hour maximum) | OpenAI API Documentation [1] |
| **Text Streaming Protocol** | Server-Sent Events (SSE) over HTTP/2 | OpenAI Chat API Specification [1] |
| **Realtime Speech Protocol** | WebRTC (browser) & WebSockets (server) | OpenAI Realtime API Docs [1] |
| **Open-Weight Attention (gpt-oss)** | GQA: 64 Query Heads, 8 KV Heads ($d_k=64$) | OpenAI gpt-oss Model Card [2] |
| **GPU Cluster Infrastructure** | Microsoft Azure NDv4/NDv5 (A100/H100 + InfiniBand) | Microsoft & OpenAI Public Filings [5] |
| **Paged Memory Waste Reduction** | 60%–80% waste reduced to $< 4\%$ | SOSP 2023 vLLM Paper [3] |
| **Continuous Batching Gain** | 2x – 5x throughput vs static batching | OSDI 2022 Orca Paper [4] |

---

## Architectural Separation: Confirmed, Reported, and Generic Designs

To understand large-scale LLM inference, we must separate confirmed OpenAI features from industry-standard serving techniques:

```mermaid
flowchart TD
    subgraph ClientTier ["1. Client & Ingress Tier [Confirmed]"]
        User["User / API Client"]
        Gateway["API Gateway & Reverse Proxy\n[Rate Limiting, SSE Transport, Auth]"]
    end

    subgraph RoutingTier ["2. Prefix-Aware Routing Tier [Generic Pattern]"]
        Router["Prefix-Aware Router / Load Balancer\n[Hash-Based Affinity for Reused Prompts]"]
    end

    subgraph DisaggregatedServing ["3. Disaggregated Serving Clusters [Generic Pattern]"]
        subgraph PrefillCluster ["Prefill Nodes (Compute-Bound / GEMM)"]
            PrefillEngine["Prefill Worker Pool (Tensor Parallel)\n[Prompt Encoding, FlashAttention-2, Cache Insertion]"]
        end
        
        subgraph DecodeCluster ["Decode Nodes (Memory-Bound / GEMV)"]
            DecodeEngine["Continuous Batching Decode Engine\n[Paged KV Cache, Speculative Draft Model]"]
            PagedKVCache[("Paged KV Cache RAM Blocks\n(< 4% Fragmentation)")]
        end
    end

    subgraph ModelArchitecture ["4. Transformer & Attention Model [Confirmed / gpt-oss]"]
        AttentionGQA["Grouped-Query Attention (GQA)\n[64 Q-Heads : 8 KV-Heads, YaRN 128k]"]
    end

    User -->|1. HTTPS Request| Gateway
    Gateway -->|2. Forward Prompt| Router
    Router -->|3. Route to Cached Node| PrefillEngine
    PrefillEngine <--> AttentionGQA
    PrefillEngine -->|4. Transfer KV Blocks (RDMA)| DecodeEngine
    DecodeEngine <--> PagedKVCache
    DecodeEngine -->|5. Token-by-Token SSE Stream| Gateway
    Gateway -->|6. data: JSON stream| User

    classDef confirmed fill:#1e3a5f,stroke:#4a90e2,stroke-width:2px,color:#ffffff;
    classDef generic fill:#2d4a3e,stroke:#27ae60,stroke-width:2px,color:#ffffff;

    class User,Gateway,AttentionGQA confirmed;
    class Router,PrefillEngine,DecodeEngine,PagedKVCache generic;
```

### Architectural Walkthrough (5 Core Stages):
1. **Client Ingress & SSE Transport (Confirmed)**: Clients initiate requests via HTTP POST. For streaming completions (`stream: true`), the API gateway opens a persistent HTTP/2 connection streaming tokens as Server-Sent Events (`text/event-stream`), terminating on a `data: [DONE]` frame [1].
2. **Prefix-Aware Routing (Generic Pattern)**: Incoming requests inspect the prompt hash. If the first 1,024+ tokens match a known system prompt prefix, the router dispatches the request to the specific GPU node currently holding those KV cache blocks in local VRAM [1].
3. **Compute-Bound Prefill Phase (Generic Pattern)**: During the prefill (prompt processing) phase, all prompt tokens are processed simultaneously in a single, highly parallel Matrix-Matrix Multiplication (GEMM) using FlashAttention, computing the initial Key and Value activations at high arithmetic intensity [3, 4].
4. **Paged Memory Allocation (Generic Pattern)**: As tokens are generated, the inference engine allocates physical memory in fixed-size blocks (PagedAttention) tracked via a software page table, eliminating contiguous virtual memory allocation and cutting fragmentation waste from ~70% to $<4\%$ [3].
5. **Memory-Bound Continuous Decode (Generic Pattern)**: The decode engine runs iterative single-token forward passes using continuous batching. Weight tensors are streamed once from HBM to process all active requests in the batch, while speculative draft models propose candidate tokens to accelerate generation speed [4].

---

## Core Mathematics of LLM Inference & Memory

### 1. KV Cache Memory Size Derivation

In autoregressive generation, each generated token attends to all previous tokens. To avoid recalculating past key and value vectors at every step, past vectors are cached in GPU memory:

$$\text{KV Cache Bytes per Token} = 2 \times L \times H_{\text{KV}} \times d_k \times P$$

where:
- Factor of $2$: Key tensor ($K$) and Value tensor ($V$).
- $L$: Number of transformer layers.
- $H_{\text{KV}}$: Number of Key-Value attention heads.
- $d_k$: Dimension per head ($d_k = \frac{d_{\text{model}}}{H_Q}$).
- $P$: Precision in bytes per element (2 bytes for FP16/BF16, 1 byte for FP8).

#### Case Study: Standard Open Model Configuration (Llama 3 70B Benchmark)
Configuration parameters: $L = 80$, $H_Q = 64$, $H_{\text{KV}} = 8$ (Grouped-Query Attention, 8:1 ratio), $d_k = 128$, Precision $P = 2\text{ bytes (FP16)}$:

$$\text{Bytes / Token} = 2 \times 80 \times 8 \times 128 \times 2 = 327{,}680 \text{ bytes / token} \approx 327.68 \text{ KB / token (derived)}$$

#### Scaling Across Context Windows:
- **At $4\text{k}$ Context ($4{,}096\text{ tokens}$)**:
  $$\text{KV Size}_{4k} = 4{,}096 \times 327{,}680 = 1{,}342{,}177{,}280 \text{ bytes} \approx 1.25 \text{ GB (1.34 GB)}$$
- **At $32\text{k}$ Context ($32{,}768\text{ tokens}$)**:
  $$\text{KV Size}_{32k} = 32{,}768 \times 327{,}680 = 10{,}737{,}418{,}240 \text{ bytes} \approx 10.0 \text{ GB (10.74 GB)}$$
- **At $128\text{k}$ Context ($131{,}072\text{ tokens}$)**:
  $$\text{KV Size}_{128k} = 131{,}072 \times 327{,}680 = 42{,}949{,}672{,}960 \text{ bytes} \approx 40.0 \text{ GB (42.95 GB)}$$

#### Attention Architecture Comparison ($L=80, d_k=128, P=2$ at $128\text{k}$ Context):
- **Multi-Head Attention (MHA)** ($H_{\text{KV}} = 64$ heads):
  $$\text{Bytes/token} = 2 \times 80 \times 64 \times 128 \times 2 = 2{,}621{,}440 \text{ bytes/token} \approx 2.62 \text{ MB/token}$$
  $$\text{At } 128\text{k context}: 131{,}072 \times 2.621\text{ MB} \approx \mathbf{320.0 \text{ GB per user sequence!}}$$
- **Grouped-Query Attention (GQA)** ($H_{\text{KV}} = 8$ heads):
  $$\text{Bytes/token} = 327.68 \text{ KB/token} \implies \mathbf{40.0 \text{ GB at } 128\text{k context (8x smaller than MHA)}}$$
- **Multi-Head Latent Attention (MLA)** ($d_c = 512, d_R = 64 \implies 576\text{ elements/layer}$):
  $$\text{Bytes/token} = 80 \times 576 \times 2 = 92{,}160 \text{ bytes/token} \approx 92.16 \text{ KB/token} \implies \mathbf{11.25 \text{ GB at } 128\text{k context (28.4x smaller)}}$$

---

### 2. GPU Memory Budget and Concurrency Limits

Consider a standard enterprise server equipped with an **80 GB GPU** (NVIDIA H100 SXM, $81{,}920\text{ MB}$ total physical VRAM).

$$\text{Available VRAM for KV Cache} = \text{Total VRAM} - M_{\text{weights}} - M_{\text{activations}} - M_{\text{CUDA}}$$

Assume a model partitioned across GPUs such that:
- $M_{\text{weights}} = 40.0\text{ GB}$ (FP8 weights).
- $M_{\text{activations+CUDA}} = 10.0\text{ GB}$.

$$\text{Available KV Space} = 80 - 40 - 10 = 30.0 \text{ GB}$$

#### Maximum Concurrent Active Sequences ($C$):
$$C = \left\lfloor \frac{\text{Available KV Space}}{\text{KV Size per Sequence}} \right\rfloor$$

- **At $4\text{k}$ Context ($1.25\text{ GB/seq}$)**:
  $$C_{4k} = \frac{30.0\text{ GB}}{1.25\text{ GB}} = 24 \text{ concurrent streams}$$
- **At $32\text{k}$ Context ($10.0\text{ GB/seq}$)**:
  $$C_{32k} = \frac{30.0\text{ GB}}{10.0\text{ GB}} = 3 \text{ concurrent streams}$$
- **At $128\text{k}$ Context ($40.0\text{ GB/seq}$)**:
  $$C_{128k} = \frac{30.0\text{ GB}}{40.0\text{ GB}} = 0.75 \implies \text{A single sequence cannot fit on one GPU!}$$

This mathematical derivation explains why serving long-context prompts ($>32\text{k}$) slashes server concurrency by **$87.5\%$**, forcing inference engines to distribute context across multiple GPUs via tensor parallelism or pipeline parallelism.

---

### 3. Memory-Bound Decoding: The Roofline Model

In deep learning hardware, execution speed is governed by the **Roofline Model**:

$$\text{Attainable Performance (TFLOPS)} = \min \left( \text{Peak Compute}, \text{Arithmetic Intensity} \times \text{Memory Bandwidth} \right)$$

#### Single-Sequence Generation ($B = 1$):
When generating 1 token for a single user, the GPU must load the entire model weight tensor ($W = 70\text{ Billion parameters} \times 1\text{ byte (FP8)} = 70\text{ GB}$) from HBM to SRAM to execute a single token forward pass:

$$\text{Arithmetic Intensity} = \frac{2 \times W \text{ FLOPs}}{W \text{ Bytes}} = 2 \text{ FLOPs / byte}$$

On an NVIDIA H100 SXM (Memory Bandwidth = $3{,}350\text{ GB/s}$, Peak FP8 Tensor Core Compute = $1{,}979\text{ TFLOPS}$):

$$\text{Attainable Compute} = 2 \text{ FLOPs/byte} \times 3{,}350 \times 10^9 \text{ bytes/s} = 6.7 \text{ TFLOPS}$$

$$\text{Compute Utilization} = \frac{6.7}{1{,}979} = 0.00338 = 0.34\% \text{ of GPU capacity!}$$

At $B=1$, the GPU's tensor cores sit starved **$99.66\%$ of the time**, waiting for memory transfers from HBM.

#### The Effect of Continuous Batching ($B = 64$):
When batching $B = 64$ active users together:
- Model weights ($W$) are loaded from HBM **once** to compute tokens for all 64 sequences simultaneously.
- Arithmetic intensity increases $64\times$:
  $$\text{Arithmetic Intensity} \approx 2 \times 64 = 128 \text{ FLOPs / byte}$$
- Compute throughput rises to:
  $$128 \times 3{,}350 \times 10^9 \approx 428.8 \text{ TFLOPS (21.7% hardware utilization)}$$

However, while model weights are shared across the batch, **each user's unique KV cache must still be read independently from HBM**. As batch size grows, KV cache memory reads gradually overtake model weights as the dominant HBM bandwidth consumer [derived].

---

### 4. Memory Fragmentation: Contiguous Pre-allocation vs. PagedAttention

Traditional serving engines allocate memory contiguously using standard CUDA memory allocators.

#### The Three Sources of Memory Waste in Contiguous Allocation:
1. **Reservation Waste**: The system pre-allocates memory for the maximum possible length ($L_{\text{max}} = 4{,}096\text{ tokens}$). If the user's conversation only reaches $800\text{ tokens}$, the remaining $3{,}296\text{ tokens}$ ($80.5\%$ of allocated memory) sits completely unused.
2. **Internal Fragmentation**: Allocating fixed-size power-of-two memory chunks causes memory to be reserved beyond actual sequence needs.
3. **External Fragmentation**: Sequences finish at different times. When shorter sequences terminate, they leave discontiguous memory holes that cannot satisfy new, longer incoming requests.

> [!NOTE]
> **Verified vLLM Benchmark**:  
> In the SOSP 2023 paper (*Efficient Memory Management for Large Language Model Serving with PagedAttention*), Kwon et al. demonstrated that traditional contiguous serving systems waste **60% to 80%** of total GPU memory on fragmentation and reservation overhead [3].

#### The PagedAttention Solution:
PagedAttention partitions physical GPU memory into small, fixed-size blocks (e.g., $16\text{ tokens / block}$), analogous to virtual memory paging in operating systems:
- Memory is allocated non-contiguously.
- Physical blocks are mapped to logical sequences via a software page table.
- Only the very last block of an active sequence suffers internal fragmentation (at most $15\text{ tokens}$):

$$\text{Memory Waste (PagedAttention)} \le \frac{15 \text{ tokens}}{L_{\text{sequence}}} < 4\% \text{ waste [3]}$$

Cutting memory waste from $70\%$ to $<4\%$ immediately doubles the number of concurrent sequences that fit into physical VRAM, doubling throughput without adding extra GPUs.

---

### 5. Prefix Caching Mathematics & OpenAI Pricing

In conversational chat and agent workflows, thousands of requests share an identical system prompt, developer instructions, or RAG reference documents.

#### Worked Example: Reusing a 3,000-Token System Prompt
Suppose an application defines a $3{,}000\text{-token}$ system prompt and processes $1{,}000{,}000\text{ API requests}$ per month, where each request appends a $200\text{-token}$ user query.

#### Standard Uncached Cost (`gpt-4o` base rate $2.50\text{ per } 1\text{M input tokens}$ [1]):
$$\text{Cost}_{\text{uncached}} = 1{,}000{,}000 \times 3{,}200 \text{ tokens} \times \frac{\$2.50}{10^6} = \$8{,}000.00$$

#### Prompt Caching Cost (OpenAI 50% cached discount $\implies \$1.25\text{ per } 1\text{M tokens}$ [1]):
- Initial prefill (1st request): $3{,}200 \times \frac{\$2.50}{10^6} \approx \$0.008$.
- Remaining $999{,}999$ requests (cached prefix $3{,}000$ tokens at $\$1.25$, user query $200$ tokens at $\$2.50$):
  $$\text{Cost}_{\text{per\_req}} = \left( 3{,}000 \times \frac{\$1.25}{10^6} \right) + \left( 200 \times \frac{\$2.50}{10^6} \right) = \$0.00375 + \$0.00050 = \$0.00425$$
  $$\text{Total Cost}_{\text{cached}} = 999{,}999 \times \$0.00425 + \$0.008 \approx \$4{,}250.00$$
  $$\text{Net Financial Savings} = \$8{,}000.00 - \$4{,}250.00 = \$3{,}750.00 \quad \mathbf{(46.9\% \text{ cost reduction!})}$$

#### Time-to-First-Token (TTFT) Speedup:
- Prefilling $3{,}000\text{ tokens}$ uncached takes ~150ms–250ms of GPU compute.
- With Prompt Caching, pre-computed $K$ and $V$ tensors are read directly from VRAM, slashing TTFT to $<50\text{ms}$ (an **$\approx 3\times\text{ to }5\times$ TTFT acceleration**) [derived].

#### Why Exact Prefix Matching is Mandatory:
Because transformer self-attention is causal, token $i$ attends to all previous tokens $j \le i$. Changing a single comma at token position 5 alters the attention representation for every subsequent token $k \ge 5$. Therefore, prompt caches can only match **exact sequential prefixes from token index 0**.

---

### 6. Reference-Counted LRU Prefix Eviction Logic

When GPU memory reaches capacity, the serving engine must evict cached prefix blocks. The following pseudo-code illustrates a reference-counted Least-Recently-Used (LRU) block eviction manager:

```python
# Reference-Counted Paged Block Cache & LRU Eviction (Generic Pattern)
from collections import OrderedDict
import time

class PagedBlock:
    def __init__(self, block_id, tokens):
        self.block_id = block_id
        self.tokens = tokens
        self.ref_count = 0  # Number of active inference streams currently reading this block
        self.last_accessed = time.time()

class PrefixCacheManager:
    def __init__(self, total_blocks):
        self.total_blocks = total_blocks
        self.allocated_blocks = {}
        # LRU queue tracks blocks with ref_count == 0 (eligible for eviction)
        self.lru_eviction_queue = OrderedDict()

    def access_block(self, block_id):
        block = self.allocated_blocks[block_id]
        block.ref_count += 1
        block.last_accessed = time.time()
        # If block was in eviction queue, remove it (it is now actively in use)
        self.lru_eviction_queue.pop(block_id, None)

    def release_block(self, block_id):
        block = self.allocated_blocks[block_id]
        block.ref_count -= 1
        if block.ref_count == 0:
            # Active request finished: place block in LRU eviction queue
            self.lru_eviction_queue[block_id] = block.last_accessed

    def evict_least_recently_used(self):
        if not self.lru_eviction_queue:
            raise MemoryError("Out of VRAM: All allocated blocks are actively in use!")
        # Pop the oldest idle block (first in OrderedDict)
        evict_id, _ = self.lru_eviction_queue.popitem(last=False)
        del self.allocated_blocks[evict_id]
        return evict_id  # Free physical VRAM block
```

---

### 7. Speculative Decoding Mathematics

Speculative decoding employs a small draft model (e.g., a 1B parameter model) to generate $k$ candidate tokens rapidly, followed by a single parallel forward pass of the large target model (e.g., 70B) to verify them.

Let:
- $\alpha$: Acceptance probability (average probability that the target model accepts a draft token, typically $0.65 \le \alpha \le 0.85$).
- $k$: Number of speculative draft tokens generated per iteration.

The expected number of accepted tokens $\mathbb{E}[\text{tokens}]$ per step is:

$$\mathbb{E}[\text{tokens}] = \frac{1 - \alpha^{k+1}}{1 - \alpha}$$

#### Worked Example:
Assume draft length $k = 4$ tokens, with acceptance rate $\alpha = 0.75$:

$$\mathbb{E}[\text{tokens}] = \frac{1 - 0.75^5}{1 - 0.75} = \frac{1 - 0.2373}{0.25} = \frac{0.7627}{0.25} = 3.05 \text{ tokens / step (derived)}$$

Instead of executing 3 slow sequential forward passes of the 70B target model, the system generates **3.05 tokens in a single target model forward pass**, achieving an **$\approx 3\times$ decode speedup** without changing the output probability distribution.

---

### 8. Capacity Math via Little's Law

Suppose an LLM cluster handles an average incoming load of $\lambda = 4{,}000\text{ requests / second}$.
Average generation time per request is $W = 3.0\text{ seconds}$ (generating ~120 tokens at 40 tok/s).

By **Little's Law** ($L = \lambda \cdot W$):

$$C = 4{,}000 \times 3.0 = 12{,}000 \text{ concurrent active decoding sequences}$$

If an enterprise 80 GB GPU node (running optimized GQA and continuous batching) sustains $B_{\text{GPU}} = 48\text{ concurrent sequences}$ without violating p99 latency SLAs:

$$\text{GPUs Required} = \frac{C}{B_{\text{GPU}}} = \frac{12{,}000}{48} = 250 \text{ physical GPUs (approx. 32 8-GPU nodes) (derived)}$$

---

## Token Streaming: SSE vs. WebSockets and Latency Metrics

Understanding streaming latency requires distinguishing between transport protocols and latency metrics.

### 1. Transport Protocol Comparison
- **Server-Sent Events (SSE)**: Standard for text completions in ChatGPT and OpenAI Chat API [1]. Operates as a unidirectional stream over standard HTTP/2 (`text/event-stream`). Lightweight, traverses firewalls effortlessly, and supports simple reconnection.
- **WebSockets & WebRTC**: Deployed for the OpenAI Realtime API [1]. Bidirectional communication is mandatory because audio streams must flow both ways simultaneously, and client interruptions (barge-in) require sending upstream control signals to halt model output.

### 2. Deconstructing the Latency Metrics
- **Time-to-First-Token (TTFT)**: Measures the duration from when the client submits the prompt until the first token appears on screen. **TTFT includes the entire prefill phase** (processing the prompt) plus network transit.
- **Time-per-Output-Token (TPOT)**: Measures the time required to generate each subsequent token during the decode phase. For interactive chat, humans read at ~5–8 words/second (~10–15 tokens/sec). A TPOT of $25\text{ ms}$ ($40\text{ tokens/sec}$) feels instantaneous to human readers.

---

## Production Post-Mortems: Real-World Outages & Infrastructure Failures

### Post-Mortem 1: The March 20, 2023 Redis Async Connection Leak
* **The Incident**: On March 20, 2023, OpenAI took ChatGPT offline after users reported seeing other users' conversation titles in their chat sidebar [5].
* **The Root Cause (Application Infrastructure Bug)**:
  - OpenAI used Redis clusters to store conversation history and user session data, utilizing the open-source `redis-py` async connection pool library.
  - A bug in `redis-py` caused cancelled async requests to leave unread responses inside the pooled connection's internal buffer.
  - When the connection was returned to the pool and reused by a completely different request, the second request read the leftover unread response meant for the previous user.
* **The Impact & Blast Radius**:
  - For 1.2% of active ChatGPT Plus users during a 9-hour window, payment confirmation emails revealed other users' first and last names, email addresses, payment addresses, and the last four digits of their credit cards [5].
* **The Remediation**:
  - OpenAI patched the async Redis library, overhauled connection pool validation to discard dirty buffers, and decoupled sensitive payment data from session caching layers.

---

### Post-Mortem 2: The December 11, 2024 Kubernetes Telemetry Saturation Outage
* **The Incident**: On December 11, 2024, ChatGPT, the OpenAI API, and Sora suffered a global 4-hour-and-22-minute outage [5].
* **The Root Cause (Control Plane Saturation)**:
  - OpenAI deployed a new cluster-wide telemetry service to collect detailed Kubernetes control plane metrics.
  - Due to an expansive query pattern, thousands of GPU worker nodes simultaneously executed heavy API queries against `kube-apiserver`.
  - The API servers collapsed under the query storm. Because internal data plane service discovery relied on DNS resolution tied to the control plane, data plane routing failed, knocking down model inference gateways [5].
* **The Remediation**:
  - Mandated progressive, staged canary rollouts for cluster telemetry, decoupled data plane DNS from control plane state, and established out-of-band administrative access to allow engineers to manage nodes during control plane outages [5].

---

## Architectural Trade-Off Matrix

| Strategy | Trade-Off Made | What Was Gained | What Was Sacrificed |
| :--- | :--- | :--- | :--- |
| **Continuous Batching** | Iteration-level sequence scheduling at every token step. | 2x to 5x higher server throughput; zero static padding waste. | Higher scheduling overhead and unpredictable per-token latency jitter. |
| **PagedAttention** | Non-contiguous memory allocation in fixed 16-token blocks. | Slashes memory waste from ~70% to $<4\%$; doubles active batch size. | Requires managing virtual-to-physical software page tables in CUDA kernels. |
| **Prefix / Prompt Caching** | Reuses KV cache for matching prompt prefixes $\ge 1,024$ tokens. | 3x–5x TTFT acceleration; 50% discount on cached input costs. | Memory lock-in: cached prefixes hold VRAM blocks that cannot be used for decoding. |
| **Speculative Decoding** | Runs a small draft model to generate $k$ candidate tokens. | 2x to 3x faster wall-clock generation speed. | Wasted compute when draft tokens are rejected; requires serving two models simultaneously. |

---

## When NOT to Copy This Architecture

Engineers frequently attempt to deploy complex self-hosted vLLM or TensorRT-LLM clusters when managed APIs are far superior:

```
                            [Daily Request Volume & Team Skills?]
                                             │
                       ┌─────────────────────┴─────────────────────┐
                       ▼                                           ▼
                 < 50,000 req / day                          > 1,000,000 req / day
                       │                                           │
         ┌─────────────┴─────────────┐                             │
         ▼                           ▼                             ▼
[Use Managed Hosted API]      [Simple Edge Serving]      [Dedicated GPU Engineering Ops?]
OpenAI / Anthropic API        Ollama / vLLM single GPU              │
Zero GPU Ops Overhead         Local Prototyping              ┌─────┴─────┐
Pay Only for Tokens Used                                     ▼           ▼
                                                            No          Yes
                                                             │           │
                                                             ▼           ▼
                                                        Managed      Self-Hosted
                                                        Cloud GPU    vLLM / TensorRT
                                                        (Anyscale,   Cluster + Paged
                                                         Together)   KV Cache Engine
```

### 1. Applications with Low Request Volumes ($< 50{,}000\text{ req/day}$)
If your product handles modest traffic, paying $\$2.50\text{ per }1\text{M tokens}$ on a commercial API is drastically cheaper than renting an 8-GPU H100 node ($\approx \$25\text{–}\$35/\text{hour} \approx \$20{,}000/\text{month}$). Managed APIs eliminate GPU idle waste entirely.

### 2. Offline Batch Processing (Evaluations, Data Extraction)
In batch workloads where users are not waiting for interactive streaming tokens, optimizing Time-to-First-Token (TTFT) and speculative decoding is wasted effort. Maximize throughput using massive batch sizes and offline queue workers instead.

### 3. Engineering Teams Without Dedicated GPU Systems Operators
Operating continuous batching, prefix-aware load balancers, and multi-node tensor parallel clusters requires deep expertise in CUDA kernel compilation, NCCL collective communications, and RoCE/InfiniBand network tuning. Teams without dedicated infra engineers should use managed inference platforms (Together AI, Fireworks, Anyscale).

---

## Frequently Asked Questions

### Does OpenAI use PagedAttention in its production infrastructure?
PagedAttention is an open-source academic technique developed by UC Berkeley (vLLM, SOSP 2023) [3]. While OpenAI has never confirmed whether it uses PagedAttention or a proprietary internal equivalent, all high-scale LLM serving systems mathematically require block-paged KV memory management to eliminate the 60%–80% memory fragmentation waste of contiguous allocation [3].

### Why is prompt caching restricted to prefixes $\ge 1,024$ tokens?
Managing prompt cache indices, software page tables, and cache routing introduces computational and memory overhead. For small prompts ($<1,024$ tokens), the overhead of cache lookup, memory locking, and routing exceeds the marginal compute saved by skipping prefill [1].

### What is the difference between OpenAI Triton and NVIDIA Triton?
**OpenAI Triton** is an open-source Python-like programming language and compiler created at OpenAI by Philippe Tillet to write high-performance custom GPU kernels (e.g., FlashAttention kernels) without writing raw CUDA C++ [2]. **NVIDIA Triton Inference Server** is a commercial, multi-framework model serving software product developed by NVIDIA. They are completely unrelated tools sharing a common name.

### Why does long-context input reduce concurrent server capacity?
For an 80-layer model using GQA, the KV cache consumes $327.68\text{ KB per token}$ [derived]. At 4k context, a sequence requires $1.25\text{ GB}$; at 128k context, it requires $40.0\text{ GB}$. Because physical GPU memory is strictly bounded (80 GB), serving one 128k-token sequence consumes the same memory that could otherwise serve 32 concurrent 4k-token sequences [derived].

### What is the difference between prefill and decode phases?
**Prefill** processes the input prompt all at once using Matrix-Matrix Multiplication (GEMM) at high arithmetic intensity, making it compute-bound [4]. **Decode** generates output tokens one by one autoregressively using Matrix-Vector Multiplication (GEMV), requiring the entire model weight tensor to be loaded from HBM for each token, making it memory-bandwidth-bound [derived].

---

## References

1. OpenAI Documentation. (2024–2026). "Prompt Caching Guide, Streaming API, and Realtime API Transport Specifications." *Official OpenAI Platform Docs*. Available at: https://platform.openai.com/docs
2. OpenAI Open-Weight Team. (2025–2026). "gpt-oss Model Card & Technical Specifications: Architecture, GQA, and Alternating Attention." *OpenAI GitHub Repository*.
3. Kwon, W., et al. (2023). "Efficient Memory Management for Large Language Model Serving with PagedAttention." *Proceedings of the 29th ACM Symposium on Operating Systems Principles (SOSP '23)*. Available at: https://arxiv.org/abs/2309.06180
4. Yu, G., et al. (2022). "Orca: A Distributed Serving System for Transformer-Based Generative Models." *Proceedings of the 16th USENIX Symposium on Operating Systems Design and Implementation (OSDI '22)*.
5. OpenAI Incident Disclosures & Status Reports. (2023–2025). "March 20, 2023 Redis Connection Bug & December 11, 2024 Kubernetes Telemetry Outage Post-Mortems." *OpenAI Official Status Page*.
6. Leviathan, Y., Kalman, M., & Matias, Y. (2023). "Fast Inference from Transformers via Speculative Decoding." *International Conference on Machine Learning (ICML '23)*. Available at: https://arxiv.org/abs/2211.17192

---

Authored by @saurabh-vision
