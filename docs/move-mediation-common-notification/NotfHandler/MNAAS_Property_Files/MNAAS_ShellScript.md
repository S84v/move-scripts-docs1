# Summary
`MNAAS_ShellScript.properties` supplies environment‑specific connection parameters and operational constants for the Move Mediation Notification Handler (MNAAS) DAO layer and ancillary services (mail, Kafka, Hive). In production it is read at runtime to configure JDBC connections to the Move Oracle database, Impala/Hive endpoints, SMTP relay, and Kafka broker used by the notification ingestion pipeline.

# Key Components
- **Oracle DB connection properties** – `ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE` (active test instance; production values are commented out).  
- **Hive/Impala connection properties** – `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`.  
- **Mail server settings** – `movegbs_mail_to`, `movegbs_mail_cc`, `movegbs_mail_from`, `movegbs_mail_host`.  
- **Kafka broker settings** – `kafka_server`, `kafka_port`.  
- **Commented-out production/dev blocks** – placeholders for quick switch between environments.

# Data Flow
| Source | Consumed By | Purpose |
|--------|-------------|---------|
| `MNAAS_ShellScript.properties` | `MOVEDAO` Java classes (DAO implementations) | Build `java.sql.Connection` for Oracle (`jdbc:oracle:thin:@//${ora_serverNameMOVE}:${ora_portNumberMOVE}/${ora_serviceNameMOVE}`) |
| Same file | `HiveUtil` / `ImpalaClient` | Construct JDBC URL `jdbc:hive2://${IMPALAD_HOST}:${IMPALAD_JDBC_PORT}/default` |
| Same file | `MailNotifier` | Populate `javax.mail.Session` with SMTP host and default sender/recipients |
| Same file | `KafkaProducerWrapper` | Initialise `org.apache.kafka.clients.producer.KafkaProducer` with bootstrap `${kafka_server}:${kafka_port}` |
| **Side Effects** | DAO writes/updates tables in `mnaas` schema; MailNotifier sends email alerts; KafkaProducer publishes notification events. |

# Integrations
- **DAO Layer** (`MOVEDAO.properties` references) → uses the Oracle credentials from this file to execute INSERT/UPDATE/SELECT statements.  
- **Hive/Impala Jobs** → downstream analytics scripts read data written by the DAO; they obtain connection details from this file.  
- **Notification Service** → after persisting a record, the service may trigger email via `MailNotifier` and publish to Kafka; both rely on the SMTP/Kafka settings defined here.  
- **External Scripts** → shell or CI jobs may source this file to export environment variables before launching Java processes.

# Operational Risks
- **Plain‑text passwords** – exposure of `ora_passwordMOVE` and other credentials. *Mitigation*: encrypt with JCEKS or use a secret manager; restrict file permissions (0600).  
- **Stale environment blocks** – commented production values may be unintentionally re‑enabled, causing accidental writes to prod DB. *Mitigation*: enforce environment‑specific property files (e.g., `MNAAS_Prod.properties`).  
- **Hard‑coded email recipients** – limits flexibility and may cause notification leakage. *Mitigation*: externalize recipient lists to a database table or separate config.  
- **Single point of failure for Kafka/SMTP** – if broker or mail host is down, notification pipeline stalls. *Mitigation*: configure fallback hosts and implement retry/back‑off logic.

# Usage
```bash
# Export properties as environment variables for a Java run
export $(grep -v '^#' MNAAS_ShellScript.properties | xargs)

# Run the notification handler (example)
java -cp move-mediation-common-notification.jar \
     -Dlog4j.configuration=file:log4j.properties \
     com.tcl.move.notification.HandlerMain
```
*Debug*: Verify loaded values with `System.out.println(System.getProperty("ora_serverNameMOVE"));` or by checking the Java `Properties` object after `Properties.load()`.

# Configuration
- **File Path**: `move-mediation-common-notification\NotfHandler\MNAAS_Property_Files\MNAAS_ShellScript.properties` (must be on classpath or referenced via `-Dproperty.file=`).  
- **Referenced Configs**: `MOVEDAO.properties` (SQL statements), `log4j.properties` (logging).  
- **Environment Variables** (optional overrides): `ORA_SERVER`, `IMPALAD_HOST`, `KAFKA_SERVER`, etc., can be set before JVM start to supersede file values.

# Improvements
1. **Secure Credential Management** – replace plain passwords with references to a vault (e.g., HashiCorp Vault, AWS Secrets Manager) and load at runtime.  
2. **Environment Segregation** – split into `MNAAS_Prod.properties`, `MNAAS_Dev.properties`, `MNAAS_Test.properties` and enforce selection via a mandatory `-Denvironment=` JVM argument to avoid accidental cross‑environment usage.