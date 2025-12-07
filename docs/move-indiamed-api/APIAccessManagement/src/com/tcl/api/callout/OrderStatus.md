**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\OrderStatus.java`

---

## 1. High‑level Summary
`OrderStatus` implements the “order‑status polling” step of the API Access Management move process. It retrieves a list of input‑group identifiers from the staging database, calls the external **Geneva Order Management API** for each group to obtain the creation status of individual orders, translates the response into a status string, and writes the consolidated results back to the database in bulk. The method is invoked by the main orchestration class (`APIMain`) as part of the nightly data‑move workflow.

---

## 2. Key Classes & Functions

| Class / Method | Responsibility |
|----------------|----------------|
| **`OrderStatus`** (public) | Wrapper for the order‑status polling logic. |
| `boolean orderStatusCheck()` | • Reads input group IDs via `RawDataAccess.fetchInputGroupId()`.<br>• For each group, performs an HTTP GET to the Geneva API.<br>• Parses JSON response (`GetOrderResponse` → `OrderResponse` → `OrderStatusList`).<br>• Builds a `HashMap<String, Tuple2>` mapping `inputRowId` → `(statusString, groupValue)`.<br>• Persists the map with `RawDataAccess.bulkStatusUpdate()`.<br>• Returns `true` on completion (exceptions are propagated). |
| **`RawDataAccess`** (DAO, external) | Provides DB access: `fetchInputGroupId()` returns `List<Tuple2>` (groupId, groupValue); `bulkStatusUpdate(String groupId, Map<String,Tuple2>)` writes status updates. |
| **`APIConstants`** (constants, external) | Holds HTTP header names/values (`CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE`). Likely populated from `APIAccessDAO.properties`. |
| **`GetOrderResponse`, `OrderResponse`, `OrderStatusList`** (DTOs) | POJOs generated for the JSON payload returned by the Geneva API. |
| **`Tuple2`** (utility) | Simple container for a pair of values (`String statusString`, `String groupValue`). |

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Inputs** | • Database table(s) containing pending input‑group IDs (accessed via `RawDataAccess.fetchInputGroupId()`).<br>• External REST endpoint: `https://api.tatacommunications.com:443/GenevaOrderManagementAPI/v1/orderCreationstatus/{inputGroupId}`.<br>• HTTP headers defined in `APIConstants` (content‑type, account authentication). |
| **Outputs** | • Bulk update to the same (or a related) database table via `RawDataAccess.bulkStatusUpdate()`. Each row receives a status string (`<STATUS>_<MESSAGE>` or `<STATUS>_SUCCESS`) and the original group value.<br>• Log entries (log4j) for error conditions and debug prints (currently `System.out`). |
| **Side Effects** | • Network traffic to the Geneva API (HTTPS, port 443).<br>• Potential DB writes (status updates).<br>• Creation of temporary `CloseableHttpClient`/`CloseableHttpResponse` objects per iteration. |
| **Assumptions** | • The API endpoint is reachable and returns JSON matching the DTO model.<br>• `APIConstants` values (especially authentication) are correctly populated from property files or environment variables.<br>• The DB schema expected by `RawDataAccess` exists and the user has write permission.<br>• No pagination – each group ID returns a complete status list. |

---

## 4. Integration Points & Call Flow

1. **Caller** – `APIMain` (or another orchestrator) invokes `new OrderStatus().orderStatusCheck()`.  
2. **DAO Layer** – `RawDataAccess` reads pending groups from the staging DB and later writes back status updates.  
3. **Constants** – `APIConstants` supplies HTTP header values; these are populated from `APIAccessDAO.properties` (see previous file).  
4. **External Service** – Geneva Order Management API (HTTPS). The URL is hard‑coded; any change requires a code change or externalization.  
5. **Logging** – log4j configuration (`log4j.properties`) controls log levels; errors are logged via `logger.error`.  
6. **Data Model** – DTO classes (`GetOrderResponse`, etc.) are used only within this file; they are shared with other callout classes that also consume the same API (e.g., `BulkInsert`, `CustomPlanDetails`).  

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **No timeout / retry** on HTTP calls | Stalls the whole batch if the remote service hangs. | Configure request timeouts (`RequestConfig`) and implement exponential back‑off retries (e.g., Apache HttpClient `HttpRequestRetryHandler`). |
| **Resource leak on exception** – `CloseableHttpClient`/`CloseableHttpResponse` may stay open if an exception occurs before `close()`. | Exhausts sockets / file descriptors. | Use *try‑with‑resources* for both client and response. |
| **Hard‑coded API URL** | Requires code change for environment moves (dev, test, prod). | Externalize URL (and possibly port) to a properties file or environment variable (`API_ENDPOINT_ORDER_STATUS`). |
| **System.out usage** | Pollutes console logs, bypasses log4j configuration. | Replace all `System.out.println` with appropriate `logger.debug/info`. |
| **Bulk map grows per group** – potential OOM if a group contains many rows. | Out‑of‑memory crash. | Process in smaller batches or stream updates directly to DB. |
| **Lack of input validation** (e.g., null `groupId.getKey()`) | NullPointerException, malformed URL. | Add defensive checks before building the URL. |
| **Uncaught generic `Exception`** in the HTTP block | Swallows specific failures, makes troubleshooting harder. | Catch specific exceptions (IO, JsonSyntax, SQL) and rethrow or log with context. |

---

## 6. Running / Debugging the Class

**Typical execution (via orchestrator):**
```bash
# Assuming the project is built with Maven/Gradle and packaged as a jar
java -cp move-indiamed-api.jar com.tcl.api.callout.APIMain
```
`APIMain` will instantiate `OrderStatus` and call `orderStatusCheck()`.

**Standalone test (quick sanity check):**
```java
public static void main(String[] args) throws Exception {
    OrderStatus os = new OrderStatus();
    boolean ok = os.orderStatusCheck();
    System.out.println("Order status check completed: " + ok);
}
```
Compile and run with the classpath that includes:
- `move-indiamed-api.jar`
- `log4j.jar`
- `httpclient.jar`, `httpcore.jar`
- `gson.jar`
- Database driver (e.g., `ojdbc8.jar`)

**Debugging tips**
1. **Enable DEBUG logging** in `log4j.properties` for `com.tcl.api.callout.OrderStatus`.
2. **Set breakpoints** on:
   - `groupIds = rda.fetchInputGroupId();`
   - Inside the `for` loop before the HTTP request.
   - After `gson.fromJson(...)` to inspect the deserialized object.
3. **Inspect the `hm` map** after each iteration to verify status strings.
4. **Check DB** – query the target status table to confirm rows were updated.
5. **Network trace** – use `curl` or `tcpdump` to verify the GET request format and response.

---

## 7. External Configuration & Environment Variables

| Config / Env | Usage |
|--------------|-------|
| `APIConstants.CONTENT_TYPE` / `CONTENT_VALUE` | HTTP `Content-Type` header (e.g., `application/json`). |
| `APIConstants.ACNT_AUTH` / `ACNT_VALUE` | Authentication header (likely a token or API key). Values are read from `APIAccessDAO.properties` (see previous file). |
| **Database connection** – managed inside `RawDataAccess` (properties file not shown). |
| **Log4j** – configuration file `log4j.properties` controls log levels and appenders. |
| **Hard‑coded URL** – `apiUrl` string inside `OrderStatus`. Not externalized; consider moving to a property (`order.status.endpoint`). |

---

## 8. Suggested Improvements (TODO)

1. **Externalize the order‑status endpoint and HTTP timeout settings** to a properties file (or environment variables) and inject them via a configuration loader.  
2. **Refactor the HTTP handling to use try‑with‑resources and a shared `CloseableHttpClient`** (connection pool) with proper timeout and retry policies, eliminating per‑iteration client creation and ensuring resources are always released.  

---