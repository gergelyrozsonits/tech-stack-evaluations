## Pruning in Apache Iceberg

**What is Pruning Used For?**
Pruning is a technique used to optimize query performance by eliminating unnecessary data files and partitions from being scanned during query execution. This is especially important when working with large datasets stored in object stores, where minimizing I/O operations directly translates to faster queries and lower costs.

**Benefits of Pruning:**

- **Optimized Query Performance:** By skipping irrelevant files and partitions, queries run faster.
- **Reduced I/O Operations:** Less data is read from the object store, saving time and reducing costs.
- **Efficient Resource Utilization:** Only the necessary data is processed, improving overall system efficiency.

**How Pruning Works in Iceberg:**

- **Metadata Pruning:** Uses manifest files and lists to identify relevant data files for a query.
- **Partition Pruning:** Excludes entire partitions based on query filters and partition metadata.
- **Manifest Pruning:** Skips manifest files that do not match query predicates using stored statistics.
- Data File Pruning:
  - Each data file (such as Parquet, ORC, or Avro) contains its own statistics (min/max values, null counts, etc.) in its metadata section, maintained by the file format itself.
  - When a data file is written to an Iceberg table, Iceberg extracts these statistics directly from the file’s metadata—there is no need for Iceberg to recalculate them.
  - Iceberg then "lifts" these statistics into its own metadata files (manifests), making them available for efficient query planning and fine-grained data skipping.
- **Hidden Partitioning:** Stores partition info in metadata, not file paths, enabling flexible partition evolution.