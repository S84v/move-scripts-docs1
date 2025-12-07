**High‑Level Documentation – `UsageData.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\UsageData.java`

---

### 1. Summary
`UsageData` is a plain‑old‑Java‑Object (POJO) that represents a customer’s usage payload exchanged between the API Access Management service and downstream consumer systems (e.g., billing, analytics, or partner portals). It carries the customer reference, the associated account number, and a list of detailed usage records (`UsageDetail`). The class is used for JSON (de)serialization in REST controllers and for mapping data retrieved from the usage‑data store.

---

### 2. Core Class & Responsibilities
| Element | Responsibility |
|---------|-----------------|
| **`public class UsageData`** | Container for usage‑related data transferred via the API. |
| **Fields** | |
| `private String customerReference;` | Holds the external identifier for the customer (e.g., MSISDN, CRM ID). |
| `private String accountNumber;` | Holds the internal account number used by downstream systems. |
| `private List<UsageDetail> usageDatailsArray;` | Collection of individual usage entries (calls, SMS, data, etc.). |
| **Getter / Setter methods** | Standard JavaBean accessors used by Jackson (or other JSON libraries) and by service‑layer code for data manipulation. |

*Note:* `UsageDetail` is another model class (not shown) that encapsulates a single usage record (timestamp, type, quantity, etc.).

---

### 3. Inputs, Outputs, Side Effects & Assumptions
| Category | Description |
|----------|-------------|
| **Inputs** | Instances are populated either by: <br>• Deserialization of inbound JSON payloads from external callers (e.g., partner API). <br>• Service‑layer code that builds a usage response from a database or file source. |
| **Outputs** | Serialized JSON sent back to callers or persisted to downstream stores (e.g., Kafka, DB). The class itself does not produce side effects. |
| **Side Effects** | None – pure data holder. |
| **Assumptions** | <br>• `UsageDetail` class is present on the classpath and correctly annotated for JSON (if needed). <br>• The field name `usageDatailsArray` contains a typo; downstream code must reference the exact name or rely on Jackson’s property naming strategy. <br>• No validation is performed inside this POJO; callers are responsible for ensuring non‑null, well‑formed values. |
| **External Dependencies** | Implicitly used by: <br>• Spring MVC / JAX‑RS controllers that map request/response bodies. <br>• Jackson (or similar) for JSON (de)serialization. <br>• Service classes that retrieve `UsageDetail` records from a repository (DB, file, or message queue). |

---

### 4. Interaction with Other Scripts / Components
| Component | Connection Point |
|-----------|------------------|
| **Controller Layer** (`UsageController.java` or similar) | Accepts `UsageData` as a request body (`@RequestBody`) or returns it as a response (`@ResponseBody`). |
| **Service Layer** (`UsageService.java`) | Constructs `UsageData` objects after aggregating `UsageDetail` rows from the usage repository. |
| **Repository / DAO** (`UsageRepository.java`) | Supplies the list of `UsageDetail` objects that populate `usageDatailsArray`. |
| **Message Queues** (e.g., Kafka producer) | Serializes `UsageData` to JSON and publishes to a topic for downstream processing. |
| **External API Clients** (`PartnerApiClient.java`) | Deserializes inbound JSON into `UsageData` for validation or enrichment. |
| **Unit / Integration Tests** (`UsageDataTest.java`) | Instantiates the POJO to verify JSON mapping and field access. |

*Because the project follows a conventional layered architecture, any change to the field names (e.g., fixing the typo) must be propagated to the controller’s DTO mapping, the service’s builder code, and any JSON schema used for contract testing.*

---

### 5. Operational Risks & Recommended Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| **Typo in field name (`usageDatailsArray`)** | JSON contracts may break; downstream parsers expecting `usageDetailsArray` will fail. | Add a `@JsonProperty("usageDetailsArray")` annotation to map the correct JSON name, and schedule a refactor to rename the field consistently. |
| **Null or empty collections** | Downstream processors may throw `NullPointerException` when iterating. | Enforce non‑null list in the service layer (`Collections.emptyList()` fallback) or add validation annotations (`@NotNull`). |
| **Version drift between POJO and external schema** | Integration partners receive malformed payloads. | Maintain a versioned OpenAPI/Swagger definition and generate POJOs from it (or vice‑versa) to keep them in sync. |
| **Unvalidated input data** | Malicious or malformed data could propagate to downstream systems. | Apply Bean Validation (`@Valid`, `@Size`, `@Pattern`) on fields where appropriate, and reject invalid payloads early in the controller. |
| **Serialization performance** | Large `usageDatailsArray` may cause high memory usage during JSON conversion. | Stream usage details when possible (e.g., using Jackson’s `JsonGenerator` or pagination). |

---

### 6. Running / Debugging the Class
1. **Unit Test Execution**  
   ```bash
   mvn test -Dtest=UsageDataTest
   ```  
   Verify that getters/setters work and that Jackson correctly maps JSON to/from the class.

2. **Local API Invocation** (assuming a Spring Boot app)  
   ```bash
   curl -X POST http://localhost:8080/api/usage \
        -H "Content-Type: application/json" \
        -d '{"customerReference":"CUST123","accountNumber":"ACC456","usageDetailsArray":[{...}]}'
   ```  
   - Set a breakpoint in `UsageController` or `UsageService` where the `UsageData` object is received.  
   - Inspect the populated fields; confirm that `usageDatailsArray` is populated despite the typo.

3. **Debugging Serialization Issues**  
   - Enable Jackson debug logging (`logging.level.com.fasterxml.jackson=DEBUG`).  
   - Check the generated JSON field names; if they differ from the contract, add `@JsonProperty` annotations.

---

### 7. External Config / Environment Variables
`UsageData` itself does not read any configuration. However, the surrounding components may rely on:  
- **`spring.jackson.property-naming-strategy`** – determines how field names are converted to JSON (e.g., `SNAKE_CASE`).  
- **Feature toggles** (e.g., `usage.detail.enabled`) that control whether `usageDatailsArray` is populated.

---

### 8. Suggested Improvements (TODO)
1. **Fix the field typo and add explicit JSON mapping**  
   ```java
   @JsonProperty("usageDetailsArray")
   private List<UsageDetail> usageDetailsArray;
   ```
   This preserves backward compatibility while aligning the Java name with the contract.

2. **Introduce Bean Validation**  
   ```java
   @NotBlank
   private String customerReference;
   @NotBlank
   private String accountNumber;
   @NotNull
   private List<@Valid UsageDetail> usageDetailsArray;
   ```
   This will catch malformed payloads early in the request lifecycle.

---