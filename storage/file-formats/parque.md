[TOC]

# Parquet File Format: Key Features and Query Engine Interaction

## 1. Why Parquet Is Unique and Popular
- **Efficient Compression:** Uses column-level compression algorithms (Snappy, Gzip, Brotli), reducing storage size and speeding up data scans.
- **Predicate Pushdown:** Stores metadata (min/max values, counts, null counts) for each column chunk, allowing query engines to skip irrelevant data and reduce I/O.
- **Self-Describing Schema:** Embeds schema information in the file, supporting flexible schema evolution and interoperability.
- **Advanced Encoding:** Employs dictionary encoding, run-length encoding, and bit-packing for further size reduction and faster access.
- **Parallel Processing:** Organizes data into row groups and column chunks, enabling distributed engines to process files in parallel.
- **Wide Ecosystem Support:** Compatible with most big data and analytics tools, making it a standard for data lakes.
- **Efficient Data Skipping:** Metadata and columnar storage allow engines to quickly skip non-relevant data, crucial for large-scale analytics [1].

## 2. Query Engine Interaction with Parquet in Object Stores
- **Parallel Row Group Download:** Query engines can download multiple row groups in parallel, improving performance.
- **Metadata Access:** Parquet metadata (footer) is stored at the end of the file and includes schema and row group info. Engines read the footer first to validate schema and constraints.
- **Column Chunk Access:** Engines can read only relevant column chunks within row groups, accelerating access when files are stored locally. Streaming is less efficient for selective column access.
- **Decompression:** Column chunks are compressed and must be decompressed before processing; reduced I/O typically outweighs CPU decompression cost.
- **Sequential Evaluation:** No built-in indexing; data within column chunks is read sequentially, with predicate pushdown helping to skip irrelevant row groups [2].

## 3. Parquet Metadata Storage and Streaming Considerations
- **Metadata Location:** Metadata is stored in the file footer, not separately. If row groups are stored as separate objects, each is a standalone Parquet file with its own footer.
- **Streaming from Object Store:**  
  - With HTTP range requests (supported by S3 and modern readers), only the footer and required data chunks are downloaded.
  - Without range requests, the entire file must be downloaded to access the footer, which is inefficient for large files [2].

# Compression

## Compression vs sorting

Sorting your dataset before storing it in Parquet format can improve compression efficiency. When similar values are grouped together in columns, compression algorithms can reduce file size more effectively. Therefore, sorting by columns with repeated values before writing to Parquet is a recommended best practice for better compression.