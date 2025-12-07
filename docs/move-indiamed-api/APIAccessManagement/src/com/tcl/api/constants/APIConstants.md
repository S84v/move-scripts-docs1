**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\constants\APIConstants.java`

---

## 1. High‑Level Summary
`APIConstants` is a pure‑Java constants holder used throughout the *API Access Management* module. It centralises URLs, HTTP header names/values, authentication tokens, service identifiers, charge descriptors, and retry policies that are required by the various call‑out classes (`OrderStatus`, `PostOrder`, `ProductDetails`, `UsageProdDetails`, etc.). By providing a single source of truth, the class enables consistent request construction and simplifies future endpoint or credential changes.

---

## 2. Important Class & Members  

| Member | Type | Responsibility / Meaning |
|--------|------|---------------------------|
| `ACNT_API_URL` | `String` | Base endpoint for **account‑number** operations. |
| `PLAN_API_URL` | `String` | Endpoint for **custom‑plan** CRUD. |
| `PROD_ATTR_URL` | `String` | Endpoint for **product‑attribute** queries. |
| `ORDR_ATTR_URL` | `String` | Endpoint for **order** actions. |
| `USAGE_ATTR_URL` | `String` | Base URL for **usage‑details** (expects a trailing identifier). |
| `CONTENT_TYPE` | `String` | HTTP header name “Content‑Type”. |
| `CONTENT_VALUE` | `String` | Header value “application/json”. |
| `ACNT_AUTH` | `String` | HTTP header name “Authorization”. |
| `ACNT_VALUE` | `String` | **Basic** auth token (Base64‑encoded credentials) used for all API calls. |
| `ACNT_ENTITY` | `String` | Logical entity identifier (“TCCSPL”). |
| `ACNT_SERVICE` | `String` | Service name displayed in audit logs (“MOVE IOT Connect”). |
| `ADDON_CHARGE`, `TEXT_CHARGE` | `String` | Human‑readable charge type descriptors. |
| `COMMERCIAL`, `TEST` | `String` | Environment tags used in payloads or routing logic. |
| `RETRY_COUNT` | `int` | Number of automatic retry attempts for transient failures. |
| `SUSPEND` | `String` | Status flag used by orchestration logic. |
| `SAFE_CHARGE` | `String` | Descriptor for “Safe Custody” fees. |

*No methods* – the class is a static container only.

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Detail |
|--------|--------|
| **Inputs** | None at runtime; values are hard‑coded. |
| **Outputs** | None – constants are read‑only. |
| **Side‑effects** | None. |
| **Assumptions** | • All call‑out classes import `APIConstants` and rely on the exact constant names. <br>• The Basic auth token (`ACNT_VALUE`) is valid for the target environment and does **not** need rotation at runtime. <br>• URLs are reachable from the JVM host and use TLS 1.2+. <br>• `RETRY_COUNT` is honoured by the HTTP client wrapper used in other classes. |

---

## 4. Integration Points  

| Consumer | How it uses the constant(s) |
|----------|-----------------------------|
| `com.tcl.api.callout.*` (e.g., `OrderStatus`, `PostOrder`, `ProductDetails`, `UsageProdDetails`) | Build HTTP requests: URL, headers (`CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`/`ACNT_VALUE`), payload tags (`COMMERCIAL`, `TEST`). |
| `com.tcl.api.connection.SQLServerConnection` | May log `ACNT_ENTITY` / `ACNT_SERVICE` for audit trails. |
| Logging / Monitoring utilities | Use `ACNT_SERVICE` to tag logs; `SAFE_CHARGE` etc. for metric categorisation. |
| Retry logic (custom wrapper) | Reads `RETRY_COUNT` to decide how many times to re‑invoke a failed HTTP call. |
| Orchestration scripts (outside Java) | May reference the same constant values via a generated properties file or by reading the compiled class (rare). |

*Note:* Because the constants are static final, any change requires a recompilation of dependent modules and a redeployment of the JAR.

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hard‑coded Basic auth token** – token may expire or be compromised. | Service outage or security breach. | Store credentials in a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and load at runtime; deprecate the static constant. |
| **Endpoint URLs change** (e.g., new API version). | Calls fail with 404/401. | Externalise URLs to a configuration file (`application.properties` or environment variables) and reload without code change. |
| **TLS certificate rotation** – URL uses a fixed host; certificate updates may break Java trust store. | Connection failures. | Use a standard trust store and enable hostname verification; monitor certificate expiry. |
| **Retry count too low** – transient network glitches cause order loss. | Incomplete data movement. | Make `RETRY_COUNT` configurable and expose a back‑off strategy. |
| **Static constants prevent hot‑swap** – any change forces a full redeploy. | Increased change‑lead time. | Move to a configuration‑driven approach (properties file, Spring `@Value`, etc.). |

---

## 6. Running / Debugging the File  

*The class itself does not run.* To verify its correctness:

1. **Compile** the module (e.g., `mvn clean compile`).  
2. **Unit‑test** a consumer (e.g., `OrderStatusTest`) that prints the constants:  
   ```java
   System.out.println(APIConstants.ACNT_API_URL);
   System.out.println(APIConstants.ACNT_VALUE);
   ```  
3. **Debug** by setting a breakpoint on any line that accesses a constant in a call‑out class. Inspect the value to ensure it matches expectations.  
4. **Check** that the Base64 token decodes to the expected `username:password` (e.g., `echo "dGNsLXpQdHFzQjM3VVlMU1RRd2Zwd0tyUmdrTVRUZTE4RVo3dllZYU5TUlU6MGNjNTA1ZjQwMGE2MTIwZGRkNGU3YjUyYmUxODUyNDQyZDgxNDliMDZiZThjMmQ1NmIyYTEwNmUxOGI3NmVjNw" | base64 -d`).  

If the token is incorrect, the API calls will return 401 – this is the quickest sanity check.

---

## 7. External Configuration / Environment Variables  

| Reference | Current Usage | Suggested Replacement |
|-----------|---------------|-----------------------|
| `ACNT_VALUE` (Basic auth) | Hard‑coded string. | Load from env var `API_BASIC_AUTH` or secret manager at startup. |
| URLs (`*_API_URL`) | Hard‑coded strings. | Load from `application.properties` (`api.account.url`, etc.) or env vars (`API_ACCOUNT_URL`). |
| `RETRY_COUNT` | Constant `2`. | Load from `API_RETRY_COUNT` env var; default to 2 if missing. |

No other files are referenced directly.

---

## 8. Suggested TODO / Improvements  

1. **Externalise all credentials and URLs** – replace static final strings with values read from a secure configuration source (properties file, Spring `@ConfigurationProperties`, or environment variables).  
2. **Add unit tests** that validate the Base64 token format and that URLs are syntactically valid (e.g., using `java.net.URI`).  

Implementing these changes will reduce operational risk, enable zero‑downtime configuration updates, and improve auditability.