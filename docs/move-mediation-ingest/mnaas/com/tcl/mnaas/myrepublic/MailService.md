# Summary
`MailService` is a utility class that sends alert e‑mails when file‑processing errors occur in the Move‑Mediation ingest pipeline. It builds a simple SMTP message using hard‑coded recipients, subject, and host, logs success or failure, and returns the exception stack trace as a string.

# Key Components
- **Class `MailService`**
  - `static void sendAlertMail(String messageBody)`: constructs and sends an e‑mail; logs outcome.
  - `private static String getStackTrace(Exception e)`: converts an exception’s stack trace to a `String` for logging.

# Data Flow
| Step | Input | Processing | Output / Side Effect |
|------|-------|------------|----------------------|
| 1 | `messageBody` (String) | Build SMTP `Properties` (host) and `Session` | N/A |
| 2 | Hard‑coded e‑mail fields (`to`, `cc`, `from`, `subject`) | Create `MimeMessage`, set headers, body | N/A |
| 3 | `Transport.send(message)` | Sends e‑mail via SMTP server `mxrelay.vsnl.co.in` | Email delivered to recipients |
| 4 | Success | `logger.info` | Log “Message sent successfully …” |
| 5 | `MessagingException` | Capture, convert stack trace via `getStackTrace` | `logger.error` with stack trace |

External services: SMTP server (`mxrelay.vsnl.co.in`). No database, queue, or file I/O.

# Integrations
- Invoked by error‑handling code in other components (e.g., `Dups_file_removal`) to notify operators.
- Relies on Log4j for logging; integrates with the application’s logging configuration.
- Uses JavaMail API; requires `javax.mail` library on the classpath.

# Operational Risks
- **Hard‑coded recipients/host**: changes require code redeployment. *Mitigation*: externalize to properties or environment variables.
- **No authentication**: SMTP server may reject unauthenticated sends. *Mitigation*: add support for `mail.smtp.auth` and credentials.
- **Blocking send**: `Transport.send` is synchronous; a slow or unavailable SMTP server can delay processing. *Mitigation*: send asynchronously or with timeout handling.
- **Missing JavaMail dependency**: runtime `NoClassDefFoundError`. *Mitigation*: ensure library is packaged with the application.

# Usage
```java
// Example from a catch block
catch (Exception ex) {
    MailService.sendAlertMail("File processing failed: " + ex.getMessage());
}
```
To debug, set Log4j level to `DEBUG` for `com.tcl.mnaas.myrepublic.MailService` and verify SMTP connectivity (e.g., `telnet mxrelay.vsnl.co.in 25`).

# Configuration
- Currently **hard‑coded** within the source:
  - `to` = `balachandar.iyanar@tatacommunications.com`
  - `cc` = `saravanan.sandirane@tatacommunications.com,anusha.surapaneni@tatacommunications.com`
  - `from` = `anusha.gurumurthy@tatacommunications.com`
  - `host` = `mxrelay.vsnl.co.in`
- Commented placeholders indicate intended system properties:
  - `moveba_mail_to`
  - `moveba_mail_cc`
  - `moveba_mail_from`
  - `moveba_mail_host`

# Improvements
1. **Externalize configuration**: read SMTP parameters from a properties file or environment variables; fallback to defaults if absent.
2. **Add authentication & TLS support**: include `mail.smtp.auth`, `mail.smtp.starttls.enable`, and credential handling to work with modern mail servers.