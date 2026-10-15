# BrowserDone — Project Engineering State

**Current As Of:** 2026-10-15  
**Repository Root:** `pnj2jpeg/`  
**Overall Status:** Active Engineering / Archive Engine Refinement (Not yet marked as production launch-ready)

---

## 1. System Overview & Architecture

BrowserDone is a client-side suite of privacy-first file utilities executing entirely in the user's browser without server-side processing.

- **Frontend Core:** React 19, TypeScript, Vite, CSS Modules.
- **Localization:** `react-i18next` with bilingual support (English and Arabic with native `dir="rtl"` layout).
- **Execution Model:** Computationally intensive tasks (compression, extraction, transcode) are delegated to dedicated Web Workers (`archiveWorker.ts`, `imageWorker.ts`, `audioWorker.ts`) to maintain UI thread responsiveness.
- **Engines:**
  - **Archive Engine:** `fflate` (ZIP / GZIP), POSIX UStar stream reader/writer (`tarUtils.ts`), `7z-wasm` (7Z), WASM RAR unpacker (`rarAdapter.ts`).
  - **Image Engine:** Canvas API, `OffscreenCanvas`, WebCodecs `ImageDecoder`, `gifenc`.
  - **Audio Engine:** Web Audio API, Web Workers, `wavesurfer.js`.
  - **Video Engine:** `@ffmpeg/ffmpeg` WebAssembly.

---

## 2. Module Implementation & Verification Status

| Module | Sub-Features | Implementation Status | Automated Testing | Browser Automation | Manual OS Verification |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Archive: Extraction** | ZIP, TAR, TAR.GZ, RAR, 7Z | **Implemented** | **Passed** | **Passed** | **Pending** (real-world user archives ongoing) |
| **Archive: Creation** | Compress to ZIP, Create TAR, Create TAR.GZ | **Implemented** | **Passed** | **Passed** | **Pending** |
| **Archive: Folder Traversal** | `Dropzone.tsx` directory upload & recursive drag-and-drop | **Implemented** | **Passed** | **Passed** (Playwright Chromium) | **Pending** (native OS file dialog modal unverified) |
| **Archive: Explorer UI** | Virtual tree, search, multi-selection, contextual download | **Implemented** | **Passed** | **Passed** | **Pending** |
| **Image Tools** | Converter, Compressor, Resizer, GIF Frame Extractor | **Implemented** | **Passed** | **Passed** (basic flows) | **Completed** (prior milestone) |
| **Audio Tools** | Converter, Compressor, Trimmer, Waveform visualizer | **Implemented** | **Passed** | **Passed** | **Pending** |
| **Video Tools** | Video compressor, converter, trimmer | **Implemented** | **Passed** | **Passed** | **Pending** |

---

## 3. Archive Folder Compression & Verification Metrics

During Phase 5.x and the folder-compression enhancement:

- **Automated Test Suite (Vitest):**
  - **251 passing tests** across 15 test files covering archive tree construction, safe duplicate path disambiguation, file/directory ancestor conflict resolution, folder traversal, path normalization, tar incremental streaming, worker message serialization, quarantine warning visibility, and adapter integrations (includes 37 tests in `archiveTree.test.ts`, 27 in `archiveDownload.test.ts`, 39 in `ArchiveExplorer.test.tsx`, 7 in `archiveCreationPathPreservation.test.ts`, and 12 in `folderTraversal.test.ts`).
- **Static Type Check:**
  - `tsc -b` completed with **0 errors**.
- **Linter (`oxlint`):**
  - Completed with **0 errors** (34 warnings reported in legacy files).
- **Production Build:**
  - `vite build` completed successfully without bundle or asset errors.
- **Headless Browser Automation (Playwright / Chromium):**
  - Single file upload name/size verification: **Passed**.
  - Folder input selection (`webkitdirectory`): **Passed** (5 files loaded).
  - Drag-and-drop directory tree simulation: **Passed** (5 files traversed).
  - ZIP creation and byte audit (`fflate.unzipSync`): **Passed** (100% path and content preservation).
  - TAR creation, download, and re-extraction roundtrip: **Passed** (5 files unpacked).
  - TAR.GZ creation, download, and re-extraction roundtrip: **Passed** (5 files unpacked).
  - Arabic RTL layout (`html[dir="rtl"]`, translations, warning notes): **Passed**.
- **Manual Operating-System Interaction:**
  - **Limitation:** The native Windows File Explorer folder-picker dialog triggered by a human user's mouse click remains unverified through an actual physical manual OS interaction.

---

## 4. Known Issues & Unresolved Technical Risks

### Incident RSK-02: Duplicate Archive Path Collision & UI Rejection
- **Location:** `folderTraversal.ts` (`sanitizeArchiveFiles`), `workerManager.ts`, `archiveWorker.ts`, `tarUtils.ts`, `archiveTree.ts`, `archiveDownload.ts`, `ArchiveExplorer.tsx`.
- **Status:** **Fully Resolved (Phase 1 Creation Collision, Phase 2 Worker Serialization, Phase 3 Extraction & Download All Tree Disambiguation, and Final Hardening Verified).**
- **Symptoms & Resolution State:**
  1. *Creation Collision:* **Resolved (Phase 1).** `sanitizeArchiveFiles` uses a reservation set (`usedPaths`) and collision counter search, allocating the next free path (`Screenshot (2).png`) when input sets contain pre-existing numbered filenames (`Screenshot (1).png`). Verified by automated regression tests.
  2. *Worker Property Loss:* **Resolved (Phase 2).** Added explicit serializable `filePaths?: string[]` to `ArchiveTask` in `types.ts`, emitted by `CompressZip.tsx`, `CreateTar.tsx`, and `CreateTarGz.tsx`, defensively populated in `ArchiveWorkerManager.run()`, and consumed in `archiveWorker.ts` and `tarUtils.ts`. Fully survives browser Structured Clone Algorithm. Verified by automated integration tests.
  3. *Extraction Virtual Tree & Download All Disambiguation:* **Resolved (Phase 3).**
     - Virtual Tree Disambiguation: `buildArchiveVirtualTree` in `archiveTree.ts` deterministically resolves duplicate full paths using sequential counter suffixes `base (1).ext` without overwriting or merging.
     - File/Directory Conflict Resolution: Intermediate folder traversal safely disambiguates colliding file nodes to free directory slots, preserving both the directory hierarchy and the file content.
     - Download All as ZIP: `buildZipFromArchiveFiles` in `archiveDownload.ts` applies identical deterministic disambiguation, streaming each file into the ZIP archive with 100% byte fidelity.
     - Explorer UI Display: `ArchiveExplorer.tsx` renders disambiguated files with clear secondary text and tooltips displaying their link to the original archive path.
  4. *Final Hardening:* **Resolved.**
     - ZIP Ancestor-Path Conflicts: `buildZipFromArchiveFiles` delegates path planning directly to `buildArchiveVirtualTree`, detecting and resolving file/directory ancestor conflicts (e.g. `docs` vs `docs/report.txt`, `a` vs `a/b` vs `a/b/c`) in forward, reversed, and repeated nested input orders without silent overwrites or discarded entries.
     - Quarantine Warning Visibility: `ArchiveExplorer.tsx` renders an accessible, localized security warning banner (`role="status"`, `aria-live="polite"`) whenever quarantined entries exist (`tree.warnings`), with count badge and quarantined file tags without exposing unsafe raw paths in markup. Bilingual English and Arabic RTL layout fully supported.
- **Impact:** Complete end-to-end resilience against duplicate paths and ancestor conflicts across archive creation, Web Worker IPC, virtual tree extraction, UI exploration, and Download All ZIP packaging.
- **Reference:** See `SECURITY_REGISTER.md` (SEC-02) and `DECISION_LOG.md` (ADR-05).

---

## 5. Next Immediate Engineering Steps

1. Conduct physical manual verification of the native Windows folder-picker dialog in Chrome and Edge.
2. Advance to the next scheduled module audit once archive folder compression closure is confirmed.
