**ProductData.java – High‑Level Documentation**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\ProductData.java`

---

### 1. One‑Paragraph Summary
`ProductData` is a plain‑old‑Java‑object (POJO) that represents the core payload for a product when interacting with the **API Access Management** service. It carries the accounting identifier, a list of custom product attributes, the service‑type classification, and the owning TCL entity. The class is used throughout the move‑scripts to build, enrich, and serialize product information for downstream provisioning, billing, and reporting APIs.

---

### 2. Important Classes & Responsibilities  

| Class / Method | Responsibility | Notes |
|----------------|----------------|-------|
| **`ProductData`** | Container for product‑level data exchanged with external APIs. Holds: <br>• `accountingCode` – billing/accounting identifier.<br>• `productAttributes` – list of `CustomProductAttribute` objects (key/value pairs specific to the product).<br>• `serviceType` – list of service‑type strings (e.g., “MOBILE”, “FIXED”).<br>• `tclEntity` – logical entity/region owning the product. | Simple getters/setters only – no business logic. |
| **`CustomProductAttribute`** (referenced) | Represents a single custom attribute (name/value) attached to a product. Defined elsewhere in the same package. | Expected to be serializable to JSON. |
| **`ProductAttributeResponse`**, **`ProductAttribute`**, **`ProductAttr`**, **`Product`**, **`PlanName…`**, **`OrdersInformation`**, etc. (from history) | Companion model classes that together form the full order/request envelope. `ProductData` is embedded in these higher‑level objects. | Provides context for how `ProductData` is consumed. |

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Aspect | Detail |
|--------|--------|
| **Inputs** | Values are set by service‑layer code that reads from: <br>• Product catalog DB (e.g., `accounting_code` column).<br>• Attribute enrichment service (populates `productAttributes`).<br>• Business rules that decide `serviceType` list.<br>• Configuration mapping TCL entity codes. |
| **Outputs** | When the containing object is serialized (Jackson/Gson) it becomes part of the JSON payload sent to external APIs (order provisioning, billing, etc.). |
| **Side Effects** | None – the class is a data holder only. |
| **Assumptions** | • Callers will initialise collections before use (no internal lazy init).<br>• `CustomProductAttribute` objects are well‑formed (non‑null name/value).<br>• The downstream API expects the exact field names as defined (camelCase). |

---

### 4. Connections to Other Scripts & Components  

| Connected Component | How `ProductData` is Used |
|---------------------|---------------------------|
| **Controller / REST endpoint** (`ProductController` or similar) | Receives a request, builds a `ProductData` instance, and returns it as part of the response body. |
| **Service Layer** (`ProductService`, `OrderService`) | Retrieves DB rows, maps them to `ProductData` via a mapper (e.g., MapStruct or manual conversion). |
| **Mapper / DTO Builder** (`ProductMapper`) | Converts domain entities (`ProductEntity`) → `ProductData`. |
| **Serialization Utility** (`ObjectMapper` from Jackson) | Serializes `ProductData` (nested inside `OrdersInformation` etc.) to JSON for HTTP POST to external provisioning system. |
| **Batch/Move Scripts** (`move‑indiamed‑api/.../scripts/*.groovy` or Java jobs) | May instantiate `ProductData` to generate bulk payload files for nightly data‑move jobs. |
| **External APIs** (`/v1/products`, `/v1/orders`) | Consume the JSON representation of `ProductData`. |
| **Configuration Files** (`application.yml`, `product-mapping.properties`) | Provide mapping rules for `accountingCode` and `tclEntity`. No direct reference in the class, but the values are populated from these configs elsewhere. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **NullPointerException** when `productAttributes` or `serviceType` are not initialised before use. | Job failure, incomplete payload. | Initialise collections to empty lists in the default constructor or use `Collections.emptyList()` as fallback. |
| **Schema drift** – downstream API changes field names or expects additional fields. | Rejection of payload, silent data loss. | Keep a versioned contract (OpenAPI/Swagger) and add integration tests that validate serialization against the contract. |
| **Incorrect accounting code** due to stale config mapping. | Billing errors, financial impact. | Externalise the mapping to a config service and add validation (e.g., regex) before setting. |
| **Large attribute list** causing payload size blow‑up. | Network timeouts, API throttling. | Enforce a maximum attribute count; truncate or paginate if needed. |
| **Missing `CustomProductAttribute` definition** leading to serialization failures. | Runtime errors in batch jobs. | Ensure the class is present on the classpath and annotated for JSON (e.g., `@JsonProperty`). |

---

### 6. Example: Running / Debugging the Class  

**Typical usage in a unit test or ad‑hoc script**

```java
public static void main(String[] args) throws Exception {
    ProductData pd = new ProductData();
    pd.setAccountingCode("ACC-12345");
    pd.setTclEntity("INDIA");

    // Populate attributes
    List<CustomProductAttribute> attrs = new ArrayList<>();
    attrs.add(new CustomProductAttribute("color", "red"));
    attrs.add(new CustomProductAttribute("speed", "5G"));
    pd.setProductAttributes(attrs);

    // Service types
    ArrayList<String> svc = new ArrayList<>();
    svc.add("MOBILE");
    pd.setServiceType(svc);

    // Serialize to JSON for quick verification
    ObjectMapper mapper = new ObjectMapper();
    String json = mapper.writerWithDefaultPrettyPrinter().writeValueAsString(pd);
    System.out.println(json);
}
```

*Debug steps*  

1. Set a breakpoint on any setter to verify incoming values.  
2. Inspect the generated JSON – ensure field names match the API contract.  
3. Run the surrounding service (e.g., `ProductService.createProduct(pd)`) and monitor logs for serialization errors.  

In production, the class is never executed directly; it is instantiated by the service layer as part of the request handling pipeline.

---

### 7. External Configuration, Environment Variables & Files  

| Item | Usage |
|------|-------|
| `application.yml` / `product-mapping.properties` | Provide mapping for `accountingCode` and `tclEntity` based on product type or region. |
| Environment variable `TCL_ENTITY_DEFAULT` (hypothetical) | May be used by the service layer to set a default `tclEntity` when none is supplied. |
| No direct file references inside `ProductData.java`. All configuration is consumed upstream before the POJO is populated. |

---

### 8. Suggested TODO / Improvements  

1. **Add defensive initialization** – Provide a no‑arg constructor that creates empty `productAttributes` (`new ArrayList<>()`) and `serviceType` (`new ArrayList<>()`) to avoid NPEs.  
2. **Implement `equals()`, `hashCode()`, and `toString()`** (or switch to Lombok `@Data`) to simplify debugging and collection handling, and to enable reliable comparisons in unit tests.  

--- 

*End of documentation.*