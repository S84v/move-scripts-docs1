**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\APICallout.java`  

---

## 1. High‑level Summary
`APICallout` is the central orchestration component of the “IndiaMed” API‑access layer. It drives the end‑to‑end processing of EID (equipment‑ID) records that flow through several lifecycle statuses (NEW, SAME‑DAY, SUSPEND, REACTIVE, CHANGE‑PLAN, TERMINATE). For each status it:

1. Retrieves raw EID rows from the staging table via `RawDataAccess`.
2. Determines whether a record has already been processed (record‑count checks).
3. Calls downstream “account‑number”, “custom‑plan” and “product‑attribute” APIs to enrich the record.
4. Persists the enriched data back to the database (account number, custom plan, product attributes, address).
5. Looks up a post‑order “input group id”, fetches the corresponding order payload and pushes it to the downstream order system (`PostOrder`).

The class also contains a large amount of “same‑day” special‑case logic that handles multiple status changes occurring on the same processing day.

---

## 2. Key Classes & Public‑Facing Methods

| Class / Method | Responsibility |
|----------------|----------------|
| **APICallout** (public class) | Orchestrates the whole data‑move pipeline for a run. |
| `void callOut()` | Entry point invoked by the scheduler/driver. Executes the full workflow (new, same‑day, suspend, reactive, change‑plan, terminate) and terminates the JVM (`System.exit`). |
| `void sameDayScenario()` | Handles the “same‑day” processing path for NEW, SUSPEND, REACTIVE, CHANGE‑PLAN and TERMINATE records that arrived on the same calendar day. |
| `void newStatusAPI()` | Processes pure “NEW” records (no same‑day logic). |
| `void statusAPI(StatusEnum status)` | Generic handler for a single status (used for SUSPEND, REACTIVE, TERMINATE). |
| `void suspendMonthly(StatusEnum)` / `void resumeMonthly(StatusEnum)` | Pulls suspend / resume data, decides if a plan‑change is required, and triggers the appropriate post‑order flow. |
| `void changePlanStatusAPI(StatusEnum)` | Handles complex change‑plan scenarios (plan‑type, frequency, commercial↔test). |
| `List<EIDRawData> getRawDataAccess(StatusEnum)` | DAO wrapper – fetches raw rows for a given status. |
| `String getAcntNum(String secsid, String accountType)` | DAO wrapper – obtains the account number for a subscriber. |
| `HashMap<String,String> getCustomPlanDetail(EIDRawData, String)` | Calls `CustomPlanDetails` API to retrieve custom‑plan attributes. |
| `HashMap<String,Object> getProductDetails(EIDRawData, String)` | Calls `ProductDetails` API; includes an in‑memory cache (`outerHashMap`). |
| `List<PostOrderDetail> getPostOrderAttr(String)` | DAO wrapper – fetches the post‑order payload for a given input‑group id. |
| `void getpostOrderDetails(List<PostOrderDetail>)` | Sends the post‑order payload to `PostOrder.postOrderDetails`. |
| Helper methods (`newStatusChangeAPI`, `newStatusSuspendAPISameDay`, …) | Perform the per‑record enrichment and DB writes for the various status branches. |
| `static String getStackTrace(Exception)` | Utility to convert an exception stack trace to a string for logging. |

*Note:* All “private” helper methods are invoked only from within this class; they are listed because they encapsulate distinct functional steps.

---

## 3. Inputs, Outputs & Side‑Effects

| Aspect | Details |
|--------|---------|
| **Primary Input** | Raw EID rows stored in the staging table (`RAW_EID_TABLE` – name defined inside `RawDataAccess`). The rows are filtered by `StatusEnum` (NEW, SAME‑DAY, SUSPEND, REACTIVE, CHANGE‑PLAN, TERMINATE). |
| **External Services / APIs** | <ul><li>`AccountNumDetails.getAccountNumDetails(EIDRawData)` – returns an account number.</li><li>`CustomPlanDetails.getCustomPlanDetails(EIDRawData, String)` – returns custom‑plan key/value pairs.</li><li>`ProductDetails.getProductDetails(EIDRawData, String)` – returns product‑attribute map.</li><li>`PostOrder.postOrderDetails(List<PostOrderDetail>)` – pushes order payload to downstream order system (likely a REST or MQ endpoint).</li></ul> |
| **Database Access** | All reads/writes go through `RawDataAccess` (DAO). Operations include:<br>• `fetchEIDRawData(StatusEnum)` – read raw rows.<br>• `newRecCountCheck*` – existence checks in target tables.<br>• `insertAccountNumber`, `insertChangeAccountNumber` – write account numbers.<br>• `getUpdatedCustomPlan`, `getUpdatedProductattr`, `getUpdatedAttrString`, `getUpdateAddress` – update enriched data.<br>• `getSequenceNumber*`, `getInputGroupId` – retrieve workflow sequencing information.<br>• `getPostOrderAttr` – fetch post‑order definition. |
| **Outputs** | <ul><li>Enriched rows persisted back to the data warehouse (account number, custom plan, product attributes, address).</li><li>Post‑order payload sent to the order‑management system.</li></ul> |
| **Side‑Effects** | • Calls `System.exit(0/1)` on normal completion or any uncaught exception – terminates the JVM (used by the batch scheduler).<br>• Populates an in‑memory cache (`outerHashMap`) that lives for the duration of the run. |
| **Assumptions** | • All DAO methods succeed or throw checked exceptions that are caught and cause an immediate exit.<br>• The status values in the raw table exactly match the constants used (`APIConstants.SUSPEND`, `"Reactive"`, etc.).<br>• Account‑type mapping logic (Monthly vs Suspend) is exhaustive for the supported frequencies.<br>• The external API classes (`AccountNumDetails`, `CustomPlanDetails`, `ProductDetails`) are thread‑safe (the code creates a new `APICallout` per call, but also re‑uses the same helper instances). |

---

## 4. Integration Points with Other Scripts / Components

| Component | Connection Detail |
|-----------|-------------------|
| **`RawDataAccess` DAO** | Provides all DB reads/writes. Configured via `APIAccessDAO.properties` (JDBC URL, user, password). |
| **`AccountNumDetails`, `CustomPlanDetails`, `ProductDetails`** | External API wrappers; their endpoint URLs, credentials, and time‑outs are read from `APIAcessMgmt.properties` (or environment variables). |
| **`PostOrder`** | Sends the final order payload; likely uses a REST client or MQ producer configured in the same property files. |
| **`OrderStatus` & `BulkInsert`** | Invoked after post‑order processing to update order status and bulk‑load data; these classes are part of the same `move-indiamed-api` module. |
| **`log4j.properties`** | Controls logging levels, appenders (file, console). |
| **Scheduler / Driver** | Not present in this file but a wrapper script (e.g., a shell or Oozie job) launches the Java class (e.g., `java -cp … com.tcl.api.callout.APICallout`). The driver expects the process to exit with 0 on success. |
| **Hive DDL scripts** (`traffic_*` files) | Not directly called here, but the enriched data written by `RawDataAccess` ends up in Hive external tables defined by those DDLs for downstream analytics. |
| **Environment Variables** | May provide DB credentials, API keys, or runtime flags (e.g., `ENV=PROD`). The code itself does not read them directly, but the DAO and API wrappers typically do. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Abrupt `System.exit`** on any exception | Entire batch job terminates, potentially leaving partially processed data and no retry. | Replace `System.exit` with a controlled exception propagation; let the scheduler capture the exit code. Implement a retry/compensation framework. |
| **Recursive instantiation (`new APICallout()` inside methods)** | Unnecessary object creation, risk of stack overflow if future changes add deeper recursion. | Refactor to use a single instance (`this`) or extract reusable services into separate beans. |
| **In‑memory cache (`outerHashMap`) grows without bound** | OOM on large daily volumes. | Add size limits or LRU eviction; or move cache to an external store (e.g., Redis) if needed. |
| **Hard‑coded status strings (`"Reactive"`, `"Suspend"` etc.)** | Typos or new status values break logic silently. | Centralize status constants (use `StatusEnum` everywhere). |
| **No transaction management** – multiple DB writes per record are not atomic. | Inconsistent state if a failure occurs after some writes. | Wrap per‑record operations in a DB transaction (commit/rollback) via DAO. |
| **Logging at INFO for every record** – high volume can fill logs quickly. | Disk pressure, difficulty finding real errors. | Reduce to DEBUG for per‑record logs; keep only summary INFO. |
| **Blocking `Thread.sleep(1000)` in `newStatusAPI`** | Slows throughput, unnecessary pause. | Remove sleep; rely on DB/queue back‑pressure. |
| **Missing null checks on DAO returns** (e.g., `acntNum` may be null) | NullPointerException later in the flow. | Validate all DAO results; fail fast with clear error messages. |

---

## 6. Typical Execution / Debug Workflow

1. **Preparation**  
   - Ensure `log4j.properties` and `APIAccessDAO.properties` are on the classpath.  
   - Verify DB connectivity and that the staging tables contain data for the target date.  
   - Confirm API endpoint URLs and credentials are correct (often via environment variables or the `APIAcessMgmt.properties` file).  

2. **Run** (usually from a scheduler)  
   ```bash
   java -cp lib/*:conf/ com.tcl.api.callout.APICallout
   ```
   - The `main` method is not defined; the driver script typically does `new APICallout().callOut();`.  
   - The process writes logs to the location defined in `log4j.properties`.  

3. **Debugging**  
   - Increase log level to `DEBUG` in `log4j.properties` and re‑run.  
   - Attach a remote debugger (e.g., `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005`) and set breakpoints in `callOut()` or any of the status‑specific methods.  
   - If the job exits with code 1, inspect the stack‑trace printed in the log – the utility `getStackTrace` captures the full exception.  

4. **Post‑run validation**  
   - Query the target tables (e.g., `mnaas.account_number`, `mnaas.custom_plan`) to confirm rows were inserted/updated.  
   - Verify that the order system received the expected payload (check MQ/REST logs).  

---

## 7. External Configuration & Property Files

| File | Purpose | Key Parameters (illustrative) |
|------|---------|------------------------------|
| `log4j.properties` | Logging configuration (appenders, levels, file locations). |
| `APIAccessDAO.properties` | JDBC URL, driver class, DB user/password, connection pool settings for `RawDataAccess`. |
| `APIAcessMgmt.properties` (in `PropertyFiles`) | API endpoint URLs, authentication tokens, timeout values for `AccountNumDetails`, `CustomPlanDetails`, `ProductDetails`. |
| Environment variables (e.g., `JAVA_OPTS`, `ENV`) | May be used by the DAO or API wrappers to select production vs. test endpoints. |

The class itself does **not** read these files directly; the dependent DAO and API classes do.

---

## 8. Suggested Improvements (TODO)

1. **Replace all `System.exit` calls with proper exception handling** – let the scheduler decide the exit code and enable graceful shutdown, retries, and resource cleanup.  
2. **Refactor the class into smaller service components** (e.g., `NewStatusProcessor`, `SameDayProcessor`, `ChangePlanProcessor`) and inject them via a lightweight DI container (Spring, Guice). This will improve testability, reduce the need for recursive `new APICallout()` calls, and make the codebase easier to maintain.  

---