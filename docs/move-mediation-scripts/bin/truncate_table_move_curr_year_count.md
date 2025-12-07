**High‑Level Documentation – `move-mediation-scripts/bin/truncate_table_move_curr_year_count.sh`**

---

### 1. Summary
A minimal Hive/Impala script that truncates the `move_curr_year_count` table. It is invoked by nightly orchestration wrappers (e.g., `runGBS.sh`, `runMLNS.sh`, or other “truncate_table_*.sh” scripts) to clear the yearly aggregation bucket before a fresh load of usage‑count data. The script issues a `COMMIT` to ensure the truncate is persisted and then exits.

---

### 2. Core Commands / Responsibilities
| Command | Responsibility |
|---------|----------------|
| `commit;` | Explicitly commits the transaction (required when Hive is running in transactional mode or when using Impala with ACID tables). |
| `truncate table move_curr_year_count;` | Removes all rows from the target table while preserving its schema, partitions, and metadata. |
| `exit;` | Terminates the Hive/Impala session cleanly. |

*No functions or classes are defined – the file is a pure SQL script executed by a Hive/Impala client.*

---

### 3. Inputs, Outputs & Side Effects
| Aspect | Details |
|--------|---------|
| **Inputs** | None (static script). Execution environment must provide a valid Hive/Impala connection (JDBC URL, Kerberos ticket, etc.). |
| **Outputs** | No file or console output is produced (unless the client prints the default “OK” messages). |
| **Side Effects** | All rows in `move_curr_year_count` are permanently removed. The operation is atomic due to the preceding `COMMIT`. |
| **Assumptions** | • The target table exists and is ACID‑compatible (or Impala‑compatible). <br>• The executing user has `TRUNCATE` privileges. <br>• No concurrent job is writing to the same table at the same time. |

---

### 4. Integration with Other Scripts / Components
| Caller / Down‑stream | Interaction |
|----------------------|-------------|
| `runGBS.sh`, `runMLNS.sh`, `truncate_table_move_curr_month_count.sh`, etc. | Typically invoked **before** a data‑load step that populates `move_curr_year_count`. The orchestration layer ensures the truncate runs first to avoid duplicate rows. |
| Scheduler (e.g., Oozie, Airflow, cron) | Scheduled as part of the “yearly‑count reset” workflow, often after the previous day’s processing has completed. |
| Monitoring / Alerting | Failure to truncate (non‑zero exit code) is captured by the wrapper script’s error‑handling logic and may trigger email/SDP tickets (as seen in `service_based_charges.sh`). |
| Hive/Impala Metastore | The script relies on the metastore to resolve the table definition. |

*Because the script contains only static SQL, it does **not** read any external config files, but the wrapper that launches it supplies the Hive/Impala connection parameters (e.g., `HIVE_JDBC_URL`, `HIVE_USER`, `HIVE_PASSWORD`, Kerberos principal).*

---

### 5. Operational Risks & Recommended Mitigations
| Risk | Mitigation |
|------|------------|
| **Accidental data loss** – truncating the wrong table or running out of schedule. | • Use a wrapper that validates the target table name against an allow‑list. <br>• Keep a recent backup (e.g., `INSERT OVERWRITE DIRECTORY` snapshot) before truncation. |
| **Concurrent writes** – another job may be inserting into the table while it is truncated, leading to partial loss. | • Serialize access with a lock file or a coordination service (Zookeeper, Airflow task dependencies). |
| **Missing transaction commit** – if the client runs in non‑transactional mode, the `COMMIT` may be ignored, leaving the table in an undefined state. | • Verify Hive/Impala session settings (`hive.txn.manager`, `hive.support.concurrency`). |
| **Permission errors** – the executing user lacks `TRUNCATE` rights, causing job failure. | • Include a pre‑flight `SHOW GRANT` check in the wrapper and alert on failure. |
| **Schema drift** – table renamed or dropped without updating the script. | • Add a sanity check (`DESCRIBE move_curr_year_count`) before truncation. |

---

### 6. Example Execution / Debugging Steps
1. **Direct Run (manual)**  
   ```bash
   # Assuming Hive CLI is used
   hive -f move-mediation-scripts/bin/truncate_table_move_curr_year_count.sh
   ```
   Or with Impala:
   ```bash
   impala-shell -i <impala-host> -f move-mediation-scripts/bin/truncate_table_move_curr_year_count.sh
   ```

2. **From a Wrapper (typical production flow)**
   ```bash
   # Example wrapper snippet
   LOG=/var/log/move/truncate_year_$(date +%F).log
   hive -f truncate_table_move_curr_year_count.sh >> $LOG 2>&1
   if [ $? -ne 0 ]; then
       echo "Truncate failed – see $LOG" | mail -s "Move Yearly Count Truncate Failure" ops@example.com
       exit 1
   fi
   ```

3. **Debugging**  
   * Check the Hive/Impala logs for “Truncate Table” statements.  
   * Verify the row count before and after:  
     ```sql
     SELECT COUNT(*) FROM move_curr_year_count;
     ```
   * Ensure the Kerberos ticket is valid (`klist`) if the cluster is secured.

---

### 7. Configuration & External Dependencies
| Item | Usage |
|------|-------|
| `HIVE_CONF_DIR` / `IMPALA_CONF_DIR` | Provides client configuration (metastore URIs, authentication). |
| `HADOOP_CONF_DIR` | Needed for HDFS access when the client writes temporary files. |
| Environment variables for the wrapper (e.g., `HIVE_JDBC_URL`, `HIVE_USER`) | Not referenced directly in this script but required for the client that executes it. |
| Table definition `move_curr_year_count` (Hive/Impala) | Must exist; schema is defined elsewhere in the data‑model repository. |

---

### 8. Suggested Improvements (TODO)
1. **Add safety guard** – prepend the script with a `DESCRIBE` or `SHOW CREATE TABLE` check and abort if the table is missing or the schema differs from expectations.  
2. **Introduce logging** – have the script emit a timestamped message (`INSERT INTO audit_log …`) or write to stdout so the wrapper can capture a clear “success” indicator.  

---