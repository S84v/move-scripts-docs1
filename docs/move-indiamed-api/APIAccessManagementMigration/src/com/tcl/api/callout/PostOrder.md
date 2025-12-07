**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\PostOrder.java`

---

## 1. High‑Level Summary
`PostOrder` builds and sends a bulk **order‑creation/modification API request** to the Geneva Order Management service (`https://api-uat.tatacommunications.com/.../order`).  
It receives a list of `PostOrderDetail` DTOs (populated earlier in the migration flow), maps each record to an `OrdersInformation` model, enriches it with a large set of attribute‑name/value pairs (via reflection), assembles the full `OrderInput` payload, posts the JSON to the external REST endpoint with retry logic, logs the request/response, and finally records the generated **input‑group‑ID** back to a control file through `RawDataAccess.updateFileWithNewGroupIDs`.

---

## 2. Key Classes & Public API

| Class / Method | Responsibility |
|----------------|----------------|
| **`PostOrder`** (public class) | Orchestrates the transformation of raw order rows into the API payload and performs the HTTP POST. |
| `void postOrderDetails(List<PostOrderDetail> postOrd, String inputGroupIdFile)` | Core entry point used by higher‑level scripts (e.g., `APIMain`). Takes the in‑memory order rows and a path to a “group‑ID” tracking file. |
| **`APIConstants`** (imported) | Holds static configuration: content‑type header, authentication header/value, retry count, etc. |
| **`RawDataAccess`** (imported) | Persists the newly generated `inputGroupId` back to the control file (used by downstream steps). |
| **`OrderInput`**, **`OrdersInformation`**, **`AttributeString`**, **`Address`** (model/DTO) | POJOs that are serialized to JSON by Gson. |
| **`Logger`** (log4j) | Writes request JSON and HTTP response objects to the configured log files (`log4j.properties`). |

*Note:* The class does **not** expose any other public methods; all work is encapsulated inside `postOrderDetails`.

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Method Parameters** | `List<PostOrderDetail> postOrd` – collection of raw order rows (populated by earlier ETL steps). <br>`String inputGroupIdFile` – filesystem path to a file that stores the generated `inputGroupId`. |
| **External Services** | - **REST API**: `https://api-uat.tatacommunications.com:443/GenevaOrderManagementAPI/v1/order` (HTTPS, expects JSON). <br>- **Database / File Store**: `RawDataAccess.updateFileWithNewGroupIDs` writes back the `inputGroupId` to the supplied file (likely a staging table or flat file). |
| **Configuration / Constants** | `APIConstants.CONTENT_TYPE`, `APIConstants.CONTENT_VALUE`, `APIConstants.ACNT_AUTH`, `APIConstants.ACNT_VALUE`, `APIConstants.RETRY_COUNT`. These are defined in `com.tcl.api.constants.APIConstants` and may be populated from environment variables or a properties file. |
| **Logging** | Uses Log4j (configuration in `log4j.properties`). Logs the full JSON payload and the raw `CloseableHttpResponse` object. |
| **Return Value** | `void`. Success is inferred from the HTTP status code (200) and the side‑effect of updating the group‑ID file. |
| **Exceptions** | Propagates `ClientProtocolException`, `UnsupportedEncodingException`, `ParseException`. Internally catches `IOException` (only prints stack trace). |
| **Assumptions** | - All required fields in `PostOrderDetail` are non‑null where accessed (e.g., `pOrd.getChargePeriod()`). <br>- The API endpoint is reachable and accepts the constructed payload. <br>- `APIConstants` values are correctly set (auth header, retry count). <br>- The input list is reasonably sized to fit in memory (the whole request body is built in a single string). |

---

## 4. Interaction with Other Scripts / Components

| Component | Connection Point |
|-----------|------------------|
| **`APIMain`** (or similar driver) | Instantiates `PostOrder` and calls `postOrderDetails`. It supplies the `postOrd` list (populated from source files / DB) and the path to the group‑ID tracking file. |
| **`PostOrderDetail`** DTOs | Produced by earlier transformation scripts (`BulkInsert`, `CustomPlanDetails`, etc.) that read raw source data and map it to this POJO. |
| **`RawDataAccess`** | Persists the generated `inputGroupId` for downstream batch steps (e.g., status polling, audit). |
| **`log4j.properties`** | Provides logging configuration (file appender, level). |
| **`APIConstants`** | Centralised constants; may be loaded from environment variables or a properties file (e.g., auth token). |
| **External Order Management System** | Receives the JSON payload; its response is only checked for HTTP 200 – no further parsing is performed here. |
| **Potential downstream consumer** | A “status‑check” script that reads the updated group‑ID file to query order processing results. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Hard‑coded API URL** | Changes to endpoint (e.g., move to production) require code change/re‑deployment. | Externalise the URL to a properties file or environment variable (`API_ENDPOINT`). |
| **Insufficient error handling** – only HTTP 200 is considered success; other status codes are silently retried up to `RETRY_COUNT` and then ignored. | Lost failures, silent data loss. | Capture non‑200 responses, log response body, and raise an exception after retries. |
| **Potential NPE** – many `pOrd.getX()` calls assume non‑null values (e.g., `getChargePeriod()`). | Job aborts mid‑batch. | Add defensive null checks or default values before accessing. |
| **Reflection overhead & fragility** – attribute extraction relies on exact getter names; any DTO change breaks it. | Missing attributes, runtime exceptions. | Replace reflection with a static mapping or use a library like Jackson’s `@JsonProperty`. |
| **Large payload in memory** – the entire request body is built as a single string before sending. | Out‑of‑memory for very large batches. | Stream JSON (e.g., `Gson` streaming API) or split batches into manageable sizes. |
| **No timeout / SSL verification config** – default HttpClient settings may block indefinitely or accept untrusted certs. | Hanging jobs, security exposure. | Configure request timeout and enforce proper SSL context. |
| **Credentials in code** – `APIConstants.ACNT_AUTH` may contain static credentials. | Security breach. | Store secrets in a vault or environment variable, rotate regularly. |

---

## 6. Running / Debugging the Class

1. **Typical Invocation** (from a driver such as `APIMain`):  
   ```java
   List<PostOrderDetail> details = ... // populated earlier
   String groupIdFile = "/opt/move/data/groupIds.txt";
   new PostOrder().postOrderDetails(details, groupIdFile);
   ```

2. **Prerequisites**  
   - `log4j.properties` on the classpath (defines log file location).  
   - `APIConstants` correctly populated (auth header, content type, retry count).  
   - Network connectivity to the UAT API endpoint.  
   - Write permission on `groupIdFile`.

3. **Debug Steps**  
   - Set log level to `DEBUG` in `log4j.properties` to see the generated JSON.  
   - Attach a debugger at the start of `postOrderDetails` to inspect the `postOrd` list.  
   - After the HTTP call, inspect `response.getStatusLine()` and optionally `EntityUtils.toString(response.getEntity())` to view any error payload.  
   - Verify that `RawDataAccess.updateFileWithNewGroupIDs` actually writes the expected group ID (check the file after execution).

4. **Unit Test Stub**  
   ```java
   @Test
   public void testPostOrderDetails() throws Exception {
       List<PostOrderDetail> mockList = Collections.singletonList(new PostOrderDetail(/* set fields */));
       String tmpFile = Files.createTempFile("group", ".txt").toString();
       new PostOrder().postOrderDetails(mockList, tmpFile);
       // assert file contains a non‑null group ID, mock HTTP server returns 200, etc.
   }
   ```

---

## 7. External Configuration & Environment Dependencies

| Item | Where Defined | Usage |
|------|---------------|-------|
| `log4j.properties` | `move-indiamed-api\APIAccessManagementMigration\src\log4j.properties` | Controls logging output (file location, level). |
| `APIConstants` class | `com.tcl.api.constants.APIConstants` | Provides HTTP headers, auth token, retry count, possibly reads from environment variables or a properties file. |
| **Authentication token / credentials** | Likely stored in `APIConstants.ACNT_VALUE` (could be loaded from env or secret manager). | Sent as `ACNT_AUTH` header to the external API. |
| **Input group‑ID tracking file** | Path supplied by caller (`inputGroupIdFile`). | Updated by `RawDataAccess` after a successful POST. |
| **Java system properties** (e.g., proxy settings) | Not shown but may be required for outbound HTTPS. | Ensure network routing works in the execution environment. |

---

## 8. Suggested Improvements (TODO)

1. **Externalise the API endpoint and authentication** – move URL, auth header/value, and retry count to a dedicated properties file or secure vault; inject via environment variables at runtime.  
2. **Robust error handling & response parsing** – capture non‑200 responses, log the response body, and throw a custom exception after retries; optionally implement exponential back‑off.  

*(Additional enhancements such as streaming JSON, batch size control, and replacing reflection with a typed mapper are also advisable but beyond the immediate scope.)*