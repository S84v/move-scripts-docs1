**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\constants\APIConstants.java`

---

## 1. High‑Level Summary
`APIConstants` is a pure‑Java constants holder used throughout the *APIAccessManagementMigration* move‑script package. It centralises static configuration values required by the various call‑out classes (e.g., `BulkInsert2`, `PostOrder`, `ProductDetails`, `UsageProdDetails`, etc.) to interact with the Geneva Order Management REST API, define HTTP headers, control retry behaviour, and provide static business‑logic strings (plan types, charge descriptions, etc.). No logic is executed; the class simply exposes `public static final` fields.

---

## 2. Important Class & Members

| Member | Type | Description |
|--------|------|-------------|
| `ACNT_API_URL` | `String` | Base URL for the *accountNumber* endpoint (UAT environment). |
| `PLAN_API_URL` | `String` | Base URL for the *customPlans* endpoint (UAT environment). |
| `CONTENT_TYPE` | `String` | HTTP header name “Content-Type”. |
| `CONTENT_VALUE` | `String` | Header value “application/json”. |
| `ACNT_AUTH` | `String` | HTTP header name “Authorization”. |
| `ACNT_VALUE` | `String` | Base‑64 encoded **Basic** auth token (hard‑coded). |
| `ACNT_ENTITY` | `String` | Entity identifier used in request payloads (`TCCSPL`). |
| `ACNT_SERVICE` | `String` | Service description (`MOVE IOT Connect`). |
| `ADDON_CHARGE` | `String` | Business label “Add on Plan Charge”. |
| `TEXT_CHARGE` | `String` | Business label “Test Plan Charge”. |
| `COMMERCIAL` / `TEST` | `String` | Plan type identifiers. |
| `RETRY_COUNT` | `int` | Number of HTTP retry attempts (default = 3). |
| `SUSPEND` | `String` | Status flag “Suspend”. |
| `SAFE_CHARGE` | `String` | Charge description “Safe Custody charges”. |

*No methods are defined; the class is a static container.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | None – all values are hard‑coded at compile time. |
| **Outputs** | None – the class only provides constants. |
| **Side‑effects** | None (no I/O, no state mutation). |
| **Assumptions** | • The UAT API endpoints are reachable from the runtime host.<br>• The hard‑coded Basic auth token is valid for the target API.<br>• Consumers will not modify the constants at runtime (they are `final`). |
| **External Services Referenced** | - Geneva Order Management API (accountNumber & customPlans).<br>- Potential downstream systems that interpret the charge/status strings. |

---

## 4. Integration Points with Other Scripts / Components

| Consuming Component | How It Uses `APIConstants` |
|---------------------|----------------------------|
| `BulkInsert2.java`, `PostOrder.java`, `ProductDetails.java`, `UsageProdDetails*.java` | Retrieve endpoint URLs (`ACNT_API_URL`, `PLAN_API_URL`), HTTP header names/values (`CONTENT_TYPE`, `CONTENT_VALUE`, `ACNT_AUTH`, `ACNT_VALUE`), and business strings (e.g., `ADDON_CHARGE`, `COMMERCIAL`). |
| `OrderStatus*.java` | May use `SUSPEND` or plan type constants to build status payloads. |
| `SQLServerConnection.java` | Does **not** reference `APIConstants` directly, but runs in the same JVM; any change to constants may affect overall transaction flow. |
| Build / CI pipeline | Compiles this class together with the rest of the `src` tree; no external configuration files are required for compilation. |

*Because the constants are static, any change requires a recompilation of all dependent classes.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hard‑coded Basic auth token** (`ACNT_VALUE`) | Credential leakage, inability to rotate secrets without code change. | Move the token to a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and load it at runtime via environment variable or config file. |
| **Static URLs point to UAT** | Accidental execution against non‑production environment in production runs. | Externalise URLs to environment‑specific property files; enforce environment validation before invoking API calls. |
| **No configurability for retry count** (`RETRY_COUNT`) | Inflexible handling of transient network issues. | Make retry count configurable (property or env var) and expose via a utility class. |
| **String literals for business logic** (`ADDON_CHARGE`, `SAFE_CHARGE`, etc.) | Typos or mismatches cause downstream validation failures. | Centralise such values in a domain‑model enum or external dictionary that can be version‑controlled and validated. |
| **Compilation dependency** | Any change forces a full rebuild, potentially delaying hot‑fixes. | Separate configuration from code (e.g., `application.properties`) to allow hot‑swap without recompilation. |

---

## 6. Running / Debugging the Constants

*The class itself does not run.* To verify that the constants are correctly loaded:

1. **Unit‑test snippet** (e.g., JUnit):
   ```java
   @Test
   public void constantsShouldBeNonNull() {
       assertNotNull(APIConstants.ACNT_API_URL);
       assertEquals("application/json", APIConstants.CONTENT_VALUE);
   }
   ```
2. **From a consuming call‑out class** (e.g., `PostOrder`), add a debug log:
   ```java
   logger.info("Calling {} with auth {}", APIConstants.ACNT_API_URL, APIConstants.ACNT_VALUE);
   ```
3. **IDE inspection** – open `APIConstants` and use “Find Usages” to see all dependent classes.

When debugging a failed API call, check the logged URL and Authorization header; they originate from this constants file.

---

## 7. External Configuration / Environment Variables

- **Current state:** All values are hard‑coded; no external config is referenced.
- **Potential external sources:**  
  - `API_URL` could be overridden by `API_ACCOUNT_URL` / `API_PLAN_URL` environment variables.  
  - `ACNT_VALUE` could be read from `API_AUTH_TOKEN`.  
  - `RETRY_COUNT` could be read from `API_RETRY_COUNT`.  

If such variables are introduced, the consuming code must be updated to read them (e.g., via `System.getenv()` or a configuration framework like Spring `@Value`).

---

## 8. Suggested TODO / Improvements

1. **Externalise Sensitive & Environment‑Specific Values**  
   - Move `ACNT_VALUE`, `ACNT_API_URL`, and `PLAN_API_URL` to a secure properties file or secret manager. Provide a fallback to environment variables.

2. **Replace String Constants with Typed Enums**  
   - Create enums for `PlanType { COMMERCIAL, TEST }` and `ChargeType { ADDON, TEXT, SAFE }` to gain compile‑time safety and avoid magic strings in payload construction.

---