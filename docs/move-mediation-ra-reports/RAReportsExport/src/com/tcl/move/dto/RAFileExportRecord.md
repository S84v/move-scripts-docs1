# Summary
`RAFileExportRecord` is a POJO that aggregates per‑file, per‑customer reconciliation metrics for Geneva‑level Revenue Assurance (RA) reports (Monthly, Customer‑Monthly, Reject). It stores raw counts, generated counts, rejected counts and derived differences for CDR volumes and usage (data, SMS, A2P, voice), together with file metadata and customer identifiers. The `calculateDifference()` method computes the delta fields from the other metrics.

# Key Components
- **Class `RAFileExportRecord`**
  - Private fields: file metadata (`fileName`, `processedDate`, `callMonth`), CDR counters (`*_CDRMed`, `*_CDRRej`, `*_CDRGen`, `*_CDRDiff`), usage amounts (`*_UsageMed`, `*_UsageRej`, `*_UsageGen`, `*_UsageDiff`), customer info (`custNum`, `custName`), business context (`proposition`, `reason`).
  - Public JavaBean getters/setters for each field.
  - `calculateDifference()` – derives all `*Diff` fields by subtracting the sum of rejected and generated values from the mediated total.

# Data Flow
- **Inputs**: Values are populated by upstream ETL/aggregation components that read raw mediation logs and compute per‑file statistics.
- **Outputs**: Instances are consumed by export modules that serialize the object (e.g., to CSV) for downstream reporting or archival storage.
- **Side Effects**: None; the class is immutable after `calculateDifference()` unless setters are invoked.
- **External Services/DBs/Queues**: None directly referenced; used as a data carrier between services.

# Integrations
- Likely instantiated in report‑generation services within `move-mediation-ra-reports` (e.g., classes that iterate over processed files and build collections of `RAFileExportRecord`).
- Serialized by CSV writers or passed to messaging pipelines that deliver RA reports to downstream analytics or billing systems.
- May be mapped from database query results when historical reconciliation data is re‑processed.

# Operational Risks
- **Risk**: `calculateDifference()` not invoked, leaving diff fields stale.  
  **Mitigation**: Enforce call after all setters or compute lazily in getters.
- **Risk**: Numeric overflow for large CDR counts (long is sufficient but aggregation bugs could exceed range).  
  **Mitigation**: Validate input ranges; monitor for abnormal spikes.
- **Risk**: Inconsistent units between mediated, rejected, and generated metrics leading to negative diffs.  
  **Mitigation**: Add sanity checks; log warnings when diffs are negative.

# Usage
```java
RAFileExportRecord rec = new RAFileExportRecord();
rec.setFileName("file123.csv");
rec.setProcessedDate("2025-12-01");
rec.setCallMonth("202512");
rec.setDataCDRMed(150000L);
rec.setDataCDRRej(5000L);
rec.setDataCDRGen(140000L);
// set other counters...
rec.calculateDifference();   // compute diff fields

// Example serialization (CSV)
String csvLine = String.join(",",
    rec.getFileName(),
    rec.getProcessedDate(),
    rec.getCallMonth(),
    String.valueOf(rec.getDataCDRMed()),
    String.valueOf(rec.getDataCDRRej()),
    String.valueOf(rec.getDataCDRGen()),
    String.valueOf(rec.getDataCDRDiff()),
    // ... other fields in required order
);
System.out.println(csvLine);
```

# Configuration
No environment variables or external configuration files are referenced within this class. Configuration is handled by the components that instantiate and populate the object (e.g., property files defining CSV column order).

# Improvements
1. **Encapsulation**: Make fields `final` and provide a builder pattern; remove mutable setters to guarantee immutability after construction.
2. **Automatic Diff Calculation**: Compute diff values lazily in their getters or within a dedicated factory method to avoid forgetting to call `calculateDifference()`.