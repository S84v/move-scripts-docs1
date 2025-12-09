# Summary
`Constants.java` defines a centralized, immutable collection of string literals used throughout the KLMReport component of the Move‑Mediation‑Ingest pipeline. The constants represent keywords, event types, product codes, usage models, bundle classifications, commercial codes, proposition identifiers, destination types, miscellaneous flags, date formats, report keys, probe types, and leg‑charging states. They are referenced by ETL scripts, Hive query builders, and Java utilities to ensure consistent terminology and avoid hard‑coded literals.

# Key Components
- **Class `Constants`** – Public final class containing only `public static final` string fields.
  - **Keyword constants** – `IN_BUNDLE`, `OUT_OF_BUNDLE`, `INCOMING`, `OUTGOING`.
  - **Event type constants** – `SIM`, `USAGE`.
  - **Product code constants** – `ACG`, `ACP`, `SBA`, `SBT`, `SMA`, `SMT`, `CCA`, `CCT`, `ADD`, `DAT`, `SMS`, `VOI`, `IPU`, `IPS`, `A2P`.
  - **Usage model constants** – `EUS`, `PPU`, `SHB`.
  - **Bundle type constants** – `COMBINED`, `COMBINED_MO_MT`, `SEPARATE_MO_MT`, `SEPARATE`.
  - **Commercial code type constants** – `SUBS_CODE`, `ADDON_CODE`.
  - **Proposition mapping constants** – `PROP_DEST`, `PROP_ADDON`, `PROP_SPONSOR`, `PROP_A_END_B_END`.
  - **Destination type constants** – `INTERNATIONAL`, `DOMESTIC`.
  - **Miscellaneous constants** – `SEP`, `ACTIVE`, `TOLLING`, `ADDON`, `ACTIVATION`, `ALL`, `SMS_BOTH`, `PROP_SEP`, `MO`, `MT`, `DATA`, `VOICE`, `MTO`.
  - **Date format constants** – `FILE_NAME_DATE_FORMAT`, `EVENT_DATE_FORMAT`, `PERIOD_DATE_FORMAT`.
  - **RA report key constants** – `INCDOM`, `INCINT`, `OUTDOM`, `OUTINT`.
  - **IPvProbe constants** – `UNIQUE`, `STD`.
  - **Leg‑charging constants** – `SINGLETON`, `REJECTED`, `PAIRED`, `LEG_A`, `LEG_B`.

# Data Flow
- **Inputs:** None (static literals).
- **Outputs:** Values are accessed by other Java classes, Hive query generators, and shell scripts via classpath import.
- **Side Effects:** None; immutable constants.
- **External Services/DBs/Queues:** Not directly referenced; constants may be used in DB queries, email routing, and file naming.

# Integrations
- Referenced by DAO layer (`MOVEDAO.properties` query builders) to construct dynamic Hive/SQL statements.
- Consumed by reporting utilities that generate Excel outputs and email notifications.
- Used by logging or monitoring components to tag events (e.g., `INCOMING`, `OUTGOING`).
- Integrated with shell scripts that invoke Java utilities; scripts rely on consistent constant values for parsing logs or filenames.

# Operational Risks
- **Risk:** Inconsistent updates across components if a constant value changes but dependent scripts are not recompiled.  
  **Mitigation:** Enforce versioned builds; run integration tests that validate constant usage.
- **Risk:** Hard‑coded strings may not reflect new product codes or usage models, causing silent data loss.  
  **Mitigation:** Maintain a change‑control checklist; add unit tests that verify presence of expected codes.
- **Risk:** Duplicate constant definitions elsewhere could cause divergence.  
  **Mitigation:** Centralize all literals in this class; perform static analysis for duplicate literals.

# Usage
```bash
# Compile (Maven example)
mvn clean compile

# Access from another class
String bundleType = Constants.COMBINED;
System.out.println("Bundle type: " + bundleType);
```
For debugging, set a breakpoint on any constant reference in an IDE (e.g., IntelliJ) and inspect the value.

# Configuration
- No environment variables or external config files are required; all values are compile‑time constants.
- Ensure the compiled `Constants.class` is on the classpath of any component that imports `com.tcl.move.constants.Constants`.

# Improvements
- **TODO 1:** Replace raw string literals with enumerations for groups such as `ProductCode`, `EventType`, and `DestinationType` to gain type safety.
- **TODO 2:** Externalize frequently changed values (e.g., date formats) to a properties file to avoid recompilation for minor updates.