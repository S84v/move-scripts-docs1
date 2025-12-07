**High‑Level Documentation – `CustomProduct.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\CustomProduct.java`

---

### 1. Purpose (One‑Paragraph Summary)
`CustomProduct` is a plain‑old‑Java‑Object (POJO) that models a custom‑product entity exchanged by the **API Access Management** service. It encapsulates the product’s logical name (`productName`) and a collection of associated plan details (`planNameDetails`). The class is used throughout the API layer to serialize/deserialize JSON payloads for inbound requests (e.g., product‑creation or update calls) and outbound responses (e.g., account‑detail queries). It does not contain business logic; its sole responsibility is to provide a type‑safe container for data that downstream services (billing, provisioning, or CRM) consume.

---

### 2. Important Classes / Functions

| Class / Method | Responsibility / Description |
|----------------|------------------------------|
| **`CustomProduct`** (this file) | POJO representing a custom product. Holds a `String productName` and a `List<PlanNameDetails> planNameDetails`. |
| `getProductName()` / `setProductName(String)` | Standard getter and setter for the product name. |
| `getPlanNameDetails()` / `setPlanNameDetails(List<PlanNameDetails>)` | Getter and setter for the list of plan‑detail objects. |
| **`PlanNameDetails`** (referenced type) | Model class (located elsewhere in `com.tcl.api.model`) that describes an individual plan (e.g., plan ID, description, pricing). `CustomProduct` aggregates these. |
| **`ObjectMapper` / JSON libraries** (used by surrounding code) | Not defined here but typical serializers that map `CustomProduct` to/from JSON in REST controllers or service adapters. |

*Note:* No constructors are defined; the default no‑arg constructor is used by Jackson (or similar) for deserialization.

---

### 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | Instances of `CustomProduct` are populated from: <br>• JSON request bodies received by the API (e.g., `/products/custom` POST). <br>• Internal service calls that construct a product from database rows or upstream system responses. |
| **Outputs** | Instances are serialized to JSON for: <br>• API responses (e.g., `GET /accounts/{id}/products`). <br>• Message payloads sent to downstream queues (Kafka, JMS) or external REST endpoints. |
| **Side Effects** | None – the class is immutable from a functional perspective (only holds data). Side effects arise only in the code that creates or consumes it (e.g., DB writes, queue publishes). |
| **Assumptions** | • `PlanNameDetails` is a fully‑defined POJO with proper JSON annotations. <br>• The surrounding framework (Spring Boot, Jersey, etc.) is configured with a JSON mapper that respects Java bean conventions. <br>• `productName` is non‑null for all valid business cases; validation is performed upstream (controller/service layer). |
| **External Services / Resources** | Not directly referenced, but typical callers interact with: <br>• **Database** (product catalog tables). <br>• **Message brokers** (Kafka topics for provisioning). <br>• **External APIs** (partner billing system). |

---

### 4. Connection to Other Scripts / Components

| Component | Relationship |
|-----------|--------------|
| **`AccountDetailsResponse` / `AccountResponseData`** | May contain a `List<CustomProduct>` to represent the products attached to an account. |
| **`PlanNameDetails`** | Direct composition; any change to this class (e.g., adding fields) requires a corresponding version bump of `CustomProduct`. |
| **REST Controllers** (`ProductController.java` or similar) | Use `CustomProduct` as request/response DTOs. |
| **Service Layer** (`ProductService.java`) | Transforms DB entities (`ProductEntity`) into `CustomProduct` objects before returning to the API layer. |
| **Message Producers** (`ProductEventProducer.java`) | Serialize `CustomProduct` into event payloads for downstream provisioning. |
| **Unit / Integration Tests** (`CustomProductTest.java`) | Validate JSON (de)serialization and bean property behavior. |
| **Build / Packaging** (`pom.xml`) | Declares the module `APIAccessManagement` which includes this model package. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema drift** – Adding/removing fields in `PlanNameDetails` without updating `CustomProduct` can cause JSON mapping failures. | Service errors, 500 responses. | Enforce contract tests (e.g., Pact) between producer and consumer services; keep model classes version‑controlled together. |
| **Null `productName`** – If upstream validation fails, downstream systems may receive a null product name, breaking downstream processing. | Data integrity issues, downstream rejections. | Add Bean Validation annotations (`@NotNull`) and enforce validation in controller layer. |
| **Large `planNameDetails` list** – Unbounded list size could cause memory pressure when serializing large payloads. | Out‑of‑memory (OOM) in API gateway or downstream consumer. | Impose a reasonable size limit in the service layer; paginate responses if needed. |
| **Incompatible JSON library versions** – Different micro‑services may use different Jackson versions, leading to subtle (de)serialization differences. | Inconsistent data across services. | Align Jackson version across all modules via the parent `pom.xml`; run integration tests with the exact runtime version. |

---

### 6. Running / Debugging the Class

1. **Unit Test Execution**  
   ```bash
   mvn -Dtest=CustomProductTest test
   ```
   Verify that getters/setters work and that JSON (de)serialization round‑trips correctly.

2. **Local API Debug**  
   - Start the API module (`APIAccessManagement`) with the Spring Boot dev profile:  
     ```bash
     mvn spring-boot:run -Dspring-boot.run.profiles=dev
     ```  
   - Issue a POST request (e.g., via `curl` or Postman) to the product endpoint with a JSON body matching `CustomProduct`.  
   - Set a breakpoint in the controller method that receives the payload; inspect the instantiated `CustomProduct` object.

3. **Serialization Check**  
   ```java
   ObjectMapper mapper = new ObjectMapper();
   String json = mapper.writeValueAsString(customProduct);
   System.out.println(json);
   ```
   Ensure the output matches the expected contract (field names, camelCase).

4. **Logging**  
   Add a debug log in the service layer after mapping DB entities to `CustomProduct`:
   ```java
   logger.debug("Mapped ProductEntity {} to CustomProduct {}", entity.getId(), customProduct);
   ```

---

### 7. External Configuration / Environment Variables

`CustomProduct` itself does not read any config, but surrounding components may rely on:

| Config / Env | Usage |
|--------------|-------|
| `spring.jackson.property-naming-strategy` | Determines whether JSON fields are `camelCase` (default) or `snake_case`. Must be consistent with the field names in this POJO. |
| `product.max-plan-details` (custom property) | May be used by validation logic to cap the size of `planNameDetails`. |
| `api.version` | Often included in response wrappers that embed `CustomProduct`. |

If any of these properties change, verify that the JSON representation of `CustomProduct` remains compatible.

---

### 8. Suggested TODO / Improvements

1. **Add Bean Validation** – Annotate fields with `@NotBlank` (for `productName`) and `@Size(max = X)` (for `planNameDetails`) to enforce constraints early in the request lifecycle.  
2. **Implement `equals()` / `hashCode()`** – Provide value‑based equality to simplify testing and collection handling (e.g., deduplication of products).  

--- 

*End of documentation.*