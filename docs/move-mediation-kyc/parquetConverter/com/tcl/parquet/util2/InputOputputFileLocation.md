# Summary
`InputOputputFileLocation` is a utility class that loads a Java `Properties` file from a supplied filesystem path and returns the populated `Properties` object. In the MOVE‑KYC pipeline it is used to read configuration files (e.g., file‑location mappings, environment‑specific settings) required by other components such as `CustomRecordBuilder` and `CustomParquetWriter`.

# Key Components
- **Class `InputOputputFileLocation`**
  - **Field `Properties prop`** – instance variable holding the last loaded properties.
  - **Method `Properties getProperty(String str)`**
    - Opens a `FileInputStream` for the path `str`.
    - Instantiates a new `Properties`, loads the stream, closes it.
    - Returns the populated `Properties` object.
    - Catches `IOException` and prints stack trace.

# Data Flow
| Step | Input | Process | Output | Side Effects |
|------|-------|---------|--------|--------------|
| 1 | Absolute or relative file path (`String str`) | `FileInputStream` → `Properties.load(InputStream)` | `Properties` instance containing key/value pairs | Opens and closes a file descriptor; prints stack trace on error |
| 2 | N/A | Returns the `Properties` object to caller | `Properties` | None |

No external services, databases, or messaging queues are involved.

# Integrations
- **`CustomRecordBuilder`** – may call `getProperty` to obtain CSV input locations or schema file paths.
- **`CustomParquetWriter`** – may use properties for output directory configuration.
- Any component that requires runtime configuration via a `.properties` file can instantiate this class and invoke `getProperty`.

# Operational Risks
- **Unchecked IOException** – stack trace printed to stdout; caller receives `null` properties, potentially causing NPEs. *Mitigation*: Propagate exception or return empty `Properties` with error logging.
- **Resource Leak on early return** – `input.close()` is redundant inside try‑with‑resources; however, if future modifications replace the block, explicit close may be omitted. *Mitigation*: Rely solely on try‑with‑resources.
- **Thread‑unsafe field `prop`** – shared mutable state across threads can lead to race conditions. *Mitigation*: Remove instance field; use a local variable.

# Usage
```java
public static void main(String[] args) {
    InputOputputFileLocation loader = new InputOputputFileLocation();
    Properties cfg = loader.getProperty("config/kysystem.properties");
    String inputDir = cfg.getProperty("csv.input.dir");
    System.out.println("CSV input directory: " + inputDir);
}
```
*Debug*: Set a breakpoint inside `getProperty` and verify that the returned `Properties` contains expected keys.

# Configuration
- **File path argument** passed to `getProperty` – absolute or relative path to a standard Java `.properties` file.
- No environment variables are read directly by this class.

# Improvements
1. **Error handling** – Replace `ex.printStackTrace()` with structured logging (SLF4J) and rethrow a custom checked exception to force caller handling.
2. **Thread safety & API simplification** – Remove the mutable `prop` field; make `getProperty` static and return a new `Properties` instance each call. Optionally cache immutable configurations using `java.util.concurrent.ConcurrentHashMap`.