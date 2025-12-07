**File:** `move-Inventory\move-MSPS\.idea\markdown-navigator.xml`  

---

## 1. High‑Level Summary
This XML file stores the **Markdown Navigator** plugin configuration for the IntelliJ IDEA project that contains the *move‑MSPS* inventory‑migration scripts. It defines how markdown files are parsed, rendered, and exported inside the IDE (preview layout, Pegdown extensions, parser options, HTML/CSS generation, and link‑mapping). The settings are used by developers and CI pipelines that render markdown documentation (e.g., README, design docs) to verify formatting before committing changes.

---

## 2. Key Sections & Their Responsibilities  

| Section (XPath) | Responsibility | Important Values |
|-----------------|----------------|------------------|
| `/project/component[@name='MarkdownProjectSettings']/PreviewSettings` | Controls the split‑view preview behaviour inside the IDE. | `splitEditorLayout="SPLIT"`, `splitEditorPreview="PREVIEW"`, `synchronizePreviewPosition="true"` |
| `/project/component/ParserSettings/PegdownExtensions/option` | Enables/disables specific Pegdown markdown extensions. | `ABBREVIATIONS=false`, `TABLES=true`, `TASKLISTITEMS=true`, `WIKILINKS=true` |
| `/project/component/ParserSettings/ParserOptions/option` | Fine‑grained parser flags (CommonMark, GFM, emoji handling, etc.). | `COMMONMARK_LISTS=true`, `EMOJI_SHORTCUTS=true`, `GFM_TABLE_RENDERING=true` |
| `/project/component/HtmlSettings` | Determines which HTML fragments (header/footer/body) are injected when rendering markdown to HTML. | All header/footer/body sections are empty → pure content rendering. |
| `/project/component/CssSettings` | CSS handling for the preview. Uses the UI scheme; no external stylesheet is referenced. |
| `/project/component/HtmlExportSettings` | Default export parameters when a user triggers “Export to HTML”. All paths are empty, meaning export uses the project’s default location. |
| `/project/component/LinkMapSettings` | Holds custom link‑map definitions; currently empty. |

These sections are **static configuration** – they do not contain executable code but affect how the IDE presents and processes markdown files that are part of the broader move‑MSPS codebase.

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Inputs** | - Markdown source files located anywhere under the `move-MSPS` project (e.g., `README.md`, design docs). <br> - Implicit IDE state (project root, UI theme). |
| **Outputs** | - Rendered markdown preview pane inside IDEA. <br> - Optional HTML export (if a user triggers it) using the configured generator/provider. |
| **Side‑Effects** | - None that affect production runtime; only developer‑experience artifacts. <br> - Potential commit of this file to VCS may propagate UI preferences to all team members. |
| **Assumptions** | - The team uses the **Markdown Navigator** plugin (v2+). <br> - All developers run IntelliJ IDEA with the same plugin version. <br> - No external CSS/JS resources are required for preview/export. |

---

## 4. Integration Points with the Rest of the System  

| Connected Component | How the Connection Manifests |
|---------------------|------------------------------|
| **Other `.idea` files** (e.g., `jsLibraryMappings.xml`, `misc.xml`) | Together they define the complete IDE project configuration. Changes in this file may need to be coordinated with those files to avoid conflicting preview or export settings. |
| **Documentation generation pipelines** (e.g., a CI job that runs `markdown-navigator` headless export) | If the pipeline invokes the plugin’s command‑line exporter, it will read this XML to decide which extensions to enable and which CSS to apply. |
| **Source code repositories** (Git) | The file is version‑controlled; any change is propagated to all clones, influencing how team members view markdown docs. |
| **Developer workflow** | When a developer opens a markdown file, the preview pane follows the layout (`SPLIT`) and parsing rules defined here. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Inconsistent markdown rendering across team members** | Documentation may look different, causing confusion. | Enforce a *single* committed version of this file; add a pre‑commit hook that validates the file against a known baseline (e.g., diff against `master`). |
| **Accidental commit of personal UI preferences** (e.g., zoomFactor, split layout) | Noise in VCS, unnecessary merge conflicts. | Document that only the *ParserSettings* and *HtmlExportSettings* sections are intended to be shared; keep UI‑specific flags (`zoomFactor`, `splitEditorLayout`) at defaults. |
| **Missing plugin on a developer’s machine** | Markdown files will render with default IDEA markdown, potentially missing extensions (tables, task lists). | Add the plugin version to the project’s `README` and include it in the onboarding checklist. |
| **Headless export failure in CI** (if used) | Build may break when generating HTML docs. | Ensure CI environment installs the same plugin version and provides a minimal `markdown-navigator.xml` with required extensions; add a sanity test that renders a sample markdown file. |

---

## 6. Running / Debugging the Configuration  

1. **Local verification**  
   - Open any `.md` file in the `move-MSPS` project.  
   - Verify the preview pane appears side‑by‑side (`SPLIT`) and that tables, task lists, and wiki‑links render correctly.  
   - Adjust a setting (e.g., toggle `TABLES` to `false`), save the XML, and observe the preview update instantly.  

2. **Export to HTML (manual)**  
   - Right‑click the markdown file → *Markdown Navigator* → *Export to HTML*.  
   - The export respects the `<HtmlExportSettings>` section; by default it writes to the same directory as the source file.  

3. **CI / headless test** (if applicable)  
   - Use the plugin’s command‑line interface: `java -jar markdown-navigator-cli.jar --config .idea/markdown-navigator.xml --input docs/README.md --output target/README.html`.  
   - Verify the generated HTML contains the expected elements (tables, task list checkboxes).  

4. **Debugging**  
   - If a markdown feature is not rendering, check the corresponding `<option name="…">` under `PegdownExtensions` or `ParserOptions`.  
   - Ensure the plugin version supports the requested extension (some older versions lack `WIKILINKS`).  

---

## 7. External Configurations, Environment Variables & Files Referenced  

| Reference | Purpose |
|-----------|---------|
| `providerId` values (`com.vladsch.idea.multimarkdown.editor.swing.html.generator`, etc.) | Identify the built‑in HTML generator and CSS provider used by the plugin. |
| `cssUri`, `cssText` (both empty) | No external stylesheet is applied; rendering uses the IDE UI theme. |
| No environment variables are read by this file. |
| No other external files are referenced; all paths (`parentDir`, `targetDir`, etc.) are left empty, meaning defaults are used. |

If a team decides to adopt a custom stylesheet, they would populate `cssUri` or `cssText` and commit the stylesheet alongside the project.

---

## 8. Suggested TODO / Improvements  

1. **Lock the plugin version in project documentation** – add a `plugins.xml` entry (or a `README` section) specifying the exact Markdown Navigator version to avoid drift.  
2. **Create a minimal CI validation step** – a small script that renders a sample markdown file using the committed `markdown-navigator.xml` and fails the build if the output does not contain expected HTML fragments (e.g., `<table>`).  

--- 

*End of documentation.*