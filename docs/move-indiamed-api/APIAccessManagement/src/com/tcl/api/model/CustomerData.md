**High‑Level Documentation – `CustomerData.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\CustomerData.java`

---

### 1. Purpose (One‑paragraph summary)
`CustomerData` is a plain‑old‑Java‑Object (POJO) that represents the core customer profile exchanged between the **API Access Management** service and downstream telecom systems (billing, CRM, product catalog). It aggregates the customer reference, primary account identifier, billing entity, currency, and the list of subscribed custom products. The class is used for JSON (de)serialization in REST endpoints and as the internal data carrier for service‑layer business logic that prepares or consumes customer‑centric payloads.

---

### 2. Key Class & Members

| Member | Type | Responsibility |
|--------|------|-----------------|
| `customerRef` | `String` | External reference (e.g., CRM GUID) that uniquely identifies the customer across systems. |
| `accountNumber` | `String` | Primary account number used by billing and provisioning subsystems. |
| `billingEntity` | `String` | Identifier of the legal entity responsible for invoicing (e.g., “TCL‑INDIA”). |
| `currency` | `String` | ISO‑4217 currency code (e.g., “INR”) for all monetary values in the payload. |
| `productNames` | `List<CustomProduct>` | Collection of `CustomProduct` objects describing each product the customer has subscribed to (name, SKU, attributes). |
| Getters / Setters | – | Standard Java bean accessors required by Jackson (or other JSON mappers) and by Spring MVC for request/response binding. |

*No additional methods* – the class is intentionally lightweight to keep serialization deterministic.

---

### 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | Values are populated by: <br>• Deserialization of inbound JSON payloads from external callers (partner APIs, UI front‑ends). <br>• Service‑layer code constructing a response after querying internal repositories (e.g., `CustomerRepository`). |
| **Outputs** | Serialized JSON sent back to callers, or passed downstream to other services (billing, provisioning) via REST, messaging (Kafka/RabbitMQ), or file‑based exchange. |
| **Side Effects** | None – the class holds data only. Side effects arise only from the code that creates or consumes it (e.g., DB reads, API calls). |
| **Assumptions** | • All fields are optional unless validated elsewhere; the class does **not** enforce non‑null constraints. <br>• `CustomProduct` is a fully‑defined model (see `CustomProduct.java`) and is serializable. <br>• The surrounding framework (Spring Boot, Jackson) is configured to use default property naming (camelCase) and to ignore unknown JSON fields. |

---

### 4. Integration Points (How it connects to other scripts/components)

| Connected Component | Relationship |
|---------------------|--------------|
| `CustomProduct.java` | Nested object list; each entry provides product‑level details (name, attributes). |
| Service Layer (`CustomerService`, `AccountService`, etc.) | `CustomerData` instances are returned from service methods (`getCustomerDataByRef(String)`) and accepted as method arguments for update/create operations. |
| REST Controllers (`CustomerController`, `AccountController`) | Used as request bodies (`@RequestBody CustomerData`) and response bodies (`@ResponseBody CustomerData`). |
| Serialization Config (`ObjectMapper` bean) | Relies on default Jackson mapping; any custom modules (e.g., `JavaTimeModule`) affect date handling in nested objects. |
| Messaging/Queue Producers (`KafkaProducer`, `RabbitTemplate`) | May be serialized to JSON before being placed on a topic/queue for downstream processing. |
| External APIs (Billing, CRM) | Payloads built from `CustomerData` are sent via HTTP clients (e.g., `RestTemplate`, `WebClient`). |
| Test Suites (`CustomerDataTest`, integration tests) | Unit tests instantiate the POJO to verify (de)serialization and business‑logic handling. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Null / Missing fields** – downstream systems may expect mandatory values (e.g., `accountNumber`). | Service failures, billing errors. | Add validation annotations (`@NotNull`, `@Size`) and enforce them in controller layer (`@Valid`). |
| **Schema drift** – changes to `CustomProduct` or added fields may break existing consumers. | Runtime `JsonMappingException`. | Version the API (e.g., `v1/customer`) and maintain backward‑compatible POJOs; use `@JsonIgnoreProperties(ignoreUnknown = true)`. |
| **Large product list** – `productNames` could become very large, causing payload bloat. | Increased latency, memory pressure. | Implement pagination or limit the number of products returned; consider streaming JSON (`MappingJacksonValue`). |
| **Incorrect currency codes** – non‑ISO codes may propagate to billing. | Financial reporting errors. | Validate against a whitelist of ISO‑4217 codes (custom validator). |
| **Serialization performance** – default Jackson may be slower under high load. | Throughput degradation. | Pre‑configure a shared `ObjectMapper` with performance‑tuned settings (e.g., `FAIL_ON_EMPTY_BEANS=false`, `WRITE_DATES_AS_TIMESTAMPS=false`). |

---

### 6. Typical Run / Debug Workflow

1. **Local Development**  
   - Run the Spring Boot application with `./mvnw spring-boot:run`.  
   - Use an IDE (IntelliJ/Eclipse) to set a breakpoint on any controller method that receives `CustomerData`.  
   - Send a test request via `curl` or Postman:  
     ```bash
     curl -X POST http://localhost:8080/api/v1/customers \
          -H "Content-Type: application/json" \
          -d '{"customerRef":"CUST123","accountNumber":"ACC456","billingEntity":"TCL-INDIA","currency":"INR","productNames":[{"productName":"Broadband"}]}'
     ```
   - Inspect the populated `CustomerData` object in the debugger.

2. **Unit Testing**  
   - Execute `mvn test` – the generated test class `CustomerDataTest` should cover (de)serialization using Jackson’s `ObjectMapper`.  
   - Verify that all fields round‑trip correctly.

3. **Integration Testing**  
   - Deploy to a test environment (e.g., Docker Compose with dependent services).  
   - Use a contract testing tool (Pact) to ensure the JSON contract matches consumer expectations.

4. **Production Debugging**  
   - Enable TRACE logging for the controller package (`logging.level.com.tcl.api=TRACE`).  
   - Tail logs (`kubectl logs -f <pod>`) to see the inbound JSON payload and the resulting `CustomerData` to‑string (if overridden).  
   - If needed, attach a remote debugger to the JVM (port configured via `JAVA_TOOL_OPTIONS`).

---

### 7. External Configuration / Environment Dependencies

| Config / Env Variable | Usage |
|-----------------------|-------|
| `spring.jackson.property-naming-strategy` (optional) | Determines JSON field naming; default is `LOWER_CAMEL_CASE`. |
| `api.version` (e.g., `v1`) | Part of the REST endpoint path that consumes/produces `CustomerData`. |
| `MAX_PRODUCT_COUNT` (custom property) | May be referenced by validation logic elsewhere to cap `productNames` size. |
| No direct file‑system or SFTP dependencies are present in this POJO. |

---

### 8. Suggested TODO / Improvements

1. **Add Bean Validation** – Annotate fields with `@NotBlank`, `@Pattern`, or custom validators (e.g., `@CurrencyCode`) and enable `@Valid` in controller signatures to enforce data integrity early.  
2. **Implement `equals()`, `hashCode()`, and `toString()`** (or switch to Lombok `@Data`) to simplify debugging, collection handling, and test assertions.  

--- 

*End of documentation.*