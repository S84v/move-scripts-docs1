**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\APIMain.java`

---

## 1. High‑Level Summary
`APIMain` is the entry point for the *API Access Management Migration* job. It boot‑straps the runtime environment by loading a property file into JVM system properties, configures the Log4j logger, and dispatches one of four operational modes supplied on the command line:

| Mode (`args[2]`) | Action |
|------------------|--------|
| `workflow`       | Executes the main data‑movement workflow (`APICallout.callOut`) using a supplied input‑group‑ID file. |
| `status_update`  | Runs the order‑status reconciliation (`OrderStatus.orderStatusCheck`). |
| `bulk_insert`    | Performs a bulk‑load of reference data (`BulkInsert.insertAll`). |
| `usage_product_details` | Generates usage‑product detail records (`UsageProdDetails.usagePrdDetails`). |

The class therefore orchestrates the high‑level flow while delegating all business logic to the other call‑out classes.

---

## 2. Important Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`APIMain` (class)** | Application bootstrap and mode dispatcher. |
| `public static void main(String[] args)` | Parses command‑line arguments, calls `initializeApp`, and invokes the appropriate worker class based on the mode argument. |
| `private static void initializeApp(String propertyFile, String logFile)` | Loads the supplied `.properties` file, copies each entry into `System` properties, sets the Log4j logfile name (`logfile.name`), and creates a logger instance. |
| `private static String getStackTrace(Exception e)` | Utility to convert an exception stack trace to a `String` for logging. |
| **External worker classes** (referenced only) | <ul><li>`APICallout` – core data‑movement / transformation logic.</li><li>`OrderStatus` – order‑status verification and update.</li><li>`BulkInsert` – bulk database insert routine.</li><li>`UsageProdDetails` – generation of usage‑product detail records.</li></ul> |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Command‑line inputs** | `args[0]` – absolute path to the property file (e.g., `APIAcessMgmt.properties`).<br>`args[1]` – absolute path to the Log4j logfile (e.g., `APICronLogs/Accessmgmt.log`).<br>`args[2]` – mode selector (`workflow`, `status_update`, `bulk_insert`, `usage_product_details`).<br>`args[3]` – *optional* input‑group‑ID file path (required only for `workflow`). |
| **File inputs** | The property file referenced by `args[0]`. It contains key/value pairs used by downstream components (DB URLs, credentials, API endpoints, etc.). |
| **Outputs** | Log statements written to the file defined by `args[1]`. Downstream worker classes may produce additional files, DB writes, or API calls – not directly visible in this class. |
| **Side effects** | <ul><li>System properties are populated for the entire JVM process (visible to all subsequent classes).</li><li>Log4j logger is instantiated and configured.</li><li>Process may terminate via `System.exit(1)` on initialization failure.</li></ul> |
| **Assumptions** | <ul><li>All required command‑line arguments are present and correctly ordered.</li><li>The property file exists, is readable, and contains all keys expected by downstream components.</li><li>Log4j is configured to read the `logfile.name` system property.</li><li>Worker classes (`APICallout`, `OrderStatus`, etc.) are on the classpath and have no further runtime dependencies missing.</li></ul> |

---

## 4. Integration Points & Connections

| Connection | Description |
|------------|-------------|
| **`APIAcessMgmt.properties`** (see HISTORY) | Provides configuration values (DB connections, API URLs, credentials, batch sizes, etc.) that are injected as system properties. |
| **Log4j configuration (`log4j.properties`)** | Reads the `logfile.name` system property set by `initializeApp` to direct logs to the appropriate file. |
| **`APICallout`** | Consumes the system properties and the input‑group‑ID file to perform the main data‑movement workflow. |
| **`OrderStatus`**, **`BulkInsert`**, **`UsageProdDetails`** | Each reads the same system properties for DB/API access and performs its specific task. |
| **Potential downstream services** | Not visible in this file but implied: <ul><li>Relational databases (via DAO layer defined in `APIAccessDAO.properties`).</li><li>External REST/SOAP APIs (e.g., provisioning, billing).</li><li>Message queues or file drops for bulk insert.</li></ul> |
| **Execution environment** | Typically launched by a scheduler (cron, Control-M, etc.) that supplies the four arguments. The scheduler may also capture the exit code for alerting. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Missing or malformed command‑line arguments** – leads to `ArrayIndexOutOfBoundsException` (currently swallowed). | Validate argument count early; exit with a clear error message and non‑zero status. |
| **`logger` may be `null` when an exception occurs before initialization** – `logger.error` will throw `NullPointerException`. | Initialise the logger *before* any file I/O (e.g., a basic console logger) or guard log calls with a null check. |
| **Silent catch of generic `Exception` in `main`** – errors are lost, making debugging hard. | Log the exception stack trace and re‑throw or exit with a distinct status code. |
| **System.exit on any init failure** – abrupt termination may leave resources (DB connections, file handles) open in a larger process. | Prefer throwing a custom exception to the scheduler, allowing graceful cleanup. |
| **Properties are set as global `System` properties** – risk of key collisions or accidental overrides in a shared JVM. | Scope properties to a dedicated configuration object passed to workers, or namespace keys (e.g., `api.migration.*`). |
| **Hard‑coded reliance on Log4j property `logfile.name`** – if Log4j config changes, logging may break. | Document the required Log4j property and consider programmatic Log4j configuration within `initializeApp`. |

---

## 6. Running & Debugging the Script

### 6.1 Typical Production Invocation (via scheduler)

```bash
java -cp /opt/move-indiamed-api/lib/*:/opt/move-indiamed-api/build/classes \
    com.tcl.api.callout.APIMain \
    /opt/move-indiamed-api/PropertyFiles/APIAcessMgmt.properties \
    /opt/move-indiamed-api/APICronLogs/Accessmgmt.log \
    workflow \
    /opt/move-indiamed-api/inputgroupidlistfile
```

*Replace the classpath with the actual JAR/compiled‑class locations.*

### 6.2 Local Development / Debugging

1. **Set up IDE** (Eclipse/IntelliJ) with the project’s source root and all dependent JARs.
2. **Create a Run Configuration** with the same four arguments as above.
3. **Enable Log4j console appender** in `log4j.properties` for immediate feedback.
4. **Breakpoints** can be placed in `initializeApp` (to verify property loading) and in each worker class (`APICallout`, `OrderStatus`, etc.) to step through the business logic.
5. **Inspect System Properties** after `initializeApp` via `System.getProperties()` to confirm expected keys are present.

### 6.3 Quick Diagnostic Script

```bash
# Verify property file can be read
java -cp ... com.tcl.api.callout.APIMain \
    /tmp/APIAcessMgmt.properties \
    /tmp/dummy.log \
    status_update
# Expect a clean exit (code 0) and a log entry in /tmp/dummy.log
echo $?
```

If the exit code is non‑zero, check the logfile for the error message printed by `logger.error`.

---

## 7. External Configuration & Environment Variables

| Item | Usage |
|------|-------|
| **`APIAcessMgmt.properties`** (passed as `args[0]`) | Loaded at runtime; each key/value pair is copied into `System` properties. Typical keys include `db.url`, `db.user`, `db.password`, `api.endpoint`, `batch.size`, etc. |
| **`logfile.name`** (system property set by `initializeApp`) | Consumed by Log4j configuration to direct all log output to the file supplied as `args[1]`. |
| **Java system properties** (e.g., `java.home`, `file.encoding`) | Unchanged; only the migration‑specific keys are added. |
| **Environment variables** | Not accessed directly in this class, but downstream components may read variables such as `JAVA_OPTS`, `CLASSPATH`, or OS‑level credentials. |

---

## 8. Suggested Improvements (TODO)

1. **Robust argument validation & usage help**  
   ```java
   if (args.length < 3) {
       System.err.println("Usage: APIMain <propertyFile> <logFile> <mode> [inputGroupIdFile]");
       System.exit(2);
   }
   ```
2. **Replace `System.exit` with custom exception propagation** to allow the scheduler or a wrapper script to perform cleanup and alerting without abruptly killing the JVM.  

   *Optional*: Introduce a `MigrationException` hierarchy and return distinct exit codes for different failure categories.

---