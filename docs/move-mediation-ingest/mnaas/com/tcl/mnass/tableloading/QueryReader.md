# Summary
`QueryReader` is a utility class that loads a plain‑text query definition file, parses each line into a key/value pair using the delimiter `:==>`, stores the mappings in memory, and returns the query string associated with a supplied process name. It is used by table‑loading jobs to retrieve Hive DDL/DML statements at runtime.

# Key Components
- **Class `QueryReader`**
  - **Fields**
    - `String queryPath` – absolute/relative path to the query definition file.
    - `String processName` – key identifying the required query.
    - `Map<String,String> queryMapping` – in‑memory map of all parsed key/value pairs.
    - `String delimiter` – constant `:==>` used to split lines.
  - **Constructor `QueryReader(String queryPath, String processName)`**
    - Assigns constructor arguments to instance fields.
  - **Method `String queryCreator()`**
    - Opens the file at `queryPath` with a try‑with‑resources `Stream<String>`.
    - Splits each line on `delimiter`, trims key/value, and populates `queryMapping`.
    - Logs the full map to `System.out`.
    - Returns `queryMapping.get(processName)`; returns `null` if the key is absent.
    - Catches `IOException`, prints stack trace, and returns `null`.

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | `queryPath` (file system path) | `Files.lines` → stream of text lines | Stream of raw lines |
| 2 | Each line | `line.split(delimiter)` → `[key, value]` | Key/value pair (trimmed) |
| 3 | All pairs | `Collectors.toMap` → `queryMapping` | In‑memory map |
| 4 | `processName` | `queryMapping.get(processName)` | Query string (or `null`) |
| 5 | Exceptions | `catch(IOException)` | Stack trace printed to console |

External services: local file system (read‑only). No DB, queue, or network interaction.

# Integrations
- Consumed by table‑loading utilities (e.g., `LatLongTableLoading`, `cdr_buid_search_temp`, `dim_date_time_3months`) to retrieve Hive statements defined in external query files.
- The returned query string is passed to a Hive JDBC `Statement` for execution.
- Logging is performed via `System.out`; downstream components may redirect stdout to a log file.

# Operational Risks
- **Malformed line** – missing delimiter or extra delimiters cause `ArrayIndexOutOfBoundsException`. *Mitigation*: validate split length before mapping.
- **Missing key** – `processName` not present returns `null`, leading to downstream `null` query execution. *Mitigation*: add explicit check and fail fast with a clear error.
- **IO failure** – file not found or permission denied results in stack trace and `null` return. *Mitigation*: pre‑flight file existence check; surface error via structured logging.
- **Large file** – loading entire file into a map may exhaust memory for very large query catalogs. *Mitigation*: limit file size or stream‑lookup only the required key.
- **Concurrent use** – `queryMapping` is not thread‑safe; concurrent invocations could corrupt the map. *Mitigation*: make method synchronized or use a local map per call.

# Usage
```java
// Example in a unit test or main method
String path = "/opt/queries/hive_queries.txt";
String process = "lat_long_table_loading";

QueryReader reader = new QueryReader(path, process);
String hiveSql = reader.queryCreator();

if (hiveSql == null) {
    System.err.println("Query not found for process: " + process);
    System.exit(1);
}

// Pass hiveSql to Hive JDBC execution engine
```

# Configuration
- **Environment / System**: None required.
- **Configuration files**: The plain‑text query file referenced by `queryPath`. Expected format per line:
  ```
  <process_name> :==> <Hive SQL statement>
  ```
  Example:
  ```
  lat_long_table_loading :==> INSERT OVERWRITE TABLE lat_long SELECT ...
  ```

# Improvements
1. **Robust parsing** – Add validation for split results, ignore blank/comment lines, and throw a descriptive exception for malformed entries.
2. **Logging & error handling** – Replace `System.out`/`printStackTrace` with a proper logging framework (e.g., SLF4J) and return an `Optional<String>` or custom exception to signal missing queries.