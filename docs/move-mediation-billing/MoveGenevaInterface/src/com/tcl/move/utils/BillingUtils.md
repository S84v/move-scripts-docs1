# Summary
`BillingUtils` is a stateless utility class that supplies date‑range calculations, numeric padding, SQL placeholder generation, and unit‑of‑measure conversions required by the Move‑Geneva mediation batch. It is invoked by services that generate Geneva control/event files and by the mediation fetch logic to compute billing periods, round timestamps, and format values for persistence or file output.

# Key Components
- **`getFirstDayofTheMonth(String monthYear)`** – Parses *mm‑yyyy* and returns the first calendar day at 00:00:00.  
- **`getLastOrCurrDayofTheMonth(String monthYear)`** – Returns the current timestamp (23:59:59) if the supplied month equals the system month; otherwise returns the last day of the supplied month at 23:59:59.  
- **`getLastDayofTheMonth(String monthYear)`** – Returns the integer day count of the month (e.g., 28‑31).  
- **`getRoundedDate(Date date)`** – Truncates a `Date` to midnight (00:00:00.000).  
- **`padNumber(Long number, int padSize)`** – Left‑pads a numeric value with zeros to a fixed width.  
- **`getQuestionMarks(int noOfQM)`** – Builds a comma‑separated string of `?` placeholders for prepared statements.  
- **`getNextDay(String date)`** – Parses *yyyy‑MM‑dd*, adds one day, and returns a `Timestamp` at midnight.  
- **`getFirstDayofNextMonth(String monthYear)`** – Returns the first day of the month following the supplied *yyyy‑MM*.  
- **`mbToUOM(Float originalUsage, String uom)`** – Converts megabytes to KB, MB, GB, or TB.  
- **`uomToMB(Float originalUsage, String uom)`** – Converts KB/GB/TB to megabytes.  
- **`getNextMonth(String monthYear)`** – Returns the calendar month after the supplied *yyyy‑MM* in the same format.

# Data Flow
| Method | Input | Output | Side Effects |
|--------|-------|--------|--------------|
| All | Primitive strings, `Date`, `Float`, `Long` | `Date`, `Timestamp`, `String`, `Integer`, `Float` | Logs format errors via Log4j; throws generic `Exception` for parsing failures. |
- No external services, DB connections, or queues are accessed.  
- All methods are pure except for logging.

# Integrations
- **`GenevaFileService`** – Uses date helpers to compute billing period boundaries for file naming and header timestamps.  
- **`MediationFetchService`** – Calls `getFirstDayofTheMonth`, `getLastOrCurrDayofTheMonth`, and unit‑conversion utilities while building event records.  
- **SQL generation** – `getQuestionMarks` supplies placeholder strings for dynamic `INSERT` statements in DAO layers.  
- **Logging** – Relies on the application’s Log4j configuration.

# Operational Risks
- **Parsing fragility** – Methods split strings on “‑” without validation; malformed inputs cause generic `Exception` and batch failure.  
- **Timezone ambiguity** – Calendar uses the JVM default timezone; production runs across regions may yield off‑by‑one‑day errors.  
- **Null handling** – Returns `null` for empty inputs; callers must guard against NPEs.  
- **Thread‑safety of `SimpleDateFormat`** – Instances are created per call, avoiding concurrency issues, but repeated allocation incurs overhead.

# Usage
```java
// Example: compute billing window for March 2024
String monthYear = "2024-03";
Date start = BillingUtils.getFirstDayofTheMonth(monthYear);
Date end   = BillingUtils.getLastOrCurrDayofTheMonth(monthYear);

// Example: pad a subscriber ID to 10 digits
String paddedId = BillingUtils.padNumber(12345L, 10); // "0000012345"

// Example: convert 5000 MB to GB
Float gb = BillingUtils.mbToUOM(5000f, "GB"); // 4.8828125
```
Run unit tests or invoke from a debugger; all methods are static and require no initialization.

# Configuration
- No environment variables or external config files are referenced.  
- Relies on Log4j configuration (`log4j.properties` / `log4j.xml`) for logger output.

# Improvements
1. **Migrate to java.time API** – Replace `Calendar`/`Date`/`SimpleDateFormat` with `LocalDate`, `YearMonth`, and `DateTimeFormatter` to eliminate timezone ambiguity and improve readability.  
2. **Introduce typed validation** – Create a dedicated `MonthYear` value object that validates format on construction, reducing repetitive split‑and‑parse logic and providing clearer error messages.