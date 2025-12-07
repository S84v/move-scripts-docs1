**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\UsageProdDetails.java`  

---

### 1. High‑level Summary
`UsageProdDetails` implements the “usage production details” step of the API Access Management move process. It reads a list of account numbers from a SQL Server table, calls the external “usage details” REST endpoint for each account, parses the JSON payload into domain objects, and up‑serts the resulting usage records into a raw‑data staging table via `RawDataAccess`. The class is invoked by the orchestration layer (e.g., a scheduler or a higher‑level callout) and logs progress and HTTP error codes through Log4j.

---

### 2. Key Classes & Functions  

| Element | Responsibility |
|---------|-----------------|
| **`UsageProdDetails`** (class) | Orchestrates the end‑to‑end flow: DB read → HTTP GET → JSON → DB up‑sert. |
| **`usagePrdDetails()`** (public method) | Core routine; loops over accounts, performs the API call, and persists usage details. |
| **`ResourceBundle daoBundle`** | Loads `APIAccessDAO.properties` to obtain the SQL query `fetch.dist.acnt`. |
| **`SQLServerConnection.getJDBCConnection()`** | Provides a live JDBC connection to the production SQL Server (external utility). |
| **`RawDataAccess`** | Helper DAO with `rowExists()`, `updateRow()`, `insertRow()` – abstracts raw‑data table operations. |
| **`APIConstants`** | Holds static configuration: base URL (`USAGE_ATTR_URL`), header names/values (`CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE`). |
| **`UsageResponse`, `UsageData`, `UsageDetail`** (model classes) | POJOs used by Gson to deserialize the usage‑details JSON payload. |
| **`Logger` (Log4j)** | Emits informational and error messages; used throughout the method. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • SQL query `fetch.dist.acnt` → returns rows with column `accountnumber`.<br>• External REST endpoint: `APIConstants.USAGE_ATTR_URL + <accountNumber>`.<br>• HTTP headers defined in `APIConstants` (content‑type, authentication). |
| **Outputs** | • Inserts or updates rows in the raw‑data usage table (via `RawDataAccess`).<br>• Log entries (INFO/ERROR) written to the configured Log4j appenders. |
| **Side Effects** | • Network I/O (HTTPS GET).<br>• Database I/O (SELECT, INSERT, UPDATE).<br>• Potentially long‑running loop if many accounts. |
| **Assumptions** | • `APIAccessDAO.properties` is on the classpath and contains a valid `fetch.dist.acnt` query.<br>• `APIConstants` supplies a reachable URL and valid authentication headers.<br>• The JSON response conforms exactly to the `UsageResponse` model hierarchy.<br>• The DB schema matches the fields used in `RawDataAccess`. |

---

### 4. Integration Points  

| Connected Component | Interaction |
|---------------------|-------------|
| **`SQLServerConnection`** | Provides the JDBC connection; likely reads DB credentials from environment variables or a separate config file. |
| **`RawDataAccess`** | Shared DAO used by other callout classes (e.g., `ProductDetails`, `OrderStatus`). Updates the same raw‑data staging area. |
| **`AlertEmail`** (previous file) | Not directly invoked here, but typical error paths (e.g., unrecoverable HTTP errors) could be extended to send alerts via this utility. |
| **Orchestration / Scheduler** | A cron job, Spring Batch job, or custom “move” engine triggers `new UsageProdDetails().usagePrdDetails();`. |
| **`APIConstants`** | Centralised constants file used by many callouts for endpoint URLs and header values. |
| **`UsageProdDetails`** may be called after `ProductDetails` or before `PostOrder` depending on the overall move workflow (the exact order is defined in the higher‑level orchestration script). |

---

### 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Unbounded loop / API throttling** – large account set may exceed rate limits. | Add pagination or batch size limits; respect `Retry-After` header; implement a back‑off strategy. |
| **Transient network failures** – HTTP request may time‑out or drop. | Wrap the call in a retry block (e.g., 3 attempts with exponential back‑off). |
| **Resource leaks** – `PreparedStatement`/`ResultSet` not closed. | Use try‑with‑resources for JDBC objects; ensure they are closed in a finally block. |
| **JSON schema mismatch** – API change could break deserialization. | Validate the response schema before parsing; log and alert on `JsonSyntaxException`. |
| **Database contention** – many concurrent up‑serts could lock rows. | Use batch up‑serts where supported; consider using `MERGE` statements in the DAO. |
| **Missing/invalid configuration** – absent `APIAccessDAO.properties` or wrong constants. | Fail fast with a clear log message; add a health‑check step before execution. |

---

### 6. Running / Debugging the Class  

1. **Compile** – The project uses Maven/Gradle (consistent with other callouts). Ensure dependencies (`log4j`, `gson`, `httpclient`, `sqlserver-jdbc`) are present.  
   ```bash
   mvn clean compile
   ```
2. **Execute** – The class has no `main`; it is normally invoked by the orchestration layer. For ad‑hoc testing you can add a temporary driver:  
   ```java
   public static void main(String[] args) throws Exception {
       new UsageProdDetails().usagePrdDetails();
   }
   ```
   Then run:  
   ```bash
   mvn exec:java -Dexec.mainClass=com.tcl.api.callout.UsageProdDetails
   ```
3. **Logging** – Adjust `log4j.properties` to set `log4j.rootLogger=DEBUG, console` for verbose output.  
4. **Debugging** – Set breakpoints inside the `while (rs.next())` loop to inspect:  
   * `accountNumber` value,  
   * Constructed `usageURL`,  
   * HTTP status code,  
   * Parsed `UsageResponse` object.  
   Verify that `RawDataAccess.rowExists` behaves as expected (mock the DAO if needed).  
5. **Verification** – After run, query the raw‑data usage table to confirm rows were inserted/updated.  

---

### 7. External Configuration & Environment Variables  

| Config Item | Source | Usage |
|-------------|--------|-------|
| **`APIAccessDAO.properties`** (ResourceBundle) | Classpath file | Provides SQL query `fetch.dist.acnt`. |
| **`APIConstants`** | Java class (static fields) | Supplies base URL, header names, and authentication values (likely populated from environment variables or a secure vault). |
| **Database credentials** | `SQLServerConnection` (internal utility) | Typically read from environment variables (`DB_URL`, `DB_USER`, `DB_PASS`) or a protected properties file. |
| **Log4j configuration** | `log4j.properties` on classpath | Controls logging output and destinations. |

---

### 8. Suggested Improvements (TODO)

1. **Resource Management** – Refactor JDBC handling to use try‑with‑resources for `Connection`, `PreparedStatement`, and `ResultSet` to guarantee closure and avoid leaks.  
2. **Resilience & Observability** – Introduce a retry policy (e.g., using Apache Commons Retry or Spring Retry) for the HTTP call and add metrics (success count, failure count, latency) to a monitoring system (Prometheus, Splunk, etc.).  

---