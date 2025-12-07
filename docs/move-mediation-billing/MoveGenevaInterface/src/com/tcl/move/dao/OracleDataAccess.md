# Summary
`OracleDataAccess` provides all Oracle‑DB interactions required by the Move Geneva billing pipeline. It retrieves SIM and Usage product definitions, generates unique identifiers, inserts event records, updates reject tables, and stores RA (reconciliation) mappings. Each method manages its own JDBC connection, prepares statements from resource‑bundle SQL templates, handles batching, and translates low‑level errors into `DBDataException` / `DBConnectionException`.

# Key Components
- **Class `OracleDataAccess`**
  - `fetchSimProductView(Date, Date, Long)`: Reads SIM product view (`gen_sim_product`) filtered by billing period and optional SECS.
  - `fetchUsageProductView(Date, Date, Long)`: Reads Usage product view (`v_gen_usage_product` join `gen_country_prof`) with similar filters.
  - `getUniqueSerialNumber()`: Retrieves next value from `seq_gene_file_id` and pads to 6 digits.
  - `getUniqueCallId(int)`: Retrieves a list of sequential call IDs from `seq_gene_call_id`.
  - `insertInTable(List<EventFile>, Long, String)`: Truncates prior records for the period, then batch‑inserts SIM or Usage events into `sim_geneva` / `usage_geneva`.
  - `getEventTypeId(String)`: Looks up `event_type_id` from `VW_EVENT_TYPE`.
  - `insertSIMRejects(Date, Date, Map<String,Long>, Long, String)`: Clears prior rejects and inserts SIM‑reject rows into `geneva_reject`.
  - `insertUsageRejects(Date, Date, Map<String,Vector<Float>>, Long, String)`: Same for Usage rejects, handling DAT and SMS payload structures.
  - `insertRARecords(List<RARecord>, Long, Date, String)`: Updates mapping status and inserts reconciliation records into `gen_file_cdr_mapping`.
  - `getStackTrace(Exception)`: Utility to convert stack trace to string for logging.

# Data Flow
| Method | Input(s) | DB Interaction | Output / Side‑Effect |
|--------|----------|----------------|----------------------|
| `fetchSimProductView` | Billing period dates, optional SECS | SELECT from `gen_sim_product` (via `sim.view.fetch` template) | `List<SimProduct>` |
| `fetchUsageProductView` | Billing period dates, optional SECS | SELECT from `v_gen_usage_product` join `gen_country_prof` (via `usage.view.fetch`) | `List<UsageProduct>` |
| `getUniqueSerialNumber` | – | SELECT `seq_gene_file_id.nextval` (via `next.fileid`) | Padded 6‑digit string |
| `getUniqueCallId` | Count | SELECT `seq_gene_call_id.nextval` with CONNECT BY (via `next.callid`) | `List<String>` |
| `insertInTable` | List\<EventFile\>, optional SECS, event type flag | UPDATE existing rows (status='REPROCESED'), then batch INSERT into `sim_geneva` or `usage_geneva` (via `sim.update`, `usage.update`, `sim.insert`, `usage.insert`) | Rows persisted, transaction committed |
| `getEventTypeId` | Event type string | SELECT `event_type_id` from `VW_EVENT_TYPE` (via `event.fetch`) | String ID |
| `insertSIMRejects` | Date range, reject map, optional SECS, reject type | UPDATE `geneva_reject` status, then INSERT rows (via `reject.update`, `sim.reject.insert`) | Reject rows persisted |
| `insertUsageRejects` | Date range, reject map, optional SECS, reject type | Same pattern with `usage.reject.insert` | Reject rows persisted |
| `insertRARecords` | List\<RARecord\>, optional SECS, month date, file name | UPDATE `gen_file_cdr_mapping` status, then batch INSERT (via `ra.update`, `ra.insert`) | Mapping rows persisted |

All methods open a new JDBC connection via `JDBCConnection.getConnectionOracleMove()`, set `autoCommit(false)` where needed, and close resources in `finally`.

# Integrations
- **Resource Bundles**: SQL statements are externalized in `MOVEDAO.properties`. Keys referenced: `sim.view.fetch`, `usage.view.fetch`, `next.fileid`, `next.callid`, `sim.update`, `usage.update`, `sim.insert`, `usage.insert`, `event.fetch`, `reject.update`, `sim.reject.insert`, `usage.reject.insert`, `ra.update`, `ra.insert`.
- **DTOs**: `SimProduct`, `UsageProduct`, `EventFile`, `RARecord` are populated and consumed by downstream billing processors.
- **Constants**: `Constants` supplies separators, event type literals, and property names used for parsing propositions.
- **Utility**: `BillingUtils` for date rounding and number padding.
- **Logging**: Log4j (`logger`) records operation start, counts, and errors.
- **Exception Handling**: Custom exceptions (`DBDataException`, `DBConnectionException`) propagate to service layer.

# Operational Risks
- **Resource Leak**: Manual close in `finally`; any early return before `finally` could still leak if an exception occurs before resources are assigned. Mitigation: Use try‑with‑resources (Java 7+).
- **Batch Size Hard‑Coded**: Fixed at 3000 (or 1000 for RA). Large payloads may cause `ORA‑01795` or memory pressure. Mitigation: Make batch size configurable.
- **SQL Injection via MessageFormat**: `secsSubSql` concatenated into SQL template before preparation. Although only internal constants, any future change could introduce injection. Mitigation: Use separate prepared statements for optional clauses.
- **Date Rounding Dependency**: Relies on `BillingUtils.getRoundedDate`; mismatched rounding could cause off‑by‑one day errors. Ensure utility is deterministic across time zones.
- **Connection Auto‑Commit Reset**: Methods that set `autoCommit(false)` do not explicitly reset it; subsequent callers using the same `JDBCConnection` instance may inherit state. Mitigation: Ensure each method obtains a fresh connection or resets auto‑commit in `finally`.

# Usage
```java
// Example: fetch SIM products for billing month Jan 2025, all SECS
OracleDataAccess dao = new OracleDataAccess();
Date monthStart = new SimpleDateFormat("yyyy-MM-dd").parse("2025-01-01");
Date nextMonthStart = new SimpleDateFormat("yyyy-MM-dd").parse("2025-02-01");
List<SimProduct> sims = dao.fetchSimProductView(monthStart, nextMonthStart, null);

// Debug: enable Log4j DEBUG for com.tcl.move.dao.OracleDataAccess
// Verify SQL in MOVEDAO.properties matches expected schema.
```
Run unit tests with an embedded Oracle or a test schema; mock `JDBCConnection` if isolation is required.

# Configuration
- **