**File:** `mediation-ddls\mnaa_Kyc_mapping_snapshot.hql`

---

### 1. High‑Level Summary
This script creates an **external Hive/Impala table** named `kyc_mapping_snapshot` that maps vehicle identification numbers (VIN) to their Know‑Your‑Customer (KYC) status and related device/SIM metadata. The table is stored as Parquet files in HDFS under the `mnaas.db` warehouse. By exposing the data as an external table, downstream analytics, compliance checks, and provisioning processes can query the latest KYC snapshot without moving or duplicating the underlying files.

---

### 2. Core Object(s) & Responsibility

| Object | Type | Responsibility |
|--------|------|-----------------|
| `kyc_mapping_snapshot` | External Hive table | Provides a read‑only, schema‑enforced view over Parquet files that contain KYC‑related attributes for each VIN (country, status, dates, device & SIM identifiers, dealer info, etc.). |
| Table properties (e.g., `spark.sql.sources.schema.*`, `impala.events.*`) | Metadata | Enable Spark and Impala to correctly interpret the Parquet schema and support catalog synchronization. |

*No procedural code (functions, procedures) is defined in this file.*

---

### 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - Parquet files located at `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/kyc_mapping_snapshot`.<br>- Implicitly depends on upstream ETL jobs that write those Parquet files (e.g., a KYC feed ingestion pipeline). |
| **Outputs** | - Hive/Impala metadata entry for `kyc_mapping_snapshot`.<br>- Queryable dataset for downstream jobs (SQL, Spark, Impala). |
| **Side Effects** | - Registers the table in the Hive metastore (no data movement).<br>- May trigger automatic statistics collection if enabled (`STATS_GENERATED='TASK'`). |
| **Assumptions** | - HDFS path `NN-HA1` is reachable and the service account has read permission.<br>- Parquet files conform exactly to the schema defined in `TBLPROPERTIES`.<br>- No partitioning is required for current query patterns.<br>- The Hive metastore is synchronized with Impala/ Spark catalog (as indicated by `impala.events.*`). |

---

### 4. Integration Points (How This File Connects to the Rest of the System)

| Connected Component | Interaction |
|---------------------|-------------|
| **Upstream KYC Feed Ingestion** (e.g., a Spark job that reads a CSV/JSON feed, transforms it, and writes Parquet to the same HDFS location). | Supplies the physical files that this external table points to. |
| **Reporting / Analytics Scripts** (e.g., `market_management_report_sims.hql`, `gbs_curr_avgconv_rate.hql`). | May join `kyc_mapping_snapshot` to enrich device/SIM metrics with KYC status. |
| **Compliance / Audit Jobs** (custom scripts that validate KYC completeness). | Query the table to produce compliance dashboards or alerts. |
| **Device Provisioning Orchestration** (e.g., a workflow that checks KYC before activating a SIM). | Reads `kycstatus`, `kycreqyured`, and date fields to decide activation eligibility. |
| **Metadata Management** (Hive Metastore, Impala Catalog, Spark Session). | The table definition is stored in the metastore; Impala and Spark rely on the `impala.events.*` and `spark.sql.*` properties for schema discovery. |

*Note:* The exact script names that reference this table are not present in the current repository snapshot; you would locate them by searching for `kyc_mapping_snapshot` across the code base.

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream ETL adds/removes columns without updating this DDL. | Queries fail or return incorrect results. | Implement a CI check that compares the Parquet schema (via `spark.read.parquet`) against the Hive table schema; fail the build if mismatched. |
| **Stale data** – the external table points to an outdated snapshot. | Compliance reports become inaccurate. | Schedule a daily refresh of the underlying Parquet files and run `MSCK REPAIR TABLE` (or `ALTER TABLE ... RECOVER PARTITIONS` if partitioned) to ensure the metastore sees new files. |
| **Large unpartitioned table** – full scans become costly as data grows. | Performance degradation, high query latency. | Add partitioning (e.g., by `feedreceiveddate` or `countrycode`) and rewrite the DDL accordingly. |
| **HDFS permission changes** – service account loses read access. | Table becomes inaccessible, causing downstream job failures. | Include a health‑check script that validates read permission on the HDFS path before running dependent jobs. |
| **Metastore / Impala catalog out‑of‑sync** – stale statistics or version mismatch. | Sub‑optimal query plans. | Periodically run `COMPUTE STATS kyc_mapping_snapshot;` and monitor `impala.lastComputeStatsTime`. |

---

### 6. Running / Debugging the Script

1. **Execute the DDL**  
   ```bash
   hive -f mediation-ddls/mnaa_Kyc_mapping_snapshot.hql
   # or using Beeline
   beeline -u jdbc:hive2://<hive-host>:10000/default -f mediation-ddls/mnaa_Kyc_mapping_snapshot.hql
   ```

2. **Validate Creation**  
   ```sql
   SHOW TABLES LIKE 'kyc_mapping_snapshot';
   DESCRIBE FORMATTED kyc_mapping_snapshot;
   ```

3. **Basic Data Check**  
   ```sql
   SELECT COUNT(*) FROM kyc_mapping_snapshot;
   SELECT vin, kycstatus, kyc_completion_date FROM kyc_mapping_snapshot LIMIT 10;
   ```

4. **Inspect Underlying Files** (HDFS)  
   ```bash
   hdfs dfs -ls -R /user/hive/warehouse/mnaas.db/kyc_mapping_snapshot
   ```

5. **Debugging Tips**  
   - If the table appears empty, verify that the upstream ETL has written Parquet files to the exact path.  
   - Use `spark-shell` or `pyspark` to read the Parquet directory directly and compare schemas.  
   - Check Hive metastore logs for errors related to `STATS_GENERATED` or `impala.events.*`.  

---

### 7. External Configuration & Environment Dependencies

| Item | How It Is Used |
|------|----------------|
| `NN-HA1` (HDFS namenode) | Hard‑coded in the `LOCATION` clause; typically injected via deployment scripts that replace the host if the cluster name changes. |
| Hive Metastore connection (usually via `hive.metastore.uris`) | Required for the DDL to register the table. |
| Impala catalog service ID (`impala.events.catalogServiceId`) | Populated automatically; used by Impala to track versioning. |
| Spark version (`spark.sql.create.version`) | Indicates the Spark version that originally generated the schema metadata; useful for compatibility checks. |

If the environment uses parameterised deployment (e.g., Ansible, Jenkins), the HDFS URI may be templated; verify the CI/CD pipeline for such substitutions.

---

### 8. Suggested Improvements (TODO)

1. **Add Partitioning** – Introduce a partition on `feedreceiveddate` (or another high‑cardinality date field) to improve query performance and enable incremental data loads.  
2. **Document Column Semantics** – Add column comments (via `COMMENT` clause) and a table comment to clarify business meaning (e.g., what `iskyccountry` represents, the source of `headunitswapped`). This aids downstream developers and data catalog tools.  

---