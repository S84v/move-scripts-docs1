# Summary
`MNAAS_Billing_tables_loading.properties` is a shell‑sourced configuration file that supplies the `MNAAS_Billing_tables_loading.sh` job with environment‑specific constants. It aggregates common and billing‑related property files, defines the script’s name, status‑file location, and log‑file path (including a date suffix). The file also provides a toggle for Bash debugging.

# Key Components
- **Sourced property files** – loads shared constants and multiple billing‑specific tables:
  - `MNAAS_CommonProperties.properties`
  - `billable_sub_with_charge_and_limit_loading.properties`
  - `customer_invoice_summary_estimated.properties`
  - `customers_with_overage_flag.properties`
  - `service_based_charges.properties`
  - `usage_overage_charge.properties`
- **Debug toggle** – `setparameter='set -x'` (comment out to enable Bash tracing).
- **Process status file variable** – `MNAAS_Billing_tables_loading_ProcessStatusFileName`.
- **Script identifier** – `MNAAS_Billing_tables_loading_ScriptName`.
- **Log file variable** – `MNAAS_Billing_tables_loadingLogPath` (includes current date).

# Data Flow
| Element | Direction | Description |
|---------|------------|-------------|
| **Inputs** | Read | All sourced property files provide constants (e.g., DB tables, Hive settings, file paths). |
| **Environment variables** | Read | `MNAASConfPath`, `MNAASLocalLogPath` (must be defined before sourcing). |
| **Script execution** | Consumes | `MNAAS_Billing_tables_loading.sh` reads the variables defined here. |
| **Status file** | Write | `MNAAS_Billing_tables_loading_ProcessStatusFileName` is updated by the script to reflect job state. |
| **Log file** | Append | `MNAAS_Billing_tables_loadingLogPath` receives stdout/stderr from the script. |
| **Side effects** | None directly | All side effects (Hive loads, DB updates) are performed by the driver script, not this config. |

# Integrations
- **Common property library** – `MNAAS_CommonProperties.properties` supplies global paths, Hadoop/Hive credentials, and generic constants.
- **Billing table definitions** – each of the six billing‑specific property files defines Hive table names, column mappings, and charge rules used by the loader script.
- **Driver script** – `MNAAS_Billing_tables_loading.sh` sources this file to obtain its configuration before executing Hive/SQL load steps.
- **Logging infrastructure** – log path integrates with the central log aggregation system via `$MNAASLocalLogPath`.

# Operational Risks
- **Hard‑coded date in log filename** – creates a new log file per run; may cause uncontrolled log growth. *Mitigation*: implement log rotation or a fixed filename with timestamped entries.
- **Debug flag exposure** – leaving `setparameter='set -x'` uncommented in production can flood logs with command traces. *Mitigation*: enforce commenting out via CI lint rule.
- **Missing environment variables** – if `MNAASConfPath` or `MNAASLocalLogPath` are undefined, sourcing fails. *Mitigation*: add validation checks at the top of the script.
- **Path coupling** – absolute paths to property files assume a fixed HDFS/Hadoop layout. *Mitigation*: parameterize base directory.

# Usage
```bash
# Ensure required env vars are set
export MNAASConfPath=/app/hadoop_users/MNAAS/MNAAS_Property_Files
export MNAASLocalLogPath=/var/log/mnaas

# Source configuration (normally done inside the driver script)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Billing_tables_loading.properties

# Run the driver script
bash $MNAAS_Billing_tables_loading_ScriptName   # uses variables defined above

# To enable Bash tracing for debugging, comment out the setparameter line in the .properties file
# then re‑source and re‑run.
```

# Configuration
- **Environment variables required**
  - `MNAASConfPath` – directory containing all property files.
  - `MNAASLocalLogPath` – base directory for log files.
- **Referenced config files**
  - `MNAAS_CommonProperties.properties`
  - `billable_sub_with_charge_and_limit_loading.properties`
  - `customer_invoice_summary_estimated.properties`
  - `customers_with_overage_flag.properties`
  - `service_based_charges.properties`
  - `usage_overage_charge.properties`
- **Local variables defined**
  - `setparameter` – Bash debug flag.
  - `MNAAS_Billing_tables_loading_ProcessStatusFileName`
  - `MNAAS_Billing_tables_loading_ScriptName`
  - `MNAAS_Billing_tables_loadingLogPath`

# Improvements
1. **Externalize log rotation** – replace date‑suffixed log filename with a static name and configure logrotate to manage retention.
2. **Add validation block** – at the end of the file, verify that `MNAASConfPath` and `MNAASLocalLogPath` exist and are writable; exit with a clear error if not.