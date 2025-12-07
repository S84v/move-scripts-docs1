# Summary
`RARecord` is a plain‑old‑Java‑Object (POJO) that represents a single usage record required for the *filename → call‑ID* mapping in the Move‑Geneva billing pipeline. It carries the source file name, call type, usage amount, CDR count, call identifier and an optional SECS identifier. The object is populated by the data‑access layer and consumed downstream by the billing engine to correlate raw CDR files with logical call records.

# Key Components
- **Class `RARecord`**
  - **Fields**
    - `String fileName` – source CDR file name.
    - `String callType` – classification of the call (e.g., voice, SMS, data).
    - `Float usage` – usage quantity (e.g., minutes, MB).
    - `Long cdrCount` – number of CDR entries represented.
    - `String callId` – logical call identifier used by Geneva.
    - `Long secsId` – optional SECS (Security) identifier.
  - **Accessors/Mutators**
    - Standard getters and setters for each field.
    - Duplicate `setCdrCount(Long)` overload (legacy support).
  - **Purpose**
    - Acts as a transport object between DAO (`UsageDataAccess`) and billing services.

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| DAO layer (`UsageDataAccess`) | Raw CDR rows from DB or flat files | Populate `RARecord` instances per row | `RARecord` objects in memory | None (pure data mapping) |
| Billing engine (`MoveGenevaProcessor`) | Collection of `RARecord` | Aggregate, correlate with other DTOs (e.g., `BundleSplitRecord`) | Billing calculations, persisted events | May trigger downstream persistence or messaging |
| Optional utilities | `RARecord` list | Serialize to CSV/JSON for audit | Audit files | File I/O only |

No external services are invoked directly from this class; it is a passive data holder.

# Integrations
- **`BundleSplitRecord`**, **`OutBundleInfo`**, **`EventFile`** – consume `RARecord` collections to compute in‑bundle/out‑of‑bundle usage.
- **DAO (`UsageDataAccess`)** – creates and fills `RARecord` objects from database queries or file parsers.
- **Billing Engine (`MoveGenevaProcessor` / `GenevaBillingService`)** – receives `RARecord` lists for charge computation.
- **Audit/Reporting modules** – may serialize `RARecord` for logging or external reporting.

# Operational Risks
- **Nullability / Primitive mismatch** – `cdrCount` is a `Long` wrapper but getter returns primitive `long`; a `null` value will cause `NullPointerException`.  
  *Mitigation*: Enforce non‑null defaults in DAO or add null checks in getters.
- **Duplicate setter (`setCdrCount(Long)`)** – May cause confusion or accidental overload usage.  
  *Mitigation*: Consolidate to a single setter with consistent type.
- **Lack of validation** – No constraints on `usage` (negative values) or `fileName` format.  
  *Mitigation*: Add validation in setters or a dedicated validator utility.
- **Serialization incompatibility** – Float may lose precision for high‑volume data.  
  *Mitigation*: Consider `BigDecimal` for monetary/precision‑critical fields.

# Usage
```java
// Example: constructing and debugging an RARecord
RARecord rec = new RARecord();
rec.setFileName("cdr_20251101.dat");
rec.setCallType("VOICE");
rec.setUsage(12.5f);
rec.setCdrCount(1L);
rec.setCallId("CALL123456");
rec.setSecsId(987654321L);

// Debug output
System.out.println("RARecord: " + rec.getFileName() + ", " + rec.getCallType()
    + ", usage=" + rec.getUsage() + ", cdrCount=" + rec.getCdrCount());
```
In unit tests, mock DAO to return a list of `RARecord` objects and verify downstream billing calculations.

# Configuration
`RARecord` itself does not reference external configuration. It relies on:
- **Environment**: None.
- **Config files**: None.
- **Frameworks**: May be managed by Spring as a prototype bean, but no explicit config required.

# Improvements
1. **Remove duplicate setter & unify type**  
   - Delete `public void setCdrCount(Long cdrCount)` and keep only the primitive version, or vice‑versa with consistent null handling.
2. **Add basic validation & utility methods**  
   - Implement `validate()` to enforce non‑null mandatory fields and non‑negative usage.  
   - Override `toString()`, `equals()`, and `hashCode()` for better logging and collection handling.