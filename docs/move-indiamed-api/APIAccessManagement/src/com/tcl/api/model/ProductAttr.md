**High‑Level Documentation – `ProductAttr.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\ProductAttr.java`

---

## 1. Purpose (one‑paragraph summary)

`ProductAttr` is a plain‑old‑Java‑object (POJO) that models the extensive set of attributes associated with a telecom product or service line item within the **IndiaMED API**. It is used by the order‑management flow (e.g., `OrderResponse`, `PostOrderDetail`, `Product`) to transport product‑level data between the API layer, internal orchestration scripts, and downstream systems such as billing, provisioning, and partner‑management services. The class contains only getters/setters and no business logic; its responsibility is to hold raw attribute values that are later serialized to/from JSON (or XML) for external consumption.

---

## 2. Important Class(es) & Responsibilities

| Class | Responsibility |
|-------|----------------|
| **`ProductAttr`** | • Holds every product‑level field required by downstream systems (billing, provisioning, partner‑tracking, GST, bundle usage, etc.).<br>• Provides JavaBean‑style getters and setters for serialization frameworks (Jackson, Gson, JAXB).<br>• Acts as a sub‑object of higher‑level models such as `Product` and `PostOrderDetail`. |

*No other classes are defined in this file.*

---

## 3. Public API (Fields & Accessors)

| Field | Type | Typical Meaning |
|-------|------|-----------------|
| circuitId | `String` | Physical/virtual circuit identifier |
| usageModel | `String` | Consumption model (e.g., prepaid, postpaid) |
| copfId | `String` | Cost‑of‑Provisioning Framework ID |
| commissioningDate / commisioningDate | `String` | Date the service was commissioned (duplicate spelling) |
| commercialCode | `String` | Commercial product code |
| serviceTypeClubbing, serviceType, serviceTypeFlavour | `String` | Service classification & variant |
| billingType, billingEffectiveDate, billableType, billablecircuit | `String` | Billing‑related flags |
| siteEnd, siteGSTINNo, siteLocationId | `String` | Site‑level identifiers |
| parentService, parentId, parentserviceName | `String/int` | Hierarchical relationship to a parent service |
| partner* (isPartner, partnerEntity, partnerName, …) | `String` | Partner‑related metadata |
| bundle* (bundleId, bundleCategory, bundleType, …) | `String` | Bundle configuration and usage counters |
| po* (poNumber, poDate, poLineItemDescription, …) | `String` | Purchase‑order details |
| gstProductCode, opportunityID, secsId, parentId | `int` | Numeric identifiers |
| eID, planName, partNumber, circuitDesignation, … | `String` | Miscellaneous product attributes |
| **In‑bundle usage fields** (e.g., `inBundleDataUsage`, `inBundleSMMODomesticUsage`, `inBundlevoiceUsage`, …) | `String` | Counters for data, SMS, voice usage inside a bundle |
| manyToOneBasedOn, propositionDestination, propostionAddOn | `String` | Mapping to proposition logic |
| sponsor, aEndBEndCharging, moCountryExclusionList | `String` | Additional business rules |
| … *(~80 total fields)* | | |

> **Note:** Several fields have inconsistent naming/casing (e.g., `TaxCompletionDate` vs `taxCompletionDate`, duplicate `commissioningDate` vs `commisioningDate`). This reflects legacy data‑source contracts.

---

## 4. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Detail |
|--------|--------|
| **Inputs** | Populated by upstream services (order intake, CRM, partner portals) via JSON/XML deserialization or manual mapping in orchestration scripts. |
| **Outputs** | Serialized back to JSON/XML when the API returns an order response, or passed to downstream batch jobs (billing, provisioning). |
| **Side Effects** | None – the class is a pure data holder. |
| **Assumptions** | • All fields are optional unless validated elsewhere.<br>• String fields may contain dates, numeric values, or codes; callers must parse/format as needed.<br>• The object may be persisted in a NoSQL store (e.g., Mongo) or relational DB via an ORM that relies on JavaBean conventions. |

---

## 5. Integration Points (How this file connects to other components)

| Connected Component | Interaction |
|---------------------|-------------|
| **`Product.java`** (sibling model) | `Product` contains a `List<ProductAttr>` (or a single `ProductAttr`) to represent the full product definition. |
| **`OrderResponse.java`** | The response payload includes a collection of `ProductAttr` objects under the product section. |
| **`PostOrderDetail.java`** | When posting an order, the request body embeds `ProductAttr` to convey all product‑level attributes to downstream provisioning. |
| **Serialization Layer** (Jackson/Gson) | The getters/setters enable automatic (de)serialization of JSON payloads exchanged with external APIs (partner portals, billing system). |
| **Orchestration Scripts** (e.g., Groovy/Java batch jobs) | Scripts read/write `ProductAttr` instances while transforming data between CRM, OSS/BSS, and partner systems. |
| **Database Mappers** (MyBatis/Hibernate) | If persisted, field names map directly to column names in the product‑attributes table. |
| **External Config** | No direct config is referenced inside the class, but mapping files (e.g., `product-attr-mapping.yml`) may define how source fields map to these attributes. |

---

## 6. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Field‑name drift / typo** (e.g., duplicate `commissioningDate` fields) | Incorrect data sent to downstream systems, causing order rejections. | Implement a code‑review checklist for POJO naming; add unit tests that verify JSON field names against a contract schema. |
| **Unvalidated data types** (numeric values stored as `String`) | Runtime parsing errors in downstream jobs. | Introduce validation layer (e.g., Bean Validation annotations) or convert to proper types where feasible. |
| **Large object size** (≈80 fields) → serialization overhead | Increased latency for API responses and higher memory consumption. | Consider splitting the model into logical sub‑objects (e.g., `PartnerInfo`, `BundleUsage`) and use lazy loading where possible. |
| **Missing required fields** (no null‑checks) | Downstream provisioning failures. | Add a validation utility that checks mandatory fields before sending to external systems. |
| **Inconsistent casing** (e.g., `TaxCompletionDate`) causing mapping failures | Data loss or mis‑interpretation. | Standardise field naming (e.g., all camelCase) and use Jackson `@JsonProperty` to map legacy names. |

---

## 7. Running / Debugging the Class

1. **Unit‑Test Example**  
   ```java
   @Test
   public void shouldSerializeAndDeserializeCorrectly() throws Exception {
       ProductAttr attr = new ProductAttr();
       attr.setCircuitId("CIR12345");
       attr.setPlanName("Premium_5G");
       attr.setBillingEffectiveDate("2024-09-01");
       // … set a few more fields

       ObjectMapper mapper = new ObjectMapper();
       String json = mapper.writeValueAsString(attr);
       ProductAttr deserialized = mapper.readValue(json, ProductAttr.class);

       assertEquals(attr.getCircuitId(), deserialized.getCircuitId());
       assertEquals(attr.getPlanName(), deserialized.getPlanName());
   }
   ```

2. **Debugging in an IDE**  
   - Set a breakpoint on any setter/getter you suspect.  
   - Run the API endpoint (e.g., `GET /orders/{id}`) that returns an `OrderResponse`.  
   - Inspect the `ProductAttr` instance inside the response object to verify field population.

3. **Integration Test** (end‑to‑end)  
   - Use a test profile that points to a sandbox BSS system.  
   - Submit a `POST /orders` payload containing a `productAttr` JSON block.  
   - Verify the downstream provisioning mock receives the exact same JSON.

---

## 8. External Configuration / Environment Variables

| Config Item | Usage |
|-------------|-------|
| **Mapping files** (e.g., `product-attr-mapping.yml`) | May define how source system fields map to the POJO properties; not referenced directly in code but used by transformation scripts. |
| **Serialization settings** (`application.yml` → `spring.jackson.*`) | Controls date format, inclusion rules, and naming strategy for JSON output. |
| **Feature flags** (`ENABLE_PRODUCT_ATTR_VALIDATION`) | Could be used by surrounding services to turn on/off validation logic for this model. |

If any of these files are missing, the default JavaBean behavior applies (field name ↔ JSON property name).

---

## 9. Suggested TODO / Improvements

1. **Introduce Lombok** – Replace the verbose getter/setter boilerplate with `@Data` (or `@Getter/@Setter`) to reduce maintenance overhead and eliminate accidental typos.
2. **Add Validation Annotations** – Use JSR‑380 (`@NotNull`, `@Pattern`, `@Size`) on mandatory fields and enforce them in the service layer to catch malformed orders early.

--- 

*End of documentation.*