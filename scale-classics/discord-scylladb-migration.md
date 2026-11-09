---
title: "How Discord Moved Trillions of Messages from Cassandra to ScyllaDB"
description: "How Discord moved trillions of messages from Cassandra to ScyllaDB, slashing cluster size from 177 to 72 nodes and stabilizing read p99 latency to 15ms with Rust."
keywords:
  - discord scylladb migration
  - how discord stores trillions of messages
  - cassandra vs scylladb
  - cassandra hot partition problem
  - request coalescing rust data service
  - seastar framework
  - distributed database migration
date: 2026-10-30
last_verified: 2026-10-30
---

# How Discord Moved Trillions of Messages from Cassandra to ScyllaDB

> [!IMPORTANT]
> **Status Update: Architecture, Licensing, and Operational Incidents (2023–2026)**  
> Discord’s message storage and persistence tier continues to run on ScyllaDB, fronted by its custom Rust data services layer.
> - **ScyllaDB Licensing Shift (2025)**: ScyllaDB discontinued its separate open-source AGPL track (final AGPL release was **ScyllaDB 6.2.x**). Starting with **ScyllaDB 2025.1**, all versions unified under a **Source-Available License**: free for clusters under **10 TB of disk storage and 50 vCPUs**, requiring paid commercial enterprise agreements above those thresholds or when used in commercial DBaaS platforms (the underlying Seastar framework remains Apache 2.0).
> - **Production Incident Disclosures**: At ScyllaDB Summit 2024, Discord database architect Bo Ingram presented a transparent post-mortem (*"So You've Lost Quorum: Lessons From Accidental Downtime"*), detailing how a stressed ScyllaDB cluster lost quorum during rolling restarts on hybrid GCP persistent disk + local NVMe RAID storage when node synchronization lagged behind live client retries.

> [!NOTE]
> **TL;DR**
> - **Cluster Size Reduction**: Discord shrank its message database footprint from **177 Apache Cassandra nodes down to 72 ScyllaDB nodes** [1].
> - **Tail Read Latency Stabilization**: Historical message read 99th percentile ($\text{p99}$) latency dropped from an erratic 40ms–125ms+ on Cassandra to a consistent **15ms** on ScyllaDB [1].
> - **Write Latency Determinism**: Write $\text{p99}$ latencies stabilized from volatile 5ms–70ms swings down to a flat **5ms** under global peak traffic [1].
> - **Storage Density Increase**: Average storage density per node more than doubled, increasing from **4 TB per node to 9 TB per node** [1].
> - **Rapid Migration Pipeline**: A custom multithreaded migrator written in Rust safely transferred the dataset across Cassandra token ranges in **9 days** [1].

Chat applications operate under pathological storage access patterns: hundreds of millions of users concurrently send messages that must be appended instantaneously, yet 95% of reads query only the most recent 50 messages in active channels. In early 2022, Discord's message storage cluster ran on Apache Cassandra across 177 nodes storing trillions of messages, but the database became a high-toil operational burden plagued by Stop-The-World Java Virtual Machine (JVM) garbage collection pauses, compaction debt, and hot partitions. The math dictates that horizontally scaling a distributed database to trillions of records should require adding dozens of additional nodes and expanding on-call overhead, but Discord cut its physical cluster footprint by 59.3% while slashing tail latency by migrating to ScyllaDB and deploying an intermediate Rust data-service proxy.

---

## Production Scale and Workload Constraints

Storing trillions of messages at global scale imposes strict throughput, density, and latency boundaries.

| Metric / Dimension | Production Target / Observed Value | Source / Evidence |
| :--- | :--- | :--- |
| **Total Stored Messages** | "Trillions of messages" | Discord Engineering Blog [1] |
| **Cassandra Cluster Footprint** | 177 nodes (across multiple clusters) | Discord Engineering Blog [1] |
| **ScyllaDB Cluster Footprint** | 72 nodes | Discord Engineering Blog [1] |
| **Legacy Read Tail Latency ($\text{p99}$)** | 40ms – 125ms+ (erratic GC spikes) | Discord Engineering Blog [1] |
| **Modern Read Tail Latency ($\text{p99}$)** | 15ms (flat, deterministic) | Discord Engineering Blog [1] |
| **Modern Write Tail Latency ($\text{p99}$)** | 5ms | Discord Engineering Blog [1] |
| **Storage Density per Node** | Increased from 4 TB/node to 9 TB/node | Discord Engineering Blog [1] |
| **Cluster Reduction Factor** | 59.3% node reduction (derived: $\frac{177-72}{177}$) | Derived from Discord Blog [1] |
| **Migration Duration** | 9 days across Cassandra token ring | Discord Engineering Blog [1] |
| **Proxy Architecture** | Custom Rust Data Services (gRPC) | Discord Engineering Blog [1, 3] |
| **Primary Partitioning Key** | `PRIMARY KEY ((channel_id, bucket), message_id)` | Discord Engineering Blog [1, 2] |

*(Terminology note: Discord's official engineering blog explicitly states "trillions of messages." The specific figure "4 trillion" reported in secondary publications is an unverified industry estimate. This case study adheres to Discord's primary documented phrasing: "trillions of messages").*

### The Operational Burden of Cassandra at Scale

Before migrating, Discord's Cassandra clusters suffered from four structural failure modes:

1. **JVM Garbage Collection (GC) Pauses**: Cassandra runs on the Java Virtual Machine. When high-volume read queries allocated large result sets into memory, the JVM's generational garbage collectors (such as G1GC) triggered Stop-The-World pauses lasting hundreds of milliseconds to several seconds. During these pauses, nodes missed intra-cluster gossip heartbeat pings, causing peers to mark healthy nodes as dead and triggering cascading query retries across the ring.
2. **Compaction Debt and Write Amplification**: Cassandra uses a Log-Structured Merge-tree (LSM) storage engine. Writes are appended to in-memory Memtables and flushed to on-disk SSTables (Sorted String Tables). Background compaction merges SSTables to discard superseded data. Under relentless message ingestion, Cassandra fell behind on compaction, causing SSTable counts to explode, which amplified read latency by forcing single read queries to seek across dozens of discrete files.
3. **The Hot Partition Problem**: In popular Discord guilds (servers with hundreds of thousands of members), channels like `#general` receive extreme read and write traffic. If a streamer announces an event, thousands of users open the channel simultaneously. In Cassandra, all reads for that channel hit the single replica set holding that partition's token range, saturating the host CPU and memory buses.
4. **Tombstone Accumulation**: In wide-column databases, deletions do not overwrite disk blocks immediately; they append deletion markers called *tombstones*. When users or moderation bots purged thousands of messages, dense ranges of tombstones formed inside partitions. Reading past these tombstones forced Cassandra to scan hundreds of thousands of dead records, exhausting the JVM heap and triggering read timeouts (`ReadTimeoutException`).

---

## End-to-End System Architecture

Discord decouples client traffic from physical database nodes by inserting an intermediate routing and coalescing layer written in Rust between the API monolith and ScyllaDB.

```mermaid
flowchart TD
    subgraph ClientTier["Client Ingress"]
        ClientApp["Discord Client (Desktop / Mobile / Web)"]
        Gateway["Elixir Gateway (WebSockets / Presence)"]
    end

    subgraph APITier["Application Monolith"]
        Monolith["Python API Monolith (Async Workers)"]
    end

    subgraph ProxyTier["Rust Data Services Tier (Consistent Hash Routed)"]
        HashRing["Consistent Hash Ring (Keyed by channel_id)"]
        subgraph WorkerNode["Data Service Instance"]
            Coalescer["Request Coalescer (Singleflight Engine)"]
            InflightRegistry["In-Flight Promise Registry (Tokio)"]
            Coalescer <--> InflightRegistry
        end
    end

    subgraph LegacyPath["Decommissioned Storage Path"]
        CassandraCluster["Apache Cassandra (177 Nodes)\nJVM Garbage Collection / Compaction Debt"]
    end

    subgraph ModernStorage["Modern Storage Tier: ScyllaDB (72 Nodes)"]
        ScyllaNode1["ScyllaDB Node 1 (C++ / Seastar)"]
        ScyllaNode2["ScyllaDB Node 2 (Direct I/O NVMe RAID)"]
        ScyllaNodeN["ScyllaDB Node N (Shard-per-Core)"]
    end

    ClientApp --> Gateway
    Gateway --> Monolith
    Monolith -.->|Legacy CQL Direct Driver (Decommissioned)| CassandraCluster
    Monolith -->|gRPC Request| HashRing
    HashRing --> WorkerNode
    WorkerNode -->|Single De-duplicated CQL Query| ModernStorage
    ScyllaNode1 --- ScyllaNode2 --- ScyllaNodeN

    classDef default fill:#1e293b,stroke:#475569,stroke-width:1px,color:#f8fafc;
    classDef highlight fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#38bdf8;
    classDef legacy fill:#3f1717,stroke:#ef4444,stroke-width:1px,color:#fca5a5;
    class Coalescer,InflightRegistry,ScyllaNode1,ScyllaNode2 highlight;
    class CassandraCluster legacy;
```

### Architectural Walkthrough

1. **Client Gateway**: Real-time client WebSocket connections terminate on an Elixir-based gateway cluster that manages session presence and event distribution [1, 2].
2. **API Monolith**: Message creation and query requests route to Discord's Python API monolith. Rather than opening thousands of direct database connections, the monolith issues gRPC calls to an intermediate proxy service [1].
3. **Consistent Hash Routing**: Monolith requests are routed to specific Rust Data Service instances using a consistent hash ring keyed by `channel_id`. This guarantees that all concurrent queries for a given channel converge on the exact same proxy instance [1].
4. **Request Coalescing (Singleflighting)**: Inside the Rust Data Service, if multiple requests arrive simultaneously for the same message bucket, only the first request queries ScyllaDB. Subsequent callers subscribe to the active in-flight future and receive the result concurrently, shielding the database from dogpiling [1].
5. **Shard-per-Core ScyllaDB Engine**: ScyllaDB receives CQL queries and maps them directly to the CPU core managing the partition's token range via the Seastar asynchronous framework, utilizing Direct I/O (`O_DIRECT`) to bypass the Linux page cache and read from local NVMe SSDs [1, 4].

---

## Core Math: Partition Sizing, Request Coalescing, and Capacity Math

### 1. Partition Sizing Math: Preventing "Super Partitions"

Cassandra and ScyllaDB performance degrades when an individual partition exceeds **100 MB** or contains more than **100,000 cells**. Discord structures its schema using a compound partition key combining `channel_id` and a static time `bucket` [1, 2]:

```sql
CREATE TABLE messages (
    channel_id bigint,
    bucket int,
    message_id bigint,
    author_id bigint,
    content text,
    PRIMARY KEY ((channel_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

The bucket duration is configured to approximately **10 days** ($864{,}000{,}000\text{ ms}$) [2].

#### Partition Growth Derivation
Let:
- $R_m$: Average message rate in messages per second ($\text{msg/sec}$).
- $S_m$: Average stored message payload size, assumed at $250\text{ bytes}$ (text content, Snowflake IDs, metadata overhead).
- $T_{\text{bucket}}$: Bucket duration in seconds ($10 \text{ days} = 864{,}000 \text{ seconds}$).

The estimated uncompressed partition size $S_{\text{partition}}$ for a single channel bucket is:

$$S_{\text{partition}} = R_m \times T_{\text{bucket}} \times S_m \quad \text{(derived)}$$

#### Scenario A: Standard Direct Message or Small Channel
- Message rate: $R_m = 0.001\text{ msg/sec}$ (~86 messages / day).
  $$S_{\text{partition}} = 0.001 \times 864{,}000 \times 250\text{ B} = 216{,}000\text{ bytes} \approx \mathbf{0.21 \text{ MB}} \quad \text{(derived)}$$
  The partition remains small, localized, and well below the 100 MB threshold.

#### Scenario B: High-Volume Community Guild (`#general` in a 500k-member Server)
- Message rate: $R_m = 2.0\text{ msg/sec}$ sustained during peak hours (~172,800 messages / day).
  $$S_{\text{partition}} = 2.0 \times 864{,}000 \times 250\text{ B} = 432{,}000{,}000\text{ bytes} \approx \mathbf{412 \text{ MB}} \quad \text{(derived)}$$
  Without bucketing, a channel running for two years at this rate would accumulate:
  $$S_{\text{unbucketed}} = 2.0 \times (730 \times 86{,}400) \times 250\text{ B} \approx \mathbf{31.5 \text{ GB}} \quad \text{(derived)}$$
  A single 31.5 GB partition creates a "super partition" that exhausts memory buffers during SSTable compaction and triggers massive GC pauses during reads. By slicing data into 10-day buckets, Discord limits partition size growth to manageable increments, rotating writes onto fresh partitions over time [1, 2].

---

### 2. Request Coalescing: Load Reduction Math and Implementation

When an @everyone announcement occurs in a busy server with 50,000 active users, thousands of clients query the same message history simultaneously.

#### Coalescing Load Reduction Math
Let $N$ represent the number of concurrent read queries arriving within a database query execution window $T_{\text{query}} \approx 15\text{ms}$.
- **Without Coalescing**: ScyllaDB receives $N$ independent CQL queries:
  $$Q_{\text{database}} = N$$
- **With Coalescing**: Exactly 1 query hits ScyllaDB; $N-1$ queries await the result:
  $$Q_{\text{database}} = 1$$
  $$\text{Database Load Reduction Factor} = \frac{N - 1}{N} \times 100\% \quad \text{(derived)}$$
  For $N = 500$ concurrent callers, database query pressure on that partition is reduced by:
  $$\text{Load Reduction} = \frac{499}{500} \times 100\% = \mathbf{99.8\%} \quad \text{(derived)}$$

#### Production Request Coalescing Pseudo-Code (Rust-Style)

```rust
// Language-neutral / Idiomatic Rust Singleflight Implementation
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{broadcast, Mutex};

#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct ChannelBucketKey {
    pub channel_id: u64,
    pub bucket: u32,
}

#[derive(Clone, Debug)]
pub struct MessageChunk {
    pub messages: Vec<u8>, // Serialized CQL result buffer
}

pub struct RequestCoalescer {
    // Registry of in-flight physical database requests
    in_flight: Arc<Mutex<HashMap<ChannelBucketKey, broadcast::Sender<MessageChunk>>>>,
}

impl RequestCoalescer {
    pub fn new() -> Self {
        Self {
            in_flight: Arc::new(Mutex::new(HashMap::new())),
        }
    }

    pub async fn query_channel_messages(
        &self,
        key: ChannelBucketKey,
    ) -> Result<MessageChunk, String> {
        let mut map = self.in_flight.lock().await;

        // Step 1: If an identical query is already executing, subscribe to its broadcast channel
        if let Some(sender) = map.get(&key) {
            let mut receiver = sender.subscribe();
            drop(map); // Release mutex lock before awaiting async I/O
            return receiver.recv().await.map_err(|e| e.to_string());
        }

        // Step 2: First caller becomes the leader; register broadcast channel
        let (sender, mut receiver) = broadcast::channel(1);
        map.insert(key.clone(), sender.clone());
        drop(map); // Release lock while executing physical storage query

        // Step 3: Execute single physical read against ScyllaDB
        let result = self.execute_scylla_query(&key).await;

        // Step 4: Remove task from in-flight registry
        let mut map = self.in_flight.lock().await;
        map.remove(&key);
        drop(map);

        // Step 5: Broadcast result to all concurrent subscribers (N-1 callers)
        let _ = sender.send(result.clone());
        Ok(result)
    }

    async fn execute_scylla_query(&self, key: &ChannelBucketKey) -> MessageChunk {
        // Physical CQL execution against ScyllaDB cluster via driver
        MessageChunk { messages: vec![] }
    }
}
```

---

### 3. Consistent-Hash Routing: Enforcing Convergence on Proxy Nodes

Request coalescing only functions if concurrent queries for `channel_id = X` route to the **same** proxy instance. If queries are round-robined across 20 proxy nodes, each node would issue an independent query to ScyllaDB, reducing coalescing efficiency by 95%.

```
                         [Hash Ring (Murmur3 / Ketama)]
                                      │
                   ┌──────────────────┼──────────────────┐
                   ▼                  ▼                  ▼
             Proxy Node A       Proxy Node B       Proxy Node C
            (Tokens 0 - 2^21) (Tokens 2^21 - 2^42) (Tokens 2^42 - 2^64)
                   │
                   ▼  Hash(channel_id = 99281) maps to Node A
             [Single In-Flight Task Collapses 500 Queries into 1]
```

#### Hash Ring Routing Algorithm Pseudo-Code

```python
# Consistent Hash Ring Implementation for Channel Routing
import bisect
import hashlib

class ConsistentHashRouter:
    def __init__(self, nodes=None, virtual_nodes=100):
        self.virtual_nodes = virtual_nodes
        self.ring = []  # Sorted list of virtual node token hashes
        self.node_map = {}  # token -> node IP

        for node in (nodes or []):
            self.add_node(node)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode('utf-8')).hexdigest(), 16)

    def add_node(self, node: str):
        for i in range(self.virtual_nodes):
            vnode_key = f"{node}#VN{i}"
            token = self._hash(vnode_key)
            self.ring.append(token)
            self.node_map[token] = node
        self.ring.sort()

    def get_proxy_instance(self, channel_id: int) -> str:
        if not self.ring:
            raise Exception("No proxy nodes registered in hash ring")
        token = self._hash(str(channel_id))
        idx = bisect.bisect_right(self.ring, token)
        if idx == len(self.ring):
            idx = 0  # Wrap around the circular ring
        return self.node_map[self.ring[idx]]
```

---

### 4. Capacity and Storage Density Math

Discord’s published metrics document a cluster reduction from **177 Cassandra nodes** to **72 ScyllaDB nodes**, while per-node storage density increased from **4 TB to 9 TB** [1].

#### Cluster Data Capacity Derivation
Let:
- $N_{\text{Cassandra}} = 177$ nodes
- $D_{\text{Cassandra}} = 4\text{ TB/node}$
- $N_{\text{Scylla}} = 72$ nodes
- $D_{\text{Scylla}} = 9\text{ TB/node}$

$$\text{Capacity}_{\text{Cassandra}} = 177 \times 4\text{ TB} = \mathbf{708 \text{ TB}} \quad \text{(derived)}$$

$$\text{Capacity}_{\text{Scylla}} = 72 \times 9\text{ TB} = \mathbf{648 \text{ TB}} \quad \text{(derived)}$$

#### Accounting for Replication Factor ($\text{RF}$)
*(Note: Discord does not explicitly state its replication factor in the March 2023 blog post, but standard production Cassandra and ScyllaDB deployments use a Replication Factor of 3 across availability zones).*

Assuming $\text{RF} = 3$:

$$\text{Unique Dataset Size} \approx \frac{648\text{ TB}}{3} = \mathbf{216 \text{ TB of unique raw messages}} \quad \text{(derived)}$$

The 59.3% reduction in node count was made possible because ScyllaDB’s C++ shard-per-core engine manages local NVMe SSDs without suffering from JVM heap garbage collection limits. Cassandra nodes are typically capped at 2–4 TB of disk storage per node because larger disk footprints cause compaction storms and heap memory exhaustion during repair sweeps [4, 5]. By removing the JVM, ScyllaDB safely supported 9 TB of disk density per node [1].

---

## Production Post-Mortems and System Failure Modes

### Incident 1: The Cassandra "Gossip Dance" and Compaction Outage

#### What Broke
In early 2022, Discord's on-call database engineers were paged repeatedly due to cluster-wide Cassandra latency spikes. Read latencies escalated past 1,000ms, query error rates climbed, and nodes repeatedly crashed or were marked down by cluster peers [1].

#### Root Cause Analysis
The outage was caused by a compounding failure loop between JVM garbage collection, tombstone scans, and Cassandra's gossip protocol:
1. Active channels experienced bulk message purges, creating millions of tombstones across contiguous SSTables.
2. When users loaded these channels, Cassandra scanned hundreds of thousands of tombstones, allocating millions of temporary Java objects into the JVM heap.
3. This triggered Stop-The-World G1GC pauses lasting over 3,000ms.
4. While the JVM paused, the node failed to respond to gossip heartbeats. Surrounding nodes marked it dead and took over its read/write traffic.
5. When the node emerged from GC, it was bombarded with queued requests, immediately triggering another GC pause. Operators were forced to execute a manual "gossip dance"—pulling nodes out of cluster rotation, executing manual compactions, and nursing them back into the ring [1].

#### The Architectural Fix
Discord migrated to ScyllaDB, which eliminates the JVM entirely. ScyllaDB is written in C++ using the Seastar framework, managing memory explicitly on a per-CPU-core basis. Furthermore, Discord added an empty bucket cache in the application layer, short-circuiting queries to partitions known to contain zero active messages [1].

---

### Incident 2: The 99.999% Migration Stalling Incident

#### What Broke
During the live migration of trillions of messages, Discord's custom Rust migrator successfully processed 99.9% of the dataset at high sustained throughput. However, at 99.999% completion, the migration stalled. A small group of token ranges consistently timed out when read from Cassandra, preventing the cutover [1].

#### Root Cause Analysis
The migrator split Cassandra's 64-bit token ring ($[-2^{63}, 2^{63}-1]$) into discrete ranges and iterated through them using SQLite for local checkpointing. The final token ranges happened to encompass ancient, abandoned channels that had experienced automated deletions years prior. Because these inactive partitions received zero new writes, Cassandra’s Size-Tiered Compaction Strategy had never merged their SSTables. When the migrator queried these token ranges with wide page limits, Cassandra choked on the stale tombstones and dropped the connections [1].

#### The Architectural Fix
Using the local SQLite task database, Discord isolated the specific failing token ranges and executed targeted `nodetool compact` commands directly on those Cassandra nodes. This forced the physical merging of SSTables and purged the expired tombstones, allowing the Rust migrator to finish the remaining 0.001% of the dataset in minutes [1].

---

### Incident 3: The ScyllaDB Quorum Loss Outage (Nov 6, 2023 / Summit 2024 Disclosure)

#### What Broke
On November 6, 2023, Discord experienced an unexpected availability incident affecting its ScyllaDB authentication cluster. A rolling restart intended as routine maintenance resulted in a loss of cluster quorum (`LOCAL_QUORUM` failures), causing widespread user login and session authentication errors across the platform [3].

#### Root Cause Analysis
In his ScyllaDB Summit 2024 presentation (*"So You've Lost Quorum: Lessons From Accidental Downtime"*), Discord database architect Bo Ingram detailed the cascading mechanism:
1. Discord runs ScyllaDB on Google Cloud Platform (GCP) using a hybrid storage topology: local NVMe SSDs configured in a RAID-1 mirror with persistent disks (PD) to combine raw NVMe I/O performance with persistent disk durability.
2. During the rolling node restarts, nodes rebooting in a specific availability zone experienced startup delays while resynchronizing local NVMe caches from persistent disk blocks.
3. While these nodes were recovering, un-throttled client traffic and aggressive upstream API retries flooded the surviving nodes.
4. The remaining nodes saturated their CPU execution queues, causing inter-node communication timeouts. Because too many nodes were delayed simultaneously, the cluster fell below the majority quorum threshold required to acknowledge reads and writes [3].

#### The Architectural Fix
Discord implemented stricter operational safeguards:
1. **Dynamic Client Concurrency Throttling**: Upstream services implement exponential backoff with jitter, preventing retry storms from overwhelming a recovering database cluster.
2. **Zone-Aware Rolling Maintenance Limits**: Rolling node restarts enforce strict concurrency gates; a node cannot restart until the previous node fully completes its storage synchronization and rejoins quorum.
3. **Queue-Aware Priority Shedding**: ScyllaDB's native workload prioritization was configured to shed non-essential background tasks during periods of node recovery.

---

## Architectural Trade-offs and What Discord Gave Up

Migrating to ScyllaDB and deploying a custom Rust data-services tier resolved operational toil, but introduced distinct engineering costs:

### 1. Database-Level Performance vs. Upstream Application Responsibility
- **The Reality**: While ScyllaDB eliminates JVM GC pauses and improves raw hardware throughput via Seastar, **ScyllaDB alone did not solve hot partitions** [1]. 
- **The Trade-off**: If 50,000 users query `#general` simultaneously, a standalone ScyllaDB cluster will still suffer from node-level hot spotting. Discord had to design, deploy, and maintain an entire intermediate tier of custom Rust Data Services to execute request coalescing and consistent-hash routing. A significant portion of Discord’s architectural stability stems from this custom proxy layer, not the database swap in isolation.

### 2. Standardized Ecosystem vs. Specialized Vendor Architecture
- **The Reality**: Cassandra boasts a massive global open-source community, mature documentation, and deep tooling integration (Spark connectors, Cassandra reaper, generic backup managers).
- **The Trade-off**: Moving to ScyllaDB requires specialized knowledge of C++, Linux asynchronous I/O (`io_uring`), and Seastar thread-per-core debugging. Furthermore, ScyllaDB's 2025 licensing shift away from the AGPL to a Source-Available model means large-scale operators must maintain commercial enterprise relationships once cluster sizes exceed 10 TB.

### 3. Hardware Lock-in to High-Performance Local NVMe
- **The Reality**: ScyllaDB achieves its throughput by issuing Direct I/O calls directly against physical disk blocks.
- **The Trade-off**: Running ScyllaDB cost-effectively requires locally attached NVMe storage arrays. Running on standard cloud network-attached storage (e.g., standard AWS EBS or GCP Persistent Disk without local caching) introduces IOPS bottlenecks and network latency variance that negate Seastar's core throughput advantages.

---

## When Not to Copy Discord's Storage Architecture

Engineers frequently cargo-cult Discord's database migration without examining their own operational scale:

```
                            [Total Message / Row Scale?]
                                         │
                       ┌─────────────────┴─────────────────┐
                       ▼                                   ▼
                 < 100 Million                       > 100 Billion
                       │                                   │
         ┌─────────────┴─────────────┐                     │
         ▼                           ▼                     ▼
[Complex Queries / ACID?]   [Simple Key-Value?]     [Team Capable of Custom
         │                           │               Proxy Infrastructure?]
         ▼                           ▼               ┌─────┴─────┐
Use PostgreSQL / MySQL      Use Managed DynamoDB     ▼           ▼
Single-Node or Read Repl.   or Firestore            No          Yes
                                                     │           │
                                                     ▼           ▼
                                                Managed NoSQL  ScyllaDB +
                                                (Cassandra /   Rust Proxy
                                                 Scylla Cloud) (Discord Model)
```

### 1. Datasets Smaller Than 100 Million Records
If your application stores fewer than 100 million records, running a 72-node distributed wide-column cluster is massive over-engineering. A single partitioned PostgreSQL instance with read replicas or a managed cloud database (DynamoDB) provides sub-10ms queries with zero cluster administration overhead.

### 2. Teams Without Resources to Maintain a Custom Proxy Tier
Discord’s stability relies heavily on its Rust Data Services layer performing consistent-hash routing and request coalescing. If your engineering team does not have the systems programming bandwidth to build and maintain high-performance stateful middleware, deploying ScyllaDB directly will leave your system vulnerable to hot-partition dogpiling.

### 3. Applications Requiring Ad-Hoc Filtering and Table Joins
Wide-column stores (Cassandra, ScyllaDB) enforce strict query paths based entirely on the primary key. You cannot perform cross-table joins, arbitrary column filters, or dynamic sorting. If your application requirements include rich search or ad-hoc analytics, relational databases or dedicated search engines (Elasticsearch, Vespa) are mandatory.

---

## Frequently Asked Questions About Discord's Storage Architecture

### Why did Discord switch from Cassandra to ScyllaDB?
Discord migrated from Cassandra to ScyllaDB primarily to eliminate JVM Garbage Collection (GC) pauses and reduce operational maintenance toil [1]. Cassandra’s Java runtime caused unpredictable `p99` latency spikes (40ms to over 125ms+) and node instability during compaction and tombstone scans. ScyllaDB is written in C++ using a shard-per-core architecture that bypasses the JVM entirely, allowing Discord to stabilize read tail latencies at 15ms while reducing cluster hardware from 177 nodes to 72 nodes [1].

### Did moving to ScyllaDB fix Discord's hot partition problem?
No. ScyllaDB provides faster hardware execution, but hot partitions are a data-modeling and traffic-pattern problem that no distributed storage engine can solve alone [1]. Discord resolved hot partitions at the application layer by placing an intermediate Rust data service between the API monolith and ScyllaDB. This proxy uses consistent hashing on `channel_id` to route requests to the same worker, executing request coalescing (singleflighting) to collapse thousands of concurrent reads into a single database query [1].

### How does Discord partition chat messages?
Discord partitions messages using a compound primary key: `PRIMARY KEY ((channel_id, bucket), message_id)` [1, 2]. The `bucket` is derived from the message Snowflake ID, dividing a channel's messages into static ~10-day time windows ($864{,}000{,}000\text{ ms}$). This prevents high-volume channels from growing into massive "super partitions" that degrade database performance over time [2].

### What is the role of Elixir in Discord's storage layer?
Elixir does not handle message storage persistence directly [1]. Elixir powers Discord's client-facing WebSocket Gateway, managing persistent socket connections, presence updates, and real-time message fanout across hundreds of millions of users. Persistence queries originate from Discord's Python API monolith, which issues gRPC calls to the Rust data services down to ScyllaDB [1].

### How did Discord migrate trillions of messages with zero downtime?
Discord migrated its data using a custom multithreaded migrator written in Rust [1]. The migrator divided the 64-bit Cassandra token ring into discrete ranges and tracked progress locally in an SQLite database. Using live dual-writing (writing to both Cassandra and ScyllaDB) and background read verification, the migrator safely backfilled trillions of historical records across ScyllaDB in 9 days without impacting live users [1].

---

## References

1. Ingram, B. (2023). "How Discord Stores Trillions of Messages." *Discord Engineering Blog*. Available at: https://discord.com/blog/how-discord-stores-trillions-of-messages
2. Howarth, J. (2017). "How Discord Stores Billions of Messages." *Discord Engineering Blog*. Available at: https://discord.com/blog/how-discord-stores-billions-of-messages
3. Ingram, B. (2024). "So You've Lost Quorum: Lessons From Accidental Downtime." *ScyllaDB Summit 2024 Presentation*. Available at: https://www.scylladb.com/summit/
4. ScyllaDB Documentation. (2024-2025). "Architecture: The Seastar Framework & Shard-per-Core Execution." *ScyllaDB Docs*. Available at: https://docs.scylladb.com/manual/architecture/
5. Apache Cassandra Project. (2023). "Apache Cassandra Documentation: Architecture, Compaction, and Tombstones." *Apache Software Foundation*. Available at: https://cassandra.apache.org/doc/latest/
6. Ingram, B. (2023). *ScyllaDB in Action*. Manning Publications. ISBN: 9781617299971.

---

Authored by @saurabh-vision
