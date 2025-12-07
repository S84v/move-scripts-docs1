# Summary
`SplitRecord` is a plain‑old‑Java‑Object (POJO) used in the Move‑Geneva mediation pipeline to capture the exact point where a usage session transitions from in‑bundle to out‑of‑bundle. It stores CDR identifiers delimiting the two intervals, a flag indicating whether a split occurred, and the usage amounts (KB/SMS/min) for each side of the split. The object is instantiated by the DAO/processing layer after analysing CDR sequences and is later consumed by the billing engine to apply correct tariff rules.

# Key Components
- **Class `SplitRecord`**
  - **Fields**
    - `Long inbEndCdrNum` – CDR number of the last in‑bundle record.
    - `Long outbStartCdrNum` – CDR number of the first out‑of‑bundle record.
    - `boolean isSplit` – Indicates if a split was detected.
    - `Long splitCdrNum` – CDR number where the split occurs (optional).
    - `String splitDestType` – Destination type at split point (e.g., “ROAMING”).
    - `String splitTraffictype` – Traffic direction/type at split point (e.g., “VOICE”, “DATA”).
    - `float inUsageKbSMSMin` – Aggregated in‑bundle usage (KB/SMS/min) up to the split.
    - `float outUsageKbSMSMin` – Aggregated out‑of‑bundle usage after the split.
  - **Accessors/Mutators** – Standard getters and setters for each field.
  - **`toString()`** – Human‑readable representation for logging/debugging.

# Data Flow
| Stage | Input | Processing | Output |
|-------|-------|------------|--------|
| DAO / CDR Analyzer | Raw CDR stream (ordered by CDR number) | Detects transition from in‑bundle to out‑of‑bundle based on tariff thresholds, aggregates usage per side | `SplitRecord` instance populated with CDR numbers and usage totals |
| Billing Engine | `SplitRecord` objects (via in‑memory collection or passed through service layer) | Applies out‑of‑bundle pricing, updates charge records | Charge entries persisted to billing DB; no side‑effects from `SplitRecord` itself |
| Logging | `SplitRecord` | `toString()` invoked for audit logs | Log entry containing split details |

No direct external services, DB connections, or message queues are referenced inside this class; it is a data carrier.

# Integrations
- **`UsageDataAccess` / CDR parsing modules** – Create and populate `SplitRecord` after evaluating usage windows.
- **Move‑Geneva billing engine** – Consumes `SplitRecord` to compute out‑of‑bundle charges.
- **Logging framework (e.g., Log4j)** – Uses `toString()` for traceability.
- **Potential serialization** – May be transferred via Java serialization or JSON when moving between micro‑services (not shown in code).

# Operational Risks
- **Incorrect split detection** – If upstream logic mis‑calculates CDR boundaries, billing will be inaccurate. *Mitigation*: unit‑test split detection with edge cases (exact threshold, zero‑usage records).
- **Null field usage** – Getters may return `null` for optional fields; downstream code must handle NPEs. *Mitigation*: enforce non‑null contracts or use `Optional`.
- **Precision loss** – `float` for usage may truncate large values. *Mitigation*: switch to `double` or `BigDecimal` for monetary‑grade precision.

# Usage
```java
// Example in a unit test or debugging session
SplitRecord split = new SplitRecord();
split.setInbEndCdrNum(10234L);
split.setOutbStartCdrNum(10235L);
split.setSplit(true);
split.setSplitCdrNum(10235L);
split.setSplitDestType("ROAMING");
split.setSplitTraffictype("DATA");
split.setInUsageKbSMSMin(1500.5f);
split.setOutUsageKbSMSMin(300.75f);

// Log the record
System.out.println(split);   // invokes toString()
```
Debugging tip: set a breakpoint on `setSplit(true)` to verify that the split flag is correctly toggled during CDR analysis.

# configuration
The class itself does not read configuration. Operational configuration affecting its population includes:
- **Tariff threshold values** – defined in `application.properties` or equivalent (e.g., `bundle.usage.limit.mb`).
- **CDR ordering source** – configured in DAO layer (file path, DB connection string).
- **Logging level** – `log4j.properties` entry for the package `com.tcl.move.dto`.

# Improvements
1. **Replace `float` with `double` or `BigDecimal`** to avoid precision loss for high‑volume usage metrics.
2. **Add validation method** (e.g., `validate()`) that asserts required fields are non‑null and that `inbEndCdrNum < outbStartCdrNum` when `isSplit` is true, throwing a domain‑specific exception on failure.