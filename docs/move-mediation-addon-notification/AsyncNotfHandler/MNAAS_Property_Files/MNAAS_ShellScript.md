# Summary
The `MNAAS_ShellScript.properties` file supplies environment‑specific connection parameters and service settings for the Move‑GBS (MNAAS) components. It defines Oracle database credentials for the Test instance, Impala/Hive endpoints, email SMTP details, and Kafka broker information used at runtime by the mediation‑addon‑notification services to persist data, query Hive, send notifications, and publish events.

# Key Components
- **Oracle DB section** – host, port, service name, username, password for the Move Test database (`comtst`).  
- **Impala/Hive section** – `IMPALAD_HOST` and `IMPALAD_JDBC_PORT` for Hive connectivity (production).  
- **Email section** – `movegbs_mail_to`, `movegbs_mail_cc`, `movegbs_mail_from`, `movegbs_mail_host` for SMTP notifications.  
- **Kafka section** – `kafka_servers_list` and `kafka_port` defining the Kafka bootstrap server for event publishing.

# Data Flow
- **Inputs**: Runtime reads this properties file to obtain connection strings and credentials.  
- **Outputs**:  
  - Oracle JDBC connections used by DAO layer to execute `INSERT` statements into `mnaas.api_notification_async`.  
  - Impala JDBC connections for Hive queries executed by analytics or reporting modules.  
  - SMTP messages sent via configured mail host for alerting.  
  - Kafka producer publishes messages to the specified broker.  
- **Side Effects**: Persistent writes to Oracle DB, Hive query execution, email dispatch, Kafka topic writes.  
- **External Services/DBs**: Oracle (`comtst`), Impala/Hive (`192.168.124.91:21050`), SMTP server (`mxrelay.vsnl.co.in`), Kafka broker (`10.133.43.96:9092`).

# Integrations
- **AsyncNotfHandler DAO** – consumes Oracle parameters to open JDBC connections for async notification persistence.  
- **Hive query modules** – read `IMPALAD_HOST`/`IMPALAD_JDBC_PORT` to construct Hive JDBC URLs.  
- **Mail utility classes** – use `movegbs_mail_*` properties to configure JavaMail sessions.  
- **Kafka producer utilities** – reference `kafka_servers_list`/`kafka_port` for `ProducerConfig.BOOTSTRAP_SERVERS_CONFIG`.

# Operational Risks
- Plain‑text passwords (`ora_passwordMOVE`) expose credentials; risk of unauthorized DB access.  
- Hard‑coded production host (`IMPALAD_HOST`) may cause accidental writes to prod from non‑prod environments.  
- Commented‑out production DB entries could be unintentionally re‑enabled, leading to data leakage.  
- Single point of failure: all services depend on this file; corruption prevents startup.

# Mitigations
- Store secrets in a vault (e.g., HashiCorp Vault, AWS Secrets Manager) and inject at runtime.  
- Separate environment‑specific property files and enforce loading via profile flags.  
- Add checksum validation or version control hooks to detect accidental modifications.  

# Usage
```bash
# Example Java snippet to load properties
Properties p = new Properties();
try (InputStream in = new FileInputStream("MNAAS_ShellScript.properties")) {
    p.load(in);
}
String url = "jdbc:oracle:thin:@" + p.getProperty("ora_serverNameMOVE") + ":" +
             p.getProperty("ora_portNumberMOVE") + "/" + p.getProperty("ora_serviceNameMOVE");
String user = p.getProperty("ora_usernameMOVE");
String pass = p.getProperty("ora_passwordMOVE");
// Use url, user, pass to obtain a JDBC connection.
```
Run the application with the classpath containing this file; the component code will read the properties at startup.

# configuration
- **File**: `move-mediation-addon-notification/AsyncNotfHandler/MNAAS_Property_Files/MNAAS_ShellScript.properties`  
- **Referenced by**: DAO layer (`MOVEDAO.properties`), Hive utilities, mail utilities, Kafka producer.  
- **Environment variables** (optional overrides): none defined; can be added to map to property keys.

# Improvements
1. Externalize sensitive credentials to a secure secret manager and replace plain‑text entries with token placeholders.  
2. Introduce environment‑specific property files (e.g., `MNAAS_ShellScript_dev.properties`, `MNAAS_ShellScript_prod.properties`) and load them based on a runtime profile flag.