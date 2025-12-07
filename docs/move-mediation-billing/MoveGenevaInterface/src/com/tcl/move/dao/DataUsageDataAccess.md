# Summary
`DataUsageDataAccess` is a DAO class that retrieves data‑traffic usage and CDR counts from Hive tables for the Move‑mediation billing pipeline. It provides methods to fetch aggregated usage per customer/proposition, per zone, per sponsor, for multiple propositions, and file‑level usage records. Results are returned as maps, vectors, or POJOs for downstream invoicing and reporting.

# Key Components
- **Class `DataUsageDataAccess`** – Extends `UsageDataAccess`; central DAO for data‑traffic queries.  
- **Logger `logger`** – Log4j logger for audit and error tracing.  
- **ResourceBundle `daoBundle`** – Loads SQL statements from `MOVEDAO.properties`.  
- **`JDBCConnection jdbcConn`** – Provides Hive connections.  
- **Method `fetchDataTrafficInfo()`** – Returns `Map<String,Vector<Float>>` keyed by `secsId|proposition|usageFor` with combined/domestic/international usage (MB) and CDR counts.  
- **Method `getDataUsageForProduct(Long secsCode, String proposition, List<String> propList, int destination, String commercialCodeType)`** – Returns `Vector<Float>` for a specific zone.  
- **Method `getDataUsageForProduct(Long secsCode, String proposition, List<String> propList, String sponsor, String commercialCodeType)`** – Returns `Vector<Float>` for a sponsor filter.  
- **Method `getDataUsageForMultiProp(Long secsCode, List<String> propList, String commercialCodeType)`** – Returns `Vector<Float>` aggregated across multiple propositions.  
- **Method `getFileLevelCountUsage(UsageProduct product, String destinationtype)`** – Returns `List<RARecord>` with per‑file usage and CDR counts.  
- **Helper `getStackTrace(Exception)`** – Converts exception stack trace to string for logging.  

# Data Flow
| Step | Input | Process | Output | Side Effects |
|------|-------|---------|--------|--------------|
| 1 | Hive connection via `jdbcConn.getConnectionHive()` | Execute SQL from `MOVEDAO.properties` (parameterised with `MessageFormat`) | `ResultSet` rows | DB read |
| 2 | `ResultSet` columns (`combined_dat`, `domestic_dat`, `international_dat`, `combined_cdr`, …) | Convert bytes to MB (`/ (1024*1024)`) and populate `Vector<Float>` or `RARecord` | `Map<String,Vector<Float>>`, `Vector<Float>`, `List<RARecord>` | None |
| 3 | Exceptions | Log stack trace, wrap in `DBDataException` or `DBConnectionException` | Thrown exception | None |
| 4 | Method return | Consumed by service layer for invoicing/report generation | – | – |

External services: Hive metastore (SQL execution), Log4j for logging.

# Integrations
- **`MOVEDAO.properties`** – Supplies all SQL templates referenced (`data.fetch`, `data.zone.fetch`, `data.sponsor.fetch`, `data.multprop.fetch`, `data.filewise.fetch`).  
- **`UsageProduct` & `RARecord` DTOs** – Used by higher‑level business logic to build invoices and reports.  
- **`BillingUtils.getQuestionMarks(int)`** – Generates placeholder strings for `IN` clause parameters.  
- **Superclass `UsageDataAccess`** – May provide shared utilities (not shown).  
- **`JDBCConnection`** – Centralised connection factory for Hive; used across DAO classes.  

# Operational Risks
- **SQL injection risk** – Dynamic SQL built with `MessageFormat`; mitigated by using prepared statements and controlled placeholders.  
- **Resource leakage** – Connections, statements, and result sets are closed in `finally`; any failure in close throws `DBConnectionException` which could propagate unexpectedly.  
- **Memory pressure** – Large result sets loaded into in‑memory `Vector`/`Map`; consider streaming or pagination for very large customers.  
- **Hard‑coded unit conversion** – Fixed divisor `(1024*1024)` assumes bytes; any schema change would require code update.  
- **Hive connectivity failures** – Transient network issues may cause `DBConnectionException`; implement retry logic at service layer.

# Usage
```java
// Example: fetch aggregated data traffic for all customers
DataUsageDataAccess dao = new DataUsageDataAccess();
Map<String, Vector<Float>> traffic = dao.fetchDataTrafficInfo();

// Example: fetch zone‑specific usage
Vector<Float> zoneUsage = dao.getDataUsageForProduct(
        123456L,               // secsCode
        "PROD_A",              // proposition
        null,                  // propList (null if single)
        2,                     // destination zone id
        "Subscription");       // commercialCodeType
```
For debugging, enable Log4j `DEBUG` level for `com.tcl.move.dao.DataUsageDataAccess` to view generated SQL and parameter values.

# Configuration
- **`MOVEDAO.properties`** – Must contain keys: `data.fetch`, `data.zone.fetch`, `data.sponsor.fetch`, `data.multprop.fetch`, `data.filewise.fetch`.  
- **`log4j.properties`** – Controls logging output; ensure `logger` level INFO or DEBUG as needed.  
- **Environment** – Hive server reachable; JDBC URL and credentials configured inside `JDBCConnection`.  

# Improvements
1. **Introduce try‑with‑resources** for `Connection`, `Statement`, and `ResultSet` to guarantee closure and simplify exception handling.  
2. **Replace `Vector` with immutable collections** (e.g., `List<Float>`) or a dedicated DTO to improve type safety and reduce legacy synchronization overhead.