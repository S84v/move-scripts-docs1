# Summary
`build.xml` is an Apache Ant build script for the MOVE‑Mediation Ingest (MNAAS) module. It compiles Java sources, packages them into a JAR, and optionally runs the main class. The script assembles the compile‑ and runtime‑classpath from Hadoop, Hive, HBase, MapReduce, HDFS, Hadoop‑CLI, generic, and MNAAS‑specific library directories defined in a properties file (`mnaas.properties`, typically `mnaas_dev.properties` or `mnaas_prod.properties`).  

# Key Components
- **Properties**
  - `src.dir`, `build.dir`, `build.lib`, `lib`, `dist.dir`, `bin.dir`, `build.classes`, `cfg.dir` – directory locations.
  - `property file="mnaas.properties"` – loads environment‑specific library path variables (`hadoop.lib`, `hive.lib`, `hbase.lib`, `mapreduce.lib`, `hdfs.lib`, `hadoopcli.lib`, `generic.lib`, `mnaas.generic.load`).
- **Path IDs**
  - `library` – aggregates all third‑party JARs from the directories above.
  - `lib` – aggregates JARs produced in the local `${build.lib}` folder.
- **Targets**
  - `env` – prints Ant and JVM environment diagnostics.
  - `prepare` – creates build directories and timestamps the build.
  - `compiles` – compiles `src/**/*.java` to `${build.classes}` using `library` + `lib` classpaths.
  - `jar` – creates `${build.lib}/${ant.project.name}.jar` from compiled classes, excluding `*Test.class`; also bundles configuration and source files.
  - `run` – executes the class specified by `${className}` with the same classpaths.
  - `all` – composite target (`clean,jar`).
  - `clean` – removes `${build.dir}` recursively.

# Data Flow
| Phase | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| **Preparation** | None | `mkdir` creates `${build.dir}`, `${build.classes}`, `${build.lib}` | Empty directories | None |
| **Compilation** | Java source files (`src/**/*.java`), external JARs (`library`), internal JARs (`lib`) | `javac` with debug & deprecation flags | Compiled `.class` files in `${build.classes}` | Populates `${build.classes}` |
| **Packaging** | Compiled classes, config files (`cfg/`), source files (`src/`) | `jar` task (excludes `*Test.class`) | `${build.lib}/${ant.project.name}.jar` | JAR written to `${build.lib}` |
| **Execution** | Main class name (`${className}`) | `java` task (forked) | Application runtime | JVM process launched |
| **Cleaning** | Generated build artifacts | `delete` task | `${build.dir}` removed | Filesystem cleanup |

# Integrations
- **External Libraries** – Hadoop, Hive, HBase, MapReduce, HDFS, Hadoop‑CLI, generic MNAAS libs; paths supplied via `mnaas.properties`.
- **Ant Runtime** – Relies on Ant 1.x core tasks (`javac`, `jar`, `java`, `mkdir`, `delete`, `tstamp`, `echo`).
- **Configuration Files** – `mnaas.properties` (environment‑specific) and optional `cfg/` resources bundled into the JAR.
- **Potential Downstream** – The produced JAR is consumed by deployment scripts, container images, or batch job schedulers in the telecom production pipeline.

# Operational Risks
- **Missing/Incorrect Property File** – Build fails if `mnaas.properties` is absent or contains wrong paths. *Mitigation*: Validate existence of required properties early; fail fast with clear message.
- **Typographical Errors in Targets** (`projet.name`, `ant.projet.name`) – May cause undefined property warnings or incorrect JAR naming. *Mitigation*: Correct spelling to `project.name`.
- **Classpath Pollution** – Including both source and config directories in the JAR may expose internal source code. *Mitigation*: Review `fileset` inclusions; exclude source if not required.
- **Hard‑coded Memory (`maxmemory="128m"` in `run`)** – May be insufficient for large data sets. *Mitigation*: Parameterize via a property (`run.memory`) and document limits.
- **No Unit Test Execution** – Tests are excluded from the JAR but never run. *Mitigation*: Add a separate `test` target invoking JUnit.

# Usage
```bash
# Set environment (choose dev or prod properties)
export ANT_OPTS="-Dproperty.file=../mnaas_dev.properties"
# Compile and package
ant -f build.xml jar

# Run the application (requires className property)
ant -f build.xml -DclassName=com.tcl.move.mnaas.Main run
```
For a full clean‑build‑run cycle:
```bash
ant -f build.xml all
```

# Configuration
- **`mnaas.properties`** (or `mnaas_dev.properties` / `mnaas_prod.properties`)  
  Required keys: `hadoop.lib`, `hive.lib`, `hbase.lib`, `mapreduce.lib`, `hdfs.lib`, `hadoopcli.lib`, `generic.lib`, `mnaas.generic.load`.
- **Optional Ant properties**  
  - `className` – Fully qualified main class to execute.  
  - `run.memory` – Override default `maxmemory` for the `run` target.  

# Improvements
1. **Correct Property Typos** – Replace all occurrences of `projet` with `project` to ensure proper property resolution and JAR naming.
2. **Add Validation Target** – Introduce a `validate` target that checks for the presence and readability of all library directories defined in `mnaas.properties`; abort the build with a descriptive error if any are missing.