**High‑Level Documentation – `AccountDetail.java`**  
*Path:* `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\AccountDetail.java`

---

### 1. Summary
`AccountDetail` is a plain‑old‑Java‑Object (POJO) that models the core attributes of a telecom customer account required by the API Access Management service. It carries the account number, type, billing entity, and status between the data‑access layer (`RawDataAccess`), the business‑logic callout (`UsageProdDetails2`), and any downstream REST or SOAP responses. The class contains only private fields with standard JavaBean getters and setters; no business logic resides here.

---

### 2. Important Class & Responsibilities
| Class / Method | Responsibility |
|----------------|----------------|
| **`AccountDetail`** | Container for four account‑level fields. Provides JavaBean‑style accessors (`get*`, `set*`). Used as a DTO between DAO, service, and API layers. |
| `getAccountNumber()` / `setAccountNumber(String)` | Retrieve / assign the unique account identifier (e.g., MSISDN or internal ID). |
| `getAccountType()` / `setAccountType(String)` | Retrieve / assign the classification (e.g., “PREPAID”, “POSTPAID”). |
| `getBillingEntity()` / `setBillingEntity(String)` | Retrieve / assign the billing entity code (e.g., “INDIA‑MED”). |
| `getAccountStatus()` / `setAccountStatus(String)` | Retrieve / assign the lifecycle status (e.g., “ACTIVE”, “SUSPENDED”). |

*No other methods are present.*

---

### 3. Inputs, Outputs, Side Effects & Assumptions
| Aspect | Details |
|--------|---------|
| **Inputs** | Values are supplied by callers (DAO reading from SQL Server, external API payloads, or test harnesses). |
| **Outputs** | The populated object is returned to callers; fields are read via getters. |
| **Side Effects** | None – the class is immutable only by convention; setters modify the in‑memory instance. |
| **Assumptions** | <ul><li>All fields are non‑null strings when the object is used downstream (validation is performed elsewhere).</li><li>Field names match column names in `ACCOUNT_DETAIL` (or similar) table accessed via `RawDataAccess`.</li><li>String values conform to enumerations defined in `APIConstants` / `StatusEnum` (e.g., account status). </li></ul> |
| **External Dependencies** | None directly. Indirectly depends on configuration for DB connection (`SQLServerConnection`) and constants (`APIConstants`, `StatusEnum`). |

---

### 4. Integration Points (How This File Connects to the Rest of the System)
| Component | Interaction |
|-----------|-------------|
| **`RawDataAccess`** | Retrieves rows from the database, maps column values to a new `AccountDetail` instance (e.g., `resultSet.getString("ACCOUNT_NUMBER")`). |
| **`UsageProdDetails2`** | Consumes `AccountDetail` objects to enrich usage‑product payloads sent to downstream APIs. |
| **`AccountAttributes`** | May be combined with `AccountDetail` to form a richer DTO for API responses. |
| **API Controllers / Callouts** | Serialize `AccountDetail` (via Jackson/Gson) into JSON/XML for external consumers. |
| **Unit / Integration Tests** | Instantiate `AccountDetail` directly to verify mapping logic. |
| **Configuration / Constants** | Field values are expected to align with constants defined in `APIConstants` (e.g., `ACCOUNT_TYPE_PREPAID`). |

---

### 5. Operational Risks & Recommended Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| **Null / Invalid Field Values** | Downstream services may reject payloads or throw NPEs. | Enforce validation in the DAO or service layer (e.g., `Objects.requireNonNull`). |
| **Schema Drift** | DB column changes break manual mapping. | Centralize mapping logic (e.g., use MyBatis or JPA) and add integration tests that fail on schema changes. |
| **Inconsistent Enum Usage** | `accountStatus` may contain values not defined in `StatusEnum`. | Convert raw strings to `StatusEnum` early; log unexpected values. |
| **Performance Overhead of Object Creation** | High‑volume batch jobs may create millions of objects. | Consider reusing objects or using lightweight record types if Java 17+ is available. |
| **Missing Documentation** | New developers may misuse the POJO. | Keep this high‑level doc in the repo and add Javadoc to each getter/setter. |

---

### 6. Running / Debugging the Class
1. **Unit Test Example**  
   ```java
   @Test
   public void testAccountDetailMapping() {
       AccountDetail ad = new AccountDetail();
       ad.setAccountNumber("1234567890");
       ad.setAccountType("POSTPAID");
       ad.setBillingEntity("INDIA-MED");
       ad.setAccountStatus("ACTIVE");

       assertEquals("1234567890", ad.getAccountNumber());
       // Additional asserts...
   }
   ```
2. **Debugging in Context**  
   - Set a breakpoint in `RawDataAccess.mapRowToAccountDetail(ResultSet rs)` (or equivalent).  
   - Inspect the `AccountDetail` instance after each setter to verify correct values from the DB.  
   - Verify that the object is correctly serialized by the API controller (e.g., using a REST client to view JSON output).  

3. **Running the Full Flow**  
   - Deploy the `move-indiamed-api` WAR/JAR to the application server.  
   - Trigger an API endpoint that fetches usage details (which internally calls `UsageProdDetails2`).  
   - Observe logs for any mapping errors; the POJO itself will not log, so ensure surrounding layers have adequate logging.  

---

### 7. External Config / Environment Variables
- **None** are referenced directly in this file.  
- Indirect dependencies:  
  - DB connection strings, credentials, and pool settings are defined in `SQLServerConnection` (environment variables or property files).  
  - Enumerated values for `accountStatus` and `accountType` are defined in `APIConstants` / `StatusEnum`.  

---

### 8. Suggested Improvements (TODO)
1. **Add Validation & Immutability** – Replace mutable setters with a builder pattern or Java record (if on Java 17+) to guarantee non‑null, validated fields at construction time.  
2. **Generate Boilerplate Automatically** – Use Lombok (`@Data`, `@Builder`) to reduce boilerplate and keep the class in sync with future field additions.  

---