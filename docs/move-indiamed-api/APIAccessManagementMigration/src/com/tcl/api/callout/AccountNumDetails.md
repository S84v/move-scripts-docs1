**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\AccountNumDetails.java`

---

## 1. High‑Level Summary
`AccountNumDetails` is a utility class that contacts the **Teleena Account Number Service** (ACNT API) to retrieve the single account number associated with a given subscriber (identified by `SECS_ID`). It builds a JSON request from an `EIDRawData` DTO, posts it via Apache HttpClient, parses the JSON response into model objects, and returns the account number when exactly one account is found; otherwise it returns `null`. All logging is performed through Log4j.

---

## 2. Important Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`AccountNumDetails`** (class) | Encapsulates the ACNT API call logic; stateless utility. |
| **`getAccountNumDetails(EIDRawData eid)`** | *Input*: `EIDRawData` containing `secsid`. <br>*Output*: `String` account number or `null`. <br>Builds request, sends HTTP POST, handles response codes, extracts the account number from the JSON payload. |
| **`getStackTrace(Exception e)`** (private static) | Helper that converts an exception’s stack trace to a `String` for logging. |
| **`logger`** (static Log4j `Logger`) | Centralised logging for request/response and error conditions. |

*Supporting model classes* (`ProductData`, `CustomProductAttribute`, `ServiceType`, `AccountResponseData`, `AccountDetailsResponse`, `AccountDetail`) are defined elsewhere in the `com.tcl.api.model` package and represent the request/response schema.

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Input** | `EIDRawData eid` – must provide a non‑null `secsid` (subscriber identifier). |
| **Output** | `String` – the account number when a single account is returned; `null` otherwise (multiple or none). |
| **Side Effects** | - Outbound HTTP POST to `APIConstants.ACNT_API_URL`. <br>- Log entries (INFO/ERROR) written to the application log (configured via `log4j.properties`). |
| **Assumptions** | - `APIConstants` holds correct endpoint URL, content‑type header, and static Authorization header (`APIConstants.ACNT_VALUE`). <br>- The ACNT service returns JSON matching the model classes. <br>- Network connectivity and TLS certificates are correctly configured on the host. <br>- Caller handles `null` result appropriately. |
| **External Services** | - **Teleena ACNT REST API** (HTTPS). <br>- **Log4j** logging subsystem. |
| **External Config** | - `APIConstants` values are populated from property files (`APIAcessMgmt.properties` or similar). <br>- Log4j configuration (`log4j.properties`). |

---

## 4. Connection to Other Scripts & Components

| Component | Relationship |
|-----------|--------------|
| **`APICallout` / `APIMain`** (src/com/tcl/api/callout) | Likely instantiate `AccountNumDetails` to enrich subscriber data before persisting or forwarding to downstream systems. |
| **`APIAccessDAO`** (SQL‑statement repository) | May store the returned account number in the migration database; not directly referenced here but part of the same job flow. |
| **`APIConstants`** | Central constants class used across the migration job; values are read from the property file `APIAcessMgmt.properties`. |
| **`EIDRawData` DTO** | Produced earlier in the pipeline (e.g., from source file parsing or DB read). |
| **Downstream systems** | After obtaining the account number, other components may push data to CRM, billing, or provisioning systems. |

---

## 5. Potential Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Hard‑coded Authorization header** (`APIConstants.ACNT_VALUE`) may expire or be leaked. | Store credentials in a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and inject at runtime; rotate regularly. |
| **No timeout / retry logic** – a hung request can stall the batch. | Configure `RequestConfig` with connection and socket timeouts; implement exponential back‑off retry for transient HTTP errors (5xx, network glitches). |
| **Resource leak on exception** – `CloseableHttpResponse`/`CloseableHttpClient` may stay open if an exception occurs before `close()`. | Use try‑with‑resources (`try (CloseableHttpClient client = HttpClients.createDefault(); CloseableHttpResponse resp = client.execute(post)) { … }`). |
| **Assumes exactly one account** – returns `null` silently for multiple accounts, potentially losing data. | Return the list of accounts or raise a business exception; add monitoring/alerting for the “multiple accounts” case. |
| **Logging of full request body** may expose sensitive subscriber identifiers. | Mask or omit PII in logs; use a logging filter or conditional debug level. |
| **No validation of `eid.getSecsid()`** – null or malformed values cause malformed JSON. | Validate input early; log and skip invalid records. |

---

## 6. Example Run / Debug Workflow

1. **Build & Package**  
   ```bash
   cd move-indiamed-api/APIAccessManagementMigration
   mvn clean package   # assumes Maven; produces a jar with all classes
   ```

2. **Execute the migration job** (normally via the main class):  
   ```bash
   java -cp target/APIAccessManagementMigration.jar com.tcl.api.callout.APIMain \
        -Dconfig.file=../PropertyFiles/APIAcessMgmt.properties
   ```

3. **Debugging `AccountNumDetails`**  
   - Set a breakpoint inside `getAccountNumDetails`.  
   - Run the job in an IDE (IntelliJ/Eclipse) with the same classpath and VM arguments.  
   - Verify that `APIConstants.ACNT_API_URL`, `ACNT_AUTH`, and `ACNT_VALUE` resolve to the expected values (print them or inspect in debugger).  
   - Use a network capture tool (e.g., Wireshark or `tcpdump`) or enable HttpClient wire logging (`org.apache.http.wire` logger) to see the raw request/response.  
   - If the method returns `null`, check the log for “Multiple accounts are created” or HTTP error codes.

4. **Unit Test (recommended)**  
   ```java
   @Test
   public void testGetAccountNumDetails_singleAccount() throws Exception {
       EIDRawData eid = new EIDRawData();
       eid.setSecsid("1234567890");
       AccountNumDetails anc = new AccountNumDetails();
       String acct = anc.getAccountNumDetails(eid);
       assertNotNull(acct);
   }
   ```
   Use a mock HTTP server (e.g., WireMock) to simulate ACNT responses.

---

## 7. External Configuration / Environment Variables

| Config Source | Key / Variable | Usage |
|---------------|----------------|-------|
| `APIConstants` (populated from `APIAcessMgmt.properties`) | `ACNT_API_URL` | Base URL for the POST request. |
| `APIConstants` | `ACNT_AUTH` (header name) | Header name for the static auth token. |
| `APIConstants` | `ACNT_VALUE` (header value) | Authorization token/value sent with every request. |
| `APIConstants` | `CONTENT_TYPE` / `CONTENT_VALUE` | `Content-Type: application/json`. |
| `log4j.properties` | Log level, appenders, file locations | Controls where `logger.info/error` messages are written. |
| **Potential env vars** (not shown in code) | `JAVA_OPTS`, `HTTPS_PROXY` | May affect HTTP client behavior; verify in the deployment environment. |

---

## 8. Suggested TODO / Improvements

1. **Add robust HTTP client handling** – switch to try‑with‑resources, configure timeouts, and implement configurable retry policy.  
2. **Externalize the Authorization token** – read from a secure secret store or encrypted property rather than a static constant; add token refresh logic if required.

---