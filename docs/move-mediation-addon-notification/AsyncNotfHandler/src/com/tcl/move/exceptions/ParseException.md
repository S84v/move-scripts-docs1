# Summary
`ParseException` is a custom checked exception used throughout the Move‑Mediation add‑on notification service to signal errors encountered while parsing inbound JSON payloads (e.g., move‑notification messages). Throwing this exception enables upstream components to differentiate parsing failures from other error categories and to trigger appropriate error‑handling logic.

# Key Components
- **class `ParseException` extends `Exception`**
  - `private static final long serialVersionUID` – ensures serialization compatibility.
  - **Constructor `ParseException(String msg, String feed, String type)`** – creates an exception with a message; parameters `feed` and `type` are currently unused placeholders for future context enrichment.
  - **Constructor `ParseException(String msg)`** – creates an exception with only a message.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| JSON parsing (e.g., in DTO builders) | Raw JSON string, source identifier (`feed`), optional type | Detect malformed JSON, missing required fields, type mismatches | Throws `ParseException` up the call stack |
| Exception handling layer | Caught `ParseException` | Log error, optionally map to error codes, decide retry or dead‑letter | Propagates error status to monitoring/alerting, may trigger message re‑queue or DB audit entry |

No direct external services, databases, or queues are accessed by this class; it merely transports error information.

# Integrations
- **DTO parsers** (e.g., `AddonRecord` deserialization) invoke `new ParseException(...)` when JSON cannot be mapped.
- **Service layer** catches `ParseException` to convert it into higher‑level error responses or to trigger compensating actions.
- **Logging/monitoring** frameworks capture the exception stack trace for operational visibility.

# Operational Risks
- **Context loss**: `feed` and `type` parameters are ignored, reducing observability of the source of the parsing error.  
  *Mitigation*: Store these values in the exception message or dedicated fields.
- **Unchecked propagation**: If callers fail to catch `ParseException`, the thread may terminate, causing message loss.  
  *Mitigation*: Enforce handling via static analysis or wrapper utilities.
- **Serialization incompatibility**: Future changes to the class without updating `serialVersionUID` could break deserialization in distributed environments.  
  *Mitigation*: Increment `serialVersionUID` when the class structure changes.

# Usage
```java
try {
    AddonRecord record = JsonUtil.fromJson(jsonString, AddonRecord.class);
} catch (ParseException e) {
    logger.error("Failed to parse move notification for feed {}: {}", feedId, e.getMessage());
    // route to dead‑letter queue or alert
}
```
To debug, set a breakpoint in the `ParseException` constructors and verify the incoming `msg`, `feed`, and `type` values.

# configuration
No environment variables, external configuration files, or runtime parameters are referenced by this class.

# Improvements
1. **Preserve parsing context** – add private fields `feed` and `type`, expose getters, and include them in `toString()`/log output.
2. **Exception chaining** – add a constructor `ParseException(String msg, Throwable cause, String feed, String type)` that forwards the cause to `super(msg, cause)` to retain root‑cause stack traces.