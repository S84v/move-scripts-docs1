# Summary
`SplitRecord` is a POJO that encapsulates the boundary information for a usage‑bundle split in the MOVE‑Mediation‑Ingest pipeline. It stores the CDR numbers that delimit the in‑bundle and out‑of‑bundle segments, a flag indicating whether a split occurred, and, for split cases, the split CDR identifier, country, call date, and the usage (KB/SMS‑minutes) allocated to each side of the split. The object is passed between DAO, business‑logic, and reporting layers to drive accurate bundle accounting for the **MOVE SIM AVKLM‑4Z‑EUS‑$500** offer.

# Key Components
- **Fields**
  - `Long inbEndCdrNum` – last CDR number belonging to the in‑bundle segment.  
  - `Long outbStartCdrNum` – first CDR number belonging to the out‑of‑bundle segment.  
  - `boolean isSplit` – true if a split point exists within the usage window.  
  - `Long splitCdrNum` – CDR number at which the split occurs (only when `isSplit` is true).  
  - `String splitCountry` – ISO country code of the split CDR.  
  - `String splitCallDate` – call date (YYYY‑MM‑DD) of the split CDR.  
  - `float inUsageKbSMSMin` – aggregated in‑bundle usage (KB for data, SMS‑minutes for voice).  
  - `float outUsageKbSMSMin` – aggregated out‑of‑bundle usage (KB/SMS‑minutes).  

- **Accessors / Mutators**
  - Standard getters and setters for each field, following JavaBean conventions.  

- **`isSplit()`**
  - Boolean accessor that returns the `isSplit` flag.  

- **`toString()`**
  - Human‑readable representation used for logging/debugging; concatenates all field values.

# Data Flow
| Stage | Input | Output | Side Effects |
|-------|-------|--------|--------------|
| DAO (`MediationDataAccess`) | Raw CDR rows from Hive/Impala | `SplitRecord` instances populated per zone/offer | None (pure data object) |
| Business Logic | `List<SplitRecord>` | Bundled/filtered usage aggregates for billing | May trigger downstream DB writes (billing tables) |
| Reporting | `SplitRecord` (via DTO conversion) | CSV/JSON reports, UI tables | File I/O or REST response generation |

No external services are invoked directly by `SplitRecord`; it is a passive data carrier.

# Integrations
- **`MediationDataAccess`** – constructs `SplitRecord` while iterating over CDR result sets; sets fields based on computed split boundaries.
- **`BundleSplitRecord`** – aggregates multiple `SplitRecord` objects per zone, exposing collections of in‑bundle and out‑of‑bundle `KLMRecord`s.
- **`KLMRecord`** – consumes usage values (`inUsageKbSMSMin`, `outUsageKbSMSMin`) from `SplitRecord` to compute overage and billing metrics.
- **Logging framework** – `toString()` is used in debug logs throughout the mediation pipeline.

# Operational Risks
- **Incorrect CDR number boundaries** – mis‑populated `inbEndCdrNum` / `outbStartCdrNum` can cause duplicate or missing usage. *Mitigation*: unit‑test DAO split logic with edge‑case CDR sequences.
- **Floating‑point rounding** – usage fields are `float`; cumulative rounding may affect billing precision. *Mitigation*: switch to `BigDecimal` or store raw bytes/minutes and convert at reporting time.
- **Null field propagation** – setters accept nulls; downstream code may assume non‑null values, leading to NPEs. *Mitigation*: enforce non‑null contracts in DAO or add validation in `SplitRecord`.
- **Thread‑safety** – mutable POJO shared across threads without copying can cause data races. *Mitigation*: treat `SplitRecord` as thread‑local or make it immutable.

# Usage
```java
// Example: creating a SplitRecord for a CDR sequence
SplitRecord split = new SplitRecord();
split.setInbEndCdrNum(10234L);
split.setOutbStartCdrNum(10235L);
split.setSplit(true);
split.setSplitCdrNum(10234L);
split.setSplitCountry("DE");
split.setSplitCallDate("2024-11-15");
split.setInUsageKbSMSMin(850.5f);
split.setOutUsageKbSMSMin(120.0f);

System.out.println(split);   // logs the populated state
```
Debugging tip: place a breakpoint after DAO populates the object and inspect `toString()` output.

# Configuration
`SplitRecord` itself does not read configuration. It relies on:
- **Hive/Impala connection settings** (defined elsewhere) that provide raw CDR data.
- **Application properties** controlling bundle thresholds; these affect the values set into the POJO but are not referenced directly.

# Improvements
1. **Make immutable** – replace setters with a constructor that accepts all fields and declare fields `final`. This eliminates accidental mutation and improves thread safety.
2. **Replace `float` with `BigDecimal`** – ensures monetary‑grade precision for usage calculations and avoids cumulative rounding errors.