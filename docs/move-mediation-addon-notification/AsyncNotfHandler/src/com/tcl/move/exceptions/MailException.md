# Summary
`MailException` is a custom checked exception that encapsulates errors occurring during mail‑related operations within the Move‑Mediation add‑on notification service. It provides a single‑argument constructor forwarding the error message to the base `Exception` class, enabling consistent error handling and logging for mail failures.

# Key Components
- **class `MailException`**
  - Extends `java.lang.Exception`.
  - Defines `serialVersionUID` for serialization compatibility.
  - Provides a constructor `MailException(String msg)` that passes the message to the superclass.

# Data Flow
- **Input:** String error message supplied by calling code when a mail operation fails.
- **Output:** An instantiated `MailException` object propagated up the call stack.
- **Side Effects:** None; solely represents an error condition.
- **External Services/DBs/Queues:** None directly; used by components that interact with SMTP servers or mail APIs.

# Integrations
- Thrown by mail utility classes (e.g., `MailSender`, `NotificationMailer`) when SMTP connection, authentication, or message composition fails.
- Caught by higher‑level service layers (`AsyncNotfHandler`, job schedulers) to trigger retry logic, alerting, or transaction rollback.
- May be logged via the application’s logging framework (e.g., Log4j) for operational visibility.

# Operational Risks
- **Uncaught `MailException`** → service thread termination or message loss. *Mitigation:* Ensure all mail‑invoking code includes try‑catch blocks that handle `MailException` and implement fallback or retry.
- **Loss of original cause** (stack trace) because only message is captured. *Mitigation:* Extend the class to accept a `Throwable cause` and pass it to `super(msg, cause)`.

# Usage
```java
try {
    mailSender.send(email);
} catch (MailException e) {
    logger.error("Mail send failed: {}", e.getMessage());
    // retry, alert, or mark notification as failed
}
```
To debug, set a breakpoint on the `MailException` constructor or catch block and inspect the message.

# Configuration
- No environment variables or external configuration files are referenced directly by this class.
- Relies on application‑wide logging configuration for output.

# Improvements
1. Add a constructor `MailException(String msg, Throwable cause)` to preserve the original exception stack trace.
2. Implement a static factory method `MailException.from(Throwable cause)` that generates a `MailException` with a standardized message format.