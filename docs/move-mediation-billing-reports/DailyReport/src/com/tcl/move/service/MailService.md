# Summary  
`MailService` is a utility class used in the Move‑Mediation‑Billing‑Reports batch job to send (1) alert e‑mails when processing errors occur and (2) the generated daily billing report as an attachment to a configured distribution list. It builds simple SMTP messages using JavaMail, logs activity via Log4j, and propagates any failure as a generic `Exception`.

# Key Components  

- **class `MailService`** – stateless service exposing two public methods:  
  - `void sendAlertMail(String messageBody) throws Exception` – sends a plain‑text alert to the address defined by `error_mail_to`.  
  - `void sendReport(String fileName) throws Exception` – sends the report file (located under `excel_path`) as a MIME attachment to the addresses defined by `export_mail_to`/`export_mail_cc`.  
- **private helper `String getStackTrace(Exception e)`** – converts an exception stack trace to a `String` for logging.  
- **Log4j logger** – records success/failure events.  

# Data Flow  

| Method | Input | Processing | Output / Side‑Effect |
|--------|-------|------------|----------------------|
| `sendAlertMail` | `messageBody` (String) | Build `MimeMessage` with `from`, `to`, `subject`, `text`. Uses system properties for SMTP host and addresses. Calls `Transport.send`. | Email sent to `error_mail_to`. Logs info or error. Throws `Exception` on failure. |
| `sendReport` | `fileName` (String) | Resolve full path via `excel_path`. Create multipart message: text part + file attachment part. Uses system properties for SMTP host, sender, recipients, CC. Calls `Transport.send`. | Email with attachment sent to `export_mail_to`/`export_mail_cc`. Logs info or error. Throws `Exception` on failure. |
| `getStackTrace` | `Exception` | Capture stack trace via `PrintWriter`. | Returns stack‑trace string (used only for logging). |

External services: SMTP server identified by `mail_host`. No database, queue, or file‑system writes beyond reading the attachment file.

# Integrations  

- **`BillingReportExport` (main batch program)** – invokes `MailService.sendAlertMail` on uncaught processing errors and `MailService.sendReport` after successful Excel/CSV generation.  
- **`ExcelExport`** – produces the file whose name is passed to `sendReport`.  
- **System properties** – populated at application start (e.g., via `-D` JVM arguments or a properties loader) and shared across the batch job.  

# Operational Risks  

1. **SMTP mis‑configuration** – missing/incorrect `mail_host`, `export_mail_from`, or recipient properties cause message loss. *Mitigation*: Validate required properties at startup; implement fallback to a default SMTP host.  
2. **Attachment file not found** – `excel_path` + `fileName` may point to a non‑existent file, resulting in `FileNotFoundException` wrapped as `MessagingException`. *Mitigation*: Verify file existence before constructing `FileDataSource`; fail fast with a clear log entry.  
3. **Unbounded exception propagation** – methods re‑throw generic `Exception`, potentially masking root causes. *Mitigation*: Define a custom `MailException` hierarchy and preserve original exception as cause.  
4. **No authentication / TLS** – uses plain SMTP without credentials; insecure in production. *Mitigation*: Extend to support `mail.smtp.auth`, `mail.smtp.starttls.enable`, and credential properties.  

# Usage  

```java
public static void main(String[] args) throws Exception {
    // Ensure required system properties are set
    System.setProperty("mail_host", "smtp.example.com");
    System.setProperty("export_mail_from", "noreply@example.com");
    System.setProperty("export_mail_to", "ops@example.com");
    System.setProperty("export_mail_cc", "team@example.com");
    System.setProperty("excel_path", "/opt/reports/");

    MailService mail = new MailService();

    // Example: send an alert
    mail.sendAlertMail("Daily export failed due to XYZ");

    // Example: send a generated report
    mail.sendReport("DailyReport_20251205.xlsx");
}
```

For debugging, enable Log4j DEBUG level for `com.tcl.move.service.MailService` to view SMTP session details.

# Configuration  

| Property | Description | Example |
|----------|-------------|---------|
| `mail_host` | SMTP server hostname (no port handling in code) | `smtp.mail.company.com` |
| `export_mail_from` | Sender e‑mail address used for both alert and report mails | `billing@company.com` |
| `error_mail_to` | Recipient address for alert mails | `ops-alerts@company.com` |
| `export_mail_to` | Primary recipients for the daily report | `billing-team@company.com` |
| `export_mail_cc` | CC list for the daily report (comma‑separated) | `manager@company.com,lead@company.com` |
| `excel_path` | Directory path where generated Excel/CSV files are stored | `/data/reports/` |

These are read via `System.getProperty` at runtime; they must be supplied as JVM arguments or loaded into the system properties before invoking the service.

# Improvements  

1. **Introduce a typed `MailException`** – replace generic `Exception` with a custom unchecked exception that preserves the original cause and provides error codes (e.g., `SMTP_CONNECTION_FAILURE`, `ATTACHMENT_NOT_FOUND`).  
2. **Add SMTP authentication and TLS support** – read `mail.smtp.user`, `mail.smtp.password`, and enable `mail.smtp.starttls.enable` to meet security compliance for production environments.  