# Summary
`ParquetConverterUtility` is a command‑line driver that orchestrates the conversion of a raw KYC CSV file into a Parquet file. It receives three file‑system paths (input CSV, output Parquet, Avro schema), builds Avro `GenericData.Record`s via `CustomRecordBuilder`, and writes them to Parquet using `CustomParquetWriter`. In production it is invoked by batch jobs or orchestration scripts to materialize KYC extracts in columnar storage for downstream analytics.

# Key Components
- **`ParquetConverterUtility` (class)**
  - Public static fields `inputFilePath`, `outputFilePath`, `schemaFilePath` – hold CLI arguments.
  - `main(String[] args)` – entry point; parses arguments, creates `CustomParquetWriter`, constructs a Hadoop `Path` for the output, and calls `writeToParquet`.
- **`CustomRecordBuilder.buildRecord(String csvPath, String schemaPath)`**
  - Reads the CSV at `csvPath`, loads the Avro schema from `schemaPath` (via `GetSchema`), returns a `List<GenericData.Record>`.
- **`CustomParquetWriter.writeToParquet(List<GenericData.Record> records, Path outPath)`**
  - Serialises the Avro records to a Parquet file at `outPath`.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | CLI args: `args[0]` (CSV), `args[1]` (Parquet), `args[2]` (Avro schema) | Assigned to static fields | In‑memory path strings |
| 2 | CSV file (local FS / HDFS) | `CustomRecordBuilder.buildRecord` parses CSV → Avro `GenericData.Record`s | `List<GenericData.Record>` |
| 3 | Avro schema file (classpath or FS) | `GetSchema` loads schema via `Schema.Parser` | `org.apache.avro.Schema` (used internally) |
| 4 | Record list + Hadoop `Path` for output | `CustomParquetWriter.writeToParquet` writes Parquet file | Parquet file created at `outputFilePath` |
| 5 | Exceptions | Caught and printed to `stderr` | No retry logic; process terminates on error |

External services: Hadoop FileSystem (via `org.apache.hadoop.fs.Path`) for output location; no DB or queue interactions.

# Integrations
- **`CustomRecordBuilder`** (package `com.tcl.parquet.util2`): converts CSV → Avro.
- **`CustomParquetWriter`** (same package): persists Avro → Parquet.
- **`GetSchema`** (package `com.tcl.parquet.util2`): supplies the Avro schema.
- **Batch orchestration** (e.g., Airflow, Oozie, cron) invokes the utility with appropriate arguments.
- **Hadoop runtime** provides the filesystem implementation for the output `Path`.

# Operational Risks
- **Static mutable fields** (`inputFilePath`, etc.) are not thread‑safe; concurrent invocations could corrupt state. *Mitigation*: refactor to local variables or immutable configuration object.
- **No argument validation** – missing or malformed paths cause `ArrayIndexOutOfBoundsException` or IO errors. *Mitigation*: add explicit checks and user‑friendly error messages.
- **Unchecked IOException** – only stack trace printed; job may silently fail in orchestrator logs. *Mitigation*: propagate exception or exit with non‑zero status.
- **Hard‑coded Hadoop home** comment suggests environment dependency; missing `winutils` on Windows leads to runtime failure. *Mitigation*: externalize Hadoop configuration or ensure proper environment setup.

# Usage
```bash
# Compile (assuming Maven/Gradle with Hadoop & Avro deps)
javac -cp "$(hadoop classpath):avro.jar:slf4j-api.jar" \
    com/tcl/parquet/utility/ParquetConverterUtility.java

# Run
java -cp ".:$(hadoop classpath):avro.jar:slf4j-api.jar" \
    com.tcl.parquet.utility.ParquetConverterUtility \
    /data/kyc/input.csv \
    /data/kyc/output.parquet \
    /schemas/kyc.avsc
```
*Debug*: attach a debugger to `main`, or replace `e.printStackTrace()` with logging at `ERROR` level.

# Configuration
- **CLI arguments** (required):
  1. `inputFilePath` – absolute/relative path to source CSV.
  2. `outputFilePath` – destination Parquet path (local FS or HDFS URI).
  3. `schemaFilePath` – path to Avro schema (`.avsc`).
- **Environment**:
  - Hadoop native libraries (`hadoop.home.dir`) may be required on Windows.
  - Classpath must include Hadoop core, Avro, and SLF4J jars.

# Improvements
1. **Refactor to immutable configuration** – replace static fields with a POJO passed to methods; enables parallel execution and clearer lifecycle.
2. **Robust error handling & logging** – validate inputs, use SLF4J for structured logs, return appropriate exit codes, and optionally retry on transient IO failures.