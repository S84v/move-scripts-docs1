# Summary
`MediationDataAccess` is a DAO class that orchestrates Hive/Impala data‑manipulation for the KLMReport component of the Move‑Mediation‑Ingest pipeline. It loads billing month metadata, populates staging tables, extracts summary and detailed usage records, creates filtered CDR subsets, and splits in‑bundle/out‑of‑bundle records for the **MOVE SIM AVKLM‑4Z‑EUS‑$500** offer.

# Key Components
- **insertMonth(String monthYear)**
  - Truncates `mnaas.month_reports` and inserts a new billing‑month row.
- **insertTrafficTableMonth()**
  - Truncates `mnaas.traffic_details_billing` and inserts daily traffic view data.
- **getKLMSummary() → Map<Integer,KLMRecord>**
  - Executes `klm.summary` query; builds a zone‑keyed map of `KLMRecord`.
- **getInBundleRecords(KLMRecord) → List<KLMRecord>**
  - Executes CTE `klm.allin` for a specific zone; returns per‑day in‑bundle usage.
- **createCDRSubset(KLMRecord)**
  - Truncates `mnaas.billing_traffic_filtered_cdr` and inserts filtered CDRs for a zone.
- **splitCDRRecords(BundleSplitRecord) → BundleSplitRecord**
  - Determines the CDR where the allocated bundle is exhausted; populates in‑/out‑bundle record lists.
- **getSplitRecord(String callDate, double bundle) → SplitRecord**
  - Finds the exact CDR that crosses the bundle boundary on a given date.
- **fetchInBundleList(SplitRecord,int) → List<KLMRecord>**
  - Retrieves all CDRs up to the in‑bundle end CDR, applying split adjustments.
- **fetchOutBundleList(SplitRecord,int) → List<KLMRecord>**
  - Retrieves all CDRs from the out‑bundle start CDR, applying split adjustments.
- **getStackTrace(Exception) → String**
  - Utility for logging full stack traces.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | `monthYear` (e.g., “2023‑04”) | `insertMonth` → truncate + insert into `mnaas.month_reports` | New month row persisted |
| 2 | – | `insertTrafficTableMonth` → truncate + insert into `mnaas.traffic_details_billing` | Staging traffic table refreshed |
| 3 | – | `getKLMSummary` → query `klm.summary` | Map of zone → `KLMRecord` (summary) |
| 4 | `KLMRecord` (zone) | `getInBundleRecords` → query `klm.allin` | List of daily in‑bundle usage |
| 5 | `KLMRecord` (zone) | `createCDRSubset` → truncate + insert filtered CDRs | `mnaas.billing_traffic_filtered_cdr` populated |
| 6 | `BundleSplitRecord` (total bundle) | `splitCDRRecords` → date‑wise aggregation, split detection, fetch in/out lists | Updated `BundleSplitRecord` with in‑/out‑bundle record collections |
| 7 | `callDate`, `bundle` | `getSplitRecord` → scan CDRs for split point | `SplitRecord` describing split CDR |
| 8 | `SplitRecord`, zone | `fetchInBundleList` / `fetchOutBundleList` → query filtered CDR view | Lists of `KLMRecord` for reporting |

External services:
- **Impala/Hive** via `JDBCConnection.getImpalaConnection()`
- **Log4j** for audit logging
- **ResourceBundle** (`MOVEDAO.properties`) for SQL statements

# Integrations
- **JDBCConnection**: Provides singleton Impala connections; used by every DAO method.
- **MOVEDAO.properties**: Central repository of all Hive/SQL statements referenced by keys (e.g., `truncate.month`, `klm.summary`).
- **DTOs**: `KLMRecord`, `BundleSplitRecord`, `SplitRecord` are consumed by downstream reporting or billing services.
- **BillingUtils**: Supplies month‑end calculations for `insertMonth`.
- **Other components**: Likely invoked by a higher‑level service (e.g., `KLMReportJob`) that schedules month‑end processing.

# Operational Risks
- **Unclosed Connections**: Only statements are closed; the `Connection` object is never explicitly closed, risking connection leaks. *Mitigation*: Use try‑with‑resources or ensure `connection.close()` in finally.
- **Hard‑coded Offer ID** (`36378L`) and proposition string in split logic; changes require code redeployment. *Mitigation*: Externalize as config parameters.
- **SQL Injection via MessageFormat** in `getSplitRecord` (though currently only concatenates an empty string). Future modifications could introduce risk. *Mitigation*: Avoid string formatting; use prepared statements exclusively.
- **Large ResultSets**: Methods like `fetchInBundleList` load entire result sets into memory; may OOM on high‑volume zones. *Mitigation*: Stream processing or pagination.
- **Exception Masking**: Generic `Exception` thrown from `getInBundleRecords`; callers cannot differentiate DB vs logic errors. *Mitigation*: Refine exception hierarchy.

# Usage
```java
// Example: Process billing for April 2023
MediationDataAccess dao = new MediationDataAccess();
dao.insertMonth("2023-04");
dao.insertTrafficTableMonth();

Map<Integer, KLMRecord> summary = dao.getKLMSummary();
for (KLMRecord zoneRec : summary.values()) {
    dao.createCDRSubset(zoneRec);
    BundleSplitRecord split = new BundleSplitRecord();
    split.setTotalBundle(500 * 1024); // 500 GB in MB?
    split.setZone(zoneRec.getZone());
    dao.splitCDRRecords(split);
    // Pass `split` to downstream billing engine
}
```
Debugging: Set Log4j level to `DEBUG` for `com.tcl.move.dao.MediationDataAccess` to trace SQL statements and stack traces.

# Configuration
- **Environment Variables**: None directly referenced; Impala connection details are managed inside `JDBCConnection`.
- **Config Files**:
  - `MOVEDAO.properties` (classpath root) – contains all SQL templates referenced by keys used in this DAO.
  - `log4j.properties` – controls logging output.
- **Constants**: Offer‑specific literals (`36378L`, `"MOVE SIM AVKLM-4Z-EUS-$500"`) are hard‑coded; consider moving to `Constants.java`.

# Improvements
1. **Resource Management** – Refactor all DAO methods to use try‑with‑