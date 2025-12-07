# Summary
`MNAAS_ShellScript.properties` supplies the Impala (Hive) connection endpoint for the **APISimCountLoader** component in production. It defines the host IP and JDBC port used to build the JDBC URL for executing Hive‑SQL statements that populate SIM inventory and activity‑count tables.

# Key Components
- **IMPALAD_HOST** – IP address of the Impala daemon for the production environment.  
- **IMPALAD_JDBC_PORT** – TCP port (21050) on which the Impala JDBC service listens.  
- **Commented DEV block** – Alternative host/port for development (currently disabled).

# Data Flow
| Stage | Description |
|-------|-------------|
| **Input** | Property file is read at runtime by the APISimCountLoader start‑up script or Java loader (`java.util.Properties`). |
| **Processing** | Values are concatenated into a JDBC URL: `jdbc:impala://<IMPALAD_HOST>:<IMPALAD_JDBC_PORT>/`. The URL is passed to the Hive/Impala client library. |
| **Output** | Established JDBC connection used to execute the SQL statements defined in `MOVEDAO.properties`. |
| **Side Effects** | None beyond establishing a network socket to the Impala service. |
| **External Services** | Impala daemon (Hive‑compatible query engine) on the specified host/port. |

# Integrations
- **APISimCountLoader** – Reads this file to configure its `ImpalaConnectionProvider`.  
- **MOVEDAO.properties** – Uses the JDBC connection created from these parameters to run the partition‑management and aggregation queries.  
- **Log4j configuration** – Logging of connection attempts is governed by `log4j.properties` in the same module.  

# Operational Risks
- **Hard‑coded IP** – Changes to the Impala cluster require manual file edit and redeployment.  
  *Mitigation*: Automate property generation from a service‑discovery layer or configuration management tool.  
- **No TLS/Authentication** – Connection is plaintext; credentials are managed elsewhere.  
  *Mitigation*: Enable Kerberos or TLS in Impala and store credentials securely (e.g., in a vault).  
- **Stale DEV block** – Commented values may cause confusion during troubleshooting.  
  *Mitigation*: Maintain separate property files per environment or use a profile selector.  

# Usage
```bash
# Example: Java loader
java -cp myloader.jar com.tcl.move.loader.APISimCountLoader \
     -Dconfig.file=./MNAAS_Property_Files/MNAAS_ShellScript.properties
```
Or within a shell script:
```bash
#!/bin/bash
source ./MNAAS_Property_Files/MNAAS_ShellScript.properties
JDBC_URL="jdbc:impala://${IMPALAD_HOST}:${IMPALAD_JDBC_PORT}/"
echo "Connecting to $JDBC_URL"
# invoke the loader binary with $JDBC_URL
```

# Configuration
- **File Path**: `move-mediation-apisimloader/APISimCountLoader/MNAAS_Property_Files/MNAAS_ShellScript.properties`  
- **Referenced By**: `APISimCountLoader` start‑up scripts, Java property loader.  
- **Environment Variables**: None required; values are read directly from the file.  

# Improvements
1. **Externalize to a secure configuration service** (e.g., Consul, Vault) and inject at runtime to avoid hard‑coded IPs and enable secret management.  
2. **Add environment selector** (e.g., `ENV=PROD|DEV`) and generate the appropriate host/port dynamically, eliminating commented blocks and reducing human error.