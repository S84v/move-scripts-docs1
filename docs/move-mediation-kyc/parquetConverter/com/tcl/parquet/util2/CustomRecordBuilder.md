# Summary
`CustomRecordBuilder` reads a CSV file, maps each row to an Avro `GenericData.Record` using a supplied Avro schema, and returns a list of those records. It is used in the MOVE‑KYC pipeline to convert raw KYC CSV extracts into Avro records that can subsequently be written to Parquet files by `CustomParquetWriter`.

# Key Components
- **Class `CustomRecordBuilder`**
  - `public static List<GenericData.Record> sampleData` – static accumulator for built records (shared across calls).
  - `public static List<GenericData.Record> buildRecord(String inputFilePath, String schemaFilePath)` – core method:
    - Reads CSV header line.
    - Reads remaining CSV rows into `List<String[]> csvColumn`.
    - For each row, creates a new `GenericData.Record` using `GetSchema.getCustomSchema(schemaFilePath)`.
    - Populates the record fields by matching header names to column values.
    - Adds each record to `sampleData`.
    - Returns `sampleData`.

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| 1 | `inputFilePath` (path to CSV) | BufferedReader reads header and rows | `header[]`, `csvColumn` (list of string arrays) | File I/O (read) |
| 2 | `schemaFilePath` (path to Avro schema JSON) | `GetSchema.getCustomSchema` parses schema into Avro `Schema` object | Avro `Schema` | File I/O (read) |
| 3 | Each CSV row | Create `GenericData.Record` with schema, populate fields via `record.put` | `GenericData.Record` per row | Adds record to static `sampleData` |
| 4 | – | Return `sampleData` | `List<GenericData.Record>` | Persistent static list may retain records across invocations |

# Integrations
- **`GetSchema`** – utility class (not shown) that loads an Avro schema from `schemaFilePath`. Required for record creation.
- **`CustomParquetWriter`** – consumes the `List<GenericData.Record>` produced by `buildRecord` to write Parquet files.
- **KYC ingestion pipeline** – invoked after CSV extraction from upstream KYC sources; feeds downstream analytics.

# Operational Risks
- **Static mutable state (`sampleData`)** – records accumulate across multiple calls, leading to memory bloat and duplicate data. *Mitigation*: Clear list at method start or return a new list instance.
- **No validation of CSV row length vs header length** – `ArrayIndexOutOfBoundsException` if rows have missing/extra columns. *Mitigation*: Validate column count before populating record.
- **Unchecked I/O exceptions** – method declares `FileNotFoundException, IOException` but callers may not handle them, causing pipeline failure. *Mitigation*: Wrap I/O in try‑catch and propagate a domain‑specific exception.
- **Schema parsing on every row** – `GetSchema.getCustomSchema` called inside loop, potentially re‑reading the schema file each iteration. *Mitigation*: Load schema once before the loop.
- **Character encoding assumptions** – uses default platform encoding; may corrupt non‑ASCII data. *Mitigation*: Specify UTF‑8 when constructing `FileReader`.

# Usage
```java
import com.tcl.parquet.util2.CustomRecordBuilder;
import org.apache.avro.generic.GenericData;
import java.util.List;

public class TestBuilder {
    public static void main(String[] args) throws Exception {
        String csvPath = "/data/kyc_input.csv";
        String schemaPath = "/schemas/kyc_avro_schema.json";

        List<GenericData.Record> records = CustomRecordBuilder.buildRecord(csvPath, schemaPath);
        System.out.println("Built " + records.size() + " Avro records.");
    }
}
```
Run with classpath including Avro libraries and the project’s compiled classes.

# Configuration
- No environment variables used.
- External files required:
  - CSV file at `inputFilePath`.
  - Avro schema JSON file at `schemaFilePath`.
- Implicit dependency on `GetSchema` class for schema loading.

# Improvements
1. **Eliminate static `sampleData`** – instantiate a local `List<GenericData.Record>` inside `buildRecord` and return it, ensuring thread‑safety and preventing memory leaks.
2. **Optimize schema loading** – retrieve the Avro `Schema` once before the row‑processing loop and reuse it for all record creations. Add validation for CSV row length and enforce UTF‑8 encoding when reading files.