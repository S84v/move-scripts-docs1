**File:** `mediation-ddls\mnaas_actives_raw_daily.hql`  

---

## 1. High‑Level Summary
This script creates an **external Hive table** `mnaas.actives_raw_daily` that maps to a partitioned Parquet dataset stored in HDFS (`hdfs://NN-HA1/.../actives_raw_daily`). The table captures a daily snapshot of “active” product records (SIMs, MSISDNs, plans, carriers, etc.) for MVNO customers. Down‑stream mediation, reporting, and analytics jobs query this table to enrich or aggregate activation‑level data.

---

## 2. Core Objects Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.actives_raw_daily` | External Hive table | Provides a schema‑on‑read view over daily raw activation files; partitioned by `partition_date`. |
| Columns (≈ 70) | Table fields | Store detailed attributes of each product (company, invoice, business unit, SIM/ICCID, MSISDN, status, timestamps, plan, carrier, zone, etc.). |
| `partition_date` | Partition column (string) | Enables daily pruning and incremental processing. |
| Table properties | Hive/Impala metadata | Control stats generation, purge behavior, and compatibility with Spark/Impala engines. |
| `LOCATION` | HDFS path | Physical storage location for the Parquet files that feed the table. |

*No procedural code (functions, procedures) is present; the file is pure DDL.*

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Inputs** | - Parquet files placed under `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/actives_raw_daily/<partition_date>/` by upstream ingestion pipelines (e.g., nightly SFTP drop, Spark ETL, or batch loader). |
| **Outputs** | - Hive/Impala metadata entry for `actives_raw_daily`. <br> - Logical table that downstream jobs SELECT from (e.g., `mnaas_actives_raw_daily` transformation scripts, reporting jobs). |
| **Side Effects** | - Registers the external table in the Hive Metastore. <br> - Because `external.table.purge='true'`, dropping the table will also delete the underlying HDFS files. |
| **Assumptions** | - HDFS path exists and is writable by the Hive/Impala service accounts. <br> - Files are in Parquet format matching the declared schema. <br> - Partition directories are named exactly as the `partition_date` string value (e.g., `2024-11-30`). <br> - Hive Metastore is reachable and configured for Impala events (properties indicate Impala integration). |

---

## 4. Integration with Other Components

| Connected Script / Component | Relationship |
|------------------------------|--------------|
| **`mnaas_actives_raw_daily.hql` (or similarly named transformation scripts)** | Likely reads from this external table, performs cleansing, enrichment, and writes to internal/managed tables for analytics. |
| **Ingestion pipelines** (e.g., Spark jobs, SFTP loaders) | Populate the HDFS location with daily Parquet files before this DDL is executed (or after, if the table already exists). |
| **Reporting / BI layers** (e.g., Tableau, PowerBI, custom dashboards) | Query the table via Impala or Hive to surface activation metrics. |
| **Data quality / partition management jobs** | May run `MSCK REPAIR TABLE mnaas.actives_raw_daily` or `ALTER TABLE … ADD PARTITION` to make new daily partitions visible. |
| **Hive Metastore / Impala Catalog** | Consumes the table properties for statistics, lineage, and purge handling. |

*Because the file creates the table *once*, subsequent runs of downstream scripts assume the table already exists.*

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Schema drift** – upstream producers add/remove columns without updating DDL. | Queries may fail or return nulls. | Implement a schema‑validation step in the ingestion pipeline; version the table (e.g., `actives_raw_daily_v2`). |
| **Missing or mis‑named partition directories** – new daily data not discovered. | Data loss for that day in downstream reports. | Automate partition discovery (`MSCK REPAIR TABLE`) after each load; monitor partition count vs expected dates. |
| **External table purge** – accidental `DROP TABLE` removes raw files. | Irrecoverable loss of source data. | Restrict DROP privileges; add a safeguard script that checks a “production‑only” flag before allowing drop. |
| **Permission mismatch** – Hive/Impala service account cannot read/write HDFS path. | Table creation fails or downstream reads error out. | Verify HDFS ACLs and Kerberos tickets during deployment; include a health‑check job. |
| **Performance degradation** – large number of partitions without proper statistics. | Slow query planning/execution. | Periodically run `COMPUTE STATS` on the table or enable incremental stats collection. |

---

## 6. Running / Debugging the Script

1. **Execute DDL**  
   ```bash
   hive -f mediation-ddls/mnaas_actives_raw_daily.hql
   # or via Impala:
   impala-shell -f mediation-ddls/mnaas_actives_raw_daily.hql
   ```

2. **Verify Creation**  
   ```sql
   SHOW CREATE TABLE mnaas.actives_raw_daily;
   DESCRIBE FORMATTED mnaas.actives_raw_daily;
   ```

3. **Check Partition Visibility** (after data lands)  
   ```sql
   SHOW PARTITIONS mnaas.actives_raw_daily;
   -- or repair
   MSCK REPAIR TABLE mnaas.actives_raw_daily;
   ```

4. **Sample Query** (to confirm data shape)  
   ```sql
   SELECT partition_date, COUNT(*) AS cnt
   FROM mnaas.actives_raw_daily
   GROUP BY partition_date
   ORDER BY partition_date DESC
   LIMIT 5;
   ```

5. **Debugging Tips**  
   - Look at Hive Metastore logs for “Table already exists” or “Invalid schema”.  
   - Verify HDFS path with `hdfs dfs -ls <location>`; ensure Parquet files are present.  
   - If queries return empty, confirm that the `partition_date` value in the file matches the directory name.  

---

## 7. External Configuration / Environment Dependencies

| Item | Usage |
|------|-------|
| **Hive Metastore connection** | Defined outside this script (usually via `hive-site.xml`). |
| **Impala catalog service ID** (`impala.events.catalogServiceId`) | Auto‑generated; required for Impala to see the table. |
| **HDFS Namenode address** (`NN-HA1`) | Hard‑coded in the `LOCATION` URI; must resolve in the cluster DNS. |
| **Kerberos / Hadoop security** | If enabled, the user executing the script must have a valid ticket. |
| **Environment variables** (e.g., `HIVE_CONF_DIR`, `IMPALA_HOME`) | Not referenced directly but required for CLI tools. |

No additional config files are imported by this DDL; all table‑level settings are embedded in the `TBLPROPERTIES`.

---

## 8. Suggested Improvements (TODO)

1. **Add `IF NOT EXISTS` guard** to make the script idempotent:  
   ```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS mnaas.actives_raw_daily ( ... );
   ```

2. **Automate partition discovery** by appending a post‑creation step:  
   ```sql
   MSCK REPAIR TABLE mnaas.actives_raw_daily;
   ```  
   or schedule a daily Airflow/Control-M job that adds the new `partition_date` partition explicitly, reducing reliance on Hive’s repair mechanism.

---