# Changelog — BrowserDone

All notable changes to the BrowserDone engineering codebase are documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [Unreleased] — Phase 5: Archive Engine & Folder Compression

### Added
- **Directory Upload & Traversal in Shared Dropzone (`src/core/ui/Dropzone.tsx`):**
  - Added `directoryUpload` prop to enable native folder selection using `<input type="file" webkitdirectory directory multiple>`.
  - Added a dedicated "Select Folder" button (`styles.folderSelectBtn`) with localized button labels (`اختر مجلداً` in Arabic).
  - Integrated `traverseDroppedItems` from `src/core/archive-engine/folderTraversal.ts` to recursively expand dropped directory hierarchies using `webkitGetAsEntry` and paginated `readEntries()`.
- **Relative Path Preservation in Archive Creation:**
  - Integrated `getFileRelativePath` and `sanitizeArchiveFiles` into `CompressZip.tsx`, `CreateTar.tsx`, and `CreateTarGz.tsx`.
  - Preserved nested directory structures (e.g. `folder/sub/file.txt`) instead of flattening entries to simple basenames (`file.name`).
  - Added support for standard POSIX UStar 155-byte directory prefix writing in `tarUtils.ts`.
- **Legitimate Zero-Byte File Support:**
  - Allowed genuine 0-byte files (such as `.gitkeep` or empty text files) to be preserved as 0-byte entries in created archives.
  - Added informative warning notices (`⚠️ 1 empty file(s) detected (0 Bytes)`) informing the user that 0-byte entries will be archived rather than failing or discarding them.
- **Archive Explorer Multi-Selection & Contextual Actions (`ArchiveExplorer.tsx`):**
  - Added folder-level "Select All" and individual file checkbox multi-selection.
  - Added contextual "Download Selected" action alongside existing "Download All as ZIP".
- **Verification Suites:**
  - Added 15 Vitest test suites (227 total tests) verifying directory tree building, path sanitization, folder traversal, and archive adapters.

### Changed
- Replaced basename-only file handling in `CompressZip.tsx`, `CreateTar.tsx`, and `CreateTarGz.tsx` with relative path resolution.
- Updated TAR creation to encode ustar directory prefixes for files with path depth.

### Fixed
- **Maintenance Fix RSK-01:** Normalized archive basename fallbacks in `ExtractTar.tsx` and `ExtractTarGz.tsx` when stripping `.tar.gz` and `.tgz` extensions, preventing empty base strings or default naming failures like `-BD-Extracted.zip`.
- **Archive Creation Collision Fix (RSK-02 Phase 1):** Resolved collision path generation in `sanitizeArchiveFiles` (`src/core/archive-engine/folderTraversal.ts`). When an archive input set contains existing numbered basenames (e.g., `Screenshot.png`, `Screenshot (1).png`, `Screenshot.png`), an unused numbered path is dynamically assigned (allocating `Screenshot (2).png`). Preserves relative directory structure, identical basenames across distinct folders, and file contents.
- **Worker Message Path Preservation (RSK-02 Phase 2):** Resolved loss of relative directory paths across Web Worker boundaries caused by Structured Clone Algorithm property stripping. Added an explicit, serializable `filePaths?: string[]` array to `ArchiveTask` in `src/core/archive-engine/types.ts`. Emitted `filePaths` deterministically from `CompressZip.tsx`, `CreateTar.tsx`, and `CreateTarGz.tsx`, with defensive auto-serialization in `ArchiveWorkerManager.run()`. Consumed `task.filePaths` across ZIP, TAR, and TAR.GZ creation pipelines (`archiveWorker.ts`, `tarUtils.ts`). Preserves path security validation, zero-byte file support, cancellation, and resource cleanup.
- **Extraction Virtual Tree & Download All Duplicate Disambiguation (RSK-02 Phase 3):** Resolved duplicate archive path handling in the extraction virtual tree (`buildArchiveVirtualTree` in `src/core/archive-engine/archiveTree.ts`) and Download All ZIP generation (`buildZipFromArchiveFiles` in `src/core/archive-engine/archiveDownload.ts`). Rather than aborting extraction with a fatal `ArchiveTreeError` or `ArchiveDownloadError` when processing archives with duplicate entries (e.g., `Screenshot (1).png`), paths are deterministically disambiguated using sequential numbering (`base (1).ext`) without silent merging or content loss. Resolves file/directory conflicts so directories and files at identical path keys can coexist without corruption. Added `allowRecovery` mode for safe fault isolation of malformed paths under `_quarantine/` so valid entries are not hidden. Updated `ArchiveExplorer.tsx` to render disambiguated filenames with clear secondary labels showing their original archive path.
- **RSK-02 Final Hardening (Ancestor Conflicts & Quarantine Visibility):**
  - **ZIP Ancestor-Path Conflicts:** Refactored `buildZipFromArchiveFiles` (`archiveDownload.ts`) to delegate path planning directly to `buildArchiveVirtualTree`, unifying ZIP entry paths with the virtual tree and resolving ancestor conflicts (`docs` vs `docs/report.txt`, `a` vs `a/b` vs `a/b/c`) in forward, reversed, and repeated nested input orders without silent overwrites, missing files, or packaging errors.
  - **Quarantine Warning Visibility:** Added an accessible security warning banner (`role="status"`, `aria-live="polite"`) and file badges to `ArchiveExplorer.tsx` when `tree.warnings` has quarantined entries, localized in English and Arabic RTL with count indicators, while avoiding exposing unescaped raw paths in markup.

### Quality & Verification Results (Phase 5)
- **Automated Tests:** 251 tests passing across 15 test files via Vitest (+9 focused tests across `archiveDownload.test.ts` and `ArchiveExplorer.test.tsx` verifying ancestor conflict resolution, unpack roundtrips, quarantine warning accessibility, and XSS path safety).
- **TypeScript:** Type check passed (`tsc -b`) with 0 errors.
- **Linter:** Code style check passed (`oxlint`) with 0 errors (34 warnings in legacy files).
- **Production Build:** Bundle build passed (`vite build`) successfully.
- **Browser Automation:** Headless Chromium automated verification completed:
  - Single file upload name and size verified.
  - Directory upload with 5 files (including nested paths, duplicate basenames in different folders, and 0-byte file) verified.
  - Downloaded ZIP inspected via `fflate.unzipSync`: 100% path and content fidelity.
  - TAR and TAR.GZ creation and extraction roundtrip verified.
  - Arabic RTL layout and translations verified.
- **Verification Limitation:** The native Windows folder-picker dialog modal spawned by the OS file manager remains unverified through actual manual physical interaction.

### Known Issues & Unresolved Technical Risks
- **Incident RSK-02:** **Fully Resolved across Phase 1, Phase 2, Phase 3, and Final Hardening.**
  - Creation-time path collisions in `sanitizeArchiveFiles` are resolved (Phase 1).
  - Web Worker message path preservation via explicit `filePaths` is resolved and verified across all creation tools (Phase 2).
  - Extraction-time duplicate path handling in `ArchiveExplorer` (`buildArchiveVirtualTree`) and `buildZipFromArchiveFiles` is resolved and verified (Phase 3).
  - ZIP ancestor-path conflicts and quarantine warning visibility are hardened and verified with 251 passing tests.

---

## [Phase 4] — Audio & Video Enhancements
- Added WebCodecs / Web Audio processing pipelines and waveform rendering using `wavesurfer.js`.
- Implemented client-side audio compressor, trimmer, and format converters.
- Refactored video router and integration wrappers.

---

## [Phase 3] — Feedback & Support System
- Enhanced feedback modal UX with Telegram bot integration, golden status animations, and language-aware topic routing.
- Fixed character sanitization issues in feedback payload submission.

---

## [Phase 2] — Image Editor Architecture Refactoring
- Decoupled view and edit state in Image Editor.
- Enhanced Image Adjust tool with CSS logical properties for RTL layout and improved mobile touch ergonomics.
