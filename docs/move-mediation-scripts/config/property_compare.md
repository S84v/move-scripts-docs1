**File:** `move-mediation-scripts/config/property_compare.sh`

---

### 1. High‑Level Summary
`property_compare.sh` is a utility script that scans the *removable* property file (`MNAAS_Removalble.properties`) and, for each key that also exists in the common property set (`MNAAS_CommonProperties.properties`), removes the corresponding entry from a dated backup copy (`MNAAS_Removalble.properties_050219`). The script prints a diagnostic line for each matched key and mutates the backup file in‑place. It is used during configuration clean‑up to prune duplicate or superseded entries before a production run of the MOVE data‑move pipelines.

---

### 2. Key Components & Responsibilities

| Component | Type | Responsibility |
|-----------|------|----------------|
| **Main `for` loop** | Bash control structure | Iterates over every non‑comment, `key=value` line in `MNAAS_Removalble.properties`. |
| **`eqcount` check** | Inline command | Guarantees the line contains an `=` before processing. |
| **`varname` extraction** | `cut` | Isolates the property key (text before the first `=`). |
| **Presence test in common properties** | `grep -n … | wc -l` | Determines whether the key also appears in `MNAAS_CommonProperties.properties`. |
| **Line‑number lookup in dated backup** | `grep -n … | cut -d':' -f1 | head -1` | Finds the first line number of the matching key in `MNAAS_Removalble.properties_050219`. |
| **Diagnostic `echo`** | Bash builtin | Emits `"$var : $varname: $line"` to STDOUT for audit. |
| **In‑place deletion** | `sed -i "$line"'d' …` | Removes the identified line from the dated backup file. |

*No functions or external libraries are defined; the script is a linear sequence of shell commands.*

---

### 3. Inputs, Outputs & Side Effects

| Item | Description |
|------|-------------|
| **Input Files** | `MNAAS_Removalble.properties` (source list), `MNAAS_CommonProperties.properties` (reference list), `MNAAS_Removalble.properties_050219` (target backup to be edited). |
| **Output** | Diagnostic lines printed to STDOUT, e.g., `someKey=value : someKey: 42`. |
| **Side Effects** | The backup file `MNAAS_Removalble.properties_050219` is **mutated in‑place** (lines deleted). No other files are touched. |
| **Assumptions** | • All three files exist in the current working directory.<br>• Property lines follow `key=value` without embedded `=` in the key.<br>• No spaces around `=` (script does not trim whitespace).<br>• The backup file is writable and the user has permission to run `sed -i`. |

---

### 4. Interaction with the Rest of the MOVE System

| Connection | How it is used |
|------------|----------------|
| **Configuration generation scripts** (e.g., `MNAAS_Traffic_Adhoc_Aggr.sh`) | May invoke `property_compare.sh` to prune duplicate entries before packaging a final `*.properties` bundle for a downstream job. |
| **Orchestration wrappers** (e.g., nightly `*_refresh.sh` scripts) | Could call this script as a pre‑step to ensure that the *removable* property set does not contain keys already present in the common set, preventing accidental overrides. |
| **Version‑control / audit pipelines** | The diagnostic echo output can be captured into a log file that is later reviewed by operators to verify which keys were removed. |
| **Downstream Java/Python loaders** | Those loaders read the final `MNAAS_Removalble.properties` (or the cleaned backup) to obtain runtime configuration; therefore this script indirectly influences the configuration passed to the data‑move jobs. |

*Exact call sites are not present in the supplied history, but the naming convention (`property_compare.sh`) and its location in `config/` strongly suggest it is part of the configuration‑pre‑processing stage.*

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Destructive edit without backup** | Permanent loss of lines if the script is run against the wrong file or with a bug. | Add `sed -i.bak` to keep a timestamped backup; or copy the file to a staging area before modification. |
| **Missing or malformed input files** | Script will exit with non‑zero status or silently skip entries, leading to stale configuration. | Pre‑flight checks: `[[ -f … ]] && [[ -r … ]]` for each file; abort with clear error message if any check fails. |
| **Keys containing spaces or multiple `=` characters** | `cut -d'=' -f1` will truncate incorrectly, causing false positives/negatives. | Use more robust parsing (e.g., `IFS='=' read -r key value <<< "$line"`). |
| **Race conditions when multiple processes run simultaneously** | Concurrent `sed -i` edits could corrupt the backup file. | Serialize execution via a lock file (`flock`) or ensure the script is only invoked from a single‑instance orchestrator. |
| **Unquoted variable expansions** | Word‑splitting may break on keys with special characters. | Quote all variable expansions (`"$varname"` etc.). |

---

### 6. Typical Execution & Debugging Steps

1. **Run the script** (from the `config/` directory):  
   ```bash
   ./property_compare.sh
   ```
2. **Capture diagnostics** (optional):  
   ```bash
   ./property_compare.sh > property_compare.log 2>&1
   ```
3. **Debug mode** – add `set -x` at the top of the script to trace each command.  
4. **Verify the backup** – after execution, inspect the modified file:  
   ```bash
   grep -n -E "$(awk -F'=' '{print $1}' MNAAS_CommonProperties.properties)" MNAAS_Removalble.properties_050219
   ```
   No matches should remain.
5. **Rollback if needed** – restore from the backup created by `sed -i.bak` (if the mitigation is applied) or from a manual copy.

---

### 7. External Config / Environment Dependencies

| Item | Usage |
|------|-------|
| `MNAAS_Removalble.properties` | Source list of removable properties. |
| `MNAAS_CommonProperties.properties` | Reference list; any key present here triggers deletion from the dated backup. |
| `MNAAS_Removalble.properties_050219` | Target backup file that is edited in‑place. |
| **No environment variables** are referenced directly; the script relies solely on the presence of the three files in the current working directory. |

---

### 8. Suggested Improvements (TODO)

1. **Add safety backup & atomic edit** – modify the `sed` command to `sed -i.bak "$line"'d' …` and optionally replace the file via a temporary copy to guarantee atomicity.
2. **Refactor to use a while‑read loop with proper quoting** – improves handling of spaces, special characters, and eliminates the need for multiple `grep`/`cut` pipelines, e.g.:

   ```bash
   while IFS='=' read -r key _; do
       [[ -z "$key" || "$key" =~ ^# ]] && continue
       if grep -q "^${key}=" MNAAS_CommonProperties.properties; then
           line=$(grep -n "^${key}=" MNAAS_Removalble.properties_050219 | cut -d: -f1 | head -1)
           echo "$key : $line"
           sed -i.bak "${line}d" MNAAS_Removalble.properties_050219
       fi
   done < <(grep -v '^#' MNAAS_Removalble.properties | grep '=')
   ```

These changes increase robustness, readability, and traceability for production operators.