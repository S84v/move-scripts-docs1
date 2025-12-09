# Summary
`Constants.java` centralises static string literals and date‑format patterns used throughout the **RAReportsExport** batch suite. The class supplies uniform identifiers for file names, sheet titles, report types and date formats, enabling consistent naming, logging and query parameterisation across shell scripts, DAO layers and Excel/CSV generation components in the MOVE‑RA‑Reports production pipeline.

# Key Components
- **Class `com.tcl.move.constants.Constants`**
  - Public static final **date‑format strings** (`YMD_HYPHEN_FORMAT`, `YM_HYPHEN_FORMAT`, …) for `SimpleDateFormat` usage.
  - Public static final **file‑name descriptors** (`FILE_DAILY`, `PPU_SUBS_DAILY`, `E2E_MONTHLY`, …) employed when constructing export file names.
  - Public static final **sheet‑name constants** (`FILE_SHEET`, `REJECT_SHEET`) for Excel workbook tabs.
  - Public static final **report‑type identifiers** (`DAILY`, `MONTHLY`, `CUST_MONTHLY`, …) used by job‑dispatch logic and configuration look‑ups.

# Data Flow
| Element | Source | Destination | Effect |
|---------|--------|--------------|--------|
| Date‑format constants | `Constants` (static) | `SimpleDateFormat` instances in DAO, utility, or script wrappers | Guarantees uniform date strings for DB queries, file naming, and logging |
| File‑name constants | `Constants` | Shell scripts (`MNAAS_ShellScript.properties`), Java exporters, email attachment builders | Drives deterministic export filenames for daily/monthly PPU reports and reconciliation files |
| Sheet‑name constants | `Constants` | Excel generation utilities (e.g., Apache POI) | Ensures consistent worksheet titles across generated reports |
| Report‑type identifiers | `Constants` | Job scheduler / dispatcher code, property look‑ups | Maps job parameters (`DAILY`, `MONTHLY`) to configuration blocks in property files |

No direct I/O occurs within the class; it is a pure compile‑time constant holder.

# Integrations
- **`MOVEDAO.properties`**: SQL statements reference date formats and report identifiers that are built using constants.
- **Shell scripts** (`RAReportsExport` batch) read property files and construct filenames via `${PPU_SUBS_DAILY}${date}.csv` patterns, where the prefix originates from `Constants`.
- **Log4j configuration** (`log4j.properties`): Log messages embed constant values for report types.
- **Java export components** (e.g., CSV/TSV writers, Excel creators) import `Constants` to name output files and worksheets.
- **Email notification module**: Uses `Constants.E2E_MONTHLY` as email subject prefix.

# Operational Risks
- **Hard‑coded literals**: Changing a name requires recompilation; risk of mismatched identifiers across environments.
- **Naming collisions**: Identical constant values for different reports may cause file overwrites.
- **Locale‑sensitive date formats**: `SimpleDateFormat` is not thread‑safe; concurrent reuse without cloning can cause data corruption.
- **Missing documentation**: New developers may misuse constants without understanding their intended scope.

**Mitigations**
- Enforce code‑review checklist for any constant modification.
- Introduce unit tests that validate generated filenames against expected patterns.
- Wrap date‑format strings in thread‑local `DateTimeFormatter` (Java 8+) where concurrency is possible.
- Maintain a mapping matrix in project wiki linking each constant to its consumer.

# Usage
```java
import com.tcl.move.constants.Constants;
import java.text.SimpleDateFormat;
import java.util.Date;

// Example: build daily PPU subscription filename
String datePart = new SimpleDateFormat(Constants.YMD_HYPHEN_FORMAT).format(new Date());
String fileName = Constants.PPU_SUBS_DAILY + datePart + ".csv";
// Result: "daily_subscription_report_2024-12-08.csv"
```
Debug by printing constant values or stepping into the calling component (e.g., DAO or export script) to verify correct substitution.

# Configuration
- No environment variables are read.
- Constants are referenced indirectly via property files:
  - `MNAAS_ShellScript.properties` (file‑name prefixes)
  - `MOVEDAO.properties` (SQL placeholders)
- Ensure the compiled JAR includes `Constants.class` on the classpath of all batch jobs.

# Improvements
1. **Externalise mutable identifiers** – Move file‑name prefixes and sheet titles to a dedicated properties file (e.g., `report-names.properties`) and load them at runtime to avoid recompilation for naming changes.
2. **Adopt type‑safe enums** – Replace string‑based report‑type identifiers (`DAILY`, `MONTHLY`, `CUST_MONTHLY`) with an `enum ReportPeriod { DAILY, MONTHLY, CUSTOMER_MONTHLY }` to enable compile‑time validation and reduce typo‑related bugs.