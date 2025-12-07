**File:** `mediation-ddls\mnaas_ra_calldate_traffic_table.hql`  

---

## 1. High‑Level Summary
This script creates an **external Hive table** named `ra_calldate_traffic_table` in the `mnaas` database. The table stores per‑record traffic metrics (CDR count, bytes, call type, etc.) that are later consumed by a family of “reject” and reporting scripts (e.g., `mnaas_ra_calldate_*_reject.hql`). Because it is external, the underlying files live in a fixed HDFS directory and are not removed when the table is dropped; the `external.table.purge='true'` flag ensures that a DROP will also delete the files, providing a controlled clean‑up path.

---

## 2. Core Artifact(s)

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `createtab_stmt` | Hive DDL statement | Defines the schema, storage format, location, and table properties for `ra_calldate_traffic_table`. |
| `ra_calldate_traffic_table` | External Hive table | Serves as the canonical source for traffic‑level CDR data used by downstream reject‑handling and reporting jobs. |

*No procedural code (functions, classes) is present in this file; the DDL is the sole executable artifact.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | - Raw CDR files (text, delimited) that are landed in the HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/ra_calldate_traffic_table` prior to table creation. <br> - Implicit Hive metastore connection (configured via Hive/Beeline client). |
| **Outputs** | - Hive metadata entry for `mnaas.ra_calldate_traffic_table`. <br> - No data is written by this script; data is expected to already exist in the target HDFS location. |
| **Side Effects** | - Registers the external table in the Hive metastore. <br> - Sets table‑level properties (stats, purge flag, Impala catalog IDs). |
| **Assumptions** | - HDFS namenode `NN-HA1` is reachable and the path exists (or can be created). <br> - The text files conform to the column order and data types defined. <br> - Hive/Impala versions support the listed SerDe and InputFormat classes. <br> - No partitioning is required for the current workload (all data lives in a flat directory). |

---

## 4. Integration Points & Connectivity

| Connected Component | How the Connection Occurs |
|---------------------|---------------------------|
| **Reject‑generation scripts** (`*_reject.hql`) | These scripts read from `ra_calldate_traffic_table` (or from derived tables) to flag records that fail validation (e.g., IMSI reject, partner reject). They rely on the column names and data types defined here. |
| **ETL ingestion jobs** (e.g., Spark, MapReduce, or Hive INSERT OVERWRITE) | Prior to running this DDL, upstream jobs copy or generate raw CDR files into the HDFS location. The table must exist before downstream SELECTs are issued. |
| **Impala** | The table properties include Impala catalog IDs, indicating that Impala queries are expected. Any Impala service must be refreshed (`INVALIDATE METADATA`) after creation. |
| **Data Quality / Monitoring** | External monitoring tools may query `SHOW TABLE EXTENDED LIKE 'ra_calldate_traffic_table'` to verify `external.table.purge` and location. |
| **Orchestration (Oozie / Airflow)** | The DDL is typically invoked as a “create table” task in a workflow that runs before any transformation steps that depend on the table. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream jobs change column order or add fields without updating this DDL. | Query failures, data mis‑interpretation. | Enforce schema version control; add a validation step (`DESCRIBE FORMATTED`) before downstream jobs. |
| **Missing or incorrect HDFS path** – directory does not exist or permissions are insufficient. | Table creation succeeds but queries return empty set or error. | Include a pre‑flight check (`hdfs dfs -test -d <path>`) in the orchestration script. |
| **Purge flag misuse** – accidental `DROP TABLE` removes raw files. | Irrecoverable loss of source CDRs. | Restrict DROP privileges; add a safeguard step that backs up the directory before any DROP. |
| **Performance degradation** – flat table without partitions leads to full scans. | Long query runtimes, resource contention. | Consider adding partitioning (e.g., by `bill_month` or `callmonth`) in a future revision. |
| **Impala catalog out‑of‑sync** – Impala queries see stale metadata. | Stale results or “Table not found” errors. | Run `INVALIDATE METADATA` or `REFRESH` in Impala after DDL execution. |

---

## 6. Running / Debugging the Script

1. **Typical execution (operator)**  
   ```bash
   # Using Hive CLI
   hive -f mediation-ddls/mnaas_ra_calldate_traffic_table.hql

   # Or via Beeline (preferred for Kerberos)
   beeline -u "jdbc:hive2://<hive-server>:10000/mnaas" -f mediation-ddls/mnaas_ra_calldate_traffic_table.hql
   ```

2. **Verification steps**  
   ```sql
   -- Verify table exists and properties
   SHOW CREATE TABLE mnaas.ra_calldate_traffic_table;
   DESCRIBE FORMATTED mnaas.ra_calldate_traffic_table;

   -- Check that files are present
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/ra_calldate_traffic_table
   ```

3. **Debugging common issues**  
   * *“Table not found”* – Ensure the Hive metastore is reachable and the database `mnaas` exists.  
   * *“Permission denied”* – Verify the executing user has read/write access to the HDFS directory.  
   * *Data type mismatch* – Sample a few rows (`SELECT * FROM ... LIMIT 10;`) and compare to expected formats.  

4. **Integration test** (developer)  
   ```bash
   # 1) Create a temporary HDFS folder with a small test file matching the schema
   hdfs dfs -mkdir -p /tmp/ra_calldate_traffic_table_test
   hdfs dfs -put test_data.txt /tmp/ra_calldate_traffic_table_test/

   # 2) Modify the DDL locally to point to the test folder, run it, then query
   hive -e "DROP TABLE IF EXISTS mnaas.ra_calldate_traffic_table_test;
            CREATE EXTERNAL TABLE mnaas.ra_calldate_traffic_table_test ( ... ) LOCATION '/tmp/ra_calldate_traffic_table_test';"
   hive -e "SELECT COUNT(*) FROM mnaas.ra_calldate_traffic_table_test;"
   ```

---

## 7. External Configuration / Environment Variables

| Config Item | Usage in this Script |
|-------------|----------------------|
| `NN-HA1` (namenode hostname) | Hard‑coded in the `LOCATION` URI. If the cluster’s namenode changes, the script must be edited or templated. |
| Hive/Impala connection parameters (e.g., `hive.metastore.uris`, `impala.catalog.service.id`) | Not referenced directly but required by the client executing the DDL. |
| Kerberos principal / keytab | Required for secure clusters; not visible in the script but must be set in the execution environment. |

*If the organization uses a templating system (e.g., Jinja, Velocity) to inject the namenode or database name, the raw file would contain placeholders; in the current version the values are static.*

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning** – Introduce `PARTITIONED BY (bill_month STRING, callmonth STRING)` to enable pruning for monthly reports and reduce scan time.  
2. **Externalize the HDFS location** – Replace the hard‑coded `LOCATION` with a variable (e.g., `${RA_CALLDATA_PATH}`) sourced from a central configuration file or environment variable, making the script portable across environments (dev/test/prod).  

--- 

*End of documentation.*