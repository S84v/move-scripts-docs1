**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\CustomPlanDetails.java`

---

## 1. High‑Level Summary
`CustomPlanDetails` is a utility class that contacts the **Geneva Order Management “custom‑plan” REST API** to retrieve pricing and plan details for a given billing account. It builds a JSON request from an `EIDRawData` record, posts it to the endpoint defined in `APIConstants.PLAN_API_URL`, parses the JSON response into domain objects, and returns a flat `HashMap<String,String>` containing currency, product name, NRC, recurring charge, frequency, start/end dates, and plan name. The method is used by the API Access Management “move” workflow to enrich order records before they are persisted or forwarded to downstream systems.

---

## 2. Key Classes & Functions

| Class / Method | Responsibility |
|----------------|----------------|
| **`CustomPlanDetails`** (class) | Encapsulates the HTTP call to the custom‑plan API and response mapping. |
| `HashMap<String,String> getCustomPlanDetails(EIDRawData eid, String acntNum)` | Core method: builds request body, performs POST with retry logic, extracts needed fields, returns them in a map. |
| **`APIConstants`** (referenced) | Holds static configuration: endpoint URL, header names/values, retry count, product name constants (`SAFE_CHARGE`, `ADDON_CHARGE`, `TEXT_CHARGE`), etc. |
| **DTO / Model classes** (`EIDRawData`, `Accountdetails`, `ProductDetail`, `CustomResponseData`, `CustomerData`, `CustomProduct`, `PlanNameDetails`) | Simple POJOs used for JSON (de)serialization via Gson. |
| **`logger`** (Log4j) | Logs request payload, HTTP response, and error conditions. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | `EIDRawData eid` – contains status, plantype, planname, etc.<br>`String acntNum` – billing account number to query. |
| **Outputs** | `HashMap<String,String>` – keys: `currency`, `prodName`, `nrc`, `rc`, `freq`, `sdate`, `edate`, `pname`. Empty map on failure. |
| **Side Effects** | - Outbound HTTPS POST to the custom‑plan API.<br>- Log entries (INFO/ERROR) via Log4j.<br>- Opens a `CloseableHttpClient` and `CloseableHttpResponse` (closed in finally block). |
| **Assumptions** | - `APIConstants.PLAN_API_URL`, header names/values, and `RETRY_COUNT` are correctly populated from property files (e.g., `APIAcessMgmt.properties`).<br>- The API endpoint is reachable from the execution host and uses basic auth supplied in the constants.<br>- `eid.getPlanname()` may contain a delimiter `___`; the code extracts the prefix if present.<br>- The response JSON matches the structure of `CustomResponseData` hierarchy. |
| **External Services** | - **Geneva Order Management API** (HTTPS).<br>- **Log4j** logging subsystem.<br>- **Gson** library for JSON handling.<br>- **Apache HttpClient** for HTTP transport. |
| **External Config / Env** | - `APIConstants` values are typically loaded from property files (`APIAcessMgmt.properties` or similar).<br>- No direct environment‑variable usage in this class, but the constants may be built from env vars in the constants class. |

---

## 4. Integration Points & Call Flow

1. **Upstream callers** – Most likely invoked from `APICallout` or `APIMain` as part of the overall “move” orchestration. Those classes iterate over raw EID records, call `getCustomPlanDetails`, and merge the returned map into the larger order payload.
2. **Downstream usage** – The returned map is consumed by bulk‑insert utilities (`BulkInsert`, `BulkInsert2`) to populate staging tables or by email alert logic (`AlertEmail`) for reporting.
3. **Configuration linkage** – `APIConstants` pulls values from the property files processed earlier in the history (e.g., `APIAcessMgmt.properties`). Any change to endpoint URLs, auth headers, or retry count must be reflected there.
4. **Error handling propagation** – The method swallows `IOException` (prints stack trace) and returns an empty map; callers must check for missing keys to detect failures.

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Network/HTTP failures** – limited to `APIConstants.RETRY_COUNT` (default unknown) and simple `Thread.sleep(1000)` between attempts. | Potential data gaps if all retries fail. | Externalize retry count and back‑off strategy; surface failure via exception rather than silent empty map. |
| **Resource leak** – `response.close()` is called unconditionally, but if `response` is never assigned (e.g., exception before assignment) a `NullPointerException` may occur. | Application thread may hang or leak sockets. | Use try‑with‑resources for `CloseableHttpClient` and `CloseableHttpResponse`. |
| **Hard‑coded header values** – `APIConstants.ACNT_AUTH` may contain static credentials. | Credential rotation requires code change/redeploy. | Load auth token from a secure vault or environment variable; rotate automatically. |
| **JSON schema drift** – If the API adds fields or changes naming, deserialization may fail silently (null objects). | Missing or incorrect pricing data. | Validate response against a schema; log warnings when expected fields are null. |
| **Logging of sensitive data** – Full request JSON is logged at INFO level. | Potential exposure of account numbers or auth data. | Redact or downgrade to DEBUG; ensure log retention policies. |
| **Single‑threaded HttpClient** – New client per call; high volume may cause socket exhaustion. | Performance degradation under load. | Reuse a pooled `HttpClient` or configure connection manager. |

---

## 6. Running / Debugging the Class

1. **Prerequisites**  
   - Java 8+ runtime, Maven/Gradle build with dependencies: `gson`, `httpclient`, `log4j`.  
   - Property files (`APIAcessMgmt.properties`) on the classpath so `APIConstants` resolves correctly.  
   - Network access to the Geneva Order Management API endpoint.

2. **Typical invocation (from another Java class or unit test)**  

```java
EIDRawData eid = new EIDRawData();
eid.setStatusproduct("Active");
eid.setPlantype("Commercial");
eid.setPlanname("100MB_500SMS___XYZ");

String accountNo = "MVI000177";

CustomPlanDetails cp = new CustomPlanDetails();
HashMap<String,String> planInfo = cp.getCustomPlanDetails(eid, accountNo);

if (planInfo.isEmpty()) {
    System.err.println("Failed to retrieve plan details");
} else {
    planInfo.forEach((k,v) -> System.out.println(k + " = " + v));
}
```

3. **Debugging tips**  
   - Set Log4j level to `DEBUG` for `com.tcl.api.callout.CustomPlanDetails` to see request/response bodies.  
   - Use a network sniffer (e.g., Wireshark or Fiddler) to verify the POST payload and headers.  
   - If the method returns an empty map, check the log for HTTP status codes and stack traces.  
   - Attach a breakpoint on the line `response = httpclient.execute(post);` to inspect the raw `HttpResponse`.  

4. **Unit test stub**  

```java
@Test
public void testGetCustomPlanDetails_success() throws Exception {
    // Mock HttpClient using WireMock or Mockito to return a predefined JSON payload.
    // Verify that the returned map contains expected keys/values.
}
```

---

## 7. External Configuration & Files

| Config Source | Key(s) Used by this Class |
|---------------|---------------------------|
| `APIConstants` (populated from `APIAcessMgmt.properties` or similar) | `PLAN_API_URL`, `CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE`, `RETRY_COUNT`, `SAFE_CHARGE`, `ADDON_CHARGE`, `TEXT_CHARGE`, `COMMERCIAL` |
| Log4j configuration (`log4j.properties`) | Controls logging output for `APICallout` logger used here. |
| Potential environment variables (if `APIConstants` reads them) | API credentials, endpoint overrides for dev/UAT/Prod. |

---

## 8. Suggested Improvements (TODO)

1. **Resource Management & Error Propagation**  
   - Refactor the HTTP call to use *try‑with‑resources* for `CloseableHttpClient` and `CloseableHttpResponse`.  
   - Replace the blanket `catch (IOException e)` with a custom exception (`CustomPlanException`) that propagates failure details to callers.

2. **Configuration & Security Hardenings**  
   - Externalize the retry count and back‑off policy to a property (e.g., `customplan.retry.count`, `customplan.retry.delay`).  
   - Move authentication credentials out of `APIConstants` into a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and inject them at runtime.

---