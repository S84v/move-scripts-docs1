# Summary
`MailService` provides a utility to send alert e‑mails when the Balance Notification batch encounters errors. It builds a JavaMail `MimeMessage` using system properties for SMTP configuration, sends it via `Transport.send`, logs success, and wraps any `MessagingException` in a custom checked `MailException` for upstream handling.

# Key Components
- **class `MailService`**
  - `private static Logger logger` – Log4j logger for operational messages.
  - `public void sendAlertMail(String messageBody) throws MailException` – Core method that:
    1. Reads SMTP and recipient settings from system properties.
    2. Creates a `Session` with the host property.
    3. Constructs a `MimeMessage` (from, TO, CC, subject, body).
    4. Sends the message via `Transport.send`.
    5. Logs success or, on failure, logs the stack trace and throws `MailException`.
  - `private String getStackTrace(Exception exception)` – Helper that converts an exception’s stack trace to a `String`.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | `messageBody` (String) | Read system properties: `notif_mail_to`, `notif_mail_cc`, `notif_mail_from`, `notif_mail_host` | Configuration values |
| 2 | System properties | Build `Properties` object, set `mail.smtp.host` | JavaMail `Session` |
| 3 | `Session` + config values | Create `MimeMessage`, set From, TO, CC, Subject, Text | Prepared e‑mail |
| 4 | `MimeMessage` | `Transport.send(message)` | E‑mail delivered to SMTP server |
| 5 | Success | `logger.info` | Audit log entry |
| 6 | `MessagingException` | Capture stack trace, log error, throw `MailException` | Propagation of failure to caller |

External services: SMTP server identified by `notif_mail_host`.

# Integrations
- **`BalanceNotificationHandler`** – Calls `MailService.sendAlertMail` when processing errors occur.
- **`MailException`** – Custom checked exception used by upstream components to trigger retry, alerting, or compensation logic.
- **System properties** – Populated at application start (e.g., via command‑line `-D` flags or property files loaded by `BalanceNotificationHandler`).

# Operational Risks
- **Missing/invalid SMTP configuration** – Mail send fails, leading to unhandled `MailException`. *Mitigation*: Validate required system properties at startup; fallback to default host.
- **Blocking on SMTP send** – `Transport.send` is synchronous; network latency can stall the batch. *Mitigation*: Configure SMTP with reasonable timeouts; consider async send or thread pool.
- **Unbounded stack‑trace logging** – Large exception traces may fill log storage. *Mitigation*: Truncate or rotate logs; limit stack‑trace size.
- **Hard‑coded subject** – No context about the specific failure. *Mitigation*: Parameterize subject or include error codes.

# Usage
```java
public static void main(String[] args) {
    MailService mailService = new MailService();
    try {
        mailService.sendAlertMail("Balance update failed for subscriber 12345");
    } catch (MailException e) {
        // handle according to batch error policy
    }
}
```
Debugging: Run with JVM options `-Dnotif_mail_to=... -Dnotif_mail_cc=... -Dnotif_mail_from=... -Dnotif_mail_host=...` and attach a debugger to step through `sendAlertMail`.

# configuration
- **System properties (required)**
  - `notif_mail_to` – Comma‑separated recipient list.
  - `notif_mail_cc` – Comma‑separated CC list (optional but expected).
  - `notif_mail_from` – Sender e‑mail address.
  - `notif_mail_host` – SMTP host name or IP.
- Typically supplied via `-D` flags or a properties file loaded by the main application.

# Improvements
1. **Add configuration validation** – Implement a static initializer that checks presence and basic format of all required system properties; fail fast with clear log messages.
2. **Introduce asynchronous mail dispatch** – Use a bounded executor service to send e‑mails off the main processing thread, with configurable timeout and retry policy to reduce batch latency.