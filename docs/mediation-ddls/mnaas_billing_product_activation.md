**File:** `mediation-ddls\mnaas_billing_product_activation.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive view `mnaas.billing_product_activation`. The view consolidates distinct product‑activation records for 5G subscriptions by pulling `tcl_secs_id`, `iccid`, and `subscription_type` from the source tables `mnaas.nsa_fiveg_product_subscription` and `mnaas.month_billing`. It filters rows based on a configurable time window (`start_with_time` / `end_with_time`) and on de‑activation status, exposing only active or currently‑valid product activations for downstream billing and reporting jobs.

---

## 2. Key Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|-----------------|
| `billing_product_activation` | Hive **VIEW** | Provides a de‑duplicated list of active 5G product subscriptions for a given billing period. |
| `mnaas.nsa_fiveg_product_subscription` | Hive **TABLE** | Source of subscription identifiers (`tcl_secs_id`, `iccid`, `subscription_type`, `activation_date`, `deactivation_date`). |
| `mnaas.month_billing` | Hive **TABLE** | Supplies the temporal context (`start_with_time`, `end_with_time`) used to bound the activation window. |
| `start_with_time` / `end_with_time` | Hive **VARIABLEs** (passed via `-hiveconf`) | Define the inclusive start and end timestamps for the billing window. |

*Note:* The script does **not** contain explicit `JOIN` logic; the two tables are listed in the `FROM` clause separated by a comma, which Hive interprets as a Cartesian product filtered by the `WHERE` predicates.

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - Hive tables `mnaas.nsa_fiveg_product_subscription` and `mnaas.month_billing` (must exist and be up‑to‑date).<br>- Hive variables `start_with_time` and `end_with_time` (ISO‑8601 timestamps or Hive‑compatible date strings). |
| **Outputs** | - Hive **VIEW** `mnaas.billing_product_activation` (overwrites if already present). |
| **Side‑Effects** | - Alters the metastore definition for the view; no data is written or modified.<br>- Subsequent jobs that depend on this view will see the new definition immediately. |
| **Assumptions** | - Both source tables contain the columns referenced in the `SELECT` and `WHERE` clauses.<br>- `tcl_secs_id` is a positive integer for valid records.<br>- The environment supplies the two time variables; otherwise the view will be created with `NULL` values, causing incorrect results. |
| **External Services** | - Hive Metastore (for view creation).<br>- Potential downstream processes (e.g., nightly billing aggregation, reporting dashboards) that query the view. |

---

## 4. Integration Points & Connectivity

| Connected Component | Relationship |
|---------------------|--------------|
| **Downstream Billing Scripts** (e.g., `mnaas_billing_active_sims.hql`, `mnaas_billing_forked_traffic_usage.hql`) | These scripts query `billing_product_activation` to enrich usage records with product‑type information. |
| **Orchestration Layer** (e.g., Airflow, Oozie, custom scheduler) | The view‑creation script is typically run as a **pre‑step** in a nightly DAG before usage‑load jobs. |
| **Configuration Repository** | Variable values (`start_with_time`, `end_with_time`) are injected from a central properties file or environment (e.g., `billing_window.properties`). |
| **Monitoring/Alerting** | Metastore change events may be captured by a monitoring job to verify that the view exists after each run. |
| **Testing Suite** | Unit‑test scripts that execute `SHOW CREATE VIEW billing_product_activation` and validate the generated SQL against expected patterns. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Cartesian product explosion** – No explicit join condition may cause the query to scan `|subscription| × |billing|` rows, leading to long compile times or OOM failures. | Performance degradation, possible job failure. | Add a proper `JOIN` clause on a common key (e.g., `tcl_secs_id` or `billing_cycle_id`). Validate row counts after view creation. |
| **Missing/Incorrect time variables** – If `start_with_time` or `end_with_time` are not supplied, the view may return all rows or none. | Incorrect billing data downstream. | Enforce variable presence via a wrapper script that checks `hiveconf` values before invoking the HQL. |
| **Schema drift** – Source tables may evolve (column rename, type change). | View creation fails or returns wrong columns. | Include a schema‑validation step (e.g., `DESCRIBE` checks) in the orchestration before view creation. |
| **Permission issues** – The executing user may lack `CREATE VIEW` rights on the `mnaas` database. | Job aborts, no view created. | Ensure the service account is granted `CREATE`/`ALTER` privileges on the target schema. |
| **Stale view** – If the view is not refreshed after source data changes, downstream jobs may use outdated data. | Billing inaccuracies. | Schedule the view recreation at the start of each billing cycle; optionally use `CREATE OR REPLACE VIEW` to guarantee freshness. |

---

## 6. Running / Debugging the Script

### Typical Execution (via Hive CLI)

```bash
# Example: run for billing period 2023‑04‑01 to 2023‑04‑30
hive -f mediation-ddls/mnaas_billing_product_activation.hql \
     -hiveconf start_with_time='2023-04-01 00:00:00' \
     -hiveconf end_with_time='2023-04-30 23:59:59'
```

### Debug Checklist

1. **Validate variables** – `echo $start_with_time $end_with_time` before running.
2. **Check view existence** – `hive -e "SHOW VIEWS LIKE 'billing_product_activation';"`  
3. **Inspect generated SQL** – `hive -e "SHOW CREATE VIEW mnaas.billing_product_activation\G"`  
4. **Row‑count sanity** – `hive -e "SELECT COUNT(*) FROM mnaas.billing_product_activation;"`  
5. **Log capture** – Redirect Hive output to a log file for post‑run analysis: `... > /var/log/billing/product_activation_$(date +%F).log 2>&1`.

### Development / Unit Test

```sql
-- In a Hive session
SET start_with_time='2023-01-01 00:00:00';
SET end_with_time='2023-01-31 23:59:59';
SOURCE mediation-ddls/mnaas_billing_product_activation.hql;
SELECT * FROM mnaas.billing_product_activation LIMIT 10;
```

---

## 7. External Configurations & Files

| Config / File | Purpose |
|---------------|---------|
| `billing_window.properties` (or similar) | Holds default values for `start_with_time` and `end_with_time`; read by the orchestration wrapper before invoking Hive. |
| Hive **environment variables** (`HIVE_CONF_DIR`, `HADOOP_CONF_DIR`) | Required for Hive client to locate metastore and Hadoop resources. |
| Service account credentials (Kerberos keytab, if enabled) | Needed for authentication to Hive Metastore. |

The HQL itself does **not** reference any other files directly; all external dependencies are injected at runtime via Hive variables or the surrounding orchestration layer.

---

## 8. Suggested Improvements (TODO)

1. **Add an explicit join condition** – Replace the implicit Cartesian product with a proper `JOIN` on a shared key (e.g., `tcl_secs_id` or a billing‑cycle identifier) to dramatically reduce execution time and resource consumption.  
   ```sql
   FROM mnaas.nsa_fiveg_product_subscription ps
   JOIN mnaas.month_billing mb
     ON ps.tcl_secs_id = mb.tcl_secs_id   -- example key
   ```

2. **Guard against missing variables** – Wrap the view creation in a small pre‑check script (Bash/Python) that aborts with a clear error if `start_with_time` or `end_with_time` are undefined or malformed. This prevents silent creation of a view with an unintended time window.  

---