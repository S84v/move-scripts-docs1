**# Summary**  
`MOVEDAO.properties` is a Java Properties resource used by the KLMReport component of the Move‑Mediation‑Ingest pipeline. It centralises all Hive/SQL statements required to extract, transform and load (ETL) traffic‑detail data for a specific KLM‑offer (`MOVE SIM AVKLM‑4Z‑EUS‑$500`). At runtime the application loads the file, substitutes the positional parameters (`?`) with values supplied by the orchestration layer, and executes the statements against the `mnaas` Hive metastore (or compatible RDBMS) to populate intermediate tables and generate billing reports.

---

**# Key Components**  

- **`klm.summary`** – Simple SELECT used for quick overview of traffic summary.  
- **`klm.allin`** – Complex CTE‑based query that filters raw traffic, resolves missing offer descriptions, joins MCC‑country mapping, and aggregates usage per day/zone.  
- **`cdr.subset.truncate` / `cdr.subset.insert`** – DDL/DML pair that rebuilds a filtered CDR staging table for the target `tcl_secs_id` and offer.  
- **`date.wise.cdr.fetch`** – Aggregates usage per calendar day from the filtered CDR table.  
- **`cdr.fetch`** – Retrieves individual CDR rows for a given day (used for detailed audit).  
- **`in.usage.fetch` / `out.usage.fetch`** – Cumulative usage queries bounded by a CDR number (used for incremental inbound/outbound usage calculations).  
- **`truncate.month` / `insert.month`** – Maintenance of the `month_reports` control table.  
- **`truncate.traffic` / `insert.traffic`** – Refresh of the `traffic_details_billing` fact table from the view `v_traffic_details_billing_klm`.  

Each key maps to a single SQL string; the application substitutes the `?` placeholders with runtime parameters (e.g., `zoneid`, `start_date`, `end_date`, `cdrnum`).

---

**# Data Flow**  

| Stage | Input | Process | Output / Side‑Effect |
|-------|-------|---------|----------------------|
| 1. Summary retrieval | None (static) | `klm.summary` SELECT | ResultSet displayed in UI / logs |
| 2. Detailed aggregation | `start_date`, `end_date`, `zoneid` | `klm.allin` (CTE) | Populated temporary result set for reporting |
| 3. CDR staging | `tcl_secs_id` (36378), `calltype='Data'`, `proposition_addon` | `cdr.subset.truncate` + `cdr.subset.insert` | Populated `billing_traffic_filtered_cdr` table |
| 4. Daily roll‑up | None | `date.wise.cdr.fetch` | Daily usage aggregates (used for batch jobs) |
| 5. Per‑day detail | `calldate` | `cdr.fetch` | Detailed CDR rows (audit/debug) |
| 6. Incremental usage | `cdrnum` bound | `in.usage.fetch` / `out.usage.fetch` | Cumulative usage up to / from a marker |
| 7. Month control | `month` parameters | `truncate.month` + `insert.month` | Updated `month_reports` control rows |
| 8. Traffic refresh | None (full refresh) | `truncate.traffic` + `insert.traffic` | Re‑populated `traffic_details_billing` from view |

External services: Hive/Impala or compatible SQL engine hosting the `mnaas` schema; JDBC driver; optional Hadoop HDFS for underlying tables.

---

**# Integrations**  

- **Java DAO layer** – `MOVEDAO` class loads this properties file via `java.util.Properties`, prepares `PreparedStatement`s, and injects parameters supplied by the orchestration service (`MoveMediationJob`).  
- **Orchestration workflow** – Called from the KLMReport batch job (Spring Batch / Oozie) after the Hadoop ingestion step.  
- **Logging** – Queries are logged through Log4j (see `log4j.properties`).  
- **Metrics** – Execution time and row counts are emitted to Prometheus via the job’s instrumentation wrapper.  

---

**# Operational Risks**  

- **Hard‑coded `tcl_secs_id` (36378)** – Limits reuse; change requires code redeploy. *Mitigation*: externalise to a config property.  
- **SQL injection via unchecked parameters** – All statements use `?` placeholders, but any dynamic string concatenation must be avoided. *Mitigation*: enforce `PreparedStatement` usage and validate inputs.  
- **Table truncation side‑effects** – `truncate` statements drop all rows; concurrent jobs could lose data. *Mitigation*: schedule exclusive windows or use partition‑level deletes.  
- **Schema drift** – Queries reference many columns; schema changes break runtime. *Mitigation*: maintain versioned DDL scripts and CI validation against a test schema.  

---

**# Usage**  

```bash
# Example: run the KLMReport job locally (debug mode)
export LOGFILE_NAME=/tmp/klmreport.log
export HIVE_JDBC_URL=jdbc:hive2://hive-prod:10000/default
export HIVE_USER=move_user
export HIVE_PASSWORD=********

# Java entry point (Spring Boot)
java -jar klmreport.jar \
  --job=klm-allin \
  --startDate=2024-01-01 \
  --endDate=2024-01-31 \
  --zoneId=10
```

The job loads `MOVEDAO.properties`, binds the supplied arguments to the `?` placeholders, and executes the defined workflow.

---

**# Configuration**  

- **Environment variables**  
  - `HIVE_JDBC_URL`, `HIVE_USER`, `HIVE_PASSWORD` – DB connection.  
  - `LOGFILE_NAME` – Log4j file target.  
- **External config files**  
  - `log4j.properties` – logging.  
  - `application.yml` (Spring) – job scheduling, thread pool sizes.  
- **Properties file** – `MOVEDAO.properties` itself is the sole source of SQL statements.  

---

**# Improvements**  

1. **Parameterise `tcl_secs_id` and offer name** – Move these constants into the properties file or a dedicated