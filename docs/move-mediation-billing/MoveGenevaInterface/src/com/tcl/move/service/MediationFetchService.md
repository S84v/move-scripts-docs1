# Summary
`MediationFetchService` aggregates raw mediation data (SIM activations, active/tolling SIM counts, data, SMS, and voice usage) for a given billing month, transforms it into Geneva‑compatible event records, handles bundle calculations, and prepares RA (reconciliation) records. It also validates commercial codes and inserts reject entries for missing data.

# Key Components
- **MediationFetchService** – main service class.
- `getSimProductViewData(String monthYear, Long secs)` – fetches SIM product view rows from Oracle.
- `getEventsForSIMFile(String monthYear, List<SimProduct> productList, Map<String, Boolean> isNeeded, Long secs)` – builds SIM event records, tracks rejects.
- `getUsageProductViewData(String monthYear, Long secs)` – fetches usage product view rows from Oracle.
- `getEventsForUsageFile(String monthYear, List<UsageProduct> productList, Map<String, Boolean> isNeeded, Long secs)` – builds usage event records, performs bundle splitting, creates RA records.
- `fillCallIDs(List<RARecord> recordList, String callId)` – assigns call IDs to RA records.
- `getInBundleDataRecord(...)` – creates in‑bundle data event.
- `getInBundleSMSVoiceRecord(...)` – creates in‑bundle SMS/voice event.
- `getOutBundleDataRecords(...)` – creates out‑of‑bundle data events from split records.
- `getOutBundleSMSVoiceRecords(...)` – creates out‑of‑bundle SMS/voice events.
- `validateCommercialCode(...)` – cross‑checks SIM/usage data against commercial codes and prepares reject maps.
- `insertRejects(...)` (overloaded) – persists reject entries to Oracle.
- `insertMonth(String monthYear)` – inserts month marker into subscription DB.
- `getStackTrace(Exception e)` – utility for logging.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | `monthYear`, optional `secs` | Compute month start/end dates via `BillingUtils`. | Dates for queries. |
| 2 | `oracleDAO` | `fetchSimProductView` / `fetchUsageProductView`. | Lists of `SimProduct` / `UsageProduct`. |
| 3 | Product lists + `isNeeded` flags | Retrieve activation, active, tolling, addon counts (`subscriptionDAO`), data usage (`dataUsageDAO`), SMS/voice usage (`smsVoiceUsageDAO`). | Maps of counts/usage, populated reject maps. |
| 4 | For each product | Build `EventFile` records, compute bundle allowances, split out‑of‑bundle usage via DAO split methods. | `List<EventFile>` and `List<RARecord>` (added to `allRARecords`). |
| 5 | After processing | Optionally call `insertRejects` to persist missing‑data rejects. | Rows in Oracle reject tables. |
| 6 | Return | `Vector<List<?>>` containing event records and RA records. | Consumed by downstream file‑generation component. |
| 7 | Validation method | Re‑fetches SIM/usage maps, removes present keys, leaves rejects. | Intended for reject insertion (currently commented). |

External services / DBs:
- Oracle (via `OracleDataAccess`).
- Hadoop‑based mediation tables (via `DataUsageDataAccess`, `SMSVoiceUsageDataAccess`, `SubscriptionDataAccess`).
- Email alerting (`MailService`).

# Integrations
- **OracleDataAccess** – provides event type IDs, unique call IDs, and reject‑insert statements.
- **SubscriptionDataAccess** – supplies SIM activation/active/tolling/addon counts and month insertion.
- **DataUsageDataAccess** & **SMSVoiceUsageDataAccess** – read raw usage from Hadoop/HDFS tables, perform CDR subset creation and bundle splitting.
- **BillingUtils** – date utilities and unit conversions.
- **MailService** – sends alert e‑mails when bundle configuration or usage data is inconsistent.
- Downstream: The returned event/RA vectors are consumed by the Geneva file writer component to produce billing files.

# Operational Risks
- **Large in‑memory maps** (`activesMap`, `tollingMap`, usage vectors) may cause OOM for high‑volume months. *Mitigation*: stream processing or paginate DAO calls.
- **Hard‑coded bundle split logic** – any change in product codes or bundle types requires code change. *Mitigation*: externalize bundle rules to configuration.
- **Silent failure of reject insertion** – calls are commented out; missing rejects may go unnoticed. *Mitigation*: enable and monitor reject insertion, add metrics.
- **Email flood** on mis‑configured products (many alerts). *Mitigation*: aggregate alerts per run, throttle mail service.
- **Thread safety** – service holds mutable state (`monthStartDate`, `activesMap`, etc.) without synchronization; concurrent invocations could corrupt data. *Mitigation*: make service stateless or synchronize access.

# Usage
```java
MediationFetchService svc = new MediationFetchService();
String month = "2024-09";
Long secs = null; // process all customers

// 1. Fetch SIM view and build SIM events
List<SimProduct> simProducts = svc.getSimProductViewData(month, secs);
Map<String, Boolean> simNeeded = new HashMap<>();
simNeeded.put(Constants.ACG, true);
simNeeded.put(Constants.SBA, true);
// ... set other flags as required
List<EventFile> simEvents = svc.getEventsForSIMFile(month, simProducts, simNeeded, secs);

// 2. Fetch usage view and build usage events
List<UsageProduct> usageProducts = svc.getUsageProductViewData(month, secs);
Map<String, Boolean> usageNeeded = new HashMap<>();
usageNeeded.put(Constants.DAT, true);
usageNeeded.put(Constants.SMS, true);
Vector<List<?>> usageResult = svc.getEventsForUsageFile(month, usageProducts, usageNeeded, secs);
List<EventFile> usageEvents = (List<EventFile>) usageResult.get(0);
List<RARecord> raRecords   = (List<RARecord>) usageResult.get(1);
```
Debugging: enable `log4j` DEBUG for `com.tcl.move.service.MediationFetchService` to trace map sizes and bundle calculations.

# Configuration
- **Log4j** configuration file (log level, appender) – controls logging output.
- **Constants** class – defines product codes, bundle types, separators, etc.
- No external environment variables are read directly; DB connection details are encapsulated inside DAO implementations (`OracleDataAccess`, `SubscriptionDataAccess`, etc.) which rely on their own configuration files (e.g., `db.properties`).

# Improvements
1. **Stateless Service** – remove mutable fields (`monthStartDate`, `activesMap`, etc.) and pass them as method parameters or encapsulate