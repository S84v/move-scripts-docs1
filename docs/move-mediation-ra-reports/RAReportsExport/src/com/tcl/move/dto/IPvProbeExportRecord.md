# Summary
`IPvProbeExportRecord` is a plain‑old‑Java‑Object (POJO) used in the Move‑Mediation revenue‑assurance reporting pipeline. It encapsulates per‑file metrics (file name, record counts, timestamps, customer, and derived daily/monthly counts) that are populated by DAO components reading Hive/SQL tables and later consumed by CSV/Excel export modules to generate daily, monthly, and Geneva reports.

# Key Components
- **Class `IPvProbeExportRecord`**
  - Private fields: `fileName`, `fileRecords`, `fileDate`, `secs`, `customerName`, `rawCount`, `dailyCount`, `monthlyCount`.
  - Public getters/setters for each field.
  - No business logic; serves solely as a data carrier between DAO and export layers.

# Data Flow
- **Input:** DAO layer (`IPvProbeFetchDAO` or similar) reads raw probe data from Hive/SQL, constructs an `IPvProbeExportRecord` instance, and sets field values.
- **Output:** Export layer (`ExcelExportDAO`, `CsvExportDAO`) receives the populated POJO, extracts values via getters, and writes rows to CSV/Excel files.
- **Side Effects:** None within the POJO; side effects occur in DAO (DB access) and export components (file I/O).
- **External Services/DBs:** Hive/SQL databases accessed by DAO; file system for report generation.

# Integrations
- **DAO Integration:** `IPvProbeExportRecord` is instantiated and populated in the data‑access layer that executes Hive/SQL queries.
- **Export Integration:** Export utilities iterate over collections of `IPvProbeExportRecord` objects to produce report files.
- **Potential Mapping:** May be referenced by service classes that orchestrate end‑to‑end report generation (e.g., `ReportGenerationService`).

# Operational Risks
- **Null Field Values:** Missing data from source tables can lead to `NullPointerException` in export logic if not guarded.
  - *Mitigation:* Validate DAO results; apply default values (e.g., `0L`) before setting.
- **Type Mismatch:** Fields are `Long`; source may provide numeric strings causing parsing errors.
  - *Mitigation:* Centralize conversion logic in DAO with error handling.
- **Schema Drift:** Adding/removing columns in source tables without updating this POJO can cause silent data loss.
  - *Mitigation:* Align POJO with database schema via automated schema‑to‑POJO generation or integration tests.

# Usage
```java
// Example in a unit test or service method
IPvProbeExportRecord record = new IPvProbeExportRecord();
record.setFileName("probe_20251208.csv");
record.setFileRecords(15000L);
record.setFileDate("2025-12-08");
record.setSecs(86400L);
record.setCustomerName("TCL");
record.setRawCount(15000L);
record.setDailyCount(14000L);
record.setMonthlyCount(420000L);

// Pass to export utility
excelExportDAO.writeIPvProbeRecord(record);
```
Debug by inspecting field values after DAO population; use IDE breakpoints on setters.

# Configuration
- No environment variables or config files are referenced directly by this class.
- Dependent components may rely on:
  - `application.properties` for DB connection strings.
  - Export path configuration (e.g., `report.output.dir`).

# Improvements
1. **Add Validation Method:** Implement `validate()` to ensure non‑null, non‑negative numeric fields before export.
2. **Override `toString()` / Implement `Serializable`:** Facilitate logging and potential distributed processing.