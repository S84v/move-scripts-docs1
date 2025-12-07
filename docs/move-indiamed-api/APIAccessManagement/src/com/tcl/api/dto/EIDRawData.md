**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\dto\EIDRawData.java`

---

## 1. High‑Level Summary
`EIDRawData` is a plain‑old‑Java‑Object (POJO) that models a single row of “raw” equipment‑identification data extracted from source systems (e.g., provisioning DB, SFTP feeds). It is the canonical data‑carrier used by the *RawDataAccess* DAO and the various call‑out services (`PostOrder`, `OrderStatus`, `ProductDetails`, etc.) to transport, transform, and persist the fields required for downstream API calls and reporting.

---

## 2. Important Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **class `EIDRawData`** | Container for 24 string attributes representing an equipment record (e.g., `eid`, `iccid`, `msisdn`, plan details, transaction status). |
| **Getters / Setters** (e.g., `getEid()`, `setEid(String)`) | Provide mutable access for DAO mapping, JSON serialization, and business‑logic transformations. |
| **`toString()`** | Human‑readable dump of all fields – primarily for logging/debugging. |
| **(implicit default constructor)** | Allows frameworks (Jackson, Spring, JDBC) to instantiate the object via reflection. |

*No business logic resides in this file; it is deliberately lightweight to keep serialization and mapping fast.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | Values are populated by: <br>• `RawDataAccess` when reading rows from the **SQL Server** database (via `ResultSet` mapping). <br>• CSV / SFTP parsers that create an instance per line. |
| **Outputs** | The populated object is passed to: <br>• Service call‑outs (`PostOrder`, `OrderStatus`, etc.) for API payload construction. <br>• Persistence layer for updates or inserts back to the DB. |
| **Side Effects** | None – the class is a pure data holder. |
| **Assumptions** | • All fields are stored as `String` (even dates, numeric IDs). <br>• Caller is responsible for null‑checking, format validation, and conversion to domain types (e.g., `java.time.LocalDate`). <br>• Field names map 1‑to‑1 to DB column names defined in `APIConstants` or DAO SQL statements. |
| **External Dependencies** | • `RawDataAccess` (DAO) for DB interaction. <br>• Configuration files that define DB connection strings (`SQLServerConnection`). <br>• Environment variables for DB credentials (not used directly here). |

---

## 4. Interaction with Other Scripts & Components

| Component | Interaction Pattern |
|-----------|---------------------|
| **`RawDataAccess` (DAO)** | Calls `new EIDRawData()` and populates fields via setters when iterating a `ResultSet`. May also read `toString()` for debug logs. |
| **Call‑out Services (`PostOrder`, `OrderStatus`, `ProductDetails`, `UsageProdDetails`, etc.)** | Accept an `EIDRawData` instance as a method argument, extract needed fields, and build external API request bodies (JSON/XML). |
| **Transformation Utilities** (if any) | May convert date strings to `LocalDate`, map status codes, or enrich the DTO before sending downstream. |
| **Batch Orchestration Scripts** (e.g., Spring Batch jobs, custom “move” scripts) | Instantiate collections of `EIDRawData` objects to represent a batch, then hand them off to the DAO or service layer. |
| **Logging / Monitoring** | `toString()` is used in log statements to trace record flow; the string is written to application logs or monitoring dashboards. |

*Because the DTO is used throughout the “move” pipeline, any change to field names or types must be reflected in all dependent DAO queries, JSON mappers, and API payload builders.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Unvalidated / malformed data** (e.g., dates as “N/A”) | Downstream API rejections, DB constraint violations. | Add a validation layer (e.g., Bean Validation annotations) before the DTO is handed to services. |
| **Sensitive data exposure via `toString()`** (ICCID, MSISDN) | Logs may be ingested by SIEMs or accessed by unauthorized personnel. | Redact or mask sensitive fields in `toString()` or use a dedicated logger that omits them. |
| **String‑only representation** leads to runtime parsing errors (e.g., numeric overflow). | Service failures, incorrect calculations. | Convert critical numeric/date fields to proper types early in the pipeline; keep the DTO as a transport object only. |
| **Schema drift** (DB column added/renamed) without DTO update. | Data loss or `null` values silently propagated. | Implement integration tests that compare DAO column metadata with DTO fields; enforce compile‑time mapping (e.g., MyBatis, JPA). |
| **Large batch memory consumption** (list of DTOs held in RAM). | Out‑of‑Memory errors under high volume. | Stream records using cursor‑based processing; avoid loading entire batch into a `List<EIDRawData>` when possible. |

---

## 6. Typical Run / Debug Workflow

1. **Start the batch job** (e.g., `java -jar move-indiamed-api.jar --job=PostOrder`).  
2. The job’s **reader** (DAO) executes a SQL query, creates an `EIDRawData` per row, and logs `EIDRawData.toString()` at DEBUG level.  
3. The **processor** receives the DTO, optionally runs validation (`Validator.validate(eidRawData)`).  
4. The **writer** (service call‑out) extracts needed fields and builds the external API payload.  
5. **Debugging**:  
   - Set a breakpoint in `RawDataAccess.mapRowToEIDRawData(ResultSet)` to inspect field population.  
   - Increase log level for the package `com.tcl.api.dto` to see the full `toString()` output.  
   - Use a unit test that constructs an `EIDRawData` with known values and asserts that downstream service builders produce the expected JSON.  

*Because the class itself contains no logic, debugging focuses on the code that populates or consumes the DTO.*

---

## 7. External Config / Environment References

| Reference | Usage |
|-----------|-------|
| **`APIConstants`** (package `com.tcl.api.constants`) | May contain column‑name constants used by DAO SQL strings that map to the DTO fields. |
| **`SQLServerConnection`** | Provides the JDBC URL, username, and password (via environment variables) used by `RawDataAccess` to fetch rows that become `EIDRawData`. |
| **`application.properties` / `move‑indiamed‑api.yml`** (not shown) | Likely defines batch size, fetch size, and logging levels that affect how many `EIDRawData` objects are held in memory. |

*No direct configuration is read inside `EIDRawData`; all external parameters are consumed by the surrounding layers.*

---

## 8. Suggested Improvements (TODO)

1. **Add Bean Validation annotations** (e.g., `@NotNull`, `@Pattern`) to critical fields (`eid`, `iccid`, `msisdn`) and expose a `validate()` method or integrate with a validation framework to catch malformed data early.  
2. **Replace manual getters/setters with Lombok** (`@Data`, `@Builder`) to reduce boilerplate, enforce immutability where possible, and automatically generate `equals()`/`hashCode()` for collection handling.  

---