# Summary
`SwapRecord` is a plain‑old‑Java‑Object (POJO) DTO used in the MOVE mediation pipeline to represent a single swap‑aggregation event. It encapsulates all fields required for persisting swap information (business unit, service, identifiers, old/new MSISDN/SIM, dates, status, etc.) and provides accessor methods, a partition‑date helper, and a `toString()` for logging.

# Key Components
- **Class `SwapRecord`**
  - Private fields: `businessUnit`, `service`, `secsId`, `commercialOffer`, `oldMSISDN`, `newMSISDN`, `oldSIM`, `newSIM`, `swapDate`, `swapIndicator`, `swapSource`, `swapStatus`.
  - Standard getters/setters for each field.
  - `getPartitionDate()` – derives `yyyy‑MM‑dd` partition key from `swapDate`.
  - Overridden `toString()` – concise, log‑friendly representation.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Ingestion | JSON payload from external swap notification service (e.g., MNAAS) | JSON → `SwapRecord` populated via Jackson/Gson (or manual mapping) | `SwapRecord` instance |
| Business Logic | `SwapRecord` object | Validation, enrichment, routing decisions | May be passed unchanged or enriched |
| Persistence | `SwapRecord` | DAO converts fields to Hive/Oracle column values; `getPartitionDate()` used for Hive partitioning | Row inserted into `swap_aggr` table |
| Logging / Monitoring | `SwapRecord` | `toString()` invoked for audit logs | Log entry |

No direct external service calls are made from this class; it is a passive data carrier.

# Integrations
- **Notification Handler** – receives swap events, creates `SwapRecord`.
- **DAO Layer** – `SwapRecordDao` (or similar) maps DTO to Hive/Oracle schemas; uses `getPartitionDate()` for Hive partition clause.
- **Batch / Streaming Jobs** – Spark, Flink, or MapReduce jobs read persisted rows; may reconstruct `SwapRecord` for downstream processing.
- **Monitoring** – log aggregation tools (e.g., ELK) ingest `toString()` output.

# Operational Risks
- **Null `swapDate`** → `getPartitionDate()` returns empty string, causing Hive partition errors. *Mitigation*: enforce non‑null date at ingestion or add fallback default.
- **Incorrect data types** (e.g., non‑numeric `secsId` in JSON) → runtime `NumberFormatException`. *Mitigation*: schema validation before DTO construction.
- **String field overflow** in DB (e.g., excessively long `commercialOffer`). *Mitigation*: define column length limits and truncate/validate upstream.

# Usage
```java
// Example: manual creation for unit test
SwapRecord rec = new SwapRecord();
rec.setBusinessUnit("BU01");
rec.setService("SwapService");
rec.setSecsId(12345);
rec.setCommercialOffer("OfferA");
rec.setOldMSISDN("8613800000000");
rec.setNewMSISDN("8613800000001");
rec.setOldSIM("8901234567890123456");
rec.setNewSIM("8901234567890123457");
rec.setSwapDate("2025-11-30T14:22:00Z");
rec.setSwapIndicator("Y");
rec.setSwapSource("API");
rec.setSwapStatus("COMPLETED");

// Persist via DAO
swapDao.insert(rec);

// Debug print
System.out.println(rec);
```

# Configuration
- No environment variables or external config files are referenced directly by this class.
- Implicit dependencies:
  - JSON mapper configuration (date format) used by the notification handler.
  - DAO connection strings (Hive/Oracle) defined elsewhere.

# Improvements
1. **Null‑Safety Enhancements** – Convert `getPartitionDate()` to return `Optional<String>` or throw a controlled exception when `swapDate` is malformed.
2. **Immutable Variant** – Introduce a builder or constructor‑based immutable version to prevent accidental field mutation after creation, improving thread‑safety in streaming contexts.