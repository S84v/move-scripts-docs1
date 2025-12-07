**High‑Level Documentation – `OrderStatusList.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\OrderStatusList.java`

---

### 1. Purpose (one‑paragraph summary)
`OrderStatusList` is a simple POJO (Plain Old Java Object) that represents the processing result of a single input row in the bulk‑order API flow. It carries the row identifier (`inputRowId`), the final status code (`status`), and a human‑readable status message (`statusMessage`). Instances are populated by the order‑validation/orchestration service and later consumed by response‑building components (e.g., `OrderResponse`, `GetOrderResponse`) to report per‑row outcomes back to callers or downstream systems.

---

### 2. Important Classes / Functions

| Class / Method | Responsibility |
|----------------|----------------|
| **`OrderStatusList`** (this file) | Holds three fields with standard getters/setters: <br>• `inputRowId` – unique identifier of the source row (often a DB key or CSV line number). <br>• `status` – short code (`SUCCESS`, `FAILURE`, `REJECTED`, etc.). <br>• `statusMessage` – descriptive text for logs or API consumers. |
| `getInputRowId()` / `setInputRowId(String)` | Accessor for the row identifier. |
| `getStatus()` / `setStatus(String)` | Accessor for the status code. |
| `getStatusMessage()` / `setStatusMessage(String)` | Accessor for the free‑form message. |

*Note:* The class is used as a collection element (e.g., `List<OrderStatusList>`) in higher‑level response models such as `OrderResponse` and `GetOrderResponse`.

---

### 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | Values are set by upstream processing code (validation, transformation, or external system calls). No direct I/O in this class. |
| **Outputs** | The populated object is serialized (typically to JSON via Jackson/Gson) and returned in API responses or written to audit logs. |
| **Side Effects** | None – pure data holder. |
| **Assumptions** | • `inputRowId` is unique within the current batch. <br>• `status` conforms to a predefined enumeration used across the API (checked elsewhere). <br>• Consumers expect non‑null strings; callers should enforce defaults. |

---

### 4. Integration Points (how it connects to other scripts/components)

| Connected Component | Connection Detail |
|---------------------|-------------------|
| `OrderResponse.java` | Contains a `List<OrderStatusList>` field named `orderStatusList`. After bulk processing, the service populates this list and returns it to the client. |
| `GetOrderResponse.java` | May embed a similar list when querying the status of previously submitted orders. |
| Validation / Orchestration services (e.g., `OrderProcessor`, `BulkOrderHandler`) | Create and update `OrderStatusList` objects per input row. |
| JSON serialization layer (Jackson, Gson) | Serializes the POJO for HTTP response bodies. |
| Logging / Auditing utilities | Log `statusMessage` for troubleshooting; sometimes the whole object is logged as JSON. |
| Database / Persistence (if any) | Some implementations persist the status list to an audit table; this class is mapped to a DTO for that purpose. |

*Because the repository follows a “model‑only” pattern, the class itself does not reference external services, but any code that instantiates it will typically read configuration (e.g., status‑code mappings) from `application.properties` or environment variables.*

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Incorrect status code** – callers may set an unsupported value, causing downstream validation failures. | API consumer receives ambiguous error. | Centralise status codes in an enum (`OrderStatus`) and enforce via setter validation or builder pattern. |
| **Missing `inputRowId`** – leads to duplicate or untraceable audit entries. | Difficulty in troubleshooting. | Validate non‑null/empty `inputRowId` at creation time; add unit tests. |
| **Large batch size** – building a huge `List<OrderStatusList>` may cause memory pressure. | Out‑of‑memory (OOM) in high‑throughput environments. | Stream processing; write intermediate status rows to a temporary store (DB or file) instead of keeping the whole list in memory. |
| **Serialization mismatch** – field names changed without updating API contracts. | Breaking change for external integrators. | Keep field names stable; use `@JsonProperty` annotations if renaming is required. |

---

### 6. Running / Debugging Guide (operator / developer)

1. **Typical Invocation**  
   - The class is never executed directly. It is instantiated by the bulk‑order service (e.g., `BulkOrderHandler.processBatch(List<OrderInput>)`).  
   - To observe its content, enable DEBUG logging for the package `com.tcl.api.model` or the service that populates it.

2. **Local Test**  
   ```java
   public static void main(String[] args) {
       OrderStatusList osl = new OrderStatusList();
       osl.setInputRowId("row-123");
       osl.setStatus("SUCCESS");
       osl.setStatusMessage("Processed successfully");
       System.out.println(new com.fasterxml.jackson.databind.ObjectMapper()
               .writeValueAsString(osl));
   }
   ```
   - Run the snippet to verify JSON output matches the API contract.

3. **Debugging Steps**  
   - Set a breakpoint on any `set*` method.  
   - Inspect the object after the processing loop completes; ensure each row has a unique `inputRowId` and an appropriate `status`.  
   - If the downstream API returns an unexpected error, check the serialized JSON payload for missing fields.

4. **Integration Test**  
   - Use the existing test suite (e.g., `OrderResponseTest`) that builds a list of `OrderStatusList` objects and asserts the response JSON schema.

---

### 7. External Configuration / Environment Variables

- **None** are referenced directly in this file.  
- Indirect dependencies: status‑code values may be defined in `application.properties` (e.g., `order.status.success=SUCCESS`). The code that sets `status` reads those properties.

---

### 8. Suggested Improvements (TODO)

1. **Replace mutable POJO with immutable value object**  
   - Use a constructor (or builder) that requires all three fields, remove setters, and make fields `final`. This prevents accidental modification after creation and improves thread‑safety.

2. **Add validation logic**  
   - Implement a small validation method (or use Bean Validation annotations) to enforce non‑null/empty `inputRowId` and restrict `status` to a known enum. Throw a custom `InvalidOrderStatusException` if validation fails.

---