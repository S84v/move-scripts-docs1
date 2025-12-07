**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\PostOrder_bkp.java`

---

## 1. High‑Level Summary
`PostOrder_bkp` is a backup implementation of the “post‑order” integration point used by the API Access Management move‑process. It receives a collection of `PostOrderDetail` objects (populated by upstream data‑move scripts), translates each record into the `OrdersInformation` model required by the Geneva Order Management API, enriches the payload with a large set of attribute‑name/value pairs (via reflection), serialises the request to JSON with Gson, and POSTs it to the UAT endpoint `https://api-uat.tatacommunications.com:443/GenevaOrderManagementAPI/v1/order`. The class also implements a simple retry loop based on a constant retry count.

---

## 2. Key Classes & Methods

| Class / Method | Responsibility |
|----------------|----------------|
| **`PostOrder_bkp`** | Container for the order‑posting logic. |
| `void postOrderDetails(List<PostOrderDetail> postOrd)` | Core routine: builds the request body, maps fields from `PostOrderDetail` → `OrdersInformation`, adds address objects, assembles attribute strings, serialises to JSON, and executes the HTTP POST with retry. |
| **Model classes (used)** | |
| `OrderInput` | Root JSON object containing a list of `OrdersInformation`. |
| `OrdersInformation` | Represents a single order record as required by the external API. |
| `PostOrderDetail` | Source POJO populated by earlier move‑scripts; contains >80 fields (e.g., product, pricing, address, tax, partner data). |
| `AttributeString` | Simple name/value holder for the “attributeString” array in the payload. |
| `Address` | Holds address components (line1‑3, city, state, country, zipcode). |
| **Utility / Constants** | |
| `APIConstants` | Holds HTTP header names/values, retry count, and possibly auth token. |
| `Gson` | Serialises the request object to JSON. |
| Apache HttpClient (`CloseableHttpClient`, `HttpPost`, `CloseableHttpResponse`) | Performs the HTTPS POST. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Input** | `List<PostOrderDetail> postOrd` – each element contains raw order data extracted from source systems (DB, CSV, etc.). |
| **Output** | No return value. Side‑effect is an HTTP POST to the Geneva Order Management API. The method prints the JSON payload and the raw `CloseableHttpResponse` to `System.out`. |
| **External Services** | - **Geneva Order Management API** (HTTPS endpoint). <br> - **Authentication** header values supplied by `APIConstants.ACNT_AUTH` / `APIConstants.ACNT_VALUE`. |
| **Databases / Queues** | None directly accessed; the method assumes upstream scripts have already read from DB / queues and supplied the `PostOrderDetail` list. |
| **File / Network I/O** | - Network call to external API. <br> - Console logging (STDOUT). |
| **Assumptions** | - All required fields in `PostOrderDetail` are non‑null or handled by null checks. <br> - `APIConstants.RETRY_COUNT` is a positive integer. <br> - The UAT endpoint is reachable and accepts the JSON schema defined by `OrderInput`. <br> - Header constants contain valid values (e.g., `Content-Type: application/json`). |
| **Resource Management** | The `CloseableHttpClient` is created but never explicitly closed (relies on GC). |

---

## 4. Integration Points & Call Flow

1. **Up‑stream data‑move scripts** (e.g., bulk extract → transformation) create a `List<PostOrderDetail>` and invoke `new PostOrder_bkp().postOrderDetails(list)`.  
2. **`PostOrder_bkp`** builds the request and posts to the **Geneva Order Management API** (UAT).  
3. **API response** is logged; success is defined as HTTP 200. On failure, the method retries up to `APIConstants.RETRY_COUNT`.  
4. **Down‑stream handling** (not in this file) may read the API response from logs or a separate monitoring component to decide on further actions (e.g., alert email via `AlertEmail`).  

*Note:* The primary production class is likely `PostOrder.java`. This backup version is kept for reference or fallback; any orchestration script that references `PostOrder` should be examined to confirm which implementation is currently wired.

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hard‑coded endpoint URL** | Deployments may unintentionally hit UAT instead of PROD. | Externalise the URL (e.g., via environment variable or `APIConstants`). |
| **No proper resource cleanup** | Potential socket leakage under high volume. | Use try‑with‑resources for `CloseableHttpClient` and `CloseableHttpResponse`. |
| **Insufficient error handling** | Non‑200 responses are silently retried; failures may go unnoticed. | Log response bodies, differentiate between client/server errors, and raise an exception after retries. |
| **Reflection‑based attribute extraction** | Missing getter or typo causes `NoSuchMethodException` → stack trace printed, but processing continues. | Replace reflection with a static mapping or use a library like Jackson’s `ObjectMapper` to serialize the POJO directly. |
| **NumberFormatException on integer fields** | Bad data (e.g., non‑numeric zipcode) crashes the whole batch. | Validate/parse with safe fallback or record‑level error handling. |
| **Plain‑text logging of sensitive data** | May expose PII or auth tokens in logs. | Redact or avoid logging full payloads; use structured logging with masking. |
| **Retry loop without back‑off** | Aggressive retries can overload the API. | Implement exponential back‑off and respect `Retry‑After` headers. |
| **Static header values** | Token rotation or environment changes require code change. | Load auth token from a secure vault or environment variable at runtime. |

---

## 6. Running / Debugging the Class

### Prerequisites
- Java 8+ (compatible with Apache HttpClient 4.x and Gson).  
- `APIConstants` populated with correct header values and retry count.  
- All model classes (`PostOrderDetail`, `OrdersInformation`, etc.) compiled and on the classpath.  

### Example Invocation (from a unit test or main method)

```java
import com.tcl.api.callout.PostOrder_bkp;
import com.tcl.api.model.PostOrderDetail;
import java.util.ArrayList;
import java.util.List;

public class PostOrderRunner {
    public static void main(String[] args) throws Exception {
        // 1. Build a minimal PostOrderDetail (populate required fields only)
        PostOrderDetail detail = new PostOrderDetail();
        detail.setInputRowId("12345");
        detail.setEid("EID001");
        detail.setStatusready("NEW");
        detail.setProductname("Broadband");
        // ... set other mandatory fields ...

        List<PostOrderDetail> payload = new ArrayList<>();
        payload.add(detail);

        // 2. Call the backup post method
        PostOrder_bkp poster = new PostOrder_bkp();
        poster.postOrderDetails(payload);
    }
}
```

### Debugging Tips
- **Breakpoints**: Set a breakpoint at the start of `postOrderDetails` to inspect the incoming list.  
- **Inspect JSON**: The method prints the generated JSON; capture it from console or redirect `System.out` to a file for validation.  
- **HTTP Response**: After the retry loop, inspect `response.getStatusLine()` and, if needed, `EntityUtils.toString(response.getEntity())` to view the body.  
- **Enable Logging**: Replace `System.out.println` with a proper logger (e.g., SLF4J) to control log levels.  

---

## 7. External Configuration & Dependencies

| Item | Usage |
|------|-------|
| `APIConstants.CONTENT_TYPE` / `CONTENT_VALUE` | Sets `Content-Type` header (likely `application/json`). |
| `APIConstants.ACNT_AUTH` / `ACNT_VALUE` | Authentication header (e.g., `Authorization: Bearer <token>`). |
| `APIConstants.RETRY_COUNT` | Number of retry attempts for the HTTP call. |
| **Hard‑coded URL** (`apiUrl` variable) | Currently points to the UAT environment; should be externalised. |
| **Gson library** | JSON serialisation of `OrderInput`. |
| **Apache HttpClient** | HTTP transport layer. |
| **Reflection** (`Method getterMethod = pOrd.getClass().getMethod(...)`) | Dynamically extracts attribute values for the `attributeString` array. |

If any of the constants are loaded from environment variables or property files, verify those sources (e.g., `application.properties`, OS env vars) during deployment.

---

## 8. Suggested Improvements (TODO)

1. **Externalise the endpoint and credentials** – move `apiUrl` and auth token into `APIConstants` or a secure configuration store (e.g., Vault) and load at runtime.  
2. **Refactor resource handling** – wrap `CloseableHttpClient` and `CloseableHttpResponse` in try‑with‑resources, add proper logging, and return a result object indicating success/failure with response details.  

---