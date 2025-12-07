**File:** `move-mediation-scripts\bin\MNAAS_New_Customer_Onboarding.sh`

---

## 1. High‑Level Summary
This interactive Bash utility is used by operations staff to add or remove a telecom customer from the MNAAS (Mobile Network Analytics & Assurance Suite) onboarding catalogue. It updates a central “customer data” property file that maps **customer name → SECS ID → source feed** and, when adding a new customer, loads a suite of per‑customer configuration property files (traffic, activation, actives, tolling) and echoes the derived associative‑array values for verification. Down‑stream batch jobs and Hadoop‑based pipelines read the same property file to drive CDR ingestion, transformation, and reporting for each onboarded customer.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| `MNAAS_Create_New_Customer` | *Interactively* collect `customer name`, `SECS ID`, and `source feed`; verify uniqueness; append a pipe‑delimited record to `${MNAAS_New_Customer_Data_PropFile}`; source per‑customer configuration property files; display the populated associative‑array variables for the new customer. |
| `MNAAS_Delete_Existing_Customer` | *Interactively* collect the same three identifiers; locate the exact line in `${MNAAS_New_Customer_Data_PropFile}`; delete it with `sed`; display the updated list. |
| **Main script block** | Shows the current customer list, prompts the operator to choose *Add* (1) or *Delete* (2), and dispatches to the appropriate function. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **User Input** | Interactive `read` prompts for: <br>• `new_customer` / `delete_customer` (string) <br>• `secs_id` (string) <br>• `src_feed` (one of `SNG_01`, `HOL_01`, `HOL_02`) |
| **Configuration Files (inputs)** | • `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_New_Customer_Onboarding.properties` (defines `MNAAS_New_Customer_Data_PropFile`, `MNAAS_New_Customer_Onboarding_LogPath`, etc.) <br>• Six additional property files sourced only on *add*: <br> `MNAAS_create_customer_cdr_traffic_files.properties` <br> `MNAAS_create_customer_cdr_activation_files.properties` <br> `MNAAS_create_customer_cdr_actives_files.properties` <br> `MNAAS_create_customer_cdr_tolling_files.properties` <br> `MNAAS_CommonProperties.properties` |
| **Data Store (outputs / side effects)** | • Appends a line `customer|secs_id|src_feed` to `${MNAAS_New_Customer_Data_PropFile}` (the master onboarding catalogue). <br>• Appends the new customer name to `${MNAAS_New_Customer_Onboarding_LogPath}` (audit log). <br>• On delete, removes the exact matching line from the same catalogue file. |
| **Console Output** | Lists current customers, echoes the associative‑array values for the newly added customer, and prints status messages. |
| **Assumptions** | • The script runs on a host with Bash, `sed`, `grep`, and `cat`. <br>• The operator has write permission on the property files and log path. <br>• The property files define associative arrays indexed by the *customer name* (e.g., `MNAAS_Customer_SECS[$new_customer]`). <br>• No concurrent executions modify the same property file (no file‑locking mechanism). |

---

## 4. Integration Points & System Connections

| Connected Component | How the connection is made |
|---------------------|----------------------------|
| **Down‑stream Hadoop / Spark jobs** | They read `${MNAAS_New_Customer_Data_PropFile}` to discover which customers to process for CDR ingestion, traffic aggregation, billing, etc. |
| **Other “create‑customer‑*” scripts** | The six per‑customer property files sourced after a successful add contain configuration (column mappings, output directories, backup locations) that are consumed by scripts such as `MNAAS_MSISDN_Level_Daily_Usage_Aggr.sh`, `MNAAS_Monthly_Billing_Export.sh`, etc. |
| **Operational monitoring / audit** | The log file `${MNAAS_New_Customer_Onboarding_LogPath}` can be tailed or ingested by a log‑aggregation system (e.g., Splunk) to track onboarding events. |
| **Potential UI wrapper** | Though not present, a higher‑level orchestration tool (e.g., Ansible, Jenkins) could invoke this script non‑interactively by feeding stdin or by refactoring to accept CLI arguments. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Human input error** (typos, wrong SECS ID, wrong feed) | Incorrect onboarding data → downstream jobs fail or mis‑route CDRs. | Add validation (regex, lookup against master SECS/Feed tables) and a confirmation prompt before committing. |
| **Concurrent modifications** (two operators editing the same file) | Race condition → lost updates or duplicate entries. | Implement a file lock (e.g., `flock`) around read‑modify‑write sections. |
| **Missing or malformed property files** | Script aborts, leaving the system in an inconsistent state. | Verify existence and syntax of each sourced file at script start; exit with clear error code if any are absent. |
| **Sed pattern mismatch** (extra whitespace, case differences) | Delete operation may remove the wrong line or none at all. | Trim whitespace, use case‑insensitive matching, and optionally backup the file before editing. |
| **No audit trail beyond simple log** | Difficult to trace who added/removed a customer. | Enrich log entries with timestamp, operator username (`$USER`), and host identifier. |
| **Hard‑coded feed list** | Adding a new feed later requires script change. | Externalize allowed feed values into a config file or environment variable. |

---

## 6. Running & Debugging the Script

### 6.1 Prerequisites
```bash
# Verify required files exist
ls /app/hadoop_users/MNAAS/MNAAS_Property_Files/*.properties
```
- Execute as a user with write permission on the property files and log directory.
- Ensure Bash is invoked (script uses `#!/bin/bash` implicitly via `set -x`).

### 6.2 Typical Execution
```bash
cd /path/to/move-mediation-scripts/bin
bash MNAAS_New_Customer_Onboarding.sh
```
- The script prints the current customer list, then prompts:
  ```
  1. Adding a New Customer
  2. Deleting Existing Customer
  Enter your choice:
  ```
- Choose `1` or `2` and follow the subsequent prompts.

### 6.3 Debug Mode
The script already enables tracing with `set -x`. For deeper inspection:
```bash
bash -x MNAAS_New_Customer_Onboarding.sh 2> /tmp/onboard_debug.log
```
- Review `/tmp/onboard_debug.log` for variable expansions and command execution flow.

### 6.4 Non‑Interactive Testing (quick sanity check)
```bash
# Simulate user input via a here‑document
bash MNAAS_New_Customer_Onboarding.sh <<EOF
1
TestCustomer
SEC123
SNG_01
EOF
```
- Adjust the values as needed; useful for automated CI checks.

---

## 7. External Configuration & Environment Variables

| Variable (defined in property files) | Purpose |
|--------------------------------------|---------|
| `MNAAS_New_Customer_Data_PropFile` | Path to the master onboarding catalogue (pipe‑delimited). |
| `MNAAS_New_Customer_Onboarding_LogPath` | Path to the audit log where new customer names are appended. |
| `MNAAS_Customer_SECS`, `FEED_CUSTOMER_MAPPING`, `MNAAS_customer_backup_dir`, `MNAAS_windows_job_mapping`, `CUSTOMER_MAPPING` | Associative arrays used by downstream jobs; displayed after a successful add. |
| `MNASS_Create_Customer_*_files_*` (e.g., `columns`, `column_position`, `out_dir`, `backup_dir`, `filter_condition`) | Per‑customer file‑generation parameters for traffic, activation, actives, and tolling pipelines. |
| `MNAAS_CommonProperties` | Global settings shared across all customer‑specific scripts (e.g., Hadoop cluster URLs, DB connection strings). |

*All of the above are sourced from the property files listed at the top of the script.*

---

## 8. Suggested Improvements (TODO)

1. **Add a non‑interactive mode** – accept command‑line arguments (`--add|--delete`, `--customer`, `--secs`, `--feed`) so the script can be driven by orchestration tools and scheduled jobs without manual stdin.
2. **Implement file locking** – wrap modifications to `${MNAAS_New_Customer_Data_PropFile}` with `flock` to prevent race conditions when multiple operators run the script concurrently.

---