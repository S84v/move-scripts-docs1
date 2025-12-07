**File:** `move-mediation-scripts\bin\compression.sh`

---

### 1. High‑Level Summary
`compression.sh` is a housekeeping utility that scans a predefined set of staging directories for compressed files (typically `.gz`) and decompresses them in‑place using `gunzip`. It logs its activity (including file names and counts) to a dated log file under `/app/hadoop_users/mkoripal/`. The script is normally invoked as a pre‑processing step before downstream ingestion/orchestration jobs (e.g., the various `*_loading.sh` or API‑related scripts) that expect plain‑text input files.

---

### 2. Key Components

| Component | Type | Responsibility |
|-----------|------|----------------|
| `CompressionLog` | Variable | Path to the log file; includes current date (`compression.log_YYYY-MM-DD`). |
| `dirs` | Array | List of glob patterns pointing to the staging directories and file types to be processed. |
| `generic_compression()` | Function | Core loop: for each directory pattern, count files, and if any exist, iterate over them (oldest first) logging the action and invoking `gunzip` on each file. |
| `set -x` | Shell option | Enables command tracing for debugging; all executed commands are echoed to stderr (captured in the log). |
| `exec 2>>$CompressionLog` | Redirection | Redirects *all* subsequent stderr output (including the trace from `set -x`) to the log file. |

---

### 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Inputs** | - Files matching the globs in `dirs` (e.g., `/Input_Data_Staging/LTESOR_InputData/*.log`, `/Input_Data_Staging/DSX_Inputdata/*.diam`, etc.).<br>- Implicitly expects those files to be gzip‑compressed (`*.gz`). |
| **Outputs** | - Decompressed files placed in the same directory (original `.gz` files are removed by `gunzip`). |
| **Side Effects** | - Writes operational logs to `/app/hadoop_users/mkoripal/compression.log_YYYY-MM-DD`.<br>- May generate error messages on `gunzip` failures (also logged). |
| **Assumptions** | - `gunzip` is installed and in the PATH.<br>- The script runs under a user with read/write permissions on the staging directories and the log directory.<br>- No concurrent processes are modifying the same files while this script runs. |

---

### 4. Interaction with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **Ingestion / Loading scripts** (e.g., `api_med_data.sh`, `addon_subs_aggr_loading.sh`, `cdr_buid_search_loading.sh`) | These downstream jobs read the same staging directories *after* this script has decompressed the files. They typically assume plain‑text input, so `compression.sh` must run **before** them. |
| **Data producers** (e.g., upstream ETL jobs, SFTP pullers) | Produce gzip‑compressed files into the staging directories. `compression.sh` is the consumer that prepares the data for the next stage. |
| **Logging infrastructure** | The log file may be harvested by a monitoring system (e.g., Splunk, ELK) to track success/failure of decompression. |
| **Cron / Scheduler** | Usually scheduled (or triggered by an orchestrator such as Oozie/Airflow) to run once per batch window, ensuring files are ready for the next step. |

*Note:* The script does **not** read any external configuration files or environment variables beyond the hard‑coded paths. If the environment changes (e.g., staging root moves), the script must be edited directly.

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Non‑gzip files matched** (e.g., a plain `.log` that is not compressed) | `gunzip` will exit with error, potentially halting the loop. | Add a file‑type guard (`[[ $file == *.gz ]]`) before invoking `gunzip`. |
| **Large volume of files** causing long runtimes or resource contention | Batch window may be missed; downstream jobs could start on partially decompressed data. | Parallelize safely (e.g., `xargs -P`), or limit the number of files per run. |
| **Log file growth** (no rotation) | Disk fill on the Hadoop user partition. | Implement log rotation (e.g., `logrotate` config) or truncate the log at start of each run. |
| **Missing or inaccessible directories** | Script will emit errors and may exit with non‑zero status. | Pre‑check directory existence and permissions; exit gracefully with a clear message. |
| **Concurrent modifications** (another process writing a file while it is being gunzipped) | Corrupted output or partial decompression. | Use file‑locking or a “ready” flag (e.g., move files to a `ready/` sub‑dir after successful decompression). |
| **`set -x` verbosity** in production | Excessive noise in logs, making troubleshooting harder. | Make tracing optional via an environment variable (e.g., `DEBUG=1`). |

---

### 6. Running / Debugging the Script

1. **Standard execution** (typically invoked by scheduler):  
   ```bash
   /app/hadoop_users/mkoripal/compression.sh
   ```
   - Exit code `0` → success; non‑zero → failure (check `$CompressionLog`).

2. **Manual run with debug output** (already enabled by `set -x`):  
   ```bash
   bash -x /app/hadoop_users/mkoripal/compression.sh 2>&1 | tee /tmp/compression_debug.log
   ```
   - Review the printed commands and any error messages.

3. **Check results**:  
   - Verify that `.gz` files in the target directories have been replaced by their decompressed counterparts.  
   - Inspect the log file:  
     ```bash
     less /app/hadoop_users/mkoripal/compression.log_$(date +%F)
     ```

4. **Troubleshooting tips**:  
   - If the log shows “gunzip: not a gzip file”, confirm the file extension or content.  
   - Ensure the user running the script has write permission on the staging directories.  
   - Use `strace -e open,unlink -p <pid>` on a running instance to see file operations if needed.

---

### 7. External Config / Environment Dependencies

| Item | Usage |
|------|-------|
| Hard‑coded paths (`/Input_Data_Staging/...`) | Directories scanned for compressed files. |
| Log directory (`/app/hadoop_users/mkoripal/`) | Destination for the runtime log. |
| `gunzip` binary | Decompression tool; must be present in `$PATH`. |
| No environment variables are referenced; any change requires script edit. |

---

### 8. Suggested Improvements (TODO)

1. **Parameterize directories & log location**  
   - Accept command‑line arguments or source a small config file (e.g., `compression.conf`) so the script can be reused across environments without code changes.

2. **Add safety checks & robust file filtering**  
   - Only process files ending with `.gz`.<br>
   - Wrap `gunzip` in a conditional that logs success/failure per file and continues on error rather than aborting the whole batch.

---