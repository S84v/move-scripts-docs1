# Summary
`MailService` provides a utility to send alert e‑mails when the MOVE‑Mediation‑KLM batch encounters errors. It builds an SMTP `MimeMessage` using system properties for recipients, sender, and host, then dispatches the message via JavaMail. Failures are logged and (intended to be) wrapped in a custom `MailException`.

# Key Components
- **Class `MailService`**
  - `private static Logger logger` – Log4j logger for operational messages.
  - `public void sendAlertMail(String messageBody) throws MailException` – Constructs and sends an alert e‑mail; logs success or error.
  - `private String getStackTrace(Exception exception)` – Converts an exception’s stack trace to a `String` for logging.

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | System properties: `geneva_mail_to`, `geneva_mail_cc`, `geneva_mail_from`, `geneva_mail_host` | Populate JavaMail `Properties` and create `Session` | Configured mail session |
| 2 | `messageBody` (String) | Build `MimeMessage` (set From, To, CC, Subject, Text) | Prepared e‑mail object |
| 3 | `Transport.send(message)` | Transmit e‑mail via SMTP host | Remote mail delivery (side effect) |
| 4 | `MessagingException` (if thrown) | Log error with full stack trace via `getStackTrace` | Log entry; intended to throw `MailException` (currently commented) |

# Integrations
- **ReportGenerator** – Calls `MailService.sendAlertMail` when batch processing fails.
- **System environment** – Relies on JVM system properties for mail configuration.
- **JavaMail API** – Uses `javax.mail` classes for SMTP communication.
- **Log4j** – Centralized logging for operational visibility.

# Operational Risks
- **Missing/invalid system properties** → SMTP connection failure; mitigated by validating properties before use.
- **SMTP host unreachable** → `MessagingException` logged but not propagated; may hide critical failure. Consider re‑throwing `MailException`.
- **Hard‑coded subject** limits reuse; externalize to configuration if subject varies.
- **No authentication support** – If SMTP requires credentials, current implementation will fail; add support for `mail.smtp.auth` and credentials.

# Usage
```java
MailService mailService = new MailService();
String body = "Batch XYZ failed at step 3 due to NullPointerException.";
try {
    mailService.sendAlertMail(body);
} catch (MailException e) {
    // handle escalation or retry
}
```
To debug, set Log4j level to `DEBUG` and ensure required system properties are defined (`-Dgeneva_mail_to=...` etc.) before JVM start.

# Configuration
- **System properties (set via JVM args or environment variables)**
  - `geneva_mail_to` – Comma‑separated list of primary recipients.
  - `geneva_mail_cc` – Comma‑separated list of CC recipients.
  - `geneva_mail_from` – Sender e‑mail address.
  - `geneva_mail_host` – SMTP host name or IP.
- No external config files referenced directly.

# Improvements
1. **Validate configuration** – Add a method to verify that all required system properties are present and syntactically valid; throw `MailException` early if not.
2. **Propagate failures** – Uncomment and refine the `throw new MailException(...)` line; optionally include the original `MessagingException` as the cause to enable upstream retry or alert escalation.