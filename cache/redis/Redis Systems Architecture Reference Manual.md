* # Redis Systems Architecture Manual

  ---

  ### High-Level Architecture Directory

  | Section                                                      | Architectural Domain       | Core Under-the-Hood Logic                                    |
  | :----------------------------------------------------------- | :------------------------- | :----------------------------------------------------------- |
  | **[1. Core Foundations (Memory & Commands)](#1-core-foundations)** | Key-Value Storage          | Keyspaces, execution of commands, data granularity, and logical namespaces [codeburst.io] |
  | **[2. Execution & Concurrency Model](#2-execution-model)**   | Processing Engine          | Single-threaded core logic, why network multi-threading exists, and the barrier mechanism [dragonflydb.io, strikefreedom.top] |
  | **[3. Storage & Durability Strategy](#3-storage-durability)** | Persistence Layer          | Periodic point-in-time snapshot logic versus real-time change logging [redis.io] |
  | **[4. Data Replication & Sync Engine](#4-replication-sync)** | Multi-Node Synchronization | Offset tracking, lineage matching, fast vs. slow sync paths, and memory optimizations [systeminternals.dev, redisgate.jp] |
  | **[5. High-Availability & Clustering](#5-clustering)**       | Distributed Scaling        | Active-Passive read scaling, Multi-Region Active-Active, and sharding via hash slots [oneuptime.com, redis.io] |
  | **[6. Data Consistency & Safety Controls](#6-consistency-safety)** | Integrity Protections      | Stale-read hazards, synchronous replication, and split-brain mitigation [redisgate.jp] |
  | **[7. Detailed Technical Mechanics (Glossary)](#7-glossary)** | low-level Details          | Internal structures (`redisDb`, `redisObject`), incremental rehashing, and replication buffer blocks [codeburst.io, redisgate.jp] |
  | **[8. Appendix: Auxiliary Technologies](#8-appendix)**       | Non-Redis Protocols        | [I/O Multiplexing (epoll)](#app-multiplexing), [DNS Anycast](#app-anycast), [BGP](#app-bgp), [Spinlocks vs. Mutexes](#app-spinlocks), [Vector Clocks](#app-vclocks), [Copy-on-Write](#app-cow) |

  ---

  <a id="1-core-foundations"></a>
  ### 1. Core Foundations (Memory & Commands)

  To understand how Redis functions under the hood, we must look at how it maps logical data types into memory. Redis is an in-memory, key-value storage engine designed to perform lookups and mutations with predictable, sub-millisecond speeds [codeburst.io].

  ```
                       ┌────────────────── redisDb ──────────────────┐
                       │                                             │
                       ├── dict (Main Keyspace)                      └── expires (TTL Space)
                       │    └── dictEntry                                 └── dictEntry
                       │         ├── Key (SDS String)                         ├── Key (SDS String)
                       │         └── Value (redisObject)                      └── Value (Expiry Timestamp)
  ```

  #### The Dual-Keyspace Model
  Each database in Redis is split into two logical hash tables [codeburst.io]:
  1.  **The Main Keyspace:** A dictionary mapping unique string keys to [encapsulated Redis value objects](#glos-redisobject) [codeburst.io].
  2.  **The Expiry Space:** A dictionary mapping string keys to absolute expiration timestamps. Placing expire metadata in a separate table ensures that keys without TTLs do not incur memory overhead.

  To prevent lookup bottlenecks, keys are stored as [Simple Dynamic Strings (SDS)](#glos-sds), which cache their own lengths to eliminate the overhead of scanning memory [codeburst.io]. 

  #### Command Granularity & Namespace Rules
  *   **Command Granularity:** Most Redis commands are single-key (e.g., `GET`, `HSET`), meaning they target a single value object [codeburst.io, Surfin.sg]. However, Redis natively supports multi-key "bulk" updates (like `MSET` or `DEL`) [Surfin.sg]. These bulk updates run atomically; because the database core is single-threaded, no other client can modify the targeted keys mid-execution [strikefreedom.top, Surfin.sg].
  *   **Logical Databases:** Redis provides numerically indexed isolated namespaces (databases `0-15`) [Wjin.org]. Commands like `SELECT <db>` switch the active namespace [Wjin.org]. However, when running in a distributed Cluster, Redis disables these logical databases and supports only database `0` to keep data routing simple and predictable [Wjin.org].

  #### Programmability via Lua Scripts
  To execute complex, multi-step transaction logic without network round-trips, Redis embeds a Lua execution engine. 
  *   **The Logic:** When a client sends a Lua script, the main thread stops parsing incoming connections and executes the script to completion. 
  *   **The Concept:** This guarantees that the entire script runs as a single, atomic operation. It reduces network latency (eliminating client-to-server wait times) by running the application logic directly on the database node.

  #### Real-time Monitoring via Keyspace Notifications
  Redis features an event-driven notification system built directly on top of its Pub/Sub engine.
  *   **The Logic:** When a write command modifies the database, internal hooks publish messages to two channels: **Keyspace** (notifying subscribers *what happened* to a specific key) and **Keyevent** (notifying subscribers *which keys* experienced a specific action).
  *   **The Concept:** This system is entirely stateless (fire-and-forget). If a monitoring client disconnects, any notifications generated during its downtime are lost forever.

  ---

  <a id="2-execution-model"></a>
  ### 2. Execution & Concurrency Model

  The core execution engine of Redis is strictly **single-threaded** [dragonflydb.io]. This design choice is a deliberate trade-off: since accessing RAM takes nanoseconds, the primary bottlenecks are network bandwidth and CPU context-switching, not raw CPU processing power [dragonflydb.io]. Running sequentially on a single thread allows Redis to completely avoid the complexity, lock overhead, and latency spikes associated with multi-threaded databases [dragonflydb.io].

  To scale with modern high-speed networks, Redis delegates intensive network serialization to background worker threads, keeping command execution on the main thread [strikefreedom.top].

  #### The Multi-Threaded I/O Pipeline
  ```mermaid
  sequenceDiagram
      autonumber
      participant Epoll as OS Multiplexer (epoll)
      participant Main as Main Thread
      participant Workers as I/O Workers (Shared Pool)
      participant DB as Core Keyspace (Memory)
  
      Epoll->>Main: Read event triggered on client sockets
      Main->>Main: Postpone read and queue client contexts
      Main->>Workers: Fan-out: Assign clients (Round-Robin)
      activate Workers
      Workers->>Workers: Read raw TCP stream & parse commands into client buffers
      Workers->>Main: Signal completion (Clear pending flag)
      deactivate Workers
      Main->>Main: Lock-free Busy-Wait Spinlock (Wait for workers)
      Main->>DB: Fan-in: Execute commands sequentially (FIFO)
      DB-->>Main: Return command outcomes
      Main->>Workers: Fan-out: Assign client write operations
      activate Workers
      Workers->>Workers: Format RESP response and write bytes to socket
      Workers->>Main: Signal completion
      deactivate Workers
      Main->>Main: Lock-free Busy-Wait Spinlock
  ```

  #### Under-the-Hood Logic
  1.  **Network Multiplexing:** The main thread monitors all client connections using an event loop driven by [I/O Multiplexing (epoll)](#app-multiplexing) [dragonflydb.io].
  2.  **Parallel Read & Parse (Fan-Out):** When a batch of client requests arrive, the main thread postpones reading the sockets [strikefreedom.top]. Instead, it queues the client contexts and distributes them across a pool of background I/O workers [strikefreedom.top]. These workers concurrently read the raw network bytes and parse them into command arguments inside the client's buffer [strikefreedom.top].
  3.  **The Synchronization Barrier:** Once the tasks are distributed, the main thread enters a lock-free [Busy-Wait Spinlock](#app-spinlocks) [strikefreedom.top]. It polls atomic status flags until every worker thread has finished parsing its assigned commands [strikefreedom.top].
  4.  **Single-Threaded Execution (Fan-In):** With all commands parsed and frozen in memory, the main thread sequentially executes them against the in-memory keyspace [strikefreedom.top]. Since execution is isolated to this single thread, no database locks or mutexes are required [dragonflydb.io, strikefreedom.top].
  5.  **Parallel Write (Fan-Out):** After execution, the output responses are generated and queued [strikefreedom.top]. The main thread hands these back to the worker threads, which format and write the network bytes back to client sockets concurrently [strikefreedom.top].

  #### Why the Synchronization Barrier is Required
  The lock-free spinlock barrier is logically mandatory to enforce three core invariants before database mutations begin [strikefreedom.top]:
  *   **Command Ordering:** Client connections are pinned to specific I/O threads [strikefreedom.top]. The barrier guarantees that all pipelined commands from each client are fully parsed and "frozen" in their exact chronological sequence before execution [strikefreedom.top]. Without it, concurrent parsing would allow commands from different clients to slip past each other, destroying transactional consistency and deterministic execution.
  *   **Race Prevention:** The barrier ensures all background threads are completely idle during the execution phase [strikefreedom.top]. Because only the main thread is active, it can modify keyspaces and global metrics with absolute safety, completely eliminating the need for database locks or mutexes [dragonflydb.io, strikefreedom.top].
  *   **Allocator Isolation:** Both parsing (resizing buffers) and executing (allocating keys) require heavy memory allocation [strikefreedom.top]. Separating these steps with a barrier prevents background thread allocations from colliding with main thread allocations in the system memory allocator (`Jemalloc`), avoiding lock contention and maintaining high throughput [strikefreedom.top].

  ---

  <a id="3-storage-durability"></a>
  ### 3. Storage & Durability Strategy

  Because RAM is volatile, Redis provides two distinct persistence mechanisms to ensure database state can survive crashes or restarts [redis.io].

  ```
                    ┌─────────────── Redis Memory ───────────────┐
                    │                                            │
                    ▼ (forks child)                              ▼ (appends changes)
           [ RDB Snapshotting ]                         [ AOF Transaction Logging ]
                    │                                            │
                    ▼                                            ▼
             [ binary.rdb ]                            [ Multi-Part AOF Directory ]
        (Compressed Point-In-Time)                               │
                                              ┌──────────────────┼──────────────────┐
                                              ▼                  ▼                  ▼
                                       [ base.rdb ]       [ incremental.aof ]  [ manifest ]
  ```

  #### Under-the-Hood Logic
  *   **RDB (Redis Database) Snapshotting:**
      *   **The Concept:** RDB is a compact, point-in-time binary snapshot of your entire database [redis.io].
      *   **The Logic:** To generate a snapshot without blocking clients, Redis forks a background child process [redis.io]. Using the operating system's [Copy-on-Write (COW)](#app-cow) memory pages, the child process writes the database state to a file (`dump.rdb`) while the main thread continues modifying the database in memory [redis.io].
  *   **AOF (Append-Only File) Transaction Logging:**
      *   **The Concept:** AOF is a continuous, fine-grained transaction log of every state-modifying write command [redis.io].
      *   **The Logic:** Every write command executed by the main thread is appended to an in-memory buffer [redis.io]. Based on the configured write policy (fsync), this buffer is flushed to disk (typically once per second) [redis.io].
      *   **Multi-Part AOF Evolution:** To prevent performance degradation during database compactions (AOF rewrites), modern Redis uses a [Multi-Part AOF](#glos-mpaof) structure [redis.io]. It splits the log into a baseline binary snapshot and lightweight incremental transaction logs [redis.io]. This eliminates the heavy disk I/O and memory spikes associated with rewriting a single, massive log file [redis.io].

  ---

  <a id="4-replication-sync"></a>
  ### 4. Data Replication & Sync Engine

  Replication in Redis allows database updates to propagate asynchronously to replicas, enabling horizontal read scaling and high availability [oneuptime.com].

  #### Lineage & Offset Tracking
  To track replication state, the master and its replicas maintain two variables [systeminternals.dev]:
  1.  **Replication ID (`replid`):** A unique identifier representing the active dataset's lineage [systeminternals.dev].
  2.  **Replication Offset (`offset`):** A monotonically increasing byte counter [systeminternals.dev]. Every write command generated by the master increments this offset [systeminternals.dev].

  #### The Handshake Protocol (`PSYNC`)
  When a replica connects (or reconnects after a disconnect), it initiates a synchronization handshake [systeminternals.dev]:

  ```mermaid
  sequenceDiagram
      autonumber
      participant Replica as Replica Instance
      participant Master as Master Instance
      participant Buffer as Shared Replication Buffer
  
      Replica->>Master: Connects & sends: PSYNC <replid> <offset>
      
      alt CASE 1: Offset exists inside Circular Backlog (Partial Sync)
          Master-->>Replica: Return: +CONTINUE
          Master->>Buffer: Read missing delta bytes starting from offset
          Buffer-->>Replica: Stream missing commands
      else CASE 2: Offset is overwritten or Replid is new (Full Sync)
          Master-->>Replica: Return: +FULLRESYNC <master_replid> <current_offset>
          Master->>Master: Fork child process & generate RDB snapshot
          Master-->>Replica: Stream RDB snapshot file over TCP socket
          Note over Master, Buffer: Accumulate incoming writes in Shared Buffer
          Replica->>Replica: Flush database, load RDB, and block reads
          Master->>Replica: Stream buffered updates from the Shared Buffer
      end
  ```

  #### Under-the-Hood Logic
  *   **Partial Synchronization (The Fast Path):** If the replica’s Replication ID matches the master's, and its reported offset is still saved inside the master’s circular replication backlog buffer, a partial sync occurs [systeminternals.dev]. The master streams only the missing command bytes, making reconnection fast and light on resources [systeminternals.dev].
  *   **Full Synchronization (The Slow Path):** If the replica is new, or if its offset has been overwritten in the master's circular backlog, a full resync is triggered [systeminternals.dev]. The master forks a child process to generate a point-in-time RDB snapshot and streams it to the replica [systeminternals.dev]. While the replica loads the snapshot, the master buffers live updates in memory [systeminternals.dev].
  *   **The Shared Replication Buffer Optimization:** Modern Redis maps the replication backlog and all active replicas to a [Unified Shared Memory Chain](#glos-sharedbuf) [redisgate.jp]. Replicas track their location in this chain using personal pointers, preventing memory replication on the master and eliminating Out-of-Memory crashes under high write loads [redisgate.jp].

  ---

  <a id="5-clustering"></a>
  ### 5. High-Availability & Clustering

  When a dataset exceeds the physical memory limits of a single machine, or when read/write traffic saturates a single CPU core, Redis must scale out across a cluster [credera.com].

  #### Active-Passive (High Availability)
  The default topology is **Active-Passive** [oneuptime.com]. A single primary master node accepts all writes, and replicas act as read-only standbys [oneuptime.com]. Clients write to the master and read from replicas [oneuptime.com]. If the master fails, an orchestrator (such as Redis Sentinel) automatically promotes a replica to master, and client libraries dynamically update their routing targets [oneuptime.com].

  #### Active-Active (Global Multi-Master)
  True **Active-Active** multi-master topologies are not supported in open-source Redis. They require enterprise extensions [redis.io]. Active-Active enables multiple geographically separated instances to accept writes concurrently [redis.io]. Regions sync asynchronously, resolving write conflicts using **Last-Write-Wins (LWW) resolution backed by [Vector Clocks](#app-vclocks)** [redis.io].

  #### Clustering & Horizontal Sharding
  To distribute data evenly across multiple nodes, Redis Cluster implements database sharding [ecer.com].

  ```
                  ┌───► [ Hash Slot 0 to 5460 ]   ──► Master Node A
   [ Key ] ─► (CRC16 % 16384) ──┼───► [ Hash Slot 5461 to 10922 ] ──► Master Node B
                  └───► [ Hash Slot 10923 to 16383] ──► Master Node C
  ```

  *   **The Logic:** The keyspace is divided into **16,384 logical Hash Slots** [ecer.com]. Every key is assigned to a slot using a deterministic formula [ecer.com]:
      $$\text{Slot} = \text{CRC16}(\text{key}) \pmod{16384}$$
  *   **The Distribution:** These 16,384 slots are divided among the master nodes in the cluster [ecer.com]. Nodes can be added or removed dynamically by migrating slot ranges (and their associated keys) between nodes with zero downtime [credera.com].
  *   **Client-Side Routing:** Clients query the cluster to download and cache the slot map locally. The client calculates the hash slot for a key and sends the command directly to the correct node, avoiding intermediate proxy delays.
  *   **Proxy-Side Routing:** Alternatively, a proxy layer (like Twemproxy) sits between client applications and the database, routing requests transparently to the correct node so client libraries do not need to be cluster-aware.

  ---

  <a id="6-consistency-safety"></a>
  ### 6. Data Consistency & Safety Controls

  By default, Redis prioritizes speed by using asynchronous replication, which introduces a tiny data loss window during failovers [oneuptime.com]. To support highly sensitive transactions, Redis offers configuration controls to enforce stronger consistency guarantees [redisgate.jp].

  #### Understanding the Stale-Read Problem
  In eventual consistency, if a client writes to the Master and immediately reads from a Replica, it may receive old data because the write has not replicated yet [oneuptime.com]. To prevent this, clients must read and write only to the Master, or leverage synchronous replication commands [redisgate.jp].

  #### Synchronous Replication via `WAIT` and `WAITAOF`
  Clients can enforce synchronous replication behavior on a per-client basis [redisgate.jp]:
  *   **The `WAIT` Logic:** A client issues `WAIT <num_replicas> <timeout>`. The master suspends the client's connection context (without blocking other clients) until $N$ replicas confirm they have received the write up to the master's exact offset [redisgate.jp].
  *   **The `WAITAOF` Logic:** For disk-level safety, `WAITAOF` blocks the client until the write is confirmed to have been fully synchronized to physical disk (fsynced) on both the master and $N$ replicas.

  #### Split-Brain Protection via Connection Thresholds
  In a network partition, an isolated master might accept writes that can never be replicated, resulting in data loss once the partition heals [oneuptime.com].
  *   **The Logic:** Setting `min-replicas-to-write` prevents this. If the number of connected replicas with a lag of under `min-replicas-max-lag` seconds drops below $N$, the master immediately enters read-only mode and rejects incoming writes with a `-NOREPLICAS` error.

  ---

  <a id="7-glossary"></a>
  ### 7. Detailed Technical Mechanics (Glossary)

  ---

  <a id="glos-sds"></a>
  #### A. Simple Dynamic String (SDS)
  SDS is the custom C string implementation used in Redis to store keys and string values [codeburst.io].
  *   **The Logic:** Standard C strings are null-terminated (`\0`), meaning calculating their length requires scanning the entire string ($O(N)$ complexity). SDS prefixes the string bytes with a header that tracks the exact string length and remaining allocated buffer space. This enables $O(1)$ length checks, eliminates buffer overflows, and allows binary-safe storage of null bytes inside values.

  ---

  <a id="glos-redisdb"></a>
  #### B. `redisDb`
  The primary C-language structure representing a single logical database inside a Redis instance [codeburst.io].
  *   **The Logic:** It contains two primary pointers: `dict *dict` (the main keyspace dictionary holding all active key-value pairs) and `dict *expires` (the expiry dictionary tracking keys with active TTLs) [codeburst.io].

  ---

  <a id="glos-redisobject"></a>
  #### C. `redisObject`
  The generic wrapper structure used to encapsulate all Redis values [codeburst.io].
  *   **The Logic:** Rather than storing raw bytes directly in the hash table, every value is wrapped in a `redisObject` [codeburst.io]. This structure contains metadata fields tracking the data's logical type (e.g., Hash, Set), its physical memory encoding format (e.g., ziplist, hashtable), reference counters for memory sharing, and idle time trackers for cache eviction [codeburst.io].

  ---

  <a id="glos-rehashing"></a>
  #### D. Incremental Rehashing
  The lock-free algorithm used by Redis to resize its hash tables as the database grows or shrinks.
  *   **The Logic:** Resizing a massive hash table all at once would freeze the database for seconds. To prevent this, every Redis dictionary contains two hash tables (`ht[0]` and `ht[1]`). When a resize is triggered, Redis migrates entries from the old table to the new table gradually over subsequent client read/write commands and background cron loops, keeping latency predictable.

  ---

  <a id="glos-mpaof"></a>
  #### E. Multi-Part AOF (MP-AOF)
  The directory-based transaction logging architecture introduced in Redis 7.0 [redis.io].
  *   **The Logic:** Instead of maintaining a single massive plaintext file, MP-AOF splits the database log into three parts: a binary Base file (an RDB snapshot representing the state at the start of the rewrite), Incremental files (plaintext logs of updates since the base file was generated), and a Manifest file (the index coordinating the sequence) [redis.io].

  ---

  <a id="glos-sharedbuf"></a>
  #### F. Shared Replication Buffer
  The unified, reference-counted replication memory pool introduced in Redis 7.0 [redisgate.jp].
  *   **The Logic:** Instead of allocating separate, isolated memory buffers to hold the outgoing write stream for each replica, the master maintains a single linked chain of data blocks (`replBufBlock`) [redisgate.jp]. Each block tracks active readers using a reference counter, allowing the backlog and all connected replicas to share a single in-memory write log [redisgate.jp].

  ---

  <a id="8-appendix"></a>
  ### 8. Appendix: Auxiliary Technologies

  ---

  <a id="app-multiplexing"></a>
  #### A. I/O Multiplexing (epoll, kqueue)
  A system-level design pattern that allows a single thread to monitor multiple network sockets simultaneously for read or write readiness [dragonflydb.io].

  ```
                       ┌── Socket Client A (Idle) ──► [No Action]
                       │
   [ epoll_wait() ] ───┼── Socket Client B (Active) ─► [ Ready List ] ──► Returns to Redis Event Loop
                       │
                       └── Socket Client C (Idle) ──► [No Action]
  ```

  *   **The Core Mechanism:** Older system calls like `select` or `poll` require the OS kernel to scan through every registered socket to find which ones have data ready ($O(N)$ complexity). 
  *   **The `epoll` Optimization (Linux):** `epoll` uses a kernel-level red-black tree to track all monitored sockets [dragonflydb.io]. When data arrives at a network card, hardware interrupts place only the active socket's file descriptor into a doubly-linked "ready list" [dragonflydb.io]. The `epoll_wait()` system call then immediately returns only the active sockets ($O(1)$ complexity) [dragonflydb.io].

  ---

  <a id="app-anycast"></a>
  #### B. DNS Anycast
  A network addressing and routing technique where multiple physical servers across different global data centers share the exact same IP address.

  ```
         [User in London]                       [User in Tokyo]
                │                                      │
                ▼ (Shortest BGP Path)                  ▼ (Shortest BGP Path)
         [London Router]                        [Tokyo Router]
                │                                      │
                ▼                                      ▼
      [Anycast Node A] (IP: 1.1.1.1)         [Anycast Node B] (IP: 1.1.1.1)
  ```

  *   **The Core Mechanism:** Geographically dispersed servers are configured with the identical IP address (e.g., `8.8.8.8`). Each location advertises this IP block to upstream routers via [BGP](#app-bgp). Routers forward client requests to the topologically nearest node (fewest hops). This reduces query latency, balances load naturally, and provides instant geographic failover if a regional node goes offline.

  ---

  <a id="app-bgp"></a>
  #### C. BGP (Border Gateway Protocol)
  The standardized routing protocol of the internet used to exchange routing information across autonomous systems (AS).
  *   **The Core Mechanism:** BGP coordinates how data packets travel across global networks. Routers use BGP to dynamically advertise which IP addresses they can reach. In an Anycast configuration, multiple routers advertise the same IP destination from different locations, allowing routers to select the shortest path for each user.

  ---

  <a id="app-spinlocks"></a>
  #### D. CPU Context-Switching (Spinlocks vs. Mutexes)
  Locks prevent multiple threads from mutating the same resource concurrently, but they differ in how they block waiting threads.

  *   **Mutex (Sleeping Lock):** When a thread tries to acquire a locked mutex, the operating system kernel puts the thread to sleep, freeing the CPU to run other threads. When the lock is released, the kernel wakes the thread up. This thread suspension and waking process incurs **CPU context-switching overhead** (saving CPU registers, reloading state).
  *   **Spinlock (Busy-Wait Lock):** When a thread tries to acquire a locked spinlock, it enters an active loop, continuously checking the lock's state in CPU registers (busy-waiting) [strikefreedom.top].
  *   **The Trade-off:** Spinlocks avoid context-switching overhead, making them faster for short-duration tasks [strikefreedom.top]. However, if the lock is held for too long, spinlocks waste CPU cycles by keeping the core busy doing no useful work [strikefreedom.top].

  ---

  <a id="app-vclocks"></a>
  #### E. Vector Clocks
  An algorithm used to track causal relationships and detect logical ordering conflicts in distributed systems without relying on physical clocks.
  *   **The Core Mechanism:** Every node in a cluster maintains an array (vector) of integer counters, with one entry for each node. When a node performs a write, it increments its own counter in its local vector. When nodes replicate data, they send their vector clock alongside the payload. If Node A's vector clock is strictly greater than Node B's across all entries, Node A's write causally succeeded Node B's. If some entries are greater while others are smaller, a concurrent write conflict is detected and resolved (e.g., via Last-Write-Wins based on physical timestamps) [redis.io].

  ---

  <a id="app-cow"></a>
  #### F. Copy-On-Write (COW)
  A resource management optimization used by operating systems to share memory pages between parent and child processes.

  ```
                    ┌─── [ Parent Process (Read/Write) ] ───► Private Copy created on Write
                    │
   [ Physical RAM ] ┼─── [ Page 1 (Shared, Read-Only) ]
                    │
                    └─── [ Child Process (Read-Only) ]  ───► Reads directly from shared page
  ```

  *   **The Core Mechanism:** When a process executes a `fork()` system call, the OS does not copy the parent's entire memory footprint. Instead, both the parent and child processes point to the same physical memory pages, which are marked as read-only.
  *   **The Write Optimization:** If either process tries to modify a page, a page fault is triggered, and the OS kernel copies only that specific page to create a private, writable copy for the modifying process. This ensures rapid fork times and minimizes memory overhead, as unmodified pages remain shared.