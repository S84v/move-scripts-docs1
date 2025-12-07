**File:** `mediation-ddls\mnaas_interconnect_traffic_details_reject.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the **`interconnect_traffic_details_reject`** table in the `mnaas` database. The table stores rejected inter‑connect traffic records (e.g., CDRs that failed validation or could not be billed). It is defined as an *insert‑only transactional* Hive table, stored as plain‑text files on HDFS under a managed warehouse location. Down‑stream mediation jobs read from this table to generate error reports, trigger re‑processing, or feed audit pipelines.

---

## 2. Core Object(s) Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `interconnect_traffic_details_reject` | Hive Managed Table | Holds rejected inter‑connect traffic rows with full CDR metadata and a `reject_reason` field. |
| `createtab_stmt` (script variable) | DDL string | Holds the `CREATE TABLE …` statement; used by the orchestration framework to execute the DDL. |

*No procedural code (functions, classes) is present; the file is pure DDL.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • Implicit: Hive/Impala execution environment.<br>• Implicit: HDFS cluster reachable via `hdfs://NN-HA1/...` (NameNode HA address). |
| **Outputs** | • A new Hive table definition stored in the Hive Metastore.<br>• Physical directory on HDFS created at `.../interconnect_traffic_details_reject` (empty until data is inserted). |
| **Side‑Effects** | • Updates Hive Metastore metadata (catalog service ID, version).<br>• May trigger Impala catalog refresh. |
| **Assumptions** | • Hive version supports *transactional* `insert_only` tables (Hive 3.x+).<br>• The HDFS path is writable by the Hive service user.<br>• Field delimiter `;` matches the format of upstream reject‑record generators.<br>• Down‑stream jobs expect the exact column list and data types (e.g., `duration` as `decimal(10,0)`). |

---

## 4. Integration with Other Scripts & Components

| Connected Component | Relationship |
|---------------------|--------------|
| `mnaas_interconnect_traffic_details_raw.hql` | Raw inter‑connect CDRs are first loaded into a *raw* table; a validation job filters out bad rows and inserts them into `*_reject`. |
| `mnaas_interconnect_traffic_details_usage.hql` (or similar usage tables) | After rejection handling, the *accepted* rows are inserted into usage‑billing tables. |
| ETL orchestration (e.g., Oozie, Airflow, or custom scheduler) | Executes this DDL as part of the **“schema‑setup”** phase before any data load jobs run. |
| Reporting / Auditing pipelines | Query `interconnect_traffic_details_reject` to produce error dashboards, SLA breach alerts, or to trigger re‑ingestion. |
| Impala / Hive clients | May run `SELECT * FROM mnaas.interconnect_traffic_details_reject` for ad‑hoc debugging. |
| External configuration files | Typically a `hive-site.xml` or environment‑specific `ddl.properties` that defines the Hive metastore URI, HDFS namenode address (`NN-HA1`), and any Kerberos principals. The script itself does not reference variables, but the execution wrapper may inject them. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – downstream jobs expect a column that is renamed or removed. | Data pipelines fail, causing billing gaps. | Enforce schema versioning; keep a change‑log and run automated compatibility tests after any DDL change. |
| **Incorrect delimiter** – upstream reject files use a different delimiter (e.g., `|`). | Rows are mis‑parsed, leading to malformed data. | Validate a sample file before loading; store delimiter as a configurable property and document it. |
| **Transactional table growth** – `insert_only` tables keep all versions, potentially filling HDFS. | Storage exhaustion, job failures. | Implement periodic compaction or TTL purge (e.g., via `ALTER TABLE … SET TBLPROPERTIES ('transactional_properties'='insert_only','transactional'='true')` and a cleanup job). |
| **Permission issues on HDFS path** – Hive user cannot write to the location. | Table creation fails, pipeline stalls. | Verify HDFS ACLs for the Hive service principal; include a pre‑flight check in the orchestration script. |
| **Impala catalog lag** – Impala queries see stale metadata after table creation. | Queries return “Table not found” errors. | Run `INVALIDATE METADATA mnaas.interconnect_traffic_details_reject;` or configure Impala to auto‑refresh. |

---

## 6. Running / Debugging the Script

1. **Standard execution** (via Hive CLI or Beeline):  
   ```bash
   hive -f mediation-ddls/mnaas_interconnect_traffic_details_reject.hql
   # or
   beeline -u jdbc:hive2://<hs2-host>:10000 -f mediation-ddls/mnaas_interconnect_traffic_details_reject.hql
   ```

2. **Orchestrated execution** (e.g., Oozie action):  
   - The action references the file path in its `<script>` element.  
   - Ensure the Oozie workflow sets `HADOOP_USER_NAME` to the Hive service user.

3. **Verification steps**:  
   ```sql
   SHOW CREATE TABLE mnaas.interconnect_traffic_details_reject;
   DESCRIBE FORMATTED mnaas.interconnect_traffic_details_reject;
   SELECT COUNT(*) FROM mnaas.interconnect_traffic_details_reject LIMIT 1;  -- should be 0 initially
   ```

4. **Debugging failures**:  
   - Check HiveServer2 logs (`/var/log/hive/hiveserver2.log`).  
   - Verify HDFS path existence and permissions: `hdfs dfs -ls /warehouse/tablespace/managed/hive/mnaas.db/interconnect_traffic_details_reject`.  
   - If the table already exists, either drop it (`DROP TABLE IF EXISTS …;`) or use `ALTER TABLE … REPLACE COLUMNS …` depending on the change policy.

---

## 7. External Configurations / Environment Variables

| Config / Env | Usage |
|--------------|-------|
| `NN-HA1` (NameNode HA address) | Embedded in the HDFS `LOCATION` URI; must resolve to the active NameNode. |
| Hive Metastore URI (`hive.metastore.uris`) | Determines where the table metadata is stored. |
| Kerberos principal/keytab (if security enabled) | Required for Hive/Impala to access HDFS. |
| Optional property file (e.g., `ddl.properties`) | May contain a variable for the table location base path, allowing the same script to be reused across environments (dev, test, prod). |

The script itself does not reference placeholders; any environment‑specific values are expected to be injected by the calling orchestration layer.

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning** – Partition the table by `record_insertime` (date) or `reject_reason` to improve query performance and enable easier data retention policies.  
   ```sql
   PARTITIONED BY (record_date DATE)
   ```

2. **Document Column Semantics** – Include `COMMENT` clauses for each column (especially `reject_reason` and `timebandindex`) to aid downstream developers and data catalog tools.

---