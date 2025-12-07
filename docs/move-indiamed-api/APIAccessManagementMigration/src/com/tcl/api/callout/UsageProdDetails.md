**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\UsageProdDetails.java`

---

## 1. High‑Level Summary
`UsageProdDetails` extracts a list of distinct account numbers from a SQL Server table, calls the Geneva Order Management “usageDetails” REST endpoint for each account, parses the JSON response into domain objects (`UsageResponse`, `UsageData`, `UsageDetail`), and up‑serts the usage records into a raw‑data staging table via `RawDataAccess`. The class is a thin orchestration component that bridges the internal DB, an external HTTP API, and the internal DAO layer.

---

## 2. Key Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`UsageProdDetails`** (public class) | Main orchestrator for the usage‑details pull. |
| `usagePrdDetails()` (public void) | - Reads account numbers from DB (`fetch.dist.acnt`). <br> - Builds the API URL per account. <br> - Executes an HTTP GET with required headers. <br> - Deserialises JSON into model objects using Gson. <br> - Calls `RawDataAccess.rowExists()`, `updateRow()`, `insertRow()` to persist each usage detail. |
| **`SQLServerConnection`** (external) | Provides a JDBC `Connection` to the production SQL Server. |
| **`RawDataAccess`** (external) | DAO that abstracts existence‑check, insert, and update of usage rows. |
| **`APIConstants`** (external) | Holds static header names/values (`CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE`). |
| **`UsageResponse`, `UsageData`, `UsageDetail`** (model) | POJOs matching the JSON payload returned by the usage‑details service. |
| **`ResourceBundle daoBundle`** | Loads `APIAccessDAO.properties` to obtain the SQL query string `fetch.dist.acnt`. |

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Inputs** | • JDBC connection (SQL Server). <br>• `APIAccessDAO.properties` entry `fetch.dist.acnt` (SQL that returns `accountnumber`). <br>• HTTP GET request to `https://api-uat.tatacommunications.com:443/GenevaOrderManagementAPI/v1/usageDetails/{accountNumber}`. |
| **Outputs** | • No direct return value. <br>• Rows inserted/updated in the raw‑data staging table via `RawDataAccess`. |
| **Side‑Effects** | • Network traffic to the external API (potentially many calls). <br>• DB writes (insert/update). <br>• Log entries via Log4j. |
| **Assumptions** | • `APIAccessDAO.properties` is on the classpath and contains a valid SELECT that returns a column named `accountnumber`. <br>• `SQLServerConnection.getJDBCConnection()` returns a live connection with appropriate credentials. <br>• API authentication headers defined in `APIConstants` are valid for the UAT environment. <br>• The JSON schema matches the model classes. |

---

## 4. Integration Points & Call Flow

1. **Caller / Scheduler** – Not present in this file; typically invoked by a batch scheduler (e.g., Spring Batch, Control‑M, or a custom shell script) that creates an instance of `UsageProdDetails` and calls `usagePrdDetails()`.
2. **Database** – Reads from the table referenced by `fetch.dist.acnt`; writes to the raw‑data table via `RawDataAccess`.
3. **External REST Service** – Calls the Geneva Order Management API (UAT endpoint). Header values are supplied from `APIConstants`.
4. **Configuration** – `APIAccessDAO.properties` (SQL query) and `log4j.properties` (logging). No environment variables are referenced directly, but the underlying `SQLServerConnection` may read DB credentials from environment or a separate config file.
5. **Model Layer** – `UsageResponse`, `UsageData`, `UsageDetail` are shared with other callout classes (e.g., `ProductDetails`, `OrderStatus`) that also consume API responses.

---

## 5. Operational Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Unbounded API calls** – One HTTP request per account; large account set can overwhelm the API or cause long runtimes. | Add pagination or batch limits; schedule during low‑traffic windows; implement a configurable max‑records per run. |
| **No retry / timeout handling** – Transient network failures cause the whole run to abort or skip accounts. | Configure `RequestConfig` with connection/read timeouts; wrap the HTTP call in a retry loop with exponential back‑off. |
| **Potential resource leaks** – `PreparedStatement` and `ResultSet` are not explicitly closed. | Use try‑with‑resources for JDBC objects; ensure `stmt` and `rs` are closed in a finally block. |
| **Hard‑coded API URL** – UAT URL baked into code; moving to production requires code change. | Externalize the base URL (e.g., `api.base.url` in a properties file or environment variable). |
| **Logging of sensitive data** – Account numbers and full JSON payload are logged at INFO level. | Reduce log level for payloads; mask or omit PII; follow data‑privacy guidelines. |
| **No pagination handling in API response** – If the API returns a limited set per call, missing usage data may go unnoticed. | Verify API contract; add logic to follow “next” links or request larger page sizes. |
| **SQL injection risk** – The query is loaded from a properties file; if it contains placeholders, they are not parameterised. | Ensure the query is static and does not embed user‑controlled values; otherwise use prepared‑statement parameters. |

---

## 6. Running / Debugging the Component

1. **Prerequisites**  
   - Java 8+ runtime.  
   - `log4j.properties` on the classpath (set `log4j.rootLogger=DEBUG, console` for verbose output).  
   - `APIAccessDAO.properties` containing `fetch.dist.acnt=SELECT accountnumber FROM <table>`.  
   - Database credentials reachable via `SQLServerConnection`.  
   - Network access to `api-uat.tatacommunications.com` on port 443.

2. **Typical Invocation** (from a Java main or batch framework):
   ```java
   public static void main(String[] args) throws Exception {
       new UsageProdDetails().usagePrdDetails();
   }
   ```
   Compile and run the class, or call it from an existing batch job that already creates the Spring/Application context.

3. **Debug Steps**  
   - Enable DEBUG logging for `com.tcl.api.callout.UsageProdDetails`.  
   - Verify the SQL query by running it directly against the DB.  
   - Use a tool like `curl` to manually hit the API endpoint with a known account number and compare the JSON to the model classes.  
   - Set breakpoints on `rda.rowExists(...)` to confirm insert vs. update paths.  
   - After execution, query the raw‑data table to confirm rows were persisted.

4. **Error Handling**  
   - All caught exceptions are printed to `stderr`. For production, replace `e.printStackTrace()` with proper Log4j error logging and optionally re‑throw a custom exception to signal batch failure.

---

## 7. External Configuration & Environment Dependencies

| Item | Usage |
|------|-------|
| **`APIAccessDAO.properties`** | Provides the SQL string `fetch.dist.acnt`. Must be on the classpath. |
| **`APIConstants`** | Supplies HTTP header names and static values (`CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE`). |
| **`log4j.properties`** | Controls logging output; adjust levels for troubleshooting. |
| **Database credentials** | Likely sourced inside `SQLServerConnection` (could be from environment variables, a separate `db.properties`, or a secrets manager). |
| **Network / proxy settings** | If the environment requires an HTTP proxy, the `HttpClients.createDefault()` call will need to be configured accordingly. |

---

## 8. Suggested Improvements (TODO)

1. **Externalize the API base URL and authentication credentials** – Move `apiUrl` and header values into a properties file or secure vault so the same code can run against UAT, QA, and Production without recompilation.

2. **Add robust retry and timeout logic** – Wrap the HTTP call in a retry policy (e.g., Apache HttpClient `RequestConfig` + a simple exponential back‑off loop) and enforce connection/read timeouts to avoid hanging on slow or unavailable endpoints.

*(Additional optional improvements: convert JDBC handling to try‑with‑resources, implement pagination handling if the API supports it, and replace `System.out`/`printStackTrace` with structured Log4j error logging.)*