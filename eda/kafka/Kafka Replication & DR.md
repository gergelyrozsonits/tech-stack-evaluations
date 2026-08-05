# Kafka Topic Mirroring, Schema Replication, and Offset Management: Architectural Summary

This document provides a highly detailed architectural overview of cross-cluster topic replication, schema synchronization, offset translation mechanics, and multi-region optimizations in Apache Kafka.

---

## 1. Replication Options & Under-the-Hood Mechanics

Replicating distributed log partitions across physically distinct clusters (or wide-area networks) requires weighing different architectures. The table below compares the three primary mirroring options:

| Feature                     | MirrorMaker 2 (MM2)                                       | Confluent Replicator                                      | Confluent Cluster Linking                                 |
| :-------------------------- | :-------------------------------------------------------- | :-------------------------------------------------------- | :-------------------------------------------------------- |
| **Architecture**            | Runs on Kafka Connect framework [conduktor.io]            | Runs on Kafka Connect framework [conduktor.io]            | Native Broker-to-Broker [conduktor.io]                    |
| **Licensing**               | Open-source (Apache 2.0)                                  | Enterprise Proprietary                                    | Enterprise Proprietary                                    |
| **Offset Alignment**        | Lossy translation via periodic checkpoints [confluent.io] | Automated translation via client interceptors             | **Identical**: Byte-for-byte exact matches [conduktor.io] |
| **Loop Prevention**         | Suffix/Prefix naming policies [dev.to]                    | Custom provenance headers [confluent.io]                  | Read-only mirror topics [confluent.io]                    |
| **Infrastructure Overhead** | High (Requires managing a Connect cluster) [conduktor.io] | High (Requires managing a Connect cluster) [conduktor.io] | **None** (Managed entirely inside brokers) [conduktor.io] |

### Apache Kafka MirrorMaker 2 (MM2)
MM2 runs on the Kafka Connect framework and uses a combination of Source, Checkpoint, and Heartbeat Connectors to replicate data and translate progress [conduktor.io].

* **Topic Renaming & Topology:** By default, MM2 prevents active-active replication loops by prepending the source cluster's name as a prefix to the target topic (e.g., replicating topic `orders` from cluster `us-west` creates `us-west.orders` on the target cluster) [dev.to]. 
* **Consumer Consumption:** Downstream applications must utilize wildcard pattern subscriptions (e.g., subscribing to `.*orders`) to consume from both local and replicated topics simultaneously [dev.to].
* **Identity Replication Policy:** You can override this prefixing behavior using an `IdentityReplicationPolicy` to keep topic names identical across clusters. However, doing so disables MM2's built-in loop prevention, requiring manual filtering inside application code to avoid infinite active-active mirroring.

### Confluent Replicator
Confluent Replicator is an enterprise-grade Connector that runs on Kafka Connect [conduktor.io].

* **Topic Renaming & Topology:** Replicator preserves identical topic names across clusters without prepending prefixes [confluent.io].
* **Provenance Headers (Loop Prevention):** To prevent infinite loops in active-active topologies, Replicator injects a custom metadata header (a **Provenance Header**) into every Kafka record it writes to the target [confluent.io]. 
* **Connect Overhead:** The Replicator instances on the target cluster read every message from the local topic, check if it contains a provenance header originating from the source cluster, and discard it if true [confluent.io]. This prevents replication loops but places significant processing and network overhead on the Kafka Connect JVM workers, which must continuously deserialize and filter out duplicate messages.

### Confluent Cluster Linking
Cluster Linking is a proprietary, modern replication protocol that runs directly at the broker level, completely bypassing Kafka Connect [conduktor.io].

* **Topic Renaming & Topology:** It natively preserves identical topic names [conduktor.io].
* **Mirror Topics:** It operates by creating a read-only **Mirror Topic** on the destination cluster [confluent.io]. The destination brokers establish a direct TCP connection back to the source brokers and replicate data at the byte level [conduktor.io]. 
* **Active-Passive Setup:** Because the destination topic is strictly read-only, local producers on the target cluster are blocked from writing to it, making Cluster Linking natively suited for active-passive topologies [confluent.io].

---

## 2. Schema Replication Options, Limitations & Best Practices

Replicating schemas alongside message payloads is critical to prevent downstream deserialization failures. 

### The Metadata Topic Access Limitation
* **Self-Managed Kafka:** In self-managed environments, Confluent Schema Registry stores its schema states in a local, compacted internal topic called `_schemas`. Teams can replicate this topic using standard mirroring tools to sync registries.
* **Confluent Cloud:** In Confluent Cloud (SaaS), **the `_schemas` topic is completely hidden and inaccessible to users**. Security policies block any attempt to directly consume or mirror this topic. To sync schemas, teams must utilize Confluent's REST-based **Schema Linking** or manage synchronization via CI/CD pipelines [confluent.io].

### The Sequential Schema ID Drift Problem
Confluent Schema Registry allocates global schema IDs sequentially (`1, 2, 3...`) as schemas are registered [conduktor.io]. 
* If **Region A** and **Region B** are both active and disconnected, a schema registered in Region A might receive ID `101`. Simultaneously, a completely different schema registered in Region B will also receive ID `101` [conduktor.io].
* When messages are replicated, the schema ID is embedded inside the first few bytes of the payload. When a consumer in Region B reads a mirrored message carrying ID `101`, it will look up ID `101` in Region B's local registry, leading to deserialization failures or data corruption due to mismatched schemas [conduktor.io].

### Active-Passive Sync: Schema Linking (IMPORT Mode)
In an active-passive setup, the passive registry is set to **`IMPORT` mode** [confluent.io].
* **Preserving IDs:** The Schema Exporter (Schema Linking) connects directly via the Schema Registry REST API [confluent.io]. Because the target is in `IMPORT` mode, the exporter bypasses automatic sequential ID generation and **explicitly writes the schema with the exact same ID and version number** as the source registry [confluent.io]. This guarantees identical ID mapping between both clusters.

### Active-Active Sync: Schema Contexts
In an active-active setup where both regions actively accept writes, you cannot use a simple read-only import mode. Instead, Confluent utilizes **Schema Contexts** (virtual sub-registries) [confluent.io].

```
[ Region A Schema Registry ]                 [ Region B Schema Registry ]
┌──────────────────────────┐                 ┌──────────────────────────┐
│  Default Context (.)     │ ──(Exporter A)─>│  Context .region-a       │ (Read-Only)
│  (Active Writes)         │                 │  (Import Mode)           │
├──────────────────────────┤                 ├──────────────────────────┤
│  Context .region-b       │<──(Exporter B)──│  Default Context (.)     │
│  (Import Mode)           │                 │  (Active Writes)         │
└──────────────────────────┘                 └──────────────────────────┘
```

1. **Isolation:** Contexts are logical namespaces (e.g., `.region-a` and `.region-b`) [confluent.io]. Schema IDs and subject names are isolated within each context, meaning ID `101` in the default context `.` is entirely separate from ID `101` in context `.region-a`.
2. **Bidirectional Exporting:** Exporter A mirrors Region A's default schemas into Region B's `.region-a` context (in `IMPORT` mode) [confluent.io]. Exporter B mirrors Region B's default schemas into Region A's `.region-b` context [confluent.io].
3. **Consumer Configuration:** Mirrored messages retain their original IDs [confluent.io]. To decode these, consumer clients in Region B reading topics mirrored from Region A are configured to point to the specific virtual context URL:
   ```properties
   schema.registry.url=https://<sr-endpoint-region-b>/contexts/.region-a
   ```

### Enterprise Best Practice: GitOps and CI/CD Schema Management
To avoid runtime synchronization overhead, enterprise environments programmatically coordinate schema registration during build pipelines using `IMPORT` mode [confluent.io].

```
                     [ Git Schema Repository ]
                                 │
                                 ▼
                     [ GitOps Pipeline Runner ]
                                 │
        ┌────────────────────────┴────────────────────────┐
        ▼                                                 ▼
[ Registry A (CA) ]                               [ Registry B (CB) ]
1. POST standard registration              2. PUT Subject into IMPORT Mode [confluent.io]
2. Capture assigned ID: 10055              3. POST payload with explicit ID 10055 [confluent.io]
                                           4. PUT Subject back to READWRITE Mode [confluent.io]
```

1. **Single Source of Truth:** Schemas are stored in a Git repository.
2. **Primary Registration:** The pipeline registers the new schema to the primary Schema Registry (CA). Registry A registers it sequentially and returns the assigned ID (e.g., `10055`).
3. **Target Subject Toggle:** The pipeline targets CB's Schema Registry and programmatically toggles the corresponding subject to **`IMPORT` mode** [confluent.io]:
   ```bash
   curl -X PUT -d '{"mode": "IMPORT"}' "https://registry-b/mode/my-topic-value?force=true"
   ```
4. **Target Registration:** The pipeline registers the schema to CB, explicitly specifying the ID `10055` captured from CA [confluent.io]:
   ```json
   { "schemaType": "AVRO", "version": 1, "id": 10055, "schema": "{...}" }
   ```
5. **Restore Mode:** The pipeline immediately reverts the subject on CB to **`READWRITE` mode** [confluent.io].

---

## 3. Offset Management & Mapping

### Why Offset Mapping is Needed
An offset is simply an incremental integer index representing a message's position within a partition. **Kafka does not guarantee that the same message will occupy the same offset across different clusters.** Even if topic configurations and partition counts are identical, physical offset logs will naturally diverge. 

If a consumer group is failed over to a DR cluster, it cannot simply request its last committed offset from the source cluster without hitting an error or experiencing massive data gaps.

### Scenarios Causing Offset Gaps & Discrepancies
Below are the critical scenarios where offsets drift, showing that **all consumer groups (not just those utilizing Exactly-Once Semantics)** are affected:

#### 1. Transaction Control Events (EOS vs. Non-EOS Impact)
* Under Exactly-Once Semantics (EOS), Kafka producers write **Control Batches** (commit or abort markers) directly into the partition log to finalize transactions. These markers occupy physical offset indices.
* Standard Connect-based replicators (like MM2 or Replicator) act as standard consumers. They resolve transactions locally, discard the control markers, and replicate **only the verified payload messages** to the target cluster [confluent.io].
* Consequently, the source partition log physically retains control markers (consuming offset slots), while the target partition log does not. Offset alignment is broken immediately, impacting even standard, non-transactional consumers.

#### 2. Producer Retries
* If a producer client experiences a network glitch while writing to the source cluster, it may retry and write duplicate messages, or write them in a different batch sequence. 
* While the source log records these retries, the mirroring tool processes the stream asynchronously and may write them to the destination under a different physical sequence, leading to offset misalignment.

#### 3. Log Compaction Drift
* Log compaction runs asynchronously on each individual broker based on local CPU cycles and segment thresholds.
* If a message with a specific key is cleaned up on the source cluster but has not yet been processed by the compaction thread on the target cluster, the active physical offset sequence will differ. 

#### 4. Log Retention Gaps
* If the source cluster has a short retention time and purges older log segments before the replication tool can fetch them, the replicator will skip those missing messages. 
* When it writes the remaining stream to the target cluster, the target log starts from a different index base, completely shifting the offset boundaries.

#### 5. Partition Count Changes
* If a topic on the source cluster is scaled up from 5 to 10 partitions, mapping must track how messages are hashed and distributed. Direct offset-to-offset replication is physically impossible in this scenario.

---

## 4. Disaster Recovery, Safety Trade-offs, & Multi-Region Optimizations

### Failover Offset Reset Safety Trade-offs
When a failover occurs and a consumer group starts up on the DR cluster, the auto-offset reset policy dictates the behavior if translated offset checkpoints are missing or point to already-purged compacted segments:

* **`auto.offset.reset=earliest`:**
  * *Outcome:* The consumer group rewinds to the beginning of the topic on the DR cluster.
  * *Trade-off:* **Guarantees zero message loss**, but triggers a massive processing storm of duplicate messages that your downstream systems must deduplicate.
* **`auto.offset.reset=latest`:**
  * *Outcome:* The consumer group jumps straight to the end of the log on the DR cluster.
  * *Trade-off:* Eliminates duplicate processing storms, but **creates a severe risk of silent data loss**, as all unread messages inside the replication lag window are skipped.

### Exactly-Once Semantics (EOS) DR Failover Realities
* **The Guarantee is Lost:** Exactly-Once Semantics are maintained via local transaction coordinators and transactional logs (`__transaction_state`) isolated to a single logical cluster. 
* **The Switchover Penalty:** These states do not replicate across physical boundaries. During a DR switchover, active transactions are aborted, and retried client writes will be treated as fresh records on the new cluster. **Duplicates are guaranteed upon failover**, making downstream, database-level idempotency mandatory.

### Access Control List (ACL) Replication
* **No Auto-Replication:** ACLs are stored in local cluster metadata (KRaft/ZooKeeper) and do not sync across the wire automatically.
* **Best Practice:** Avoid using runtime connector synchronization (such as MM2 `SyncGroup` policies), as they are slow and require highly elevated security permissions. Instead, utilize **GitOps infrastructure-as-code (Terraform or Ansible)** to apply identical ACL templates to both clusters simultaneously.

### Multi-Region "Stretch" Clusters
For environments running a single logical Kafka cluster spanning multiple geographic regions over a high-latency WAN, Kafka optimizes write and read paths using two primary features:

```
[ Producer ] ──(acks=all)──> [ Region A (Leader) ] 
                                  │ (Sync replication)
                                  ▼
                             [ Region A (Follower) ]  <-- Fast local write path
                                  │
                                  ░░░░░ WAN Link (High Latency) ░░░░░
                                  │
                             [ Region B (Observer) ]  <-- Asynchronous replication (No write blockage!)
```

* **Confluent Observers (Write Optimization):** In standard replication, `acks=all` forces the producer to wait for all In-Sync Replicas (ISR) across the WAN, destroying performance [automq.com]. Confluent introduced *Observers* [automq.com]. Observers replicate data asynchronously and **never join the ISR** [automq.com]. Producers writing with `acks=all` only wait for local synchronous replicas (within Region A), while the observer in Region B fetches asynchronously, bypassing WAN latency [automq.com].
* **Follower Fetching (Read Optimization - KIP-392):** Consumers use rack awareness properties (`client.rack` matching the broker's `broker.rack`) to read directly from their closest local replica or observer rather than fetching from the partition leader over the WAN. This eliminates cross-region network egress charges and read latency.
* **Tuning WAN Replication:** To minimize observer replication lag, brokers must be tuned to maximize throughput over high-latency links:
  ```properties
  # Increase replication threads to handle parallel cross-region fetch loops
  num.replica.fetchers=4
  
  # Allow larger batch transfers per fetch request to combat WAN high round-trip times
  replica.fetch.max.bytes=10485760         # 10MB (Default 1MB)
  replica.fetch.response.max.bytes=52428800 # 50MB (Default 10MB)
  ```