# Summary
Defines environment‑specific parameters for the **Customer_Invoice_Summary_Estimated** nightly Move‑Mediation loading pipeline. Supplies log path, process‑status file, Hive database/table identifiers, Java loader class names, Hive refresh statements, and the shell script name invoked by the nightly job.

# Key Components
- `MNAAS_customer_invoice_summary_estimated_LogPath` – absolute log file path (date‑stamped).  
- `customer_invoice_summary_estimated_ProcessStatusFileName` – path to the process‑status flag file.  
- `Dname_customer_invoice_summary_estimated` – Hive database name used by the pipeline.  
- `customer_invoice_summary_estimated_temp_classname` – Java class for loading the temporary staging table.  
- `customer_invoice_summary_estimated_classname` – Java class for loading the final table.  
- `customer_invoice_summary_estimated_temp_tblname` – Hive temporary table name.  
- `customer_invoice_summary_estimated_tblname` – Hive final table name.  
- `customer_invoice_summary_estimated_temp_tblname_refresh` – Hive `REFRESH` command for the temp table.  
- `customer_invoice_summary_estimated_tblname_refresh` – Hive `REFRESH` command for the final table.  
- `MNAAS_customer_invoice_summary_estimated_tblname_ScriptName` – shell script invoked by the nightly scheduler (`customer_invoice_summary_estimated.sh`).  

# Data Flow
1. **Input** – Source data extracted by the Java loader classes (`*_temp_classname` and `*_classname`).  
2. **Processing** – Data loaded into Hive temporary table, refreshed, then merged/inserted into final Hive table.  
3. **Output** – Populated Hive tables (`customer_invoice_summary_estimated_table_temp`, `customer_invoice_summary_estimated_table`).  
4. **Side Effects** – Log written to `$MNAAS_customer_invoice_summary_estimated_LogPath`; process‑status file updated; Hive metadata refreshed.  
5. **External Services** – Hadoop/Hive cluster, Java runtime, shell environment.  

# Integrations
- Sources common properties from `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`.  
- Invoked by the nightly scheduler via `customer_invoice_summary_estimated.sh`.  
- Java loader classes (`com.tcl.mnaas.billingviews.*`) are compiled JARs referenced by the shell script.  
- Hive database name (`$dbname`) resolved from common properties; used in refresh statements.  

# Operational Risks
- **Log path overflow** – date‑stamped log may grow without rotation. *Mitigation*: implement logrotate.  
- **Stale process‑status file** – failure to clear flag can block downstream jobs. *Mitigation*: ensure script cleans up on exit (trap).  
- **Hive refresh failure** – if `$dbname` is undefined, refresh commands error. *Mitigation*: validate common properties load before execution.  
- **Java class version mismatch** – changes to loader classes without corresponding script update. *Mitigation*: enforce CI/CD version lock.  

# Usage
```bash
# Source common properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Enable debug (optional)
set -x   # or comment out setparameter line to disable

# Execute the pipeline
./customer_invoice_summary_estimated.sh
```
To debug, uncomment `setparameter='set -x'` or run the script with `bash -x`.  

# Configuration
- **Environment Variables**: `MNAASLocalLogPath`, `dbname` (provided by common properties).  
- **Referenced Config Files**: `MNAAS_CommonProperties.properties`.  
- **Hard‑coded Paths**: Process‑status file at `/app/hadoop_users/MNAAS/MNAAS_Configuration_Files/...`.  

# Improvements
1. **Add log rotation** – integrate `logrotate` configuration for `$MNAAS_customer_invoice_summary_estimated_LogPath`.  
2. **Parameter validation** – insert pre‑run checks that `MNAASLocalLogPath`, `dbname`, and Java class JARs exist; abort with clear error if missing.