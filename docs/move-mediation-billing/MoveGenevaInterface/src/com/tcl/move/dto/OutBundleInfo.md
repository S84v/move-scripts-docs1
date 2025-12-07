# Summary
`OutBundleInfo` is a POJO that encapsulates aggregated out‑of‑bundle usage records for a single SIM/transaction. It stores event metadata (date, org, SIM, country, MCC, direction, destination type, TADIG, sponsor), quantitative metrics (usage, CDR count), and a mapping of source file names to per‑file CDR count and usage. The object is populated by the DAO layer (`UsageDataAccess`) and consumed by the Move‑Geneva billing engine to compute out‑of‑bundle charges.

# Key Components
- **Class `OutBundleInfo`**
  - Private fields: `eventDate`, `orgNo`, `usage`, `country`, `mcc`, `cdrCount`, `sim`, `direction`, `destinationType`, `tadig`, `sponsor`, `fileCDRCountMapping`.
  - Getters/Setters for each field.
  - `addFileCDRCountMapping(String filename, float cdrCount, float usage)`: convenience method to insert a two‑element `Vector<Float>` (CDR count, usage) into `fileCDRCountMapping`.
  - `toString()`: human‑readable representation for logging/debugging.

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| DAO (`UsageDataAccess.createOutBundleInfo`) | Raw Hive CDR rows (filtered by product, date range) | Aggregates per‑SIM metrics, populates fields, builds `fileCDRCountMapping` | `OutBundleInfo` instance | None (pure data object) |
| Billing Engine | `OutBundleInfo` objects (list) | Calculates out‑of‑bundle charges, writes to event backup table | Charge records, audit logs | DB writes (event table) |
| Monitoring/Logging | `OutBundleInfo.toString()` | Serialized for log files | Log entries | Log storage |

# Integrations
- **`UsageDataAccess`**: constructs and returns `OutBundleInfo` objects after executing Hive queries.
- **`BundleSplitRecord`**: may reference `OutBundleInfo` when assembling final split results.
- **Move‑Geneva billing pipeline**: consumes collections of `OutBundleInfo` to generate billing events.
- **Event backup persistence**: fields map directly to columns in the *event* backup table.

# Operational Risks
- **Null field usage**: getters may return `null` leading to NPEs downstream. *Mitigation*: enforce non‑null defaults or validation before use.
- **Thread‑unsafe mutable collections**: `fileCDRCountMapping` (HashMap) and inner `Vector<Float>` are not synchronized. *Mitigation*: confine instance to a single thread or replace with concurrent/immutable structures.
- **Precision loss**: usage stored as `Float`; cumulative sums may suffer rounding errors. *Mitigation*: switch to `BigDecimal` for monetary‑grade calculations.
- **Legacy `Vector`**: unnecessary synchronization overhead. *Mitigation*: replace with `ArrayList<Float>`.

# Usage
```java
// Example: create and populate an OutBundleInfo for debugging
OutBundleInfo outInfo = new OutBundleInfo();
outInfo.setEventDate(new Date());
outInfo.setOrgNo("ORG123");
outInfo.setSim("SIM001");
outInfo.setCountry("US");
outInfo.setMcc("310");
outInfo.setDirection("OUTBOUND");
outInfo.setDestinationType("MOBILE");
outInfo.setTadig("TADIG01");
outInfo.setSponsor("SPONSORX");
outInfo.setUsage(125.5f);
outInfo.setCdrCount(42L);

// Add per‑file aggregation
outInfo.addFileCDRCountMapping("fileA.csv", 30f, 85.0f);
outInfo.addFileCDRCountMapping("fileB.csv", 12f, 40.5f);

// Log the object
System.out.println(outInfo);
```
Debugging tip: inspect `outInfo.getFileCDRCountMapping()` to verify per‑file counts.

# configuration
No environment variables or external configuration files are referenced directly by this class. Configuration is handled upstream (DAO query parameters, DB connection settings).

# Improvements
1. **Replace mutable legacy collections**: use `Map<String, List<BigDecimal>>` with `ArrayList` and `BigDecimal` for precise usage aggregation; make the map immutable after construction.
2. **Add validation builder**: implement a static factory or builder that validates mandatory fields (e.g., `eventDate`, `orgNo`, `sim`) and prevents creation of partially populated instances.