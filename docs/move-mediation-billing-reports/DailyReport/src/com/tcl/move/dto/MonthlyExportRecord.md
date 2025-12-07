# Summary
`MonthlyExportRecord` is a plain‑old‑Java‑Object (POJO) DTO used in the Move‑Mediation‑Billing‑Reports pipeline to carry aggregated monthly billing metrics (organization ID, CDR count, call attributes, usage totals, etc.) between parsers, aggregators, and CSV/Excel exporters. It contains only fields with standard getters and setters and no business logic.

# Key Components
- **Class `MonthlyExportRecord`**
  - Private member variables: `orgNo`, `cdrCount`, `callType`, `tadigCode`, `sponsor`, `companyName`, `accountNumber`, `customerNumber`, `imsi`, `direction`, `mappedImsi`, `duration`, `billableDuration`, `bytes`, `retailDuration`.
  - Public accessor methods (`getX`, `setX`) for each field.

# Data Flow
- **Inputs:** Populated by upstream components (e.g., CDR parsers, aggregation services) that map raw call detail records to the DTO fields.
- **Outputs:** Consumed by downstream exporters (CSV, Excel, database persistence layers) that read the getters to generate monthly reports.
- **Side Effects:** None; the class is immutable only by convention (no validation or state changes beyond setter calls).
- **External Services/DBs/Queues:** Not referenced directly; the DTO is passed through in‑memory collections or messaging payloads.

# Integrations
- **Export Pipelines:** `MonthlyExportRecord` instances are created in aggregation modules (e.g., `MonthlyReportAggregator`) and placed into collections passed to `MonthlyReportWriter` or similar exporters.
- **Serialization:** May be serialized by frameworks (Jackson, Gson) for JSON transport or by Java serialization for messaging queues.
- **Mapping Layers:** Likely mapped from database result sets or Hadoop MapReduce outputs via reflection or manual mapping utilities.

# Operational Risks
- **Data Type Mismatch:** Use of `float` for monetary‑related durations may cause precision loss; consider `double` or `BigDecimal`.
- **Missing Validation:** No null/format checks; malformed input can propagate to reports.
- **Schema Drift:** Adding/removing fields requires coordinated changes across all pipeline components; lack of versioning can cause runtime `NoSuchMethodError`.

# Usage
```java
// Example creation in aggregation step
MonthlyExportRecord rec = new MonthlyExportRecord();
rec.setOrgNo(12345L);
rec.setCdrCount(2500L);
rec.setCallType("VOICE");
rec.setDuration(3600.5f);
// ... set remaining fields ...

// Pass to exporter
monthlyReportWriter.write(rec);
```
Debug by inspecting object state via IDE watch windows or logging:
```java
log.debug("MonthlyExportRecord: {}", rec);
```

# Configuration
No environment variables or external configuration files are referenced directly by this class. Configuration affecting its usage resides in upstream aggregation or downstream export modules (e.g., property files defining CSV column order).

# Improvements
1. **Replace `float` with `double` or `BigDecimal`** for duration and byte fields to avoid precision loss in financial calculations.
2. **Add field validation** (e.g., non‑negative numeric values, non‑null mandatory strings) either in setters or via a builder pattern to ensure data integrity before export.