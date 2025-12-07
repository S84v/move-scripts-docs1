**File:** `mediation-ddls\mnaas_kyc_iccid_wise_country_mapping.hql`  

---

## 1. High‑Level Summary
This script creates an **external Hive/Impala table** named `mnaas.kyc_iccid_wise_country_mapping`. The table maps each SIM ICCID (and related device identifiers) to KYC‑related attributes such as country of sale, KYC status, dealer information, and timestamps. Data is stored as **Parquet** files under a fixed HDFS location, making the table instantly queryable by downstream billing, fraud‑prevention, and reporting jobs without moving the data again.

---

## 2. Core Objects Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `kyc_iccid_wise_country_mapping` | External Hive table | Provides a read‑only logical view over Parquet files that contain KYC‑ICCID mapping records. |
| Columns (`vin`, `countrycode`, `kycstatus`, … `lastupdatedate`) | Table fields | Capture device, SIM, dealer, and KYC lifecycle attributes needed by downstream processes. |
| `ROW FORMAT SERDE … ParquetHiveSerDe` | Storage descriptor | Instructs Hive/Impala to interpret the underlying files as Parquet. |
| `LOCATION 'hdfs://NN-HA1/.../kyc_iccid_wise_country_mapping'` | Table location | Physical directory where upstream ingestion jobs drop Parquet files. |
| `TBLPROPERTIES` (e.g., `external.table.purge='true'`) | Table metadata | Controls catalog behavior (purge on DROP), Impala event tracking, and audit timestamps. |

*No procedural code (functions, procedures) is present; the file is a pure DDL definition.*

---

## 3. Interfaces – Inputs, Outputs, Side‑Effects, Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | - Parquet files placed by upstream KYC ingestion pipelines (e.g., batch jobs that read from dealer portals, CRM, or external KYC services).<br>- HDFS namespace `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/kyc_iccid_wise_country_mapping`. |
| **Outputs** | - Hive/Impala metadata entry that makes the data queryable (`SELECT … FROM mnaas.kyc_iccid_wise_country_mapping`).<br>- Impala catalog events (via `impala.events.*` properties). |
| **Side‑Effects** | - Registers the external table in the Hive metastore.<br>- Because `external.table.purge='true'`, a `DROP TABLE` will **delete** the underlying HDFS files (potential data loss if mis‑used). |
| **Assumptions** | - The HDFS path exists and is writable by the Hive service account (`root` in the DDL, but actual runtime user may differ).<br>- Parquet schema matches the column list exactly (order and types).<br>- Impala catalog is reachable (catalog service ID provided).<br>- No partitioning is required for current query patterns. |

---

## 4. Integration Points with Other Scripts / Components

| Connected Component | Interaction |
|---------------------|-------------|
| **Upstream KYC ingestion jobs** (e.g., `mnaas_kyc_ingest_*.hql` or Spark/MapReduce pipelines) | Write Parquet files to the table’s HDFS location. |
| **Billing & Revenue Assurance jobs** (e.g., `mnaas_billing_*`, `mnaas_fiveg_*`) | Join on `iccid` / `vin` to enrich usage records with KYC status before invoicing. |
| **Reporting / Dashboard services** (e.g., Tableau, Superset) | Query the table via Impala for KYC compliance dashboards. |
| **Data Quality / Validation scripts** (e.g., `mnaas_kyc_quality_check.hql`) | Run `SELECT COUNT(*)`, null checks, or compare against reference tables. |
| **Orchestration layer** (Oozie / Airflow) | Executes this DDL as a “create‑table” task before any downstream jobs that depend on the mapping. |
| **Security / Auditing** (Ranger / Sentry) | Policies reference the table name to control read/write access for different roles. |

*Because the table is external, downstream jobs can read it without needing a separate `INSERT OVERWRITE` step, reducing data movement.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream process adds/removes columns not reflected in the DDL. | Query failures, silent data loss. | Version‑control DDL; run a nightly schema‑validation job that compares Parquet schema to Hive definition. |
| **Accidental DROP** – `external.table.purge=true` will delete source files. | Irrecoverable loss of KYC data. | Restrict DROP privileges; add a pre‑drop approval step in the orchestration; consider setting `external.table.purge=false` if data retention is required. |
| **Permission mismatch** – Hive/Impala service account cannot read/write the HDFS directory. | Table creation fails or queries return empty results. | Verify HDFS ACLs; include a health‑check task that `hdfs dfs -test -e <path>` before DDL execution. |
| **Impala catalog out‑of‑sync** – stale metadata after file changes. | Stale query results or errors. | Run `INVALIDATE METADATA mnaas.kyc_iccid_wise_country_mapping` after each load, or enable `REFRESH` in downstream jobs. |
| **Performance degradation** – full table scans on large unpartitioned data. | High query latency, resource contention. | See improvement suggestions below (add partitioning). |

---

## 6. Execution / Debug Guide

1. **Run the DDL**  
   ```bash
   hive -f mediation-ddls/mnaas_kyc_iccid_wise_country_mapping.hql
   # or via Impala:
   impala-shell -i <impala-host> -f mediation-ddls/mnaas_kyc_iccid_wise_country_mapping.hql
   ```

2. **Verify creation**  
   ```sql
   SHOW CREATE TABLE mnaas.kyc_iccid_wise_country_mapping;
   DESCRIBE FORMATTED mnaas.kyc_iccid_wise_country_mapping;
   ```

3. **Check data visibility** (after upstream load)  
   ```sql
   SELECT COUNT(*) FROM mnaas.kyc_iccid_wise_country_mapping;
   SELECT iccid, kycstatus FROM mnaas.kyc_iccid_wise_country_mapping LIMIT 10;
   ```

4. **Debugging tips**  
   - If the table appears empty, confirm that Parquet files exist under the HDFS location (`hdfs dfs -ls <path>`).  
   - Use `impala-shell -i <host> -q "INVALIDATE METADATA mnaas.kyc_iccid_wise_country_mapping;"` to force catalog refresh.  
   - Review Hive metastore logs for schema‑mismatch warnings.  

5. **Typical operator workflow**  
   - **Step 1:** Ensure upstream KYC load job completed successfully (check its log or a marker file).  
   - **Step 2:** Run `INVALIDATE METADATA` (or `REFRESH`) in Impala.  
   - **Step 3:** Execute downstream billing or reporting queries that join on this table.  

---

## 7. Configuration / External Dependencies

| Dependency | How it is used |
|------------|----------------|
| **HDFS Namenode address** (`NN-HA1`) | Hard‑coded in the `LOCATION` clause; must resolve from the Hive/Impala nodes. |
| **Hive Metastore** | Stores the table definition; must be reachable and running. |
| **Impala Catalog Service ID** (`b9c115786e934562:94a82089945b48d6`) | Populated automatically; used for Impala event tracking. |
| **Environment variables** (e.g., `HIVE_CONF_DIR`, `IMPALA_HOME`) | Not referenced directly in the script but required for the CLI tools that execute it. |
| **Security policies** (Ranger/Sentry) | External to the script; enforce read/write access based on the table name. |

If any of these change (e.g., a new HDFS HA cluster), the DDL must be updated accordingly.

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning** – Partition by `feedreceiveddate` (or `countrycode`) to limit scan size for daily loads and improve query performance.  
   ```sql
   PARTITIONED BY (feedreceiveddate DATE)
   ```
2. **Introduce Column Comments & Data Dictionary** – Document each field (e.g., `iccid` – “SIM identifier from dealer”, `kycstatus` – “KYC verification state”) to aid downstream developers and data catalog tools.  

--- 

*End of documentation.*