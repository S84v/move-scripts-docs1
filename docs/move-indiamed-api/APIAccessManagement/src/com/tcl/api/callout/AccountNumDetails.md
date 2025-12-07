**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\AccountNumDetails.java`  

---

## 1. High‑Level Summary
`AccountNumDetails` is a utility class that contacts the Teleena “Account Number” REST API to resolve a single **account number** for a given device identifier (`SECS_ID`). It builds a JSON request from the supplied `EIDRawData`, posts it to the endpoint defined in `APIConstants`, parses the JSON response into model objects, and returns the account number when exactly one account is found; otherwise it returns `null`. All activity is logged via Log4j.

---

## 2. Key Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`AccountNumDetails`** (class) | Encapsulates the HTTP call‑out logic for the Account Number API. |
| **`getAccountNumDetails(EIDRawData eid)`** (public) | • Constructs request payload (`ProductData`). <br>• Sends HTTP POST to `APIConstants.ACNT_API_URL`. <br>• Handles HTTP status codes, logs outcomes. <br>• Parses response (`AccountResponseData → AccountDetailsResponse → List<AccountDetail>`). <br>• Returns the single account number or `null`. |
| **`getStackTrace(Exception e)`** (private static) | Utility to convert an exception’s stack trace to a `String` for logging. |

### Dependent Types (defined elsewhere)

| Type | Usage |
|------|-------|
| `EIDRawData` | Input DTO containing `secSid` (the SECS_ID). |
| `ProductData`, `CustomProductAttribute`, `ServiceType` | Request body model objects. |
| `AccountResponseData`, `AccountDetailsResponse`, `AccountDetail` | Response model hierarchy. |
| `APIConstants` | Holds static configuration values (URL, headers, auth token, entity, service type). |
| `Gson` | JSON serialization / deserialization. |
| `CloseableHttpClient`, `HttpPost`, `CloseableHttpResponse` | Apache HttpClient for the REST call. |
| `Logger` (Log4j) | Operational logging. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Input** | `EIDRawData eid` – must provide a non‑null `secSid` (SECS_ID). |
| **Output** | `String` – the resolved account number when exactly one account exists; otherwise `null`. |
| **Side‑Effects** | • HTTP request to external Teleena API.<br>• Log entries (INFO/ERROR) to the configured Log4j appender.<br>• Network I/O (socket creation, TLS handshake). |
| **Assumptions** | • `APIConstants` values are correctly populated (URL, auth header, content‑type). <br>• The API returns JSON matching the model classes. <br>• Only one account per SECS_ID is expected; multiple accounts are treated as an error condition. <br>• The caller handles `null` results appropriately. |
| **External Services** | - **Account Number REST endpoint** (`APIConstants.ACNT_API_URL`). <br>- **Authentication** via static header (`APIConstants.ACNT_AUTH`). |
| **External Config** | - `APIConstants` likely reads from `APIAcessMgmt.properties` (or similar). <br>- Log4j configuration (`log4j.properties`). |

---

## 4. Integration Points & Call Flow

1. **Upstream callers** – `APICallout` or `APIMain` (other classes in `com.tcl.api.callout`) instantiate `AccountNumDetails` and invoke `getAccountNumDetails`.  
2. **Downstream usage** – The returned account number is typically used to enrich mediation data before persisting to Hive tables (e.g., `traffic_details_raw_daily_with_no_dups`).  
3. **Configuration source** – `APIConstants` pulls values from property files (`APIAcessMgmt.properties`) that are packaged with the `move-indiamed-api` module.  
4. **Logging** – All log statements flow to the central Log4j configuration, which may forward to files, syslog, or a monitoring platform.  

*Note:* The class does **not** expose any public static entry point; it is intended to be used as a library component within the larger “move‑indiamed‑api” orchestration.

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Network/HTTP failures** (timeouts, DNS, TLS) | API call fails → `null` account number, downstream data may be incomplete. | Add configurable timeouts and retry logic (exponential back‑off). Use `try‑with‑resources` to guarantee client/response closure. |
| **Authentication/Authorization errors** (401/403) | No data returned; logs only error. | Externalize auth token to a secure vault; rotate regularly. Alert on repeated 401/403. |
| **Multiple accounts for a SECS_ID** | Returns `null` silently; downstream may treat as missing data. | Surface a warning/exception to the caller, or decide on a deterministic selection rule. |
| **Resource leakage** (client/response not closed on exception) | Exhaustion of sockets/threads. | Use `try‑with‑resources` for `CloseableHttpClient` and `CloseableHttpResponse`. |
| **Sensitive data in logs** (SECS_ID, auth header) | Potential compliance breach. | Mask or omit sensitive fields in log statements; configure Log4j to filter. |
| **Hard‑coded constants** (entity, service type) | Changes require code redeploy. | Move these values into the same property file as the URL/auth token. |

---

## 6. Running / Debugging the Class

1. **Compile** – The module is built with Maven/Gradle; ensure dependencies (`httpclient`, `gson`, `log4j`) are present.  
   ```bash
   mvn clean compile
   ```
2. **Unit Test** – Write a JUnit test that injects a mock `CloseableHttpClient` (e.g., using Mockito) and supplies a fabricated `EIDRawData`. Verify the returned account number for the single‑account case and `null` for multi‑account responses.  
3. **Standalone Execution (quick sanity)**  
   ```java
   public static void main(String[] args) throws Exception {
       AccountNumDetails anc = new AccountNumDetails();
       EIDRawData eid = new EIDRawData();
       eid.setSecsid("1234567890");
       System.out.println("Account: " + anc.getAccountNumDetails(eid));
   }
   ```
   Run with the appropriate classpath and ensure `APIConstants` points to a test endpoint.  
4. **Debugging** – Set breakpoints inside `getAccountNumDetails` (especially before the HTTP call and after response parsing). Verify the constructed JSON payload (`post` + `jsonRequestBody`) and the raw response string (`temp`).  
5. **Log Inspection** – Review the Log4j output (usually under `logs/` as defined in `log4j.properties`) for INFO lines showing request bodies and response payloads, and ERROR lines for any HTTP status issues.

---

## 7. External Configuration & Environment Variables

| Config Item | Source | Usage |
|-------------|--------|-------|
| `APIConstants.ACNT_API_URL` | `APIAcessMgmt.properties` (or similar) | Target REST endpoint. |
| `APIConstants.ACNT_AUTH` / `APIConstants.ACNT_VALUE` | Same property file | HTTP `Authorization` header (basic auth token). |
| `APIConstants.CONTENT_TYPE` / `CONTENT_VALUE` | Same property file | `Content-Type: application/json`. |
| `APIConstants.ACNT_ENTITY` | Same property file | Fixed value for request field `tclEntity`. |
| `APIConstants.ACNT_SERVICE` | Same property file | Service type string added to request. |
| Log4j configuration (`log4j.properties`) | `move-indiamed-api\APIAccessManagement\src\log4j.properties` | Controls log destinations and levels. |

If any of these properties are missing or malformed, the HTTP call will fail or be rejected.

---

## 8. Suggested Improvements (TODO)

1. **Resource Management Refactor** – Convert the HTTP client usage to a *try‑with‑resources* block to guarantee closure even when exceptions occur.  
2. **Retry & Circuit‑Breaker** – Introduce a configurable retry policy (e.g., 3 attempts with back‑off) and a circuit‑breaker (e.g., using Resilience4j) to protect downstream processing from prolonged API outages.  

---