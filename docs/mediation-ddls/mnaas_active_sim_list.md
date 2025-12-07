**File:** `mediation-ddls\mnaas_active_sim_list.hql`  

---

### 1. High‑Level Summary
This script creates (or re‑creates) the Hive external table **`mnaas.active_sim_list`**. The table stores a flat‑file view of active SIM records, partitioned by month, and points to an HDFS directory that is populated by upstream ingestion jobs. Because it is defined as *EXTERNAL* with `external.table.purge='true'`, Hive does not own the underlying files; dropping the table will delete the data files. The definition is used by downstream analytics, reporting, and data‑movement pipelines that query or export active SIM information.

---

### 2. Core DDL Elements & Responsibilities  

| Element | Responsibility |
|---------|-----------------|
| `CREATE EXTERNAL TABLE mnaas.active_sim_list` | Registers the table in the Hive Metastore without moving data. |
| Columns (`sim`, `tcl_secs_id`, `orgname`, `commercialoffer`, `eid`) | Define the schema of each SIM record. |
| `PARTITIONED BY (month STRING)` | Enables month‑level pruning and incremental loading. |
| `ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'` | Parses plain‑text, delimiter‑separated files (default tab/space). |
| `STORED AS INPUTFORMAT ...TextInputFormat` / `OUTPUTFORMAT ...HiveIgnoreKeyTextOutputFormat` | Indicates line‑oriented text files; no key/value output. |
| `LOCATION 'hdfs://NN-HA1/user/hive/warehouse/mnaas.db/active_sim_list'` | Physical path where upstream jobs drop files. |
| `TBLPROPERTIES` (e.g., `DO_NOT_UPDATE_STATS='true'`, `external.table.purge='true'`) | Controls statistics collection, automatic purge on DROP, and Impala catalog integration. |

*No procedural code (functions, procedures) is present; the file is pure DDL.*

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/active_sim_list` containing month‑partitioned text files.<br>- Implicit Hive/Impala configuration (e.g., `hive.metastore.uris`). |
| **Outputs** | - Metadata entry in the Hive Metastore for `mnaas.active_sim_list`.<br>- Impala catalog updates (via `impala.events.*` properties). |
| **Side‑Effects** | - If the table already exists, the statement will **fail** (no `IF NOT EXISTS`).<br>- Dropping the table later will delete the underlying files because of `external.table.purge='true'`. |
| **Assumptions** | - The HDFS namenode alias `NN-HA1` resolves in the execution environment.<br>- Data files conform to the LazySimpleSerDe expectations (default delimiter, no complex types).<br>- Upstream processes create month partitions under the location (e.g., `.../active_sim_list/month=202401/`).<br>- Hive/Impala services have permission to read/write the target HDFS path. |

---

### 4. Integration Points  

| Connected Component | Interaction |
|---------------------|-------------|
| **Upstream ingestion jobs** (e.g., nightly ETL that extracts active SIMs from CRM) | Write raw text files into the HDFS location, creating new `month=` partitions. |
| **Partition discovery scripts** (often `MSCK REPAIR TABLE` or custom `ALTER TABLE ADD PARTITION`) | After files land, they must be registered in the Metastore so queries see them. |
| **Downstream analytics / reporting jobs** (Impala queries, Spark jobs, BI tools) | Read from `mnaas.active_sim_list` to generate usage reports, churn analysis, etc. |
| **Data‑move orchestration framework** (e.g., Airflow, Oozie) | Executes this DDL as part of a “schema‑setup” task before the first load of a new environment. |
| **Impala catalog service** | Consumes the `impala.events.*` table properties for change notifications. |

*Because the script is part of the `mediation-ddls` folder, it is typically run once per environment (dev/test/prod) during schema provisioning.*

---

### 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Table recreation failure** – running the script when the table already exists aborts the job. | Add `IF NOT EXISTS` or wrap in a try‑catch block (`DROP TABLE IF EXISTS …; CREATE EXTERNAL TABLE …`). |
| **Stale partitions** – new month directories may not be visible to Hive/Impala. | Schedule `MSCK REPAIR TABLE mnaas.active_sim_list;` or use `ALTER TABLE … ADD PARTITION` after each load. |
| **Accidental data loss** – `external.table.purge='true'` causes data deletion on DROP. | Restrict DROP privileges, document the property, and enforce a “soft‑delete” policy (e.g., never drop, only truncate). |
| **Statistics not collected** – `DO_NOT_UPDATE_STATS='true'` can lead to poor query plans. | Periodically run `ANALYZE TABLE mnaas.active_sim_list COMPUTE STATISTICS;` if query performance degrades. |
| **Permission mismatches** – Hive/Impala may lack HDFS read/write rights. | Verify HDFS ACLs (`hdfs dfs -ls …`) and ensure service principals are granted `rwx` on the directory. |
| **Schema drift** – upstream changes to file layout (new columns) will break queries. | Version the DDL, add a change‑log, and coordinate schema updates with upstream owners. |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|-------------------|
| **Execute** | `hive -f mediation-ddls/mnaas_active_sim_list.hql`  (or `beeline -u jdbc:hive2://<host>:10000 -f …`) |
| **Impala verification** | `impala-shell -i <impala-host> -q "SHOW CREATE TABLE mnaas.active_sim_list;"` |
| **Check partitions** | `hive -e "SHOW PARTITIONS mnaas.active_sim_list;"` |
| **Validate data** | `hdfs dfs -ls hdfs://NN-HA1/user/hive/warehouse/mnaas.db/active_sim_list/month=202401/` |
| **Debug missing partitions** | Run `MSCK REPAIR TABLE mnaas.active_sim_list;` and re‑query. |
| **Log inspection** | Review HiveServer2 logs (`/var/log/hive/hiveserver2.log`) for DDL errors; Impala catalog logs for `impala.events.*` updates. |

---

### 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| `NN-HA1` (HDFS namenode alias) | Resolved by the Hadoop client configuration (`core-site.xml`). |
| Hive Metastore URI (`hive.metastore.uris`) | Determines where the table definition is stored. |
| Impala catalog service ID (`impala.events.catalogServiceId`) | Auto‑populated; required for Impala change notifications. |
| Hadoop user / Kerberos principal (if security enabled) | Must have read/write access to the target HDFS path. |
| Hive/Impala configuration files (`hive-site.xml`, `impala-default.xml`) | Control default SerDe, file format handling, and purge behavior. |

No additional property files are referenced directly in the script.

---

### 8. Suggested Improvements (TODO)

1. **Make the DDL idempotent** – prepend `DROP TABLE IF EXISTS mnaas.active_sim_list;` or add `IF NOT EXISTS` to the `CREATE EXTERNAL TABLE` statement to allow safe re‑execution during environment rebuilds.  
2. **Automate partition registration** – embed a post‑creation step that runs `MSCK REPAIR TABLE` or a custom `ALTER TABLE … ADD PARTITION` loop, ensuring newly landed month directories become queryable without manual intervention.  

---