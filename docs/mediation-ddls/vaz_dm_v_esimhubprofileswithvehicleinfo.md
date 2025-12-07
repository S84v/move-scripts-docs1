**File:** `mediation-ddls\vaz_dm_v_esimhubprofileswithvehicleinfo.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates an **external Parquet table** named `mnaas.vaz_dm_v_esimhubprofileswithvehicleinfo`. The table stores enriched eSIM hub profile records together with vehicle information (VIN, UPID, etc.). Because it is external, Hive only registers the metadata; the actual data files reside under a fixed HDFS location (`hdfs://NN-HA1/.../vaz_dm_v_esimhubprofileswithvehicleinfo`). Down‑stream mediation and analytics jobs read this table to join eSIM profile data with vehicle‑related attributes for reporting, billing, or fraud‑prevention use‑cases.

---

## 2. Core Objects Defined in the Script  

| Object | Type | Responsibility / Description |
|--------|------|--------------------------------|
| `vaz_dm_v_esimhubprofileswithvehicleinfo` | Hive external table | Holds eSIM hub profile rows enriched with vehicle identifiers and customer context. |
| Columns (`vin`, `upid`, `updatedate`, `profiletype`, `msisdn`, `iccid`, `esimhubstatus`, `eid`, `devicetype`, `customerid`, `countryofsale`) | Table schema | Provide the data model for downstream joins (e.g., with inventory, usage, or billing tables). |
| `ROW FORMAT SERDE … ParquetHiveSerDe` | SerDe definition | Enables Hive to read/write Parquet files with Snappy compression. |
| `LOCATION 'hdfs://NN-HA1/.../vaz_dm_v_esimhubprofileswithvehicleinfo'` | Physical storage path | Points to the HDFS directory where the Parquet files are stored. |
| `TBLPROPERTIES` (bucketing_version, parquet.compression, transient_lastDdlTime) | Table metadata | Controls internal Hive behaviour (bucketing version, compression codec, DDL timestamp). |

*No procedural code (functions, classes) is present – the file is pure DDL.*

---

## 3. Inputs, Outputs & Side‑Effects  

| Aspect | Details |
|--------|---------|
| **Inputs** | None at execution time. The script consumes only Hive metastore configuration and the HDFS namenode address (`NN-HA1`). |
| **Outputs** | 1. A new entry in the Hive metastore for `mnaas.vaz_dm_v_esimhubprofileswithvehicleinfo`. <br>2. No data files are created; the table points to an existing (or future) HDFS directory. |
| **Side‑Effects** | - Registers the external table metadata. <br>- May overwrite an existing table definition if the same name already exists (Hive will replace the metadata). |
| **Assumptions** | - The HDFS path exists and is writable by the Hive service account. <br>- Data files placed there conform to the declared Parquet schema (Snappy compression). <br>- Hive metastore is reachable and the `mnaas` database already exists. <br>- Down‑stream jobs expect column names and types exactly as defined. |

---

## 4. Integration Points with Other Scripts / Components  

| Connected Component | Relationship |
|---------------------|--------------|
| **`move_sim_inventory_status*.hql`**, **`move_sim_inventory_count.hql`** | Likely join on `iccid` / `msisdn` to enrich inventory counts with vehicle info. |
| **`msisdn_level_daily_usage_aggr.hql`** | May use `msisdn` to correlate usage aggregates with vehicle‑linked eSIM profiles. |
| **ETL orchestration layer (e.g., Oozie / Airflow / custom scheduler)** | Executes this DDL as part of a “schema provisioning” phase before data ingestion jobs run. |
| **Data ingestion pipelines (Kafka → Hive, SFTP loaders, etc.)** | Write Parquet files into the HDFS location; the table provides the read‑side view. |
| **Reporting / BI tools (Tableau, PowerBI, custom dashboards)** | Query this table to surface vehicle‑linked eSIM metrics. |
| **External configuration** | The HDFS namenode (`NN-HA1`) and Hive metastore connection are typically supplied via environment variables (`HIVE_CONF_DIR`, `HADOOP_CONF_DIR`) or a central `hive-site.xml`. No hard‑coded values besides the location path appear in the script. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Path drift / missing HDFS directory** | Table appears but queries return empty or error. | Automate a pre‑check that the directory exists and has correct permissions before running the DDL. |
| **Schema mismatch between incoming Parquet files and table definition** | Query failures, data corruption. | Enforce schema validation in the ingestion pipeline (e.g., Avro/Parquet schema registry) and add a nightly Hive `DESCRIBE FORMATTED` audit. |
| **Uncontrolled growth of files (no partitioning)** | Poor query performance, long scan times. | Add partitioning (e.g., by `countryofsale` or `updatedate` date) in a future version of the table. |
| **External table replacement overwriting existing metadata** | Down‑stream jobs may break if column order/type changes. | Use `CREATE EXTERNAL TABLE IF NOT EXISTS` or versioned table names; keep DDL under source‑control with change‑approval workflow. |
| **Insufficient ACLs on HDFS path** | Unauthorized read/write, security breach. | Apply HDFS ACLs limiting access to the Hive service account and the ingestion service account only. |
| **Lack of table comment / documentation** | New developers cannot quickly understand purpose. | Add a `COMMENT` clause in the DDL and maintain a data‑dictionary entry. |

---

## 6. Running / Debugging the Script  

1. **Execute**  
   ```bash
   hive -f mediation-ddls/vaz_dm_v_esimhubprofileswithvehicleinfo.hql
   ```
   *Or via the orchestration tool that runs all DDLs in the `mediation-ddls` folder.*

2. **Verify creation**  
   ```sql
   SHOW TABLES LIKE 'vaz_dm_v_esimhubprofileswithvehicleinfo';
   DESCRIBE FORMATTED mnaas.vaz_dm_v_esimhubprofileswithvehicleinfo;
   ```

3. **Check underlying files** (as the Hive service user)  
   ```bash
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/vaz_dm_v_esimhubprofileswithvehicleinfo
   ```

4. **Sample query** (ensure data is readable)  
   ```sql
   SELECT vin, msisdn, updatedate
   FROM mnaas.vaz_dm_v_esimhubprofileswithvehicleinfo
   LIMIT 10;
   ```

5. **Debugging tips**  
   - If Hive reports “Table already exists”, decide whether to drop & recreate or use `IF NOT EXISTS`.  
   - Permission errors (`Permission denied`) indicate HDFS ACL issues – check with `hdfs dfs -chmod` / `-setfacl`.  
   - Parquet schema errors appear as “Cannot deserialize” – verify the producer’s schema matches the column list.

---

## 7. External Config / Environment Dependencies  

| Item | How it is used |
|------|----------------|
| `HIVE_CONF_DIR` / `hive-site.xml` | Provides Hive metastore URI, authentication, and default database location. |
| `HADOOP_CONF_DIR` / `core-site.xml` | Supplies the HDFS namenode address (`NN-HA1`) and client settings. |
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` clause; must resolve via DNS or `/etc/hosts`. |
| Optional environment variables (e.g., `HIVE_EXECUTION_ENGINE`) | May affect how the DDL is parsed/executed but are not referenced directly in the script. |

---

## 8. Suggested Improvements (TODO)

1. **Add Partitioning** – Redesign the table to be partitioned by `countryofsale` (or by `updatedate` date) to improve query performance and enable incremental data loads.  
2. **Include a Table Comment & Column Descriptions** – Add a `COMMENT 'eSIM hub profiles enriched with vehicle info'` clause and `COMMENT` on each column to aid data‑dictionary generation and future onboarding.  

---