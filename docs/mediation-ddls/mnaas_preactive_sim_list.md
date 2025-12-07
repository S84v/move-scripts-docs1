**File:** `mediation-ddls\mnaas_preactive_sim_list.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the **external** table `mnaas.preactive_sim_list`. The table stores a flat‑file view of SIM cards that have been provisioned but not yet activated (pre‑active). It is partitioned by `month` and points to a fixed HDFS location under the Hive warehouse. The table is used by downstream mediation jobs (e.g., daily/monthly reconciliation, billing, and inventory‑status aggregation scripts) to join pre‑activation data with usage, billing, and port‑in/out processes.

---

## 2. Core Object(s) Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `preactive_sim_list` | External Hive table | Holds raw pre‑activation records (SIM, MSISDN, organization, offer, dates, activation flag). Provides a read‑only source for downstream ETL jobs. |
| `month` | Partition column (string) | Enables time‑based pruning for daily/monthly jobs; each partition corresponds to a calendar month (e.g., `2024-11`). |

*No procedural code (functions, procedures) is present in this file.*

---

## 3. Inputs, Outputs & Side Effects  

| Aspect | Details |
|--------|---------|
| **Input data** | Text files (delimited by default Hive SerDe) placed in HDFS path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/preactive_sim_list`. Files are expected to contain the columns defined in the table schema, in order. |
| **Output** | Logical table metadata registered in Hive Metastore; physical data remains in the external HDFS location. Downstream jobs read from this table. |
| **Side effects** | - Creates/overwrites the table definition in the Metastore.<br>- Sets table properties (`DO_NOT_UPDATE_STATS`, `external.table.purge`, etc.) that affect statistics collection and automatic cleanup. |
| **Assumptions** | - The HDFS directory exists and is accessible to Hive/Impala users.<br>- Files follow the LazySimpleSerDe default delimiter (Ctrl‑A `\001`).<br>- Partition `month` will be added by downstream ingestion jobs (e.g., `ALTER TABLE … ADD PARTITION`).<br>- No DML (INSERT/LOAD) is performed here; data loading is handled elsewhere. |

---

## 4. Integration Points  

| Connected Component | How it uses `preactive_sim_list` |
|---------------------|-----------------------------------|
| **MNP reconciliation scripts** (`mnaas_mnpportinout_*`) | Join on `sim`/`msisdn` to verify whether a SIM was pre‑active before a port‑in/out event. |
| **Billing aggregation jobs** (`mnaas_month_*`, `mnaas_month_ppu_billing.hql`) | Exclude pre‑active SIMs from usage‑based billing until `activated='Y'`. |
| **SIM inventory status aggregation** (`mnaas_move_sim_inventory_status_Aggr.hql`) | Update inventory status based on activation flag and dates. |
| **Daily usage aggregation views** (`mnaas_msisdn_level_daily_usage_aggr_*`) | Filter out non‑activated SIMs when calculating daily usage metrics. |
| **Org‑level reporting** (`mnaas_org_details.hql`) | Correlate `orgname` with pre‑active inventory for capacity planning. |

*Note:* The exact join keys and filters are defined in the downstream scripts; this DDL merely provides the source table.

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale or missing partition data** | Queries may return incomplete results if the expected `month` partition is absent. | Enforce a nightly validation job that checks for the presence of the current month partition and alerts if missing. |
| **Incorrect file format / delimiter** | Data ingestion failures or mis‑aligned columns. | Document the required delimiter (`\001`) and run a schema‑validation Spark/MapReduce job after each file drop. |
| **External table purge mis‑use** (`external.table.purge='true'`) | Accidental `DROP TABLE` will delete underlying HDFS files. | Restrict DROP privileges to a limited admin role and enable audit logging. |
| **Statistics not collected** (`DO_NOT_UPDATE_STATS='true'`) | Query planners may generate sub‑optimal plans. | Periodically run `ANALYZE TABLE … COMPUTE STATISTICS` manually if needed for performance‑critical queries. |
| **Permission drift on HDFS path** | Hive/Impala may lose read access, causing job failures. | Use a centralized ACL management script to enforce `r-x` for the Hive service principal. |

---

## 6. Running / Debugging the Script  

1. **Preparation**  
   - Verify that the Hive Metastore service is up and the user has `CREATE` privileges on schema `mnaas`.  
   - Ensure the HDFS directory exists:  
     ```bash
     hdfs dfs -test -d /user/hive/warehouse/mnaas.db/preactive_sim_list
     ```
2. **Execution**  
   - From the Hive CLI or Beeline:  
     ```sql
     hive -f mediation-ddls/mnaas_preactive_sim_list.hql
     ```
   - Or via Impala (DDL is compatible):  
     ```bash
     impala-shell -f mediation-ddls/mnaas_preactive_sim_list.hql
     ```
3. **Verification**  
   - List the table metadata:  
     ```sql
     SHOW CREATE TABLE mnaas.preactive_sim_list;
     ```
   - Check partitions (should be empty initially):  
     ```sql
     SHOW PARTITIONS mnaas.preactive_sim_list;
     ```
   - Sample data (if any partitions exist):  
     ```sql
     SELECT * FROM mnaas.preactive_sim_list LIMIT 10;
     ```
4. **Debugging Tips**  
   - If the command fails with “Table already exists”, run `DROP TABLE IF EXISTS mnaas.preactive_sim_list;` before re‑creating.  
   - For permission errors, inspect HDFS ACLs and Hive Metastore logs.  
   - Use `hdfs dfs -cat` on a sample file to confirm delimiter and column order.

---

## 7. External Configuration / Environment Variables  

| Item | Usage |
|------|-------|
| `NN-HA1` (NameNode HA address) | Hard‑coded in the `LOCATION` clause; must resolve to the active HDFS cluster. |
| Hive/Impala service principals | Required to read/write the external directory; not referenced directly in the script but assumed by the environment. |
| Metastore connection properties (e.g., `hive.metastore.uris`) | Needed for the DDL to register the table; managed outside this file. |
| Optional: `HIVE_CONF_DIR` or `IMPALA_CONF_DIR` | Determines where the client picks up cluster‑specific configs (e.g., Kerberos). |

If the deployment moves to a different HDFS namespace, the `LOCATION` string must be updated accordingly.

---

## 8. Suggested Improvements (TODO)

1. **Add a partition creation helper** – Include an `ALTER TABLE … ADD PARTITION` statement for the current month to guarantee that downstream jobs have a target partition immediately after table creation.  
2. **Parameterize the HDFS location** – Replace the hard‑coded `NN-HA1` with a Hive variable (e.g., `${hdfs_namenode}`) sourced from a central configuration file, making the script portable across environments (dev/test/prod).  

---