**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\APIMain.java`

---

## 1. High‑Level Summary
`APIMain` is the command‑line entry point for the *API Access Management* micro‑service. It bootstraps the application by loading a properties file (key/value pairs that become JVM system properties) and configuring Log4j with a runtime‑specified log file. Based on a third command‑line argument it delegates to one of four functional modules:

| Mode                | Class invoked                | Primary purpose |
|---------------------|------------------------------|-----------------|
| `workflow`          | `APICallout`                 | Calls external REST/SOAP APIs to push or pull data. |
| `status_update`     | `OrderStatus`                | Queries order status and updates internal tables. |
| `bulk_insert`       | `BulkInsert`                 | Performs high‑volume batch inserts into target stores. |
| `usage_product_details` | `UsageProdDetails`      | Extracts and persists usage‑product relationship data. |

The class is deliberately lightweight – it contains only `main`, a small initialization helper, and a utility to stringify stack traces.

---

## 2. Important Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`APIMain`** (public class) | Application bootstrap and mode dispatch. |
| `public static void main(String[] args)` | Parses three arguments (`propertyFile`, `logFile`, `mode`), calls `initializeApp`, then routes to the appropriate worker class. |
| `private static void initializeApp(String propertyFile, String logFile)` | Loads the supplied properties file, copies each entry into `System` properties, sets `logfile.name` for Log4j, and creates the static `logger`. Terminates the JVM (`System.exit(1)`) on any failure. |
| `private static String getStackTrace(Exception e)` | Converts an exception’s stack trace to a `String` for logging. |
| **External worker classes** (referenced but not defined here) | `APICallout`, `OrderStatus`, `BulkInsert`, `UsageProdDetails` – each implements a distinct business workflow. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | 1. `args[0]` – absolute path to a Java `.properties` file (e.g., `APIAcessMgmt.properties`).<br>2. `args[1]` – absolute path to the log file that Log4j should write to.<br>3. `args[2]` – execution mode (`workflow`, `status_update`, `bulk_insert`, `usage_product_details`). |
| **Outputs** | - Log entries written to the file defined by `args[1]` (via Log4j).<br>- Side‑effects produced by the invoked worker class (DB updates, API calls, file writes, etc.). |
| **Side Effects** | - System properties are populated for the entire JVM (used by downstream classes for DB URLs, credentials, API endpoints, etc.).<br>- Application may terminate the JVM (`System.exit`) on configuration errors.<br>- Potential external calls (HTTP, DB, SFTP) performed by the worker classes. |
| **Assumptions** | - The properties file exists, is readable, and contains all keys required by downstream components.<br>- Log4j is on the classpath and configured to honour the `logfile.name` system property.<br>- Exactly three arguments are supplied; otherwise an `ArrayIndexOutOfBoundsException` will be thrown (caught but silently ignored).<br>- Worker classes are compiled, on the classpath, and have no additional runtime arguments. |

---

## 4. Integration Points & Connectivity

| Connection | Description |
|------------|-------------|
| **Configuration Store** | `APIAcessMgmt.properties` (see `move-indiamed-api\APIAccessManagement\PropertyFiles\APIAcessMgmt.properties`). This file typically contains JDBC URLs, API endpoint URLs, authentication tokens, and other environment‑specific values. |
| **Logging** | Log4j configuration is external (`log4j.properties` located in `src` and `bin` directories). The runtime log file path is injected via the `logfile.name` system property. |
| **External Services** | - **APICallout**: likely invokes REST/SOAP services (e.g., partner APIs, provisioning systems).<br>- **OrderStatus**: may query an order‑management database or service.<br>- **BulkInsert**: writes large data sets to Hive, HBase, or a relational DB.<br>- **UsageProdDetails**: extracts usage data from mediation tables (e.g., Hive tables created by the DDL scripts in the HISTORY). |
| **Scheduler / Cron** | The class is designed to be invoked by a cron job or an orchestration tool (e.g., Oozie, Airflow) that supplies the three arguments. The commented‑out block shows a typical development‑time workspace path, indicating that production runs use absolute paths supplied by the scheduler. |
| **Other Java Modules** | The worker classes (`APICallout`, `OrderStatus`, etc.) likely depend on DAO classes such as `APIAccessDAO` (see `APIAccessDAO.properties`) for database connectivity. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing / malformed arguments** | Application starts, catches generic `Exception`, then silently exits without logging. | Validate `args.length == 3` at the top of `main`; log a clear error and exit with a non‑zero status. |
| **Property file not found or unreadable** | `logger` may be `null`, leading to a `NullPointerException` when logging the error. | Initialise `logger` **before** loading the properties (e.g., using a static Log4j config) or fallback to `System.err`. |
| **Silent catch in `main`** | Any runtime exception from worker classes is swallowed, making troubleshooting difficult. | Propagate the exception or at least log it with stack trace before exiting. |
| **System.exit on configuration error** | Abrupt termination may kill other threads in the same JVM (if run inside a container). | Throw a custom unchecked exception and let the container handle exit codes, or return an error code from `main`. |
| **Hard‑coded system property names** | Future changes to property keys require code changes. | Centralise property key constants in a dedicated config class. |
| **Log file path injection** | If the path is unwritable, Log4j may fallback to console, losing audit trail. | Verify write permissions on the supplied log file directory during initialization. |

---

## 6. Running & Debugging the Application

### 6.1 Typical Production Invocation (via cron / scheduler)

```bash
java -cp "/opt/move-indiamed-api/lib/*:/opt/move-indiamed-api/APIAccessManagement/target/classes" \
     com.tcl.api.callout.APIMain \
     /opt/move-indiamed-api/APIAccessManagement/PropertyFiles/APIAcessMgmt.properties \
     /var/log/move-indiamed-api/Accessmgmt.log \
     workflow
```

- **Classpath** must include compiled classes and all third‑party JARs (Log4j, JDBC drivers, HTTP client libs, etc.).
- The three arguments correspond to *property file*, *log file*, and *mode*.

### 6.2 Local Development / Debugging

1. **Set up IDE** (Eclipse/IntelliJ) with the project source root `src`.
2. **Create a Run Configuration**:
   - Main class: `com.tcl.api.callout.APIMain`
   - Program arguments:  
     ```
     ${workspace_loc:/APIAccessManagement/PropertyFiles/APIAcessMgmt.properties}
     ${workspace_loc:/APIAccessManagement/APICronLogs/Accessmgmt.log}
     workflow
     ```
3. **Breakpoints** can be placed in `initializeApp` to verify that all expected system properties are loaded.
4. **Inspect Log4j output** – ensure `logfile.name` is resolved correctly; the log file should appear under the specified path.
5. **Unit‑test worker classes** independently (e.g., `APICalloutTest`) to isolate failures from the bootstrap logic.

### 6.3 Debugging Tips

- If `logger` is `null` during property loading, add a temporary `System.err.println` before the `try` block.
- Use `-Dlog4j.debug=true` JVM flag to see Log4j configuration steps.
- Verify that the properties file contains all keys required by downstream DAOs (compare against `APIAccessDAO.properties`).

---

## 7. External Configuration & Environment Variables

| File / Variable | Purpose |
|-----------------|---------|
| **`APIAcessMgmt.properties`** (passed as `args[0]`) | Holds key/value pairs that become JVM system properties (e.g., `db.url`, `api.endpoint`, `auth.token`). |
| **`logfile.name`** (system property set in `initializeApp`) | Used by Log4j’s `RollingFileAppender` (or similar) to direct logs to the supplied file path. |
| **`log4j.properties`** (on classpath) | Defines log levels, appenders, and pattern layouts. The property file may reference `${logfile.name}`. |
| **`APIAccessDAO.properties`** (referenced by DAO classes) | Provides DB connection details; values are typically read via `System.getProperty`. |
| **Environment variables** (not directly used here) | May be referenced inside the property files (e.g., `${ENV_HOME}`) – verify at runtime. |

---

## 8. Suggested Improvements (TODO)

1. **Robust Argument Validation & Error Reporting**  
   ```java
   if (args == null || args.length != 3) {
       System.err.println("Usage: APIMain <propertyFile> <logFile> <mode>");
       System.exit(2);
   }
   ```
   Add explicit validation and meaningful exit codes.

2. **Replace Silent Catch in `main` with Logging**  
   ```java
   } catch (Exception e) {
       logger.error("Unhandled exception in APIMain", e);
       System.exit(3);
   }
   ```
   This ensures any unexpected failure is captured in the audit log.

---