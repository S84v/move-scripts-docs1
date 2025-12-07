# Summary
`XLSToCSV` converts the first sheet of an Excel (.xls/.xlsx) workbook to a delimited CSV file. It reads the source file, iterates rows and cells, formats each cell value according to its type, applies CSV escaping, writes the result to a new file with the same base name and a `.csv` extension, and closes all streams.

# Key Components
- **class `XLSToCSV`**
  - `void convertXLSToCSV(String fileName, String delimiter) throws IllegalFormatException, IOException`  
    Opens the Excel file, creates a CSV writer, processes each row/cell, writes escaped values separated by `delimiter`, and closes resources.
  - `String toCSV(String value)`  
    Performs CSV escaping: doubles internal quotes, wraps the value in quotes if it contains a quote, the delimiter (`;`), or a newline.

# Data Flow
- **Input:** Path to an Excel workbook (`fileName`), delimiter string (e.g., `;`).
- **Processing:**  
  1. Open `FileInputStream` → `Workbook` via `WorkbookFactory`.  
  2. Retrieve first sheet (`getSheetAt(0)`).  
  3. Determine column count from the first row.  
  4. Iterate rows → iterate cells up to `maxColumns`.  
  5. Convert each cell to a string based on `CellType`.  
  6. Escape the string with `toCSV`.  
  7. Write delimiter‑separated values to a `BufferedWriter`.
- **Output:** CSV file written to the same directory with `.csv` extension.
- **Side Effects:** File system I/O (read Excel, write CSV). No external services, DBs, or queues.

# Integrations
- Invoked by `FileConverterUtil.main(String[] args)` which parses command‑line arguments and creates an `XLSToCSV` instance.  
- Used in the Move‑Mediation ingest pipeline when non‑CSV source data must be transformed before downstream processing.

# Operational Risks
- **Large workbook memory consumption:** `WorkbookFactory.create(is)` loads the entire workbook into memory; may cause OOM for very large files. *Mitigation:* Use streaming API (`SXSSFWorkbook`) for large files.
- **Hard‑coded sheet index (0):** Fails if source files contain data on a different sheet. *Mitigation:* Add configurable sheet selection or validation.
- **Delimiter hard‑coded to `;` in `toCSV` detection:** CSV escaping does not consider the runtime delimiter; may produce malformed CSV when a different delimiter is used. *Mitigation:* Pass delimiter to `toCSV` or adjust detection logic.
- **Resource leaks on exception:** Streams are closed only after normal completion. *Mitigation:* Use try‑with‑resources for `InputStream` and `BufferedWriter`.

# Usage
```bash
# Compile (if not using Maven/Gradle)
javac -cp ".:poi-5.x.jar:poi-ooxml-5.x.jar:commons-collections4-4.x.jar" \
    com/tcl/mnaas/noncsvdataload/XLSToCSV.java

# Run via FileConverterUtil (recommended)
java -cp ".:poi-5.x.jar:poi-ooxml-5.x.jar:commons-collections4-4.x.jar" \
    com.tcl.mnaas.noncsvdataload.FileConverterUtil /path/input.xls ";"
```
Or directly from code:
```java
XLSToCSV converter = new XLSToCSV();
converter.convertXLSToCSV("/data/input.xls", ";");
```

# Configuration
- No environment variables or external config files are referenced.  
- Delimiter is supplied at runtime via method argument or command‑line parameter.

# Improvements
1. Refactor `convertXLSToCSV` to use try‑with‑resources for deterministic stream closure and to handle exceptions cleanly.  
2. Enhance `toCSV` to accept the active delimiter, ensuring proper quoting when a delimiter other than `;` is used.