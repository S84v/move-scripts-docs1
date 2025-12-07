**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\UsageProdDetails2.java`

---

## 1. High‑Level Summary
`UsageProdDetails2` is a stand‑alone Java utility that extracts a list of account numbers from a SQL Server database, calls the Tata Communications “usageDetails” REST endpoint for each account, and persists the returned usage records into a local “raw data” table.  It performs an up‑sert (update if the row already exists, otherwise insert) for every usage detail received.  The class is intended to be executed on a schedule (e.g., via a cron job) to keep the internal usage repository synchronized with the external API.

---

## 2. Key Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`UsageProdDetails2` (class)** | Entry point (`main`). Orchestrates configuration loading, DB access, HTTP calls, JSON parsing, and DB up‑serts. |
| **`main(String[] args)`** | - Loads `APIAcessMgmt.properties` and sets them as system properties.<br>- Configures Log4j file name.<br>- Retrieves the list of accounts (`fetch.dist.acnt` query) from the `APIAccessDAO` resource bundle.<br>- For each account: builds the API URL, issues an HTTP GET, parses the JSON response into `UsageResponse` → `UsageData` → `List<UsageDetail>`.<br>- Calls `RawDataAccess.rowExists`, `updateRow`, or `insertRow` to persist each usage detail. |
| **`SQLServerConnection.getJDBCConnection()`** (external) | Provides a JDBC `Connection` to the production SQL Server. |
| **`RawDataAccess`** (external DAO) | Helper that abstracts `rowExists`, `updateRow`, and `insertRow` operations on the raw usage table. |
| **`APIConstants`** (external) | Holds static header names/values (`CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE`). |
| **Model classes (`UsageResponse`, `UsageData`, `UsageDetail`)** (external) | POJOs used by Gson to deserialize the API payload. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • `APIAcessMgmt.properties` (workspace‑relative) – defines system properties such as DB credentials, log level, etc.<br>• `APIAccessDAO.properties` (resource bundle) – provides the SQL query key `fetch.dist.acnt`.<br>• Database table(s) referenced by the query (account list) and the raw usage table (target of up‑serts). |
| **External Services** | • **SQL Server** – read account numbers, write usage rows.<br>• **HTTPS REST API** – `https://api-uat.tatacommunications.com:443/GenevaOrderManagementAPI/v1/usageDetails/{accountNumber}` – returns JSON usage payload.<br>• **Log4j** – writes operational logs to `APICronLogs\Accessmgmt.log`. |
| **Outputs** | • Updated/inserted rows in the raw usage table.<br>• Log entries (INFO/ERROR) in the configured log file.<br>• Console stack traces on uncaught exceptions (via `e.printStackTrace()`). |
| **Side Effects** | • Alters system properties for the JVM process (affects any downstream code that reads `System.getProperty`).<br>• Opens a JDBC connection and an HTTP client per execution; current code closes them after the loop but not in a `finally` block. |
| **Assumptions** | • The workspace path (`E:\\eclipse-workspace\\APIAccessManagement\\`) is valid on the host machine.<br>• The properties file contains all required keys (DB URL, user, password, etc.).<br>• The API returns a well‑formed JSON matching the model classes.<br>• No pagination or rate‑limit handling is required for the usage endpoint. |

---

## 4. Integration Points & Connectivity

| Component | Interaction |
|-----------|-------------|
| **`UsageProdDetails.java`** (sibling) | Likely an earlier version; both scripts share the same DAO and model classes. `UsageProdDetails2` may replace or run in parallel with it. |
| **Cron Scheduler / Job Runner** | The class is intended to be invoked by an external scheduler (e.g., Windows Task Scheduler, cron, or an enterprise job orchestrator). The scheduler must set the working directory so the relative paths resolve. |
| **`RawDataAccess` DAO** | Central persistence layer used by many call‑out scripts (e.g., `PostOrder`, `OrderStatus`). Any schema change in the raw usage table impacts this script. |
| **`SQLServerConnection`** | Shared connection factory used across the API Access Management module. |
| **`APIAccessDAO.properties`** | Provides the SQL query; other call‑out classes also reference this bundle for their queries. |
| **Log4j configuration** | The log file name is set at runtime; other modules may rely on the same logging directory. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Hard‑coded workspace path** | Fails on any server where the path differs. | Externalize the base path via an environment variable or a property (`workspace.path`). |
| **Resource leaks (HTTP client, DB connection)** | Potential exhaustion under long‑running or frequent executions. | Use try‑with‑resources or finally blocks to guarantee `close()` on `CloseableHttpClient`, `CloseableHttpResponse`, and `Connection`. |
| **No retry / back‑off on HTTP failures** | Transient network glitches cause data gaps. | Implement a retry policy (e.g., 3 attempts with exponential back‑off) for non‑2xx responses. |
| **No pagination handling** | Large accounts may return truncated data, leading to incomplete usage records. | Verify API contract; if pagination exists, loop until all pages are consumed. |
| **Uncontrolled exception handling (`printStackTrace`)** | Stack traces may be lost in logs or flood console. | Log exceptions via Log4j (`logger.error("...", e)`) and allow the process to continue or fail gracefully. |
| **Direct `System.setProperty` for all entries** | Overwrites JVM‑wide properties, possibly affecting other threads or libraries. | Load only required properties into a dedicated configuration object; avoid polluting the global system properties. |
| **No transaction management for up‑serts** | Partial updates if a failure occurs mid‑loop. | Wrap each account’s up‑sert operations in a DB transaction (`connection.setAutoCommit(false)`) and commit/rollback appropriately. |

---

## 6. Running & Debugging the Script

### 6.1 Prerequisites
1. Java 8+ installed on the execution host.  
2. All dependent JARs on the classpath (Log4j, Apache HttpClient, Gson, SQL Server JDBC driver, internal `tcl.api.*` libraries).  
3. `APIAcessMgmt.properties` and `APIAccessDAO.properties` placed under the workspace directory as expected.  
4. Database credentials correctly defined in the properties file.  

### 6.2 Execution Command
```bash
# Example (Windows)
set CLASSPATH=.;lib/*   # include all required JARs
java com.tcl.api.callout.UsageProdDetails2
```
*If the script is invoked by a scheduler, ensure the working directory is the root of the workspace (`E:\eclipse-workspace\APIAccessManagement\`).*

### 6.3 Debugging Steps
1. **Increase Log Level** – edit `log4j.properties` (or set `log4j.rootLogger=DEBUG, FILE`) to capture detailed HTTP request/response data.  
2. **Run in IDE** – set the **Program Arguments** empty, configure the **Working Directory** to the workspace root, and add a breakpoint inside the `while (rs.next())` loop.  
3. **Validate SQL Query** – run the `fetch.dist.acnt` query directly against the DB to confirm it returns the expected `accountnumber` column.  
4. **Inspect API Response** – temporarily log the raw JSON (`logger.debug("Raw JSON: " + temp);`) before deserialization.  
5. **Check DAO Operations** – enable DEBUG logging for `RawDataAccess` to see generated SQL for `INSERT`/`UPDATE`.  

---

## 7. External Configuration & Files

| File | Purpose | How It Is Used |
|------|---------|----------------|
| **`APIAcessMgmt.properties`** | Holds environment‑specific settings (DB URL, user, password, log level, etc.). | Loaded at start, each entry is copied into `System` properties. |
| **`APIAccessDAO.properties`** (resource bundle `APIAccessDAO`) | Contains SQL statements referenced by keys. | `daoBundle.getString("fetch.dist.acnt")` supplies the query that fetches account numbers. |
| **Log4j configuration** (implicit) | Determines log formatting and destination. | `System.setProperty("logfile.name", logFile)` tells Log4j where to write the cron log. |
| **Model POJOs (`UsageResponse`, `UsageData`, `UsageDetail`)** | Represent the JSON payload structure. | Gson deserializes the API response into these objects. |
| **`SQLServerConnection` class** | Centralised JDBC connection factory. | Provides the `Connection` used for both the account query and the up‑sert operations. |

---

## 8. Suggested Improvements (TODO)

1. **Refactor Configuration Loading** – Replace the manual `FileInputStream` + `System.setProperty` pattern with a dedicated configuration class (e.g., using Apache Commons Configuration) that supplies typed values and avoids polluting the global system properties.

2. **Introduce Robust Resource Management** – Convert the main processing loop to use try‑with‑resources for `CloseableHttpClient`, `CloseableHttpResponse`, and `Connection`. Also, wrap each account’s DB updates in a transaction with proper commit/rollback handling.

---