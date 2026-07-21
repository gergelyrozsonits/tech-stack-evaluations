| Parameter                              | Avro                                      | ORC                                         | Parquet                                     |
|-----------------------------------------|-------------------------------------------|----------------------------------------------|---------------------------------------------|
| **Purpose of the format**               | Data serialization and exchange           | Efficient storage and processing in Hadoop   | Efficient analytics and storage in Hadoop   |
| **OLTP/OLAP**                           | OLTP, streaming, and data interchange     | OLAP, analytics                             | OLAP, analytics                            |
| **Columnar or Row Format**              | Row-based                                 | Columnar                                    | Columnar                                    |
| **Primary Usages**                      | Data serialization, Kafka, ETL pipelines  | Hive, Hadoop analytics, big data storage    | Spark, Hive, Impala, big data analytics    |
| **Limitations of Usages**               | Not optimized for analytics, less efficient for columnar queries | Less support outside Hadoop ecosystem, not for OLTP | Not for OLTP, limited schema evolution     |
| **Compression Support**                 | Yes (Snappy, Deflate, Bzip2, etc.)        | Yes (Zlib, Snappy, LZO, etc.)               | Yes (Snappy, Gzip, Brotli, LZO, etc.)      |
| **Metadata Section & Capabilities**     | Header contains schema and metadata        | Footer contains rich metadata and indexes    | Footer contains metadata and statistics     |
| **Schema Embedding in Data Types**      | Yes, schema in JSON in file header         | Yes, schema in file footer                   | Yes, schema in file metadata                |
| **Schema Support (Nested Structures)**  | Supports complex and nested structures     | Supports complex and nested structures       | Supports complex and nested structures      |
| **Schema Evolution**                    | Strong support: add, remove, change types (with rules) | Add/remove columns, limited type changes     | Add/remove columns, limited type changes    |
| **Chunking / Partitioning Options**     | Block-based, not optimized for partitioning| Stripes and row groups, supports partitioning| Row groups, supports partitioning           |