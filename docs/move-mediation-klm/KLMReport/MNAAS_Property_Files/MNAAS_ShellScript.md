# Summary
`MNAAS_ShellScript.properties` is a Java‑style properties file consumed by the KLMReport component of the Move‑Mediation‑Ingest pipeline. It centralises environment‑specific connection parameters (Oracle, Impala/Hive), email routing settings, and a static Excel output path. At runtime shell scripts or Java utilities load the file to obtain credentials and endpoint values required for ETL jobs, Hive queries, and notification emails.

# Key Components
- **oracle.*_MOVE** – JDBC host, port, service name, username, password for the Move Oracle database (Prod/Dev/Test).  
- **IMPALAD_HOST / IMPALAD_JDBC_PORT** – Impala/Hive Thrift endpoint for Hive metastore access (Prod/Dev).  
- **geneva_mail_*** – SMTP sender, recipient, CC, and host for automated email alerts.  
- **excel_path** – Absolute filesystem location for generated Excel reports.  

# Data Flow
| Direction | Source / Destination | Description |
|-----------|-----------------------|-------------|
| Input | Shell/Java ETL jobs | Load properties via `java.util.Properties` or `source` in Bash. |
| Output | Oracle DB (MOVE) | JDBC connections using `ora_*_MOVE` values for data extraction/insertion. |
| Output | Hive/Impala | Thrift connections using `IMPALAD_HOST`/`IMPALAD_JDBC_PORT` for query execution. |
| Side‑effect | Email system | `geneva_mail_*` values drive SMTP notifications (e.g., job success/failure). |
| Side‑effect | Filesystem | `excel_path` determines where the KLMReport Excel file is written. |

# Integrations
- **KLMReport Java classes** (`MOVEDAO`, `HiveExecutor`, `MailNotifier`) read this file to configure their data sources and notification channels.  
- **Shell scripts** (`run_klm_report.sh`, `load_data.sh`) source the file to export environment variables for `sqlplus`, `impala-shell`, and `mailx`.  
- **CI/CD pipelines** inject the file into the container image; the runtime component expects it at `MNAAS_Property_Files/MNAAS_ShellScript.properties`.  

# Operational Risks
- **Plain‑text credentials** – Exposure if the file is checked into source control or accessed by unauthorized users. *Mitigation*: encrypt passwords (e.g., JCEKS) or use a secret manager (Vault, AWS Secrets Manager).  
- **Hard‑coded absolute paths** (`excel_path`) – Breaks on node re‑provisioning or OS changes. *Mitigation*: make the path configurable via an environment variable or command‑line argument.  
- **Stale commented sections** – May cause confusion about the active environment. *Mitigation*: maintain a single active block per environment and version‑control comments.  
- **Static SMTP host** – Failure if the mail relay changes. *Mitigation*: externalise SMTP configuration to a central properties file or environment variable.  

# Usage
```bash
# Example: Java utility loading the properties
java -cp myapp.jar com.tcl.move.klmreport.Main \
     -Dconfig.file=./MNAAS_Property_Files/MNAAS_ShellScript.properties

# Example: Bash script sourcing the file
#!/bin/bash
source ./MNAAS_Property_Files/MNAAS_ShellScript.properties
export ORACLE_URL="jdbc:oracle:thin:@${ora_serverNameMOVE}:${ora_portNumberMOVE}/${ora_serviceNameMOVE}"
sqlplus ${ora_usernameMOVE}/${ora_passwordMOVE}@${ORACLE_URL} @run_etl.sql
```

# Configuration
- **File location**: `move-mediation-klm/KLMReport/MNAAS_Property_Files/MNAAS_ShellScript.properties` (must be on the classpath or accessible to scripts).  
- **Environment variables** (optional overrides): `ORACLE_HOST`, `ORACLE_PORT`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`, `EXCEL_PATH`, `SMTP_HOST`.  
- **Dependent files**: `MOVEDAO.properties` (SQL statements), `log4j.properties` (logging), shell wrappers (`run_klm_report.sh`).  

# Improvements
1. **Secure secrets** – Replace clear‑text passwords with references to a vault service; load at runtime via a thin decryption wrapper.  
2. **Dynamic environment selection** – Introduce a `profile=PROD|DEV|TEST` property and programmatically switch the appropriate block, eliminating manual comment toggling.  