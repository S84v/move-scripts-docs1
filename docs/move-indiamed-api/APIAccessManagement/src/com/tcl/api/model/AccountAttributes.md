**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\model\AccountAttributes.java`

---

## 1. High‑Level Summary
`AccountAttributes` is a plain‑old‑Java‑Object (POJO) that models the static and tax‑related properties of a telecom customer account. It is populated by the data‑access layer (`RawDataAccess`, `UsageProdDetails*`) after querying the SQL‑Server back‑end and is later consumed by the API‑exposure layer (e.g., call‑out classes in `com.tcl.api.callout`) to build JSON responses for external partners. The class carries no business logic; it simply holds fields such as service type, billing method, GST details, and a list of associated `Product` objects.

---

## 2. Important Classes / Functions

| Element | Responsibility |
|---------|-----------------|
| **AccountAttributes** (this file) | Container for account‑level attributes; provides getters/setters for all fields. |
| **Product** (referenced type) | Represents an individual product/sub‑service attached to the account; likely defined in `com.tcl.api.model.Product`. |
| **RawDataAccess** (DAO) | Retrieves raw rows from SQL Server, maps them into `AccountAttributes` (and `Product`) objects. |
| **UsageProdDetails / UsageProdDetails2** (call‑out) | Consume `AccountAttributes` to construct API payloads for usage‑related endpoints. |
| **APIConstants / StatusEnum** | Provide constant values (e.g., GST codes) that may be used when populating `AccountAttributes`. |

*All getters and setters are standard Java bean methods; no additional behavior is present.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | Data sourced from the `EIDRawData` DTO (raw DB rows) and/or stored‑procedure results. The DAO translates column values into the fields of `AccountAttributes`. |
| **Outputs** | An instantiated `AccountAttributes` object that is passed downstream to: <br>• JSON marshaller (Jackson/Gson) for REST responses <br>• Internal transformation pipelines (e.g., enrichment, validation) |
| **Side Effects** | None – the class is immutable from the perspective of external services; it only holds state. |
| **Assumptions** | <ul><li>All string fields may be `null` if the source column is missing; callers must handle nulls.</li><li>`products` list is never `null` after DAO mapping (empty list if no products).</li><li`Product` class follows a similar POJO pattern.</li></ul> |

---

## 4. Integration Points with Other Scripts / Components

| Component | Interaction |
|-----------|-------------|
| **SQLServerConnection** | Provides the JDBC connection used by `RawDataAccess` to fetch rows that populate `AccountAttributes`. |
| **RawDataAccess** | Calls `new AccountAttributes()` and invokes the setters to fill data. |
| **UsageProdDetails / UsageProdDetails2** | Accept an `AccountAttributes` instance (often via a wrapper DTO) to build the response payload for the “usage‑product‑details” API endpoint. |
| **API Call‑out Layer (`com.tcl.api.callout`)** | Serialises the object to JSON; may also apply business rules based on `serviceType`, `billingMethod`, etc. |
| **External Config** | No direct config usage inside this class, but field values may be validated against constants defined in `APIConstants`. |
| **Testing / Validation Scripts** | Unit tests for DAO mapping will instantiate `AccountAttributes` directly to verify field population. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Mitigation |
|------|------------|
| **Null‑Pointer Exceptions** when downstream code assumes non‑null fields (e.g., `getGstinNumber().length()`). | Enforce null‑checks or use `Optional<String>` in future revisions; add defensive validation in the DAO layer. |
| **Schema Drift** – DB column changes not reflected in this POJO cause missing data. | Implement automated schema‑to‑POJO verification (e.g., using JOOQ code‑gen) and include a regression test that fails on mismatches. |
| **Large `products` List** leading to memory pressure during bulk API calls. | Stream products where possible; consider pagination or lazy loading in the DAO. |
| **Incorrect GST Mapping** (e.g., wrong state code). | Centralise GST validation logic in a utility class and reference it from the DAO before setting fields. |

---

## 6. Running / Debugging the Class

1. **Unit Test** – Create a JUnit test that constructs an `AccountAttributes` instance, sets each field, and asserts getters return the same values.  
2. **DAO Debug** – Set a breakpoint in `RawDataAccess.mapToAccountAttributes(ResultSet rs)` (or equivalent) and step through the setter calls to verify correct mapping.  
3. **API End‑to‑End** – Invoke the relevant REST endpoint (e.g., `/api/v1/account/details`) with a known account ID; capture the JSON response and confirm the fields match the expected DB values.  
4. **Logging** – If needed, add temporary `log.debug("AccountAttributes populated: {}", this)` in the DAO after object construction to view the full state.

---

## 7. External Configuration / Environment Variables

- **None directly used** in this POJO.  
- Indirectly depends on:  
  - `APIConstants` (static values for GST codes, service types).  
  - Database connection parameters defined for `SQLServerConnection` (JDBC URL, credentials).  
  - Any feature‑toggle flags that may affect which fields are populated (e.g., a flag to suppress GST details).

---

## 8. Suggested Improvements (TODO)

1. **Add Validation Method** – Implement a `validate()` method that checks mandatory fields (e.g., `serviceType`, `profileId`) and throws a domain‑specific exception if constraints are violated. This centralises data integrity checks before the object is sent to the API layer.  
2. **Replace Raw Strings with Value Objects** – Wrap GST‑related fields (`gstinNumber`, `gstProductCode`, address components) in dedicated value‑object classes to enforce format rules and improve readability.  

---