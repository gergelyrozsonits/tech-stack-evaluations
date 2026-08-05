# The Distributed Polling Outbox Pattern

The Transactional Outbox Pattern ensures that database updates and their corresponding events are published to a message broker atomically. When implementing this pattern without Change Data Capture (CDC), the system must rely on **periodic database scans (polling)**. 

To scale this across a distributed environment, the system must solve three key problems:
1. **Concurrency Control:** Preventing multiple instances from processing the same messages simultaneously.
2. **Resilience:** Automatically recovering and reprocessing messages if an instance fails mid-execution.
3. **Ordering:** Ensuring correlated messages (e.g., events for the same `user_id` or `order_id`) are processed in strict chronological order.

---

## 1. Transaction Boundaries: The Two-Phase State Machine

To scale periodic polling, the database transaction must be kept as short as possible. 

### The Long-Lived Transaction Antipattern
Holding a database transaction open while performing network I/O (e.g., publishing to Kafka or RabbitMQ) blocks database rows and quickly exhausts connection pools.

### The Two-Phase Solution
We decouple the database transaction from network dispatching by splitting the operation into two distinct phases:

```
┌────────────────────────────────────────────────────────┐
│ PHASE 1: Claiming the Batch (Fast DB Transaction)      │
│  1. BEGIN TX                                           │
│  2. SELECT ... FOR UPDATE SKIP LOCKED                  │
│  3. UPDATE state to 'PROCESSING', write locked_by/at  │
│  4. COMMIT TX (Connection released)                     │
└───────────────────────────┬────────────────────────────┘
                            │ (Local In-Memory Queue)
                            ▼
┌────────────────────────────────────────────────────────┐
│ PHASE 2: Dispatching (Long-Running Network I/O)        │
│  1. Dispatch event to Message Broker (Network call)    │
│  2. Once ACK received:                                 │
│     BEGIN NEW TX ──> UPDATE state to 'DELIVERED' ──> COMMIT│
└────────────────────────────────────────────────────────┘
```

---

## 2. The Janitor Pattern (Resilience & Recovery)

Because Phase 1 commits and unlocks the database rows before the events are dispatched, the database does not natively know if a worker instance fails. If a worker crashes during Phase 2 (e.g., due to an OOM, VM shutdown, or network partition), those records will remain stuck in the `PROCESSING` state forever.

### The Solution
A lightweight background process (the "Janitor" or "Reaper") periodically scans the table for stalled records and resets them.

#### Recovery SQL Query:
```sql
UPDATE outbox 
SET status = 'PENDING', 
    locked_by = NULL, 
    locked_at = NULL 
WHERE status = 'PROCESSING' 
  AND locked_at < (NOW() - INTERVAL '5 minutes');
```

*Note: Since the Janitor might reclaim records from a worker that is merely slow (e.g., experiencing a long Garbage Collection pause), this recovery can lead to duplicate message deliveries. Downstream consumers **must** be idempotent.*

---

## 3. Distributed Polling Patterns at Scale

### Pattern 1: Database-Level Lock Partitioning (Pessimistic Locking)
Workers use native database engine features to claim records concurrently without collision.

*   **How it works:** Instances execute a query using `FOR UPDATE SKIP LOCKED`. If Instance A locks a batch of records, Instance B automatically skips them and locks the next available batch.
*   **Query Example:**
    ```sql
    SELECT * FROM outbox 
    WHERE status = 'PENDING' 
    LIMIT 100 
    FOR UPDATE SKIP LOCKED;
    ```
*   **Pros:** Simplest implementation; requires no external coordinator.
*   **Cons:** High database connection contention at extreme scales.

### Pattern 2: Distributed Scheduler with Leader Election (Orchestrated)
Utilizes an external coordinator to elect a single active worker or distribute work leases.

*   **How it works:** Workers attempt to acquire an execution lock from a distributed lock manager (e.g., **Redis/ShedLock, ZooKeeper, or Consul**). The instance that wins the lock executes the polling cycle.
*   **Resiliency:** If the active leader crashes, its lease expires via Time-To-Live (TTL), allowing a healthy instance to acquire the lock and resume polling.
*   **Pros:** Prevents redundant database polling; protects database resources.
*   **Cons:** Introduces additional infrastructure dependencies.

### Pattern 3: Bucket/Shard-Based Ownership (Choreographed Allocation)
Partitions the workload into static or dynamic shards for high-throughput scaling.

*   **How it works:** Outbox records are assigned a `bucket_id` based on a hash of their partition key (e.g., `hash(correlation_key) % 1024`). Workers register in a service registry and divide ownership of the bucket IDs among themselves. Each instance polls only its assigned buckets.
*   **Resiliency:** If a worker fails, its assigned buckets are reallocated to the remaining active workers via a rebalancing protocol.
*   **Pros:** Highly scalable; completely eliminates row contention.
*   **Cons:** Complex state-synchronization and rebalancing logic.

---

## 4. Preserving Message Order for Correlated Events

When multiple instances poll concurrently, there is a risk that subsequent events for the same entity (e.g., `OrderPaid` after `OrderCreated`) are dispatched out of order. Enforcing order requires ensuring that only one worker processes a given correlation group at any time.

### Strategy A: Correlation Key Sequential Blocking (Dynamic SQL Locks)
This database-driven approach ensures a message is never claimed if an earlier message with the same correlation key is currently in progress.

#### Query Example:
```sql
BEGIN;

SELECT * FROM outbox o1
WHERE o1.status = 'PENDING'
  -- 1. Block if an older message for this correlation key is currently PROCESSING
  AND NOT EXISTS (
      SELECT 1 FROM outbox o2 
      WHERE o2.correlation_key = o1.correlation_key 
        AND o2.status = 'PROCESSING'
  )
  -- 2. Enforce FIFO: only grab the oldest PENDING message for this key
  AND o1.id = (
      SELECT MIN(o3.id) FROM outbox o3 
      WHERE o3.correlation_key = o1.correlation_key 
        AND o3.status = 'PENDING'
  )
ORDER BY o1.id ASC
LIMIT 50
FOR UPDATE SKIP LOCKED;

-- Update the status of claimed records to 'PROCESSING' and COMMIT
COMMIT;
```

### Strategy B: Hash-Based Bucket Locking (Metadata Control Tables)
To avoid running complex subqueries on large outbox tables, locking is delegated to a separate control table of virtual "buckets".

```
[Outbox Table]                             [Outbox Buckets Table]
┌────┬─────────────┬───────────┬─────────┐   ┌───────────┬────────┬───────────┐
│ ID │ Customer ID │ Bucket ID │ Status  │   │ Bucket ID │ Status │ Locked By │
├────┼─────────────┼───────────┼─────────┤   ├───────────┼────────┼───────────┤
│ 1  │ Cust_99     │ Bucket_2  │ PENDING │   │ Bucket_1  │ FREE   │ NULL      │
│ 2  │ Cust_99     │ Bucket_2  │ PENDING │   │ Bucket_2  │ LOCKED │ Worker_A  │
└────┴─────────────┴───────────┴─────────┘   └───────────┴────────┴───────────┘
```

1. **Lock the Bucket:** Workers compete to lock a bucket in `outbox_buckets` using `FOR UPDATE SKIP LOCKED`.
2. **Process Sequentially:** The worker that successfully locks `Bucket_2` retrieves and dispatches all `PENDING` outbox records assigned to `Bucket_2` in strict FIFO order (`ORDER BY id ASC`).
3. **Release:** Once the batch is processed, the bucket is unlocked and set back to `FREE`.