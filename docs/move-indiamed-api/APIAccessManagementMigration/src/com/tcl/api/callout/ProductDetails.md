**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\ProductDetails.java`

---

## 1. High‑Level Summary
`ProductDetails` is a call‑out component that retrieves detailed product‑level information for a given billing account from the **Geneva Order Management API** (UAT endpoint). It builds a JSON request based on the transaction’s plan type, posts it to the `/productAttributes` service, parses the complex response hierarchy into a flat `HashMap<String,Object>` that downstream migration/orchestration scripts consume (e.g., order creation, data‑load, reporting). The method is invoked with an `EIDRawData` record (containing transaction metadata) and the target account number.

---

## 2. Key Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`ProductDetails`** (class) | Encapsulates the product‑attribute call‑out logic. |
| **`getProductDetails(EIDRawData eid, String acntNum)`** | Main public method – builds request, performs HTTP POST with retry, parses response, populates a flat `HashMap` of all needed fields, and returns it. |
| **`APIConstants`** (import) | Holds static configuration: endpoint URL, HTTP headers, retry count, content‑type values, and constant strings such as `COMMERCIAL`, `ADDON_CHARGE`, `TEXT_CHARGE`. |
| **Model classes** (`ProductInput`, `ProductName`, `ProductAttributeResponse`, `ProductAttribute`, `Product`, `AccountAttributes`, `CustomerAttributes`, `Address`, `EventSourceDetails`, `PropositionDetails`, etc.) | POJOs generated from the API schema; used by Gson for (de)serialization. |
| **`logger`** (Log4j) | Emits request payload, raw response, and error conditions. |

*Note:* The class does **not** expose any other public API; all other call‑outs (`AccountNumDetails`, `AlertEmail`, `BulkInsert`, etc.) are separate components that may consume the map returned here.

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • `EIDRawData eid` – contains transaction status, date, plan type, and EID.<br>• `String acntNum` – billing account number to query.<br>• Implicit configuration from `APIConstants` (URL, headers, retry count). |
| **Outputs** | `HashMap<String,Object>` – a flat key/value collection of >150 fields (customer, account, product, address, GST, billing, partner, proposition, usage, etc.). Keys are short strings (e.g., `customerRef`, `servType`, `pdseq`, `commissioningDate`). |
| **Side‑Effects** | • Outbound HTTPS POST to the external **Geneva Order Management API**.<br>• Logging to the application log (INFO/ERROR).<br>• Network I/O; consumes a TCP socket. |
| **Assumptions** | • The API endpoint is reachable and returns JSON matching the model classes.<br>• `eid.getPlantype()` is non‑null and matches `APIConstants.COMMERCIAL` when appropriate.<br>• `eid.getTransactionstatus()` is one of `New`, `CHANGE_NEW`, or another known value.<br>• `APIConstants.ACNT_AUTH` contains a valid Basic‑Auth header (or token) pre‑populated elsewhere.<br>• No proxy or SSL certificate issues in the runtime environment. |

---

## 4. Integration Points & Call Flow

1. **Up‑stream callers** – Typically `APIMain` or other orchestration scripts iterate over EID records, instantiate `ProductDetails`, and invoke `getProductDetails`. The returned map is merged with other attribute maps (e.g., from `AccountNumDetails`, `CustomPlanDetails`) before persisting to the target system (DB, CSV, or downstream API).  
2. **Down‑stream consumers** – The map feeds bulk‑insert utilities (`BulkInsert`, `BulkInsert2`) that write to staging tables, or it is used by `PostOrder` to construct order creation payloads.  
3. **External services** – The only external dependency is the **Geneva Order Management API** (UAT). All authentication/authorization details are supplied via static headers in `APIConstants`.  
4. **Configuration files** – `APIConstants` is the single source of configuration (URL, headers, retry count). No environment variables are read directly in this class, but the constants may be populated at application start from property files or CI/CD secret stores.  

*Typical call sequence:*  

```
EIDRawData eid = ...   // from input file / DB
String acct = ...      // derived from EID or upstream logic
ProductDetails pd = new ProductDetails();
Map<String,Object> prodMap = pd.getProductDetails(eid, acct);
// prodMap merged with other maps → persisted or sent further
```

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **API unavailability / latency** | Job stalls, retries may exhaust, downstream steps receive incomplete data. | – Implement circuit‑breaker / timeout (e.g., `RequestConfig` with connect/read timeouts).<br>– Surface HTTP status in job metrics; alert on repeated 5xx/timeout. |
| **Response schema drift** | Missing fields cause `NullPointerException` or silent data loss. | – Validate response against a versioned schema; log missing mandatory fields.<br>– Add defensive null checks before accessing nested objects. |
| **Resource leakage** | Duplicate `response.close()` / `httpclient.close()` may throw `IOException` on second close, and exceptions in the `try` block bypass cleanup. | – Use try‑with‑resources for `CloseableHttpClient` and `CloseableHttpResponse`.<br>– Remove redundant close calls. |
| **Key collisions in the flat map** | Overwrites when multiple products share the same attribute name (e.g., `pdseq` from the last product only). | – Prefix keys with product index or store product list as a collection inside the map.<br>– Document expected cardinality. |
| **Hard‑coded UAT URL** | Promotion to production requires code change. | – Externalize endpoint URL to a properties file or environment variable; reference via `APIConstants`. |
| **Logging of sensitive data** | Potential exposure of account numbers or auth headers. | – Mask or omit PII in logs; configure Log4j to filter sensitive fields. |

---

## 6. Running / Debugging the Component

1. **Prerequisites**  
   - Java 8+ runtime, Maven/Gradle build with dependencies (`httpclient`, `gson`, `log4j`).  
   - `APIConstants` correctly populated (URL, headers, retry count).  
   - Network access to `https://api-uat.tatacommunications.com`.

2. **Typical invocation (from another Java class or unit test)**  

```java
EIDRawData eid = new EIDRawData();
// populate eid fields (transactionstatus, plantype, transactiondate, eid, planname, etc.)
String acct = "MVI000177";

ProductDetails pd = new ProductDetails();
Map<String,Object> result = pd.getProductDetails(eid, acct);

// Inspect result
result.forEach((k,v) -> System.out.println(k + " = " + v));
```

3. **Debugging tips**  
   - Set Log4j level to `DEBUG` for `com.tcl.api.callout.ProductDetails` to see request payload and raw JSON response.  
   - Use a network sniffer (e.g., Wireshark or Fiddler) to verify the POST body and headers.  
   - If the API returns non‑200, the method logs the status code; check the log for `400 Bad Request`, `401 Unauthorized`, etc.  
   - To step through the parsing, place breakpoints after `gson.fromJson` and inspect the generated POJOs (`ProductAttributeResponse`, `ProductAttribute`, etc.).  

4. **Unit test stub**  

```java
@Test
public void testGetProductDetails() throws Exception {
    EIDRawData eid = mockEid(); // create a minimal valid object
    String acct = "MVI000179";

    ProductDetails pd = new ProductDetails();
    Map<String,Object> map = pd.getProductDetails(eid, acct);

    assertNotNull(map.get("customerRef"));
    // additional assertions based on expected keys
}
```

---

## 7. External Configuration & Dependencies

| Item | Usage |
|------|-------|
| `APIConstants.CONTENT_TYPE` / `APIConstants.CONTENT_VALUE` | Sets `Content-Type: application/json` header. |
| `APIConstants.ACNT_AUTH` / `APIConstants.ACNT_VALUE` | Supplies authentication header (likely `Authorization: Basic …`). |
| `APIConstants.RETRY_COUNT` | Controls the maximum number of HTTP attempts. |
| `APIConstants.COMMERCIAL`, `APIConstants.ADDON_CHARGE`, `APIConstants.TEXT_CHARGE` | Determines which product name to request based on `eid.getPlantype()`. |
| **Third‑party libraries** | Apache HttpClient (`CloseableHttpClient`), Google Gson, Log4j. |
| **Model POJOs** (`com.tcl.api.model.*`) | Must be on the classpath; generated from the API contract. |

No direct environment variables are read in this file, but the constants may be populated from external property files at application start.

---

## 8. Suggested Improvements (TODO)

1. **Resource Management Refactor** – Switch to *try‑with‑resources* for `CloseableHttpClient` and `CloseableHttpResponse` to guarantee proper closure and eliminate duplicate `close()` calls.  
2. **Response Validation & Structured Output** – Instead of flattening everything into a single `HashMap`, return a typed DTO (e.g., `ProductDetailsResult`) that contains a list of `ProductInfo` objects and separate sections for customer, account, and partner data. This prevents key collisions and improves type safety for downstream consumers.  

---