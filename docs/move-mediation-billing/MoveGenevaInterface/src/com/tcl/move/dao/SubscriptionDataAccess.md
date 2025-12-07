# Summary
`SubscriptionDataAccess` is a DAO that queries Hadoop (Hive) tables to retrieve activation, active, tolling, and addon SIM counts for billing cycles. It also inserts month‑billing metadata. Each method opens its own Hive connection, executes SQL from the `MOVEDAO` resource bundle, maps result sets to Java collections, logs activity, and translates SQL exceptions into `DBDataException` or `DBConnectionException`.

# Key Components
- **Class `SubscriptionDataAccess`**
  - `fetchActivations()` – returns `Map<Long,Long>` of SECS → activation count.
  - `fetchActivationsCOWise()` – returns `Map<String,Long>` keyed by `secsId|commercialOffer`.
  - `fetchActiveSIMs()` – returns active SIM counts per SECS/commercialOffer.
  - `fetchTollingSIMs()` – returns tolling SIM counts per SECS/commercialOffer.
  - `fetchAddonSIMs(Date startDate, Date endDate)` – returns addon SIM counts per SECS/commercialOffer for a month.
  - `fetchSimCountForProduct(Date startDate, Date endDate, Long secsCode, List<String> propList, String countType)` – generic count (ACTIVATION/ACTIVE/TOLLING/ADDON) for a set of propositions.
  - `insertMonth(String monthYear)` – truncates `mnaas.month_billing` and inserts a row for the supplied month.
  - `getStackTrace(Exception e)` – utility to convert stack trace to `String`.

# Data Flow
| Method | Input(s) | DB Interaction | Output | Side Effects |
|--------|----------|----------------|--------|--------------|
| `fetchActivations` | none (uses static SQL) | Hive `billing_sim_activations` | `Map<Long,Long>` | None |
| `fetchActivationsCOWise` | none | Hive `billing_sim_activations` (group by commercialoffer) | `Map<String,Long>` | None |
| `fetchActiveSIMs` | none | Hive `billing_active_sims` | `Map<String,Long>` | None |
| `fetchTollingSIMs` | none | Hive `billing_tolling_sims` | `Map<String,Long>` | None |
| `fetchAddonSIMs` | `Date startDate, Date endDate` (used to format month) | Hive `addon_subs_aggr` (CTE) | `Map<String,Long>` | None |
| `fetchSimCountForProduct` | `Date startDate, Date endDate, Long secsCode, List<String> propList, String countType` | Hive tables (`billing_sim_activations`, `billing_active_sims`, `billing_tolling_sims`, `addon_subs_aggr`) | `Long` (or `null` if no rows) | None |
| `insertMonth` | `String monthYear` | Hive `month_billing` (TRUNCATE + INSERT) | void | Table truncated & row inserted |

All methods acquire a new Hive connection via `JDBCConnection.getConnectionHive()`, close `ResultSet`, `Statement/PreparedStatement`, and `Connection` in a `finally` block.

# Integrations
- **Resource Bundle**: `MOVEDAO.properties` supplies all SQL statements referenced by keys (`activations.fetch`, `activations.plan.fetch`, etc.).
- **Utility Classes**: `BillingUtils` for date formatting and question‑mark generation; `Constants` for separators and count‑type strings.
- **Exception Types**: `DBDataException`, `DBConnectionException` propagate errors to higher‑level services (e.g., billing orchestration jobs).
- **Logging**: Log4j logger records operation start, counts, and error stack traces.
- **External Consumers**: Likely invoked by billing batch jobs that aggregate SIM counts per billing cycle.

# Operational Risks
- **Connection Leak**: If an exception occurs before the `finally` block, resources may remain open. Mitigation: use try‑with‑resources (Java 7+).
- **SQL Injection via `propList`**: Question marks are generated safely, but misuse of `MessageFormat` could expose placeholders. Ensure `propList` is validated.
- **Hard‑coded Date Formats**: Locale‑dependent `SimpleDateFormat` may cause parsing errors in non‑English locales. Use `DateTimeFormatter` with explicit locale.
- **Batch Performance**: Each count query opens a new connection; high concurrency could exhaust Hive connection pool. Consider connection pooling or reusing connections.
- **Error Masking**: Stack trace is logged but only the message is propagated; downstream may lose root cause. Propagate original exception as cause.

# Usage
```java
// Example: retrieve activation counts for a billing run
SubscriptionDataAccess dao = new SubscriptionDataAccess();
try {
    Map<Long, Long> activations = dao.fetchActivations();
    // process map...
} catch (DBDataException | DBConnectionException e) {
    // handle error
}
```
For debugging, enable Log4j DEBUG level for `com.tcl.move.dao.SubscriptionDataAccess` to see generated SQL and parameter values.

# Configuration
- **Environment Variables**: None directly referenced; Hive JDBC URL, credentials, etc., are encapsulated in `JDBCConnection`.
- **Config Files**:
  - `MOVEDAO.properties` – contains all SQL templates referenced by keys used in this class.
  - Log4j configuration (`log4j.properties` or XML) – controls logging output.

# Improvements
1. **Refactor resource handling** – replace manual `close()` calls with try‑with‑resources to guarantee deterministic cleanup and reduce boilerplate.
2. **Consolidate common query logic** – extract a generic method that accepts SQL key, parameters, and a row‑mapper to eliminate duplicated connection/statement/resultset patterns across the fetch methods.