**File:** `mediation-ddls\mnaas_v_ictraffic_details_ppu_billing.hql`

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.v_ictraffic_details_ppu_billing`. The view consolidates raw inter‑connect traffic records with the monthly “price‑per‑unit” (PPU) billing configuration, filters out invalid or incomplete records, normalises fields (e.g., call type, sponsor, usage calculations), and de‑duplicates CDRs using a window function. The resulting dataset is the canonical source for downstream inter‑connect billing jobs that generate invoices, revenue reports, and settlement files.

---

## 2. Core Components & Responsibilities  

| Component | Type | Responsibility |
|-----------|------|-----------------|
| **`v_ictraffic_details_ppu_billing`** | Hive **VIEW** | Exposes a cleaned, enriched, and de‑duplicated set of inter‑connect CDRs ready for PPU billing. |
| **`filtered_cdr` CTE** | Common Table Expression | Joins `interconnect_traffic_details_raw` with `month_ppu_billing`, applies time‑window filter (`record_insertime BETWEEN start_inserttime AND end_inserttime`), validates mandatory fields, derives derived columns (`sponsor`, `calltype`, `usage`, `actualusage`, `usagefor`, etc.). |
| **`row_number() … OVER (PARTITION BY cdrid, chargenumber, timebandindex ORDER BY filename DESC)`** | Window function | Implements deterministic de‑duplication – keeps the most recent file version for each unique CDR key. |
| **`decode()` calls** | Hive built‑in function | Maps raw `calltype` values to internal codes (`ICSMS`, `ICVoice`) and selects the appropriate usage metric (`icbillableduration` for voice, constant `1` for SMS). |
| **`substring(imsi,1,5) sponsor`** | Expression | Extracts the first five digits of the IMSI to identify the sponsoring operator. |
| **`CAST(customernumber AS INT) tcl_secs_id`** | Expression | Normalises the customer identifier for downstream joins. |
| **`filename gen_filename`** | Alias | Preserves the original source filename for audit / traceability. |
| **`partition_date`** | Column (passed through) | Enables downstream partition‑pruned queries (e.g., daily/monthly aggregates). |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Source tables** | `mnaas.interconnect_traffic_details_raw` (raw CDR dump) <br> `mnaas.month_ppu_billing` (monthly PPU configuration) |
| **Hive variables / parameters** | `start_inserttime` – lower bound of `record_insertime` <br> `end_inserttime` – upper bound of `record_insertime` |
| **Output** | Hive **VIEW** `mnaas.v_ictraffic_details_ppu_billing` (no physical storage; logical definition only) |
| **Side effects** | DDL operation – creates or replaces the view definition in the metastore. No data is written or modified. |
| **Assumptions** | • Both source tables exist and contain the columns referenced in the SELECT. <br> • `record_insertime`, `customernumber`, `calltype`, `traffictype`, `partnertype` are populated correctly. <br> • Hive/Impala engine supports `decode`, `substring`, `row_number` window functions. <br> • The orchestrating job supplies `start_inserttime` and `end_inserttime` as Hive variables. |

---

## 4. Integration Points  

| Connected Component | Relationship |
|---------------------|--------------|
| **Previous DDLs** (e.g., `mnaas_traffic_details_billing.hql`, `mnaas_traffic_details_aggr_daily.hql`) | Down‑stream scripts query this view to compute billing aggregates, generate settlement files, or feed reporting tables. |
| **ETL Orchestrator** (Airflow / Oozie / custom scheduler) | Executes this script as part of the “inter‑connect billing” DAG, typically after the raw CDR ingestion job and before aggregation jobs. |
| **Configuration Store** (`month_ppu_billing`) | Provides the per‑unit price and billing rules that are joined in the view. Any change to this table automatically reflects in the view without script modification. |
| **Monitoring / Alerting** | Job logs (Hive CLI / Beeline) are parsed by the monitoring framework to detect view creation failures. |
| **Data Quality Checks** | Subsequent scripts (e.g., `mnaas_traffic_details_billing_late_cdr.hql`) may run row‑count or null‑value checks against this view to ensure completeness before proceeding to invoicing. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or mismatched source columns** (schema drift) | View creation fails; downstream jobs break. | Add a pre‑run schema validation step (e.g., `DESCRIBE` + `INFORMATION_SCHEMA.COLUMNS`). |
| **`start_inserttime` / `end_inserttime` not supplied** | Full table scan → long run time, possible OOM. | Enforce variable presence via `SET hivevar:start_inserttime;` and abort if null. |
| **Duplicate CDRs not correctly resolved** (wrong `row_num` logic) | Over‑billing or under‑billing. | Unit‑test the window function on a representative sample; add a checksum column for audit. |
| **Large daily volume causing Hive job OOM** | Job failure, delayed billing cycle. | Ensure proper YARN memory settings; consider partitioning `interconnect_traffic_details_raw` on `record_insertime` and enable partition pruning. |
| **Incorrect `decode` mapping** (new call types introduced) | Mis‑classification of traffic. | Centralise call‑type mapping in a lookup table and join instead of hard‑coded `decode`. |
| **View not refreshed after underlying table changes** | Stale data used in billing. | Use `CREATE OR REPLACE VIEW` (already done) and schedule the script after every raw‑CDR load. |

---

## 6. Running / Debugging the Script  

1. **Typical execution (via scheduler)**  
   ```bash
   hive -hiveconf start_inserttime=2025-12-01T00:00:00 \
        -hiveconf end_inserttime=2025-12-01T23:59:59 \
        -f mediation-ddls/mnaas_v_ictraffic_details_ppu_billing.hql
   ```

2. **Manual run (developer)**  
   ```bash
   beeline -u jdbc:hive2://<hive-host>:10000/default \
           -n <user> -p <pwd> \
           --hivevar start_inserttime='2025-12-01 00:00:00' \
           --hivevar end_inserttime='2025-12-01 23:59:59' \
           -f mediation-ddls/mnaas_v_ictraffic_details_ppu_billing.hql
   ```

3. **Debug steps**  
   * Verify variables are set: `SET start_inserttime; SET end_inserttime;`  
   * Run the CTE alone to inspect row counts:  
     ```sql
     WITH filtered_cdr AS ( … same SELECT … )
     SELECT COUNT(*) FROM filtered_cdr;
     ```  
   * Check de‑duplication:  
     ```sql
     SELECT cdrid, COUNT(*) FROM filtered_cdr GROUP BY cdrid HAVING COUNT(*) > 1;
     ```  
   * Inspect a sample of the view: `SELECT * FROM mnaas.v_ictraffic_details_ppu_billing LIMIT 10;`

4. **Log monitoring**  
   * Hive job logs are written to YARN’s application UI; look for “CREATE VIEW” success message.  
   * Set up an alert on non‑zero exit code from the script.

---

## 7. External Config / Environment Dependencies  

| Item | Usage |
|------|-------|
| **Hive variables** `start_inserttime`, `end_inserttime` | Define the ingestion window; passed by the orchestrator (Airflow/Oozie) or manually via `-hiveconf`. |
| **Metastore connection** | Required to register the view; configured in the Hive client (`hive-site.xml`). |
| **YARN / Hadoop resource settings** | Must be sufficient for the join between raw CDRs and billing config (memory, containers). |
| **`month_ppu_billing` table** | Populated daily/monthly by a separate pricing feed; the view assumes it is up‑to‑date. |

---

## 8. Suggested Improvements (TODO)

1. **Parameter Validation Block** – Add a small pre‑SQL block that aborts with a clear error if `start_inserttime` or `end_inserttime` are missing or malformed.  
   ```sql
   SET hivevar:missing_params = '';
   IF '${hiveconf:start_inserttime}' = '' THEN SET hivevar:missing_params = concat(${hivevar:missing_params}, ' start_inserttime'); END IF;
   IF '${hiveconf:end_inserttime}' = '' THEN SET hivevar:missing_params = concat(${hivevar:missing_params}, ' end_inserttime'); END IF;
   IF '${hivevar:missing_params}' != '' THEN
       SELECT concat('Missing required Hive variables:', ${hivevar:missing_params}) AS error_msg;
       EXIT;
   END IF;
   ```

2. **Externalise Call‑Type Mapping** – Replace the hard‑coded `decode(calltype, …)` with a lookup table (`mnaas.calltype_mapping`) to simplify future additions (e.g., data, video). This also enables non‑SQL consumers to maintain the mapping.  

---