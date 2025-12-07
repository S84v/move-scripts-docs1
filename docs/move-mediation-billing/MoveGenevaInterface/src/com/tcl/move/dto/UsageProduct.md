# Summary
`UsageProduct` is a POJO that models a SIM‑level usage‑related product record retrieved from the Geneva view. It stores product metadata, bundle definitions, proposition details, and provides logic to normalize bundle values (UOM conversion, usage‑model scaling) and to compute total bundle quantities used by the Move‑Geneva billing pipeline for charge calculation.

# Key Components
- **Class `UsageProduct`**
  - Fields: product metadata (`productName`, `effectFrom/To`, `productCode`, `commercialCode`, `secsCode`, `eventSource`, `uom`, `usageModel`, `bundleType`), bundle amounts (`combinedBundle`, `incomingBundle`, `outgoingBundle`, `domInCBundle`, `domOutGBundle`, `intInCBundle`, `intOutGBundle`), proposition data (`propDestination`, `propAddon`, `propSponsor`, `proposition`, `multiProposition`, `propList`), in‑country profile (`inCountryProfId`, `inCountry`, `inCountryList`), aggregation flags (`simLevelAggrNeeded`, `activeOrTolling`), SIM count, and total bundle aggregates.
  - Getters/Setters: standard accessors with null‑safety and conversion logic (e.g., `setCombinedBundle`, `setDomOutGBundle` use `BillingUtils.uomToMB` for DAT products).
  - Helper methods:
    - `getCallType()` – derives “DATA”, “VOICE”, or “SMS” from `productCode`.
    - `getBundleType()` / `setBundleType()` – default bundle type based on product code.
    - `calculateAndSetBundles()` – computes total bundle values according to `bundleType` and `usageModel`.
    - `getFullBundle(Float bundle)` – applies usage‑model scaling (EUS, PPU, SHB).
    - `addCountry(String)`, `addProposition(String)` – collection helpers.
  - `toString()` – diagnostic representation.

# Data Flow
- **Input:** Populated by DAO layer reading rows from the Geneva view (SQL result set). Fields are set via setters, often with raw string values that are parsed/converted.
- **Processing:** 
  1. UOM conversion (`BillingUtils.uomToMB`) normalizes data bundles to MB.
  2. `calculateAndSetBundles()` derives total bundle amounts based on `bundleType` and `usageModel`.
- **Output:** The fully populated `UsageProduct` instance is passed downstream to billing calculators that determine chargeable usage per SIM or group.
- **Side Effects:** None (pure data object). No external services invoked directly; only static utility `BillingUtils`.

# Integrations
- **DAO Layer:** `UsageProduct` is instantiated and populated from the Geneva view query results.
- **Constants:** Relies on `com.tcl.move.constants.Constants` for product codes, bundle type identifiers, usage model strings.
- **Utility:** Uses `com.tcl.move.utils.BillingUtils` for UOM conversion.
- **Billing Engine:** Consumed by downstream billing components that compute activation, subscription, management, and add‑on charges.

# Operational Risks
- **Null Pointer Risks:** Several methods assume `productCode` and `uom` are non‑null (e.g., `setCombinedBundle`). Mitigation: validate inputs at DAO level or add defensive checks.
- **Incorrect UOM Conversion:** If `BillingUtils.uomToMB` does not handle unexpected units, bundle values may be wrong. Mitigation: unit tests covering all supported UOMs.
- **Bundle Type Mismatch:** `calculateAndSetBundles()` throws generic `Exception` for unknown bundle types, potentially causing job failure. Mitigation: replace with a specific checked exception and log context.
- **Parsing Errors:** `setPropDestination(String)` may throw `NumberFormatException` on malformed input. Mitigation: catch and log, default to `null`.

# Usage
```java
// Example in a unit test or debugging session
UsageProduct up = new UsageProduct();
up.setProductCode(Constants.DAT);
up.setUom("GB");
up.setCombinedBundle(2.5f); // 2.5 GB
up.setUsageModel(Constants.EUS);
up.setSimCount(10);
up.setBundleType(Constants.COMBINED);
up.calculateAndSetBundles(); // totalCombinedBundle = 2.5 * 1024 * 10
System.out.println(up);
```
Debug by stepping through `calculateAndSetBundles()` and verifying `total*Bundle` fields.

# configuration
- No environment variables or external config files are referenced directly. Relies on:
  - `Constants` class (static values).
  - `BillingUtils` (static conversion logic).
  - Database view schema (DAO mapping).

# Improvements
1. **Replace generic `Exception` with domain‑specific `BundleCalculationException`** and include detailed error context (productCode, bundleType).  
2. **Add comprehensive null‑checks and input validation** in setters (especially `productCode`, `uom`, `combinedBundle`) to prevent runtime NPEs and parsing failures.