**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\AlertEmail.java`  
**Type:** Java utility class – sends alert e‑mail notifications for the API Access Management “move” process.

---

## 1. High‑level Summary
`AlertEmail` is a small, self‑contained utility that loads runtime configuration from `APIAcessMgmt.properties`, initialises Log4j, and sends a plain‑text alert e‑mail (subject *ALERT – RA Reports Export error*) via an unauthenticated SMTP relay. It is invoked either directly via its `main` method (for ad‑hoc testing) or programmatically by other components (e.g., `APICallout`, `APIMain`) when a processing error occurs. The class does not return data; its side‑effect is the delivery of an e‑mail and a Log4j entry.

---

## 2. Important Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`AlertEmail` (class)** | Encapsulates email‑sending logic; loads configuration, creates a Log4j logger, and provides `sendAlertMail`. |
| **`main(String[] args)`** | Test harness: loads properties, sets them as JVM system properties, configures Log4j file name, and sends a sample alert (`"Sample alert message"`). |
| **`sendAlertMail(String messageBody)`** | Reads e‑mail parameters from system properties, builds a `MimeMessage`, and sends it via `Transport.send`. Logs success or failure. |
| **`getStackTrace(Exception e)`** (private static) | Utility to convert an exception stack trace to a `String` for logging. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • `APIAcessMgmt.properties` (path hard‑coded to `E:\eclipse-workspace\APIAccessManagement\PropertyFiles\APIAcessMgmt.properties`). <br>• System properties required for e‑mail: `error_mail_to`, `email_from`, `email_cc`, `mail_host`. |
| **Outputs** | • No return value. <br>• Log4j entry in `Accessmgmt.log`. |
| **Side Effects** | • Sends an e‑mail via the SMTP host defined in `mail_host`. <br>• Writes to the Log4j file (`Accessmgmt.log`). |
| **Assumptions** | • The property file exists at the hard‑coded Windows path and contains all required keys. <br>• The SMTP host (`mail_host`) accepts unauthenticated connections from the JVM host. <br>• Log4j is correctly configured via the `log4j.properties` file located elsewhere in the project. <br>• The JVM has permission to write to `E:\eclipse-workspace\APIAccessManagement\APICronLogs\Accessmgmt.log`. |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Connected Component | Interaction |
|---------------------|-------------|
| **`APIAcessMgmt.properties`** | Provides all runtime configuration (e‑mail addresses, SMTP host, log file name). |
| **Log4j configuration (`log4j.properties`)** | Determines logging format, level, and appenders for the `logger` used in this class. |
| **`APICallout`, `APIMain`, `AccountNumDetails`** (other Java classes in the same package) | Likely catch exceptions during API calls and invoke `new AlertEmail().sendAlertMail(errorMsg)` to raise an alert. The exact call sites should be verified in those source files. |
| **Cron / Scheduler** | The `APIAccessManagement` package is executed on a schedule (e.g., via a Unix/Windows cron). When a scheduled run fails, the scheduler’s wrapper may invoke `AlertEmail` to notify operators. |
| **External SMTP relay (`mail_host`)** | The only external service used; the host is defined in the properties file (e.g., `mxrelay.vsnl.co.in`). |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hard‑coded workspace path** (`E:\eclipse-workspace\...`) | Breaks when deployed to a different environment (Linux, different directory). | Externalise the base path via an environment variable or a property (`workspace.path`). |
| **Missing or malformed properties** | `NullPointerException` or failed e‑mail delivery. | Validate required properties at startup; fail fast with clear log messages. |
| **Unauthenticated SMTP** | Mail may be rejected or become a vector for spoofing. | Switch to authenticated SMTP (add `mail.smtp.auth`, `mail.smtp.starttls.enable`, credentials). |
| **No retry/back‑off** on transient mail failures. | Lost alerts during temporary network issues. | Implement a simple retry loop (e.g., 3 attempts with exponential back‑off). |
| **Log4j static logger initialisation after `System.setProperty("logfile.name")`** may not affect already‑configured Log4j. | Log file may not be created or may go to a default location. | Initialise Log4j *after* setting the system property, or use a programmatic configuration. |
| **Blocking call to `Transport.send`** may delay the main job. | Potential job timeout. | Send mail asynchronously (e.g., via a thread pool) or use a non‑blocking mail library. |

---

## 6. Running / Debugging the Class

1. **Prerequisites**  
   - JDK 8+ installed.  
   - `APIAcessMgmt.properties` present at the hard‑coded location (or adjust the path).  
   - Log4j JARs on the classpath (as used by the rest of the project).  
   - Network access to the SMTP host defined in `mail_host`.

2. **Compile** (example using Maven/Gradle or plain `javac`):  
   ```bash
   javac -cp ".;path/to/log4j.jar;path/to/mail.jar" \
         src/com/tcl/api/callout/AlertEmail.java
   ```

3. **Execute the test harness**:  
   ```bash
   java -cp ".;path/to/log4j.jar;path/to/mail.jar" \
        com.tcl.api.callout.AlertEmail
   ```
   - Observe `Accessmgmt.log` for the “Message sent successfully” entry.  
   - Verify receipt of the e‑mail at the addresses listed in `error_mail_to`.

4. **Debugging tips**  
   - Set `log4j.rootLogger=DEBUG, console` in `log4j.properties` to see detailed logs.  
   - If the e‑mail is not sent, check: (a) property values, (b) network connectivity to `mail_host`, (c) firewall rules, (d) SMTP server logs.  
   - Use a breakpoint on `Transport.send(message)` to inspect the constructed `MimeMessage`.  

5. **Integration test** (called from another class):  
   ```java
   AlertEmail ae = new AlertEmail();
   ae.sendAlertMail("Test alert from APICallout");
   ```
   Ensure the calling class has already loaded the same properties (or invoke `AlertEmail.main` first).

---

## 7. External Configuration & Environment Variables

| Config Item | Source | Usage |
|-------------|--------|-------|
| `APIAcessMgmt.properties` | File system (`E:\eclipse-workspace\APIAccessManagement\PropertyFiles\APIAcessMgmt.properties`) | Loaded at runtime; each key is copied to a JVM system property. |
| `logfile.name` | Set programmatically (`System.setProperty("logfile.name", logFile)`) | Used by Log4j to determine the log file location (`Accessmgmt.log`). |
| `error_mail_to`, `email_from`, `email_cc`, `mail_host` | Defined inside `APIAcessMgmt.properties` | Consumed by `sendAlertMail` to build the e‑mail. |
| `log4j.properties` | Project root (`move-indiamed-api\APIAccessManagement\src\log4j.properties` or `bin\log4j.properties`) | Configures Log4j logger; not referenced directly in this file but required for logging. |

*If any of these files are missing or the path changes, the class will throw an `IOException` or fail to send mail.*

---

## 8. Suggested Improvements (TODO)

1. **Externalise the workspace base path** – replace the hard‑coded `E:\\eclipse-workspace\\APIAccessManagement\\` with a configurable property or environment variable (`WORKSPACE_ROOT`). This makes the code portable across OSes and deployment environments.

2. **Add SMTP authentication & TLS** – modify `sendAlertMail` to read `mail_user`, `mail_password`, and enable `mail.smtp.starttls.enable`. Use `Session.getInstance(properties, authenticator)` to create a secure, authenticated session, reducing the risk of mail relay abuse and improving deliverability. 

*(Both changes can be implemented without affecting existing functionality; they will require updates to `APIAcessMgmt.properties` and a rebuild.)*