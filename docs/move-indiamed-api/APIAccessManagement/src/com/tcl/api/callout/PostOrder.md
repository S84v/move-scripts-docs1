**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\PostOrder.java`

---

## 1. High‑Level Summary
`PostOrder` is the HTTP‑client component that transforms a collection of `PostOrderDetail` POJOs (produced by earlier “move” steps) into the JSON payload expected by the **Geneva Order Management API** (`/order`). It builds an `OrderInput` object containing a list of `OrdersInformation` records, enriches each record with a large set of attribute‑name/value pairs (via reflection), adds address objects, and POSTs the payload to the API endpoint defined in `APIConstants`. The method retries on non‑200 responses and logs the request/response.

---

## 2. Key Classes & Public API

| Class / Method | Responsibility |
|----------------|----------------|
| **`PostOrder`** | Service class that performs the HTTP POST to the order‑management API. |
| `void postOrderDetails(List<PostOrderDetail> postOrd)` | Core public method. Accepts a list of `PostOrderDetail` objects, builds the request body, sends it, and retries according to `APIConstants.RETRY_COUNT`. |
| **`APIConstants`** (external) | Holds static configuration: `ORDR_ATTR_URL`, header names/values, `RETRY_COUNT`, etc. |
| **Model classes** (from `com.tcl.api.model`): `OrderInput`, `OrdersInformation`, `AttributeString`, `Address`, `PostOrderDetail` | Data structures that are populated and serialized to JSON via Gson. |
| **`Logger`** (log4j) | Logs request JSON and HTTP response objects. |

*Note:* The class does **not** expose any other public methods; it is intended to be invoked by higher‑level orchestrators such as `APIMain` or `APICallout` (see history).

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Input** | `List<PostOrderDetail>` – each element contains raw order fields (e.g., `eid`, `statusready`, address lines, pricing, etc.). |
| **Output** | No return value. Side‑effect is an HTTP POST to the external order‑management service. The method logs the JSON payload and the raw `CloseableHttpResponse`. |
| **External Services** | - **Geneva Order Management API** (HTTPS endpoint). <br> - **Log4j** logging subsystem. |
| **Databases / Queues** | None directly accessed in this class; upstream scripts may read/write DB/queues before calling this method. |
| **Assumptions** | - `APIConstants` is correctly populated (URL, headers, retry count). <br> - All required fields in `PostOrderDetail` are non‑null where used (e.g., `chargePeriod`, `statusready`). <br> - The target API returns HTTP 200 on success; other codes trigger retry. <br> - The JVM has permission to open outbound HTTPS connections. |
| **Resource Management** | `CloseableHttpClient` is created but never explicitly closed; relies on GC. `CloseableHttpResponse` is also not closed. |

---

## 4. Integration Points & Call Flow

1. **Upstream Producer** – Scripts such as `BulkInsert`, `OrderStatus`, or `APICallout` read source data (files, DB, MQ) and create `PostOrderDetail` objects.  
2. **Orchestrator** – `APIMain` (or a similar driver) instantiates `PostOrder` and invokes `postOrderDetails(...)`.  
3. **Configuration** – `APIConstants` is loaded at application start (often from a properties file or environment variables).  
4. **Downstream Consumer** – The external **Geneva Order Management API** receives the JSON payload, processes the order, and returns a response that is logged but not parsed further in this class.  

*Because the class only logs the raw response, downstream error handling (e.g., persisting failed orders) is expected to be performed by the caller.*

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Unclosed HTTP resources** – `httpclient` and `response` are never closed, potentially leaking sockets under high volume. | Resource exhaustion, connection‑pool starvation. | Use *try‑with‑resources* for `CloseableHttpClient` and `CloseableHttpResponse`. |
| **Blind retry on any non‑200** – No distinction between client errors (4xx) and transient server errors (5xx). | Unnecessary retries, possible throttling or lock‑out. | Inspect status code; retry only on 5xx or network exceptions. |
| **Reflection‑based attribute extraction** – Relies on exact getter naming; any typo or missing getter throws an exception that is only printed (`e.printStackTrace()`). | Missing attributes, silent data loss, noisy logs. | Replace reflection with a static mapping or use a library like Jackson’s `@JsonAnyGetter`. |
| **Hard‑coded string comparisons** (e.g., `statusready` values) – Case‑insensitive but scattered; new status values require code changes. | Maintenance burden, risk of incorrect `actionType`. | Centralize status handling in an enum with mapping to `actionType` and `changeOrderType`. |
| **No response body handling** – Success/failure is inferred solely from HTTP status; any error details from the API are lost. | Difficulty diagnosing order rejections. | Capture and log response entity; optionally surface to caller. |
| **Potential NumberFormatException** – Parsing of numeric fields (`prodQuantity`, zip codes) assumes valid numeric strings. | Runtime exception, job failure. | Validate/guard numeric parsing; fallback to defaults or record error. |
| **Thread.sleep in retry loop** – Fixed 1 s delay may be insufficient under load. | Throttling or prolonged latency. | Implement exponential back‑off or configurable delay. |

---

## 6. Running / Debugging the Component

### Prerequisites
1. **Classpath** must include:  
   - `gson` library  
   - Apache HttpClient (`httpclient`, `httpcore`)  
   - Log4j  
   - Project’s model packages (`com.tcl.api.model`) and `APIConstants`.
2. **Configuration** – Ensure `APIConstants` properties (URL, headers, retry count) are set, typically via a `*.properties` file or environment variables referenced inside that class.

### Typical Invocation (e.g., from a unit test or driver)

```java
import com.tcl.api.callout.PostOrder;
import com.tcl.api.model.PostOrderDetail;
import java.util.List;
import java.util.ArrayList;

public class PostOrderDemo {
    public static void main(String[] args) throws Exception {
        // Build a minimal PostOrderDetail (real code would populate all needed fields)
        PostOrderDetail pod = new PostOrderDetail();
        pod.setEid("1234567890");
        pod.setStatusready("NEW");
        pod.setChargePeriod("Monthly");
        pod.setPlantype("COMMERCIAL");
        // ... set other mandatory fields ...

        List<PostOrderDetail> batch = new ArrayList<>();
        batch.add(pod);

        PostOrder postOrder = new PostOrder();
        postOrder.postOrderDetails(batch);
    }
}
```

### Debugging Tips
| Situation | Action |
|-----------|--------|
| **No request is sent** | Verify `APIConstants.ORDR_ATTR_URL` is non‑empty; enable Log4j DEBUG for `PostOrder`. |
| **HTTP 4xx/5xx response** | Check the logged JSON payload; add code to log `EntityUtils.toString(response.getEntity())`. |
| **NullPointerException** | Ensure every field accessed (e.g., `pOrd.getChargePeriod()`) is non‑null; add defensive checks or default values. |
| **Reflection failures** | The stack trace printed in the catch block will indicate missing getters; confirm naming conventions in `PostOrderDetail`. |
| **Resource leak detection** | Run with a profiler or enable `-Dorg.apache.http.impl.conn.PoolingHttpClientConnectionManager.maxTotal=...` to monitor open connections. |

---

## 7. External Configuration & Environment Dependencies

| Item | Source | Usage |
|------|--------|-------|
| `APIConstants.ORDR_ATTR_URL` | Typically loaded from a properties file (`api.properties`) or environment variable. | Target endpoint for the POST request. |
| `APIConstants.CONTENT_TYPE`, `APIConstants.CONTENT_VALUE` | Same source as above. | HTTP `Content-Type` header (e.g., `application/json`). |
| `APIConstants.ACNT_AUTH`, `APIConstants.ACNT_VALUE` | Same source. | Authentication header (e.g., API key or bearer token). |
| `APIConstants.RETRY_COUNT` | Same source. | Number of retry attempts on non‑200 responses. |
| Log4j configuration (`log4j.properties` or XML) | External file. | Controls logging level and destination for `logger`. |

*If any of these constants are missing or malformed, the POST will fail or be sent to the wrong endpoint.*

---

## 8. Suggested Improvements (TODO)

1. **Resource Management Refactor** – Wrap `CloseableHttpClient` and `CloseableHttpResponse` in try‑with‑resources; close them explicitly to avoid socket leaks.  
2. **Attribute Mapping Modernization** – Replace the reflection loop with a static map or use Jackson’s `@JsonAnyGetter` to serialize a `Map<String,String>` of attributes, improving performance and error visibility.  

*(Additional enhancements such as response parsing, configurable back‑off, and enum‑driven status handling are also advisable but beyond the immediate scope.)*