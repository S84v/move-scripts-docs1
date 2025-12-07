# Summary
`BundleSplitRecord` is a plain‑old‑Java‑Object (POJO) that encapsulates the result of splitting a usage bundle into in‑bundle and out‑of‑bundle portions for a SIM. It stores aggregate counts, usage volumes, country lists, flags, and the detailed CDR/RA records required by the Move‑Geneva billing engine to compute per‑product charges.

# Key Components
- **Class `BundleSplitRecord`**
  - Fields  
    - `totalBundle` – size of the original bundle (float).  
    - `inCountryList` – list of country codes considered “in‑bundle”.  
    - `simLevelAggrNeeded` – flag indicating whether aggregation must be performed per SIM.  
    - In‑bundle CDR counters (`inbDomIncCDRCount`, `inbIntIncCDRCount`, `inbDomOutgCDRCount`, `inbIntOutgCDRCount`).  
    - In‑bundle usage totals (`inbDomIncUsage`, `inbIntIncUsage`, `inbDomOutgUsage`, `inbIntOutgUsage`).  
    - `outbRecords` – list of `OutBundleInfo` objects (out‑of‑bundle summary per product/zone).  
    - `inBundleRecs` – map **SIM‑ID → List\<RARecord\>** for in‑bundle detailed records.  
    - `outBundleRecs` – list of `RARecord` for out‑of‑bundle detailed records.  
  - Accessors (getter/setter) for each field.  
  - `toString()` – human‑readable dump of all fields.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| DAO (`UsageDataAccess` → `createCDRSubset` etc.) | Product definition, billing period, SIM list | Queries Hive tables, builds `RARecord` objects | Populates `inBundleRecs` and `outBundleRecs` in a `BundleSplitRecord` instance | Hive connection opened/closed, SQL exceptions wrapped |
| Bundle split logic (`UsageDataAccess` methods) | `BundleSplitRecord` (empty) + raw CDR list | Iterates CDRs, classifies by direction, geography, updates counters, aggregates usage, populates `outbRecords` | Fully populated `BundleSplitRecord` | None (pure computation) |
| Billing engine (`MoveGenevaInterface`) | Populated `BundleSplitRecord` | Uses counters/usage to compute chargeable units, applies pricing rules | Charge lines, invoice data | Writes to billing DB, logs |

# Integrations
- **DAO Layer** – `BundleSplitRecord` is instantiated and filled by `UsageDataAccess` (and indirectly by `SMSVoiceUsageDataAccess`).  
- **DTOs** – `OutBundleInfo` and `RARecord` are sibling DTOs used to store detailed out‑of‑bundle and raw record data.  
- **Billing Engine** – Consumes the POJO to calculate in‑bundle vs out‑of‑bundle charges per SIM/product.  
- **Logging** – `toString()` is used in debug logs throughout the pipeline.  
- **Persistence** – The final charge data derived from this object is persisted via other DAO classes (e.g., `SubscriptionDataAccess`).

# Operational Risks
- **Memory Bloat** – `inBundleRecs` and `outBundleRecs` hold full CDR objects; large SIM volumes can cause OOM. *Mitigation*: stream processing, limit batch size, or purge intermediate records after aggregation.  
- **Floating‑Point Precision** – Usage fields are `Float`; cumulative rounding errors may affect billing accuracy. *Mitigation*: switch to `BigDecimal` or `double` with defined rounding.  
- **Thread Safety** – POJO is mutable and not synchronized; concurrent reuse can corrupt data. *Mitigation*: instantiate per thread or enforce immutability after population.  
- **Null Handling** – Collections are initialized, but `inCountryList` may remain null if not set, causing NPE in downstream code. *Mitigation*: default to empty list in getter or constructor.

# Usage
```java
// Example in a unit test or debugging session
BundleSplitRecord split = new BundleSplitRecord();
split.setTotalBundle(5000f);
split.setInCountryList(Arrays.asList("US","CA"));
split.setSimLevelAggrNeeded(true);

// Populate with DAO result (pseudo)
List<RARecord> raw = usageDao.fetchRecords(product, period, simId);
BundleSplitter.split(raw, split); // static helper that fills counters

System.out.println(split); // invokes toString()
```
To debug, set logger level to DEBUG for the package `com.tcl.move.dao` and inspect the `BundleSplitRecord` printed after each processing step.

# Configuration
- No environment variables or external config files are referenced directly by this class.  
- Dependent configuration resides in DAO resource bundles (`MOVEDAO.properties`) that define Hive queries used to generate the `RARecord` list fed into this DTO.

# Improvements
1. **Immutable Design** – Replace mutable fields with final attributes and provide a builder pattern; eliminates thread‑safety concerns.  
2. **Precision Upgrade** – Change usage fields from `Float` to `BigDecimal` (or `double` with explicit rounding) to ensure billing‑grade accuracy.