**File:** `mediation-ddls\mnaas_traffic_details_billing_late_cdr.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) an **external Hive/Impala table** named `traffic_details_billing_late_cdr` in the `mnaas` database. The table stores *late‑arriving* Call Detail Records (CDRs) that are required for billing reconciliation. Data is partitioned by `bill_month` and resides on HDFS under the path `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/traffic_details_billing_late_cdr`. The table is defined with a wide set of string‑typed columns (reflecting the raw CDR payload) plus a few numeric and timestamp fields, enabling downstream analytics, aggregation, and billing jobs to query late CDRs without re‑ingesting the source files.

---

## 2. Core Artefacts & Responsibilities  

| Artefact | Responsibility |
|----------|-----------------|
| **`createtab_stmt`** (DDL block) | Defines the external table schema, partitioning, storage format, location, and table properties. |
| **Columns (≈70)** | Capture every attribute of a billing‑related CDR (identifiers, network info, usage metrics, pricing tags, timestamps, etc.). |
| **`PARTITIONED BY (bill_month string)`** | Enables efficient pruning for month‑level queries and incremental loading. |
| **`ROW FORMAT SERDE … LazySimpleSerDe`** | Parses the raw delimited text files as they are read. |
| **`STORED AS INPUTFORMAT … TextInputFormat`** | Indicates that source files are plain text (likely CSV/TSV). |
| **`LOCATION …`** | Points to the HDFS directory where the late‑CDR files are landed by upstream ingestion jobs. |
| **`TBLPROPERTIES`** | Controls statistics collection, external table purge behavior, and Impala catalog integration. |

*No procedural code (e.g., loops, UDFs) is present; the file is purely declarative.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Input data** | Raw late‑CDR files dropped into the HDFS location `.../traffic_details_billing_late_cdr`. Files are expected to be delimited text matching the column order defined in the DDL. |
| **Output artefact** | An external Hive/Impala table metadata entry. No data is copied; the table simply references the files in place. |
| **Side‑effects** | - Registers the table in the Hive metastore / Impala catalog.<br>- May trigger automatic statistics collection (disabled via `DO_NOT_UPDATE_STATS='true'`).<br>- Enables `external.table.purge='true'` so that dropping the table will also delete the underlying files. |
| **Assumptions** | - The HDFS path exists and is writable by the Hive/Impala service user.<br>- The underlying files conform to the column order and data types (most are strings, so type coercion is minimal).<br>- Partition directories (`bill_month=YYYYMM`) are created by upstream loaders before queries run.<br>- The environment provides a Hive metastore and Impala catalog that share the same `mnaas` database. |
| **External services** | - HDFS (namenode `NN-HA1`).<br>- Hive Metastore (or Impala catalog).<br>- Possibly an orchestration tool (e.g., Oozie, Airflow) that drops/creates the table as part of a nightly pipeline. |

---

## 4. Integration Points with Other Scripts / Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnaas_traffic_details_billing.hql`** | Holds the *regular* (on‑time) billing CDRs. Downstream jobs often UNION this table with the *late* table to produce a complete billing view. |
| **`mnaas_traffic_details_billing_interconnect_late_cdr.hql`** | Similar late‑CDR table for inter‑connect traffic; may be joined on common keys (e.g., `cdrid`). |
| **ETL ingestion jobs** (not shown) | Likely a Spark/MapReduce job that moves late CDR files from an SFTP drop zone into the HDFS location and creates the `bill_month` partition directories. |
| **Aggregation scripts** (e.g., `mnaas_traffic_details_billing_aggr_daily.hql`) | Consume this table to compute daily/monthly billing aggregates. |
| **Reporting / Billing engine** | Queries this table via Impala or Hive to reconcile late usage against invoices. |
| **Orchestration framework** (Airflow/Oozie) | Executes this DDL as part of a “create‑external‑tables” task, often preceded by a `DROP TABLE IF EXISTS` step. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream source adds/removes columns. | Queries may fail or return incorrect results. | Implement a schema‑validation step in the ingestion pipeline; version the DDL and keep it in source control. |
| **Missing or mis‑named partitions** (`bill_month`). | Queries that filter by month will scan the entire table, causing performance degradation. | Enforce partition creation in the loader; run `MSCK REPAIR TABLE` or `ALTER TABLE … RECOVER PARTITIONS` after each load. |
| **Incorrect file format / delimiter**. | LazySimpleSerDe may mis‑parse rows, leading to data corruption. | Store a small “sample” file with known good format; use a data‑quality check (e.g., Spark validation job) before files are placed in the directory. |
| **Stale statistics** (disabled updates). | Query optimizer may choose sub‑optimal plans. | Periodically run `COMPUTE STATS` on the table or enable automatic stats collection if storage permits. |
| **Accidental table drop** (purge enabled). | Underlying raw files could be deleted. | Restrict DROP privileges; add a pre‑drop backup step (e.g., copy to a retention HDFS path). |
| **Permission changes on HDFS path**. | Hive/Impala may lose read access, causing job failures. | Use ACLs managed by the orchestration tool; audit permissions after any HDFS admin change. |

---

## 6. How to Run / Debug the Script  

1. **Execution** (one‑off or as part of a pipeline)  
   ```bash
   # Hive CLI
   hive -f mediation-ddls/mnaas_traffic_details_billing_late_cdr.hql

   # Or Impala
   impala-shell -f mediation-ddls/mnaas_traffic_details_billing_late_cdr.hql
   ```

2. **Typical orchestration pattern**  
   ```sql
   DROP TABLE IF EXISTS mnaas.traffic_details_billing_late_cdr;
   -- then run the CREATE EXTERNAL TABLE statement (the file above)
   ```

3. **Verification steps**  
   - `SHOW CREATE TABLE mnaas.traffic_details_billing_late_cdr;` – confirm DDL matches expectations.  
   - `DESCRIBE FORMATTED mnaas.traffic_details_billing_late_cdr;` – check location, partition column, and table properties.  
   - `SELECT COUNT(*) FROM mnaas.traffic_details_billing_late_cdr LIMIT 10;` – ensure data is readable.  
   - `MSCK REPAIR TABLE mnaas.traffic_details_billing_late_cdr;` – refresh partitions if new `bill_month` directories were added after table creation.

4. **Debugging common issues**  
   - *“Table not found”* → Verify the `mnaas` database exists and the metastore is reachable.  
   - *“File not found”* → Confirm the HDFS path exists (`hdfs dfs -ls …/traffic_details_billing_late_cdr`).  
   - *“SerDe parsing error”* → Inspect a sample raw file; check delimiter and escape characters.  
   - *Permission errors* → Review HDFS ACLs (`hdfs dfs -getfacl …`) and Hive service user rights.

---

## 7. External Configuration / Environment Variables  

| Config Item | Usage in this Script |
|-------------|----------------------|
| **HDFS namenode address** (`NN-HA1`) | Hard‑coded in the `LOCATION` clause. Changing the cluster requires updating the path (or using a variable substitution mechanism in the orchestration layer). |
| **Hive/Impala metastore connection** | Not referenced directly; assumed to be provided by the client environment (`hive-site.xml` or `impala-shell` config). |
| **Database name (`mnaas`)** | Fixed in the DDL; if the environment uses a different schema, the script must be edited or templated. |
| **Table properties** (e.g., `external.table.purge`) | Hard‑coded; can be overridden via session variables if needed (`SET hive.exec.dynamic.partition=true;`). |
| **Partition value (`bill_month`)** | Populated by upstream loaders; not a variable in this file. |

If the organization uses a templating engine (e.g., Jinja, Velocity) to inject environment‑specific values, those placeholders would appear in the script; none are present here.

---

## 8. Suggested Improvements (TODO)

1. **Add `IF NOT EXISTS` guard** – Prevents accidental failures when the table already exists:  
   ```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS mnaas.traffic_details_billing_late_cdr ( ... )
   ```

2. **Automate partition discovery** – Include a post‑creation step in the orchestration pipeline:  
   ```bash
   impala-shell -q "ALTER TABLE mnaas.traffic_details_billing_late_cdr RECOVER PARTITIONS;"
   ```
   or embed `MSCK REPAIR TABLE` in a downstream Hive script to keep the metastore in sync with newly landed `bill_month` directories.

*Optional*: Add column comments and a small `COMMENT` on the table to aid data‑dictionary generation.

---