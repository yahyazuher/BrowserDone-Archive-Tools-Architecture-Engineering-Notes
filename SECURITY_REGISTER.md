# Security Register — BrowserDone

**Scope:** BrowserDone Application Security, Archive Engine & Client-Side File Processing  
**Last Updated:** 2026-10-15  
**Security Model:** 100% Client-Side Processing (Zero Server-Side Uploads)

---

## 1. Security Architecture & Threat Model

BrowserDone processes user files entirely within the browser sandbox using JavaScript and WebAssembly. No files, metadata, or uncompressed buffers are ever transmitted to any remote backend server.

### Core Security Defenses:
1. **Archive Bomb & Resource Exhaustion Defense (`ArchiveBombValidator`):**
   - Limits max file count (`maxFiles`) per archive according to plan constraints.
   - Enforces max total uncompressed payload size (`maxTotalSize`).
   - Calculates compression ratio threshold (`maxRatio`) to abort recursive expansion attacks.
2. **Path Traversal & Directory Escape Prevention (`getSafePathAndDepth`):**
   - Strips leading forward and backward slashes.
   - Forbids and rejects parent directory navigation (`../`, `..\`) that could escape logical archive roots.
   - Detects and rejects null-byte injections (`\0`).
3. **Operating System Filename Sanitization (`sanitizeDiskSegment`):**
   - Strips Windows-forbidden characters (`< > : " / \ | ? *`) and control characters (`0x00-0x1F`).
   - Neutralizes Windows reserved device names (`CON`, `PRN`, `AUX`, `NUL`, `COM1-9`, `LPT1-9`) by prefixing an underscore (e.g. `_CON.txt`).
4. **Memory-Adaptive Streaming:**
   - Incremental streaming using `Blob.slice()` and Web Worker streams to avoid loading multi-gigabyte uncompressed archives into main-thread JavaScript heap.

---

## 2. Security Risk & Incident Register

| Risk ID | Title | Component | Severity | Status | Next Action |
| :--- | :--- | :--- | :---: | :---: | :--- |
| **SEC-01** | Empty Basename Normalization (RSK-01) | `ExtractTar`, `ExtractTarGz` | **Low** | **Resolved** | Regression test maintained in suite |
| **SEC-02** | Duplicate Path Incident / UI Denial-of-Service (RSK-02) | `folderTraversal.ts`, `workerManager.ts`, `archiveWorker.ts`, `archiveTree.ts`, `archiveDownload.ts`, `ArchiveExplorer.tsx` | **Medium** | **Resolved** | Regression suites maintained across 15 test files (251 tests) |

---

## 3. Detailed Security Incident Entries

### SEC-01 (RSK-01): Archive Basename Fallback Sanitization
- **Component:** `src/tools/archive/extract-tar/ExtractTar.tsx` and `ExtractTarGz.tsx`
- **Evidence:** Files named `.tar.gz` or `.tgz` produced an empty base name string when extensions were stripped, resulting in default download names like `-BD-Extracted.zip`.
- **Impact:** Malformed download filename; potential confusion in local file system.
- **Severity Rationale:** Low. No data leakage or code execution risk.
- **Status:** **Resolved.**
- **Implemented Fix:** Safe fallback function `getTarBaseName` ensuring normalization and defaulting to `Archive` if the resulting string is empty or whitespace.

---

### SEC-02 (RSK-02): Duplicate Path Collision & Extraction UI Denial-of-Service
- **Component:**
  - `src/core/archive-engine/folderTraversal.ts` (`sanitizeArchiveFiles`)
  - `src/core/archive-engine/workerManager.ts` & `archiveWorker.ts`
  - `src/core/archive-engine/archiveTree.ts` (`buildArchiveVirtualTree`)
  - `src/core/archive-engine/archiveDownload.ts` (`buildZipFromArchiveFiles`)
  - `src/tools/archive/shared/ArchiveExplorer.tsx`
- **Evidence:**
  1. A user created a TAR archive from 3 files and upon extracting it in `ExtractTar.tsx`, the UI displayed:
     `خطأ في بنية الأرشيف — Duplicate file path detected: "Screenshot (1).png"`.
  2. Investigation confirmed that `sanitizeArchiveFiles` generated colliding paths when given `['Screenshot.png', 'Screenshot (1).png', 'Screenshot.png']`, outputting `['Screenshot.png', 'Screenshot (1).png', 'Screenshot (1).png']`.
  3. `archiveWorker.ts` dropped custom expando properties (`file.relativePath`) during Web Worker `postMessage` transfer due to Structured Clone property omission, causing files from different folders to fall back to identical basenames.
  4. `buildArchiveVirtualTree` and `buildZipFromArchiveFiles` threw fatal unhandled errors on duplicate complete paths, causing `ArchiveExplorer` to crash into an error card and hiding all archive contents.
- **Impact:**
  - **Availability:** Complete Denial of Service for viewing and extracting archives with duplicate complete paths.
  - **Data Integrity:** "Download All as ZIP" threw `ArchiveDownloadError`, blocking batch recovery of extracted files.
  - **Confidentiality:** No data leakage.
- **Severity Rationale:** Medium. High UX and availability failure on legitimate archives containing duplicate paths (common in append-mode Unix TAR archives or Windows screenshot folders); zero remote code execution risk.
- **Status:** **Resolved (Phase 1, Phase 2, Phase 3, and Final Hardening Verified).**
- **Implemented Fixes:**
  1. *Phase 1 (Creation Disambiguation):* `sanitizeArchiveFiles` uses a reservation set (`usedPaths`) and collision counter search, allocating the next free path (`Screenshot (2).png`) when input sets contain pre-existing numbered filenames (`Screenshot (1).png`).
  2. *Phase 2 (Worker Message IPC):* Added explicit, serializable `filePaths?: string[]` to `ArchiveTask` in `types.ts`, populated by creation tools and `ArchiveWorkerManager.run()`, and consumed in `archiveWorker.ts` and `tarUtils.ts`.
  3. *Phase 3 (Virtual Tree & Download All Disambiguation):*
     - `buildArchiveVirtualTree` in `archiveTree.ts` deterministically resolves duplicate paths using sequential counter suffixes `base (1).ext` without overwriting or merging.
     - Resolves file/directory conflicts so directories and files at identical path keys can coexist without data loss or unhandled exceptions.
     - `buildZipFromArchiveFiles` in `archiveDownload.ts` applies identical deterministic disambiguation, preserving all files and their contents in the generated ZIP.
     - `ArchiveExplorer.tsx` displays disambiguated filenames with clear secondary text and tooltips showing their original archive path.
     - `allowRecovery` option safely sanitizes and isolates malformed or traversal paths under `_quarantine/` with warnings, preventing one invalid entry from hiding unrelated valid files.
  4. *Final Hardening (Ancestor Conflicts & Quarantine Visibility):*
     - Unified `buildZipFromArchiveFiles` with `buildArchiveVirtualTree`, ensuring ZIP ancestor-path conflicts (e.g. `docs` vs `docs/report.txt`, `a` vs `a/b` vs `a/b/c`) are deterministically resolved in both directions without silent data loss.
     - Added accessible security warning banner (`role="status"`, `aria-live="polite"`) and file badges to `ArchiveExplorer.tsx`, rendering localized warnings with count indicators in English and Arabic RTL without exposing unescaped raw paths. Verified by 251 passing tests.
