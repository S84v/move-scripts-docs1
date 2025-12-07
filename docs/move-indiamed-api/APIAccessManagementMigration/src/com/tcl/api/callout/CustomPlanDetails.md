**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\CustomPlanDetails.java`

---

## 1. High‑Level Summary
`CustomPlanDetails` provides a single public method `getCustomPlanDetails` that invokes the *Custom Plans* REST endpoint of the Geneva Order Management API. It builds a JSON request from an `EIDRawData` record and a billing account number, posts the request, retries on failure, parses the JSON response into a flat `HashMap<String,String>` containing currency, product name and the first plan‑level charge details (NRC, recurring charge, frequency, start/end dates, plan name). The method is used by the API Access Management Migration job to enrich account records with pricing information retrieved from the external API.

---

## 2. Important Classes & Functions

| Class / Method | Responsibility |
|----------------|----------------|
| **`CustomPlanDetails`** (public) | Wrapper for the custom‑plan API call. Holds a Log4j logger (shared with `APICallout`). |
| **`HashMap<String,String> getCustomPlanDetails(EIDRawData eid, String acntNum)`** | *Core function* – builds request body, performs HTTP POST with retry logic, parses response, returns a map of selected fields. |
| **`Accountdetails`** (DTO) | Request payload container – holds billing account number and a list of `ProductDetail`. |
| **`ProductDetail`** (DTO) | Part of request – indicates product name (`ADDON_CHARGE` or `TEXT_CHARGE`) and a list of plan names. |
| **`CustomResponseData`**, **`CustomerData`**, **`CustomProduct`**, **`PlanNameDetails`** (DTOs) | Response model used by Gson to deserialize the JSON payload returned by the API. |
| **`APIConstants`** (utility) | Holds static configuration values: endpoint URL, header names/values, retry count, content‑type, plan‑type constants, etc. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | `EIDRawData eid` – contains `plantype` and `planname` fields used to decide product name and plan list.<br>`String acntNum` – billing account number supplied to the API. |
| **Outputs** | `HashMap<String,String>` – keys may include: `currency`, `prodName`, `nrc`, `rc`, `freq`, `sdate`, `edate`, `pname`. If the API call fails or returns no data, the map may be empty. |
| **Side Effects** | - Outbound HTTPS POST to `APIConstants.PLAN_API_URL`.<br>- Logging via Log4j (info & error).<br>- Creation of `CloseableHttpClient` and `CloseableHttpResponse` objects (network resources). |
| **Assumptions** | - `APIConstants` is correctly populated (URL, headers, retry count).<br>- The external *Geneva Order Management* service is reachable and returns JSON matching the DTO hierarchy.<br>- `eid.getPlantype()` and `eid.getPlanname()` are non‑null.<br>- Only the first `CustomProduct` and its first `PlanNameDetails` are needed (the code overwrites map entries in loops). |
| **External Services** | - HTTPS REST endpoint (`PLAN_API_URL`).<br>- Possibly a basic‑auth token embedded in `APIConstants.ACNT_AUTH` header. |
| **External Config** | - `APIConstants` values are likely loaded from property files such as `APIAcessMgmt.properties` (see HISTORY).<br>- Log4j configuration (`log4j.properties`). |

---

## 4. Integration Points & Call Flow

1. **Caller** – Other callout classes (e.g., `APICallout`, `BulkInsert`, `APIMain`) invoke `new CustomPlanDetails().getCustomPlanDetails(eid, acctNum)` to enrich a record before persisting it via DAO classes (`APIAccessDAO`).  
2. **Configuration** – `APIConstants` pulls values from the shared property files; any change to the endpoint or headers must be reflected there.  
3. **DAO Layer** – The returned map is later used by DAO insert/update statements defined in `APIAccessDAO.properties`.  
4. **Error Handling** – Errors are logged but not propagated; callers must check for an empty map to detect failure.  
5. **Batch Orchestration** – The migration job orchestrates multiple callouts (account details, custom plan, alerts) in a single transaction; this class contributes the “custom plan” piece.

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Unbounded retry loop** – If the API never returns `200 OK`, the method will exhaust `RETRY_COUNT` but still log “Unhandled Error”. | Add a back‑off strategy and fail fast after max retries, throwing a custom exception. |
| **Resource leak** – `CloseableHttpResponse` is closed only after the loop; if an exception occurs before `response.close()`, resources stay open. | Use *try‑with‑resources* for `CloseableHttpClient` and `CloseableHttpResponse`. |
| **Hard‑coded map keys** – Overwrites values when multiple products/plans exist, potentially losing data. | Return a richer structure (e.g., `List<Map<String,String>>`) or a dedicated DTO. |
| **No timeout** – Default HttpClient may wait indefinitely on network stalls. | Configure connection & socket timeouts via `RequestConfig`. |
| **Sensitive data in logs** – Full JSON request body is logged at INFO level. | Redact or downgrade to DEBUG; avoid logging credentials. |
| **Null pointer risk** – `eid.getPlantype()` or `eid.getPlanname()` could be null. | Validate inputs and throw `IllegalArgumentException` early. |

---

## 6. Running / Debugging the Class

1. **Prerequisites** – Ensure the property files (`APIAcessMgmt.properties`, `log4j.properties`) are on the classpath and contain valid `PLAN_API_URL`, header values, and `RETRY_COUNT`.  
2. **Standalone Test**  
   ```java
   public static void main(String[] args) throws Exception {
       EIDRawData eid = new EIDRawData();
       eid.setPlantype("Commercial");          // or "Residential"
       eid.setPlanname("100MB_500SMS___XYZ");
       String acct = "MVI000177";

       CustomPlanDetails cp = new CustomPlanDetails();
       Map<String,String> result = cp.getCustomPlanDetails(eid, acct);
       System.out.println("Result: " + result);
   }
   ```
   Run with the appropriate JVM options to pick up Log4j config (`-Dlog4j.configuration=file:log4j.properties`).  

3. **Debugging Tips**  
   - Set Log4j level to `DEBUG` for `com.tcl.api.callout` to see request/response bodies.  
   - Use a network sniffer (e.g., Wireshark or Fiddler) to verify the outbound POST.  
   - If the method returns an empty map, check the logs for the “Unhandled Error” line and the HTTP status code.  

---

## 7. External Configuration & Environment Variables

| Config Source | Key / Variable | Usage |
|---------------|----------------|-------|
| `APIConstants` (populated from property files) | `PLAN_API_URL` | Target endpoint for the POST request. |
| `APIConstants` | `CONTENT_TYPE`, `CONTENT_VALUE` | HTTP `Content-Type` header (e.g., `application/json`). |
| `APIConstants` | `ACNT_AUTH`, `ACNT_VALUE` | Authentication header (likely a Basic token). |
| `APIConstants` | `RETRY_COUNT` | Number of retry attempts on non‑200 responses. |
| `APIConstants` | `COMMERCIAL`, `ADDON_CHARGE`, `TEXT_CHARGE` | Business constants used to select product name. |
| `log4j.properties` | Log level, appenders | Controls logging output for troubleshooting. |

If any of these values are missing or malformed, the call will fail silently (empty map) and log an error.

---

## 8. Suggested Improvements (TODO)

1. **Refactor resource handling** – Switch to *try‑with‑resources* for `CloseableHttpClient` and `CloseableHttpResponse` and ensure all streams are closed even on exception.  
2. **Return a typed DTO** – Replace the flat `HashMap<String,String>` with a dedicated `CustomPlanResult` object that can hold multiple products/plans and preserves the full response hierarchy, improving downstream readability and reducing data loss.  

---