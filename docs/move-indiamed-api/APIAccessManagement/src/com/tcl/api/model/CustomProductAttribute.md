**High‑Level Documentation – `CustomProductAttribute.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\CustomProductAttribute.java`

---

### 1. Purpose (One‑paragraph summary)  
`CustomProductAttribute` is a plain‑old‑Java‑object (POJO) that represents a single name/value pair describing a custom attribute of a product in the API Access Management service. Instances are embedded in the `CustomProduct` model (see `CustomProduct.java`) and are serialized to/from JSON when the service receives or returns product‑customisation data via its REST endpoints. The class carries no business logic; it simply holds data for downstream transformation, persistence, or downstream system integration.

---

### 2. Important Types & Responsibilities  

| Type / Member | Responsibility |
|---------------|----------------|
| **Class `CustomProductAttribute`** | Data container for a product attribute. |
| **Fields** `productAttrName`, `productAttrValue` | Store the attribute’s key and its string value. |
| **Constructor** `CustomProductAttribute(String name, String value)` | Initialise a fully‑populated attribute instance. |
| **Getter** `getProductAttrName()` / `getProductAttrValue()` | Provide read‑access for serialization and business logic. |
| **Setter** `setProductAttrName(String)` / `setProductAttrValue(String)` | Allow mutation when building objects from incoming payloads or when mapping from DB rows. |

*No other methods (e.g., validation, equals/hashCode) are present in the current version.*

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Aspect | Detail |
|--------|--------|
| **Inputs** | Values passed to the constructor or setters – both expected to be non‑null, UTF‑8 strings representing attribute name and value. |
| **Outputs** | Getter methods return the stored strings; the object is later marshalled to JSON (Jackson/Gson) or persisted via JPA/Hibernate (if mapped). |
| **Side Effects** | None – the class is immutable only by convention; it does not touch external resources. |
| **Assumptions** | • The surrounding service validates that `productAttrName` conforms to a known catalogue of attribute keys.<br>• `productAttrValue` may be any free‑form string, but downstream systems may impose length or format constraints.<br>• The class is used only within the `move-indiamed-api` code‑base; no reflection‑based frameworks rely on additional annotations. |

---

### 4. Integration Points (How this file connects to other components)  

| Connected Component | Relationship |
|---------------------|--------------|
| **`CustomProduct.java`** | Holds a `List<CustomProductAttribute>` (or similar collection). When a `CustomProduct` is built for an API response, each attribute is instantiated from this class. |
| **REST Controllers** (e.g., `ProductController`) | Accept/return JSON payloads that include an array of `{productAttrName, productAttrValue}` objects; Jackson automatically maps them to `CustomProductAttribute`. |
| **Service Layer** (e.g., `ProductService`) | May construct `CustomProductAttribute` objects from DB rows (`ResultSet`) or from external system responses (e.g., SFTP file, third‑party API). |
| **Persistence Layer** (if JPA is used) | Potentially mapped via `@Embeddable` or a separate table; the class would be referenced from an entity representing a custom product. |
| **Message Queues / Event Bus** | When a product change event is published, the attribute list (including this POJO) is serialized into the event payload. |
| **Testing Utilities** | Unit tests for `CustomProduct` or API contract tests instantiate this class directly. |

*Note:* No direct file‑system, SFTP, or external service calls originate from this class; it is purely a data model.

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Null / Empty Attribute Names** | May cause downstream validation failures or DB constraint violations. | Enforce non‑null, non‑empty checks in the service layer or add a simple validation method inside the class. |
| **Inconsistent Naming** (e.g., case‑sensitivity) | Could lead to duplicate attributes or mismatched look‑ups. | Define a canonical format (e.g., upper‑case) when setting the name, or document naming conventions. |
| **Serialization Mismatch** (field renamed without updating JSON mapping) | API contract breakage. | Keep field names stable; if a rename is required, add `@JsonProperty` annotations and version the API. |
| **Missing `equals`/`hashCode`** | Collections (e.g., `Set`) may treat distinct objects as duplicates or fail to deduplicate correctly. | Implement `equals` and `hashCode` based on both fields. |
| **Unbounded Attribute Value Length** | Could overflow DB columns or message payload size limits. | Add length validation in the service layer or annotate with JPA `@Column(length=…)` if persisted. |

---

### 6. Running / Debugging the Class  

| Scenario | Steps |
|----------|-------|
| **Unit Test** | ```java\n@Test\npublic void testAttributeCreation() {\n    CustomProductAttribute attr = new CustomProductAttribute(\"color\", \"red\");\n    assertEquals(\"color\", attr.getProductAttrName());\n    assertEquals(\"red\", attr.getProductAttrValue());\n}\n``` |
| **API Debug** | 1. Start the `move-indiamed-api` Spring Boot (or equivalent) service.<br>2. Issue a GET/POST request that returns a `CustomProduct` payload.<br>3. Inspect the JSON array under `attributes` – each element should map to this class.<br>4. Use a debugger to step into the controller/service where `new CustomProductAttribute(..)` is called. |
| **Serialization Check** | ```java\nObjectMapper mapper = new ObjectMapper();\nCustomProductAttribute a = new CustomProductAttribute(\"size\",\"L\");\nString json = mapper.writeValueAsString(a);\nSystem.out.println(json); // {"productAttrName":"size","productAttrValue":"L"}\n``` |
| **Database Mapping Test** (if JPA is used) | Verify the entity mapping file or annotations; run an integration test that persists a `CustomProduct` containing this attribute and reads it back. |

---

### 7. External Configuration / Environment Dependencies  

| Config / Env | Usage |
|--------------|-------|
| **None** | The class itself does not read any configuration files, environment variables, or system properties. All external dependencies (e.g., JSON mapper settings, DB column lengths) are defined elsewhere in the application configuration. |

If the surrounding service uses a property file to define allowed attribute names or max lengths, those settings will be consulted by the service layer—not by this POJO.

---

### 8. Suggested TODO / Improvements  

1. **Add Basic Validation** – Implement a `validate()` method (or use Bean Validation annotations `@NotBlank`) to ensure `productAttrName` is non‑null/non‑empty and conforms to allowed patterns.  
2. **Override `equals()`, `hashCode()`, and `toString()`** – Improves collection handling, logging, and test readability.  

*Optional*: Annotate with Jackson’s `@JsonProperty` to decouple JSON field names from Java field names, allowing future refactoring without breaking the API contract.