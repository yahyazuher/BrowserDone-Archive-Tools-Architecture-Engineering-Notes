# Architecture Decision Log (ADR) — BrowserDone

This document records key architectural, design, and scoping decisions made in the BrowserDone project.

---

## Decision Status Legend
- **APPROVED:** Explicitly approved by the project owner and implemented.
- **PROPOSED:** Documented based on technical investigation; awaiting owner approval.
- **REJECTED:** Considered and rejected with documented rationale.

---

## ADR-01: 100% Client-Side Processing Architecture
- **Date:** 2026-09-01
- **Status:** **APPROVED**
- **Context:** BrowserDone must differentiate itself from conventional online converters and archive utilities that upload user files to cloud servers.
- **Decision:** All file conversions, extractions, compressions, and audio/video processing must execute entirely client-side using JavaScript, Canvas, Web Audio, and WebAssembly. No user file data or metadata may be sent to any backend server.
- **Consequences:** Strong privacy guarantee; zero backend storage infrastructure costs; operations are strictly bound by the user's browser memory and CPU capabilities.

---

## ADR-02: Web Worker Offloading for Heavy Operations
- **Date:** 2026-09-10
- **Status:** **APPROVED**
- **Context:** Parsing large archives (ZIP, 7Z, TAR, RAR) or decoding media frames freezes the browser main thread and degrades UI responsiveness.
- **Decision:** Encapsulate all archive operations, GZIP compression, and WASM loaders inside dedicated Web Workers (`archiveWorker.ts`, `imageWorker.ts`, etc.) communicating asynchronously via `postMessage`.
- **Consequences:** UI remains responsive with progress reporting; requires careful handling of Structured Clone limitations when transferring non-standard object properties.

---

## ADR-03: Shared Archive Explorer Component with Virtual Tree
- **Date:** 2026-10-05
- **Status:** **APPROVED**
- **Context:** Five separate archive extraction tools (ZIP, TAR, TAR.GZ, RAR, 7Z) previously used divergent UI paradigms for displaying extracted files.
- **Decision:** Unify all five extractors around a single shared component (`ArchiveExplorer.tsx`) backed by an in-memory virtual directory tree (`archiveTree.ts`) providing breadcrumb navigation, search, and multi-selection.
- **Consequences:** Consistent UX across all archive tools; centralized maintenance of tree state and download actions.

---

## ADR-04: Folder Traversal & Directory Picker Scope in Shared Dropzone
- **Date:** 2026-10-14
- **Status:** **APPROVED**
- **Context:** The ZIP, TAR, and TAR.GZ compression tools accepted only flat files and failed when users dragged folders or attempted to preserve directory structures.
- **Decision:**
  1. Enhance `src/core/ui/Dropzone.tsx` with a `directoryUpload` prop, adding an explicit "Select Folder" button and a native `<input type="file" webkitdirectory directory multiple>`.
  2. Implement recursive drag-and-drop directory traversal via `traverseDroppedItems` and `webkitGetAsEntry` with paginated `readEntries()` support.
  3. Scope folder selection support to archive creation tools, preserving existing single-file and flat multi-file behaviors for other tools.
- **Consequences:** Enables full folder compression across ZIP, TAR, and TAR.GZ; maintains backward compatibility with image and media dropzones.

---

## ADR-05: Handling Duplicate Paths & Conflicts in Archive Creation & Extraction
- **Date:** 2026-10-15
- **Status:** **APPROVED**
- **Context:**
  - The original design in `buildArchiveVirtualTree` and `buildZipFromArchiveFiles` treated duplicate complete paths as a fatal error (`throw new ArchiveTreeError`), crashing the Explorer UI with `Duplicate file path detected`.
  - In incident RSK-02, creating an archive from files containing numbered basenames (e.g. `Screenshot.png` and `Screenshot (1).png`) caused `sanitizeArchiveFiles` to generate colliding paths, rendering the resulting archive unviewable in `ExtractTar`.
- **Decision:**
  1. **Creation Stage:** Update `sanitizeArchiveFiles` to use a reservation loop `while (takenPaths.has(candidate))` to ensure output files are strictly unique. Pass explicit serializable `filePaths` arrays to Web Workers.
  2. **Extraction Stage:** Replace fatal exceptions in `buildArchiveVirtualTree` and `buildZipFromArchiveFiles` with deterministic auto-disambiguation (`name (counter).ext`), resolving both exact duplicate paths and file/directory ancestor conflicts.
  3. **ZIP Hierarchy Parity:** Delegate `buildZipFromArchiveFiles` path planning directly to `buildArchiveVirtualTree`, ensuring complete identity between UI displayed paths, internal node identities, and generated ZIP entries.
  4. **Quarantine Transparency:** When `allowRecovery` is enabled, route malformed paths to `_quarantine/` and render an accessible security notice banner in `ArchiveExplorer.tsx` without exposing unescaped raw markup.
- **Alternatives Considered:**
  - *Keep fatal exception:* Rejected because legitimate archives (such as append-mode Unix TAR archives or archives created by external utilities) become completely inaccessible to users.
  - *Silently overwrite:* Rejected because overwriting causes silent data loss.
- **Consequences:** Prevents UI denial of service; guarantees users can view and download all files from archives containing duplicate paths or ancestor conflicts without data loss.

---

## ADR-06: Verification Tiering & Explicit OS Dialog Limitations
- **Date:** 2026-10-15
- **Status:** **APPROVED**
- **Context:** Automated testing and browser automation tools (Playwright/Puppeteer) simulate file and folder inputs via CDP (`setInputFiles`), but cannot interact with native operating system window manager modals (such as the Windows File Explorer folder picker dialog).
- **Decision:** Formally categorize verification into three tiers:
  1. *Tier 1: Unit & Integration (Vitest)*
  2. *Tier 2: Headless Browser Automation (Playwright)*
  3. *Tier 3: Manual OS Verification*
  A feature relying on native OS file pickers may not be marked as fully manually verified until an actual human operator conducts physical interaction on the host OS.
- **Consequences:** Prevents false verification claims while maintaining high automated verification coverage.
