---
title: "How Cursor Codebase Indexing Works: Merkle Trees, AST Chunks, and Speculative Edits"
description: "How Cursor codebase indexing works: using Merkle trees, Tree-sitter AST chunking, Turbopuffer vector search, and speculative edits for 1,000 token/sec coding."
keywords:
  - how cursor codebase indexing works
  - cursor merkle tree indexing
  - tree-sitter code chunking
  - codebase RAG embeddings
  - speculative decoding code completion
  - turbopuffer
  - fast apply
date: 2026-10-30
last_verified: 2026-10-30
---

# How Cursor Codebase Indexing Works: Merkle Trees, AST Chunks, and Speculative Edits

> [!NOTE]
> **TL;DR**
> - **Collaborative Index Reuse**: Cursor avoids redundant monorepo indexing by computing a similarity hash of the workspace Merkle tree, enabling new developers to reuse up to 92% of pre-computed embeddings from teammates [1].
> - **Fast Apply Generation Throughput**: By utilizing the existing code buffer as a strong prior in a speculative edit pipeline, Cursor streams code changes at ~1,000 tokens per second (~3,500 characters/sec) with a fine-tuned 70B parameter model [2].
> - **Search Activation Threshold**: Background codebase indexing unlocks semantic search and codebase-wide context retrieval once the workspace index reaches 80% completion [4].
> - **Object-Store Vector Scale**: Cursor delegates billions of vector embeddings across millions of active codebases to Turbopuffer, a serverless vector database operating on cloud object storage with SSD caching [3, 5].
> - **Cryptographic Access Verification**: Remote vector search returns only obfuscated paths and line ranges; the client verifies code chunk access via Merkle cryptographic proofs before pulling source text locally [1, 5].

A modern enterprise repository contains millions of lines of code across tens of thousands of files, yet developers expect natural-language codebase queries and inline autocomplete to respond within milliseconds of a keystroke. Standard dense vector embedding models take hours to sequentially index a 100,000-file repository while saturating local CPU cores, and frontier reasoning models require hundreds of milliseconds just to stream their first token over wide-area networks. The math dictates that client-side codebase retrieval and real-time code transformations should either exhaust local laptop hardware or introduce intolerable latency, but Cursor bypasses these boundaries through differential Merkle trees, collaborative similarity hashing, and speculative edit pipelines.

---

## Production Scale and Operational Constraints

Indexing code for real-time Retrieval-Augmented Generation (RAG) imposes constraints that do not exist in traditional web document retrieval.

| Metric / Dimension | Production Target / Observed Value | Source / Classification |
| :--- | :--- | :--- |
| **Index Sharing Overlap** | ~92% similarity across team workspaces | Confirmed (Cursor Engineering Blog) [1] |
| **Fast Apply Throughput** | ~1,000 tokens/sec (~3,500 chars/sec) | Confirmed (Cursor Research Blog) [2] |
| **Fast Apply Base Architecture**| Fine-tuned 70B parameter model with speculative edits | Confirmed (Cursor Research Blog) [2] |
| **Semantic Search Activation** | Activates at 80% background completion | Confirmed (Cursor Official Documentation) [4] |
| **Vector Database Layer** | Serverless vector database on object storage | Confirmed (Turbopuffer / Simon Willison) [3, 5] |
| **Vector Search Metadata** | Obfuscated relative paths and line numbers | Confirmed (Cursor Security / Turbopuffer) [1, 5] |
| **Code Completion Architecture**| Custom sparse model trained for next-action prediction | Confirmed (Cursor Engineering Blog) [3] |
| **Change Detection Mechanism** | Recursive Merkle hash tree of workspace files | Confirmed (Cursor Engineering Blog) [1] |
| **Local File Exclusion** | Evaluated via `.cursorignore` patterns | Confirmed (Cursor Documentation) [4] |

### The Pathologies of Indexing Source Code

Standard document retrieval strategies fail when applied to software engineering projects:

1. **Brittle Syntactic Boundaries**: Arbitrary character or token chunking splits classes, loops, and function signatures down the middle. When an embedding model encodes a chunk containing variable assignments without their enclosing types or method declarations, semantic search relevance degrades.
2. **High-Frequency Incremental Mutation**: Developers save files multiple times per minute. Re-embedding a whole repository on every keystroke or git branch switch saturates client CPU and network bandwidth. Change detection must isolate changed syntax scopes in sub-millisecond time.
3. **Multi-User Monorepo Redundancy**: In engineering organizations with hundreds of developers checking out the same repository, having each engineer compute identical embeddings on identical commit snapshots wastes gigawatt-hours of compute and causes hours of onboarding delay.

---

## End-to-End Codebase Retrieval and Edit Architecture

Cursor structures its codebase retrieval and generation workflow into an edge client layer, a cryptographic synchronization pipeline, a serverless vector store, and a speculative inference engine.

```mermaid
flowchart TD
    subgraph ClientEditor["Client Editor Tier (Local Machine)"]
        FSWatcher["Filesystem Watcher & .cursorignore"]
        MerkleTree["Local Merkle Tree (File & Folder Hashes)"]
        TreeSitter["Tree-sitter AST Chunker (Syntax Hoisting)"]
        LocalDisk["Raw Source Code Files (Local Disk)"]
        ClientBuffer["Active Editor Buffer (Keystrokes / Edits)"]
    end

    subgraph SyncTier["Cryptographic Sync & Sharing Tier"]
        SimHash["SimHash Generator (Workspace Fingerprint)"]
        ProofEngine["Merkle Access Proof Engine"]
    end

    subgraph RemoteStorageTier["Remote Vector Storage (Turbopuffer)"]
        VectorDB["Turbopuffer Serverless Vector Store"]
        MetadataStore["Obfuscated Paths & Line Range Metadata"]
        VectorDB <--> MetadataStore
    end

    subgraph ModelServingTier["Inference & Speculative Engine"]
        FrontierModel["Frontier Planning Model (Claude / GPT)"]
        SpeculativeDrafter["Speculative Fast Apply Engine (70B Model)"]
        TabModel["Cursor Tab Sparse Next-Action Model"]
    end

    FSWatcher --> MerkleTree
    MerkleTree --> TreeSitter
    MerkleTree --> SimHash
    SimHash -->|Query Pre-built Index| ProofEngine
    TreeSitter -->|Chunk Vectors| VectorDB
    ProofEngine -->|Authorize Search Scope| VectorDB

    ClientBuffer --> TabModel
    ClientBuffer --> FrontierModel
    FrontierModel -->|Planned Code Edits| SpeculativeDrafter
    SpeculativeDrafter -->|Full File Rewrites (1000 tok/s)| ClientBuffer

    VectorDB -->|Return Matching Pointers| ProofEngine
    ProofEngine -->|Resolved File & Line Ranges| LocalDisk
    LocalDisk -->|Inject Retrieved Context| FrontierModel

    classDef default fill:#1e293b,stroke:#475569,stroke-width:1px,color:#f8fafc;
    classDef highlight fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#38bdf8;
    class MerkleTree,SpeculativeDrafter,VectorDB highlight;
```

### Architectural Walkthrough

1. **Local Change Detection**: When a workspace opens, Cursor constructs a local Merkle tree of cryptographic hashes representing all tracked files and directories, ignoring paths defined in `.cursorignore` [1, 4].
2. **Collaborative Index Resolution**: The client computes a similarity hash (SimHash) of its local Merkle tree and queries Cursor’s cloud infrastructure. If a teammate has already indexed a matching Git commit or identical repository state, the client downloads index references instead of re-embedding the codebase [1].
3. **AST-Aware Chunking**: Unindexed or dirty files are parsed via Tree-sitter. Code is sliced along semantic boundaries (methods, functions, classes), hoisting enclosing parent signatures and import declarations into the chunk metadata to preserve syntactic context.
4. **Vector Persistence in Turbopuffer**: The client sends vector embeddings to Turbopuffer, where vectors are stored alongside obfuscated relative paths and line numbers [3, 5]. Raw source code text is not retained in the remote vector database [1, 5].
5. **Speculative Generation**: For code generation and editing, Cursor decouples planning from application. A conversational frontier model formulates the edit plan, and a fine-tuned 70B parameter model uses the existing buffer as draft tokens in a speculative edit pipeline, rewriting code at ~1,000 tokens per second [2].

---

## Core Algorithms, Hash Math, and Speculative Decoding

### 1. Merkle Tree Differential Synchronization and SimHash Matching

A Merkle tree represents directory structures where leaf nodes are cryptographic hashes of individual file contents, and parent nodes are hashes of their combined children.

$$\text{Hash}(\text{Directory}) = H\left(\text{Hash}(\text{Child}_1) \;\parallel\; \text{Hash}(\text{Child}_2) \;\parallel\; \dots \;\parallel\; \text{Hash}(\text{Child}_n)\right)$$

When a developer edits a single function inside `src/auth/login.ts`:
1. Only the leaf hash for `login.ts` changes.
2. The hash mutation propagates exclusively up the path ancestors: `src/auth/` and `src/` to `Root`.
3. To determine what changed, the client and server exchange root hashes. If roots differ, they traverse only the diverging branches in $O(k \log N)$ steps, where $k$ is the number of modified files and $N$ is total repository size.

To identify whether a teammate has an index matching a local repository that may differ by minor local branches or uncommitted files, Cursor computes a **Similarity Hash (SimHash)** over the Merkle structure [1].

#### SimHash Mathematical Derivation
Let the set of features $F = \{f_1, f_2, \dots, f_m\}$ represent the Merkle tree paths and hashes, where each feature has weight $w_i$ and a $b$-bit hash $h(f_i) \in \{0, 1\}^b$. A $b$-dimensional accumulator vector $\vec{V} = [v_1, v_2, \dots, v_b]$ is initialized to zeros.

For each feature $f_i$:
$$v_j \leftarrow v_j + \begin{cases} w_i, & \text{if } j\text{-th bit of } h(f_i) = 1 \\ -w_i, & \text{if } j\text{-th bit of } h(f_i) = 0 \end{cases}$$

The final $b$-bit fingerprint $\vec{S}$ is generated by:
$$S_j = \begin{cases} 1, & \text{if } v_j > 0 \\ 0, & \text{if } v_j \le 0 \end{cases}$$

Workspaces with high structural similarity yield low Hamming distances between their fingerprints:
$$D_H(\vec{S}_1, \vec{S}_2) = \sum_{j=1}^b (S_{1,j} \oplus S_{2,j})$$

If $D_H$ falls below a configured threshold, the client reuses the remote index and synchronizes only the divergent file subtrees [1].

---

### 2. Speculative Decoding and Speculative Edits

Traditional autoregressive token generation requires a full forward pass through model parameters for every emitted token. If a 70B parameter model requires 25ms per forward pass, generating 40 tokens sequentially takes:

$$T_{\text{autoregressive}} = 40 \times 25\text{ms} = 1{,}000\text{ms} \quad \text{(derived)}$$

Cursor's Fast Apply relies on **speculative edits**, a variant of speculative decoding optimized for code modification [2]. In code editing, 80% to 90% of the original file content remains identical across modifications. The original file text serves as a zero-cost draft sequence proposal.

```
Existing File Buffer (Draft Prior):
[function] [calculateTax] [(] [income] [:] [number] [)] [{] [return] [income] [*] [0.2] [;] [}]

Modified Intended Sequence:
[function] [calculateTax] [(] [income] [:] [number] [)] [{] [return] [income] [*] [0.25] [;] [}]
                                                                                  ▲
                                                                     Divergence Point
```

#### Speculative Acceptance Criterion
Let $q(x \mid C)$ be the draft proposal probability distribution (derived from the existing file buffer or a fast draft model), and $p(x \mid C)$ be the target model probability distribution. A candidate token $x$ is accepted with probability:

$$\alpha = \min\left(1, \frac{p(x \mid C)}{q(x \mid C)}\right)$$

If a draft token is rejected at index $k$, the target model resamples token $x_k$ from the adjusted distribution:

$$p'(x) = \frac{\max(0, p(x) - q(x))}{\sum_y \max(0, p(y) - q(y))}$$

#### Worked Example: Speedup Derivation
Assume a block of $M = 20$ tokens is validated. The original file matches the target code for 16 tokens before a variable name is updated.
- Standard generation: 20 sequential forward passes $\times 25\text{ms} = 500\text{ms}$.
- Speculative verification: All 16 matching tokens plus the divergence token are processed concurrently in a **single forward pass** ($\approx 30\text{ms}$).
- Speedup factor for that block:
  $$\text{Speedup} = \frac{16 \times 25\text{ms}}{30\text{ms}} \approx 13.3\times \quad \text{(derived)}$$

This parallel verification allows Cursor’s Fast Apply to operate at ~1,000 tokens per second [2].

---

### 3. Tree-Sitter AST Scope Chunking Implementation

Below is a structural implementation in Rust showing how an AST-aware chunker hoists parent scope metadata to preserve context across function boundaries:

```rust
use std::path::PathBuf;

#[derive(Debug, Clone)]
pub struct AstSemanticChunk {
    pub file_path: PathBuf,
    pub symbol_identifier: String,
    pub parent_scope: Option<String>,
    pub hoisted_context: String,
    pub content: String,
    pub start_line: usize,
    pub end_line: usize,
}

pub struct CodeScopeChunker {
    max_chunk_tokens: usize,
}

impl CodeScopeChunker {
    pub fn new(max_chunk_tokens: usize) -> Self {
        Self { max_chunk_tokens }
    }

    /// Slices code while maintaining enclosing class and import context
    pub fn chunk_source_file(
        &self,
        file_path: PathBuf,
        imports: &[String],
        class_name: Option<&str>,
        functions: &[(String, String, usize, usize)], // (name, body, start, end)
    ) -> Vec<AstSemanticChunk> {
        let mut chunks = Vec::new();
        let hoisted_imports = imports.join("\n");

        for (fn_name, fn_body, start, end) in functions {
            let mut context_prefix = hoisted_imports.clone();
            if let Some(cls) = class_name {
                context_prefix.push_str(&format!("\n// Enclosing Context: class {}\n", cls));
            }

            chunks.push(AstSemanticChunk {
                file_path: file_path.clone(),
                symbol_identifier: fn_name.clone(),
                parent_scope: class_name.map(|s| s.to_string()),
                hoisted_context: context_prefix,
                content: fn_body.clone(),
                start_line: *start,
                end_line: *end,
            });
        }

        chunks
    }
}
```

---

## Production Post-Mortem: Real Failure Modes of Codebase Indexing

*(Note: The following analysis addresses the inherent, publicly documented architectural failure modes of distributed codebase vector indexing systems, rather than an isolated internal Cursor outage).*

### Failure Mode: The Git Branch-Switching Re-indexing Storm

#### What Breaks
When an engineer switches between disparate Git branches (e.g., checking out a mainline `main` branch containing 15,000 modified files compared to a long-running feature branch), local filesystem watchers emit thousands of dirty event notifications within hundreds of milliseconds. In naive indexing architectures, this triggers a concurrent re-indexing storm that:
1. Slashes local IDE responsiveness as AST parsing threads saturate all CPU cores.
2. Floods the remote vector database with delete and insert requests, causing write-queue throttling.
3. Generates stale search queries where the editor's semantic context maps to file coordinates from the previous branch.

#### Root Cause Analysis
Filesystem watchers (such as Linux `inotify` or macOS `FSEvents`) report file modifications at the raw filesystem layer, without awareness of Git branch context. A branch switch does not create new logic from scratch; it swaps known Git trees. 

If the indexing engine attempts to compute line-level AST diffs linearly across 15,000 files:
- The local memory footprint of in-flight file buffers spikes.
- Vector upsert calls issue individual API requests with high network overhead.
- Because embedding generation takes time, queries executed immediately after checking out a branch retrieve vectors pointing to invalid line numbers.

#### Remediation and Architectural Mitigation
Modern codebase indexing architectures employ a three-tier defense:
1. **Merkle Commit Pinning**: The client binds Merkle trees directly to Git commit object identifiers (`commit SHA`). When a branch switch occurs, the client checks if the target commit hash already exists in the local or team index cache. If cached, the index pointer swaps instantaneously without re-parsing files.
2. **Debounced Event Aggregation**: Filesystem watcher notifications are buffered across a sliding temporal window (e.g., 500ms to 2,000ms). If a large volume of file mutations occur concurrently, the engine suspends individual file processing and performs a single top-down Merkle tree comparison against disk.
3. **Obfuscated Path and Line Indirection in Vector Stores**: As implemented in Turbopuffer, vectors store obfuscated path and line metadata rather than rigid chunk text [1, 5]. When a search executes, the client validates the returned line ranges against its active local file buffers before passing context to the model prompt, filtering out desynchronized chunks.

---

## Architectural Trade-offs and What Cursor Gave Up

Designing a responsive, AI-first code editor requires trading theoretical capabilities for operational latency:

### 1. Remote Index Sharing vs. Zero-Knowledge Code Privacy
By generating Merkle tree similarity hashes and enabling index sharing across teams, Cursor reduces repository onboarding from hours to seconds [1]. However, this requires uploading structural metadata, directory fingerprints, and embeddings to cloud infrastructure. While raw code text is not persisted in the vector store and cryptographic proofs guard chunk retrieval, strict zero-trust or air-gapped environments cannot utilize index sharing and must rely on purely local processing [1, 5].

### 2. Full-File Speculative Rewrites vs. Minimal Unified Diffs
Cursor’s Fast Apply model generates full file rewrites at 1,000 tokens/sec instead of emitting line-by-line diff patches [2]. 
- **The Gain**: Full-file streaming bypasses the fragility of unified diff parsing, where minor whitespace mismatches cause patch rejections.
- **The Cost**: Rewriting entire files consumes substantially more inference tokens than applying a 5-line diff patch. For massive 10,000-line files, full-file generation places higher bandwidth and memory demands on the serving tier.

### 3. Serverless Object-Store Vectors vs. In-Memory Graph Indexing
Cursor utilizes Turbopuffer, which manages vector embeddings on cloud object storage (such as AWS S3 or Google Cloud Storage) backed by SSD and memory caches [3, 5], rather than maintaining an in-memory HNSW (Hierarchical Navigable Small World) graph on dedicated cluster instances.
- **The Gain**: Dramatically lower infrastructure costs. Storing embeddings across millions of inactive repositories in RAM would be economically prohibitive.
- **The Cost**: Cold-start latency. Querying a repository that has not been accessed recently requires reading index partitions from object storage into NVMe cache, resulting in higher initial query latency compared to dedicated in-memory vector clusters.

### 4. Specialized Sparse Models vs. Frontier Model Autocomplete
Cursor Tab uses custom sparse language models fine-tuned specifically for next-action prediction and cursor navigation jumps, rather than prompting general-purpose frontier models (such as Claude 3.5 Sonnet or GPT-4o) [3]. This limits inline autocomplete to local structural transformations, leaving complex multi-step reasoning to intentional chat or composer interactions.

---

## When Not to Copy Cursor's Codebase Indexing Architecture

Building a Merkle-based hybrid retrieval engine and speculative editing pipeline is inappropriate for most standard software engineering projects:

```
                            [Codebase Scale?]
                                    │
                  ┌─────────────────┴─────────────────┐
                  ▼                                   ▼
          < 1,000 Files                        > 1,000 Files
                  │                                   │
      ┌───────────┴───────────┐                       │
      ▼                       ▼                       ▼
Use In-Memory SQLite    Use Standard Inverted   [Team Monorepo with
(FTS5 / Simple Vectors) Index (ripgrep / BM25)   Shared Workspaces?]
                                                ┌─────┴─────┐
                                                ▼           ▼
                                               No          Yes
                                                │           │
                                                ▼           ▼
                                           Local AST    Merkle Index
                                           Indexing     Sharing + Vector
                                           (LlamaIndex) DB (Turbopuffer)
```

### 1. Codebases Under 1,000 Files
For repositories containing fewer than 1,000 files, the overhead of building Merkle trees, managing SimHash fingerprints, and maintaining a vector database tier is unnecessary. A local `ripgrep` process combined with an in-memory lexical index (such as SQLite FTS5) provides sub-50ms search retrieval with zero cloud infrastructure complexity.

### 2. Strict Air-Gapped or Regulatory Zero-Cloud Environments
If your application operates under regulatory frameworks (such as defense, banking, or medical records) that forbid transmitting code metadata, hashes, or embeddings over the internet, Cursor's cloud-assisted vector architecture cannot be deployed. You must run localized embedding models on local GPU workstations or rely entirely on local regex search.

### 3. Non-Code Document Collections
Source code exhibits unique structural properties: strict syntax trees, functional scopes, and repetitive patterns across files. Natural language prose (such as markdown documentation, research papers, or legal contracts) lacks rigid scope hierarchies. Applying AST chunking and speculative buffer validation to unstructured text yields poor retrieval performance.

### 4. Workloads Dominated by Exact Symbol Lookups
If developer queries primarily target exact symbol definitions, interface implementations, and reference tracking, building a semantic vector index is the wrong engineering choice. A Language Server Protocol (LSP) engine using tree-sitter or SCIP (Source Code Intelligence Protocol) indexers provides 100% deterministic accuracy with zero embedding cost.

---

## Frequently Asked Questions About Cursor Codebase Indexing

### How does Cursor index a large codebase?
Cursor indexes large codebases by parsing repository files into Abstract Syntax Trees (ASTs) using Tree-sitter, generating vector embeddings for structural code chunks, and storing them in Turbopuffer [1, 3]. To avoid long indexing times on massive projects, Cursor builds a local Merkle tree of file hashes and computes a similarity hash (SimHash). This allows new users to instantly reuse pre-computed embeddings created by teammates working on the same repository [1].

### What vector database does Cursor use?
Cursor uses Turbopuffer, a serverless vector database designed to run on top of cloud object storage [3, 5]. Turbopuffer stores vector embeddings alongside obfuscated metadata (relative paths and line ranges) without storing raw source code text, allowing Cursor to perform nearest-neighbor semantic search cost-effectively across billions of vectors [1, 5].

### How does Cursor Fast Apply achieve 1000 tokens per second?
Cursor Fast Apply achieves ~1,000 tokens per second (~3,500 characters/sec) by using a fine-tuned 70B parameter model coupled with speculative edits [2]. Instead of generating tokens one by one or parsing fragile diff files, the model uses the existing file buffer as a draft sequence, allowing it to validate and stream unchanged code chunks in parallel forward passes while regenerating only modified tokens [2].

### Is my code private when Cursor indexes my codebase?
Under Cursor’s security model, raw source code is never permanently stored in plaintext within the remote vector database [1, 5]. Codebase embeddings and obfuscated metadata are transmitted securely, and search results return file pointers that are resolved against local files on your machine. Furthermore, users can enable Privacy Mode to prevent code data from being stored or used for model training [5].

### How does Cursor Tab differ from GitHub Copilot?
While traditional code completion tools (like early versions of GitHub Copilot) primarily focus on inserting text immediately following the active cursor position, Cursor Tab uses a custom sparse model trained for next-action prediction [3]. Cursor Tab can suggest full multi-line edits around the cursor, modify existing statements, and predict cursor navigation jumps to the next edit location [3].

---

## References

1. Stribling, J. (2024). "Securely indexing large codebases." *Cursor Engineering Blog*. Available at: https://cursor.com/blog/instant-codebase-indexing
2. Sanger, A. (2024). "Editing Files at 1000 Tokens per Second." *Cursor Research Blog*. Available at: https://cursor.com/blog/editing-files-at-1000-tokens-per-second
3. Cursor Team (2024). "Cursor Tab: Next Action Prediction with Custom Sparse Models." *Cursor Blog & Product Updates*. Available at: https://cursor.com/blog
4. Cursor Documentation (2024). "Context: Codebase Indexing and Semantic Search." *Cursor Official Documentation*. Available at: https://docs.cursor.com/context/codebase-indexing
5. Willison, S. (2025). "Cursor: Security and Vector Storage Architecture." *Simon Willison's Weblog*. Available at: https://simonwillison.net/2025/May/11/cursor-security/
6. Leviathan, Y., Kalman, M., & Matias, Y. (2023). "Fast Inference from Transformers via Speculative Decoding." *Proceedings of the 40th International Conference on Machine Learning (ICML)*. Available at: https://arxiv.org/abs/2211.17192
7. Charikar, M. S. (2002). "Similarity Estimation Techniques from Rounding Algorithms." *Proceedings of the thiry-fourth annual ACM symposium on Theory of computing (STOC)*. Available at: https://dl.acm.org/doi/10.1145/509907.509965

---

Authored by @saurabh-vision
