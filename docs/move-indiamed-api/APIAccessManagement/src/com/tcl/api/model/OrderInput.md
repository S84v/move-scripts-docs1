**High‑Level Documentation – `OrderInput.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\OrderInput.java`

---

### 1. Purpose (one‑paragraph summary)  
`OrderInput` is a POJO that models the inbound payload for the “Create Order” API exposed by the **API Access Management** service. It groups together a logical *inputGroupId* (used for batch tracking), the originating *sourceSystem* identifier, and a collection of `OrdersInformation` objects that contain the detailed order data. The class is serialized/deserialized to/from JSON (or XML) by the REST controller layer and passed downstream to the order‑processing orchestration scripts that move the data into downstream fulfillment, billing, and CRM systems.

---

### 2. Important Classes / Functions  

| Element | Responsibility |
|---------|-----------------|
| **`OrderInput` (class)** | Container for the top‑level request body of the “Create Order” endpoint. |
| `String inputGroupId` | Identifier that groups multiple orders for a single batch run (used for audit, idempotency, and correlation). |
| `String sourceSystem` | Logical name / code of the external system that originated the request (e.g., “CRM”, “WebPortal”). |
| `List<OrdersInformation> ordersInformation` | Collection of per‑order detail objects (defined elsewhere). |
| `getInputGroupId()` / `setInputGroupId(String)` | Standard JavaBean accessor for `inputGroupId`. |
| `getSourceSystem()` / `setSourceSystem(String)` | Standard JavaBean accessor for `sourceSystem`. |
| `getOrdersInformation()` / `setOrdersInformation(List<OrdersInformation>)` | Standard JavaBean accessor for the order list. |

*No business logic resides in this class; it is a pure data holder.*

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Aspect | Detail |
|--------|--------|
| **Input source** | HTTP request body (JSON) received by the REST controller `OrderController#createOrder`. |
| **Deserialization** | Jackson (or similar) maps JSON fields `inputGroupId`, `sourceSystem`, `ordersInformation` to this POJO. |
| **Output / downstream usage** | Passed as a method argument to service layer `OrderService.process(OrderInput)` which then drives the move‑scripts (e.g., `order-move.sh`, Java batch jobs). |
| **Side effects** | None directly; side effects occur later in the processing pipeline (DB writes, file generation, message queue publishing). |
| **Assumptions** | <ul><li>`OrdersInformation` class is correctly defined and serializable.</li><li>All required fields are present; validation is performed upstream (e.g., using Bean Validation annotations or controller‑level checks).</li><li>`inputGroupId` is unique per batch; duplicate detection is handled elsewhere.</li></ul> |

---

### 4. Integration Points (how it connects to other scripts/components)  

| Component | Connection Detail |
|-----------|-------------------|
| **REST Controller** (`com.tcl.api.controller.OrderController`) | Receives HTTP POST `/orders`, deserializes JSON → `OrderInput`. |
| **Service Layer** (`com.tcl.api.service.OrderService`) | `process(OrderInput)` consumes the POJO, extracts `OrdersInformation` and forwards each to the **Move Engine**. |
| **Move Engine** (shell/Java batch scripts under `move-indiamed-api\scripts\order`) | Receives a Java object or a generated JSON file; scripts read the list, transform data, and invoke downstream adapters (SFTP, MQ, DB). |
| **Persistence** (`OrderRepository`) | May persist the `OrderInput` metadata (group id, source) for audit and retry. |
| **External Systems** | <ul><li>**SFTP** – for bulk file hand‑off to partner OSS.</li><li>**Message Queue** (e.g., Kafka, IBM MQ) – for real‑time order events.</li><li>**Billing/CRM APIs** – invoked later in the pipeline.</li></ul> |
| **Logging / Monitoring** | `OrderInput` fields are logged (masked as needed) for traceability; metrics are emitted with `inputGroupId` as a tag. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing/invalid fields** (e.g., null `inputGroupId`) | Order rejected downstream, possible data loss. | Enforce Bean Validation (`@NotNull`) in the controller; return clear 400 responses. |
| **Large payloads** (many `OrdersInformation` items) | Memory pressure on the API JVM, slower processing. | Set a maximum batch size; stream‑process large lists (e.g., Jackson streaming) or chunk into multiple `OrderInput` objects. |
| **Incorrect `sourceSystem` values** | Mis‑routing of orders, audit confusion. | Maintain a whitelist of allowed source codes; reject unknown values early. |
| **Serialization incompatibility** (future schema changes) | Runtime `JsonMappingException`. | Version the API (`/v1/orders`) and keep backward‑compatible POJOs; add unit tests for serialization. |
| **Untracked duplicate `inputGroupId`** | Duplicate order creation. | Implement idempotency check in `OrderService` using a DB unique constraint or Redis cache. |

---

### 6. Running / Debugging the Component  

1. **Local Development**  
   ```bash
   # Start the Spring Boot API (assumes Maven)
   mvn spring-boot:run -Dspring-boot.run.profiles=local
   ```
2. **Invoke the endpoint** (curl example):
   ```bash
   curl -X POST http://localhost:8080/api/v1/orders \
        -H "Content-Type: application/json" \
        -d '{
              "inputGroupId":"batch-20251204-001",
              "sourceSystem":"WebPortal",
              "ordersInformation":[{...}]
            }'
   ```
3. **Debugging**  
   - Set a breakpoint in `OrderController#createOrder` or `OrderService#process`.  
   - Verify that the deserialized `OrderInput` contains the expected list size.  
   - Check logs for the `inputGroupId` tag to trace the order through downstream scripts.  
   - If the move‑engine fails, inspect the generated JSON file (usually under `/tmp/order-batch-*.json`) that the engine consumes.  

---

### 7. External Configuration / Environment Variables  

| Config / Env | Usage |
|--------------|-------|
| `order.input.maxSize` (application.yml) | Maximum number of `OrdersInformation` items allowed per request. |
| `order.source.allowed` (comma‑separated list) | Whitelist of valid `sourceSystem` identifiers. |
| `move.engine.path` | Filesystem path where the Java service writes the serialized order batch for the move scripts. |
| `logging.level.com.tcl.api` | Controls verbosity of POJO logging (useful for debugging). |

If any of these keys are missing, defaults are applied (e.g., no size limit, all sources accepted). Verify the `application-{profile}.yml` files used in the active Spring profile.

---

### 8. Suggested TODO / Improvements  

1. **Add Bean Validation Annotations** – annotate fields with `@NotBlank`, `@Size(max=…)`, and create a custom `@ValidSourceSystem` validator to enforce the whitelist at the POJO level.  
2. **Implement a Builder Pattern** – provide a fluent `OrderInputBuilder` to simplify test creation and future immutability (make fields `final` and remove setters).  

--- 

*End of documentation.*