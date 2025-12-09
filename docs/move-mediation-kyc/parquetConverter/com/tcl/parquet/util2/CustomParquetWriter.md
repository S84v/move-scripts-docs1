# Summary
`CustomParquetWriter` encapsulates the creation of a Parquet file using Apache Avro schema. It receives a list of `GenericData.Record` objects and writes them to a Hadoop `Path` with SNAPPY compression. In the MOVE‑KYC pipeline it is used to persist transformed KYC data sets as column‑ariented Parquet files for downstream analytics and archiving.

# Key Components
- **class `CustomParquetWriter`**
  - `writeToParquet(List<GenericData.Record> recordsToWrite, Path pathtoParquet)`: opens an `AvroParquetWriter` with a schema obtained from `GetSchema.getCustomSchema`, writes each record, and closes the writer via try‑with‑resources.
- **External classes**
  - `GetSchema.getCustomSchema(String schemaPath)`: static helper that loads an Avro schema from the file system (referenced through `ParquetConverterUtility.schemaFilePath`).
  - `ParquetConverterUtility`: holds the global configuration `schemaFilePath`.
- **Libraries**
  - Apache Avro (`GenericData.Record`)
  - Hadoop (`Configuration`, `Path`)
  - Parquet (`AvroParquetWriter`, `ParquetWriter`, `CompressionCodecName.SNAPPY`)

# Data Flow
| Stage | Source | Destination | Transformation |
|-------|--------|-------------|----------------|
| Input | In‑memory `List<GenericData.Record>` (produced by upstream KYC processing) | – | No modification; records are streamed directly |
| Write | `ParquetWriter` (AvroParquetWriter) | HDFS/S3 path (`Path pathtoParquet`) | Serializes records to Parquet with SNAPPY compression |
| Side Effects | – | File system (HDFS/S3) | Creates/overwrites Parquet file; may trigger downstream jobs that consume the file |

# Integrations
- **Upstream**: Called by KYC data transformation services that convert raw CSV/JSON into Avro `GenericData.Record` objects.
- **Downstream**: Generated Parquet files are consumed by analytics jobs (Hive, Spark) and archival processes.
- **Configuration**: Relies on `ParquetConverterUtility.schemaFilePath` to locate the Avro schema; schema is shared across the KYC subsystem.

# Operational Risks
- **Schema Mismatch**: If `schemaFilePath` points to an incompatible schema, writer throws runtime exception. *Mitigation*: Validate schema version before invoking writer.
- **IO Failure**: HDFS/S3 write errors (disk full, permission) cause `IOException`. *Mitigation*: Implement retry logic at caller level; monitor storage quotas.
- **Large Record Sets**: Loading all records into a `List` may cause OOM for massive datasets. *Mitigation*: Stream records from source iterator instead of materializing full list.
- **Resource Leak on Failure**: Although try‑with‑resources closes writer, failure in `GetSchema` before writer creation can leave open file handles. *Mitigation*: Ensure schema loading occurs outside critical path or is wrapped in its own try/catch.

# Usage
```java
// Example in a unit test or job
List<GenericData.Record> records = MyRecordBuilder.buildSampleRecords();
Path outputPath = new Path("hdfs://namenode:8020/user/klm/kyc/output.parquet");

CustomParquetWriter writer = new CustomParquetWriter();
writer.writeToParquet(records, outputPath);
```
- Add required Hadoop configuration to classpath.
- Ensure `ParquetConverterUtility.schemaFilePath` points to a valid `.avsc` file.

# Configuration
- **Environment Variable / System Property**: `ParquetConverterUtility.schemaFilePath` – absolute or classpath location of the Avro schema file.
- No additional runtime configuration; Hadoop `Configuration` is instantiated with defaults.

# Improvements
1. **Streaming API**: Replace `List<GenericData.Record>` with `Iterator<GenericData.Record>` to reduce memory footprint for large datasets.
2. **Retry & Metrics**: Add configurable retry policy for write failures and emit Prometheus/SLF4J metrics (records written, latency, error count).