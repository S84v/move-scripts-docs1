**High‑Level Documentation – `EventSourceDetails.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\EventSourceDetails.java`

---

### 1. Purpose (One‑paragraph Summary)
`EventSourceDetails` is a simple POJO (Plain Old Java Object) that models the source metadata for an event processed by the API Access Management service. It encapsulates two string fields – `eventTypeName` and `eventSource` – together with standard getters and setters. Instances of this class are used throughout the API layer (e.g., request/response payloads, logging, audit trails) to convey *what* type of event occurred and *where* it originated, enabling downstream orchestration and transformation logic to make routing or business‑rule decisions.

---

### 2. Important Classes / Functions

| Element | Responsibility |
|---------|-----------------|
| **`EventSourceDetails` (class)** | Holds event‑type and source identifiers; provides JavaBean‑style accessors for serialization (Jackson/Gson) and for internal business logic. |
| `getEventTypeName()` | Returns the logical name of the event type (e.g., “ACCOUNT_CREATE”, “PRODUCT_UPDATE”). |
| `setEventTypeName(String)` | Mutates the event‑type name. |
| `getEventSource()` | Returns the origin identifier (e.g., “WEB_PORTAL”, “BATCH_JOB”, “SFTP”). |
| `setEventSource(String)` | Mutates the event‑source value. |

*No other methods or business logic are present.*

---

### 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Detail |
|--------|--------|
| **Inputs** | Values supplied by callers (controllers, services, or deserialization from JSON/XML) when constructing or populating an `EventSourceDetails` instance. |
| **Outputs** | The object itself, typically serialized back to JSON for downstream services, persisted to a database, or logged for audit. |
| **Side Effects** | None – the class is immutable‑by‑convention (no static state, no I/O). |
| **Assumptions** | <ul><li>String values are non‑null and conform to a predefined enumeration in the broader system (e.g., `EventType` enum, `EventSource` enum).</li><li>Serialization framework (Jackson, Gson, etc.) will use the getters/setters automatically.</li><li>Consumers treat the object as a value object; they do not rely on identity semantics.</li></ul> |

---

### 4. Integration Points (How it Connects to Other Scripts/Components)

| Connected Component | Role of `EventSourceDetails` |
|---------------------|------------------------------|
| **Request DTOs** (e.g., `AccountResponseData`, `CustomResponseData`) | Embedded as a field to convey the origin of the request that generated the response. |
| **Service Layer** (business‑logic classes handling events) | Used to route processing based on `eventTypeName` and `eventSource`. |
| **Logging / Auditing** (e.g., `EventLogger`, audit DB tables) | Serialized and stored for traceability. |
| **Message Queues** (Kafka, RabbitMQ) | Serialized into the payload of event messages for downstream micro‑services. |
| **Transformation Scripts** (Move‑type ETL jobs) | Mapped to target schema fields when moving data between systems (e.g., from API DB to billing system). |
| **Configuration / Validation** | May be validated against values defined in external config files (`event-types.yml`, `event-sources.yml`). |

*Because the repository contains many other model classes (Accountdetails, CustomerData, etc.), it is reasonable to assume that `EventSourceDetails` is referenced as a nested object in those DTOs.*

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Invalid / Unexpected Strings** (e.g., typo in `eventTypeName`) | Downstream routing may fail or mis‑process events. | Enforce validation against a whitelist (enum or config file) at the API boundary; reject or default unknown values. |
| **Missing Fields in Serialized Payload** | Consumers may encounter `null` and throw NPEs. | Mark fields as `@NotNull` (Bean Validation) and configure the JSON mapper to fail on missing properties. |
| **Version Skew** (new fields added in future without backward compatibility) | Older services may ignore new data, causing loss of audit info. | Use tolerant deserialization (`IGNORE_UNKNOWN`) and document versioning strategy. |
| **Uncontrolled Logging** (event source may contain sensitive identifiers) | Potential leakage of internal system names. | Scrub or mask sensitive source identifiers before logging; apply log‑level controls. |

---

### 6. Running / Debugging the Class

| Step | Action |
|------|--------|
| **Unit Test** | Write a simple JUnit test that creates an `EventSourceDetails`, sets both fields, and asserts the getters return the same values. |
| **Serialization Test** | Use Jackson’s `ObjectMapper` to serialize an instance to JSON and deserialize back; verify round‑trip integrity. |
| **Integration Check** | Run the API endpoint that returns a response containing `EventSourceDetails` (e.g., `/account/{id}`) and inspect the JSON payload with `curl` or Postman. |
| **Debugging** | Set a breakpoint in the setter methods if the values appear incorrect during request handling; verify the incoming payload matches expected enums. |

Typical Maven/Gradle command (assuming the project uses Maven):

```bash
mvn -Dtest=EventSourceDetailsTest test
```

---

### 7. External Configuration / Environment Dependencies

| Config / Env | Usage |
|--------------|-------|
| **`event-types.yml` / `event-sources.yml`** (hypothetical) | Provides the allowed list of values; referenced by validation utilities elsewhere in the codebase. |
| **Serialization Settings** (`application.yml` → `spring.jackson.*`) | Controls how nulls, unknown properties, and naming strategies are handled for this POJO. |
| **Logging Configuration** (`logback.xml`) | May define patterns that include `eventSource` for audit logs. |

*The class itself does not read any config directly, but surrounding components do.*

---

### 8. Suggested TODO / Improvements

1. **Add Validation Annotations** – Annotate fields with `@NotBlank` and, if enums exist, `@Pattern` or a custom `@EnumValidator` to enforce allowed values at the model level.
2. **Implement `toString()`, `equals()`, and `hashCode()`** – Useful for logging, collection handling, and test assertions; can be generated via Lombok (`@Data`) or manually.

---