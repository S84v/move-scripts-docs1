**File:** `mediation-ddls\mnaas_geneva_usage.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates an **external** table named `geneva_usage` that maps raw Geneva‑originated usage records stored as CSV files in HDFS (`/user/hive/warehouse/mnaas.db/geneva_usage`). The table schema captures call‑level identifiers, timestamps, service identifiers, usage type, status, and source. Because the table is external and marked with `external.table.purge='true'`, Hive will not delete the underlying files when the table is dropped, but it will purge metadata on drop. Down‑stream mediation jobs (e.g., `mnaas_gen_usage_product.hql`, `mnaas_billing_traffic_usage.hql`) read from this table to enrich, filter, and aggregate usage for billing and reporting.

---

## 2. Core Objects Defined in the File

| Object | Type | Responsibility |
|--------|------|-----------------|
| `geneva_usage` | External Hive table | Provides a logical view over raw Geneva CSV files; columns: `call_id`, `event_date`, `secs_id`, `usage_type`, `event_status`, `event_source`. |
| Table properties (`bucketing_version`, `external.table.purge`, `last_modified_by`, `last_modified_time`, `transient_lastDdlTime`) | Metadata | Control table behavior (purge on drop) and audit information. |
| SerDe configuration (`field.delim=','`, `line.delim='\n'`) | Serialization/Deserialization | Instruct Hive how to parse the CSV files. |
| Input/Output format (`TextInputFormat` / `HiveIgnoreKeyTextOutputFormat`) | File handling | Define how Hive reads the raw text files. |
| `LOCATION` | HDFS path | Points to the physical directory that holds the source CSV files. |

*No procedural code (functions, procedures) is present; the file is purely declarative.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Input data** | CSV files placed (by upstream ETL or SFTP ingest) in `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/geneva_usage`. Expected delimiter is a comma, newline‑terminated rows. |
| **Output** | Logical table `geneva_usage` exposing the CSV rows to Hive queries. No data transformation occurs here. |
| **Side effects** | - Registers metadata in the Hive metastore.<br>- May trigger Hive metastore replication if enabled.<br>- Because `external.table.purge='true'`, dropping the table will also delete the underlying files (purge). |
| **Assumptions** | - The HDFS namenode `NN-HA1` is reachable and the path exists with appropriate permissions (Hive service user can read/write).<br>- Source CSV files conform to the column order defined.<br>- No partitioning is required for this raw table (full scan is acceptable for downstream jobs).<br>- Downstream scripts will handle data quality, filtering, and enrichment. |
| **External services** | - HDFS (namenode `NN-HA1`).<br>- Hive Metastore (for table registration).<br>- Possibly an upstream ingestion service (SFTP, Kafka Connect, or batch copy) that lands files into the location. |

---

## 4. Integration Points with Other Scripts / Components

| Connected Component | Interaction |
|---------------------|-------------|
| **`mnaas_gen_usage_product.hql`** | Reads from `geneva_usage` (or a derived staging view) to generate usage‑product records for billing. |
| **`mnaas_billing_traffic_usage.hql`** | May join `geneva_usage` with other usage sources (e.g., CDRs) to build a consolidated usage view. |
| **Ingestion pipelines** (e.g., nightly SFTP drop, Spark batch job) | Place raw CSV files into the HDFS location before this DDL is executed (or after, as the table is external). |
| **Data quality / validation jobs** (not listed) | Could read from `geneva_usage` to validate schema, null checks, or duplicate detection before downstream processing. |
| **Hive Metastore backup / replication** | The table definition is stored in the metastore; any replication mechanism will propagate it to standby clusters. |
| **Monitoring / alerting** | Scripts that check file arrival timestamps in the HDFS path may raise alerts if no new files appear within the expected window. |

*Because the table is external, downstream jobs can be re‑run without re‑creating the table, provided the location remains unchanged.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Incorrect CSV format** (missing columns, extra delimiters) | Downstream jobs fail or produce corrupt billing data. | Implement a schema‑validation Spark/MapReduce job that runs after file landing; reject or quarantine malformed files. |
| **Stale or missing data** (no new files in HDFS) | Billing pipelines may miss usage for a period. | Add a monitoring job that checks file timestamps and file count; raise alerts if no new files for > N hours. |
| **Accidental table drop** (purge deletes raw files) | Permanent loss of raw usage data. | Restrict DROP privileges; enforce a change‑management process; consider removing `external.table.purge='true'` if raw data must be retained after table drop. |
| **Permission drift** (Hive user loses HDFS read/write rights) | Table becomes unreadable, causing pipeline failures. | Periodic permission audit; use ACLs or Kerberos delegation tokens managed by a central security team. |
| **HDFS namespace saturation** (unbounded growth of raw files) | Storage exhaustion, performance degradation. | Implement a retention policy (e.g., delete files older than 90 days) via a scheduled HDFS cleanup job. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Hive CLI / Beeline access with a user that has `CREATE` privileges on the `mnaas` database.  
   - HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/geneva_usage` exists and is writable.

2. **Execution**  
   ```bash
   hive -f mediation-ddls/mnaas_geneva_usage.hql
   ```
   *or* via Beeline:
   ```bash
   beeline -u jdbc:hive2://<hive-host>:10000/default -f mediation-ddls/mnaas_geneva_usage.hql
   ```

3. **Verification**  
   ```sql
   SHOW CREATE TABLE geneva_usage;
   DESCRIBE FORMATTED geneva_usage;
   SELECT COUNT(*) FROM geneva_usage LIMIT 1;
   ```
   - The first query confirms the DDL persisted correctly.  
   - The second shows storage location and SerDe settings.  
   - The third checks that Hive can read at least one row (returns 0 if no files yet).

4. **Debugging Tips**  
   - If `SHOW CREATE TABLE` fails, verify Hive metastore connectivity.  
   - If `SELECT` returns errors like *“Cannot parse line …”*, inspect the raw CSV files for delimiter issues.  
   - Use `hdfs dfs -ls <location>` to confirm files are present and have correct permissions.  
   - Check Hive logs (`/var/log/hive/hive.log`) for detailed parsing errors.

---

## 7. External Configuration / Environment Variables

| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` (or equivalent) | Determines Hive metastore connection; must point to the correct cluster config. |
| `HADOOP_USER_NAME` (or Kerberos principal) | Provides HDFS access rights for the Hive service user. |
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` URI; any change to the HDFS cluster requires updating the DDL or using a variable substitution mechanism (e.g., `${hdfs.namenode}`) if the project supports it. |
| `mnaas.db` (Hive database) | The script assumes the database already exists; creation is handled elsewhere. |

If the project uses a templating system (e.g., Jinja, Velocity) for DDL generation, these values may be injected at runtime; otherwise they are static.

---

## 8. Suggested Improvements / TODOs

1. **Add Partitioning** – If the volume of Geneva usage grows, consider partitioning the external table by `event_date` (or `event_month`) to improve query performance for downstream jobs.  
2. **Replace Hard‑Coded Namenode** – Externalize the HDFS URI into a configuration property (e.g., `${geneva.usage.hdfs.uri}`) so that environment changes (cluster migration, HA) require only a config update, not a DDL edit.  

---