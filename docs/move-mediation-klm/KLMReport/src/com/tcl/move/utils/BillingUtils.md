# Summary
`BillingUtils` is a pure‑utility class that supplies date‑manipulation, number‑formatting, and unit‑conversion helpers for the MOVE‑Mediation‑KLM billing pipeline. It is invoked by services that prepare billing periods, generate SQL statements, and format usage metrics for reporting and persistence.

# Key Components
- **`getFirstDayofTheMonth(String monthYear)`** – Returns a `Date` set to the first day (00:00:00) of the supplied *yyyy‑mm* month; falls back to the current month on format error.  
- **`getLastOrCurrDayofTheMonth(String monthYear)`** – Returns the current date‑time (23:59:59) if the supplied month equals the current month; otherwise returns the last day (23:59:59) of the supplied month.  
- **`getLastDayofTheMonth(String monthYear)`** – Returns the numeric last‑day of the month; throws on format error.  
- **`getRoundedDate(Date date)`** – Clears hour/minute/second/millisecond fields.  
- **`padNumber(Long number, int padSize)`** – Left‑pads a numeric value with zeros.  
- **`getQuestionMarks(int noOfQM)`** – Generates a comma‑separated “?, ?, …” placeholder string for prepared‑statement batch inserts.  
- **`getNextDay(String date)`** – Parses *yyyy‑MM‑dd*, adds one day, and returns a `Timestamp` at 00:00:00.  
- **`getFirstDayofNextMonth(String monthYear)`** – Returns a `Date` for the first day of the month following the supplied month.  
- **`mbToUOM(Float originalUsage, String uom)`** – Converts megabytes to BYTE, KB, GB, or TB.  
- **`uomToMB(Float originalUsage, String uom)`** – Converts any of the above units back to megabytes.  
- **`getNextMonth(String monthYear)`** – Returns a *yyyy‑MM* string for the calendar month after the supplied month (handles non‑leap‑year Jan edge case).  
- **`getListAsString(List<String> list, String separator)`** – Concatenates list elements with a custom separator.  
- **`addSeconds(Date baseDate, int interval)`** – Adds a second‑based offset to a `Date`.  

All methods are `public static` and log format errors via Log4j.

# Data Flow
| Method | Input | Output | Side‑effects |
|--------|-------|--------|--------------|
| `getFirstDayofTheMonth` | `String monthYear` | `Date` (first day) | Logs format error |
| `getLastOrCurrDayofTheMonth` | `String monthYear` | `Date` (last or current day) | Logs format error |
| `getLastDayofTheMonth` | `String monthYear` | `Integer` (last day) | Throws on format error |
| `getRoundedDate` | `Date` | `Date` (midnight) | – |
| `padNumber` | `Long`, `int` | `String` | – |
| `getQuestionMarks` | `int` | `String` | – |
| `getNextDay` | `String yyyy‑MM‑dd` | `Timestamp` (next day midnight) | Prints stack trace on error |
| `getFirstDayofNextMonth` | `String monthYear` | `Date` | Throws on format error |
| `mbToUOM` / `uomToMB` | `Float`, `String` | `Float` | – |
| `getNextMonth` | `String monthYear` | `String yyyy‑MM` | Throws on format error |
| `getListAsString` | `List<String>`, `String` | `String` | – |
| `addSeconds` | `Date`, `int` | `Date` | – |

No direct DB, queue, or network calls; utilities are pure computational helpers.

# Integrations
- **`MediationFetchService`** – Uses date helpers to compute billing periods for Hadoop/DB extraction.  
- **`ExcelExport`** – May call `padNumber` or unit‑conversion methods when formatting usage values for Excel sheets.  
- **SQL builders** – `getQuestionMarks` supplies placeholder strings for batch `INSERT` statements in DAO layers.  
- **Reporting jobs** – `getNextMonth`, `getFirstDayofNextMonth`, and `getLastOrCurrDayofTheMonth` drive month‑over‑month report generation.

# Operational Risks
1. **Calendar month indexing bugs** – Manual `month-1` adjustments can produce off‑by‑one errors, especially around December/January transitions. *Mitigation*: migrate to `java.time` (`YearMonth`, `LocalDate`) which handles month boundaries natively.  
2. **Silent format fallback** – On malformed `monthYear` strings the methods fall back to the current month, potentially masking data‑quality issues. *Mitigation*: propagate a checked exception or return `Optional<Date>` to force caller validation.  
3. **Thread‑unsafe `SimpleDateFormat`** – Instances are created per call, but any future static reuse would be unsafe. *Mitigation*: keep per‑call instantiation or switch to `DateTimeFormatter`.  
4. **Logging of stack traces via `e.printStackTrace()`** in `getNextDay` bypasses Log4j and may pollute stdout. *Mitigation*: replace with `logger.error("...", e)`.  

# Usage
```java
// Example: compute billing window for March 2024
String monthYear = "2024-03";

Date periodStart = BillingUtils.getFirstDayofTheMonth(monthYear); // 2024‑03‑01 00:00:00
Date periodEnd   = BillingUtils.getLastOrCurrDayofTheMonth(monthYear); // 2024‑03‑31 23:59:59

// Convert 1500 MB to GB for reporting
Float usageGb = BillingUtils.mbToUOM(1500f, "GB"); // 1.46484375

// Build a prepared‑statement placeholder for 5 rows
String placeholders = BillingUtils.getQuestionMarks(5); // "?, ?, ?, ?, ?"
```
Debugging tip: set Log4j level