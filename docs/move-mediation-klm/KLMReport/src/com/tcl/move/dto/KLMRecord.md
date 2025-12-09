# Summary
`KLMRecord` is a POJO representing a single usage record for the MOVE‑Mediation‑Ingest pipeline. It stores raw usage metrics (bytes) and pre‑computed bundle limits, provides getters/setters, calculates derived units (KB/MB/GB) and overage values, and supplies a helper to populate all unit fields from the base values. It is used downstream for bundle‑split, billing, and reporting logic.

# Key Components
- **Class `KLMRecord`**
  - Private fields: `orgNo`, `custName`, `proposition`, `callDate`, `zone`, `country`, `usageBytes`, `usageKB`, `usageMB`, `usageGB`, `bundleBytes`, `bundleKB`, `bundleMB`, `bundleGB`.
  - Standard JavaBean getters and setters for each field.
  - **`getOverageBytes/KB/MB/GB`** – compute positive overage (usage – bundle) per unit; return `0F` when no overage.
  - **`calculateUsageAndBundle()`** – derives KB/MB/GB from `usageBytes` and derives bundle bytes/KB/MB from `bundleGB`.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| DAO / Service layer | Raw DB/Hive rows (orgNo, custName, …, usageBytes, bundleGB) | Populate a `KLMRecord` instance via setters; invoke `calculateUsageAndBundle()` | Fully populated `KLMRecord` with all unit fields and overage helpers | None (in‑memory only) |
| Business logic (e.g., `BundleSplitRecord`) | Collection of `KLMRecord` objects | Group, filter, compute in‑bundle/out‑of‑bundle sets | Modified collections passed to reporting or persistence layers | May trigger DB writes elsewhere, but not within this class |

External dependencies: none (pure data object). Relies on callers to supply correct numeric values.

# Integrations
- **`BundleSplitRecord`** – aggregates `KLMRecord` instances for zone‑level bundle split.
- **`MediationDataAccess`** – reads raw rows from Impala/Hive, maps each row to a `KLMRecord`.
- **Reporting / Billing modules** – consume overage getters to calculate charges.
- **Logging / Exception handling** – performed by surrounding DAO/service code; not present in this class.

# Operational Risks
- **Floating‑point precision** – use of `Float` may cause rounding errors in large byte counts; consider `Double` or `BigDecimal`.
- **Division by zero** – `calculateUsageAndBundle()` assumes non‑null `usageBytes` and `bundleGB`; null values cause `NullPointerException`.
- **Unit consistency** – method derives bundle bytes from `bundleGB`; if source data already provides bundle bytes, inconsistency may arise.
- **Thread safety** – mutable POJO; shared instances across threads could lead to race conditions. Ensure each thread works on its own instance.

# Usage
```java
// Example creation and calculation
KLMRecord rec = new KLMRecord();
rec.setOrgNo(12345L);
rec.setCustName("Acme Corp");
rec.setProposition("MOVE SIM AVKLM-4Z-EUS-$500");
rec.setCallDate("2024-11-30");
rec.setZone(1);
rec.setCountry("DE");
rec.setUsageBytes(5_242_880F);   // 5 MB
rec.setBundleGB(0.01F);          // 10 MB bundle

rec.calculateUsageAndBundle();

float overageMB = rec.getOverageMB(); // returns 0 if usage ≤ bundle
```
Debug by inspecting field values after `calculateUsageAndBundle()`; unit conversion follows 1024‑based scaling.

# Configuration
No environment variables or external configuration files are referenced directly by this class.

# Improvements
1. **Replace `Float` with `double` or `BigDecimal`** to improve precision for large byte volumes and monetary calculations.
2. **Add null‑safety checks** in `calculateUsageAndBundle()` and overage methods; throw a descriptive `IllegalArgumentException` if required fields are missing.