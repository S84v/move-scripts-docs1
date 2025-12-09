# Summary
`MOVE_CURR_YEAR_COUNT.java` is an auto‑generated Sqoop ORM class that maps the Oracle table **MOVE_CURR_YEAR_COUNT** to a Hadoop‑compatible record. It implements `DBWritable` and `Writable`, enabling Sqoop to read rows from Oracle, serialize them, and load them into Hive/Impala tables used by the nightly Move‑Mediation pipelines (e.g., balance‑update and KYC reporting jobs).

# Key Components
- **Class `MOVE_CURR_YEAR_COUNT`** – extends `SqoopRecord`; core ORM representation.  
- **Field definitions** – `BUSS_UNIT_ID` (String), `PROD_STATUS` (String), `COUNT_SIM` (BigDecimal), `STATUS_MONTH_YEAR` (String).  
- **Getters / Setters / Fluent `with_` methods** – standard Java bean accessors.  
- **`readFields(ResultSet)` / `write(PreparedStatement)`** – JDBC ↔ object mapping for Sqoop import/export.  
- **`readFields(DataInput)` / `write(DataOutput)`** – Hadoop serialization for map‑reduce/shuffle.  
- **`parse(...)` methods** – text‑based parsing using `RecordParser` with input delimiter `;`.  
- **`toString(...)`** – CSV‑style output using delimiter `,`.  
- **`getFieldMap()` / `setField(String,Object)`** – reflective field access used by Sqoop runtime.  
- **`equals(Object)`** – logical equality based on all fields.  
- **Static delimiter definitions** – `__outputDelimiters` (`,`) and `__inputDelimiters` (`;`).  

# Data Flow
| Step | Source / Destination | Operation |
|------|----------------------|-----------|
| 1 | **Oracle DB** (`MOVE_CURR_YEAR_COUNT` table) | Sqoop launches a JDBC `SELECT`; each row is materialized into a `MOVE_CURR_YEAR_COUNT` instance via `readFields(ResultSet)`. |
| 2 | **Hadoop** (MapReduce / Spark job) | Instances are serialized (`write(DataOutput)`) and shipped to reducers or directly written to HDFS. |
| 3 | **Hive/Impala** (target table) | Sqoop’s `--hive-import` uses the class to generate a Hive table; data lands in a partitioned Hive table (partition key typically `notification_event_month` in related scripts). |
| Side‑effects | None beyond I/O; class itself is stateless. |
| External services | Oracle JDBC driver, Hadoop FileSystem, Hive Metastore. |

# Integrations
- **Sqoop import jobs** – referenced in pipeline definitions (e.g., `sqoop import --class-name MOVE_CURR_YEAR_COUNT ...`).  
- **Hive scripts** – downstream Hive/Impala jobs (e.g., `MNAAS_msisdn_details_kyc_bal_update_script.hql`) consume the Hive table populated by this import.  
- **Move‑Mediation nightly orchestration** – orchestrated by Oozie/Airflow; this class is part of the Java jar shipped to the cluster.  

# Operational Risks
- **Schema drift** – Adding/removing columns in Oracle breaks generated code; mitigated by regenerating the class on every schema change and version‑controlling the generated source.  
- **Precision loss** – `COUNT_SIM` stored as `BigDecimal(?,2)`; mismatched precision in Hive may truncate values. Enforce matching Hive column type (`DECIMAL(?,2)`).  
- **Null handling** – Custom boolean flag before each field; any change in delimiter or record format can cause parsing errors. Validate input delimiters in the Sqoop job configuration.  
- **Performance** – Large `BigDecimal` serialization can be CPU‑intensive; monitor mapper memory usage and consider column pruning if not all fields are needed downstream.  

# Usage
```bash
# Compile (if not using Sqoop’s auto‑generated jar)
javac -cp $(sqoop classpath) MOVE_CURR_YEAR_COUNT.java

# Example Sqoop import
sqoop import \
  --connect jdbc:oracle:thin:@oracle-host:1521/ORCL \
  --username prod_user \
  --password prod_pwd \
  --table MOVE_CURR_YEAR_COUNT \
  --class-name MOVE_CURR_YEAR_COUNT \
  --target-dir /user/hive/warehouse/move_curr_year_count \
  --hive-import \
  --hive-table move_curr_year_count \
  --hive-partition-key notification_event_month \
  --hive-partition-value $(date +%Y-%m) \
  --null-string '\\N' \
  --null-non-string '\\N' \
  --fields-terminated-by ',' \
  --lines-terminated-by '\n'
```
*Debug*: Run the class locally with a mock `ResultSet` or use `sqoop eval` to fetch a single row and invoke `readFields` manually.

# Configuration
- **Environment variables** (set by orchestration):  
  - `ORACLE_JDBC_URL`, `ORACLE_USER`, `ORACLE_PASSWORD` – Oracle connection.  
  - `HIVE_CONF_DIR` – Hive metastore configuration.  
  - `SQOOP_OPTS` – JVM options for Sqoop.  
- **Config files**:  
  - `sqoop-site.xml` – connector definitions.  
  - `hive-site.xml` – Hive table/partition metadata.  
- **HiveConf variable** `hivetable` – supplied by the calling script to specify the target Hive table name.  

# Improvements
1. **Add explicit schema versioning** – embed a static `SCHEMA_VERSION` constant and validate it at runtime against the Oracle metadata to detect drift early.  
2. **Implement `hashCode()`** consistent with `equals()` and provide unit tests for serialization/deserialization round‑trips to guarantee data integrity across Hadoop pipelines.  