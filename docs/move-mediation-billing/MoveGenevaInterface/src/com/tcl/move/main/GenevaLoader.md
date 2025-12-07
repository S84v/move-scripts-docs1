# Summary
`GenevaLoader` is the entry‑point batch driver for the Move‑Geneva mediation pipeline. It loads configuration, inserts the processing month, extracts SIM and usage charge data from the database via `MediationFetchService`, transforms the data into Geneva‑compatible event files using `GenevaFileService`, and optionally validates commercial codes. Errors trigger alert e‑mails via `MailService` and cause the batch to terminate.

# Key Components
- **class `GenevaLoader`**
  - `main(String[] args)`: orchestrates batch execution; reads properties, sets up logger, invokes processing methods.
  - `insertMonth(String monthYear)`: persists the month identifier; on `DatabaseException` sends alert mail.
  - `loadSIMChargesFile(String monthYear, Long secs)`: retrieves SIM product data, builds a map of required charge types, obtains event records, and calls `genevaFileService.loadGenevaFiles`.
  - `loadUsageChargesFile(String monthYear, Long secs) throws Exception`: retrieves usage product data, determines needed charge flags, obtains event and RA records, and invokes `genevaFileService.loadGenevaFiles`.
  - `validateCommercialCode(String monthYear)`: fetches SIM/usage products and validates commercial codes against Geneva data.
  - `getStackTrace(Exception)`: utility to convert stack trace to `String`.

- **External services**
  - `MediationFetchService`: DAO layer for DB reads (`insertMonth`, `getSimProductViewData`, `getUsageProductViewData`, `getEventsForSIMFile`, `getEventsForUsageFile`, `validateCommercialCode`).
  - `GenevaFileService`: writes Geneva‑format files.
  - `MailService`: sends alert e‑mails (`sendAlertMail`).

- **Exception types**
  - `DatabaseException`, `MailException` – custom checked exceptions used for error handling.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | Command‑line args or hard‑coded defaults (monthYear, propertyFile, logFile, valFlag, customer) | Load properties file into `System` properties | Environment configuration |
| 2 | `monthYear` | `mediationFetchService.insertMonth` | DB insert (month record) |
| 3a | `monthYear`, optional `secs` | `mediationFetchService.getSimProductViewData` → list of `SimProduct` | In‑memory product list |
| 3b | Same | Build `isNeeded` map, call `mediationFetchService.getEventsForSIMFile` | List\<EventFile\> |
| 3c | Event list | `genevaFileService.loadGenevaFiles` | Geneva SIM charge files written to filesystem |
| 4a | `monthYear`, optional `secs` | `mediationFetchService.getUsageProductViewData` → list of `UsageProduct` | In‑memory product list |
| 4b | Same | Build `isNeeded` map, call `mediationFetchService.getEventsForUsageFile` | List\<EventFile\>, List\<RARecord\> |
| 4c | Event & RA lists | `genevaFileService.loadGenevaFiles` | Geneva usage files written |
| 5 | `monthYear` | `mediationFetchService.validateCommercialCode` | Validation logs / DB updates |
| Error paths | Any `DatabaseException` | Log, compose alert mail, `mailService.sendAlertMail`, `System.exit(1)` | Alert e‑mail, batch termination |
| Generic exception | Any other | Log stack trace, `System.exit(1)` | Batch termination |

# Integrations
- **Database**: accessed via `MediationFetchService` (SQL queries against Move‑Geneva schema).
- **File System**: reads property file; writes Geneva output files via `GenevaFileService`.
- **Logging**: Apache Log4j, log file path supplied via `logfile.name` system property.
- **Email**: `MailService` uses SMTP configuration from properties to send alerts.
- **Constants**: `com.tcl.move.constants.Constants` provides charge code literals.
- **Utility**: `BillingUtils.getFirstDayofTheMonth` for usage file naming.

# Operational Risks
- **Hard‑coded paths**: Production runs may inadvertently use development workspace; mitigate by enforcing command‑line args or environment variables.
- **System.exit calls**: Immediate JVM termination can bypass cleanup hooks; replace with controlled shutdown handling.
- **Unchecked `Exception` in `loadUsageChargesFile` signature**: Propagates to caller but not caught in `main`; could cause silent failure; ensure all exceptions are handled.
- **Single‑threaded execution**: No parallelism; large data volumes may exceed runtime windows; consider batch chunking or multithreading.
- **Mail service failure**: If alert mail cannot be sent, batch still exits; add fallback logging and retry logic.

# Usage
```bash
# Production (args expected)
java -cp move-mediation-billing.jar com.tcl.move.main.GenevaLoader \
    2020-09 /opt/move/config/MNAAS_ShellScript.properties /var/log/move/Geneva.log N 0

# Debug (uses hard‑coded defaults)
java -cp move-mediation-billing.jar com.tcl.move.main.GenevaLoader
```
- Set `logfile.name` system property if custom log location is required.
- Verify that the properties file contains DB, SMTP, and file‑output configurations.

# configuration
- **Properties file** (`MNAAS_ShellScript.properties`): loaded at runtime; keys are copied to `System` properties (e.g., DB URL, user, password, SMTP host, output directories).
- **System properties**: `logfile.name` (log file path) and any keys defined in the properties file.
- **Environment**: Java 8+, Log4j configuration (log4j.properties) on classpath.

# Improvements
1. **Parameterize all inputs** – replace hard‑coded workspace paths with command‑line arguments or environment variables; add validation of required args.
2. **Refactor error handling** – centralize exception-to‑mail mapping, eliminate `System.exit` in favor of a graceful shutdown hook, and implement retry logic for transient DB/mail failures.