**File:** `move-indiamed-api\APIAccessManagementMigration\PropertyFiles\APIAcessMgmt.properties`

---

## 1. High‑Level Summary
This properties file supplies runtime configuration for the *API Access Management Migration* move‑script suite. It defines the JDBC connection string and credentials for the `Mediation_IndiaPOP` SQL Server database, as well as SMTP settings and recipient addresses used by the migration jobs to send status or error notifications. All scripts in the `APIAccessManagementMigration` package read this file at start‑up to obtain the values they need to access the source database and to dispatch email alerts.

---

## 2. Key Configuration Entries (Responsibilities)

| Property | Purpose | Typical Consumer |
|----------|---------|------------------|
| `db.url` | JDBC URL for the Mediation_IndiaPOP database (SQL Server, port 1433). | Database connection pool / DAO classes. |
| `db.username` | DB user for the above URL. | Same as above. |
| `db.password` | Password for `db.username`. | Same as above. |
| `error_mail_to` | Primary recipient for error notifications. | Email utility invoked on script failure. |
| `email_from` | Sender address for all migration‑related e‑mails. | Same as above. |
| `email_cc` | CC address for all migration‑related e‑mails. | Same as above. |
| `mail_host` | SMTP host (currently `localhost`). | JavaMail / SMTP client used by the scripts. |

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Inputs** | The file itself is read as a Java `Properties` object (or equivalent in the scripting language). No external input is required to parse it. |
| **Outputs** | Values are supplied to: <br>• JDBC driver → DB connections <br>• Mail client → SMTP messages (status, error alerts) |
| **Side Effects** | • Opening a network connection to `INMPROXY01.tccspl.local` on port 1433.<br>• Sending e‑mail via the local SMTP server.<br>• Potentially logging the property values (e.g., password) if debug logging is enabled. |
| **Assumptions** | • The SQL Server instance `INMPROXY01.tccspl.local` is reachable from the host running the move scripts.<br>• The `TEST_ONE` account has sufficient privileges on `Mediation_IndiaPOP`.<br>• An SMTP service is listening on `localhost` and accepts the supplied `email_from` address.<br>• The properties file is placed on the classpath or a known configuration directory that the scripts reference. |

---

## 4. Interaction with Other Components

| Component | Connection Point |
|-----------|------------------|
| **Migration Scripts** (`*.groovy`, `*.java`, `*.sh` in `APIAccessManagementMigration`) | Each script loads this file (e.g., `new Properties().load(new FileInputStream("APIAcessMgmt.properties"))`). |
| **Database Layer** (`DAO` or `JdbcTemplate` classes) | Uses `db.url`, `db.username`, `db.password` to obtain a `Connection`. |
| **Email Utility** (`MailSender`, `JavaMailSender`, or custom wrapper) | Reads `mail_host`, `email_from`, `email_cc`, `error_mail_to` to construct and send messages. |
| **Logging / Monitoring** | If scripts log configuration values, this file indirectly influences log output. |
| **Deployment / CI** | The file is typically packaged with the move‑script JAR/WAR or copied to a known config directory during deployment. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Plain‑text credentials** (`db.password`) | Credential leakage, unauthorized DB access. | Store passwords in a secret manager (e.g., HashiCorp Vault, Azure Key Vault) and inject at runtime via environment variables or encrypted property files. |
| **Hard‑coded e‑mail addresses** | Notification mis‑routing if personnel change. | Externalize addresses to a separate “notification” properties file or environment variables; keep a single source of truth. |
| **Static SMTP host (`localhost`)** | Failure if local MTA is down or mis‑configured. | Make `mail_host` configurable per environment; add health‑check for SMTP before sending. |
| **No encryption flag on JDBC URL** (`encrypt=false`) | Data in transit could be intercepted. | Enable TLS (`encrypt=true`) on the SQL Server connection; verify server certificate. |
| **Potential typo in filename (`APIAcessMgmt.properties` vs `APIAccessMgmt.properties`)** | Scripts may fail to locate the file. | Standardize naming; add a startup validation step that logs missing/incorrect config files. |

---

## 6. Running / Debugging the Migration

1. **Preparation**  
   - Ensure the file is present in the directory referenced by the script (e.g., `config/` on the classpath).  
   - Verify network connectivity to `INMPROXY01.tccspl.local:1433` and that the SMTP service on `localhost` is reachable.

2. **Execution** (example for a Groovy/Java based move script)  
   ```bash
   cd move-indiamed-api/APIAccessManagementMigration
   ./runMigration.sh   # wrapper that sets CLASSPATH and invokes the main class
   ```
   The wrapper typically logs “Loading properties from APIAcessMgmt.properties”.

3. **Debugging**  
   - Increase log level to `DEBUG` to see the loaded property values (ensure password masking).  
   - If DB connection fails, test the JDBC URL directly with a client (e.g., `sqlcmd`).  
   - If e‑mail is not sent, use `telnet localhost 25` to verify SMTP connectivity and check the mail server logs.

4. **Verification**  
   - After successful run, check the `error_mail_to` inbox for a “Migration completed” message.  
   - Inspect the target database tables for expected changes.

---

## 7. External References (Config / Env)

| Reference | Usage |
|-----------|-------|
| **Environment Variables** (if any) | Not currently used; could be introduced for `DB_PASSWORD`, `MAIL_HOST`, etc. |
| **Other Property Files** | Likely a master `application.properties` that defines the location of this file; not shown here. |
| **Secret Management** | Not integrated; recommended for future improvement. |

---

## 8. Suggested Improvements (TODO)

1. **Secure Credential Management** – Replace the plain‑text `db.password` with a reference to a secret store and load it at runtime (e.g., `db.password=${vault:mediation/dbPassword}`).

2. **Enable TLS for DB Connections** – Change the JDBC URL to `encrypt=true;trustServerCertificate=false;` and configure the appropriate trust store to protect data in transit.