**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\UsageProdDetails2.java`

---

## 1. High‑Level Summary
`UsageProdDetails2` is a stand‑alone Java utility that extracts a list of account numbers from a SQL Server database, calls the Geneva Order Management “usageDetails” REST endpoint for each account, parses the JSON response into domain objects, and upserts the resulting usage records into a staging table via `RawDataAccess`. It is intended to be executed as a scheduled job (cron) that keeps the internal usage‑detail store synchronized with the external API.

---

## 2. Key Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **UsageProdDetails2** (public class) | Entrypoint containing `main`. Orchestrates configuration loading, DB access, HTTP calls, JSON deserialization, and DB upserts. |
| `main(String[] args)` | - Loads `APIAcessMgmt.properties` and sets them as system properties.<br>- Configures Log4j logger.<br>- Retrieves the base API URL (hard‑coded).<br>- Opens a JDBC connection via `SQLServerConnection`.<br>- Executes the SQL query from `APIAccessDAO` (`fetch.dist.acnt`).<br>- For each account: builds request URL, performs HTTP GET, parses response (`Gson` → `UsageResponse`).<br>- Calls `RawDataAccess.rowExists`, `updateRow`, `insertRow` for each `UsageDetail`. |
| **ResourceBundle daoBundle** | Loads SQL statements from `APIAccessDAO.properties`. |
| **RawDataAccess rda** | DAO helper that abstracts `rowExists`, `updateRow`, `insertRow` for usage detail persistence. |
| **SQLServerConnection.getJDBCConnection()** | Provides a live `java.sql.Connection` to the production SQL Server. |
| **APIConstants** | Holds static header names/values (`CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE`). |
| **Model classes** (`UsageResponse`, `UsageData`, `UsageDetail`) | POJOs generated (or hand‑written) to match the JSON payload returned by the external API. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Description |
|----------|-------------|
| **Inputs** | 1. `APIAcessMgmt.properties` (file system path hard‑coded).<br>2. `APIAccessDAO.properties` (resource bundle on classpath) – provides the SQL query `fetch.dist.acnt`.<br>3. System properties set from the above file (e.g., DB credentials, proxy settings). |
| **External Services** | - **SQL Server** (via JDBC) – read account numbers, write usage rows.<br>- **Geneva Order Management API** (`https://api-uat.tatacommunications.com:443/GenevaOrderManagementAPI/v1/usageDetails/{account}`) – HTTP GET, JSON response.<br>- **Log4j** – writes to `APICronLogs\Accessmgmt.log`. |
| **Outputs** | - Log entries (INFO/ERROR) in the configured log file.<br>- Updated/Inserted rows in the usage‑detail staging table (via `RawDataAccess`). |
| **Side Effects** | - Mutates DB state (upserts usage rows).<br>- Consumes network bandwidth and API rate limits.<br>- May create temporary HTTP connections that are closed per iteration. |
| **Assumptions** | - The property file exists at the hard‑coded workspace path.<br>- The `APIAccessDAO` bundle contains a valid `fetch.dist.acnt` query returning column `accountnumber`.<br>- API authentication is static (header values from `APIConstants`).<br>- The JSON schema matches the model classes exactly.<br>- No pagination or throttling is required for the API. |

---

## 4. Integration Points & Connectivity

| Component | How `UsageProdDetails2` Connects |
|-----------|-----------------------------------|
| **Other Callout Scripts** (e.g., `UsageProdDetails.java`) | Likely a newer version; both share the same DAO and model packages. The cron schedule that runs `UsageProdDetails2` may be coordinated with other scripts to avoid overlapping DB writes. |
| **Cron Scheduler / Job Runner** | Executed as a Java class from a scheduled task (Windows Task Scheduler or Linux cron) that invokes `java -cp <classpath> com.tcl.api.callout.UsageProdDetails2`. |
| **SQLServerConnection** | Centralised connection factory used across the migration package; shares DB pool configuration defined in the properties file. |
| **RawDataAccess** | Shared DAO used by multiple callout classes (`BulkInsert`, `PostOrder`, etc.) to persist raw data. |
| **APIConstants** | Central constants file used by all HTTP callout classes for header management. |
| **Logging Infrastructure** | Log4j configuration (likely `log4j.properties` on classpath) reads `logfile.name` system property to route output. |
| **Configuration Management** | All environment‑specific values (DB URL, credentials, API base URL) are expected to be supplied via the properties file; other scripts read the same file, ensuring consistency. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Hard‑coded workspace paths** (`E:\\eclipse-workspace\\...`) | Breaks when deployed to another server or OS. | Externalize the base path via an environment variable or a property entry; use `Path` APIs for OS‑independent handling. |
| **Per‑iteration HTTP client shutdown** (`httpclient.close()` inside the loop) | Re‑creates TCP connections for every account → performance degradation, possible socket exhaustion. | Create a single `CloseableHttpClient` outside the loop and close it after processing all accounts (or use a connection pool). |
| **No try‑with‑resources / resource leaks** (ResultSet, PreparedStatement, Connection) | Potential DB connection leaks under error conditions. | Wrap JDBC resources in try‑with‑resources; ensure `finally` blocks close them. |
| **Static API authentication** (header values from `APIConstants`) | If credentials rotate, code must be recompiled. | Load auth token/credentials from the properties file or a secret manager at runtime. |
| **No pagination / rate‑limit handling** | Large account sets may cause API throttling or incomplete data. | Implement pagination support (if API provides) and back‑off/retry logic based on HTTP 429 or custom headers. |
| **Silent JSON parsing failures** (Gson may throw if schema changes) | Data loss or runtime exceptions not captured in logs. | Validate JSON against a schema, catch `JsonSyntaxException`, and log problematic payloads for later analysis. |
| **Logging of raw response (`temp`)** | May expose PII or large payloads in log files. | Mask sensitive fields before logging or log only summary information. |
| **Hard‑coded API URL (`api-uat`)** | Production runs may inadvertently hit UAT environment. | Parameterise the base URL via properties and enforce environment‑specific configuration. |

---

## 6. Running & Debugging the Script

### 6.1 Prerequisites
1. Java 8+ runtime and the compiled JAR/classpath containing all dependent packages (`SQLServerConnection`, `RawDataAccess`, model classes, Apache HttpClient, Gson, Log4j).
2. `APIAcessMgmt.properties` placed at `E:\eclipse-workspace\APIAccessManagement\PropertyFiles\`.
3. `APIAccessDAO.properties` (or equivalent) on the classpath with key `fetch.dist.acnt`.
4. Network connectivity to the SQL Server and the Geneva API endpoint.

### 6.2 Execution Command
```bash
# Example (Windows)
java -cp "lib/*;target/classes" com.tcl.api.callout.UsageProdDetails2
```
*Adjust the classpath (`lib/*`) to include all third‑party JARs.*

### 6.3 Debugging Steps
| Step | Action |
|------|--------|
| **Enable Verbose Logging** | Set Log4j level to `DEBUG` in `log4j.properties` or via system property `log4j.rootLogger=DEBUG, FILE`. |
| **Validate Property Loading** | After startup, check the log for each `System.setProperty` entry; confirm DB URL, credentials, and API base URL are correct. |
| **Inspect SQL Query** | Verify the query retrieved from `APIAccessDAO` by logging `sql`. Run it manually against the DB to ensure it returns `accountnumber`. |
| **Check HTTP Response** | If status codes other than 200 appear, the log will show them. Use a tool like `curl` with the same URL and headers to reproduce. |
| **Catch JSON Errors** | Wrap `gson.fromJson` in a try‑catch for `JsonSyntaxException` and log the raw payload for analysis. |
| **Resource Leak Detection** | Run the JVM with `-XX:+HeapDumpOnOutOfMemoryError` and analyze heap dumps for unclosed `Connection` or `HttpClient` objects. |
| **Unit Test Isolation** | Mock `SQLServerConnection`, `RawDataAccess`, and the HTTP client to verify business logic without external dependencies. |

---

## 7. External Configuration & Files

| File | Purpose | How Used |
|------|---------|----------|
| **APIAcessMgmt.properties** | Holds environment‑specific values (DB URL, username, password, possibly proxy settings). | Loaded at start; each entry is copied into `System` properties for downstream components (`SQLServerConnection`, `APIConstants`). |
| **APIAccessDAO.properties** (resource bundle `APIAccessDAO`) | Stores SQL statements, notably `fetch.dist.acnt`. | Retrieved via `ResourceBundle.getBundle("APIAccessDAO")` and used to prepare the `PreparedStatement`. |
| **log4j.properties** (or equivalent) | Configures Log4j appenders, pattern, and log level. | The script sets `logfile.name` system property to direct output to `Accessmgmt.log`. |
| **Model POJOs** (`UsageResponse`, `UsageData`, `UsageDetail`) | Represent the JSON payload structure. | Deserialized by Gson; fields must match the API response. |
| **APIConstants.java** | Centralised HTTP header names/values. | Used to set request headers for every API call. |

---

## 8. Suggested Improvements (TODO)

1. **Refactor Resource Management**  
   - Use *try‑with‑resources* for `CloseableHttpClient`, `CloseableHttpResponse`, `Connection`, `PreparedStatement`, and `ResultSet`.  
   - Move `httpclient.close()` outside the `while (rs.next())` loop to reuse the connection pool.

2. **Externalise Environment‑Specific Values**  
   - Replace hard‑coded workspace paths and the base API URL with entries in `APIAcessMgmt.properties` (e.g., `workspace.path`, `api.base.url`).  
   - Add validation at startup to fail fast if required properties are missing.

*Additional optional enhancements*: implement pagination/back‑off for the API, batch DB upserts, and add unit tests with mocks for DAO and HTTP client.

---