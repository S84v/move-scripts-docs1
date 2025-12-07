# Summary
`UsageDataAccess` is a DAO that interacts with Hive‑based mediation tables to (1) populate a filtered CDR subset for a given product, (2) determine the split point where a usage bundle is exhausted, (3) compute in‑bundle counts/usage, and (4) aggregate out‑of‑bundle records. It is used by the Move‑Geneva billing engine to derive per‑product usage metrics for invoicing.

# Key Components
- **class `UsageDataAccess`**
  - `createCDRSubset(UsageProduct, String, String)`: Truncates the temp table, builds a dynamic INSERT‑SELECT statement, and loads filtered CDRs.
  - `splitCDRRecords(BundleSplitRecord)`: Orchestrates bundle‑cross detection, invokes helper methods, and populates `BundleSplitRecord`.
  - `fetchInBundleCountAndUsage(BundleSplitRecord, SplitRecord)`: Executes two aggregated queries (regular and RA) to fill in‑bundle counts, usage, and per‑record RA data.
  - `getSplitRecord(String, double, List<String>)`: Scans CDRs for a specific day to locate the exact CDR where the bundle limit is crossed; handles split‑CDR logic.
  - `fetchOutBundleRecords(SplitRecord, List<String>, boolean)`: Builds and runs a Hive query (or union of queries) to aggregate out‑of‑bundle usage, optionally at SIM level.
  - `getStackTrace(Exception)`: Utility to convert stack trace to string for logging.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | `UsageProduct`, `destinationType`, `direction` | Dynamic SQL built from `MOVEDAO` resource bundle; parameters bound; temp table `billing_traffic_filtered_cdr` truncated then populated. | Populated filtered CDR table (Hive). |
| 2 | `BundleSplitRecord` (contains product bundle size, country list, etc.) | Query daily aggregated usage (`date.wise.cdr.fetch`) → iterate to find split point. | Updated `BundleSplitRecord` with split info, in‑bundle counts, and list of `OutBundleInfo`. |
| 3 | Split point (`SplitRecord`) | Two aggregated queries (`in.count.usage.fetch`, `in.count.usage.ra.fetch`) limited by `cdrnum <= splitEnd`. | In‑bundle counts/usage stored in `BundleSplitRecord`; RA records stored in map. |
| 4 | Split point, country list, SIM‑aggregation flag | Build one of four Hive CTE queries (`out.usage.fetch*`) with optional UNION for split‑CDR. | List of `OutBundleInfo` (date, direction, country, usage, CDR count, file‑level breakdown). |
| 5 | Exceptions | Stack trace captured via `getStackTrace`. | Logged error details. |

External services:
- Hive via `JDBCConnection.getConnectionHive()`.
- Logging via Log4j.
- Configuration via `ResourceBundle MOVEDAO`.

# Integrations
- **Upstream**: Called by service layer (e.g., `UsageService`) that prepares `UsageProduct` and `BundleSplitRecord` based on subscriber data.
- **Downstream**: Returns populated DTOs (`BundleSplitRecord`, `OutBundleInfo`) to billing calculation components that generate invoices.
- **Shared utilities**: `BillingUtils` for placeholder generation, `Constants` for domain literals, `JDBCConnection` for connection pooling.

# Operational Risks
- **SQL injection via dynamic MessageFormat**: Parameters are bound, but placeholder strings (`destTypeSubStr`, etc.) are concatenated; malformed bundle types could produce invalid SQL. *Mitigation*: Validate bundle type against whitelist before building query.
- **Resource leakage**: ResultSet/Statement/Connection closed in finally, but `resultSet` is never used in `createCDRSubset`; harmless but unnecessary. Ensure all resources are closed even on early return.
- **Hive query performance**: Large CDR volumes; use of `ROW_NUMBER()` and multiple CTEs may cause long runtimes. *Mitigation*: Add appropriate partitioning/filters, monitor query plans.
- **Precision loss**: Usage conversions (KB ↔ MB) use float; rounding errors may affect billing. *Mitigation*: Switch to `BigDecimal` for monetary‑critical calculations.
- **Hard‑coded resource bundle keys**: Missing keys cause `MissingResourceException`. *Mitigation*: Validate bundle at startup.

# Usage
```java
// Example from a unit test or service method
UsageProduct product = new UsageProduct();
product.setSecsCode(12345L);
product.setCallType("Voice");
product.setBundleType(Constants.SEPARATE);
product.setProposition("Standard");
product.setPropList(Arrays.asList("PROP1","PROP2"));
product.setPropDestination(100);
product.setCommercialCodeType(Constants.ADDON_CODE);

UsageDataAccess dao = new UsageDataAccess();
dao.createCDRSubset(product, "DOMESTIC", "MO");

BundleSplitRecord splitRec = new BundleSplitRecord();
splitRec.setTotalBundle(5000f); // MB
splitRec.setInCountryList(Arrays.asList("US","CA"));
splitRec.setSimLevelAggrNeeded(true);

BundleSplitRecord result = dao.splitCDRRecords(splitRec);
// result now contains in‑bundle counts and out‑of‑bundle list
```
To debug, enable Log4j `DEBUG` for `com.tcl.move.dao.UsageDataAccess` and inspect generated SQL in the logs.

# configuration
- **Resource bundle** `MOVEDAO.properties` (classpath) containing:
  - `cdr.subset.truncate`
  - `cdr.subset.insert`
  - `date.wise.cdr.fetch`
  - `in.count.usage.fetch`
  - `in.count.usage.ra.fetch`
  - `out.usage.fetch`, `out.usage.fetch.all`, `out.usage.fetch.country`, `out.usage.fetch.split`
- **JDBCConnection** expects Hive JDBC URL, username, password (environment‑specific, typically via system properties or a separate config file).
- **Log4j** configuration for logger `com.tcl.move.dao.UsageDataAccess`.

# Improvements
1. **Refactor dynamic SQL construction** – replace `MessageFormat` placeholders with a query builder (e.g., jOOQ or plain StringBuilder) to avoid concatenating SQL fragments and to enforce parameterization.
2. **Replace float arithmetic with `BigDecimal`** for all usage calculations and expose a utility method for unit conversion to guarantee billing‑grade precision.