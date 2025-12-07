**File:** `mediation-ddls\traffic_aggr_adhoc.hql`  
**Type:** HiveQL DDL – creates an **external** table `mnaas.traffic_aggr_adhoc`.

---

## 1. High‑level Summary
This script defines the Hive/Impala external table `traffic_aggr_adhoc` that stores ad‑hoc traffic‑aggregation records (voice, data, SMS, cost, SIM, etc.). The table is partitioned by a string `partition_date` and reads raw, semicolon‑delimited text files located under  
`hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_aggr_adhoc`.  
Down‑stream analytics, billing, or reporting jobs query this table; upstream ingestion jobs drop files into the HDFS location (or add partitions) without Hive managing the data files themselves.

---

## 2. Core Objects Defined

| Object | Responsibility |
|--------|-----------------|
| **Table `mnaas.traffic_aggr_adhoc`** | Holds a flat‑file view of ad‑hoc traffic aggregates; each row represents a usage record enriched with customer, product, and cost information. |
| **Columns** (selected) | • `customernumber`, `customername` – customer identifiers.<br>• `businessunit*`, `department*`, `costcenter*` – organisational hierarchy.<br>• `traffictype`, `calltype`, `mcc`, `mnc`, `country` – network metadata.<br>• `data_usage_in_mb/kb`, `voice_usage_in_secs/mins`, `sms_usage` – raw usage metrics.<br>• `whole_sale_cost_1…5` – cost breakdowns.<br>• `tadigcode`, `apn`, `technologytype` – technical attributes.<br>• `sim`, `sim_type`, `msisdn`, `vin` – device identifiers.<br>• `noofcdrids` – distinct CDR count per row.<br>• `partition_date` – partition key (string). |
| **Partition** | `partition_date` (string) – enables daily/periodic pruning and selective scans. |
| **SerDe** | `LazySimpleSerDe` with field delimiter `;` and line delimiter `\n`. |
| **Storage format** | TextInputFormat / HiveIgnoreKeyTextOutputFormat (plain text files). |
| **Location** | `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_aggr_adhoc`. |
| **Table properties** | ‑ `external.table.purge='true'` (auto‑delete files on DROP).<br>‑ Impala catalog metadata (`impala.events.*`).<br>‑ Statistics disabled (`DO_NOT_UPDATE_STATS='true'`). |

---

## 3. Inputs, Outputs, Side‑effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | - Raw text files placed in the HDFS location, matching the column order and semicolon delimiter.<br>- Each file must contain a `partition_date` value that matches the Hive partition directory (e.g., `.../partition_date=20231201/`). |
| **Outputs** | - Hive metastore entry for `traffic_aggr_adhoc` (metadata only).<br>- Logical table view for downstream queries (Impala, Hive, Spark). |
| **Side‑effects** | - Registers the external table in the metastore.<br>- No data movement; files remain where they are.<br>- `external.table.purge='true'` will delete underlying files if the table is dropped. |
| **Assumptions** | - HDFS namenode alias `NN-HA1` resolves in the cluster environment.<br>- The directory exists and the Hive/Impala service accounts have read/write permissions.<br>- Data files are correctly delimited and column‑aligned; no schema evolution is expected without DDL change.<br>- Partition directories are created manually or by an upstream ETL (e.g., `INSERT OVERWRITE DIRECTORY ... PARTITION (partition_date='20231201')`). |

---

## 4. Interaction with Other Scripts / Components

| Connected Component | Interaction Point |
|----------------------|-------------------|
| **Upstream ingestion jobs** (e.g., `move_traffic_*.hql`, Spark/MapReduce jobs) | Write/append semicolon‑delimited files into the table’s HDFS location and add partitions via `ALTER TABLE ... ADD PARTITION`. |
| **Downstream analytics** (billing, reporting, KPI dashboards) | Query `mnaas.traffic_aggr_adhoc` directly; often joined with `customer`, `product`, or `rate` reference tables defined in other DDL scripts (e.g., `supplier_rates.hql`). |
| **Impala** | Uses the same metastore; the `impala.events.*` properties indicate that Impala catalog sync is expected. |
| **Metadata management** (e.g., `move_master_consolidated_1.hql`) | May reference this table in view definitions or data‑move pipelines that consolidate daily aggregates. |
| **Orchestration layer** (Airflow, Oozie, custom scheduler) | Executes this DDL as part of a “create‑schema” or “refresh‑partition” DAG/task. |
| **Configuration** | No explicit external config in the script; however, environment variables for Hive/Impala connection strings, Kerberos principals, or HDFS namenode address are typically supplied by the orchestration environment. |

---

## 5. Operational Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Schema drift** – upstream jobs change column order or add fields → query failures. | Enforce schema validation in the upstream job; version the table (e.g., `traffic_aggr_adhoc_v2`). |
| **Missing / mis‑named partitions** – data lands in wrong directory → rows invisible to queries. | Automate partition creation (`ALTER TABLE ... ADD IF NOT EXISTS PARTITION`) and add a health‑check DAG that lists expected partitions vs. actual. |
| **Data format mismatch** – delimiter change or corrupted lines → parsing errors. | Include a lightweight pre‑load validation step (e.g., Spark job that samples files and checks column count). |
| **Storage exhaustion** – uncontrolled file growth in HDFS location. | Set HDFS quotas on the directory; schedule periodic archival or purge of old partitions. |
| **Permission issues** – Hive/Impala service accounts lose access to the HDFS path. | Periodic ACL audit; store path permissions in a central config and enforce via CI pipeline. |
| **Accidental DROP** – `external.table.purge=true` will delete files. | Require `DROP TABLE` to be gated behind a manual approval step or use a soft‑delete flag in the orchestration layer. |

---

## 6. Running / Debugging the Script

1. **Execute the DDL**  
   ```bash
   hive -f mediation-ddls/traffic_aggr_adhoc.hql
   # or, for Impala:
   impala-shell -i <impala-host> -f mediation-ddls/traffic_aggr_adhoc.hql
   ```

2. **Verify creation**  
   ```sql
   SHOW CREATE TABLE mnaas.traffic_aggr_adhoc;
   DESCRIBE FORMATTED mnaas.traffic_aggr_adhoc;
   ```

3. **Check HDFS location**  
   ```bash
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/traffic_aggr_adhoc
   ```

4. **Add a test partition (if not done by upstream)**  
   ```sql
   ALTER TABLE mnaas.traffic_aggr_adhoc ADD IF NOT EXISTS PARTITION (partition_date='20231201')
   LOCATION 'hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_aggr_adhoc/partition_date=20231201';
   ```

5. **Run a sample query**  
   ```sql
   SELECT COUNT(*) FROM mnaas.traffic_aggr_adhoc WHERE partition_date='20231201';
   ```

6. **Debugging tips**  
   - If the query returns zero rows, verify that files exist under the correct partition directory and that the delimiter is `;`.  
   - Use `hive --hiveconf hive.exec.dynamic.partition=true` to allow dynamic partition creation if needed.  
   - Check the Hive metastore logs for “Table not found” or “Partition not found” errors.  

---

## 7. External Config / Environment Dependencies

| Item | Usage |
|------|-------|
| `NN-HA1` (namenode alias) | Hard‑coded in the `LOCATION` clause; resolved via the cluster’s DNS or `/etc/hosts`. |
| Hive/Impala connection parameters | Supplied by the orchestration environment (e.g., `HIVE_CONF_DIR`, `IMPALA_HOST`). |
| Kerberos principal / keytab (if enabled) | Required for HDFS access; not referenced in the script but assumed to be active in the runtime context. |
| Optional: table‑location base path variable | Not present; could be externalized to a properties file to avoid hard‑coding the HDFS URI. |

---

## 8. Suggested Improvements (TODO)

1. **Parameterize the HDFS location** – replace the hard‑coded URI with a Hive variable (e.g., `${hdfs.base.path}`) read from a central config file, enabling the same DDL to be used across environments (dev, test, prod).  
2. **Add `IF NOT EXISTS` guard** – modify the `CREATE EXTERNAL TABLE` statement to `CREATE EXTERNAL TABLE IF NOT EXISTS` to make the script idempotent and safe for repeated execution in CI/CD pipelines.  

---