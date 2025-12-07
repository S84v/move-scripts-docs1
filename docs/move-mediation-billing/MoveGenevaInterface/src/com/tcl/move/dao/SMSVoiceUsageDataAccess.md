# Summary
`SMSVoiceUsageDataAccess` is a DAO class that retrieves SMS and voice traffic usage and CDR counts from a Hive‑based mediation database for billing. It provides methods to fetch aggregate usage per SECS/Proposition, usage per product (by zone, sponsor, or multiple propositions), and file‑level usage records. Errors are wrapped in custom `DBDataException` and `DBConnectionException` types.

# Key Components
- **Class `SMSVoiceUsageDataAccess`** – Extends `UsageDataAccess`; central DAO for SMS/Voice usage.
- **`fetchSMSVoiceTrafficInfo(String usageType)`** – Returns a `Map<String,Vector<Float>>` keyed by `SECS__PROPOSITION__COMMERCE` with aggregated usage and CDR counts for all customers.
- **`getSMSVoiceUsageForProduct(Long secsCode, String proposition, List<String> propList, int destination, String usageType, String commercialCodeType)`** – Retrieves usage for a specific SECS, proposition(s), zone, and usage type.
- **`getSMSVoiceUsageForProduct(Long secsCode, String proposition, List<String> propList, String sponsor, String usageType, String commercialCodeType)`** – Same as above but filtered by sponsor code.
- **`getSMSVoiceUsageForMultiProp(Long secsCode, List<String> propList, String commercialCodeType, String usageType)`** – Retrieves usage for a SECS across multiple propositions.
- **`getFileLevelCountUsage(UsageProduct product, String direction, String destinationtype)`** – Returns a list of `RARecord` containing file‑wise usage and CDR counts.
- **`getStackTrace(Exception e)`** – Utility to convert stack trace to `String`.
- **Dependencies** – `JDBCConnection` for Hive connections, `BillingUtils` for dynamic SQL placeholders, `Constants`, `RARecord`, `UsageProduct`, Log4j logger.

# Data Flow
| Method | Input Parameters | DB Interaction | Output | Side Effects |
|--------|------------------|----------------|--------|--------------|
| `fetchSMSVoiceTrafficInfo` | `usageType` (SMS/VOICE) | Executes `sms.voice.fetch` SQL on Hive, binds `usageType` | `Map<String,Vector<Float>>` (key = `secsId|proposition|commercialCode`) | Opens/closes JDBC resources |
| `getSMSVoiceUsageForProduct` (zone) | SECS, proposition or list, zone, usageType, commercialCodeType | Executes `sms.voice.zone.fetch` with dynamic `IN` clause | `Vector<Float>` (7 usage + 7 CDR values) | Same as above |
| `getSMSVoiceUsageForProduct` (sponsor) | SECS, proposition or list, sponsor, usageType, commercialCodeType | Executes `sms.voice.sponsor.fetch` | `Vector<Float>` (same layout) | Same as above |
| `getSMSVoiceUsageForMultiProp` | SECS, proposition list, commercialCodeType, usageType | Executes `sms.voice.multprop.fetch` | `Vector<Float>` (same layout) | Same as above |
| `getFileLevelCountUsage` | `UsageProduct` (contains secs, proposition(s), destination, etc.), direction, destinationtype | Executes `sms.voice.filewise.fetch` | `List<RARecord>` (filename, calltype, usage, cdr, secsId) | Same as above |

All methods convert voice usage from seconds to minutes (rounded) when `usageType == Constants.VOICE`.

# Integrations
- **Hive Database** – Accessed via `JDBCConnection.getConnectionHive()`. Queries are defined in `MOVEDAO.properties` (keys: `sms.voice.fetch`, `sms.voice.zone.fetch`, etc.).
- **BillingUtils** – Generates comma‑separated `?` placeholders for `IN` clauses.
- **Constants** – Provides static strings for call types, directions, and conversion logic.
- **RARecord / UsageProduct** – DTOs used by downstream billing components.
- **Logging** – Log4j used for audit trails; messages include query parameters and result counts.

# Operational Risks
- **Resource Leaks** – Failure to close `ResultSet`, `PreparedStatement`, or `Connection` on exceptions could exhaust DB connections. Mitigation: Ensure `finally` block always executes; consider try‑with‑resources.
- **SQL Injection** – Dynamic `IN` clause built from `propList` uses placeholders, safe. However, misuse of `MessageFormat` could introduce formatting issues if placeholders mismatch.
- **Performance** – Large result sets loaded into memory (`Vector<Float>` and `Map`). May cause OOM for high‑volume months. Mitigation: Stream processing or pagination.
- **Incorrect Voice Conversion** – Rounding after division by 60 may lose precision; verify business rule.
- **Hard‑coded SQL Strings** – Changes to schema require property updates; lack of compile‑time validation.

# Usage
```java
SMSVoiceUsageDataAccess dao = new SMSVoiceUsageDataAccess();

// Example: fetch all SMS usage
Map<String, Vector<Float>> smsMap = dao.fetchSMSVoiceTrafficInfo(Constants.SMS);

// Example: fetch voice usage for a specific SECS, zone 2, proposition list
List<String> props = Arrays.asList("PROP1","PROP2");
Vector<Float> voiceVec = dao.getSMSVoiceUsageForProduct(
        123456L, null, props, 2, Constants.VOICE, Constants.SUBSCRIPTION);
```
For debugging, enable Log4j DEBUG level to view generated SQL and parameter values.

# Configuration
- **Resource Bundle** `MOVEDAO.properties` – Contains SQL templates referenced by keys:
  - `sms.voice.fetch`
  - `sms.voice.zone.fetch`
  - `sms.voice.sponsor.fetch`
  - `sms.voice.multprop.fetch`
  - `sms.voice.filewise.fetch`
- **Environment** – Hive JDBC URL, credentials, and driver are resolved inside `JDBCConnection`.
- **Constants** – `Constants.SMS`, `Constants.VOICE`, `Constants.INCOMING`, `Constants.MT`, etc., must be available on classpath.

# Improvements
1. **Refactor to try‑with‑resources** – Replace manual close blocks with `try (Connection c = …; PreparedStatement ps = …; ResultSet rs = …) {}` to guarantee resource release.
2. **Introduce a unified result DTO** – Replace repetitive `Vector<Float>` constructions with a typed `UsageMetrics` object containing named fields (combined, incoming, outgoing, etc.) for readability and type safety.