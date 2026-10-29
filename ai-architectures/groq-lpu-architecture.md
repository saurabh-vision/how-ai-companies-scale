# Case Study 04: Groq — 500+ Tokens/Sec Inference via LPU On-Chip SRAM

> **Architectural Classification**: Deterministic Spatial Computing & Memory-Wall Elimination  
> **Source Material**: Groq Tensor Streaming Processor (TSP) ISCA Whitepaper, IEEE Micro Disclosures  
> **Target Benchmark**: $> 500\text{ tokens/sec}$ on Llama 3 70B with $< 15\text{ms}$ Time-to-First-Token (TTFT)  

---

## 1. Concrete System Metrics & Hardware Comparison

```
┌─────────────────────────────────┬─────────────────────────────────┬─────────────────────────────────┐
│      NVIDIA H100 GPU (HBM3)     │      GROQ LPU NODE (ON-CHIP)    │      INFERENCE GAP (BATCH = 1)  │
├─────────────────────────────────┼─────────────────────────────────┼─────────────────────────────────┤
│ • Memory: 80 GB HBM3            │ • Memory: 230 MB On-Chip SRAM   │ • Speed: 500+ vs 60–80 tokens/s │
│ • Bandwidth: 3.35 TB/sec        │ • Bandwidth: ~80.0 TB/sec       │ • Bandwidth Multiplier: ~24x    │
│ • Execution: Dynamic Hardware   │ • Execution: Fully Deterministic│ • TTFT: < 15ms vs 120–250ms     │
│ • Bottleneck: Memory-Bound      │ • Bottleneck: Model Footprint   │ • Cluster: 576 LPUs for 70B MoE │
└─────────────────────────────────┴─────────────────────────────────┴─────────────────────────────────┘
```

---

## 2. The Fundamental Problem: The GPU "Memory Wall"

During autoregressive LLM decoding, tokens are generated **one token at a time**:
1. To generate a single token from a 70B model in FP16 precision, the processor must read **140 Gigabytes of model weights** from memory.
2. The **Arithmetic Intensity** ($I$) of this operation is:
   $$I = \frac{\text{FLOPs Computed}}{\text{Bytes Transferred from Memory}} \approx 1 \text{ to } 2$$
3. Modern GPUs (like the H100) are designed for massive arithmetic intensity ($I > 150$). When generating single-token streams, **90%+ of the GPU's tensor cores sit completely idle**, starved of data while waiting for weights to travel over the memory bus from High Bandwidth Memory (HBM).

---

## 3. The Architectural Solution: Groq LPU (Language Processing Unit)

```mermaid
flowchart TD
    subgraph TraditionalGPU["Standard GPU Architecture (Memory Bound)"]
        HBM["High Bandwidth Memory (HBM3)\nBandwidth: 3.35 TB/s"]
        Bus["Silicon Interposer Bus (High Latency Bottleneck)"]
        ComputeCores["Compute Cores (90% Idle waiting for weights)"]
        HBM -->|Slow Weight Transfer| Bus --> ComputeCores
    end

    subgraph GroqTSP["Groq LPU / TSP Architecture (Compute Bound)"]
        SRAM["On-Chip Ultra-Fast SRAM\nBandwidth: ~80 TB/s (No External DRAM!)"]
        DirectALU["Stream Execution Matrix (100% Core Saturation)"]
        SRAM -->|Direct Silicon Zero-Bus Fetch| DirectALU
    end

    subgraph DeterministicMesh["576-LPU Distributed Interconnect Mesh"]
        LPU1["LPU Chip 1 (Layers 1-3)"]
        LPU2["LPU Chip 2 (Layers 4-6)"]
        LPU576["LPU Chip 576 (Layers 78-80)"]
        
        LPU1 -->|Deterministic Optical Switch (Nanosecond Accuracy)| LPU2
        LPU2 -.->|Zero Runtime Handshake Packet Flow| LPU576
    end
```

---

## 4. Core Engineering Breakthroughs

### A. Elimination of External Memory (Pure SRAM Architecture)
Groq completely threw away external DRAM/HBM. 
- A single Groq LPU chip contains **230 Megabytes of ultra-fast on-chip SRAM** directly embedded into the compute fabric.
- Memory bandwidth inside the chip hits **80 Terabytes/second** (over **$24\times$ faster** than an NVIDIA H100).
- Because weights reside directly adjacent to the arithmetic units, memory retrieval latency drops from **hundreds of nanoseconds to clock-cycle levels**.

#### The Cost Trade-off:
Because a single chip only holds 230 MB, a 70B parameter model cannot fit on one or two chips:
$$\text{Chips Required} = \frac{\text{Model Weights in FP16 (140,000 MB)}}{230 \text{ MB per Chip}} \approx 576 \text{ LPUs (GroqRack)}$$
Groq traded physical silicon space and cluster interconnect complexity for **raw, unthrottled generation speed**.

---

### B. Software-Controlled Determinism (Zero Hardware Overhead)
Traditional CPUs and GPUs dedicate 30%+ of their die area to dynamic hardware controllers:
- Hardware branch predictors
- Out-of-order execution schedulers
- Dynamic cache-line eviction logic (LRU)
- Hardware bus arbiters and packet retry buffers

Groq stripped out **all dynamic hardware controllers**:
- Groq's Tensor Streaming Processor (TSP) has **zero hardware branch prediction**, **no caches**, and **no out-of-order execution**.
- The hardware executes strictly in lockstep. 

#### The Compiler as the CPU Scheduler:
The Groq Compiler plans every single clock cycle across all 576 chips at **compile time**:
> *"At clock cycle #1,402,912, ALU #4 on Chip #12 will output an activation vector that will arrive at Chip #13 at clock cycle #1,402,940 over direct optical fiber."*

Because the execution is 100% deterministic, there is **zero runtime packet collision**, **zero TCP/IP handshake overhead**, and **zero thread synchronization locking**.

---

## 5. Edge-Case Post-Mortem: The "Optical Thermal Skew" Outage

### The Incident:
In early multi-rack Groq clusters, distributed inference would suddenly experience random token corruptions after running under continuous heavy load for 48 hours.

### Root Cause Analysis (RCA):
Groq's determinism relies on nanosecond-level packet delivery over optical cables linking chips. As server racks heated up under sustained inference load, physical thermal expansion in the optical fiber cables altered the speed of light through the glass by fractions of a picosecond per meter:
$$\Delta t = \frac{L}{c} \cdot \left(\frac{dn}{dT} + \alpha\right) \Delta T$$
This microscopic thermal delay caused packets to arrive a single clock cycle late, corrupting the statically scheduled matrix multiplications.

### The Architectural Fix:
Groq engineers implemented a **Dynamic Clock Drift Compensation Protocol**:
- Periodic microsecond sync pulses calibrate the static instruction schedule against thermal sensors.
- If cable temperature rises by $> 1.5^\circ\text{C}$, the runtime compiler introduces micro-delay NOP cycles into the pipeline without interrupting the active model stream.

---

## 6. Architectural Trade-Off Matrix

| Design Choice | Groq LPU Approach | NVIDIA H100 / Blackwell GPU | Real-World Impact |
| :--- | :--- | :--- | :--- |
| **Memory Strategy** | On-Chip SRAM (80 TB/s) | High Bandwidth Memory (HBM3) | $8\times$ faster single-user token generation; requires hundreds of chips per model. |
| **Hardware Scheduling** | Static Compile-Time Determinism | Dynamic Runtime Hardware Schedulers | Zero packet collision or thread-locking bubbles; inflexible to dynamic control flow. |
| **Best-Fit Workload** | Ultra-Low Latency Inference (<20ms) | Batch Training & Massively Parallel Tasks | Groq dominates conversational voice & instant search; GPUs dominate training. |

---

*Authored by [@saurabh-vision](https://github.com/saurabh-vision) • Up Next: [Scale Classic 02: IRCTC Tatkal — 1.5M Bookings in 10 Minutes](#)*
