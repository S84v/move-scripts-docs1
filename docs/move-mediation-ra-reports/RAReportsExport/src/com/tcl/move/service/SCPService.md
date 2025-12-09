# Summary
`SCPService` is a production‑grade utility that transfers generated report files from the Move‑Mediation batch process to a remote data lake via SCP over SSH. It establishes an SSH session using JSch, iterates over a list of files, and streams each file to the configured remote directory. Errors are logged and re‑thrown as generic `Exception` to be handled by the caller.

# Key Components
- **class `SCPService`**
  - `sendReport(String fileType, List<File> files)`: orchestrates SSH connection, selects destination path based on `fileType` (`D` for daily, otherwise monthly), and invokes `copyFileToServer` for each file.
  - `copyFileToServer(Session session, File sourceFile, String destinationPath)`: implements the SCP protocol (timestamp, permissions, file size, content) over the provided JSch `Session`.
  - `static int checkAck(InputStream in)`: parses SCP acknowledgment codes (0 success, 1 error, 2 fatal, -1 EOF) and prints error messages.
  - `private String getStackTrace(Exception e)`: converts an exception stack trace to a `String` for logging.

# Data Flow
- **Inputs**
  - `fileType` (String) – determines remote sub‑directory.
  - `files` (List\<File>) – local report files to be transferred.
  - System properties: `Datalake_host`, `Datalake_port`, `Datalake_user`, `Datalake_password`, `Datalake_daily_path`, `Datalake_monthly_path`.
- **Outputs**
  - Remote files placed at `<destinationPath>/<filename>` on the data lake server.
  - Log entries (INFO/ERROR) via Log4j.
- **Side Effects**
  - Opens TCP connection to remote host.
  - Authenticates with password.
  - Writes file bytes to remote filesystem.
- **External Services**
  - SSH/SCP server (data lake endpoint).
  - JSch library for SSH handling.
  - Log4j for logging.

# Integrations
- Invoked by higher‑level services (e.g., `ExportService` or batch orchestrators) after report generation.
- Relies on system properties set by deployment scripts or environment configuration.
- Errors propagate as generic `Exception` to be caught by the calling batch job, which may trigger `MailService` alerts.

# Operational Risks
- **Hard‑coded password in system property** – risk of credential leakage. *Mitigation*: use key‑based authentication or secure vault.
- **`System.exit(0)` on SCP ack failure** – terminates JVM, potentially aborting unrelated jobs. *Mitigation*: replace with exception throwing.
- **No retry logic** – transient network failures cause immediate job failure. *Mitigation*: implement exponential back‑off and retry.
- **Plain‑text logging of sensitive info** – stack traces may expose credentials. *Mitigation*: sanitize logs.

# Usage
```java
// Example from a unit test or batch job
List<File> reports = Arrays.asList(new File("/tmp/daily_report.csv"));
SCPService scp = new SCPService();
try {
    scp.sendReport("D", reports);
} catch (Exception e) {
    // handle failure, e.g., send alert via MailService
}
```
To debug, enable Log4j DEBUG level for `com.tcl.move.service.SCPService` and verify system properties are correctly set.

# Configuration
| Property                | Description                                 |
|-------------------------|---------------------------------------------|
| `Datalake_host`          | Remote SSH host (e.g., `datalake.example.com`) |
| `Datalake_port`          | SSH port (default 22)                      |
| `Datalake_user`         | SSH username                               |
| `Datalake_password`     | SSH password (or key passphrase)          |
| `Datalake_daily_path`   | Remote directory for daily reports         |
| `Datalake_monthly_path` | Remote directory for monthly reports       |

Properties are read via `System.getProperty()`; they must be supplied at JVM startup (`-D` flags) or via a wrapper script.

# Improvements
1. **Replace `System.exit` calls with custom `SCPException`** to avoid JVM termination and allow graceful error handling.
2. **Add configurable authentication method** (public‑key support) and externalize credentials to a secure secret manager.