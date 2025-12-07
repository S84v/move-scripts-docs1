# Summary
`SubsCountUtils` is a pure‑utility class that supplies date‑related helper methods for the APISimCountLoader batch job. It computes month‑boundary timestamps (first day, last day, previous month range) and generates a list of day strings for the current or previous month. The class is used by DAO and loader components to build Hive partition predicates and to validate that raw and aggregated data are synchronized before loading SIM‑count tables.

# Key Components
- **`getMonthStartDate()`** – Returns the first day of the current month at 00:00:00 formatted as `yyyy‑MM‑dd`.
- **`getMonthCurrentDate()`** – Returns yesterday’s date (current day minus one) at 00:00:00 formatted as `yyyy‑MM‑dd`.
- **`getPrevMonthStartDate()`** – Returns the first day of the previous month at 00:00:00 formatted as `yyyy‑MM‑dd`.
- **`getPrevMonthLastDate()`** – Returns the last day of the previous month at 00:00:00 formatted as `yyyy‑MM‑dd`.
- **`isFirstOfMonth()`** – Boolean flag indicating whether today is the first calendar day of the month.
- **`getMonthDates(boolean isPrev)`** – Generates a `List<String>` of all dates (`yyyy‑MM‑dd`) from the 1st up to the day before today for the current month (`isPrev = false`) or for the entire previous month (`isPrev = true`). Includes a `System.out.println` debug output for the previous‑month case.

# Data Flow
| Method | Input | Output | Side Effects / External Interaction |
|--------|-------|--------|--------------------------------------|
| `getMonthStartDate` | none | `String` (e.g., `2025-12-01`) | None |
| `getMonthCurrentDate` | none | `String` (e.g., `2025-11-30`) | None |
| `getPrevMonthStartDate` | none | `String` (e.g., `2025-11-01`) | None |
| `getPrevMonthLastDate` | none | `String` (e.g., `2025-11-30`) | None |
| `isFirstOfMonth` | none | `boolean` | None |
| `getMonthDates` | `boolean isPrev` | `List<String>` of dates | Prints a `System.out` line when `isPrev` is `true` |

All methods rely exclusively on `java.util.Calendar` and `java.text.SimpleDateFormat`; they do **not** access databases, message queues, or external services.

# Integrations
- **`SubsCountDAO`** – Calls `SubsCountUtils` to obtain partition start/end dates for Hive table truncation and insertion statements.
- **`SubsCountLoader`** – Uses the utility to decide whether the job should run (e.g., only on the first day of the month) and to construct partition predicates for validation steps.
- **Batch Scheduler / Cron** – The loader is invoked by an external scheduler; the utility functions provide the date context required for the scheduled run.

# Operational Risks
1. **Timezone / Locale Dependency** – `Calendar.getInstance()` uses the JVM default timezone and locale, which may differ between development, test, and production clusters, causing off‑by‑one‑day errors.  
   *Mitigation*: Explicitly set `TimeZone` (e.g., `Calendar.getInstance(TimeZone.getTimeZone("UTC"))`) or migrate to `java.time` with a fixed zone.
2. **Thread‑Safety** – `SimpleDateFormat` is instantiated per call, so thread‑safety is not an issue, but repeated creation incurs overhead.  
   *Mitigation*: Cache a thread‑local formatter or switch to `DateTimeFormatter`.
3. **Debug Output** – `System.out.println` in `getMonthDates(true)` writes to stdout, polluting logs and potentially exposing large volumes of data.  
   *Mitigation*: Replace with a logger at `DEBUG` level or remove entirely.
4. **Calendar Mutability** – The same `Calendar` instance is mutated throughout a method, which can be error‑prone if future modifications are added.  
   *Mitigation*: Use immutable `java.time` objects.

# Usage
```java
// Example: generate yesterday’s date for partition predicate
String yesterday = SubsCountUtils.getMonthCurrentDate();

// Example: list all dates of the previous month for validation
List<String> prevMonthDays = SubsCountUtils.getMonthDates(true);

// Example: check if today is the first day of the month before triggering load
if (SubsCountUtils.isFirstOfMonth()) {
    // invoke DAO load methods
}
```
To debug, run a unit test or a simple `main` method that prints each utility result.

# Configuration
No external configuration files, environment variables, or system properties are referenced by this class. All behavior is derived from the JVM’s default locale and timezone.

# Improvements
1. **Migrate to `java.time` API** – Replace `Calendar`/`SimpleDateFormat` with `LocalDate`, `ZoneId`, and `DateTimeFormatter` to obtain immutable, thread‑safe, and clearer date handling.
2. **Remove `System.out.println`** – Substitute with `logger.debug(...)` (or eliminate) to keep output under the application’s logging framework and avoid unintended stdout noise.