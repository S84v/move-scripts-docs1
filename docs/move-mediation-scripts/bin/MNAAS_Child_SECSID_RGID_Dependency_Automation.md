**MNAAS_Child_SECSID_RGID_Dependency_Automation.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)

This Bash utility automates the maintenance of the *Child SECS‑ID ↔ Rating‑Group‑ID* mapping used by the MNAAS mediation platform. Operators can interactively add a new child‑customer mapping or delete an existing one. The script updates a flat‑file property list, loads the refreshed data into a Hive table (`${child_scesid_rgid_mapping_tbl_name}`), and refreshes the corresponding Impala metadata, ensuring downstream billing and traffic‑load jobs see the latest hierarchy information.

---

### 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **MNAAS_Create_New_Child_SECSID_RGID_Mapping** | Prompts the operator for child & parent SECS IDs, customer names, country, and RGID; validates against the existing property file; appends a new pipe‑delimited record; calls `MNAAS_Load_Child_SECSID_RGID_Mapping` to push the change to Hive/Impala. |
| **MNAAS_Delete_Existing_Child_SECSID_RGID_Mapping** | Displays current mappings, collects the exact record to delete, verifies existence, removes the line from the property file using `sed`, then reloads the table via `MNAAS_Load_Child_SECSID_RGID_Mapping`. |
| **MNAAS_Load_Child_SECSID_R​GID_Mapping** | Executes a Hive `LOAD DATA LOCAL INPATH` to overwrite the target table with the current property file; on success, issues an Impala `REFRESH` to make the new data visible; logs success/failure messages. |
| **Main script block** | Shows a simple menu, reads the operator’s choice, and dispatches to the appropriate function. |

---

### 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **User Input (interactive)** | `childsecsid`, `childcustomer`, `childcountry`, `rgid`, `parentsecsid`, `parentcustomer`, and menu choice (`1` or `2`). |
| **Configuration File (sourced)** | `MNAAS_Child_SECSID_RGID_Dependency_Automation.properties` – defines at least:<br>• `MNAAS_New_Child_SECSID_RGID_PropFile` – path to the pipe‑delimited mapping file.<br>• `MNAAS_New_Customer_Onboarding_LogPath` – log file for audit trails.<br>• `child_scesid_rgid_mapping_tbl_name` – Hive table name.<br>• `IMPALAD_HOST` – Impala daemon host. |
| **External Services** | • **Hive** – for `LOAD DATA` command.<br>• **Impala** – for `REFRESH` command (`impala-shell`). |
| **Outputs** | • Updated `${MNAAS_New_Child_SECSID_RGID_PropFile}` (flat file).<br>• Hive table `${child_scesid_rgid_mapping_tbl_name}` refreshed with new data.<br>• Log entries appended to `${MNAAS_New_Customer_Onboarding_LogPath}`. |
| **Side‑Effects** | • Overwrites the entire Hive mapping table each time the script runs (not incremental).<br>• Potentially impacts downstream traffic‑loading jobs if the load fails. |
| **Assumptions** | • The operator has read/write permission on the property file and log path.<br>• Hive and Impala services are reachable from the host where the script runs.<br>• The property file uses the exact pipe‑delimited format: `SECSID|Customer|Country|RGID|ParentSECSID|ParentCustomer`.<br>• No concurrent executions (the script is not lock‑protected). |

---

### 4. Integration Points & System Connections

| Connected Component | How this script interacts |
|---------------------|---------------------------|
| **MNAAS Billing / CDR ingestion pipelines** | They query `${child_scesid_rgid_mapping_tbl_name}` (Hive/Impala) to resolve child‑customer hierarchy for rating and billing. |
| **Other “MNAAS_*_Dependency_Automation” scripts** | Likely share the same property file and table name; changes made here affect all downstream scripts that rely on the child‑RGID mapping. |
| **KYC Database** | The script asks the operator to enter a country that must match the KYC DB; no direct call, but downstream jobs may validate against KYC. |
| **Operational monitoring / alerting** | Not built‑in, but failures are printed to stdout; typical practice is to pipe script output to a log collector (e.g., Splunk, ELK). |
| **Scheduler (e.g., cron, Oozie)** | Usually invoked manually; could be scheduled for periodic verification or bulk updates. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Human input error** (typos in SECS IDs, RGIDs, etc.) | Wrong mapping → billing mis‑allocation, traffic routing errors. | Add validation against master reference tables (e.g., `SELECT` from Hive to confirm SECS ID exists) before committing. |
| **Concurrent edits** (two operators run script simultaneously) | Race condition → lost updates or corrupted property file. | Implement a file lock (`flock`) around read‑modify‑write sections. |
| **Hive/Impala load failure** (network, permission, schema change) | Mapping table not refreshed → downstream jobs use stale data. | Capture exit codes, retry with exponential back‑off, and send an alert (email/Slack). |
| **Overwrite of entire table** on each run | Accidental deletion of all rows if property file is empty or truncated. | Verify file size > 0 before `LOAD DATA`; keep a timestamped backup of the property file. |
| **Missing or malformed properties file** | Script aborts early, no logging. | Add sanity checks after sourcing the properties file; exit with clear error message if required variables are undefined. |

---

### 6. Running & Debugging the Script

**Typical execution (interactive):**
```bash
$ /app/hadoop_users/MNAAS/MNAAS_Property_Files/../bin/MNAAS_Child_SECSID_RGID_Dependency_Automation.sh
```
1. The script prints the current mapping list and a menu.  
2. Enter `1` to add a new child mapping or `2` to delete an existing one.  
3. Follow the prompts; the script will confirm success or display an error.

**Debugging steps:**
1. Enable Bash tracing: `set -x` (or uncomment the `#set -v` line).  
2. Verify that the properties file loads correctly: `echo $MNAAS_New_Child_SECSID_RGID_PropFile`.  
3. After a run, check the Hive table:  
   ```bash
   hive -e "SELECT * FROM ${child_scesid_rgid_mapping_tbl_name} LIMIT 10;"
   ```
4. Review the onboarding log: `cat ${MNAAS_New_Customer_Onboarding_LogPath}`.  
5. If the load fails, inspect Hive/Impala logs on the respective hosts for permission or syntax errors.

---

### 7. External Configuration & Environment Variables

| Variable (from properties) | Description | Usage |
|----------------------------|-------------|-------|
| `MNAAS_New_Child_SECSID_RGID_PropFile` | Path to the pipe‑delimited mapping file. | Read for display, appended to, and used as Hive load source. |
| `MNAAS_New_Customer_Onboarding_LogPath` | Path to an audit log file. | Append operation details for traceability. |
| `child_scesid_rgid_mapping_tbl_name` | Hive table that stores the mapping. | Target of `LOAD DATA` and Impala `REFRESH`. |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon. | Used by `impala-shell -i`. |
| *(Potential others)* | May include Hadoop user, Kerberos principal, etc. | Not referenced directly in this script but could be required by Hive/Impala commands. |

---

### 8. Suggested Improvements (TODO)

1. **Add Input Validation & Lookup** – Before committing a new record, query Hive/KYC tables to confirm that the supplied SECS IDs, RGIDs, and country values exist and are consistent. This reduces manual entry errors.

2. **Implement Concurrency Control & Backup** – Wrap the read‑modify‑write sequence in a `flock` lock file and automatically create a timestamped backup of `${MNAAS_New_Child_SECSID_RGID_PropFile}` before any modification. This protects against race conditions and accidental data loss.