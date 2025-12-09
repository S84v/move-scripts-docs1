# Summary
`QueryResult.java` is an auto‑generated Sqoop ORM class that maps the result set of an Oracle query to a Hadoop‑compatible record. It implements `DBWritable` (for JDBC read/write) and Hadoop `Writable` (for serialization in MapReduce). In production it is packaged into the Sqoop import job that extracts organization‑level data (e.g., via the **Org_details_Sqoop** job) and writes the rows to HDFS as delimited text or Avro/Parquet downstream.

---

# Key Components
- **Class `QueryResult`** – Extends `SqoopRecord`; core data holder.
- **Fields** – `STATUS`, `SECS_ID`, `PROPOSITION`, `PRODUCT_CODE` (String); `CDR`, `USAGE` (BigDecimal); `MONTH` (String).
- **Getter / Setter / Fluent (`with_…`)** – Accessors for each column.
- **`init0()` & `setters` map** – Populates a runtime map of field‑name → setter command for dynamic field assignment.
- **`readFields(ResultSet)` / `write(PreparedStatement, int)`** – JDBC ↔ object conversion.
- **`readFields(DataInput)` / `write(DataOutput)`** – Hadoop Writable serialization.
- **`toString(DelimiterSet, boolean)`** – Generates delimited text representation (default CSV).
- **`parse(...)`** – Parses a delimited record back into an object.
- **`getFieldMap()` / `setField(String, Object)`** – Reflective field map utilities used by Sqoop runtime.
- **`clone()`** – Shallow clone support.

---

# Data Flow
| Stage | Source / Destination | Operation |
|-------|----------------------|-----------|
| **Import** | Oracle DB (`ResultSet`) | `readFields(ResultSet)` populates object fields. |
| **Serialization** | Hadoop MapReduce / HDFS | `write(DataOutput)` serializes fields; `readFields(DataInput)` deserializes. |
| **Text Export** | HDFS delimited file | `toString()` produces CSV line; Sqoop’s `--as-textfile` uses this. |
| **Load Back** | HDFS → DB (optional) | `write(PreparedStatement, int)` writes object back to a JDBC target (rare in import‑only jobs). |
| **Dynamic Field Set** | Sqoop runtime (`--class-name`) | `setField(String, Object)` enables generic field population. |

*Side Effects*: None beyond object state mutation; all I/O is explicit via the caller (Sqoop runtime).

*External Services*: Oracle database (via JDBC), Hadoop Distributed File System (via Writable serialization), optional downstream Hive/Impala tables.

---

# Integrations
- **Sqoop Import Jobs** – Referenced by property files such as `Org_details_Sqoop.properties`. The Sqoop command includes `--class-name QueryResult` and `--target-dir` to store the serialized output.
- **Bash Configuration** – Shared `MNAAS_CommonProperties.properties` supplies DB connection strings (`ORACLE_JDBC_URL`, credentials) used by the Sqoop command that loads this class.
- **MapReduce Pipelines** – If the import is chained to a MapReduce job, the class serves as the key/value type for the mapper/reducer.
- **Hive/Impala** – Downstream scripts (e.g., `Move_Table_Compute_Stats`) may consume the CSV files generated from `QueryResult.toString()`.

---

# Operational Risks
- **Schema Drift** – Adding/removing columns in the source Oracle view breaks the generated class (field count mismatch). *Mitigation*: Regenerate the class after any DDL change; version‑control the generated source.
- **Precision Loss** – `BigDecimal` fields (`CDR`, `USAGE`) are written with a fixed scale of 2; mismatched DB precision can truncate data. *Mitigation*: Verify source column precision and adjust `writeBigDecimal` scale if needed.
- **Null Handling** – The generated code writes a boolean flag before each field; downstream parsers must respect this format. Corrupted files (missing flags) cause deserialization failures. *Mitigation*: Use Sqoop’s built‑in integrity checks (`--check-column`, `--incremental`).
- **Performance Overhead** – Reflection‑based `setters` map incurs a small runtime cost for dynamic field assignment. *Mitigation*: Prefer static setters in high‑throughput jobs or disable `--direct` mode if not needed.

---

# Usage
```bash
# Example Sqoop import using the generated class
sqoop import \
  --connect jdbc:oracle:thin:@${ORACLE_HOST}:${ORACLE_PORT}/${ORACLE_SID} \
  --username ${ORACLE_USER} \
  --password ${ORACLE_PWD} \
  --query "SELECT STATUS, SECS_ID, PROPOSITION, PRODUCT_CODE, CDR, USAGE, MONTH FROM ORG_TABLE WHERE \$CONDITIONS" \
  --target-dir ${MNAASLocalStagingPath}/org_details \
  --class-name QueryResult \
  --as-textfile \
  --fields-terminated-by ',' \
  --lines-terminated-by '\n' \
  --null-string '\\N' \
  --null-non-string '\\N' \
  --num-mappers 4
```
*Debug*: Run the generated jar locally with a mock `ResultSet` or use `sqoop eval` to validate column mapping.

---

# Configuration
- **Environment Variables / Props (via `MNAAS_CommonProperties.properties`)**
  - `ORACLE_JDBC_URL`, `ORACLE_USER`, `ORACLE_PWD`
  - `MNAASLocalStagingPath` – HDFS staging directory for the import.
  - `MNAASLocalLogPath` – Log location for Sqoop job output.
- **Sqoop‑specific Options**
  - `--class-name QueryResult`
  - `--as-textfile