# Summary
`MailService` provides a single responsibility: sending alert e‑mails when the Move‑Mediation add‑on notification service encounters errors. It builds a JavaMail `MimeMessage` using system properties for SMTP configuration, sends it via `Transport.send`, logs success, and wraps any `MessagingException` in a custom checked `MailException`.

# Key Components
- **class `MailService`**
  - `private static Logger logger` – Log4j logger for operational messages.
  - `public void sendAlertMail(String messageBody) throws MailException` – Core method that:
    1. Reads SMTP and recipient configuration from system properties.
    2. Creates a `Session` with the host property.
    3. Constructs a `MimeMessage` (from, TO, CC, subject, body).
    4. Sends the message via `Transport.send`.
    5. Logs success or, on failure, logs the stack trace and throws `MailException`.
  - `private String getStackTrace(Exception exception)` – Utility to convert an exception’s stack trace to a `String` for logging.

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | `messageBody` (String) | Read system properties: `notif_mail_to`, `notif_mail_cc`, `notif_mail_from`, `notif_mail_host` | Configuration values |
| 2 | Config values | Create `Properties` → set `mail.smtp.host` → obtain `Session` | JavaMail `Session` |
| 3 | `Session`, config, `messageBody` | Build `MimeMessage` (set From, TO, CC, Subject, Text) | Prepared e‑mail |
| 4 | `MimeMessage` | `Transport.send(message)` | E‑mail delivered to SMTP server |
| 5 | Success | Log INFO | Audit trail |
| 6 | `MessagingException` | Capture stack trace, log ERROR, throw `MailException` | Propagation of failure to caller |

External services: SMTP server identified by `notif_mail_host`. No database or queue interaction.

# Integrations
- **`AsyncNotificationHandler`** – Calls `MailService.sendAlertMail` when JSON parsing fails or other runtime errors occur.
- **`MailException`** – Custom exception used by upstream components to trigger error‑handling workflows.
- **System properties** – Populated at application start (e.g., via JVM `-D` flags or property files loaded by `AsyncNotificationHandler`).

# Operational Risks
- **Missing/invalid system properties** → `NullPointerException` or malformed address; mitigate by validating properties before use.
- **SMTP host unreachable** → `MessagingException` leads to alert mail failure; mitigate with retry logic or fallback SMTP.
- **Large `messageBody`** may exceed SMTP limits; mitigate by truncating or attaching as file.
- **Blocking send** – `Transport.send` is synchronous; high latency can stall processing; mitigate by async execution or thread pool.

# Usage
```java
public static void main(String[] args) {
    // Set required system properties (example)
    System.setProperty("notif_mail_to", "ops@example.com");
    System.setProperty("notif_mail_cc", "dev@example.com");
    System.setProperty("notif_mail_from", "noreply@example.com");
    System.setProperty("notif_mail_host", "smtp.example.com");

    MailService mailService = new MailService();
    try {
        mailService.sendAlertMail("Test alert from Move‑Mediation service");
    } catch (MailException e) {
        e.printStackTrace();
    }
}
```
Debugging: attach a breakpoint inside `sendAlertMail`, verify property values, and mock `Transport.send` if needed.

# configuration
- **System properties** (set via JVM `-D` or programmatically):
  - `notif_mail_to` – Comma‑separated recipient list.
  - `notif_mail_cc` – Comma‑separated CC list.
  - `notif_mail_from` – Sender address.
  - `notif_mail_host` – SMTP server hostname (optional port via `mail.smtp.port` if needed).

# Improvements
1. **Property validation** – Add a private method to verify that all required properties are non‑null and syntactically valid before constructing the message; throw a descriptive `MailException` early.
2. **Asynchronous sending** – Refactor `sendAlertMail` to submit the `MimeMessage` to an executor service, returning a `Future` or using a callback, to prevent blocking the main processing thread.