**File:** `mediation-ddls\mnaas_nsa_fiveg_product_subscription.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates (or replaces) the managed table `mnaas.nsa_fiveg_product_subscription`. The table stores raw subscription records for 5G products received from the NSA (Network Service Aggregator) feed. Each row captures the logical link between a SIM (ICCID) and a product, together with activation/de‑activation timestamps, the source file name, and a processing‑time marker. The table is defined as an *insert‑only transactional* Hive table stored as plain‑text files on HDFS, using a lazy‑simple SerDe with semicolon delimiters. Down‑stream “move” scripts (billing, usage aggregation, reporting) read from this table to enrich charging and analytics pipelines.

---

## 2. Core Object(s) Defined

| Object | Type | Responsibility / Description |
|--------|------|------------------------------|
| `mnaas.nsa_fiveg_product_subscription` | Hive Managed Table (Transactional, Insert‑Only) | Holds raw subscription events for 5G products. Columns: <br>• `tcl_secs_id` – integer key from source system <br>• `iccid` – SIM identifier (string) <br>• `product_id` – product catalogue key (string) <br>• `subscription_type` – e.g., “NEW”, “UPGRADE” (string) <br>• `activation_date` – string (expected format `yyyyMMdd` or ISO) <br>• `deactivation_date` – string (nullable) <br>• `fileupdateddatetime` – ingestion timestamp (string) <br>• `filename` – source file name (string) |
| SerDe | `org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe` | Parses semicolon‑delimited text files; line delimiter is newline. |
| Input/Output Formats | `TextInputFormat` / `HiveIgnoreKeyTextOutputFormat` | Reads raw text files; writes text output (no key). |
| Table Location | HDFS path `hdfs://NN-HA1/warehouse/tablespace/managed/hive/mnaas.db/nsa_fiveg_product_subscription` | Physical storage location for the managed table files. |
| Table Properties | Transactional (`insert_only`), Impala catalog metadata, stats generation flag, bucketing version, etc. | Enables ACID‑style inserts, Impala visibility, and basic statistics collection. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | No runtime input; the script itself is the input. The table will later be populated by upstream ingestion jobs (e.g., Sqoop, Spark, or custom “move” scripts) that write semicolon‑delimited text files to the table’s HDFS location. |
| **Outputs** | Creation of a Hive managed table. The DDL produces a directory on HDFS (if not already present) and registers the metadata in the Hive Metastore. |
| **Side‑Effects** | • If the table already exists, Hive will raise an error unless `CREATE TABLE IF NOT EXISTS` is added manually. <br>• The table is *transactional*; subsequent INSERTs will generate hidden delta files. <br>• The location is a managed path; dropping the table will delete the data. |
| **Assumptions** | • Hive/Impala services are running and have write permission to the HDFS path. <br>• The HDFS namenode alias `NN-HA1` resolves correctly in the cluster’s network. <br>• Down‑stream scripts expect the column types as defined (e.g., `string` for dates). <br>• The source feed uses a semicolon (`;`) as field delimiter and newline as record delimiter. <br>• No partitioning is required for the current data volume. |

---

## 4. Integration Points with Other Scripts / Components

| Connected Component | How It Connects |
|---------------------|-----------------|
| **Ingestion “move” scripts** (e.g., `mnaas_nsa_fiveg_product_subscription_load.hql` or Spark jobs) | They write raw records into this table using `INSERT INTO mnaas.nsa_fiveg_product_subscription SELECT …` or by placing files directly into the table’s HDFS location (Hive will pick them up on the next read). |
| **Billing pipelines** (`mnaas_month_billing.hql`, `mnaas_month_ppu_billing.hql`) | Join on `iccid` and `product_id` to apply 5G product pricing and generate monthly invoices. |
| **Usage aggregation views** (`mnaas_msisdn_level_daily_usage_aggr_view.hql`) | May reference this table to enrich usage rows with subscription type or activation windows. |
| **Reporting / Reconciliation scripts** (`mnaas_mnp_portinout_*` series) | Although focused on MNP, they share the same `iccid` key; cross‑checks can be performed to ensure a SIM is not simultaneously in two states. |
| **Impala** | The table properties expose the metadata to Impala; analysts can query the table directly via `impala-shell` for ad‑hoc analysis. |
| **Data Governance / Lineage tools** (e.g., Apache Atlas) | The table creation registers a new entity; downstream lineage will show this table as a source for billing and usage jobs. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – downstream jobs expect date columns as `DATE` but they are defined as `STRING`. | Query failures, incorrect date comparisons. | Add a conversion layer in downstream scripts (`to_date(activation_date, 'yyyyMMdd')`) or change column types to `DATE` after validating source format. |
| **Uncontrolled data growth** – No partitioning; large daily loads may cause full‑table scans. | Degraded query performance, high storage cost. | Introduce partitioning (e.g., by `activation_date` month) and adjust ingestion jobs accordingly. |
| **Incorrect delimiter handling** – Source feed changes delimiter or adds escaped characters. | Corrupted rows, mis‑aligned columns. | Validate incoming files with a pre‑load checksum or schema‑validation step; consider using `OpenCSVSerde` for more robust parsing. |
| **Transactional insert‑only mode** – Accidental `INSERT OVERWRITE` could delete data. | Data loss. | Enforce role‑based access control (RBAC) on Hive; audit DML statements; add a safeguard comment in the DDL. |
| **HDFS path permission changes** – Admin revokes write rights to the managed directory. | Ingestion jobs fail, pipeline stalls. | Periodically run a health‑check script that verifies directory existence and write permission; alert on failure. |
| **Metastore inconsistency** – Table metadata out‑of‑sync after a failed DDL execution. | Queries return stale schema. | Run `MSCK REPAIR TABLE` or `ALTER TABLE … RECOVER PARTITIONS` after any manual file movement; monitor Hive Metastore logs. |

---

## 6. Example: Running / Debugging the Script

1. **Execute the DDL**  
   ```bash
   hive -f mediation-ddls/mnaas_nsa_fiveg_product_subscription.hql
   # or, for Impala:
   impala-shell -i <impala-coordinator> -f mediation-ddls/mnaas_nsa_fiveg_product_subscription.hql
   ```

2. **Verify creation**  
   ```sql
   SHOW CREATE TABLE mnaas.nsa_fiveg_product_subscription;
   DESCRIBE FORMATTED mnaas.nsa_fiveg_product_subscription;
   ```

3. **Check HDFS location**  
   ```bash
   hdfs dfs -ls /warehouse/tablespace/managed/hive/mnaas.db/nsa_fiveg_product_subscription
   ```

4. **Common debugging steps**  
   - **Metastore errors** – Look at HiveServer2 logs (`/var/log/hive/hiveserver2.log`).  
   - **Permission denied** – Verify the Hive user (often `hive`) has `rwx` on the target HDFS directory.  
   - **SerDe parsing issues** – Load a small sample file manually:  
     ```sql
     LOAD DATA LOCAL INPATH '/tmp/sample.txt' INTO TABLE mnaas.nsa_fiveg_product_subscription;
     SELECT * FROM mnaas.nsa_fiveg_product_subscription LIMIT 10;
     ```  
     Inspect column alignment; adjust `field.delim` if needed.  

5. **Rollback** (if needed)  
   ```sql
   DROP TABLE IF EXISTS mnaas.nsa_fiveg_product_subscription;
   ```

---

## 7. External Configurations / Environment Variables

| Item | Usage |
|------|-------|
| `NN-HA1` (HDFS namenode alias) | Resolved by the Hadoop client configuration (`core-site.xml`). Must point to the active namenode. |
| Hive Metastore connection (`hive.metastore.uris`) | Determines where the table metadata is stored. |
| Impala catalog service ID (`impala.events.catalogServiceId`) | Auto‑generated; used by Impala for cache invalidation. |
| Optional: `HIVE_CONF_DIR` / `HADOOP_CONF_DIR` | Provide the cluster’s configuration files when running the script from a non‑cluster node. |

No additional property files are referenced directly in the DDL.

---

## 8. Suggested TODO / Improvements

1. **Add Partitioning** – Modify the DDL to partition by `activation_date` (e.g., `PARTITIONED BY (activation_month STRING)`) and update ingestion jobs to populate the partition column. This will dramatically improve query performance for time‑range scans.

2. **Upgrade Date Columns** – Change `activation_date` and `deactivation_date` from `STRING` to `DATE` (or `TIMESTAMP`) after confirming the source format. Include a comment in the DDL explaining the expected format and any conversion logic required in upstream loads.  

   ```sql
   ALTER TABLE mnaas.nsa_fiveg_product_subscription
     CHANGE activation_date activation_date DATE,
     CHANGE deactivation_date deactivation_date DATE;
   ```

   *Note:* Perform a one‑time data migration to cast existing strings to dates before applying the change.

---