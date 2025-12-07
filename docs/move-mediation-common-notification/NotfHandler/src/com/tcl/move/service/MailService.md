# Summary
`MailService` provides a utility to send alert e‑mails when the notification consumer encounters errors. It builds a JavaMail `MimeMessage` using SMTP host and address parameters supplied via system properties, sends the message synchronously, logs success, and throws a custom `MailException` on failure.

# Key Components
- **Class `MailService`**
  - `sendAlertMail(String messageBody)`: constructs and sends an alert e‑mail; logs outcome; propagates `MailException` on error.
  - `getStackTrace(Exception e)`: converts an exception’s stack trace to a `String` for logging and inclusion in the e‑mail body.
- **Logger `logger`**: Log4j logger for informational and error messages.
- **Custom exception `MailException`**: Wraps mailing failures.

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | `messageBody` (String) | Retrieve SMTP configuration from system properties (`notif_mail_to`, `notif_mail_cc`, `notif_mail_from`, `notif_mail_host`). | Configuration values used for mail composition. |
| 2 | Config values | Create `Properties` → set `mail.smtp.host`. Obtain default `Session`. | `Session` object. |
| 3 | `Session` + `messageBody` | Build `MimeMessage`: set From, To, CC, Subject, Text. | Fully populated e‑mail object. |
| 4 | `MimeMessage` | `Transport.send(message)`. | Email dispatched to SMTP server. |
| 5 | Success | Log info `"Message sent successfully ..."`. | No further output. |
| 6 | Failure (`MessagingException`) | Capture stack trace via `getStackTrace`, log error, throw `MailException`. | Exception propagated to caller. |

External services: SMTP server identified by `notif_mail_host`.

# Integrations
- **Notification Consumer**: Invokes `MailService.sendAlertMail` when processing errors occur.
- **System Property Provider**: Values are injected at JVM start (e.g., via `-Dnotif_mail_to=...`).
- **Log4j**: Centralized logging framework used across the MOVE mediation suite.
- **Custom exception handling**: `MailException` is part of the MOVE exception hierarchy, enabling upstream error handling.

# Operational Risks
- **Missing/incorrect system properties** → mail send fails; mitigated by validating properties at startup.
- **SMTP host unavailability** → messages dropped; mitigated by monitoring SMTP health and implementing retry/back‑off.
- **Blocking send**: `Transport.send` is synchronous; could delay processing threads. Mitigate by off‑loading to an async executor or using a non‑blocking mail library.
- **Sensitive data exposure**: E‑mail body may contain raw error details; ensure logs and e‑mail recipients are authorized.

# Usage
```java
// Example in a unit test or debugging session
MailService mailService = new MailService();
String alert = "Notification Consumer failed at record XYZ.";
try {
    mailService.sendAlertMail(alert);
} catch (MailException e) {
    // handle or assert failure in test
}
```
Run with required system properties:
```
java -Dnotif_mail_to=ops@example.com \
     -Dnotif_mail_cc=dev@example.com \
     -Dnotif_mail_from=no-reply@example.com \
     -Dnotif_mail_host=smtp.example.com \
     -cp yourapp.jar com.tcl.move.service.MailService
```

# Configuration
- `notif_mail_to` – Comma‑separated list of primary recipients.
- `notif_mail_cc` – Comma‑separated list of CC recipients.
- `notif_mail_from` – Sender e‑mail address.
- `notif_mail_host` – SMTP server hostname (no authentication handling in current code).

All values are read from Java system properties at runtime.

# Improvements
1. **Add property validation**: Verify that all required system properties are present and non‑empty during bean initialization; throw a clear configuration exception early.
2. **Introduce asynchronous sending with retry**: Use a thread pool or non‑blocking mail client (e.g., Jakarta Mail with `Transport.sendMessage` in a separate task) and implement exponential back‑off retries for transient SMTP failures.