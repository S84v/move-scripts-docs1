# Summary
`MOVE_CURR_MONTH_COUNTT` is an auto‑generated Sqoop ORM class that maps the Oracle table **MOVE_CURR_MONTH_COUNTT** to a Hadoop‑compatible record. It implements `DBWritable` and `Writable`, providing JDBC‑based read/write, Hadoop serialization, and text‑based parsing. In production it is used by Sqoop import jobs to extract monthly SIM‑count metrics from Oracle and load them into Hive/Impala tables that feed downstream Move‑Mediation analytics (e.g., the KYC‑balance update scripts).

# Key Components
- **Class `MOVE_CURR_MONTH_COUNTT`** – Extends `SqoopRecord`; core data container.  
- **Field definitions** – `BUSS_UNIT_ID` (String), `PROD_STATUS` (String), `COUNT_SIM` (BigDecimal), `STATUS_DATE` (String).  
- **`init0()` & `setters` map** – Runtime reflection for generic field setting (`setField`).  
- **JDBC read/write** – `readFields(ResultSet)`, `write(PreparedStatement, int)`.  
- **Hadoop serialization** – `write(DataOutput)`, `readFields(DataInput)`.  
- **Text parsing & toString** – `parse(...)` overloads, `toString(DelimiterSet)`.  
- **Field map utilities** – `getFieldMap()`, `setField(String, Object)`.  
- **Delimiter definitions** – Output (comma) and input (semicolon) delimiters for CSV handling.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| **Sqoop Import** | Oracle JDBC `ResultSet` from `MOVE_CURR_MONTH_COUNTT` table | `readFields(ResultSet)` populates object fields; `write(DataOutput)` serializes for Hadoop shuffle | Serialized records written to HDFS (intermediate files) |
| **MapReduce / Hive Load** | Serialized records from HDFS | Deserialized via `readFields(DataInput)`; optionally parsed from CSV using `parse(...)` | Hive/Impala table rows (partitioned by `notification_event_month` in downstream scripts) |
| **Downstream Analytics** | Hive tables populated by import job | Queried by HiveQL scripts (`MNAAS_msisdn_details_kyc_bal_update_script.hql`, etc.) | Reporting dashboards, KPI calculations |

External services:
- Oracle database (source)
- Hadoop Distributed File System (intermediate storage)
- Hive Metastore (target tables)

# Integrations
- **Sqoop job definitions** – referenced via `--class-name MOVE_CURR_MONTH_COUNTT` in the Move‑Mediation nightly import pipeline.  
- **Hive scripts** – the imported Hive table is later read by `MNAAS_msisdn_details_kyc_bal_update_script.hql` (partitioned by `notification_event_month`).  
- **Move‑Mediation orchestration** – invoked by the overall Move‑Mediation workflow (e.g., Oozie or Airflow DAG) that schedules the Sqoop import before the Hive ETL steps.

# Operational Risks
- **Schema drift** – Adding/removing columns in the Oracle source breaks the generated class. *Mitigation*: Regenerate the ORM after any DDL change; version‑control the class.  
- **Precision loss** – `COUNT_SIM` stored as `BigDecimal` with scale 2; mismatched precision may cause truncation. *Mitigation*: Validate Oracle column definition matches the generated precision.  
- **Null handling** – Custom boolean flag for nulls; any change in upstream null semantics could cause `NullPointerException`. *Mitigation*: Include unit tests for null scenarios.  
- **Class name typo** (`COUNTT` vs expected `COUNT`) may cause confusion in downstream scripts. *Mitigation*: Align naming conventions; add alias or wrapper if required.

# Usage
```bash
# Example Sqoop import command
sqoop import \
  --connect jdbc:oracle:thin:@//oracle-host:1521/PRODDB \
  --username prod_user \
  --password **** \
  --table MOVE_CURR_MONTH_COUNTT \
  --class-name MOVE_CURR_MONTH_COUNTT \
  --target-dir /user/hive/warehouse/mnaas.db/move_curr_month_count \
  --as-avrodatafile \
  --m 4 \
  --null-string '\\N' \
  --null-non-string '\\N'
```
*Debugging*:  
- Run `sqoop import --verbose` to see generated SQL.  
- Use a small `--where "ROWNUM <= 10"` clause and inspect the Avro/CSV output with `hdfs dfs -cat`.  
- Instantiate the class in a unit test, call `readFields(ResultSet)` with a mock, and verify `toString()` output.

# Configuration
- **Oracle connection parameters** – supplied via Sqoop `--connect`, `--username`, `--password`.  
- **Hive target directory** – defined in the Sqoop `--target-dir` argument; downstream Hive scripts expect the table in `mnaas` database.  
- **HiveConf variable `hivetable`** – used by downstream Hive scripts to reference the imported table name.  
- **Delimiter settings** – hard‑coded in the class (`__outputDelimiters` = comma, `__inputDelimiters` = semicolon). Adjust only if downstream parsers change.

# Improvements
1. **Add explicit JavaDoc** for each public method and field, clarifying expected data types (e.g., `STATUS_DATE` should be a `java.sql.Date` rather than a raw `String`).  
2. **Rename class to match source table** (`MOVE_CURR_MONTH_COUNT`) or provide a thin wrapper that forwards