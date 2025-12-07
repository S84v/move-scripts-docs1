**File:** `move-mediation-scripts/bin/balancer_kill.sh`  

---

### 1. High‑Level Summary
`balancer_kill.sh` is a tiny utility script used in the mediation data‑move platform to forcibly terminate the **Dproc_balancer** daemon (the load‑balancing component that distributes mediation jobs across processing nodes). It is typically invoked during maintenance windows, emergency recovery, or automated restart cycles to ensure the balancer process is stopped before a fresh instance is launched.

---

### 2. Core Logic (Functions / Commands)

| Element | Type | Responsibility |
|---------|------|-----------------|
| `PID=\`pgrep -f Dproc_balancer\`` | Bash command | Retrieves the process ID(s) of any running process whose full command line contains the string `Dproc_balancer`. |
| `echo "Killing Balancer with PID $PID"` | Bash command | Logs the PID(s) that will be terminated (stdout). |
| `kill -9 $PID` | Bash command | Sends **SIGKILL** (`-9`) to the identified PID(s), guaranteeing immediate termination. |

*No functions or external libraries are defined; the script is a linear sequence of shell commands.*

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - No positional arguments.<br>- Relies on the existence of a running process whose command line contains `Dproc_balancer`.<br>- Implicitly depends on the environment’s `PATH` containing `pgrep` and `kill`. |
| **Outputs** | - Standard output: a single line `Killing Balancer with PID <pid>` (or empty if no PID found).<br>- Exit status: `0` if `kill` succeeds, non‑zero otherwise (e.g., no matching process). |
| **Side Effects** | - Sends `SIGKILL` to the target process, causing immediate termination without cleanup.<br>- May affect downstream jobs that the balancer was coordinating (e.g., incomplete mediation runs). |
| **Assumptions** | - The operator has sufficient OS permissions to send `SIGKILL` to the balancer (usually root or the same user that started it).<br>- Only one balancer instance runs per host; if multiple PIDs match, all will be killed. |
| **External Services / Resources** | - The underlying OS process table (via `pgrep`).<br>- No network, database, or file‑system interactions beyond the process table. |

---

### 4. Interaction with the Rest of the System  

| Connected Component | How `balancer_kill.sh` Relates |
|---------------------|--------------------------------|
| **Dproc_balancer** (started by scripts such as `api_med_data.sh` or other orchestration jobs) | This script is the counterpart “stop” utility; after a balancer start script runs, `balancer_kill.sh` may be called to cleanly shut it down before a redeploy or when a failure is detected. |
| **Job Scheduler / Cron** | Often scheduled as part of a maintenance window or invoked by a monitoring alert (e.g., via a wrapper script that detects a hung balancer). |
| **Logging / Monitoring** | The echo line can be captured by the caller’s logging framework; otherwise, no dedicated log file is written. |
| **Other “kill” scripts** (e.g., `balancer_restart.sh` if it exists) | May call `balancer_kill.sh` as the first step before launching a fresh balancer instance. |

*Because the script uses `pgrep -f`, any future script that changes the balancer’s command line must still contain the literal `Dproc_balancer` for this script to locate it.*

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Abrupt termination (SIGKILL)** | In‑flight mediation jobs may lose state, leading to partial data loads or duplicate processing on restart. | Prefer a graceful stop (`kill -TERM`) if the balancer supports it; add a fallback to `-9` only after a timeout. |
| **Killing the wrong process** | If another unrelated process contains the string `Dproc_balancer`, it will be terminated unintentionally. | Tighten the `pgrep` pattern (e.g., `pgrep -x Dproc_balancer` or use a PID file). |
| **No PID found** | Script exits with a non‑zero status but may be interpreted as success by callers. | Explicitly check for empty `$PID` and exit with a distinct code (e.g., `1`) and a clear message. |
| **Permission errors** | Operator may lack rights, causing the script to fail silently. | Document required OS user/group; optionally prepend `sudo` in the wrapper. |
| **Multiple instances** | All matching PIDs are killed, potentially disrupting a high‑availability setup. | Ensure only one balancer runs per host, or modify script to target a specific PID file. |

---

### 6. Running / Debugging the Script  

| Step | Command | Expected Result |
|------|---------|-----------------|
| **Basic execution** | `./balancer_kill.sh` | Prints `Killing Balancer with PID <pid>` and terminates the process. |
| **Dry‑run (no kill)** | `PID=$(pgrep -f Dproc_balancer); echo "Would kill PID $PID"` | Shows which PID would be targeted without sending a signal. |
| **Verbose debugging** | `set -x; ./balancer_kill.sh` | Bash prints each command as it runs, useful to verify the PID resolution. |
| **Check exit status** | `./balancer_kill.sh; echo $?` | `0` on successful kill, non‑zero if `kill` failed or no PID found. |
| **Log to file** | `./balancer_kill.sh >> /var/log/balancer_kill.log 2>&1` | Captures stdout/stderr for audit. |

*If the script appears to do nothing, verify that a process containing `Dproc_balancer` is actually running (`ps -ef | grep Dproc_balancer`).*

---

### 7. External Configuration / Environment Variables  

| Variable / File | Usage |
|-----------------|-------|
| None explicitly referenced | The script relies solely on the system’s `PATH` to locate `pgrep` and `kill`. |
| Implicit: `Dproc_balancer` string | Must match the exact identifier used by the balancer process. If the balancer binary or launch script is renamed, this script must be updated accordingly. |

---

### 8. Suggested Improvements (TODO)

1. **Graceful shutdown fallback** – Implement a two‑step termination: first send `SIGTERM` and wait (e.g., 30 s) for the process to exit; if still alive, then send `SIGKILL`.  
2. **PID file handling** – Adopt a standard PID file (e.g., `/var/run/dproc_balancer.pid`) and modify the script to read the PID from that file, eliminating reliance on `pgrep -f` and reducing the risk of accidental kills.  

--- 

*End of documentation.*