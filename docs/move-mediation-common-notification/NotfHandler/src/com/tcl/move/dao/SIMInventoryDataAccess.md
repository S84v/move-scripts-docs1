# Summary
`SIMInventoryDataAccess` is a DAO that updates SIM inventory records in the Oracle MOVE database. It provides methods to change subscription status, update IMEI, and add new IMSIs to a SIM, handling SQL preparation, timestamp conversion, logging, and exception wrapping into `DatabaseException`.

# Key Components
- **Class `SIMInventoryDataAccess`**
  - `boolean updateStatus(String sim, String msisdn, String newStatus, String eventDate, String secs)` – updates `prod_status`, `LAST_PROD_STATUS_CHNG_DT`, `SIM_AVAILABLE` (and optionally `CUST_NUM`) for a SIM.
  - `void updateIMEI(String sim, String newImei, String source, String newTimestamp)` – updates `imei`, `imei_source`, `imei_timestamp`.
  - `boolean checkAndAddIMSI(String sim, List<String> newIMSIs)` – reads existing IMSI columns, determines if new IMSIs are needed, and updates up to ten IMSI slots.
  - `private Vector<String> getIMSIandProduct(Map<String, List<String>> imsiList, List<String> newIMSIs)` – selects the product with the most IMSIs and builds a vector indicating update necessity.
  - `private String getStackTrace(Exception e)` – utility to convert stack trace to string for logging.

# Data Flow
| Method | Input | DB Interaction | Output / Side‑Effect |
|--------|-------|----------------|----------------------|
| `updateStatus` | SIM ICCID, MSISDN, new status, event date string, SECS ID | `UPDATE MOVE_SIM_INVENTORY_STATUS` (two possible SQLs) via Oracle connection | Returns `true` if ≥1 row updated; logs actions; throws `DatabaseException` on error. |
| `updateIMEI` | SIM ICCID, new IMEI, source, timestamp string | `UPDATE move_sim_inventory_status` (IMEI fields) | No return value; logs rows updated; throws on error. |
| `checkAndAddIMSI` | SIM ICCID, list of new IMSI strings | `SELECT` existing IMSI columns + `PRODUCT_ID`; conditional `UPDATE` of up to 10 IMSI columns | Returns `true` if update succeeded or not needed; logs detailed steps; throws on error. |
| `getIMSIandProduct` | Map of product → existing IMSIs, list of new IMSIs | Pure in‑memory computation | Vector indicating update flag, product ID, start index, and new IMSIs. |
| `getStackTrace` | Exception | None | String representation of stack trace. |

External services:
- Oracle database accessed via `JDBCConnection.getConnectionOracle()`.
- Logging via Log4j.

# Integrations
- **`JDBCConnection`** – provides the Oracle `Connection` used by all DAO methods.
- **Resource bundle `MOVEDAO`** – supplies SQL statements (`notif.status.update`, `notif.status.update.preact`, `notif.imei.update`, `notif.get.imsi`, `notif.add.imsi`).
- **`DatabaseException`** – custom exception propagated to higher layers (e.g., service or handler classes) for transaction management.
- Potential callers: notification handler services that process SIM lifecycle events (status change, IMEI update, IMSI provisioning).

# Operational Risks
- **Timestamp parsing fragility** – manual substring handling assumes fixed format; malformed dates cause `ParseException`. *Mitigation*: validate input format or use `java.time` parsing.
- **Hard‑coded column order** – update statements rely on exact column positions; schema changes break logic. *Mitigation*: externalize column mapping or use named parameters (e.g., `PreparedStatement` with metadata).
- **Resource leaks on exception** – `finally` block rethrows `DatabaseException` after logging, potentially masking original cause. *Mitigation*: separate close handling from business exception propagation.
- **Concurrent updates** – no explicit locking; race conditions may cause lost IMSI updates. *Mitigation*: use optimistic locking (e.g., version column) or serialize updates per SIM.

# Usage
```java
SIMInventoryDataAccess dao = new SIMInventoryDataAccess();

// Example: update status
try {
    boolean ok = dao.updateStatus("8945001234567890123", "15551234567",
                                  "Active", "2024-11-01 14:23:45", "SEC123");
    System.out.println("Status update success: " + ok);
} catch (DatabaseException e) {
    e.printStackTrace();
}

// Example: add IMSIs
List<String> newImsis = Arrays.asList("310150123456789", "310150987654321");
try {
    boolean updated = dao.checkAndAddIMSI("8945001234567890123", newImsis);
    System.out.println("IMSI add result: " + updated);
} catch (DatabaseException e) {
    e.printStackTrace();
}
```
Run within the application server where `MOVEDAO.properties` and Oracle JDBC driver are available.

# Configuration
- **System properties / environment**: none directly; JDBC credentials are read inside `JDBCConnection`.
- **`MOVEDAO.properties`** (resource bundle) must contain:
  - `notif.status.update`
  - `notif.status.update.preact`
  - `notif.imei.update`
  - `notif.get.imsi`
  - `notif.add.imsi`
- **Log4j configuration** for class `com.tcl.move.dao.SIMInventoryDataAccess`.

# Improvements
1. Replace manual date‑string manipulation with `java.time.format.DateTimeFormatter` and `LocalDateTime` to improve robustness and readability.
2. Refactor IMSI handling to use a dynamic collection (e.g., a separate IMSI table) instead of ten fixed columns, eliminating the need for complex index calculations and reducing schema coupling.