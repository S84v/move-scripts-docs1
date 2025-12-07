**High‑Level Documentation – `UsageResponse.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\UsageResponse.java`

---

### 1. Purpose (one‑paragraph summary)
`UsageResponse` is a plain‑old‑Java‑object (POJO) that models the top‑level JSON payload returned by the **Usage** service of the IndiaMed API. It encapsulates the HTTP‑level response metadata (`responseCode`, `message`, `status`) together with the domain payload (`UsageData`). The class is used throughout the APIAccessManagement module for deserialising inbound usage responses, for forwarding them to downstream services, and for serialising outbound responses to callers (e.g., REST controllers, message queues).

---

### 2. Important Classes / Functions

| Element | Responsibility |
|---------|-----------------|
| **`public class UsageResponse`** | Container for the usage service response. |
| `int responseCode` | Numeric status returned by the upstream usage service (e.g., 200, 404). |
| `String message` | Human‑readable description of the response (error text, success note). |
| `String status` | Business‑level status flag (e.g., `"SUCCESS"` / `"FAILURE"`). |
| `UsageData data` | Embedded domain object holding the actual usage details (see `UsageData.java`). |
| `getResponseCode()/setResponseCode(int)` | Accessor & mutator for `responseCode`. |
| `getMessage()/setMessage(String)` | Accessor & mutator for `message`. |
| `getStatus()/setStatus(String)` | Accessor & mutator for `status`. |
| `getData()/setData(UsageData)` | Accessor & mutator for the nested `UsageData` payload. |

*No business logic resides here; the class is deliberately lightweight to enable fast (de)serialization by Jackson/Gson or similar libraries.*

---

### 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | JSON (or equivalent) payload from the external **Usage** API that maps to the fields above. Deserialization is typically performed by a controller/service layer using a JSON mapper. |
| **Outputs** | An instance of `UsageResponse` that may be: <br>• Returned to a REST client (via Spring MVC, JAX‑RS, etc.) <br>• Sent to a message broker (Kafka, RabbitMQ) <br>• Persisted to a DB (as a JSON column or flattened fields). |
| **Side Effects** | None – the class holds data only. Side effects are introduced only by callers that serialize/deserialize or persist the object. |
| **Assumptions** | <br>• `UsageData` is a fully‑populated, non‑null object when `status` is `"SUCCESS"`.<br>• The JSON field names match the Java property names (or appropriate Jackson annotations exist elsewhere).<br>• Consumers treat `responseCode` as an HTTP‑style code; no custom mapping is required inside this class. |

---

### 4. Integration Points (how this file connects to other components)

| Connected Component | Relationship |
|---------------------|--------------|
| **`UsageData.java`** (same package) | Nested payload; `UsageResponse` holds a reference to a fully‑populated `UsageData`. |
| **REST Controllers** (e.g., `UsageController.java`) | Controllers receive `UsageResponse` from service layer and serialize it to HTTP responses. |
| **Service Layer** (e.g., `UsageServiceImpl.java`) | Calls external usage endpoint, deserialises JSON into `UsageResponse`. |
| **External Usage API** (HTTP/HTTPS) | Source of the raw JSON that maps to this model. |
| **Message Queues** (Kafka topics like `usage.response`) | May publish the `UsageResponse` (or its JSON) for downstream processing (billing, analytics). |
| **Persistence Layer** (e.g., JPA repository `UsageResponseEntity`) | If audit logging is required, the object may be stored as a JSON column or mapped to relational fields. |
| **Error‑handling utilities** (e.g., `ApiExceptionMapper`) | May inspect `responseCode`/`status` to translate into standardized error responses. |

*Because the class lives in the `model` package, it is referenced by any code that needs a typed representation of the usage service reply.*

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – upstream usage API adds/renames fields not reflected in this POJO. | Deserialization failures → downstream processing stops. | Implement version‑tolerant deserialization (Jackson `@JsonIgnoreProperties(ignoreUnknown = true)`) and maintain a contract test suite against the upstream schema. |
| **Null `data` payload** when `status` is `"SUCCESS"` – callers may NPE. | Runtime exceptions in downstream services. | Enforce validation in the service layer (`Objects.requireNonNull(response.getData())`) or add defensive getters (`Optional<UsageData> getDataOptional()`). |
| **Incorrect `responseCode` mapping** – callers treat non‑200 codes as success. | Billing/analytics may ingest erroneous data. | Centralise response‑code handling in a utility class; log any non‑2xx codes before further processing. |
| **Uncontrolled object size** – `UsageData` may contain large collections. | Memory pressure on high‑throughput nodes. | Apply streaming deserialization for large payloads; consider pagination at the API level. |

---

### 6. Running / Debugging the Class

1. **Unit Test Example**  
   ```java
   @Test
   public void deserializeUsageResponse() throws Exception {
       String json = Files.readString(Paths.get("src/test/resources/usage-response.json"));
       ObjectMapper mapper = new ObjectMapper();
       UsageResponse resp = mapper.readValue(json, UsageResponse.class);
       
       assertEquals(200, resp.getResponseCode());
       assertEquals("SUCCESS", resp.getStatus());
       assertNotNull(resp.getData());
   }
   ```
   *Purpose:* verifies that the JSON contract matches the POJO.

2. **Debugging in Service Layer**  
   - Set a breakpoint on `UsageServiceImpl.callUsageApi()` right after the HTTP call.  
   - Inspect the `UsageResponse` instance; verify `responseCode`, `status`, and that `data` contains expected fields.  
   - If deserialization fails, check the raw JSON payload and Jackson configuration (e.g., naming strategy, modules).

3. **Running the API**  
   - The class itself is not executable. It is instantiated automatically by the JSON mapper when a controller returns `UsageResponse` or when a service consumes the external API.  
   - To manually test end‑to‑end, invoke the REST endpoint (e.g., `GET /api/v1/usage/{msisdn}`) using `curl` or Postman and observe the JSON output.

---

### 7. External Configuration / Environment Variables

| Config Item | Usage in this file (indirect) |
|-------------|------------------------------|
| **Jackson ObjectMapper settings** (e.g., `FAIL_ON_UNKNOWN_PROPERTIES`) | Determines how tolerant deserialization is when the upstream payload diverges. |
| **API endpoint URL** (`USAGE_API_URL`) | Not used here, but the service that populates `UsageResponse` reads this env variable. |
| **Feature flags** (`usage.response.includeDebugInfo`) | May affect whether additional fields are added to `UsageData`; the POJO must stay compatible. |

*No direct references to config files or env vars exist inside `UsageResponse.java`; all configuration is applied by the surrounding framework.*

---

### 8. Suggested TODO / Improvements

1. **Add Jackson annotations for robustness**  
   ```java
   @JsonIgnoreProperties(ignoreUnknown = true)
   public class UsageResponse { ... }
   ```
   This prevents deserialization failures if the upstream API adds new fields.

2. **Provide a convenience `isSuccess()` method**  
   ```java
   public boolean isSuccess() {
       return "SUCCESS".equalsIgnoreCase(status) && responseCode == 200;
   }
   ```
   Centralises success‑check logic and reduces boilerplate in callers.

--- 

*End of documentation.*