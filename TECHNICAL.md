# **TECHNICAL DOCUMENTATION: Pandoc Plugin Fork **

## **📌 Project Overview**

**Project Name**: Obsidian Pandoc Plugin (Aliased Citations Fork)

**Description**: This is a forked version of the [Obsidian Pandoc Plugin](https://github.com/OliverBalfour/obsidian-pandoc). The primary goal of this fork is to change the Obsidian preview (onscreen rendering) to correctly display aliased Pandoc-style citation links. Specifically, links like `[[@citationKey|Alias Text]]` should render only as `Alias Text` in the Obsidian preview. Non-aliased links like `[[@citationKey]]` should remain unchanged for preview.  

Scope clarification: This change is preview-only. It does not alter export behavior or how Pandoc processes citations. You can continue using this plugin for exports as before (including Pandoc 3.7.0.2); the alias handling is only for what you see inside Obsidian.

**Primary Language(s)**: TypeScript  

**Environment**: Obsidian Desktop App (macOS / Linux / Windows)  

**Dependencies**: Node.js, npm, Obsidian API, esbuild. All required packages are listed in package.json.

## **🗂️ Directory Structure**

The project follows the standard structure for an Obsidian community plugin.

/obsidian-pandoc  
│  
├── main.ts             # Plugin entry point, commands, settings, lifecycle. (Add preview post-processor here)  
├── renderer.ts         # Export-time Markdown→HTML conversion and post-processing (unchanged by this feature)  
├── pandoc.ts           # Manages the execution of the Pandoc CLI (exports).  
├── settings.ts         # Settings UI for the plugin.  
├── global.ts           # Shared interfaces and helpers.  
├── styles/             # CSS injected into HTML exports.  
├── main.js             # Compiled bundle that Obsidian runs (generated).  
├── manifest.json       # Plugin metadata for Obsidian.  
├── package.json        # Project dependencies and scripts.  
├── tsconfig.json       # TypeScript compiler configuration.  
├── esbuild.config.mjs  # Build script configuration.  
└── README.md           # Original project README.

**`main.ts`**: Initializes the plugin, adds the export commands to the command palette, and creates the settings tab.  

**`renderer.ts`**: Handles export-time pre-processing. It uses Obsidian's renderer to create HTML for export and cleans that HTML. Note: the aliased-citation preview behavior is not implemented here.  

Preview behavior change location: Implemented via a Markdown post-processor registered in `main.ts`, which runs when Obsidian renders a note in preview mode.

**`pandoc.ts`**: A wrapper for spawning the Pandoc executable with the correct arguments for file conversion.  
**settings.ts**: Defines the user-facing settings panel within Obsidian.  

**`main.js`**: This is the bundled, production-ready file that Obsidian actually loads. It is generated from the TypeScript source files during the build process (npm run dev or npm run build). You should not edit this file directly.

## **⚙️ Setup Instructions**

These instructions are for setting up a local development environment to modify and build the plugin.

### **1. Fork & Clone the Repository**

Fork the original repository and clone your version into your Obsidian vault's plugin directory.

# Navigate to your vault's plugin folder  
`cd /path/to/your/vault/.obsidian/plugins/`

# Clone your forked repository  
```bash
git clone https://github.com/sjelms/obsidian-pandoc.git  
cd obsidian-pandoc
```

### **2. Install Dependencies**

Install the required Node.js packages using npm.
```bash
npm install
```

### **3. Build the Plugin**

Run the development script. This will compile the TypeScript source into main.js and will automatically re-compile whenever you save a change to a source file.
```bash
npm run dev
```
Keep this script running in a terminal window while you are making edits.

### **4. Enable in Obsidian**

* Open Obsidian.  
* Go to ``Settings > Community plugins``.  
* Disable the official "Pandoc Plugin" if you have it installed.  
* Enable your forked version of the plugin (it will have the same name).  
* To see changes after saving a file, you may need to reload Obsidian (you can find "Reload app without saving" in the Command Palette).

## **🧱 Project Architecture**

### **🌀 Workflow Overview**

There are two relevant flows: preview rendering (onscreen) and exporting.

- **Preview (onscreen) flow**:  
  1) When Obsidian renders a note in preview, the plugin’s Markdown post-processor (registered in `main.ts`) runs.  
  2) It finds aliased citation links of the form `[[@citekey|Alias]]` and replaces the link node with its alias text so the preview displays only `Alias`.  
  3) No export behavior is affected by this change.  

- **Export flow** (unchanged):  
  1) Export commands are registered in `main.ts`.  
  2) Depending on settings, the plugin either exports from Markdown or renders to HTML via `renderer.ts` and then calls Pandoc via `pandoc.ts`.  
  3) Citation rendering for exports still depends on Pandoc options and bibliography configuration; this feature does not modify export.

## **🧼 Post-Processing Rules**

This section details the specific modification to achieve the desired aliased citation behavior.

### **✨ Target: Aliased Citation Link Preview**

**Goal**: In Obsidian preview, display only the alias text for `[[@citationKey|Alias]]` links, leaving non-aliased citations `[[@citationKey]]` unchanged for preview. This avoids awkward renders while editing, without changing export behavior.

### **🔎 Implementation Details (Preview)**

Implement a Markdown post-processor in `main.ts` that adjusts the preview DOM:

1. **File to Modify**: `main.ts`  
2. **Location**: Inside `onload()`, register a Markdown post-processor using `this.registerMarkdownPostProcessor(...)`.  
3. **Logic**: For each preview container `el`, find anchors representing internal links and, when the link target appears to be a citation key (`@key`) and the link is aliased (display text differs from the target), replace the link element with its alias text.

Example snippet:
```ts
this.registerMarkdownPostProcessor((el) => {
  const anchors = el.querySelectorAll('a.internal-link, a');
  anchors.forEach((a: HTMLAnchorElement) => {
    const display = a.textContent?.trim() ?? '';
    const target = (a.getAttribute('data-href') || a.getAttribute('href') || '').trim();
    if (!display || !target) return;
    // Handle URL encoding and both @ and %40 (encoded @)
    const decoded = decodeURIComponent(target);
    const isCitation = decoded.startsWith('@') || target.startsWith('%40');
    const isAliased = decoded !== display;
    if (isCitation && isAliased) {
      // Replace link with plain text alias in preview
      a.replaceWith(document.createTextNode(display));
    }
  });
});
```

Notes:
- This affects only the preview DOM. It does not mutate the underlying Markdown or export pipeline.
- Using `replaceWith(document.createTextNode(...))` is safer than `outerHTML = ...` for avoiding unintended HTML parsing.

## **🧪 Testing & Validation**

**Testing Method**: Manual testing in Obsidian preview (and optionally verify exports are unchanged).  
**Test Case 1: Aliased Citation (preview)**  
  Input: `This is an aliased citation [@Felstead2016-ut|Learning Outside The Formal System]`  
  Expected (preview): `This is an aliased citation Learning Outside The Formal System`  
**Test Case 2: Non-aliased Citation (preview)**  
  Input: `This is a standard citation [@Felstead2016-ut]`  
  Expected (preview): No change introduced by this feature (Obsidian shows the link normally).  
**Test Case 3: Internal Link (preview)**  
  Input: `See [[My Other Note|Custom Title]]`  
  Expected (preview): Unchanged; feature only targets citation-shaped links.

Exports: Confirm that exporting behaves exactly as before. If you need citation rendering like `(Author Year)` in exported files, configure Pandoc citeproc and bibliography in “Extra Pandoc arguments” (outside the scope of this feature).

## **📦 Future Enhancements**

**Make Behavior Optional**: Add a toggle in `settings.ts` to enable/disable the preview-only alias handling (e.g., “Flatten aliased citations in preview”). Wrap the post-processor logic behind this setting.

**Code Hygiene**: Remove the unused/broken import `import { outputFormats } from 'pandoc';` in `renderer.ts` to avoid build issues.

## **📚 References**

**Original Plugin Repository**: [https://github.com/OliverBalfour/obsidian-pandoc](https://github.com/OliverBalfour/obsidian-pandoc)  
**Obsidian API Documentation**: [https://github.com/obsidianmd/obsidian-api](https://github.com/obsidianmd/obsidian-api)  
**Pandoc Documentation**: [https://pandoc.org/](https://pandoc.org/)

## **🧑‍💻 Author & Contact**

**Author**: Stephen Elms  
**GitHub**: [https://github.com/sjelms/obsidian-pandoc](https://github.com/sjelms/obsidian-pandoc)
