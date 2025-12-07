**File:** `mediation-ddls\mnaas_msisdn_level_daily_usage_aggr_view.hql`  

---

## 1. High‑Level Summary
This script creates (or replaces) the Hive/Impala view `mnaas.msisdn_level_daily_usage_aggr_view`. The view enriches the daily per‑MSISDN usage aggregation (`msisdn_level_daily_usage_aggr`) with country information (via `mcc_cntry_mapping`) and a derived flag indicating whether the SIM is 5G‑enabled (by left‑joining the `nsa_fiveg_product_subscription` table). The resulting view is the canonical source for downstream billing, sponsor, and reporting jobs that need a flat, denormalised record per MSISDN per day.

---

## 2. Key Objects & Responsibilities  

| Object | Type | Responsibility |
|--------|------|-----------------|
| `msisdn_level_daily_usage_aggr_view` | **VIEW** | Provides a denormalised, daily‑level usage record per MSISDN with enriched geography and 5G status. |
| `mnaas.msisdn_level_daily_usage_aggr` (alias **m**) | **TABLE** | Source of raw daily usage metrics (voice, data, SMS, cost, device info, etc.). |
| `mnaas.mcc_cntry_mapping` (alias **c**) | **TABLE** | Maps Mobile Country Code (`mcc`) to human‑readable country name (`cntry_name`). |
| `mnaas.nsa_fiveg_product_subscription` (alias **fiveg**) | **TABLE** | Holds SIM‑level 5G subscription records; used to compute the `5g_enabled` flag. |
| `decode(nvl(fiveg.iccid,'N'),'N','N','Y')` | **Expression** | Returns `'Y'` when a matching 5G subscription exists, otherwise `'N'`. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - `mnaas.msisdn_level_daily_usage_aggr` (must contain all columns listed in the SELECT).<br>- `mnaas.mcc_cntry_mapping` (must have columns `mcc` and `cntry_name`).<br>- `mnaas.nsa_fiveg_product_subscription` (must have columns `iccid` and `activation_date`). |
| **Outputs** | - Hive/Impala **VIEW** `mnaas.msisdn_level_daily_usage_aggr_view`. No physical data files are written; the view metadata is stored in the metastore. |
| **Side‑Effects** | - Alters the metastore (creates or replaces the view).<br>- May invalidate dependent cached query plans. |
| **Assumptions** | - The execution environment is Hive/Impala with support for `CREATE VIEW` and `decode`/`nvl` functions.<br>- `partition_date` is a string/date column present in `msisdn_level_daily_usage_aggr` and matches the format of `activation_date` (YYYY‑MM‑DD…); the `substring(...,1,10)` extracts the date portion.<br>- No column name collisions beyond those explicitly aliased (e.g., `m.mcc mcc`).<br>- The underlying tables are refreshed daily before this view is created. |

---

## 4. Integration Points  

| Connected Component | How It Connects |
|----------------------|-----------------|
| **Downstream ETL jobs** (e.g., `mnaas_month_billing.hql`, `mnaas_mmr_sponsor_master.hql`) | Query this view to obtain enriched usage for billing, sponsor attribution, and reconciliation. |
| **Reporting dashboards** (BI tools, Tableau, PowerBI) | Consume the view directly for daily usage dashboards. |
| **Data quality / reconciliation scripts** (e.g., `mnaas_mnpportinout_daily_recon_inter.hql`) | May join against this view to validate usage against network events. |
| **Partition‑pruning jobs** | The view includes `partition_date` in the SELECT list, enabling downstream jobs to filter on it for incremental loads. |
| **Configuration / environment** | The script relies on the default Hive/Impala database `mnaas`. If a different schema is used, an environment variable (e.g., `TARGET_DB`) would need to be substituted. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Schema drift** – source tables gain/drop columns used in the view. | View creation fails; downstream jobs break. | Implement a CI validation step that compares the view definition against the current table schemas before deployment. |
| **Cartesian product** – missing or mismatched join keys (e.g., `m.mcc` not present in `c`). | Unexpected row explosion, performance degradation. | Add explicit `WHERE c.mcc IS NOT NULL` or enforce foreign‑key‑like constraints via data quality checks. |
| **5G flag logic error** – `activation_date` format changes. | Incorrect `5g_enabled` values. | Normalize `activation_date` to a DATE type in the source table; use `to_date()` instead of string `substring`. |
| **View recreation without `IF NOT EXISTS`** – accidental overwrite during incremental deployments. | Potential loss of view permissions or comments. | Use `CREATE OR REPLACE VIEW` (if supported) or wrap in a conditional block that checks existence first. |
| **Performance** – large daily usage table leads to slow view creation. | Deployment windows missed. | Partition the source table on `partition_date` and ensure the view does not force full scans; test with `EXPLAIN` plans. |

---

## 6. Running / Debugging the Script  

1. **Execute**  
   ```bash
   hive -f mediation-ddls/mnaas_msisdn_level_daily_usage_aggr_view.hql
   # or with beeline
   beeline -u jdbc:hive2://<host>:10000/mnaas -f mediation-ddls/mnaas_msisdn_level_daily_usage_aggr_view.hql
   ```

2. **Verify Creation**  
   ```sql
   SHOW CREATE VIEW mnaas.msisdn_level_daily_usage_aggr_view;
   ```

3. **Sample Query** (quick sanity check)  
   ```sql
   SELECT COUNT(*) AS rows, 
          SUM(data_usage_in_mb) AS total_mb,
          COUNT(DISTINCT msisdn) AS unique_msisdn
   FROM mnaas.msisdn_level_daily_usage_aggr_view
   WHERE partition_date = CURRENT_DATE - INTERVAL '1' DAY;
   ```

4. **Debugging Tips**  
   - If the script fails, run the SELECT part alone to isolate column or join issues.  
   - Use `DESCRIBE FORMATTED mnaas.msisdn_level_daily_usage_aggr;` to confirm column names/types.  
   - Check the Hive metastore logs for permission or syntax errors.  

---

## 7. External Config / Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `TARGET_DB` (optional) | If the deployment environment uses a different schema, the script could be templated to replace `mnaas` with `${TARGET_DB}`. Not present in the current file but commonly used in the repo. |
| Hive/Impala connection properties (e.g., `hive.metastore.uris`) | Required for the CLI/Beeline to reach the metastore; defined outside this script (usually in `hive-site.xml` or environment profiles). |

---

## 8. Suggested Improvements (TODO)

1. **Add defensive view creation** – prepend the statement with `CREATE OR REPLACE VIEW` (or a conditional `IF NOT EXISTS`) to make deployments idempotent and preserve view metadata.  
2. **Explicit column alias for ambiguous names** – the SELECT currently uses `m.mcc mcc` (missing a comma) which may cause parsing errors on some engines; rewrite as `m.mcc AS mcc`. Also add comments describing each column for future maintainers.  

--- 

*End of documentation.*