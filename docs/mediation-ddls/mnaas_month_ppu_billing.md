**File:** `mediation-ddls\mnaas_month_ppu_billing.hql`  

---

## 1. High‑Level Summary
This script creates an **external Hive table** named `mnaas.month_ppu_billing`. The table stores the start and end timestamps of a data‑load window together with the corresponding partition date strings. The data lives in an HDFS directory (`/user/hive/warehouse/mnaas.db/month_ppu_billing`) and is managed by Hive/Impala as a read‑only source for downstream billing and reporting jobs.

---

## 2. Core Artefacts

| Artefact | Type | Responsibility |
|----------|------|-----------------|
| `createtab_stmt` | DDL statement | Issues a `CREATE EXTERNAL TABLE` with four columns (`start_inserttime`, `end_inserttime`, `start_part_date`, `end_part_date`). |
| Table `mnaas.month_ppu_billing` | Hive external table | Provides a logical view over raw files in the specified HDFS location; used by downstream ETL jobs that calculate per‑unit (PPU) billing. |
| Table properties (e.g., `external.table.purge`, `impala.events.*`) | Metadata | Controls purge behaviour on DROP, tracks Impala catalog version, and records audit information. |

*No procedural code, functions, or classes are defined in this file; it is a pure DDL definition.*

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | - Implicit: Hive/Impala runtime, HDFS namenode address (via environment or Hive config).<br>- External: The HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/month_ppu_billing` must exist (or be creatable) and contain files matching the default text format. |
| **Outputs** | - Hive metastore entry for `mnaas.month_ppu_billing`.<br>- No data is written by this script; it only registers the location. |
| **Side Effects** | - Registers a new external table; if the table already exists, Hive will error unless `IF NOT EXISTS` is added.<br>- Because `external.table.purge='true'`, dropping the table will also delete the underlying HDFS files. |
| **Assumptions** | - The HDFS namenode `NN-HA1` is reachable and the Hive service has permission to read/write the target directory.<br>- The Hive/Impala catalog service IDs (`impala.events.catalogServiceId`, `catalogVersion`) are current; they are static in the script but may need updating after a catalog refresh. |
| **External Services** | - HDFS (namenode `NN-HA1`).<br>- Hive Metastore (for DDL registration).<br>- Impala (for downstream queries). |

---

## 4. Integration Points

| Connected Component | How the Table Is Used |
|---------------------|-----------------------|
| **Upstream ingestion jobs** (e.g., nightly batch that writes raw PPU billing files) | Write text files into the HDFS location; the external table automatically reflects new files. |
| **Downstream reporting scripts** (e.g., `mnaas_month_billing.hql`, `mnaas_month_ppu_billing_report.hql` – not shown) | Query `mnaas.month_ppu_billing` to retrieve load windows and join with detailed billing data. |
| **Data quality / reconciliation jobs** (e.g., `mnaas_mnpportinout_daily_recon_inter.hql`) | May reference the table to verify that a load window matches expected record counts. |
| **Impala catalog** | The table properties contain Impala catalog IDs; Impala queries will use the same metadata. |
| **Orchestration layer** (Airflow, Oozie, or custom scheduler) | Executes this DDL as part of a “create‑schema” task before any data is landed. |

*Because all DDL files in `mediation-ddls` follow the same naming convention, the orchestration framework likely runs them in alphabetical or dependency order.*

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental DROP** – `external.table.purge='true'` causes data loss if the table is dropped. | Permanent loss of raw billing files. | Restrict DROP privileges; add a pre‑drop backup step; consider setting `external.table.purge='false'` if data must be retained. |
| **Path Permission Issues** – Hive user may lack write permission on the HDFS directory. | Table creation fails; downstream jobs cannot read data. | Verify HDFS ACLs for the Hive service principal; include a `hdfs dfs -chmod` step in the deployment script. |
| **Stale Impala Catalog IDs** – Hard‑coded `catalogServiceId`/`catalogVersion` may become out‑of‑sync after a catalog restart. | Impala queries may fail or return stale metadata. | Remove static catalog properties; let Impala auto‑populate them, or refresh the catalog after DDL execution (`INVALIDATE METADATA`). |
| **Schema Drift** – Upstream jobs may start emitting additional columns, causing query failures. | Downstream jobs break. | Add column comments and versioning; implement a schema‑evolution policy (e.g., add new columns as nullable). |
| **Incorrect Data Format** – The table expects plain text with default delimiters; upstream may write CSV/TSV with different delimiters. | Data parsing errors. | Explicitly define `FIELDS TERMINATED BY` and `SERDE` properties matching upstream format. |

---

## 6. Running / Debugging the Script

1. **Execute** (typical in production):  
   ```bash
   hive -f mediation-ddls/mnaas_month_ppu_billing.hql
   # or using Beeline:
   beeline -u jdbc:hive2://<hive-host>:10000/default -f mediation-ddls/mnaas_month_ppu_billing.hql
   ```

2. **Validate Creation**:  
   ```sql
   SHOW CREATE TABLE mnaas.month_ppu_billing;
   DESCRIBE FORMATTED mnaas.month_ppu_billing;
   ```

3. **Check HDFS Path**:  
   ```bash
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/month_ppu_billing
   ```

4. **Debugging Tips**  
   - Look at Hive server logs (`/var/log/hive/hive-server2.log`).  
   - If the table already exists, add `IF NOT EXISTS` to the DDL or drop it first (`DROP TABLE IF EXISTS mnaas.month_ppu_billing;`).  
   - To force Impala to see the new table, run `INVALIDATE METADATA mnaas.month_ppu_billing;` in Impala shell.  

---

## 7. External Configuration & Environment Variables

| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` / `HADOOP_CONF_DIR` | Determines Hive metastore and HDFS connection details. |
| `NN-HA1` (namenode hostname) | Hard‑coded in the `LOCATION` URI; must resolve in the network. |
| `HIVE_WAREHOUSE_DIR` (default `/user/hive/warehouse`) | Base path for the external table location; the script appends `mnaas.db/month_ppu_billing`. |
| Impala catalog IDs (`impala.events.catalogServiceId`, `impala.events.catalogVersion`) | Currently static; may be overridden by environment if the catalog is refreshed. |

*If the deployment environment uses a different namenode or warehouse directory, the script must be edited or templated accordingly.*

---

## 8. Suggested Improvements (TODO)

1. **Add `IF NOT EXISTS` Guard** – Prevents failures when the table already exists:  
   ```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS `mnaas`.`month_ppu_billing` ( ... );
   ```

2. **Explicit Data Format Definition** – Specify delimiter and optional compression to avoid mismatches:  
   ```sql
   ROW FORMAT DELIMITED
     FIELDS TERMINATED BY '\t'
   STORED AS TEXTFILE
   ```

   *(Adjust delimiter/compression to match upstream producer.)*

---