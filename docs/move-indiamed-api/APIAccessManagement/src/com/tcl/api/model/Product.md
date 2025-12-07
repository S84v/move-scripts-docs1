**High‑Level Documentation – `Product.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\Product.java`

---

### 1. Summary
`Product` is a core domain model representing a telecom product line item within the IndiaMED API. It aggregates product‑level attributes (quantity, pricing, lifecycle dates), technical details (event source, proposition), and a rich set of address objects (site, CPE delivery, contracting, GSTIN). The class is a plain‑old‑Java‑Object (POJO) used for JSON (de)serialization in request/response payloads for order‑creation, order‑status, and post‑order processing flows.

---

### 2. Important Classes / Functions

| Element | Responsibility |
|---------|-----------------|
| **Product** (this class) | Holds all product‑specific data transferred between the API layer and downstream provisioning/billing systems. |
| **getters / setters** (e.g., `getProductName()`, `setProductQuantity(int)`) | Provide JavaBean access for serialization frameworks (Jackson, Gson) and for service‑layer code that builds or reads product objects. |
| **EventSourceDetails** (field `eventSourceDetails`) | Captures source‑system event metadata; used by downstream event‑driven pipelines. |
| **PropositionDetails** (field `propositionDetails`) | Holds proposition‑level configuration (e.g., plan, bundle) required for pricing and eligibility checks. |
| **Address** (multiple fields) | Represents physical locations (sites, CPE delivery, contracting) needed for logistics, tax, and regulatory processing. |
| **ProductAttr** (`productAttributes`) | Encapsulates extensible key‑value attribute set for product‑specific flags (e.g., “SIM aggregation”, “tax exempt”). |

*Note:* No business logic resides in this class; it is a data carrier.

---

### 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | Populated from inbound API JSON (e.g., `OrderInput` → `Product` list) or from downstream services (billing, provisioning) that enrich the object. |
| **Outputs** | Serialized back to JSON in API responses (`OrderResponse`, `GetOrderResponse`, `PostOrderDetail`). May also be sent to message queues (Kafka, JMS) or persisted to a relational/NoSQL store. |
| **Side Effects** | None – the class itself does not perform I/O. Side effects arise only when the object is passed to other components (e.g., DB DAO, external REST client). |
| **Assumptions** | <ul><li>All referenced model classes (`EventSourceDetails`, `PropositionDetails`, `Address`, `ProductAttr`) are present and correctly annotated for (de)serialization.</li><li>Numeric fields (`int`) default to `0` if not supplied; callers must treat `0` as “not set” where appropriate.</li><li>String fields may be `null`; downstream services must handle nullability.</li></ul> |

---

### 4. Integration Points

| Connected Component | How `Product` is Used |
|---------------------|-----------------------|
| **OrderInput** | `OrderInput` contains a `List<Product>` that represents the items being ordered. |
| **OrderResponse / GetOrderResponse** | Returns the same `Product` objects (often enriched with pricing, status, and address verification). |
| **PostOrderDetail** | After order acceptance, `Product` data is embedded in post‑order payloads sent to provisioning systems. |
| **Billing/Provisioning Services** | `Product` objects are marshalled to JSON and posted to internal REST endpoints or placed on a message bus for asynchronous processing. |
| **Database Mappers** (e.g., MyBatis, JPA) | May map `Product` fields to `PRODUCT` table columns for persistence of order details. |
| **External Config** | Serialization behavior (field naming, inclusion rules) is driven by `application.yml`/`application.properties` (Jackson `ObjectMapper` settings) and possibly by environment variables that toggle “strict null checking”. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing / Null Critical Fields** (e.g., `productName`, `productQuantity`) | Downstream provisioning may reject the order, causing order‑level failures. | Add validation layer (Bean Validation `@NotNull`, custom validator) before the object is sent downstream. |
| **Inconsistent Pricing Fields** (`ovrdnInitPrice`, `OvrdnPeriodicPrice`) stored as `int` may overflow or lose precision. | Incorrect billing, revenue leakage. | Switch to `long` or `BigDecimal` and enforce currency scaling; add unit tests for edge values. |
| **Serialization Mismatch** (field name changes break JSON contracts). | API clients receive 400/500 errors. | Keep field names stable; use Jackson `@JsonProperty` to decouple Java name from JSON contract. |
| **Address Object Nullability** (multiple address fields may be null). | NullPointerExceptions in downstream services that assume non‑null addresses. | Initialize address fields to empty `Address` objects or add null‑checks in service code. |
| **Uncontrolled Attribute Growth** (`ProductAttr` may become a catch‑all map). | Hard to maintain schema, potential security exposure. | Define a whitelist of allowed attribute keys; consider versioned attribute objects. |

---

### 6. Example Run / Debug Workflow

1. **Local Development**  
   ```java
   Product p = new Product();
   p.setProductseq(1);
   p.setProductName("4G Data Plan");
   p.setProductQuantity(2);
   p.setTerminationDate("2025-12-31");
   // Populate nested objects
   p.setEventSourceDetails(new EventSourceDetails(...));
   p.setPropositionDetails(new PropositionDetails(...));
   p.setSiteAAddress(new Address(...));
   // … set other fields as needed
   ```
2. **Serialization (Jackson)**  
   ```java
   ObjectMapper mapper = new ObjectMapper();
   String json = mapper.writeValueAsString(p);
   System.out.println(json);
   ```
3. **Debugging**  
   - Set breakpoints on getters/setters to verify values after deserialization.  
   - Use `mapper.readValue(json, Product.class)` to test round‑trip integrity.  
   - Log the object with `mapper.writerWithDefaultPrettyPrinter().writeValueAsString(p)` for quick inspection.  

4. **Running in Production**  
   - The `Product` instances are created by the `OrderService` when processing an incoming REST request.  
   - They are passed to `OrderProcessor`, which validates, enriches (pricing, tax), and forwards to the provisioning queue.  
   - Monitoring: ensure that the JSON payload size stays within limits (large address lists can inflate payload).  

---

### 7. External Configuration / Environment Variables

| Config Item | Usage |
|-------------|-------|
| `spring.jackson.property-naming-strategy` (e.g., `SNAKE_CASE`) | Determines JSON field naming for `Product` fields. |
| `product.price.scale` (custom env var) | May be read by a pricing utility that calculates `ovrdnInitPrice` and `OvrdnPeriodicPrice`. |
| `address.validation.enabled` | Toggles runtime validation of nested `Address` objects before sending downstream. |
| `feature.toggle.productAttrStrict` | Controls whether unknown keys in `ProductAttr` are rejected. |

If any of these variables are missing, defaults are applied (e.g., no validation, standard Jackson naming).

---

### 8. Suggested TODO / Improvements

1. **Add Bean Validation Annotations** – Annotate critical fields (`@NotNull`, `@Size`, `@Positive`) and integrate a validation step in the request‑handling pipeline to catch malformed `Product` objects early.  
2. **Replace Primitive `int` Pricing Fields with `BigDecimal`** – Improves monetary precision and aligns with industry best practices for financial data handling.  

--- 

*End of documentation for `Product.java`.*