**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\model\UsageData.java`

---

## 1. High‑Level Summary
`UsageData` is a plain‑old‑Java‑Object (POJO) that represents a customer‑level usage payload exchanged by the **API Access Management Migration** service. It carries the customer reference, the associated account number, and a list of `UsageDetail` records (the individual usage items). The class is used throughout the migration flow for JSON/XML (de)serialization, mapping to downstream billing or analytics systems, and for validation before persisting or forwarding the data.

---

## 2. Key Class & Members

| Class / Member | Responsibility |
|----------------|----------------|
| **`UsageData`** | Container for a single usage request. Holds three fields and provides standard JavaBean getters/setters. |
| `private String customerReference` | Identifier of the customer (e.g., MSISDN, subscriber ID). |
| `private String accountNumber` | Billing/account identifier linked to the customer. |
| `private List<UsageDetail> usageDatailsArray` | Collection of per‑record usage entries. (`UsageDetail` is defined elsewhere in the same package.) |
| `getCustomerReference()` / `setCustomerReference(String)` | Accessor & mutator for `customerReference`. |
| `getAccountNumber()` / `setAccountNumber(String)` | Accessor & mutator for `accountNumber`. |
| `getUsageDatailsArray()` / `setUsageDatailsArray(List<UsageDetail>)` | Accessor & mutator for the usage details list. |

*Note:* The field name `usageDatailsArray` contains a typographical error (“datails”). It is retained for backward compatibility but should be addressed (see TODO).

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Input source** | Populated by deserialization of inbound API payloads (e.g., HTTP POST body, SFTP file, or MQ message). |
| **Output destination** | Passed to downstream services (billing, analytics, or persistence layers) as a Java object or re‑serialized to JSON/XML. |
| **Side effects** | None intrinsic to the POJO; side effects arise only from callers that modify the object (e.g., adding `UsageDetail` entries). |
| **Assumptions** | <ul><li>`UsageDetail` class exists and is correctly mapped.</li><li>Calling code respects JavaBean conventions (e.g., uses getters/setters).</li><li>No validation is performed inside this class – validation is expected upstream.</li></ul> |
| **External dependencies** | None directly; however the class is part of a larger Maven/Gradle module that includes Jackson/Gson for (de)serialization, Spring/Dropwizard for REST handling, and possibly a messaging framework (Kafka, JMS). |

---

## 4. Integration Points & Connectivity

| Connected Component | How `UsageData` is used |
|---------------------|--------------------------|
| **API Controllers** (`com.tcl.api.controller.*`) | Incoming HTTP requests are unmarshalled into `UsageData` instances (Jackson). |
| **Service Layer** (`com.tcl.api.service.*`) | Business logic validates the fields, enriches the `usageDatailsArray`, and forwards the object to downstream adapters. |
| **Data Transfer Objects** (`Product*` models from previous files) | In some migration scenarios, `UsageData` may be combined with product information to build a composite request for the target system. |
| **Message Producers/Consumers** (`com.tcl.api.kafka.*` or JMS) | The object is serialized and placed on a queue/topic for asynchronous processing. |
| **Persistence/Repository** (`com.tcl.api.repository.*`) | If the migration stores raw usage records, the POJO is mapped to an ORM entity (e.g., JPA) before persisting. |
| **External Systems** | Target billing/analytics APIs expect a payload that mirrors the structure of `UsageData`. The class therefore acts as the contract model. |

*Because the file is a model class, it does not contain any runtime logic; its connectivity is defined by the code that references it.*

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Typo in field name (`usageDatailsArray`)** | Breaks downstream JSON mapping if a different name is expected; may cause silent data loss. | Add a `@JsonProperty("usageDetailsArray")` annotation (Jackson) to map the correct JSON key, and schedule a refactor to rename the field while preserving backward compatibility. |
| **Lack of validation** | Corrupt or incomplete payloads could propagate to billing systems, causing financial errors. | Enforce validation in the service layer (e.g., using Bean Validation `@NotNull`, custom validators). |
| **Mutable POJO** | Unintended modifications after object creation can lead to race conditions in multi‑threaded pipelines. | Consider making the class immutable (final fields, constructor injection) if the usage pattern permits. |
| **Version drift** | If other services evolve the JSON schema, this class may become out‑of‑sync. | Maintain a versioned schema definition (e.g., OpenAPI) and generate model classes from it. |
| **Serialization incompatibility** | Different libraries (Jackson vs. Gson) may treat the typo differently. | Standardize on a single serialization library across the project. |

---

## 6. Running / Debugging the Class

1. **Unit Test Example**  
   ```java
   @Test
   public void testSerialization() throws Exception {
       UsageDetail d1 = new UsageDetail("CALL", 120);
       UsageDetail d2 = new UsageDetail("SMS", 30);
       UsageData data = new UsageData();
       data.setCustomerReference("CUST12345");
       data.setAccountNumber("ACC98765");
       data.setUsageDatailsArray(Arrays.asList(d1, d2));

       ObjectMapper mapper = new ObjectMapper();
       String json = mapper.writeValueAsString(data);
       // Verify JSON contains the expected keys
       assertTrue(json.contains("\"customerReference\":\"CUST12345\""));
       // If using @JsonProperty, also verify correct key for usage details
   }
   ```

2. **Debugging in IDE**  
   - Set a breakpoint on any setter or on the controller method that receives the payload.  
   - Inspect the `usageDatailsArray` field; if it is `null`, verify that the incoming JSON key matches the typo or the `@JsonProperty` mapping.  

3. **Running the Full Migration Service**  
   - Build the module: `mvn clean package` (or `./gradlew build`).  
   - Deploy the generated JAR/WAR to the application server (Tomcat, Jetty, or Spring Boot).  
   - Use a tool like `curl` or Postman to POST a JSON payload to the endpoint that expects `UsageData`.  
   - Monitor logs (`/var/log/tcl-api/*.log`) for any deserialization errors (e.g., `UnrecognizedPropertyException`).  

---

## 7. External Configuration / Environment Variables

The class itself does **not** read any configuration files, environment variables, or external resources. Its behavior is entirely driven by the callers. However, the surrounding application may rely on:

| Config Item | Usage |
|-------------|-------|
| `API_BASE_URL` | Endpoint where the serialized `UsageData` is posted. |
| `KAFKA_TOPIC_USAGE` | Topic name used by the producer that forwards `UsageData`. |
| `SPRING_PROFILES_ACTIVE` | Determines which serializer (Jackson vs. Gson) is active. |

If any of these change, ensure that the serialization format (field naming) remains compatible with the `UsageData` definition.

---

## 8. Suggested TODO / Improvements

1. **Rename / Annotate the typo field**  
   ```java
   @JsonProperty("usageDetailsArray")
   private List<UsageDetail> usageDatailsArray;
   ```
   Then, after all dependent services are updated, rename the field to `usageDetailsArray` and remove the annotation.

2. **Add Bean Validation annotations**  
   ```java
   @NotBlank
   private String customerReference;
   @NotBlank
   private String accountNumber;
   @NotEmpty
   private List<UsageDetail> usageDatailsArray;
   ```
   This enforces mandatory data early in the processing pipeline.

---