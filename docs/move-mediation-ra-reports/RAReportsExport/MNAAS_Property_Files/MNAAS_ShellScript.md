# Summary
`MNAAS_ShellScript.properties` supplies environment‑specific configuration for the RAReportsExport batch jobs. It defines Hive/Impala connection parameters, e‑mail notification settings, local extract/backup directories, and SFTP credentials/paths used to stage daily and monthly PPU files to the data lake. The properties are read at runtime by the export shell scripts and Java components to drive data extraction, file transfer, and error reporting in the MOVE‑RA‑Reports production workflow.

# Key Components
- **IMPALAD_HOST / IMPALAD_JDBC_PORT** – Target Impala service for Hive queries.  
- **error_mail_to / error_mail_from / mail_host** – SMTP details for failure notifications.  
- **extract_path** – Local directory where generated RA reports are written.  
- **backup_path** – Local directory for archiving reports before/after transfer.  
- **Datalake_* (port, user, host, password)** – SFTP credentials for the data‑lake ingress node.  
- **Datalake_daily_path / Datalake_monthly_path** – Remote directories where daily and monthly PPU files are uploaded.

# Data Flow
1. **Input**:  
   - Hive/Impala queries executed by Java/SQL components using `IMPALAD_HOST:IMPALAD_JDBC_PORT`.  
   - Generated CSV/PPU files placed in `extract_path`.  

2. **Processing**:  
   - Scripts copy files from `extract_path` to `backup_path` for retention.  
   - Files are transferred via SFTP (port 22) to the data lake host (`Datalake_host`) using `Datalake_user`/`Datalake_password`.  
   - Destination path chosen based on report frequency (daily → `Datalake_daily_path`; monthly → `Datalake_monthly_path`).  

3. **Output**:  
   - Remote files stored in the data‑lake directory structure.  
   - Success/failure logs written to the application logger.  

4. **Side Effects**:  
   - Email alerts sent to `error_mail_to` on any exception, using `mail_host`.  
   - Local backup files retained in `backup_path` for audit/compliance.

# Integrations
- **RAReportsExport Java service** – reads the properties to configure JDBC connections and invoke `CustomParquetWriter`/CSV exporters.  
- **Shell scripts** (`runRAExport.sh`, etc.) – source this file to obtain SFTP and path variables for `scp`/`sftp` commands.  
- **SMTP server** (`mxrelay.vsnl.co.in`) – used by the Java mail utility for error notifications.  
- **Data Lake ingestion pipeline** – consumes files placed in `Datalake_daily_path` and `Datalake_monthly_path` for downstream analytics.

# Operational Risks
- **Plain‑text credentials** – password exposed in file; risk of compromise. *Mitigation*: encrypt with a vault (e.g., HashiCorp Vault) or use OS‑level protected keystore.  
- **Hard‑coded host IPs** – lack of DNS flexibility; changes require file edit and redeploy. *Mitigation*: externalize to environment‑specific property sets or service discovery.  
- **SFTP connectivity failures** – network or auth issues block file delivery. *Mitigation*: implement retry logic and alerting; monitor SFTP logs.  
- **Email delivery failure** – alerts may be missed. *Mitigation*: configure fallback logging and escalation to monitoring system (e.g., PagerDuty).

# Usage
```bash
# Load properties in a shell script
source /opt/move/mnaas/MNAAS_Property_Files/MNAAS_ShellScript.properties

# Example: run daily export
java -cp move-ra-reports.jar com.tcl.raexport.RAExportJob \
    --impalaHost=${IMPALAD_HOST} --impalaPort=${IMPALAD_JDBC_PORT} \
    --outputDir=${extract_path}

# Transfer daily file
sftp -P ${Datalake_port} ${Datalake_user}@${Datalake_host} <<EOF
put ${extract_path}/daily_report.csv ${Datalake_daily_path}/daily_report_$(date +%Y%m%d).csv
EOF
```
To debug, enable verbose logging in the Java job (`-Dlog4j.debug=true`) and add `-v` to the `sftp` command.

# Configuration
- **File**: `MNAAS_ShellScript.properties` (located in `RAReportsExport/MNAAS_Property_Files`).  
- **Referenced by**:  
  - `RAExportJob` Java class (reads Hive and mail settings).  
  - Shell scripts (`runRAExport.sh`, `transfer_to_datalake.sh`).  
- **Environment variables**: None required; all values are defined in the properties file.

# Improvements
1. **Secure credential management** – replace plain‑text `Datalake_password` with a reference to a secret store and load at runtime.  
2. **Parameterize hostnames** – use DNS names or a configuration service to avoid hard‑coded IPs, enabling seamless environment promotion.