

# The Software Engineer's Guide to Relational Database Internals & Design

This guide consolidates relational database systems theory, design principles, and physical engine mechanics. It translates theoretical concepts directly into system-level behaviors, performance trade-offs, and practical application patterns.

---

## 1. Database Design & Relational Modeling

### Cardinality & SQL Implementation
How entities relate to one another dictates physical table layouts and foreign key placement:
*   **One-to-One (1:1):** Split attributes into separate tables for domain boundaries. Implement by placing a foreign key (FK) in one table pointing to the other, marking that FK column as `UNIQUE`.
*   **One-to-Many (1:N):** Place the FK strictly on the **"Many"** side of the relationship (e.g., `Posts` table gets `UserID` FK pointing to `Users`).
*   **Many-to-Many (N:M):** Requires an associative **Junction Table** (bridge table) storing composite primary keys `(EntityA_ID, EntityB_ID)` acting as FKs to both parent tables. Junction tables are highly versatile because they can store **relationship metadata** (e.g., a `PostLikes` junction table storing a `LikedAt` timestamp).

---

### The Rules of Normalization (1NF ➡️ 2NF ➡️ 3NF)
Normalization is an iterative mathematical process to eliminate data redundancy and prevent data anomalies (Insertion, Update, Deletion).

```
┌────────────────────────────────────────────────────────┐
│ NORMALIZATION PROGRESSION                              │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1NF: Atomic Cells & Primary Keys                      │
│         │                                              │
│         ▼                                              │
│  2NF: No Partial Dependencies (Single PK = Auto 2NF)   │
│         │                                              │
│         ▼                                              │
│  3NF: No Transitive Dependencies (Non-Key -> Non-Key)  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

#### 1. First Normal Form (1NF)
*   **The Rule:** All table cells must contain **atomic (indivisible) values**. Multi-valued attributes (like comma-separated lists: `"Tech, SQL"`) are strictly banned.
*   **The Key:** Every table must have a unique identifier (Primary Key).

#### 2. Second Normal Form (2NF)
*   **The Rule:** Must be in 1NF, and must have **no partial dependencies**. Every non-key column must depend on the *entire* primary key, not just a portion of it.
*   **The Single-Key Shortcut:** If a table's Primary Key is a single column (e.g., `BookingID`), partial dependencies are mathematically impossible. **A single-column PK table is automatically in 2NF.**
*   **Composite Key Violation Example:** In `WarehouseStock(WarehouseID, ProductID, WarehouseCity, Quantity)`:
    *   The primary key is composite: `(WarehouseID, ProductID)`.
    *   `WarehouseCity` depends solely on `WarehouseID` (part of the key), violating 2NF.
    *   *Resolution:* Extract `WarehouseCity` into a separate `Warehouses(WarehouseID, WarehouseCity)` table.

#### 3. Third Normal Form (3NF)
*   **The Rule:** Must be in 2NF, and must have **no transitive dependencies**. Non-key columns must depend *only* on the primary key, never on other non-key columns.
*   **Formula:** If $A \rightarrow B$ and $B \rightarrow C$, then $A \rightarrow C$ is a transitive dependency that must be removed.
*   **Violation Example:** In `Warehouses(WarehouseID, ManagerID, ManagerEmail)`:
    *   `WarehouseID` (PK) determines `ManagerID` (Non-Key).
    *   `ManagerID` determines `ManagerEmail` (Non-Key).
    *   *Resolution:* Extract to `Managers(ManagerID, ManagerEmail)` and keep `ManagerID` as an FK in `Warehouses`.

---

### Denormalization & System Trade-Offs
Denormalization is the intentional re-introduction of redundancy to optimize read-heavy systems by bypassing expensive table `JOIN` operations.

*   **The Classic Example:** Storing a pre-computed `LikeCount` column directly inside a `Posts` table instead of performing a `COUNT()` aggregation over millions of rows in a `PostLikes` table.
*   **Write Overhead & Lock Contention:** The trade-off for instant reads is write performance. Under high concurrency (e.g., a viral post), thousands of transactions trying to update the *same row* to increment `LikeCount` will hit severe **lock contention** (causing threads to queue) or trigger frequent optimistic locking failures.
*   **Alternative Architectures:**
    *   *Eventual Consistency (CDC):* Subscribe to transaction logs using Change Data Capture (CDC) to update read-optimized views asynchronously.
    *   *Application Caching:* Cache read results in memory (e.g., Redis) with explicit cache invalidation policies.

---

### Storage Mechanics: `VARCHAR` vs. `TEXT`
*   **`VARCHAR(N)` (Variable Character):** Holds predictable, inline text up to length `N`. The data is stored directly within the database row page on disk. This allows rapid sequential page reads but makes rows wider. Standard B-Tree indexes can be built directly on `VARCHAR` columns.
*   **`TEXT`:** Designed for massive, unstructured blocks of data (up to 2GB or 4GB). Because of its size, databases use **Off-Row Storage**. The main table page stores only an 8-to-24-byte **pointer** pointing to a separate Large Object (LOB) storage area. 
    *   *Performance Impact:* Reading a `TEXT` column requires an extra disk seek to follow the pointer. Sorting queries containing `TEXT` columns can cause engines (like MySQL) to write temporary tables to disk, slowing down execution. Traditional B-Tree indexes cannot be built on whole `TEXT` columns; they require prefix indexing or Full-Text Search (FTS) indexes.
    *   *Postgres Exception:* PostgreSQL treats `VARCHAR` (without limit) and `TEXT` almost identically under the hood with its internal TOAST (The Oversized-Attribute Storage Technique) mechanism, meaning there is no performance penalty for choosing one over the other in Postgres.

---

## 2. Physical Engine Mechanics (Under the Hood)

The database engine coordinates fast volatile memory (RAM) with durable, non-volatile storage (Disk) to balance speed and data integrity.

```
                  [ BUFFER POOL (RAM) ]
                 Contains Table & Index
                    Pages in Memory
                           │
       ┌───────────────────┴───────────────────┐
       ▼ (Sequential Append)                   ▼ (Asynchronous Checkpoint)
[ WAL FILE (DISK) ]                     [ MAIN STORAGE (DISK) ]
  Chronological Logs                      Main Table & Index Files
 (Guarantees Durability)                    (Updated in Batches)
```

### The Three Storage Layers
1.  **Buffer Pool (RAM):** The database's primary workspace. It caches 8KB/16KB data and index pages. Reads and writes are executed directly in RAM first. Pages modified in memory but not yet written to disk are called **Dirty Pages**.
2.  **Write-Ahead Log (WAL / Redo Log) (Disk):** A lightweight, append-only file on disk. It records a sequential, chronological history of physical changes before they are applied to the main tables.
3.  **Main Data & Index Files (Disk):** Organized B-Tree structures containing actual table records and indexes.

### The Lifecycle of a Write (e.g., `UPDATE Users SET Age = 31 WHERE ID = 1`)
1.  **RAM Modification:** The page containing `ID = 1` is loaded into the Buffer Pool. The record's `Age` is modified to `31` in RAM. The associated Index Page in memory is also updated. The pages are now marked as "Dirty."
2.  **WAL Buffering:** A compact change record is written to the WAL Buffer in memory: *"Tx #482 changed User 1 Age to 31 at LSN 101."*
3.  **The Flush (`fsync`):** The engine forces an `fsync()` system call to write the WAL Buffer record sequentially to the physical **WAL file on disk**.
4.  **Commit Confirmation:** Once the WAL write is confirmed on disk, the database replies "Success" to the client. **The main table and index files on disk still contain the old data.**
5.  **Checkpointing:** Periodically, a background thread (the Checkpointer) flushes the batched dirty data and index pages from the Buffer Pool to the main files on disk, ensuring both disk files are fully updated.

### Crash Recovery (WAL Replay)
If power cuts out before a checkpoint occurs, the RAM is wiped. On reboot, the engine recovers using the WAL:
*   **Redo Phase:** It starts from the last recorded checkpoint in the data files and replays all committed transactions from the WAL forward, restoring the correct table and index pages in memory.
*   **Undo Phase:** Any transactions that were active but uncommitted at the moment of the crash are rolled back to preserve database consistency.

---

## 3. Concurrency, Transactions & Locking

Transactions group multiple SQL statements into a single atomic unit of work governed by the ACID rules.

### The Concurrency Read Anomalies
When transactions run simultaneously without proper isolation, three read bugs can occur:
1.  **Dirty Read:** Transaction A reads uncommitted data written by Transaction B. Transaction B then rolls back. Transaction A is left working with invalid, non-existent data.
2.  **Non-Repeatable Read (Fuzzy Read):** Transaction A reads a row. Transaction B updates that row and commits. Transaction A reads the same row again and gets different values.
3.  **Phantom Read:** Transaction A queries a *range* of rows (e.g., `Price > 100` yielding 5 rows). Transaction B inserts a *new* row in that range and commits. Transaction A runs the range query again and suddenly sees 6 rows.

---

### Application Locking Strategies
Developers use locking paradigms to prevent the "Lost Update" anomaly (where two users read the same value and overwrite each other's changes).

#### 1. Pessimistic Locking
*   **Philosophy:** "Prevent conflicts by locking immediately."
*   **Implementation:** Use the `SELECT ... FOR UPDATE` SQL clause to place an Exclusive (Write) Lock on the target rows. Other transactions attempting to read or modify these rows are blocked and put on hold.
*   **Use Case:** High-conflict scenarios where data correctness is critical (e.g., seat booking systems, financial bank transfers).

#### 2. Optimistic Locking
*   **Philosophy:** "Assume conflicts are rare; check for modifications at the finish line."
*   **Implementation:** Add a `version` (INT) or `updated_at` (TIMESTAMP) column to the table. Read the row and note its current version. When updating, explicitly check if the version has changed:
    ```sql
    UPDATE Products SET Stock = 9, version = version + 1 WHERE ID = 42 AND version = 5;
    ```
    If another transaction updated the row first, the version is no longer 5, zero rows are updated, and the application rolls back and triggers a retry.
*   **Use Case:** High-scale, low-conflict web systems (e.g., wiki page edits, profile details) where holding open database locks would degrade performance.

---

### Deadlocks: Mechanics & Prevention
A **Deadlock** occurs when Transaction A holds Lock 1 and waits for Lock 2, while Transaction B holds Lock 2 and waits for Lock 1. Both threads are frozen indefinitely.

```
┌────────────────────────────────────────────────────────┐
│ DEADLOCK CYCLE                                         │
├────────────────────────────────────────────────────────┤
│                                                        │
│   Transaction A  ──────── Holds Lock 1 ───────► (Row 1)│
│        │                                          │    │
│    Waits For                                  Waits For│
│        ▼                                          ▼    │
│     (Row 2) ◄────────── Holds Lock 2 ─────── Transaction B
│                                                        │
└────────────────────────────────────────────────────────┘
```

#### Under-The-Hood Detection & Resolution
*   **The Graph:** The database engine maintains an in-memory directed graph called a **Waits-For Graph**. Nodes are active transactions, and edges represent transactions waiting on locks held by others.
*   **The Detector:** A background thread scans the graph for closed cycles (loops).
*   **The Victim:** Upon finding a deadlock loop, the engine selects a "victim" transaction (usually the one that has done the least physical write work) and aborts/rolls it back. This releases its locks, allowing the surviving transaction to finish.

#### Prevention via Consistent Lock Ordering
To mathematically prevent deadlocks from occurring, enforce consistent resource ordering. 
*   *The Strategy:* If your application modifies multiple rows (e.g., transferring funds between Account ID 99 and Account ID 12), the application code must sort the IDs and lock the **smaller ID first**, then the larger ID.
*   *Why it works:* Both transactions are forced to queue up for the same starting resource (ID 12) rather than locking separate resources and trapping each other. No dependency cycle can form.

---

### Multi-Version Concurrency Control (MVCC)
Modern database engines (PostgreSQL, MySQL InnoDB) bypass lock overhead for read operations using **MVCC**. This ensures that **writers do not block readers, and readers do not block writers**.

*   **The Concept:** The database never overwrites data on disk. Instead, an `UPDATE` or `DELETE` creates a **new version** of the row while keeping the old one intact.
*   **Postgres Hidden Columns:**
    *   `xmin`: The Transaction ID that created this row version.
    *   `xmax`: The Transaction ID that deleted/superseded this row version (or `0` if active).
*   **Snapshots:** When Transaction B starts, it receives a transaction snapshot. When reading, Postgres checks the `xmin` and `xmax` against this snapshot to serve the exact version of the row that was committed at the moment Transaction B started. Uncommitted changes from other transactions are invisible.
*   **Autovacuum:** Because updates create multiple "dead" row versions on disk, PostgreSQL runs a background **Autovacuum** daemon to clean up "dead tuples" (row versions older than any active transaction snapshot) and reclaim physical disk space.

---

## 4. Indexing & Query Optimization

Indexes are separate B-Tree structures used to avoid slow **Sequential Scans** (reading every page on disk) in favor of fast logarithmic lookups.

### Clustered vs. Non-Clustered Indexes
*   **Clustered Index (Index-Organized Table):**
    *   **The table *is* the index.** The physical table rows are sorted on disk in the exact order of the clustered index key.
    *   Because physical files can only be sorted one way, you can have only **one** clustered index per table (typically the Primary Key).
*   **Non-Clustered Index:**
    *   A completely separate B-Tree file.
    *   The leaf nodes store only the indexed column value and a **pointer** back to the main table row.
        *   *Pointer in a Heap Table:* A Row Identifier (RID) pointing to the exact physical page and slot.
        *   *Pointer in a Clustered Table:* The Clustered Primary Key value.

---

### The Performance Tipping Point & Key Lookups
If you run a query using a non-clustered index, but ask for columns that are not stored in that index:
```sql
SELECT Email, Salary FROM Employees WHERE LastName = 'Smith';
```
1.  **Key Lookup (RID Lookup):** The database finds `'Smith'` in the non-clustered index, reads the pointer, and performs an extra random disk read to fetch `Email` and `Salary` from the main table page.
2.  **The Tipping Point:** Because random I/O is expensive, if your query returns more than **2% to 5%** of the table's total rows (e.g., 5,000 "Smith" records), the optimizer will reject the index entirely and perform a **Sequential Table Scan**, as reading sequential disk sectors is faster than making thousands of random disk jumps.

---

### Composite Indexes & The Left-Prefix Rule
A composite index indexes multiple columns, e.g., `INDEX(LastName, FirstName)`. It acts exactly like a physical **Phone Book** (sorted by Last Name first, then by First Name).

```
INDEX(LastName, FirstName)
Sorted: [Abbott, John] -> [Abbott, Mark] -> [Baker, John] -> [Smith, Alice]
```

*   **Left-Prefix Rule:** The index can only be navigated if the leftmost columns of the index are present in the `WHERE` clause.
    *   `WHERE LastName = 'Smith'` ➡️ **Can use index (Seek)**
    *   `WHERE LastName = 'Smith' AND FirstName = 'John'` ➡️ **Can use index (Seek)**
    *   `WHERE FirstName = 'John'` ➡️ **Cannot use index (Requires Table Scan)**
*   **The Range Trap:** If a query contains an inequality or range operator (like `>=`, `<`, `BETWEEN`, `LIKE`) on the first column, it breaks the sorted order of the second column:
    ```sql
    WHERE OrderDate >= '2026-01-01' AND ProductCategoryID = 5
    ```
    The engine can use `OrderDate` to seek, but it must perform a manual scan over all rows in that date range to filter for `ProductCategoryID = 5`.
*   **The Fix (Equality First, Range Last):** Define the index as `INDEX(ProductCategoryID, OrderDate)`. Because category is an exact match, the engine jumps directly to category `5`, where all entries are perfectly sorted by date, allowing a highly efficient range scan.

---

### Covering Indexes & The `INCLUDE` Clause
A **Covering Index** contains all columns requested by a query, allowing the database to execute an **Index Only Scan** and skip the expensive Key Lookup step entirely.

*   **Implicit PK Covering:** Because non-clustered indexes in clustered tables store the Primary Key as their pointer, the Primary Key is always implicitly covered by default.
*   **The `INCLUDE` Clause (Non-Key Columns):**
    Instead of making a massive composite index key, use `INCLUDE` to attach passive payload data only to the leaf nodes:
    ```sql
    CREATE INDEX ix_employees_lastname_covering 
    ON Employees(LastName) 
    INCLUDE (Email, Salary);
    ```
    *   *Why it's better:* The database only sorts the index by `LastName` (keeping parent nodes small and sorting costs low). `Email` and `Salary` are simply read from the leaf node when a match is found, eliminating Key Lookups without slowing down write operations.

---

This guide covers the core theoretical and practical elements of relational databases, equipping you with the architectural knowledge needed to design scalable, highly performant systems.