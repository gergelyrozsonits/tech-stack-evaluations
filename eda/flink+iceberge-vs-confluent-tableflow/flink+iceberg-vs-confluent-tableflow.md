## Confluent TableFlow vs. Flink + Iceberg Pipeline

| Feature                  | **Confluent TableFlow**                                  | **Flink + Iceberg Pipeline**                 |
| ------------------------ | -------------------------------------------------------- | -------------------------------------------- |
| **Table Creation**       | Automated (based on Schema Registry)                     | Manual or scripted (via Flink SQL/API)       |
| **Schema Evolution**     | Automated (detects and updates table schema)             | Manual/scripted (requires monitoring)        |
| **Commit Frequency**     | Managed by Confluent (optimized for latency & file size) | User-controlled via Flink checkpointing      |
| **Operational Overhead** | Minimal (fully managed service)                          | High (self-managed infrastructure)           |
| **Flexibility**          | Limited to supported formats/clouds                      | Highly customizable (sources, sinks, logic)  |
| **Monitoring/Scaling**   | Built-in                                                 | User-managed                                 |
| **Integration**          | Tightly integrated with Confluent Cloud ecosystem        | Open ecosystem, supports custom integrations |

### Flink Example: Create Iceberg Table and Stream Data

```
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.*;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

public class FlinkIcebergKafkaApp {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

        // Set checkpointing interval (commit frequency)
        env.enableCheckpointing(300000); // 5 minutes

        // Register Iceberg Glue catalog
        tableEnv.executeSql(
            "CREATE CATALOG glue_catalog WITH (" +
            "  'type'='iceberg'," +
            "  'catalog-type'='hive'," +
            "  'catalog-impl'='org.apache.iceberg.aws.glue.GlueCatalog'," +
            "  'warehouse'='s3://your-bucket/warehouse/'" +
            ")"
        );

        // Create Iceberg table if not exists
        tableEnv.executeSql(
            "CREATE TABLE IF NOT EXISTS glue_catalog.default.kafka_iceberg_table (" +
            "  id STRING," +
            "  value STRING," +
            "  ts TIMESTAMP(3)" +
            ") PARTITIONED BY (ts)"
        );

        // Register Kafka source table
        tableEnv.executeSql(
            "CREATE TABLE kafka_source (" +
            "  id STRING," +
            "  value STRING," +
            "  ts TIMESTAMP(3)" +
            ") WITH (" +
            "  'connector' = 'kafka'," +
            "  'topic' = 'your-kafka-topic'," +
            "  'properties.bootstrap.servers' = 'your-kafka-broker:9092'," +
            "  'scan.startup.mode' = 'earliest-offset'," +
            "  'format' = 'avro'," +
            "  'avro-schema-registry.url' = 'http://your-schema-registry:8081'" +
            ")"
        );

        // Start streaming: insert data from Kafka to Iceberg
        tableEnv.executeSql(
            "INSERT INTO glue_catalog.default.kafka_iceberg_table " +
            "SELECT id, value, ts FROM kafka_source"
        );
    }
}
```

### Commit Interval in Flink

- **Flink’s Iceberg sink commits data on successful checkpoints.**

- You control how frequently data is written to Iceberg by setting the checkpointing interval:

  java

  ```
  env.enableCheckpointing(300000); // Commits every 5 minutes
  ```

- **Longer intervals:** Less frequent commits, larger files, higher latency.

- **Shorter intervals:** More frequent commits, smaller files, lower latency.

- TableFlow abstracts this away and manages commit frequency automatically.