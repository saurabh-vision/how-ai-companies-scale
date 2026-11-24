---
title: "Groq LPU Architecture Explained: SRAM-Only Serving and the Memory Bandwidth Bottleneck"
description: "Groq LPU architecture explained: how on-chip SRAM and deterministic static scheduling eliminate the GPU memory wall to achieve ultra-fast LLM decoding."
keywords:
  - groq lpu architecture explained
  - groq vs gpu inference speed
  - llm memory bandwidth bottleneck
  - deterministic inference hardware
  - on-chip sram llm serving
  - tensor streaming processor
  - artificial analysis groq benchmark
date: 2026-10-30
last_verified: 2026-10-30
---

# Groq LPU Architecture Explained: SRAM-Only Serving and the Memory Bandwidth Bottleneck

> [!IMPORTANT]
> **Status Update: Corporate & Architecture Milestones (December 2025 / 2026)**  
> In December 2025, NVIDIA entered into a historic **$20 billion non-exclusive technology licensing and talent agreement** with Groq ($17B licensing payment for Groq's low-latency inference IP and patent portfolio, plus a $3B employee retention pool). Groq founder Jonathan Ross, President Sunny Madra, and core LPU compiler and hardware architects transitioned to NVIDIA to integrate deterministic inference designs into future datacenter platforms. Groq continues to operate as an independent commercial AI inference cloud provider led by CEO Simon Edwards, operating as an official NVIDIA Cloud Partner (NCP) running multi-architecture workloads (combining its legacy LPU clusters with NVIDIA accelerated computing).

> [!NOTE]
> **TL;DR**
> - **The Memory Wall**: During single-stream autoregressive generation ($\text{Batch} = 1$), an NVIDIA H100's Tensor Cores sit idle over 99% of the time (derived: 6.7 TFLOPS realized vs. 1,979 TFLOPS peak), strictly starved by 3.35 TB/s HBM3 memory bandwidth [5].
> - **On-Chip SRAM Capacity**: A single first-generation GroqChip 1 (TSP1) embeds 230 MB of on-chip Static Random-Access Memory (SRAM) with 80 TB/s on-chip memory bandwidth, bypassing external DRAM and HBM entirely [1].
> - **Hardware Footprint per Model**: Storing a 70B parameter model at INT8 precision requires a minimum of 305 chips (derived: $70{,}000\text{ MB} / 230\text{ MB}$); production clusters deploy 576 chips across 8 racks (132.5 GB total SRAM) to accommodate activations, KV caches, and network topologies [3].
> - **Independently Benchmarked Speeds**: Artificial Analysis benchmarks record Groq achieving ~877 tokens/sec on Llama 3 8B, ~610–627 tokens/sec on Llama 3.1 8B, and ~280–305 tokens/sec on Llama 3/3.3 70B in single-stream decoding [4].
> - **Software-Defined Determinism**: The Groq hardware strips out dynamic branch predictors, cache controllers, and runtime bus arbiters; the Groq compiler pre-plans cycle-by-cycle instruction execution and chip-to-chip routing statically at compile time [1, 2].

Autoregressive large language model (LLM) generation operates as a memory-bandwidth-bound workload where every decoded token forces the processor to stream gigabytes of static neural network weights across physical silicon buses. High-end GPUs engineered for massive parallel training suffer severe hardware underutilization during single-user interactive queries, spending hundreds of clock cycles waiting for off-chip High Bandwidth Memory (HBM) to deliver weights to starved tensor execution pipelines. The math dictates that accelerating single-stream inference requires orders of magnitude higher memory bandwidth than external DRAM can physically supply, leading Groq to replace external memory entirely with on-chip SRAM across hundreds of deterministically synchronized chips.

---

## Production Scale and Hardware Parameters

Comparing Groq’s Tensor Streaming Processor (TSP/LPU) architecture against modern data-center GPUs reveals fundamentally divergent optimization targets: single-stream latency versus aggregate batched throughput.

| Hardware / Architectural Dimension | Groq LPU (GroqChip 1 / TSP1) | NVIDIA H100 SXM5 GPU | Source / Evidence |
| :--- | :--- | :--- | :--- |
| **Primary Memory Technology** | On-Chip Embedded SRAM | Off-Chip High Bandwidth Memory (HBM3) | ISCA 2020 [1] / NVIDIA Datasheet [5] |
| **Per-Chip Memory Capacity** | 230 MB | 80 GB | ISCA 2020 [1] / NVIDIA Datasheet [5] |
| **Memory Bandwidth per Chip** | 80.0 TB/s (On-Chip SRAM) | 3.35 TB/s (External HBM3 Bus) | ISCA 2020 [1] / NVIDIA Datasheet [5] |
| **Execution Microarchitecture** | Statically Scheduled / Deterministic | Dynamic Out-of-Order Hardware Schedulers | ISCA 2020 [1] / ISCA 2022 [2] |
| **Hardware Branch Prediction** | None (Compiler Scheduled) | Hardware Branch Predictors & Caches | ISCA 2020 [1] |
| **Cluster Topology for 70B Model** | 576 LPUs (8 Racks $\times$ 9 Servers $\times$ 8 Chips) | 8 GPUs (1 HGX Chassis) | SemiAnalysis Report [3] |
| **Total Memory for 70B Cluster** | 132.5 GB (SRAM across 576 chips) | 640 GB (HBM3 across 8 GPUs) | Derived ($576 \times 230\text{ MB}$) / [5] |
| **Llama 3 8B Single-Stream Decode** | ~877 tokens/sec | ~100–160 tokens/sec (Typical Cloud) | Artificial Analysis Benchmarks [4] |
| **Llama 3 70B Single-Stream Decode** | ~284–305 tokens/sec | ~40–80 tokens/sec (Typical Cloud) | Artificial Analysis Benchmarks [4] |
| **Host System Overhead (70B Cluster)**| 144 CPUs, 144 TB System RAM | 2 CPUs, 1–2 TB System RAM | SemiAnalysis Report [3] |

### The Root Cause: Why Autoregressive Decoding Chokes GPUs

Large language model inference splits into two distinct operational phases:

1. **The Prefill Phase (Compute-Bound)**: When processing the input prompt, all prompt tokens are evaluated concurrently in parallel matrix-matrix multiplications ($\text{GEMM}$). Arithmetic intensity is high:
   $$I = \frac{\text{FLOPs}}{\text{Bytes Transferred}} \gg 50$$
   Here, GPU Tensor Cores are fully saturated, and GPUs operate at peak efficiency.
2. **The Autoregressive Decode Phase (Memory-Bound)**: Generating text occurs one token at a time. To emit a single output token, the model must read all parameters of the neural network from memory into compute registers to multiply against a single activation vector ($\text{GEMV}$). 
   The arithmetic intensity collapses to:
   $$I \approx \frac{2 \times P}{P \times b} = \frac{2}{b} \le 2 \text{ FLOPs/byte}$$
   *(where $P$ is parameter count and $b$ is bytes per parameter)*.

Because modern GPUs possess high theoretical compute capacity ($> 1{,}000\text{ TFLOPS}$) relative to their memory bandwidth ($2–3\text{ TB/s}$), single-stream decoding is starved of data: the arithmetic units finish calculations in a fraction of a nanosecond and spend the remainder of the time stalled waiting for weights to arrive over the memory interposer.

---

## End-to-End System Architecture: GPU HBM vs. Groq LPU SRAM

The fundamental difference lies in memory proximity and scheduling determinism:

```mermaid
flowchart TD
    subgraph GPUArchitecture["NVIDIA H100 Architecture (HBM Memory Bound)"]
        HBM3["High Bandwidth Memory (HBM3)\nCapacity: 80 GB | Bandwidth: 3.35 TB/s"]
        Interposer["Silicon Interposer Bus (High Latency Physical Traverse)"]
        SMArray["Streaming Multiprocessors (SMs)\nDynamic Schedulers / Warp Arbiters"]
        
        HBM3 -->|Off-Chip Memory Transfer| Interposer
        Interposer -->|Wait for Weights (Cores Starved)| SMArray
    end

    subgraph GroqLPUArchitecture["Groq LPU Architecture (SRAM Compute Bound)"]
        SRAMArray["On-Chip Embedded SRAM (230 MB)\nBandwidth: 80 TB/s (Zero External Bus)"]
        ExecutionSlices["Functionally-Sliced Compute Columns\n(MXM Matrix + VXM Vector Units)"]
        
        SRAMArray <-->|Direct Clock-Cycle Silicon Bus| ExecutionSlices
    end

    subgraph GroqMultiChipScaleOut["576-LPU Distributed Cluster (RealScale Mesh)"]
        Rack1["Rack 1: LPUs 1–72 (Layers 1–10)"]
        Rack2["Rack 2: LPUs 73–144 (Layers 11–20)"]
        Rack8["Rack 8: LPUs 505–576 (Layers 71–80)"]

        Rack1 -->|Deterministic C2C Optical Links\n(Zero Collision / Pre-Scheduled)| Rack2
        Rack2 -.->|Pipelined Activation Streaming| Rack8
    end

    classDef default fill:#1e293b,stroke:#475569,stroke-width:1px,color:#f8fafc;
    classDef highlight fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#38bdf8;
    class SRAMArray,ExecutionSlices,Rack1,Rack2,Rack8 highlight;
```

### Architectural Walkthrough

1. **Functional Slicing**: Unlike traditional GPU symmetric multiprocessors that combine control, memory, and arithmetic units in repeated square tiles, the GroqChip organizes functional units into vertical columns: Matrix Execution Units (MXM), Vector Execution Units (VXM), and Memory Units (MEM) [1].
2. **Pure On-Chip Storage**: The chip integrates 230 MB of high-speed SRAM directly adjacent to execution units, delivering 80 TB/s aggregate on-chip bandwidth at single-clock-cycle access latencies [1].
3. **Software-Defined Instruction Pipeline**: Instructions do not decode dynamically in hardware. The compiler constructs a wide horizontal instruction word, orchestrating data movement across vertical functional slices with cycle-level precision [1, 2].
4. **Deterministic Multi-Chip Scale-Out**: To host large models, hundreds of GroqChips connect via point-to-point chip-to-chip (C2C) copper and optical cables. Inter-chip communication uses synchronous time-division multiplexing: the compiler schedules exactly when a data packet leaves Chip A and when it arrives at Chip B, eliminating dynamic packet headers, routing arbiters, and network congestion [2].
5. **Streaming Model Pipelining**: Model layers are partitioned across the 576-LPU cluster. Activations stream forward from chip to chip through the pre-scheduled mesh, with each chip executing matrix-vector operations on local SRAM-resident weights without pausing for external memory fetches [2, 3].

---

## Core Math: The Memory-Bandwidth Roofline & Chip-Count Derivations

### 1. The Roofline Model: Maximum Tokens/Second per Stream

The theoretical decoding speed of an autoregressive model running with batch size $B=1$ is strictly upper-bounded by the ratio of aggregate memory bandwidth to the total bytes of weights transferred per token:

$$R_{\text{decode}} \le \frac{\text{Aggregate Memory Bandwidth (Bytes/sec)}}{\text{Bytes of Weights Read per Token}} \quad \text{(derived)}$$

#### Case A: Single NVIDIA H100 SXM5 ($80\text{ GB}$ HBM3, $3.35\text{ TB/s}$)
- **FP16 Precision ($b = 2\text{ bytes/param}$)**: A 70B parameter model requires $70 \times 10^9 \times 2 = 140\text{ GB}$ of weights. It **cannot physically fit** in a single 80 GB GPU.
- **INT8 / FP8 Precision ($b = 1\text{ byte/param}$)**: Model weights require $70\text{ GB}$, fitting within the 80 GB HBM3 buffer.
- Maximum theoretical single-stream decode speed:
  $$R_{\text{max}} = \frac{3{,}350 \times 10^9 \text{ bytes/sec}}{70 \times 10^9 \text{ bytes}} \approx \mathbf{47.86 \text{ tokens/sec}} \quad \text{(derived)}$$
- **Tensor Core Underutilization Derivation**:
  An H100 SXM5 provides $1{,}979 \text{ TFLOPS}$ ($1{,}979{,}000 \text{ GFLOPS}$) of FP8 Tensor Core compute.
  Each generated token executes $2 \times 70 \times 10^9 = 140 \text{ GFLOPs}$.
  At $47.86 \text{ tokens/sec}$, the active compute performance is:
  $$\text{Compute Realized} = 47.86 \times 140 = 6{,}700.4 \text{ GFLOPS} \approx \mathbf{6.70 \text{ TFLOPS}} \quad \text{(derived)}$$
  $$\text{Tensor Core Utilization} = \frac{6.70 \text{ TFLOPS}}{1{,}979 \text{ TFLOPS}} \approx \mathbf{0.34\%} \quad \text{(derived)}$$
  The GPU’s tensor compute cores sit idle **$99.66\%$ of the time**, stalled on the HBM3 physical bus.

#### Case B: 8-GPU NVIDIA H100 HGX Cluster (Tensor Parallelism $\text{TP}=8$)
- Aggregate memory bandwidth: $8 \times 3.35 \text{ TB/s} = 26.8 \text{ TB/s} = 26{,}800 \text{ GB/s}$.
- At FP16 ($140 \text{ GB}$ weights):
  $$R_{\text{max}} = \frac{26{,}800 \text{ GB/s}}{140 \text{ GB}} \approx \mathbf{191.43 \text{ tokens/sec}} \quad \text{(derived)}$$
- At INT8 / FP8 ($70 \text{ GB}$ weights):
  $$R_{\text{max}} = \frac{26{,}800 \text{ GB/s}}{70 \text{ GB}} \approx \mathbf{382.86 \text{ tokens/sec}} \quad \text{(derived)}$$
- *(Real-world efficiency: In production serving engines like vLLM or TensorRT-LLM, inter-GPU NVLink All-Reduce communication latencies and kernel launch overhead reduce observed single-stream INT8 decode speeds to ~100–180 tokens/sec).*

#### Case C: Groq LPU Cluster (SRAM-Resident Weights)
- On Groq, all model parameters reside permanently in on-chip SRAM across chips.
- Because internal SRAM bandwidth is $80\text{ TB/s}$ per chip, memory bandwidth is no longer the operational ceiling.
- **What actually limits LPU decoding speed?**
  1. **Inter-Chip Interconnect Bandwidth**: Activations must traverse C2C links between chips.
  2. **Pipeline Latency**: The sequential layer-by-layer dependency requires activations to pass through all 576 chips before the next token can be sampled.
  3. **Arithmetic Pipeline Depth**: The latency of matrix-vector multiplication through the functional slices.
  On Llama 3 8B, this yields ~877 tokens/sec; on Llama 3 70B, it achieves ~284–305 tokens/sec [4].

#### How Batching Inverts the Economics
The speed advantage of Groq applies specifically to **single-stream interactive queries** ($B=1$).
If a GPU cluster serves a batch of $B=64$ users simultaneously:
- The 140 GB of model weights is fetched from HBM **once** and reused across all 64 user tokens.
- Arithmetic intensity increases $64\times$ ($I = 64 \text{ FLOPs/byte}$), and GPU Tensor Core utilization climbs from $<1\%$ to over $40\%$.
- An 8-H100 system generates thousands of aggregate tokens per second across 64 streams, achieving drastically lower hardware cost per token than replicating 576-chip LPU clusters.

---

### 2. Chip-Count Math: Minimum Silicon vs. Production Reality

To host a model on pure SRAM, total model footprint must fit within the aggregate SRAM of the cluster:

$$\text{Minimum Chips} = \left\lceil \frac{\text{Model Weights Memory (MB)}}{230 \text{ MB per Chip}} \right\rceil \quad \text{(derived)}$$

#### Mathematical Minimums for 70B Parameter Model
- **At FP16 ($140{,}000\text{ MB}$)**:
  $$\text{Minimum Chips}_{\text{FP16}} = \left\lceil \frac{140{,}000}{230} \right\rceil = \lceil 608.7 \rceil = \mathbf{609 \text{ chips}} \quad \text{(derived)}$$
- **At INT8 ($70{,}000\text{ MB}$)**:
  $$\text{Minimum Chips}_{\text{INT8}} = \left\lceil \frac{70{,}000}{230} \right\rceil = \lceil 304.3 \rceil = \mathbf{305 \text{ chips}} \quad \text{(derived)}$$

#### Why Production Clusters Deploy 576 Chips
SemiAnalysis documented that Groq serves 70B / Mixtral models on clusters of **576 LPUs** [3]. The gap between 305 minimum chips and 576 deployed chips is explained by:
1. **Total Available SRAM**: $576 \times 230\text{ MB} = 132{,}480\text{ MB} \approx 132.5\text{ GB}$.
2. **Weight Allocation**: An INT8 quantized model consumes ~70 GB, leaving ~62.5 GB of SRAM across the cluster.
3. **KV Cache & Activation Buffers**: Context windows (e.g., 4k–8k tokens) require significant SRAM to maintain Key-Value caches directly on-chip.
4. **Network Topology Constraints**: Groq clusters scale across structured chassis topologies (8 chips per server $\times$ 9 servers per rack $\times$ 8 racks = 576 chips) to maintain balanced Dragonfly/Clos communication routes [3].

---

### 3. Software-Controlled Determinism: Pseudo-Code Scheduling Model

Groq removes hardware arbiters; the compiler plans every instruction and wire transfer cycle-by-cycle at compile time [1, 2]. Below is a conceptual illustration of this deterministic execution model:

```python
# Simplified Conceptual Model of Groq Compile-Time Static Scheduling
# (Illustrative representation of cycle-accurate software-defined execution)

class StaticCompilerSchedule:
    def __init__(self, num_chips: int = 576):
        self.num_chips = num_chips
        # Each clock cycle contains pre-assigned execution commands per functional slice
        self.cycle_schedule = {}

    def plan_cycle_execution(self):
        # The compiler dictates exact clock cycles for execution and C2C data transfers
        # Cycle 100: Chip 12 executes matrix multiplication on Layer 4 weights
        self.schedule_op(
            cycle=100,
            chip_id=12,
            unit="MXM_COLUMN_0",
            instruction="MATMUL_FP16_VECTOR",
            operands={"weight_sram_addr": 0x4F00, "activation_reg": 3}
        )

        # Cycle 120: Output activation is pushed to Chip-to-Chip (C2C) optical link
        self.schedule_c2c_transfer(
            cycle_send=120,
            src_chip=12,
            dst_chip=13,
            wire_transit_cycles=15,  # Fixed physical time of flight over optical cable
            cycle_arrive=135,
            target_reg=1
        )

        # Cycle 136: Chip 13 begins Vector Activation (VXM) without polling or handshakes
        self.schedule_op(
            cycle=136,
            chip_id=13,
            unit="VXM_COLUMN_1",
            instruction="SILU_ACTIVATION",
            operands={"activation_reg": 1}
        )

    def schedule_op(self, cycle, chip_id, unit, instruction, operands):
        self.cycle_schedule[(cycle, chip_id, unit)] = (instruction, operands)

    def schedule_c2c_transfer(self, cycle_send, src_chip, dst_chip, wire_transit_cycles, cycle_arrive, target_reg):
        # Guaranteed delivery: No TCP/IP headers, no packet collision buffers, zero dropouts
        pass
```

---

## Production Post-Mortems and System Limitations

*(Note: Groq has not suffered a famous public catastrophic outage. The following analysis dissects verified architectural failure modes, physical scaling limitations, and operational bottlenecks documented by independent hardware researchers).*

### Failure Mode 1: The SRAM-Capacity-Driven Chip Explosion

#### What Breaks
Attempting to deploy large-parameter models (such as DeepSeek-V3 with 671B parameters or Llama 3.1 405B) on first-generation LPU hardware results in an unsustainable physical footprint explosion. Storing a 671B model even at FP8 precision ($671\text{ GB}$ weights) requires over 2,900 discrete LPU chips, demanding massive rack space, tens of thousands of optical interconnects, and megawatts of power to host a single model instance [3].

#### Root Cause Analysis
SRAM provides unmatched bandwidth ($80\text{ TB/s}$) but suffers from terrible physical density:
- A 6T-SRAM cell requires 6 physical transistors per bit, compared to 1 transistor and 1 capacitor for DRAM.
- SRAM occupies massive silicon die area. On TSMC 14nm / 7nm nodes, embedding more than 200–300 MB of SRAM on a reticle-limited chip leaves minimal space for arithmetic logic.
- As models scale past 100B parameters, the chip count scales linearly with model memory footprint, requiring massive numbers of host CPUs, motherboards, and power distribution units just to provide memory storage capacity [3].

#### Architectural Mitigation
Groq entered into a manufacturing partnership with **Samsung Foundry** to build next-generation LPUs on a 4-nanometer process node at Samsung’s Taylor, Texas facility, designed to significantly increase on-chip SRAM density and compute capacity per die [6]. Furthermore, following the December 2025 NVIDIA licensing deal, Groq restructured its cloud to deploy hybrid architectures—using GPUs for high-density, memory-heavy workloads and LPUs for ultra-low-latency voice and agent streams.

---

### Failure Mode 2: Compile-Time Inflexibility and Dynamic Shape Freezes

#### What Breaks
In early Groq deployments, serving variable-length conversations resulted in high compilation latency or pipeline stalls. If a user prompt arrived with an input sequence length that had not been pre-compiled, the inference engine could not process the query dynamically, requiring padding to fixed buckets or triggering expensive offline compilation cycles.

#### Root Cause Analysis
Dynamic execution in modern CPUs and GPUs handles arbitrary sequence lengths naturally through hardware instruction pointers and runtime while-loops. 
- Groq’s determinism requires the compiler to lay out every instruction and memory address for every clock cycle in advance [1, 2].
- If a sequence length changes from 128 to 129 tokens, the dynamic dependencies shift, invalidating the cycle-level packet delivery schedule.
- True dynamic runtime control flow cannot execute on a statically scheduled VLIW/spatial processor without hardware stalls.

#### Architectural Mitigation
Groq solved dynamic sequence variation through **Static Sequence Bucketing**:
1. The compiler generates pre-compiled instruction kernels for discrete sequence lengths (e.g., buckets of 128, 512, 1024, 2048 tokens).
2. Input prompts are padded to the nearest pre-compiled bucket boundary.
3. While padding introduces minor wasted arithmetic cycles, the $80\text{ TB/s}$ SRAM bandwidth processes padding tokens so fast that the latency penalty is negligible compared to dynamic GPU scheduling overhead [1].

---

## Architectural Trade-offs and What Groq Gave Up

The LPU architecture represents an extreme philosophical commitment to single-stream latency, accepting severe compromises elsewhere:

### 1. Single-Stream Speed vs. Total Cluster Capital Expenditure (CapEx)
- **The Gain**: Unmatched single-stream decoding speeds (~877 t/s on 8B, ~300 t/s on 70B), delivering the sub-20ms latency required for real-time voice conversations and automated reasoning loops [4].
- **The Cost**: Immense capital cost per model instance. Serving one 70B model requires 576 chips, 144 host CPUs, and 8 server racks [3]. An 8-GPU NVIDIA H100 HGX system fits in a single 10U chassis and costs a fraction of the capital investment.

### 2. Static Determinism vs. Dynamic Model Flexibility
- **The Gain**: Complete elimination of hardware schedulers, dynamic caches, branch predictors, and network arbitration logic, freeing over 30% of die area for pure compute and memory [1].
- **The Cost**: The compiler must solve an NP-hard scheduling problem across 576 chips. Adding new architectures (e.g., novel sparse attention mechanisms, hybrid state-space models like Mamba, or complex MoE routing) requires months of custom compiler development.

### 3. Pure Inference Specialization vs. Training Inability
- **The Gain**: Microarchitecture stripped of backward-pass state tracking, checkpointing overhead, and dynamic optimizer state buffers.
- **The Cost**: Complete inability to train models. Groq chips cannot execute backpropagation at scale cost-effectively; they are strictly inference engines.

---

## When Not to Copy the Groq LPU Architecture

Replicating an on-chip SRAM, statically scheduled spatial architecture is completely inappropriate under the following circumstances:

```
                            [Workload Objective?]
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
          Batch / Offline Inference              Real-Time / Voice
            or Model Training                  Interactive Inference
                    │                                   │
                    ▼                                   ▼
          Use NVIDIA / AMD GPUs               [Target Parameter Scale?]
         (H100 / B200 / MI300X)                         │
         High HBM Density & Batching          ┌─────────┴─────────┐
                                              ▼                   ▼
                                       < 70B Parameters     > 100B / Dense
                                              │                   │
                                              ▼                   ▼
                                         Groq LPU /         Use High-Density
                                         Cerebras CS-3      GPU Clusters
                                         (SRAM Dominates)   (Memory Footprint)
```

### 1. High-Throughput, Batch-Heavy Offline Workloads
If your system processes asynchronous document summarization, batch embeddings, or background data processing where latency is secondary to cost per token, GPUs dominate. Running batches of 64 or 128 on an H100 amortizes weight loading and achieves far higher token throughput per dollar than an LPU cluster.

### 2. Large Mixture-of-Experts (MoE) Models Exceeding 100 Billion Parameters
Models like DeepSeek-V3 (671B parameters) or large dense models (Llama 405B) cannot be hosted cost-effectively on SRAM chips. The physical chip count exceeds thousands of processors, rendering interconnect complexity, cabling failures, and power delivery impractical.

### 3. Engineering Teams Without World-Class Compiler Talent
A deterministic spatial processor is only as good as its compiler. If the compiler cannot perfectly schedule operations without inserting massive NOP bubbles, hardware utilization drops to near zero. Unless an organization can dedicate hundreds of engineers to compiler graph optimization, standard CUDA-compatible hardware is mandatory.

---

## Frequently Asked Questions About Groq LPU Architecture

### Why is Groq faster than NVIDIA GPUs for LLM inference?
Groq is faster than GPUs for single-stream inference because it stores all model weights in on-chip SRAM rather than external High Bandwidth Memory (HBM) [1]. Standard GPUs are memory-bandwidth-bound during single-token decoding, spending over 90% of their time stalled waiting for weights to travel across physical memory buses. Groq’s on-chip SRAM delivers 80 TB/s bandwidth per chip with single-clock-cycle latencies, keeping execution units continuously saturated [1].

### What is the difference between an LPU and a GPU?
A GPU (Graphics Processing Unit) is a general-purpose parallel processor with dynamic hardware schedulers, multi-level cache hierarchies, and external DRAM/HBM designed to handle complex training and high-batch workloads. An LPU (Language Processing Unit) is a deterministic, statically scheduled processor with on-chip SRAM and no external DRAM [1]. The LPU relies on its compiler to plan exact cycle-by-cycle instruction execution and chip-to-chip routing in advance, eliminating hardware scheduling bubbles [2].

### Why does Groq require so many chips to run a 70B model?
Groq requires hundreds of chips (typically 576 chips in production clusters) because each chip contains only 230 MB of on-chip SRAM [1, 3]. Storing an INT8 quantized 70B model requires ~70 GB of memory. To host the weights, activations, Key-Value caches, and compiler routing buffers, the model must be partitioned across hundreds of discrete chips connected via point-to-point interconnects [3].

### What happened between NVIDIA and Groq in December 2025?
In December 2025, NVIDIA entered into a $20 billion non-exclusive technology licensing and talent agreement with Groq. NVIDIA licensed Groq’s low-latency inference IP and patent portfolio, and Groq founder Jonathan Ross and key engineering leaders joined NVIDIA. Groq restructured into an independent AI inference cloud provider led by CEO Simon Edwards and became an official NVIDIA Cloud Partner (NCP).

### Can Groq be used to train AI models?
No. The Groq LPU was designed specifically for autoregressive inference. It lacks the massive off-chip memory capacity required to store optimizer states (Adam momentum and variance), activation checkpoints, and gradient tensors, and its static compiler schedule is optimized for forward-pass execution graphs rather than dynamic backpropagation loops.

---

## References

1. Abts, D., Ross, J., et al. (Groq Architecture Team). (2020). "Think Fast: A Tensor Streaming Processor (TSP) for Accelerating Deep Learning Workloads." *Proceedings of the 47th International Symposium on Computer Architecture (ISCA)*. IEEE/ACM. Available at: https://ieeexplore.ieee.org/document/9138980
2. Abts, D., et al. (2022). "A Software-Defined Tensor Streaming Multiprocessor for Large-Scale Machine Learning." *Proceedings of the 49th International Symposium on Computer Architecture (ISCA)*. IEEE/ACM. Available at: https://ieeexplore.ieee.org/document/9869680
3. Patel, D., & Ahmad, A. (2024). "Groq Inference Tokenomics: Speed, But At What Cost?" *SemiAnalysis Research Report*. Available at: https://semianalysis.com/2024/02/27/groq-inference-tokenomics-speed-but-at-what-cost/
4. Artificial Analysis. (2024). "Independent LLM Hosting Benchmarks: Groq Speed & Latency Evaluations." *Artificial Analysis Documentation*. Available at: https://artificialanalysis.ai
5. NVIDIA Corporation. (2023). "NVIDIA H100 Tensor Core GPU Architecture Whitepaper." *NVIDIA Technical Documentation*. Available at: https://www.nvidia.com/en-us/data-center/h100/
6. Samsung Electronics & Groq. (2023). "Groq Contracts Next-Generation LPU Manufacturing with Samsung Foundry 4nm Node in Taylor, Texas." *Press Release*. Available at: https://groq.com

---

Authored by @saurabh-vision
