# Summary
`MNAAS_ShellScript.properties` supplies runtime configuration for the Move‑Mediation‑Billing‑Reports daily job. It defines the Impala (Hive) connection endpoint, email routing parameters, and the local Excel export directory. The properties are consumed by shell/Java DAO scripts to establish database connections, generate CSV/Excel reports, and dispatch status/error notifications.

# Key Components
- **IMPALAD_HOST / IMPALAD_JDBC_PORT** – Target Impala service for all SELECT/INSERT statements defined in `MOVEDAO.properties`.
- **error_mail_to, export_mail_to, export_mail_cc, export_mail_from** – Email addresses used by the notification module to send success, failure, and export‑ready messages.
- **mail_host** – SMTP relay host for outbound mail.
- **excel_path** – Filesystem root where generated Excel workbooks are written before archival or downstream consumption.

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| 1. Script start | `MNAAS_ShellScript.properties` | Load properties into environment variables | In‑memory config map | – |
| 2. DB access | `IMPALAD_HOST`, `IMPALAD_JDBC_PORT` + SQL from `MOVEDAO.properties` | JDBC/ODBC connection → execute queries | Result sets → CSV/Excel files | Network traffic to Impala cluster |
| 3. Report generation | Query result sets, `excel_path` | Transform → Excel workbook (Apache POI) | Files under `excel_path` | Disk I/O |
| 4. Notification | Email properties, job status | Build MIME message → send via `mail_host` | Email to recipients | SMTP traffic |
| 5. Cleanup | Temporary files | Delete/archival | None | File system changes |

# Integrations
- **DAO Layer** (`MOVEDAO.properties`) reads this file to construct prepared statements.
- **Shell wrapper** (`run_daily_report.sh` or equivalent) sources the properties before invoking Java/Scala jobs.
- **Mail utility** (`sendMail.jar` or native `mailx`) consumes the SMTP and address settings.
- **File system** – Excel files are later consumed by downstream billing/analytics pipelines (e.g., Hadoop ingest or S3 upload).

# Operational Risks
- **Hard‑coded host IP** – IP change in Impala cluster breaks connectivity. *Mitigation*: externalize to DNS or environment variable.
- **Static email addresses** – Personnel turnover requires manual updates. *Mitigation*: maintain a central address book or use distribution lists.
- **Local Windows path** (`C:\\Anu Office\\`) – Not portable across OSes; may lack write permission. *Mitigation*: parameterize path per environment and validate at startup.
- **Missing credentials** – No authentication fields; reliance on Kerberos or external config may cause silent failures. *Mitigation*: document required Kerberos keytab location and enforce presence check.

# Usage
```bash
# Example: run the daily report using the properties file
export PROPS_FILE=move-mediation-billing-reports/DailyReport/MNAAS_Property_Files/MNAAS_ShellScript.properties
source $PROPS_FILE          # loads variables into the shell
java -cp lib/* com.tata.mnaas.ReportRunner \
    --host $IMPALAD_HOST \
    --port $IMPALAD_JDBC_PORT \
    --excelDir "$excel_path" \
    --mailHost $mail_host \
    --to $export_mail_to \
    --cc $export_mail_cc \
    --from $export_mail_from
```
For debugging, add `set -x` after sourcing the file to trace variable expansion.

# configuration
- **Environment variables**: None required; all values are defined in this properties file.
- **Referenced config files**: `MOVEDAO.properties` (SQL statements), optional Kerberos keytab (`/etc/security/keytabs/mnaas.keytab`).
- **External services**: Impala cluster (`IMPALAD_HOST:IMPALAD_JDBC_PORT`), SMTP relay (`mail_host`).

# Improvements
1. **Externalize sensitive/volatile settings** – Move `IMPALAD_HOST`, `mail_host`, and email addresses to environment variables or a secure vault (e.g., HashiCorp Vault) and reference them via `${VAR}` syntax.
2. **Platform‑agnostic path handling** – Replace hard‑coded Windows path with a configurable property that supports Unix separators, and validate write permissions at startup.