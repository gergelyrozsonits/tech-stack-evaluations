## Transaction Commits, Locks, and Consistency Guarantees in Apache Iceberg

**Overview:**
Iceberg ensures reliable and consistent table operations through atomic commits, optimistic locking, and snapshot isolation, providing strong consistency guarantees for both reads and writes.

**Atomic Commit Protocol:**
Table metadata updates are committed atomically via the catalog. Only one commit succeeds if multiple writers attempt concurrent updates, preventing partial or conflicting changes.

**Optimistic Locking and Retry Mechanism:**

- Writers prepare new metadata and attempt to update the catalog.
- Before committing, the writer checks that the catalog’s metadata version matches its expectation.
- If another concurrent commit has updated the catalog, the operation fails due to a version mismatch.
- **Retry Logic:** The processing engine detects the failure, reloads the latest metadata, reapplies its changes, and retries the commit. This ensures that all changes are based on the most recent state and prevents lost updates.

**Snapshot Isolation:**
Every table change creates a new snapshot. Readers always see a consistent snapshot, ensuring read-after-write consistency and preventing partial/in-progress changes.

**Multi-table Transactions:**
If supported by the catalog (e.g., Nessie), atomicity can be extended to operations involving multiple tables, so all changes are committed together.

**Manifest and Data File Validation:**
During commit, Iceberg validates the existence and integrity of referenced files, ensuring no corruption or overlap.

**Schema and Partition Spec Validation:**
Changes to schema and partition specs are validated for compatibility and correctness.

**Garbage Collection:**
Expired snapshots and orphaned files are managed to prevent data loss and ensure only valid files remain.

**Consistency Guarantees:**

- Atomicity and isolation for table operations
- Consistent, reliable reads and writes
- Protection against lost updates and conflicting changes