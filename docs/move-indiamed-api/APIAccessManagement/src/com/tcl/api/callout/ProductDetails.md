**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\ProductDetails.java`

---

## 1. High‑Level Summary
`ProductDetails` is a call‑out utility that retrieves detailed product‑ and account‑level attributes from the **Geneva Order Management API** for a given billing account. It builds a JSON request based on the incoming `EIDRawData` record, posts it to the external service, parses the response into a rich `HashMap<String,Object>` that downstream move‑scripts consume (e.g., order creation, billing, provisioning). The map contains customer, account, product, address, and partner information required for downstream orchestration.

---

## 2. Key Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`ProductDetails`** (class) | Central call‑out component; orchestrates request creation, HTTP execution, response parsing, and map population. |
| **`getProductDetails(EIDRawData eid, String acntNum)`** (method) | • Builds request payload (`ProductInput`). <br>• Determines product name (`SAFE_CHARGE`, `ADDON_CHARGE`, or `TEXT_CHARGE`) based on `eid`. <br>• Executes HTTP POST with retry logic (`APIConstants.RETRY_COUNT`). <br>• Parses `ProductAttributeResponse` → extracts >150 fields into a `HashMap`. <br>• Returns the populated map. |
| **`APIConstants`** (external) | Holds static configuration: endpoint URL (`PROD_ATTR_URL`), HTTP headers (`CONTENT_TYPE`, `ACNT_AUTH`), retry count, etc. |
| **Model / DTO classes** (`EIDRawData`, `ProductInput`, `ProductAttributeResponse`, `ProductAttribute`, `Product`, `Address`, …) | Represent request and response structures; used for JSON (de)serialization via Gson. |
| **`logger`** (Log4j) | Emits request payload, raw response, and diagnostic messages. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | `EIDRawData eid` – contains transaction status, plan type, statusproduct, transaction date, etc.<br>`String acntNum` – billing account number passed to the API. |
| **Outputs** | `HashMap<String,Object>` – key/value pairs for every attribute extracted (customerRef, custSeg, product‑specific fields, address lines, GST info, partner data, commissioning dates, etc.). |
| **Side Effects** | • Outbound HTTPS POST to `APIConstants.PROD_ATTR_URL`.<br>• Logging of request and raw response (potentially sensitive data).<br>• Network I/O; consumes a TCP socket. |
| **Assumptions** | • `eid` and `acntNum` are non‑null and contain required fields.<br>• `APIConstants` values are correctly populated (URL, auth header).<br>• The external API is reachable and returns JSON matching the model classes.<br>• The JVM has internet access and appropriate TLS certificates. |

---

## 4. Integration Points & Call Flow

1. **Up‑stream** – Typically invoked by higher‑level move scripts (e.g., `OrderStatus`, `PostOrder`) after an event record is read from the source system. Those scripts pass the `EIDRawData` object derived from the inbound payload.
2. **Down‑stream** – The returned `HashMap` is consumed by:
   * Database bulk‑insert utilities (`BulkInsert`, `BulkInsert2`) to persist product attributes.
   * Transformation scripts that build downstream API calls (e.g., provisioning, billing).
   * Alerting utilities (`AlertEmail`) when required fields are missing.
3. **Configuration** – All endpoint URLs, header values, and retry counts live in `APIConstants`. Changing environments (DEV/QA/PROD) is done by swapping the constants file or loading a properties file that populates `APIConstants`.
4. **External Service** – **Geneva Order Management API** (`/productAttributes`). Authentication is performed via a static header (`ACNT_AUTH`) that contains a base‑64 encoded Teleena user/password (managed outside the code).

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Network/HTTP failures** – limited retry (default `RETRY_COUNT`) may still exhaust on prolonged outage. | Partial or missing data downstream, job failure. | Increase retry count, add exponential back‑off, and surface a clear error code to the orchestrator. |
| **Resource leaks** – `response.close()` and `httpclient.close()` are called twice; if an exception occurs before the first close, resources stay open. | Socket exhaustion under load. | Refactor using Java *try‑with‑resources* for `CloseableHttpClient` and `CloseableHttpResponse`. |
| **Sensitive data in logs** – full request/response JSON is logged at INFO level. | Compliance breach, credential leakage. | Log at DEBUG level, mask or omit PII/credential fields, and ensure log rotation. |
| **Hard‑coded product‑name logic** – relies on exact string values (`"Suspended"`, `"COMMERCIAL"`). | Future product changes break mapping. | Externalize mapping to a configuration file or enum. |
| **Large map size** – >150 entries per product; may cause memory pressure when processing many records concurrently. | Out‑of‑memory errors. | Stream processing, or split map into logical sub‑maps; consider using a POJO instead of a generic map. |
| **No explicit error propagation** – method returns an empty map on exception, losing context. | Silent failures, downstream null pointer errors. | Throw a custom checked exception or return an `Optional<HashMap>` with error details. |

---

## 6. Running / Debugging the Component

### Stand‑alone Invocation (for dev / testing)

```java
// Prepare a minimal EIDRawData instance (populate required fields)
EIDRawData eid = new EIDRawData();
eid.setStatusproduct("Active");
eid.setPlantype("Residential");
eid.setTransactionstatus("New");
eid.setTransactiondate("2023-07-01 12:00:00.0000000");
eid.setEid("1234567890");
eid.setPlanname("BasicPlan");

// Billing account number from source system
String acctNum = "MVI000177";

ProductDetails pd = new ProductDetails();
HashMap<String,Object> result = pd.getProductDetails(eid, acctNum);

// Inspect result
result.forEach((k,v) -> System.out.println(k + " = " + v));
```

### Debugging Tips

| Step | Action |
|------|--------|
| **Enable detailed logging** | Set Log4j level to `DEBUG` for `com.tcl.api.callout.ProductDetails`. |
| **Mock the HTTP call** | Use a library such as **WireMock** or **Mockito** to stub `PROD_ATTR_URL` and return a static JSON payload. This isolates parsing logic. |
| **Validate request payload** | The method logs the JSON request (`logger.info(jsonRequestBody)`). Verify product name selection logic matches expectations. |
| **Check retry behavior** | Force a 500 response on the first call; ensure the loop retries up to `APIConstants.RETRY_COUNT`. |
| **Resource leak detection** | Run with a profiler (e.g., VisualVM) or enable `-Dorg.apache.http.impl.execchain.HttpRequestExecutor.debug=true` to ensure connections close. |

---

## 7. External Configuration & Environment Variables

| Config Source | Key / Variable | Usage |
|---------------|----------------|-------|
| `APIConstants` (Java class / properties file) | `PROD_ATTR_URL` | Target endpoint for product‑attribute API. |
| `APIConstants` | `CONTENT_TYPE`, `CONTENT_VALUE` | HTTP `Content-Type` header (usually `application/json`). |
| `APIConstants` | `ACNT_AUTH`, `ACNT_VALUE` | Authentication header (e.g., `Authorization: Basic <base64>`). |
| `APIConstants` | `RETRY_COUNT` | Number of HTTP retry attempts on non‑200 responses. |
| **Potential env vars** (not shown in code) | May be used by `APIConstants` to inject URL or credentials per environment (DEV/QA/PROD). Verify the constants class for `System.getenv` usage. |

---

## 8. Suggested Improvements (TODO)

1. **Resource Management Refactor** – Convert the HTTP client usage to *try‑with‑resources* to guarantee proper closure, and remove duplicate `close()` calls.
2. **Separate Mapping Logic** – Extract the massive field‑by‑field map population into a dedicated mapper class (e.g., `ProductResponseMapper`). This improves readability, testability, and allows unit tests for each mapping rule.

---