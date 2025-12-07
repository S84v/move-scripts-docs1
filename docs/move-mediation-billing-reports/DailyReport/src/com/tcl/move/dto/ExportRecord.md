# Summary
`ExportRecord` is a plain‑old‑Java‑Object (POJO) DTO that encapsulates all fields required for a daily mediation billing report. It is used throughout the Move‑Mediation‑Billing‑Reports pipeline to transport raw CDR‑derived metrics between parsers, aggregators, and CSV/Excel exporters.

# Key Components
- **Class `ExportRecord`**
  - Private member variables representing report columns (dates, identifiers, usage metrics, billing flags, network attributes, etc.).
  - Public getter/setter pairs for each field, following JavaBean conventions.
  - No business logic; solely a data carrier.

# Data Flow
| Stage | Input | Transformation | Output |
|-------|-------|----------------|--------|
| CDR parsing (outside this file) | Raw CDR records (CSV, DB, MQ) | Populate an `ExportRecord` instance via setters | `ExportRecord` objects passed to aggregation services |
| Aggregation/Calculation | List of `ExportRecord` objects | Compute totals, apply business rules, update fields (e.g., `dataAct`, `outVoiceAct`) | Updated `ExportRecord` list |
| Exporter (CSV/Excel writer) | `ExportRecord` list | Serialize fields in defined column order | Report files written to filesystem or S3 |

Side effects: none within `ExportRecord`. All I/O occurs in external components that consume or produce instances.

# Integrations
- **Parser modules** (e.g., `CDRParser`, `MediationProcessor`) instantiate and fill `ExportRecord`.
- **Aggregation services** (`ReportAggregator`, `MetricsCalculator`) read/write fields.
- **Export utilities** (`CsvReportWriter`, `ExcelReportWriter`) reflectively access getters to generate output.
- **Spring/DI container** may inject `ExportRecord` as prototype beans (not shown in code).

# Operational Risks
- **Missing validation** – setters accept any value; malformed data can propagate to reports. *Mitigation*: add input validation or use builder pattern with constraints.
- **Inconsistent naming** – duplicate getters (`getIPUCount` vs `getiPUCount`) may cause reflection errors. *Mitigation*: consolidate to a single canonical accessor.
- **Floating‑point precision** – usage metrics stored as `float`; rounding errors may affect billing. *Mitigation*: switch to `java.math.BigDecimal` for monetary/usage values.

# Usage
```java
// Example in a unit test or debugging session
ExportRecord rec = new ExportRecord();
rec.setCallDate("2025-12-01");
rec.setMsisdn("447700900123");
rec.setData(1.23f);
rec.setOutVoice(0.45f);
// Pass to aggregator or writer
reportWriter.writeRecord(rec);
```
Run the application with the standard Maven command:
```
mvn clean package
java -jar target/daily-report.jar
```
Inspect `ExportRecord` values via debugger or log statements.

# Configuration
`ExportRecord` itself does not read configuration. Relevant external configs:
- **application.properties / yaml** – defines report output path, date formats, field ordering.
- **Environment variables** – e.g., `REPORT_OUTPUT_DIR`, `REPORT_DATE_FORMAT`.

# Improvements
1. **Consolidate duplicate accessors** – remove `getiPUCount`/`setiPUCount` and `getiPSCount`/`setiPSCount`; keep only the correctly cased versions.
2. **Introduce immutable builder** – replace mutable setters with a `ExportRecordBuilder` that validates required fields and enforces type safety (e.g., `BigDecimal` for usage).