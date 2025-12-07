**File:** `mediation-ddls\mnaas_ipvprobe_raw_daily.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the external table **`mnaas.ipvprobe_raw_daily`**. The table stores raw daily IPvProbe event records that are ingested from an HDFS location (`/user/hive/warehouse/mnaas.db/ipvprobe_raw_daily`). Data is semicolon‑delimited, timestamp‑typed, and partitioned by a string field `partition_date`. Because the table is *external* and has `external.table.purge='true'`, dropping the table will also delete the underlying files. The table is intended to be the landing zone for daily IPvProbe logs that downstream mediation jobs (e.g., usage aggregation, billing) will read.

---

## 2. Important Objects & Their Responsibilities
| Object | Type | Responsibility |
|--------|------|-----------------|
| `ipvprobe_raw_daily` | Hive external table | Holds raw IPvProbe daily logs; provides a partitioned view for downstream processing. |
| `partition_date` | Partition column (string) | Enables efficient pruning by day; used by downstream jobs to target specific dates. |
| `location` | HDFS path | Physical storage location for the raw files; must be accessible by Hive/Impala. |
| Table properties (`DO_NOT_UPDATE_STATS`, `external.table.purge`, etc.) | Hive metadata | Control statistics collection, automatic data purge on DROP, and Impala catalog integration. |
| SerDe (`LazySimpleSerDe`) & SerDe properties (`field.delim=';'`, `line.delim='\n'`) | Input format definition | Define how raw text files are parsed into columns. |

*No procedural code (functions, classes) exists in this file; it is a pure DDL definition.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - Raw daily IPvProbe log files placed under the HDFS location `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/ipvprobe_raw_daily`.<br>- Files must be plain‑text, semicolon‑delimited, with a newline terminator.<br>- Each record must contain the columns defined in the DDL, in the exact order. |
| **Outputs** | - Hive/Impala metadata entry for `mnaas.ipvprobe_raw_daily`.<br>- Partition directories (`partition_date=YYYYMMDD`) created automatically when data is loaded (via `ALTER TABLE … ADD PARTITION` or `MSCK REPAIR TABLE`). |
| **Side Effects** | - Registers the external table in the Hive Metastore and Impala catalog.<br>- Because `external.table.purge='true'`, a `DROP TABLE` will delete the underlying HDFS files (risk of data loss). |
| **Assumptions** | - HDFS namenode alias `NN-HA1` resolves correctly in the execution environment.<br>- The Hive Metastore service is reachable and has write permissions for the `mnaas` database.<br>- Impala catalog service IDs (`impala.events.catalogServiceId`, `catalogVersion`) are up‑to‑date; they are only informational here. |
| **External Services** | - HDFS (for storage).<br>- Hive Metastore (metadata).<br>- Impala (optional query engine). |
| **Environment Variables / Config** | - `HIVE_CONF_DIR` / `IMPALA_HOME` may be required for the client invoking the script.<br>- No hard‑coded variables inside the script; the HDFS URI (`hdfs://NN-HA1`) is a cluster‑specific constant that may be overridden by a cluster‑wide configuration. |

---

## 4. Integration Points with Other Scripts / Components
| Connected Component | Relationship |
|---------------------|--------------|
| **Downstream mediation jobs** (e.g., `mnaas_gen_usage_product.hql`, `mnaas_fiveg_nsa_count_report.hql`) | These scripts typically read from `ipvprobe_raw_daily` (or its derived tables) to compute usage, generate billing records, or produce reports. |
| **Ingestion pipelines** (e.g., nightly SFTP/FTP drop, Kafka → HDFS consumer) | External processes place raw IPvProbe files into the HDFS location. A separate “add‑partition” script (often a Bash/Airflow task) runs `ALTER TABLE ipvprobe_raw_daily ADD PARTITION (partition_date='20231204') LOCATION '.../partition_date=20231204'`. |
| **Data quality / validation jobs** | Scripts that run `SELECT COUNT(*) FROM ipvprobe_raw_daily WHERE ...` to verify row counts, schema compliance, or duplicate detection. |
| **Metadata refresh** | Impala may require `INVALIDATE METADATA ipvprobe_raw_daily;` or `REFRESH ipvprobe_raw_daily;` after new partitions are added. |
| **Retention / purge processes** | A housekeeping job may drop old partitions (`ALTER TABLE … DROP PARTITION …`) to enforce data retention policies. Because the table is external, the underlying files are removed automatically. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental data loss on DROP** | `external.table.purge='true'` deletes files when the table is dropped. | Enforce change‑control: require a review before any `DROP TABLE` operation. Consider removing the purge flag if the table is ever recreated. |
| **Partition drift / missing partitions** | Queries may scan the entire table if partitions are not added correctly. | Automate partition addition (e.g., via `MSCK REPAIR TABLE` or a scheduled `ADD PARTITION` job). Verify partition existence after each ingestion run. |
| **Schema mismatch** (incoming files with extra/missing columns) | Query failures downstream. | Add a pre‑ingest validation step that samples a file and runs `SELECT * FROM tmp_external_table LIMIT 1`. |
| **HDFS path permission changes** | Table becomes unreadable. | Document required HDFS ACLs (`rwx` for Hive/Impala service accounts). Periodically audit permissions. |
| **Growth of raw data** | Storage exhaustion. | Implement a retention policy (e.g., keep 90 days) and schedule partition drops. Monitor HDFS usage alerts. |
| **SerDe delimiter change** | Parsing errors. | Keep delimiter definition in a central config file; version‑control changes. |

---

## 6. Example Execution & Debugging Workflow

### 6.1 Running the DDL
```bash
# Using Hive CLI
hive -f mediation-ddls/mnaas_ipvprobe_raw_daily.hql

# Or using Impala Shell (DDL is compatible)
impala-shell -i impala-coordinator.example.com -f mediation-ddls/mnaas_ipvprobe_raw_daily.hql
```

### 6.2 Verifying Creation
```sql
-- In Hive or Impala
DESCRIBE FORMATTED mnaas.ipvprobe_raw_daily;
SHOW PARTITIONS mnaas.ipvprobe_raw_daily;
```
Check that:
* `Location` points to the expected HDFS path.
* `Table Type` is `EXTERNAL_TABLE`.
* No partitions are listed yet (they will appear after data load).

### 6.3 Adding a Partition (typical daily step)
```sql
ALTER TABLE mnaas.ipvprobe_raw_daily
ADD IF NOT EXISTS PARTITION (partition_date='20231204')
LOCATION 'hdfs://NN-HA1/user/hive/warehouse/mnaas.db/ipvprobe_raw_daily/partition_date=20231204';
```
*Or* run `MSCK REPAIR TABLE mnaas.ipvprobe_raw_daily;` if the directory layout follows Hive conventions.

### 6.4 Debugging Common Issues
| Symptom | Likely Cause | Debug Steps |
|---------|--------------|-------------|
| `File not found` or `Permission denied` | HDFS path missing or ACLs wrong | `hdfs dfs -ls <path>`; check ACLs with `hdfs dfs -getfacl`. |
| `Invalid column` errors when querying | Incoming file delimiter mismatch | Sample a raw file, verify fields separated by `;`. |
| No rows returned after load | Partition not added | Run `SHOW PARTITIONS`; add missing partition. |
| Impala “Table not found” after Hive DDL | Impala catalog not refreshed | `INVALIDATE METADATA mnaas.ipvprobe_raw_daily;` or restart Impala daemon. |

---

## 7. External Configuration / Environment Dependencies
| Item | Usage |
|------|-------|
| `hdfs://NN-HA1` | Hard‑coded namenode URI; may be overridden by cluster‑wide `fs.defaultFS` in `core-site.xml`. |
| Hive Metastore connection (`hive.metastore.uris`) | Required for DDL execution; defined in `hive-site.xml`. |
| Impala catalog service IDs (`impala.events.catalogServiceId`, `catalogVersion`) | Informational; kept in sync automatically by Impala. |
| SerDe properties (`field.delim=';'`) | Must match the delimiter used by upstream data producers (e.g., SFTP drop scripts). |
| `external.table.purge` | Controls delete‑on‑drop behavior; set in the DDL, not external config. |

No additional config files are referenced directly in this script, but the surrounding orchestration (Airflow, Oozie, etc.) will typically supply the HDFS ingestion path and partition values.

---

## 8. Suggested TODO / Improvements
1. **Automate Partition Discovery** – Add a companion script (Bash/Python/Airflow) that scans the HDFS directory for new `partition_date=` folders and runs `ALTER TABLE … ADD PARTITION` automatically, then issues `REFRESH` in Impala.
2. **Add Column Comments & Data Quality Constraints** – Extend the DDL with `COMMENT` clauses for each column and, where feasible, `NOT NULL` constraints or `CHECK` expressions to catch malformed records early. This improves downstream data quality checks.