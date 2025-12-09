# Summary
`GetSchema` provides a utility method to load an Avro schema from a class‑path resource and expose it as a reusable `org.apache.avro.Schema` instance. In the MOVE‑KYC pipeline it supplies the schema required by `CustomRecordBuilder` to translate CSV rows into Avro `GenericData.Record`s, which are later persisted by `CustomParquetWriter`.

# Key Components
- **Class `GetSchema`**
  - `private static final Logger LOGGER` – SLF4J logger for diagnostic messages.
  - `public static Schema SCHEMA` – mutable static holder for the last loaded schema (shared across threads).
  - `public static Schema getCustomSchema(String schemaLocation)` – loads the schema file located at `schemaLocation`, parses it, stores it in `SCHEMA`, and returns the parsed `Schema` object.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | `schemaLocation` (e.g., `"/schemas/kyc.avsc"`), a class‑path resource path | Opens an `InputStream` via `GetSchema.class.getResourceAsStream(schemaLocation)` inside a try‑with‑resources block. | `InputStream` (closed automatically). |
| 2 | `InputStream` | Parses the Avro schema using `new Schema.Parser().parse(new File(schemaLocation))`. (Current implementation incorrectly parses a `File` rather than the stream.) | `Schema` object assigned to static `SCHEMA`. |
| 3 | – | Returns the `Schema` instance to the caller. | Caller receives a ready‑to‑use Avro schema. |
| 4 | – | On `IOException` (e.g., resource not found) logs an error and throws a `RuntimeException`. | Exception propagates; error is logged. |

External dependencies: class‑path resources (schema files), Apache Avro library, SLF4J logging.

# Integrations
- **`CustomRecordBuilder`** calls `GetSchema.getCustomSchema(schemaPath)` to obtain the Avro schema for CSV‑to‑Avro conversion.
- **`CustomParquetWriter`** indirectly depends on the schema via the records produced by `CustomRecordBuilder`.
- The utility is packaged in the `parquetConverter` JAR and is available to any component within the MOVE‑KYC subsystem that requires schema loading.

# Operational Risks
- **Incorrect parsing logic** – Using `new File(schemaLocation)` treats the argument as a filesystem path, which fails when the schema is packaged inside the JAR. Results in `FileNotFoundException` and pipeline stoppage.
- **Static mutable state** – `SCHEMA` is shared across threads; concurrent calls may overwrite each other, leading to nondeterministic behavior.
- **Unchecked RuntimeException** – Propagates up the call stack without graceful degradation; may cause batch job failure.
- **Logging of raw path** – May expose internal file structure in logs if not sanitized.

**Mitigations**
- Replace file‑based parsing with stream‑based parsing (`new Schema.Parser().parse(inStream)`).
- Remove the static `SCHEMA` field or make it immutable (`final`) per request.
- Wrap the exception in a domain‑specific checked exception or return `Optional<Schema>` to allow caller handling.
- Ensure log messages do not expose sensitive path information.

# Usage
```java
// Example: load an Avro schema located at src/main/resources/schemas/kyc.avsc
String schemaPath = "/schemas/kyc.avsc";
Schema kycSchema = GetSchema.getCustomSchema(schemaPath);

// Verify schema loaded
if (kycSchema == null) {
    throw new IllegalStateException("Failed to load Avro schema");
}
```
*Debugging tip*: Run the class in a unit test with the schema on the test class‑path; verify that `kycSchema.getFields().size()` matches expectations.

# Configuration
- **Environment variables**: None.
- **Configuration files**: The method expects a class‑path resource path (e.g., `/schemas/kyc.avsc`). The actual schema file must be packaged with the application JAR under `src/main/resources`.

# Improvements
1. **Correct schema parsing** – Refactor `getCustomSchema` to parse directly from the `InputStream`:
   ```java
   try (InputStream in = GetSchema.class.getResourceAsStream(schemaLocation)) {
       return new Schema.Parser().parse(in);
   }
   ```
2. **Eliminate static mutable state** – Remove `public static Schema SCHEMA` or replace it with a thread‑local cache if repeated loads are required, ensuring thread safety and predictability.