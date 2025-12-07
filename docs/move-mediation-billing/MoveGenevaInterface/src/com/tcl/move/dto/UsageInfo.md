# Summary
`UsageInfo` is a POJO that encapsulates aggregate usage metrics for a SIM or a group of SIMs within the Move‑Geneva mediation pipeline. It stores total usage, a flag indicating the presence of out‑of‑bundle usage, the number of SIMs contributing to the aggregate, and the allowed usage quota (SMS‑only). The object is populated by DAO/processing components after CDR aggregation and consumed downstream by billing calculators to determine chargeable usage.

# Key Components
- **Class `UsageInfo`**
  - `Float totalUsage` – cumulative usage amount (KB/Minutes/SMS) for the scope.
  - `Boolean outBundlePresent` – true if any portion of the usage exceeds the bundle.
  - `Long SimCount` – count of distinct SIM identifiers represented.
  - `Float allowedUsage` – quota allocated for SMS usage; used for over‑quota checks.
  - Getter/Setter methods for each field.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| DAO / Aggregation | Raw CDR records (DB query, file read) | Sum usage per SIM/group, detect out‑of‑bundle, count SIMs, fetch SMS quota | `UsageInfo` instance populated via setters | None (pure data transformation) |
| Billing Engine | `UsageInfo` object | Evaluate against product rules, compute charges | Billing records, charge events | May trigger downstream persistence or messaging |

# Integrations
- **Data‑Access Layer** – retrieves CDRs from relational DB or Hadoop store; creates `UsageInfo`.
- **Billing Calculation Module** – consumes `UsageInfo` to apply bundle rules, interacts with `SimProduct`, `RARecord`, `SplitRecord`.
- **Reporting / Auditing** – may serialize `UsageInfo` for logs or metrics dashboards.

# Operational Risks
- **Null fields** – missing setters may propagate `null` leading to NPE in downstream calculations. *Mitigation*: enforce non‑null defaults or validation after construction.
- **Floating‑point precision** – using `Float` can cause rounding errors for large usage volumes. *Mitigation*: switch to `BigDecimal` or `double` with defined rounding policy.
- **Inconsistent naming** – field `SimCount` starts with uppercase, violating Java conventions; may cause reflection‑based mapping errors. *Mitigation*: rename to `simCount` and adjust getters/setters.

# Usage
```java
// Example in a unit test or debugging session
UsageInfo usage = new UsageInfo();
usage.setTotalUsage(1250.5f);
usage.setOutBundlePresent(Boolean.TRUE);
usage.setSimCount(3L);
usage.setAllowedUsage(500.0f);

// Pass to billing calculator
BillingResult result = billingEngine.calculateCharges(usage);
System.out.println(result);
```

# configuration
No environment variables or external configuration files are referenced directly by this class. Configuration is handled upstream (e.g., DB connection strings, quota values) and injected via setters.

# Improvements
1. **Rename `SimCount` to `simCount`** and adjust accessor methods to follow JavaBean naming conventions, eliminating potential reflection mismatches.
2. **Replace `Float` with `BigDecimal`** for `totalUsage` and `allowedUsage` to improve precision and support monetary‑grade calculations.