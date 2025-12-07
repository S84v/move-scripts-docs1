# Summary
The `build.xml` defines an Apache Ant build for the **MOVE‑Mediation Ingest (MNAAS)** module. It compiles Java sources, assembles a JAR containing the compiled classes and configuration files, and provides a runnable target. The script resolves external Hadoop‑related libraries and project‑specific JARs, enabling the module to be built and executed in the telecom production move pipeline.

# Key Components
- **Properties**
  - `src.dir`, `build.dir`, `build.lib`, `lib`, `dist.dir`, `bin.dir`, `build.classes`, `cfg.dir` – directory locations.
  - `mnaas.properties` – external property file loaded at build time.
- **Path definitions**
  - `library` – aggregates Hadoop, Hive, HBase, MapReduce, HDFS, Hadoop CLI, and generic library JARs via `${hadoop.lib}`, `${hive.lib}`, etc.
  - `lib` – includes JARs produced in `${build.lib}`.
- **Targets**
  - `env` – prints environment diagnostics.
  - `prepare` – creates build directories and timestamps.
  - `compiles` – compiles Java sources (`src.dir`) to `${build.classes}` using the `library` and `lib` classpaths.
  - `jar` – packages compiled classes, configuration files, and source files into `${build.lib}/${ant.project.name}.jar`, excluding test classes.
  - `run` – executes the main class `${className}` with the assembled classpath.
  - `all` – depends on `clean` and `jar`; performs a full clean‑build‑package cycle.
  - `clean` – removes `${build.dir}` and all generated artifacts.

# Data Flow
- **Inputs**
  - Java source files under `src/`.
  - Configuration files under `cfg/`.
  - External JARs from directories referenced by `${hadoop.lib}`, `${hive.lib}`, `${hbase.lib}`, `${mapreduce.lib}`, `${hdfs.lib}`, `${hadoopcli.lib}`, `${generic.lib}`.
  - Property values from `mnaas.properties`.
- **Outputs**
  - Compiled `.class` files in `${build.classes}`.
  - Packaged JAR `${build.lib}/${ant.project.name}.jar` placed in `dist/` (via `jar` target).
- **Side Effects**
  - Creation of build directories.
  - Deletion of `${build.dir}` on `clean`.
  - Console logging of environment information (`env` target).
- **External Services/DBs/Queues**
  - None directly; the built JAR may later interact with Hadoop/HBase/Hive services at runtime.

# Integrations
- **Library Integration**
  - Pulls Hadoop ecosystem JARs (HDFS, MapReduce, Hive, HBase) and generic project libraries into the compile and runtime classpaths.
- **Runtime Integration**
  - The `run` target expects a system property `className` (or Ant property) that points to the main class of the MNAAS component, which will in turn use services such as `NotificationService`, `SwapService`, and `JSONParseUtils` from the broader MOVE‑mediation codebase.
- **Configuration Integration**
  - Loads `mnaas.properties` for environment‑specific values (e.g., library paths, Hadoop configuration).

# Operational Risks
- **Missing or mismatched external library paths** – build fails if any `${*lib}` property points to a non‑existent directory. *Mitigation*: validate paths before execution; provide default fallback locations.
- **Hard‑coded memory limit in `run` target (`maxmemory="128m"` )** – may be insufficient for large payload processing. *Mitigation*: expose memory limit as a configurable property.
- **Typographical errors in target names (`compiles`, `projet.name`)** – can cause confusion or incorrect artifact naming. *Mitigation*: rename targets to conventional names (`compile`) and correct property references.
- **Excluding only `**/*Test.class`** – other test artifacts (e.g., resources) may be unintentionally packaged. *Mitigation*: refine exclusion patterns or use a dedicated test output directory.

# Usage
```bash
# Compile sources
ant compiles

# Package JAR (creates dist/${ant.project.name}.jar)
ant jar

# Run the main class (set className property)
ant -DclassName=com.tcl.move.Main run
```
For a full clean‑build‑package cycle:
```bash
ant all
```

# Configuration
- **Property Files**
  - `mnaas.properties` – defines `${hadoop.lib}`, `${hive.lib}`, `${hbase.lib}`, `${mapreduce.lib}`, `${hdfs.lib}`, `${hadoopcli.lib}`, `${generic.lib}` and any other environment‑specific values.
- **Ant Properties**
  - `className` – fully qualified main class for the `run` target.
  - Optional overrides for directory properties (`src.dir`, `build.dir`, etc.) via `-D` command‑line arguments.

# Improvements
1. **Correct target and property names** – rename `compiles` to `compile`, fix typo `projet.name` to `project.name`, and ensure consistent naming across targets.
2. **Parameterize JVM memory** – replace hard‑coded `maxmemory="128m"` with a property `${jvm.maxmem}` that can be set externally, enabling scaling for production workloads.