# Summary
`MailService` provides production‑grade email capabilities for the Move‑Mediation RA Reports export process. It sends alert notifications on failures and distributes generated report files to configured recipients via SMTP.

# Key Components
- **class `MailService`**
  - `sendAlertMail(String messageBody) throws Exception` – builds a simple text email using system properties (`error_mail_to`, `ra_mail_from`, `mail_host`) and sends it via `Transport.send`. Logs success or throws a generic `Exception` on `MessagingException`.
  - `sendReport(String subject, List<File> files) throws Exception` – composes a multipart MIME message with a static body, attaches each file in `files`, and sends to `ra_mail_to`/`ra_mail_cc`. Uses same SMTP host property. Logs activity and wraps any `MessagingException` or generic `Exception` in a new `Exception`.
  - `private String getStackTrace(Exception exception)` – utility to convert an exception stack trace to a `String` for logging.

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | `messageBody` (alert) or `subject` + `List<File>` (report) | Retrieve SMTP configuration from system properties; create `Session`; build `MimeMessage`; attach files (report) | Email sent via SMTP server (`mail_host`). |
| 2 | Exceptions during message creation/sending | Capture stack trace, log via Log4j, re‑throw as generic `Exception`. | Caller receives exception; possible alert escalation. |
| 3 | Logging | `logger.info` / `logger.error` writes to configured Log4j appenders. | Audit trail in log files. |

External services:
- SMTP server identified by `mail_host`.
- No direct DB, queue, or file system writes beyond reading attachment files.

# Integrations
- Invoked by `RAReportExport` (batch entry point) when report generation succeeds or fails.
- Relies on system properties set by the batch launcher or wrapper scripts (`error_mail_to`, `ra_mail_to`, `ra_mail_cc`, `ra_mail_from`, `mail_host`).
- Attachments are produced by `ExportService`/`ExcelExportDAO` and passed as `File` objects.

# Operational Risks
- **Hard‑coded system properties**: missing or malformed values cause `MessagingException`. *Mitigation*: validate properties at startup; provide defaults or fail fast.
- **Generic `Exception` propagation**: loses original exception type, complicates error handling. *Mitigation*: throw custom `MailException` preserving cause.
- **No authentication/TLS**: SMTP host may require credentials; current code assumes open relay. *Mitigation*: extend to support `mail.smtp.auth`, `mail.smtp.starttls.enable`, and credential properties.
- **Large attachment handling**: loading entire files into memory via `FileDataSource` may OOM for huge reports. *Mitigation*: stream attachments or enforce size limits.

# Usage
```java
MailService mailService = new MailService();

// Alert example
try {
    mailService.sendAlertMail("Export failed due to XYZ");
} catch (Exception e) {
    // handle or log
}

// Report example
List<File> reports = Arrays.asList(new File("/tmp/daily.csv"));
try {
    mailService.sendReport("Daily RA Report", reports);
} catch (Exception e) {
    // handle or log
}
```
Run within the batch process after setting required system properties, e.g.:

```bash
java -Dmail_host=smtp.example.com \
     -Dra_mail_from=ra@tcl.com \
     -Dra_mail_to=user1@tcl.com,user2@tcl.com \
     -Dra_mail_cc=ops@tcl.com \
     -Derror_mail_to=admin@tcl.com \
     -cp yourclasspath com.tcl.move.main.RAReportExport ...
```

# Configuration
- **System properties (set via `-D` or environment wrapper)**
  - `mail_host` – SMTP server hostname.
  - `ra_mail_from` – Sender address for report mails.
  - `ra_mail_to` – Comma‑separated recipient list for reports.
  - `ra_mail_cc` – Comma‑separated CC list for reports.
  - `error_mail_to` – Recipient for alert mails.
- No external config files referenced directly.

# Improvements
1. **Replace generic `Exception` with `MailException`** – preserve original cause, improve caller handling.
2. **Add SMTP authentication and TLS support** – read `mail.smtp.user`, `mail.smtp.password`, enable `mail.smtp.auth` and `mail.smtp.starttls.enable` for secure production environments.