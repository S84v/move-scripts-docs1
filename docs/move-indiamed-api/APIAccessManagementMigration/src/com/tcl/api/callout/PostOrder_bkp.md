**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\PostOrder_bkp.java`  

---

## 1. High‑Level Summary
`PostOrder_bkp` is a legacy “backup” implementation that transforms a list of `PostOrderDetail` objects (populated earlier in the migration pipeline) into the JSON payload required by the Geneva Order Management REST API (`/v1/order`). It builds an `OrderInput` containing one or more `OrdersInformation` records, enriches each record with dozens of optional attributes, and issues an HTTP POST with retry logic. The method logs the request/response and throws no value – success is inferred from a 200 HTTP status.

---

## 2. Important Classes & Functions  

| Class / Method | Responsibility |
|----------------|----------------|
| **`PostOrder_bkp`** | Wrapper class containing the only public method `postOrderDetails`. |
| **`postOrderDetails(List<PostOrderDetail>)`** | Core routine: 1) map each `PostOrderDetail` → `OrdersInformation`; 2) assemble `OrderInput`; 3) serialize to JSON (Gson); 4) POST to the Geneva API with configurable retry count; 5) log request/response. |
| **`OrderInput`** (model) | Root JSON object expected by the API; holds `inputGroupId`, `sourceSystem`, and a list of `OrdersInformation`. |
| **`OrdersInformation`** (model) | Represents a single order line; populated with fields such as `actionType`, `productName`, address objects, attribute strings, etc. |
| **`AttributeString`** (model) | Simple key/value pair used to carry dynamic attributes that are not part of the fixed schema. |
| **`Address`** (model) | Holds address components (line1‑3, city, state, country, zipcode) for CPE delivery, site A/B, contracting, GST‑IN, etc. |
| **`APIConstants`** | Holds static configuration values: HTTP headers, retry count, etc. (referenced but not defined here). |
| **`Logger` (log4j)** | Used for request/response logging; instantiated with the name of the original `PostOrder` class. |

*Note:* The class mirrors the production `PostOrder` implementation; the “_bkp” suffix indicates it is kept for reference or fallback.

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Input** | `List<PostOrderDetail> postOrd` – each element contains raw order data extracted from upstream sources (e.g., CSV, DB, or previous API call). All fields are accessed via getters; many are optional. |
| **Output** | No return value. Success is implied when the HTTP response status is 200. On failure, the method silently exits after retries (exception is only printed). |
| **Side‑Effects** | • HTTP POST to `https://api-uat.tatacommunications.com:443/GenevaOrderManagementAPI/v1/order`.<br>• Log of the JSON payload and raw `CloseableHttpResponse` object (via Log4j). |
| **Assumptions** | • `APIConstants.CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE`, and `RETRY_COUNT` are correctly populated (likely from a properties file or environment).<br>• The target API is reachable from the runtime host and uses basic auth header defined in `APIConstants`.<br>• All numeric fields (e.g., zip codes, quantities) are parsable as integers when present.<br>• `postOrd` is non‑null; individual fields may be null. |
| **External Services** | - **Geneva Order Management API** (REST, HTTPS).<br>- **Log4j** logging infrastructure (file/console). |
| **External Config / Env** | - `APIConstants` static values (headers, retry count).<br>- Potentially a Java system property or external `.properties` file that populates `APIConstants`. No direct env‑var usage in this file. |

---

## 4. Connection to Other Scripts / Components  

| Component | Relationship |
|-----------|--------------|
| **`APIMain` / `APICallout`** (previous files) | Likely orchestrates the overall migration flow, reads source data, creates `PostOrderDetail` objects, and invokes `new PostOrder().postOrderDetails(list)`. The backup class mirrors the same contract, so any caller that expects `PostOrder` can swap to `PostOrder_bkp` without code change. |
| **`BulkInsert*`, `OrderStatus`, `CustomPlanDetails`, etc.** | These classes prepare or enrich the `PostOrderDetail` objects (e.g., adding attributes, calculating rates). `PostOrder_bkp` consumes the final enriched list. |
| **`APIConstants`** | Shared constant holder used across the callout package; defines HTTP headers, retry count, etc. |
| **Downstream** | The Geneva API processes the posted order and returns a response (not parsed here). Downstream systems (billing, provisioning) will react to the order creation. |
| **Upstream** | Data extraction scripts (not shown) populate the `PostOrderDetail` POJO from source systems (legacy DB, CSV, etc.). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Silent failure after retries** – method only prints stack trace; callers receive no exception or status. | Orders may be lost without alert. | Propagate a custom exception on final failure; integrate with monitoring/alerting (e.g., send email via `AlertEmail`). |
| **Hard‑coded API URL** (UAT endpoint). | Promotion to production may require code change. | Externalize URL in a properties file or env‑var; use a configuration service. |
| **Unvalidated numeric parsing** (`Integer.parseInt`) can throw `NumberFormatException` which is not caught. | Job aborts mid‑batch. | Validate/clean numeric fields before parsing; catch `NumberFormatException` and log as bad data. |
| **Large request bodies** – all orders are sent in a single POST. | Potential payload size limits, memory pressure. | Implement chunking (batch size configurable) or stream the request. |
| **Logging of full JSON payload** may expose PII. | Compliance breach. | Mask sensitive fields before logging or reduce log level in production. |
| **Static logger name (`PostOrder.class`)** – may cause confusion when reading logs. | Log traceability issue. | Use `PostOrder_bkp.class` or pass logger name as parameter. |
| **Retry loop does not back‑off** – rapid retries may hammer the API. | Throttling or temporary outage escalation. | Add exponential back‑off with jitter. |

---

## 6. Running / Debugging the Class  

1. **Prerequisites**  
   - Java 8+ runtime, Maven/Gradle build that includes dependencies: `httpclient`, `gson`, `log4j`.  
   - `APIConstants` populated (usually via a properties file on the classpath).  

2. **Typical Invocation (from another class)**  

```java
List<PostOrderDetail> orders = OrderExtractor.fetchPendingOrders(); // custom code
PostOrder_bkp poster = new PostOrder_bkp();
try {
    poster.postOrderDetails(orders);
} catch (Exception e) {
    // handle or rethrow
}
```

3. **Debug Steps**  
   - Set a breakpoint at the start of `postOrderDetails`.  
   - Verify `postOrd` size and inspect a sample `PostOrderDetail`.  
   - Step through the mapping loop; confirm that required fields (e.g., `statusready`, `chargePeriod`) are populated.  
   - After `gson.toJson`, inspect the generated JSON string.  
   - If the HTTP call fails, check `response.getStatusLine()` and the logged response entity (add `EntityUtils.toString`).  
   - Review Log4j output for the request payload and any stack traces.  

4. **Unit Test Stub**  

```java
@Test
public void testPostOrderDetails_success() throws Exception {
    PostOrderDetail d = new PostOrderDetail();
    d.setInputRowId("123");
    d.setStatusready("NEW");
    // set minimal required fields …
    List<PostOrderDetail> list = Collections.singletonList(d);
    new PostOrder_bkp().postOrderDetails(list);
    // assert via mock HttpClient or verify logs
}
```

---

## 7. External Configuration / Environment Variables  

| Item | Source | Usage |
|------|--------|-------|
| `APIConstants.CONTENT_TYPE` / `CONTENT_VALUE` | Static constant (likely `"Content-Type"` / `"application/json"`). | Sets HTTP header. |
| `APIConstants.ACNT_AUTH` / `ACNT_VALUE` | Static constant (e.g., `"Authorization"` / `"Basic …"`). | Auth header for API. |
| `APIConstants.RETRY_COUNT` | Static int (default maybe 3). | Controls retry loop. |
| **API endpoint URL** (`apiUrl` variable) | Hard‑coded in method (`https://api-uat.tatacommunications.com:443/...`). | Target REST service. |
| **Log4j configuration** | `log4j.properties` or XML on classpath. | Determines log destination and level. |

If any of these constants are loaded from external property files, the file name/path should be documented in the project’s README (not visible here).

---

## 8. Suggested TODO / Improvements  

1. **Error Propagation & Monitoring** – Replace the silent `e.printStackTrace()` with a structured exception (e.g., `PostOrderException`) and integrate with the existing `AlertEmail` utility to notify operations on failure.  

2. **Externalize Configurable Values** – Move the API URL, retry count, and header values into a dedicated properties file or environment variables, and inject them via a configuration loader (e.g., Spring `@Value` or a custom `ConfigProvider`). This will simplify promotion between environments (UAT → PROD).  

---