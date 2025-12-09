# Summary
`BundleSplitRecord` is a plain‑old‑Java‑Object (POJO) that encapsulates the result of a usage‑bundle split operation for a given zone. It stores the total bundle allowance, the zone identifier, and two collections of `KLMRecord` objects representing in‑bundle and out‑of‑bundle call‑detail‑records (CDRs). The class is used by the Mediation layer to transport split data between DAO processing, business logic, and reporting components of the MOVE‑Mediation‑Ingest pipeline.

# Key Components
- **Class `BundleSplitRecord`**
  - `float totalBundle` – aggregate bundle quota for the zone.
  - `int zone` – numeric identifier of the geographic or tariff zone.
  - `List<KLMRecord> inbRecords` – mutable list of CDRs that fall within the bundle.
  - `List<KLMRecord> outbRecords` – mutable list of CDRs that exceed the bundle.
  - Getter/Setter methods for each field, providing JavaBean compliance.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| DAO (`MediationDataAccess`) | Raw CDR rows from Hive/Impala | Group by zone, compute bundle consumption, allocate each `KLMRecord` to in‑bundle or out‑of‑bundle list | `BundleSplitRecord` instance per zone | None (pure data container) |
| Business Logic | `BundleSplitRecord` objects | Apply pricing, generate charge records | Enriched DTOs or persisted charge rows | May trigger DB writes via DAO |
| Reporting (`KLMReport`) | Collection of `BundleSplitRecord` | Aggregate for CSV/Parquet export | Report files, Hive tables | File system I/O, Hive inserts |

# Integrations
- **`KLMRecord`**: referenced type; defined elsewhere in `com.tcl.move.dto`. Represents a single usage event.
- **`MediationDataAccess`**: creates and populates `BundleSplitRecord` during bundle‑splitting queries.
- **`Constants`**: may provide zone identifiers or bundle‑type keys used when populating the POJO.
- **Logging**: external Log4j configuration (via `JDBCConnection` or other classes) logs lifecycle events; `BundleSplitRecord` itself is not logged but may be serialized for debug.

# Operational Risks
- **Mutable Lists**: Direct exposure of internal `ArrayList` allows callers to modify collections without defensive copying, risking data corruption across threads.
  - *Mitigation*: Return unmodifiable views or deep‑copy in getters.
- **Floating‑Point Precision**: `float totalBundle` may introduce rounding errors in monetary calculations.
  - *Mitigation*: Switch to `java.math.BigDecimal` for monetary values.
- **Null Handling**: No null checks in setters; callers may assign `null` lists leading to `NullPointerException` downstream.
  - *Mitigation*: Validate inputs or initialize with empty collections in setters.

# Usage
```java
// Example: constructing a BundleSplitRecord for zone 3
BundleSplitRecord split = new BundleSplitRecord();
split.setZone(3);
split.setTotalBundle(500.0f);

List<KLMRecord> inBundle = fetchInBundleRecords(zone);
List<KLMRecord> outBundle = fetchOutBundleRecords(zone);

split.setInbRecords(inBundle);
split.setOutbRecords(outBundle);

// Pass to downstream processor
billingProcessor.process(split);
```
*Debug*: Use `System.out.println(split.getZone() + ":" + split.getTotalBundle());` or log via Log4j.

# Configuration
No direct configuration; relies on:
- Classpath inclusion of `com.tcl.move.dto.KLMRecord`.
- Environment variables for DB connections (`JDBCConnection`) used upstream.
- Logging configuration (`log4j.properties`) for any debug statements.

# Improvements
1. **Encapsulation** – Replace mutable list exposure with immutable collections or defensive copies to ensure thread safety.
2. **Precision** – Change `totalBundle` from `float` to `BigDecimal` and adjust getters/setters accordingly.