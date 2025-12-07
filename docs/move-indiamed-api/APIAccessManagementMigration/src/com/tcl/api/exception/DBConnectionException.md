**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\exception\DBConnectionException.java`

---

## 1. High‑Level Summary
`DBConnectionException` is a custom checked exception that represents any failure related to database connectivity or data‑integrity issues within the **API Access Management Migration** pipeline. It is thrown by low‑level components such as `SQLServerConnection`, `RawDataAccess`, and any DAO/service classes that interact with the SQL Server back‑end. By encapsulating DB‑specific error information, it enables the orchestration scripts (e.g., `PostOrder_bkp.java`, `ProductDetails.java`) to differentiate between infrastructure failures and business‑logic errors, allowing targeted retries, alerts, or compensation actions.

---

## 2. Important Class & Responsibilities

| Class / Interface | Responsibility |
|-------------------|----------------|
| **`DBConnectionException`** (extends `Exception`) | • Carries a descriptive error message for DB‑related failures.<br>• Provides a single‑argument constructor used throughout the migration code base to propagate DB errors up the call stack.<br>• Acts as a marker type for catch blocks that need to trigger DB‑specific remediation (e.g., retry, alert, rollback). |

*No additional methods or fields are defined beyond the standard `Exception` behavior.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | A `String` message supplied by the caller that describes the DB problem (e.g., “Unable to obtain connection from pool”, “SQL query returned null”). |
| **Outputs** | An instantiated `DBConnectionException` object that can be thrown or logged. |
| **Side Effects** | None – the class is immutable and does not interact with external resources. |
| **Assumptions** | • Callers will catch this exception explicitly or allow it to propagate to a global error handler.<br>• The message supplied is meaningful for operations teams (includes context such as environment, query, or connection details).<br>• The surrounding infrastructure (connection pools, JDBC drivers) throws or translates lower‑level `SQLException`/`SQLTimeoutException` into this custom type where appropriate. |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Component | Interaction |
|-----------|--------------|
| `SQLServerConnection` | Wraps `SQLException`/`SQLTimeoutException` into `DBConnectionException` when establishing or validating a connection. |
| `RawDataAccess` (DAO) | Declares `throws DBConnectionException` on methods that query or update the migration tables. |
| Call‑out scripts (`PostOrder_bkp.java`, `ProductDetails.java`, `UsageProdDetails*.java`) | Catch `DBConnectionException` to decide whether to retry, abort, or move the record to a dead‑letter queue. |
| Global error‑handling framework (if any) | May map `DBConnectionException` to a specific error code in monitoring dashboards (e.g., “DB_CONN_ERR”). |
| Logging utilities (e.g., SLF4J, Log4j) | Typically log `e.getMessage()` and stack trace when caught. |

*Because the exception is a plain Java class, there are no direct configuration or runtime dependencies; its impact is realized through the places that throw or catch it.*

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Uncaught `DBConnectionException`** – leads to job termination without proper cleanup. | Partial data may be left in an inconsistent state. | Ensure every public DAO method declares `throws DBConnectionException` and that the top‑level orchestration script has a catch‑all block that logs, alerts, and performs any needed compensation (e.g., transaction rollback, message re‑queue). |
| **Loss of diagnostic detail** – only a generic message is passed. | Troubleshooting becomes time‑consuming. | Encourage callers to include the original `SQLException` message and relevant context (SQL statement, parameters, environment) in the exception message or as a suppressed exception. |
| **Exception swallowing** – developers catch the exception and do nothing. | Silent failures may cause data loss. | Enforce coding standards (e.g., static analysis rule) that require logging or re‑throwing after catch. |
| **Version drift** – multiple modules define their own DB exception types, causing fragmented handling. | Inconsistent error handling across the pipeline. | Consolidate all DB‑related exceptions under `DBConnectionException` or a hierarchy (e.g., `TransientDBException`, `PermanentDBException`). |

---

## 6. Example: Running / Debugging the Exception in Context

1. **Typical usage pattern** (inside a DAO method):
   ```java
   public List<EIDRawData> fetchRawData(String subscriberId) throws DBConnectionException {
       try (Connection con = sqlServerConnection.getConnection();
            PreparedStatement ps = con.prepareStatement(SQL_FETCH)) {
           ps.setString(1, subscriberId);
           ResultSet rs = ps.executeQuery();
           // map rows → DTOs …
       } catch (SQLException e) {
           // Wrap and propagate
           throw new DBConnectionException(
               "Failed to fetch raw data for subscriber " + subscriberId + ": " + e.getMessage());
       }
   }
   ```

2. **Operator debugging steps**:
   - Locate the job log (e.g., `migration-job-2025-12-04.log`).
   - Search for `DBConnectionException` entries; note the message and stack trace.
   - Verify the underlying DB health (connection pool size, network latency, DB server logs).
   - If the message contains a SQL statement, run it manually in a DB client to reproduce the error.
   - Use a debugger to set a breakpoint on the `throw new DBConnectionException` line if source code is available.

3. **Unit test example** (JUnit):
   ```java
   @Test(expected = DBConnectionException.class)
   public void testFetchRawData_connectionFailure() throws Exception {
       when(mockConnectionProvider.getConnection()).thenThrow(new SQLException("timeout"));
       dao.fetchRawData("12345");
   }
   ```

---

## 7. External Configuration / Environment Variables

`DBConnectionException` itself does **not** read any configuration. However, the components that throw it rely on:

| Config / Env | Used By | Purpose |
|--------------|---------|---------|
| `DB_URL`, `DB_USER`, `DB_PASSWORD` | `SQLServerConnection` | JDBC connection parameters. |
| Connection‑pool settings (`maxPoolSize`, `checkoutTimeout`) | `SQLServerConnection` | Influence the likelihood of connection‑related exceptions. |
| Application‑wide error‑code mapping file (e.g., `error-codes.properties`) | Global error handler | May map `DBConnectionException` to a numeric code for monitoring. |

If the message construction references any of these values, ensure they are correctly populated in the runtime environment.

---

## 8. Suggested TODO / Improvements

1. **Add a constructor that accepts a cause**  
   ```java
   public DBConnectionException(String msg, Throwable cause) {
       super(msg, cause);
   }
   ```  
   This preserves the original `SQLException` stack trace, improving root‑cause analysis.

2. **Introduce a hierarchy for transient vs. permanent DB failures**  
   Create `TransientDBConnectionException` and `PermanentDBConnectionException` extending `DBConnectionException`. This enables the orchestration layer to automatically retry only transient errors.

---