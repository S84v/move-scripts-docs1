**File:** `mediation-ddls\supplier_rates.hql`  

---

## 1. High‑Level Summary
This script creates an **external Hive table** `mnaas.supplier_rates` that maps telecom‑supplier rate keys to their pricing attributes (call type, traffic type, zones, tariffs, markup, etc.). The table points to a flat‑file location on HDFS (`hdfs://NN-HA1/.../supplier_rates`) and uses a semicolon (`;`) as field delimiter. It is intended for downstream mediation, billing, and analytics jobs that need to look up per‑rate pricing information.

---

## 2. Core Artifact

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `CREATE EXTERNAL TABLE mnaas.supplier_rates` | Hive DDL | Registers the schema and location of the supplier‑rate flat file so that Hive/Impala can query it directly. |
| Table columns (`ratekeyid`, `sponsor_tadig`, … `creationdate`) | Metadata | Provide the data model required by downstream rate‑lookup logic. |
| SerDe & file format settings | Configuration | Define how Hive parses the source text file (LazySimpleSerDe, `;` delimiter, newline line terminator). |
| Table properties (`external.table.purge`, `impala.*`, timestamps) | Metadata | Control purge behavior on DROP, track catalog version for Impala, and store audit info. |

*No procedural code (functions, classes) is present – the file is pure DDL.*

---

## 3. Interfaces

| Aspect | Description |
|--------|-------------|
| **Inputs** | - HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/supplier_rates` containing one or more text files with fields delimited by `;`. <br> - Implicit dependencies on the Hadoop NameNode (`NN-HA1`) and Hive Metastore. |
| **Outputs** | - Hive/Impala **external table** `mnaas.supplier_rates` exposing the data as a queryable relation. |
| **Side Effects** | - Registers metadata in the Hive Metastore. <br> - Because `external.table.purge='true'`, dropping the table will also delete the underlying HDFS files. |
| **Assumptions** | - Files already exist and conform to the declared column order and data types. <br> - The Hive user executing the script has read/write permission on the HDFS path. <br> - No partitions are defined; the whole dataset is read as a single logical table. |

---

## 4. Integration Points

| Connected Component | How it uses `supplier_rates` |
|---------------------|------------------------------|
| **Billing / Mediation jobs** (e.g., `move_master_consolidated_1.hql`, `msisdn_level_daily_usage_aggr.hql`) | Join on `ratekeyid` to fetch tariff, markup, and currency for charge calculations. |
| **Rate‑change pipelines** (not shown) | May replace the underlying HDFS files nightly; this DDL provides the stable view for downstream jobs. |
| **Impala UI / Reporting** | Queries the external table directly for rate‑lookup dashboards. |
| **Data quality / ETL validation scripts** | May read the table to verify row counts, nullability, or date ranges. |

*Because all DDL files live under `mediation-ddls/` and share the same `mnaas` database, any script that references `mnaas.<table>` can potentially join to `supplier_rates`.*

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – source file column order changes without DDL update. | Query failures / wrong pricing. | Add a **schema‑validation step** (e.g., a Hive `DESCRIBE` vs. a sample file header) in the ingestion pipeline. |
| **Missing or malformed source files** – HDFS path empty or contains bad rows. | Empty result set or job crashes. | Implement a **pre‑load health check** (file count, size, sample parse) before running dependent jobs. |
| **External table purge** – accidental `DROP TABLE` removes raw files. | Permanent data loss. | Restrict DROP privileges; enforce a **code‑review gate** for any DDL that includes `external.table.purge='true'`. |
| **Performance** – full scan of a large flat file on every query. | High latency, resource contention. | Consider **partitioning** (e.g., by `startdate`/`enddate`) or converting to a **managed ORC/Parquet** table. |
| **Permission issues** – Hive user cannot read/write HDFS location. | Table creation fails, downstream jobs break. | Verify HDFS ACLs and Kerberos tickets before deployment. |

---

## 6. Running / Debugging the Script

1. **Execute**  
   ```bash
   hive -f mediation-ddls/supplier_rates.hql
   # or via Ambari/Beeline:
   beeline -u jdbc:hive2://<hs2-host>:10000 -f supplier_rates.hql
   ```

2. **Validate creation**  
   ```sql
   SHOW CREATE TABLE mnaas.supplier_rates;
   DESCRIBE FORMATTED mnaas.supplier_rates;
   ```

3. **Sample query** (ensure data is readable)  
   ```sql
   SELECT ratekeyid, tariff1, currency
   FROM mnaas.supplier_rates
   LIMIT 10;
   ```

4. **Debugging tips**  
   - If the table is empty, check HDFS path: `hdfs dfs -ls /user/hive/warehouse/mnaas.db/supplier_rates`.  
   - Verify delimiter: `hdfs dfs -cat <file> | head -5` and confirm `;` separation.  
   - Look at Hive Metastore logs for permission or SerDe errors.  

---

## 7. Configuration / External Dependencies

| Item | Where it appears | Usage |
|------|------------------|-------|
| `NN-HA1` (NameNode host) | HDFS URI in `LOCATION` clause | Determines the Hadoop cluster where the source files reside. |
| Hive Metastore connection (implicit) | Not in script | Required for DDL registration. |
| Environment variables for Kerberos / Hadoop (`HADOOP_CONF_DIR`, `HIVE_CONF_DIR`) | Runtime environment | Needed for Hive/Impala to access HDFS and Metastore. |
| Impala catalog IDs (`impala.events.catalogServiceId`, `catalogVersion`) | Table properties | Used by Impala to keep its catalog in sync; no manual action required. |

If the deployment environment changes (e.g., a new NameNode address), the `LOCATION` string must be updated accordingly.

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning** – Introduce a partition on `startdate` (or `creationdate`) to limit scans to relevant date ranges and improve query performance.  
2. **Introduce a Validation Script** – Create a lightweight Hive/Presto job that checks row counts, nullability, and delimiter conformity of the source files before downstream jobs consume the table.

---