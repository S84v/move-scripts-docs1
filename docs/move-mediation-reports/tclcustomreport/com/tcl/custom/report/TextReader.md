# Summary
`TextReader` is a utility class that loads a plain‑text file containing key‑value pairs, parses each line using a configurable delimiter, stores the mappings in a `Map`, and returns the value associated with a supplied *process name*. In the Move‑Mediation reporting pipeline it is used to retrieve Hive SQL query templates (or other per‑process text assets) required by downstream components such as `ReportWriter`.

# Key Components
- **Class `TextReader`**
  - **Fields**
    - `String textToReadPath` – absolute or relative file system path of the source text file.
    - `String processName` – key whose value is to be returned.
    - `String delimiter` – token that separates key and value on each line (defaults to `":==>"`).
    - `Map<String,String> textMapping` – in‑memory map populated from the file.
  - **Constructor `TextReader(String queryPath, String processName, String delimiter)`**
    - Assigns constructor arguments to fields.
    - Applies default delimiter when `null` is supplied.
  - **Method `String getText()`**
    - Opens the file via `Files.lines(Paths.get(textToReadPath))` (try‑with‑resources).
    - Splits each line on the delimiter, trims both parts, and collects into `textMapping`.
    - Returns `textMapping.get(processName)`.
    - Prints stack trace on `IOException`.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1 | `textToReadPath` (file path), `processName`, optional `delimiter` | File is streamed line‑by‑line; each line is split on the delimiter; trimmed key/value pairs are stored in `textMapping`. | Populated `Map<String,String>` in memory. |
| 2 | `processName` (key) | Lookup in `textMapping`. | Returns the associated value (e.g., a Hive query string) or `null` if absent. |
| Side‑Effects | File system read | I/O operation; possible `IOException`. | Stack trace printed to `stderr`. No external services or DB connections are invoked. |

# Integrations
- **`ReportWriter`** – Calls `new TextReader(queryPath, processName, delimiter).getText()` to obtain the Hive SQL template for the current reporting process.
- **`ReportExporterMain`** – Supplies the `processName` argument that propagates down to `TextReader` via `ReportWriter`.
- **Configuration files** – The `queryPath` argument typically points to a directory containing `.txt` files where each line follows the `<processName><delimiter><value>` format.

# Operational Risks
1. **File Not Found / Permission Issues** – `Files.lines` throws `IOException`; current implementation only prints stack trace, potentially masking the failure.  
   *Mitigation*: Validate file existence and readability before streaming; surface a clear exception to the caller.
2. **Malformed Lines** – Lines without the delimiter or with missing key/value cause `ArrayIndexOutOfBoundsException` during `l[1]`.  
   *Mitigation*: Guard against split results length; log and skip malformed entries.
3. **Delimiter Collisions** – If the delimiter appears within a value, splitting will truncate the value.  
   *Mitigation*: Use a more robust parsing strategy (e.g., limit split to 2 parts) or enforce delimiter uniqueness in source files.
4. **Memory Footprint** – Entire file is materialized into a `Map`; very large files could exhaust heap.  
   *Mitigation*: Stream‑process only the required key or impose size limits.
5. **Thread Safety** – `textMapping` is mutable and not synchronized; concurrent calls on the same instance could lead to race conditions.  
   *Mitigation*: Make `TextReader` immutable or synchronize access; preferably instantiate a new object per request.

# Usage
```java
// Example in a unit test or debugging session
String queryFile = "/opt/move-mediation/reports/queries.txt";
String process = "RevenueReport";
String delim = ":==>";   // optional; null will use default

TextReader reader = new TextReader(queryFile, process, delim);
String hiveQuery = reader.getText();

System.out.println("Loaded query: " + hiveQuery);
```
*Debugging tip*: Place a breakpoint inside `getText()` to verify the contents of `textMapping` after the stream completes.

# Configuration
- **Environment / System Properties**: None required directly by `TextReader`.
- **External Config Files**:
  - `queryPath` – Path supplied by the caller (often read from a properties file in `ReportWriter`).
  - Delimiter – Optional; if omitted, defaults to `":==>"`.

# Improvements
1. **Robust Error Handling**  
   Replace `e.printStackTrace()` with a structured logging call and rethrow a custom unchecked exception (e.g., `TextReaderException`) to allow upstream components to react appropriately.
2. **Immutable Design & API Simplification**  
   - Declare fields `final`.  
   - Build `textMapping` lazily on first `getText()` call or, better, expose a static utility method `String readValue(Path file, String key, String delimiter)` that reads only the needed line, eliminating the full map and reducing memory usage.  

---