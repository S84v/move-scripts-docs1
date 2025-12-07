**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\OrderStatus.java`

---

## 1. High‑Level Summary
`OrderStatus` implements the “order‑creation status” polling step of the API Access Management Migration job. For each *input group* retrieved from the staging database, it calls the Geneva Order Management REST endpoint, parses the JSON response, builds a map of row‑level status strings, and writes the results back to the database in bulk. The class is invoked by the main orchestration script (e.g., `APIMain`) and runs as part of the nightly data‑move pipeline.

---

## 2. Key Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`OrderStatus`** (public class) | Encapsulates the end‑to‑end status‑check workflow. |
| **`orderStatusCheck()`** (public boolean) | *Core method* – fetches input group IDs, iterates over them, performs HTTP GET to the order‑status API, transforms the JSON payload into domain objects, aggregates per‑row status, and persists the aggregated map via `RawDataAccess.bulkStatusUpdate`. Returns `true` on completion; propagates DB and IO exceptions. |
| **`RawDataAccess`** (used) | DAO layer that supplies `fetchInputGroupId()` (SELECT of pending groups) and `bulkStatusUpdate(String groupId, Map<String,Tuple2>)` (batch UPDATE of status). Defined elsewhere in `com.tcl.api.dao`. |
| **`APIConstants`** (used) | Holds static header names/values (`CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE`). Populated from property files (e.g., `APIAcessMgmt.properties`). |
| **Model classes** (`GetOrderResponse`, `OrderResponse`, `OrderStatusList`) | POJOs generated from the API JSON schema; used by Gson for deserialization. |
| **`Tuple2`** (used) | Simple container for a pair of values – here `(statusString, groupValue)`. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • List of *input group IDs* from `RawDataAccess.fetchInputGroupId()` (each `Tuple2` holds `key = inputGroupId`, `value = groupValue`).<br>• HTTP GET request to `https://api-uat.tatacommunications.com:443/GenevaOrderManagementAPI/v1/orderCreationstatus/{inputGroupId}`.<br>• Header values from `APIConstants` (content‑type, authentication). |
| **Outputs** | • Bulk update of order‑status rows via `RawDataAccess.bulkStatusUpdate(inputGroupId, hm)` where `hm` maps `inputRowId → Tuple2(statusString, groupValue)`. |
| **Side Effects** | • Network traffic to the external Geneva Order Management API.<br>• Database writes (status updates).<br>• Log entries via Log4j (error levels for HTTP failures). |
| **Assumptions** | • The staging DB contains a non‑empty set of pending input groups.<br>• API endpoint is reachable from the execution host and the auth token in `APIConstants.ACNT_VALUE` is valid.<br>• JSON response conforms to the expected model hierarchy.<br>• No pagination or throttling is required for the number of groups processed. |

---

## 4. Integration Points & Call Flow

1. **Up‑stream** – `APIMain` (or another orchestrator) creates an instance of `OrderStatus` and invokes `orderStatusCheck()`.  
2. **DAO Layer** – `RawDataAccess` reads pending groups from the *raw* schema (`fetchInputGroupId`) and writes back status (`bulkStatusUpdate`). The SQL statements for these operations are defined in `APIAccessDAO.properties`.  
3. **Configuration** – Header values and possibly the base URL are sourced from `APIAcessMgmt.properties` via `APIConstants`.  
4. **External Service** – Calls the Geneva Order Management REST API (UAT environment).  
5. **Down‑stream** – After bulk update, subsequent scripts (e.g., `BulkInsert`, `AlertEmail`) may read the updated status to decide on further actions (e.g., notifying stakeholders or moving rows to final tables).

---

## 5. Operational Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Network/HTTP failures** (timeouts, DNS, SSL) | Configure `HttpClient` with reasonable timeouts, enable retry/back‑off, and wrap calls in a `try‑with‑resources` block. |
| **Resource leakage** (unclosed `CloseableHttpResponse` / `CloseableHttpClient`) | Use try‑with‑resources for both client and response; currently `close()` is called only in the happy path. |
| **Hard‑coded endpoint** | Externalize the base URL to a property file (e.g., `api.order.status.url`) and read via `APIConstants`. |
| **Large group list causing memory pressure** | Process groups in configurable batches; clear the `hm` map after each bulk update. |
| **Insufficient error handling** (only logs, continues loop) | Propagate a custom exception or record failed groups for later re‑processing; add alerting for non‑2xx responses. |
| **Schema drift** (API response structure changes) | Add defensive null checks and version the model classes; consider schema validation before deserialization. |

---

## 6. Running / Debugging the Class

1. **Typical Invocation** (no `main` method):  
   ```bash
   # From the job’s launch script (e.g., APIMain)
   java -cp lib/*:target/classes com.tcl.api.callout.APIMain
   ```
   `APIMain` will instantiate `OrderStatus` and call `orderStatusCheck()`.

2. **Standalone Test (for debugging)**  
   ```java
   public static void main(String[] args) throws Exception {
       OrderStatus os = new OrderStatus();
       os.orderStatusCheck();
   }
   ```
   Compile with the project classpath and run; enable Log4j DEBUG level to see HTTP request/response details.

3. **Debug Steps**  
   - Verify that `RawDataAccess.fetchInputGroupId()` returns expected rows (run the underlying SQL manually).  
   - Use a tool like `curl` or Postman to hit the API URL with the same headers to confirm response format.  
   - Set a breakpoint on `response.getStatusLine().getStatusCode()` to inspect non‑200 codes.  
   - Check the `bulkStatusUpdate` SQL in `APIAccessDAO.properties` to ensure the correct target table/columns.  
   - Review Log4j output (`log4j.properties`) for error messages; increase logger level to `DEBUG` if needed.

---

## 7. External Configuration & Environment Dependencies

| Config / Env | Usage |
|--------------|-------|
| **`APIAcessMgmt.properties`** (referenced by `APIConstants`) | Supplies `CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE` (authentication token). |
| **`log4j.properties`** | Controls logging output, levels, and appenders for this class. |
| **`APIAccessDAO.properties`** | Holds the SQL for `fetchInputGroupId` and `bulkStatusUpdate`. |
| **System properties / JVM args** (if any) | Not directly used in this file, but the overall job may rely on DB connection strings, proxy settings, etc., passed via environment variables. |

---

## 8. Suggested Improvements (TODO)

1. **Resource Management** – Refactor the HTTP client usage to a try‑with‑resources block and reuse a single `CloseableHttpClient` (or a connection pool) across all group iterations.  
2. **Configuration‑driven Endpoint** – Move the hard‑coded base URL (`https://api-uat.tatacommunications.com:443/...`) into a property (e.g., `order.status.base.url`) and read it via `APIConstants` or a dedicated config loader.  

---