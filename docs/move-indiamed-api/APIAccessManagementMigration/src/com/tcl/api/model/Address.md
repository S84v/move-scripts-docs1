**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\model\Address.java`

---

### 1. High‑Level Summary
`Address` is a plain‑old‑Java‑Object (POJO) that represents a physical mailing address used throughout the API Access Management Migration project. It is instantiated by DAO and service layers (e.g., `RawDataAccess`, `Accountdetails`, `AccountDetail`) to carry address information between the database, internal transformation logic, and downstream REST responses.

---

### 2. Core Class & Responsibilities  

| Class / Interface | Responsibility | Notes |
|-------------------|----------------|-------|
| **`Address`** | Holds address fields (`addrLine1`, `addrLine2`, `addrLine3`, `city`, `state`, `country`, `zipcode`) with standard getters/setters. No business logic – purely a data carrier. | Used as a member of higher‑level DTOs (`Accountdetails`, `AccountDetail`, etc.). |

*No additional functions or inner classes are defined in this file.*

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Detail |
|--------|--------|
| **Inputs** | Values are supplied via setters by callers (DAO mapping from DB rows, JSON deserialization, or manual construction in tests). |
| **Outputs** | Getter methods expose the stored values to callers (service layer, response builders, logging). |
| **Side‑Effects** | None – the class does not perform I/O, network calls, or mutate external state beyond its own fields. |
| **Assumptions** | <ul><li>`zipcode` is an integer; callers assume it fits within Java `int` range and that leading zeros are not required.</li><li>All string fields may be `null` unless validated elsewhere.</li><li>The class is instantiated via the default constructor (implicit) and populated via setters.</li></ul> |
| **External Dependencies** | None. The class does not reference configuration files, environment variables, or external services. |

---

### 4. Interaction with Other Scripts & Components  

| Component | How `Address` is Used |
|-----------|-----------------------|
| **DAO Layer (`RawDataAccess.java`)** | Maps DB columns (`addr_line1`, `city`, etc.) to an `Address` instance when reading account records. |
| **DTOs (`Accountdetails.java`, `AccountDetail.java`, `AccountAttributes.java`)** | Contains an `Address` field to embed address data in higher‑level objects that are later serialized to JSON for API responses. |
| **Service / Business Logic** | May receive an `Address` from the DAO, enrich or validate it, then pass it downstream to response builders or external APIs. |
| **Testing / Mock Scripts** | Test harnesses instantiate `Address` directly to feed known data into service methods. |
| **Potential Future Integration** | If the migration later pushes address data to an external CRM via REST or SFTP, the same POJO will be serialized and transmitted. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Invalid Zipcode Format** – integer loses leading zeros (e.g., “01234”). | Incorrect address data in downstream systems, possible billing errors. | Store zipcode as `String` or add a formatting utility; validate length/pattern before persisting. |
| **Null Pointer Exceptions** – callers assume non‑null fields. | Runtime failures in serialization or downstream processing. | Enforce non‑null constraints in DAO mapping or add defensive checks in setters. |
| **Missing Validation** – no checks on field length, allowed characters, or country codes. | Data quality issues, downstream API rejections. | Introduce a validation method (or use Bean Validation annotations) to enforce business rules. |
| **Lack of Equality/Hashing** – default `Object.equals` may cause duplicate detection failures. | Incorrect deduplication or caching behavior. | Override `equals()` and `hashCode()` (or generate via Lombok). |
| **No `toString()`** – debugging logs show object reference only. | Harder troubleshooting. | Implement a concise `toString()` that prints all fields. |

---

### 6. Running / Debugging the Class  

| Step | Action |
|------|--------|
| **Compile** | `mvn clean compile` (or the project's standard build command). The class is compiled automatically as part of the `com.tcl.api.model` package. |
| **Unit Test** | Write a simple JUnit test: instantiate `Address`, set each field, assert getters return the same values. Example: <br>`Address a = new Address(); a.setCity("Mumbai"); assertEquals("Mumbai", a.getCity());` |
| **Debug** | Set breakpoints in any DAO or service method that creates an `Address`. Inspect field values after population to verify mapping correctness. |
| **Log Inspection** | If `toString()` is added, enable logging of the object at `DEBUG` level to see full content in application logs. |

---

### 7. External Config / Environment References  

- **None** – `Address` does not read configuration files, environment variables, or external resources. All values are supplied by callers.

---

### 8. Suggested TODO / Improvements  

1. **Add Validation** – Implement a `validate()` method (or use JSR‑380 Bean Validation annotations) to enforce non‑null, length, and format constraints for each field, especially `zipcode`.  
2. **Replace `int zipcode` with `String zipcode`** – Preserve leading zeros and support alphanumeric postal codes used in some regions. Update DAO mapping and any dependent code accordingly.  

--- 

*End of documentation.*