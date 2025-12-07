# Summary
`ExportUtils` provides static helper methods to populate various Excel worksheets (Actives, Tolling, IPvProbe, eSIM Hub, Addon, Usage) with data from `ExportRecord` objects and includes a date utility (`getLastOrCurrDayofTheMonth`). It is used in the daily move‑mediation billing report generation pipeline to transform in‑memory DTOs into a formatted XLSX workbook.

# Key Components
- **fillActivesSheet(List<ExportRecord> holActives, List<ExportRecord> sngActives, XSSFSheet actSheet)**
  - Writes header and rows for HOL and SNG active SIM counts side‑by‑side.
- **fillTollingSheet(List<ExportRecord> holTolling, List<ExportRecord> sngTolling, XSSFSheet tollSheet)**
  - Writes header and rows for HOL and SNG tolling SIM counts, including “Billable?” flag.
- **fillIpvProbeSheet(List<ExportRecord> ipvProbe, XSSFSheet ipvProbeSheet)**
  - Writes header and rows for IPvProbe subscription counts (standard & unique).
- **fillESimHubSheet(List<ExportRecord> eSimAPIs, XSSFSheet esimSheet)**
  - Writes header and rows for eSIM Hub API transaction counts.
- **fillAddonSheet(List<ExportRecord> allAddon, XSSFSheet addonSheet)**
  - Writes header and rows for addon subscription counts with start/end month.
- **fillUsageSheet(List<ExportRecord> usage, XSSFSheet usageSheet)**
  - Writes header and rows for detailed usage metrics (CDR, data, voice, SMS, country, etc.).
- **getLastOrCurrDayofTheMonth(String monthYear)**
  - Returns a string `yyyy-MM-dd` representing the last day of the supplied month (or current day if month matches today). Handles malformed input with fallback to current date.

# Data Flow
| Method | Input | Output | Side Effects |
|--------|-------|--------|--------------|
| `fill*Sheet` | `List<ExportRecord>` + pre‑created `XSSFSheet` | Populated `XSSFSheet` (in‑memory) | Mutates sheet (adds rows, styles) |
| `getLastOrCurrDayofTheMonth` | `String monthYear` (`yyyy-MM`) | `String` formatted date | None (uses `Calendar`/`YearMonth`) |

All methods operate purely in memory; no external services, DB connections, or message queues are invoked.

# Integrations
- **Report Generator**: Called by the daily report job (likely a Spring/Quartz task) after retrieving `ExportRecord` collections from DAO/services.
- **Apache POI**: Utilized for workbook/sheet manipulation.
- **Log4j**: Used for error logging in date parsing.
- **ExportRecord DTO**: Must provide getters (`getCallMonth()`, `getOrgNo()`, `getCustName()`, etc.) matching the columns written.

# Operational Risks
- **Header Column Mismatch**: Hard‑coded column indices may diverge from downstream consumers; any change to column order requires code update.
- **Date Parsing Failure**: `getLastOrCurrDayofTheMonth` swallows parsing errors, defaulting to current month, potentially producing incorrect dates without alert.
- **Memory Pressure**: Large `List<ExportRecord>` can cause high heap usage when populating sheets; no streaming or chunking.
- **Thread Safety**: Methods are static but rely on mutable `XSSFSheet` objects; concurrent use on the same sheet would corrupt data.

# Usage
```java
// Example in a unit test or job
XSSFWorkbook workbook = new XSSFWorkbook();
XSSFSheet actSheet = workbook.createSheet("Actives");

// Assume holList and sngList are populated List<ExportRecord>
ExportUtils.fillActivesSheet(holList, sngList, actSheet);

// Write workbook to file
try (FileOutputStream out = new FileOutputStream("DailyReport.xlsx")) {
    workbook.write(out);
}
```
Debug by stepping into any `fill*Sheet` method; verify that the `ExportRecord` fields contain expected values.

# Configuration
- No environment variables or external config files are referenced directly.
- Relies on Log4j configuration for logging output.
- Date format is fixed to `yyyy-MM-dd` within the utility.

# Improvements
1. **Streamlined Header Generation** – Refactor repeated header creation into a reusable method that accepts a column‑name array to reduce duplication and risk of mismatched headers.
2. **Robust Date Utility** – Replace the silent fallback in `getLastOrCurrDayofTheMonth` with explicit validation and throw a checked exception or return `Optional<String>` to surface malformed inputs. Consider using `java.time` exclusively (e.g., `LocalDate`) for clarity and thread safety.