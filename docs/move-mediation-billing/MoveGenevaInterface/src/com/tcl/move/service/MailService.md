# Summary
`MailService` provides a utility to send alert e‑mails when errors occur in the Move‑Geneva mediation batch. It builds an SMTP message using system properties for recipients, sender, and host, then transmits the message via JavaMail. Failures are logged and (intended to be) wrapped in a custom `MailException`.

# Key Components
- **class `MailService`**
  - `sendAlertMail(String messageBody) throws MailException` – constructs and sends an alert e‑mail; logs success or error.
  - `private String getStackTrace(Exception exception)` – converts an exception’s stack trace to a `String` for logging.
- **Logger** – `org.apache.log4j.Logger` instance for operational logging.
- **External dependency** – JavaMail API (`javax.mail.*`).

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | `messageBody` (String) | Retrieve SMTP configuration from system properties (`geneva_mail_to`, `geneva_mail_cc`, `geneva_mail_from`, `geneva_mail_host`). | Configuration values used in subsequent steps. |
| 2 | Config values | Create `Properties` with `mail.smtp.host`; obtain `Session`. | `Session` object. |
| 3 | `Session`, `messageBody` | Build `MimeMessage`: set From, To, CC, Subject, Text. | Fully populated e‑mail message. |
| 4 | `MimeMessage` | `Transport.send(message)`. | E‑mail dispatched to SMTP server. |
| 5 | Success | Log info. | No further output. |
| 6 | `MessagingException` | Log error with stack trace via `getStackTrace`. (Current code swallows exception; intended to throw `MailException`). | Error recorded; batch may continue unless caller handles. |

# Integrations
- **GenevaLoader** – Calls `MailService.sendAlertMail` when batch‑level errors occur.
- **System properties** – Supplied at JVM start (e.g., via `-Dgeneva_mail_to=...`).
- **SMTP server** – External mail transport defined by `geneva_mail_host`.
- **Logging framework** – Log4j for audit trails.

# Operational Risks
- **Silent failure**: `MessagingException` is caught but not re‑thrown (commented out), causing callers to assume success. *Mitigation*: Un‑comment and propagate `MailException`.
- **Missing/invalid system properties**: Null or malformed addresses cause `AddressException` or runtime errors. *Mitigation*: Validate properties before use; provide defaults or fail fast.
- **SMTP host unreachable**: Network issues block alert delivery, potentially delaying incident response. *Mitigation*: Implement retry logic and fallback SMTP host.
- **Hard‑coded subject**: Limits flexibility for different alert types. *Mitigation*: Parameterize subject.

# Usage
```java
MailService mailService = new MailService();
try {
    mailService.sendAlertMail("Batch XYZ failed due to XYZ reason.");
} catch (MailException e) {
    // Handle alert‑mail failure (e.g., log, raise alarm)
}
```
*Debug*: Set JVM system properties before execution, enable Log4j DEBUG level, and optionally attach a mock `Transport` for unit testing.

# configuration
- `geneva_mail_to` – Comma‑separated list of primary recipients.
- `geneva_mail_cc` – Comma‑separated list of CC recipients.
- `geneva_mail_from` – Sender e‑mail address.
- `geneva_mail_host` – SMTP server hostname (or IP).
These are read via `System.getProperty` at runtime.

# Improvements
1. **Exception propagation** – Reinstate `throw new MailException(...)` inside the catch block to ensure callers detect failures.
2. **Input validation & defaults** – Add checks for null/empty system properties, validate e‑mail address formats, and provide configurable defaults or fail‑fast behavior.