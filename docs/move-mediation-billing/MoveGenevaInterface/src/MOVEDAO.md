# Summary
`MOVEDAO.properties` is a property‑file repository of named SQL statements used by the **MoveGenevaInterface** component. In production it supplies all Hive/Hadoop and Oracle queries required to extract, aggregate, and persist activation, usage, CDR, and reject‑handling data for the Move‑mediation billing reconciliation workflow.

# Key Components
- **SQL query entries** – Named statements (e.g., `activations.fetch`, `data.fetch`, `cdr.subset.insert`, `sim.reject.insert`) that are loaded by the DAO layer and executed via JDBC/Hive drivers.  
- **Parameterized placeholders** – `{0}`, `{1}`, `?` markers that are replaced at runtime with dynamic filter lists or bind variables.  
- **Batch/CTE statements** – Queries employing Hive CTEs (`with … as`) for multi‑step aggregation of traffic usage.  
- **DDL/DML for metadata** – `truncate`, `insert`, `update` statements that manage staging tables (`month_billing`, `geneva_reject`, `sim_geneva`, `usage_geneva`).  

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Extraction | Hadoop/Hive tables (`mnaas.billing_*`, `mnaas.traffic_details_raw_daily`) and Oracle tables (`gen_sim_product`, `v_gen_usage_product`) | Parameter substitution → JDBC/Hive execution | ResultSets consumed by Java service for further aggregation |
| Staging | `month_billing` temporary table | `truncate` + `insert` | Populated month‑range metadata |
| CDR Filtering | Raw CDR tables (`traffic_details_raw_daily`) | `cdr.subset.truncate` + `cdr.subset.insert` (window functions, decode) | `billing_traffic_filtered_cdr` populated |
| Aggregation | Filtered CDRs | `date.wise.cdr.fetch`, `in.count.usage.fetch`, `out.usage.fetch*` | Aggregated usage metrics per day, destination, traffic type |
| Persist / Reject handling | Aggregated metrics | `sim.reject.insert`, `usage.reject.insert`, `sim.update`, `usage.update` | Oracle `geneva_reject`, `sim_geneva`, `usage_geneva` tables updated |
| Mapping | File‑CDR mapping tables | `ra.truncate`, `ra.insert`, `ra.update` | `gen_file_cdr_mapping` refreshed |

External services: Hive metastore, Oracle DB, optional logging/monitoring via Log4j.

# Integrations
- **MoveGenevaInterface Java module** – Loads this file via `java.util.Properties`, maps keys to prepared statements, and executes them through `org.apache.hadoop.hive.jdbc.HiveDriver` or Oracle `oracle.jdbc.OracleDriver`.  
- **Gradle build** – `gradle-wrapper.properties` ensures consistent Gradle version for compiling the DAO classes that reference this file.  
- **Log4j configuration** – `log4j.properties` controls logging of query execution, errors, and performance metrics.  
- **Batch orchestration** – Invoked by scheduled Spark/MapReduce jobs or Spring Batch flows that drive the move‑mediation pipeline.

# Operational Risks
- **SQL injection via placeholder substitution** – Ensure all `{0}` list expansions are sanitized and bound as prepared‑statement parameters.  
- **Schema drift** – Hive/Oracle table changes (column rename, type change) break queries; maintain versioned schema contracts and automated integration tests.  
- **Performance bottlenecks** – Complex CTEs and window functions on large CDR volumes may cause long runtimes; monitor Hive execution plans and add partition pruning.  
- **Missing/incorrect property keys** – Runtime `MissingResourceException` if a required query key is absent; enforce property‑file validation at startup.

# Usage
```bash
# Run a single query from the CLI (example)
java -cp move-geneva-interface.jar com.example.dao.QueryRunner \
     -prop move-mediation-billing/MoveGenevaInterface/src/MOVEDAO.properties \
     -key data.fetch \
     -param "2023-01-01" "2023-01-31"
```
*In Java code*:  
```java
Properties daoProps = new Properties();
try (InputStream is = getClass().getResourceAsStream("/MOVEDAO.properties")) {
    daoProps.load(is);
}
String sql = MessageFormat.format(daoProps.getProperty("data.fetch"), paramList);
PreparedStatement ps = hiveConn.prepareStatement(sql);
ResultSet rs = ps.executeQuery();
```

# configuration
- **Environment variables** – `HIVE_JDBC_URL`, `ORACLE_JDBC_URL`, `HIVE_USER`, `ORACLE_USER`, `HIVE_PASSWORD`, `ORACLE_PASSWORD`.  
- **Referenced config files** – `log4j.properties` (logging), `gradle-wrapper.properties` (build), any external `application.yml` that supplies DB connection pools.  

# Improvements
1. **Migrate to parameterised prepared statements** – Replace string interpolation (`{0}`) with `?` placeholders and use `PreparedStatement` batch binding to eliminate injection risk and improve plan caching.  
2. **Introduce schema versioning** – Store a `dao.schema.version` property and validate against a versioned DDL manifest at startup; automate migration scripts when the version changes.