|                     | Kafka Streams                                                | Apache Flink                                                 |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Runtime environment | Kafka Streams is provided as a **Java library** that you embed into your own applications. It **does not require a dedicated cluster** or a separate execution environment; instead, it runs wherever your Java application runs. **Scaling is achieved horizontally by starting additional instances of the application**, which **Kafka coordinates through partition assignment**. | Apache Flink offers **different runtime environments for development and production** use cases.<br/>For testing or local development, Flink provides a Local Execution Environment and a MiniCluster API, allowing developers to run, test, and debug Flink jobs entirely on their local machine without requiring a full cluster. This mode is suitable for unit tests, iterative development, and job prototyping.<br/>**In production scenarios, Flink runs on a** scalable, **distributed cluster** typically composed of JobManagers and TaskManagers:<br/>- **JobManager**: Responsible for **receiving** and managing **job submissions**, **translating** the logical plan into a **physical execution plan**, **scheduling tasks** onto TaskManagers, **coordinating checkpoints and savepoints for fault tolerance**, handling job **recovery**, **managing resources**, and providing the cluster’s REST API and web UI for monitoring.<br/>- **TaskManager**: **Executes the** actual **tasks** as defined in the job’s execution graph, **manages local state** (possibly backed by RocksDB), **exchanges data** streams **with other TaskManagers**, **handles backpressure**, and **reports status,** metrics, and progress to the JobManager.<br/>Flink can be deployed in different cluster modes, including Standalone, Apache YARN, Kubernetes, and Mesos. |
|                     |                                                              |                                                              |
|                     |                                                              |                                                              |

TODO

- Deployment model
- sink / source support
- physical execution plan / vs logical execution plan + graph optimizations
- parallelism
- serialization mechanisms (internal & external)
- deployment methods
- branching support
- schema evolution
- transactionality
- exactly-once
- checkpointing
- recovery
  - state store etc
- watermark

