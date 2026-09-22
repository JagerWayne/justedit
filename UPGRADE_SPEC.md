# JustEdit Pro — Upgrade Specification

**Status:** Draft  
**Target artifact:** `justedit_edgeone_dev.html` (single-file build)  
**Hard constraint:** The application **must remain a single HTML file**. No module split, no
bundler/dev server, and no service-worker/PWA. All upgrades are delivered in place. Items that
would violate this are explicitly marked out of scope.  
**Purpose:** Define a prioritized, phased roadmap to raise JustEdit Pro from a feature-rich
single-file prototype to a professional-grade, secure, maintainable editor.

---

## 1. Executive Summary

JustEdit Pro is a browser-based code editor delivered as **one self-contained HTML file**
(~1,930 lines) built on **CodeMirror 5**. It already supports:

- Multi-tab editing with dirty-state tracking and session restore
- 45+ language modes and syntax highlighting
- A virtualized hex editor (mobile-optimized)
- An image viewer with non-destructive adjustments (brightness/contrast/rotate/flip/format export)
- Markdown split preview and sandboxed HTML live preview
- Base64 encode/decode, Data-URL import/export
- Four themes (Dracula, Monokai, GitHub Light, Eclipse)

The strengths are portability and breadth of features. The weaknesses are architectural
(single monolithic file, inline handlers, EOL CodeMirror 5), security (no SRI/CSP, unsafe
sandbox and Markdown fallback), and operational (no build, tests, docs, or offline support).

This document defines **8 workstreams** and **5 delivery phases**. Each item includes a
priority, effort estimate, and acceptance criteria.

### Priority legend

| Tag | Meaning |
|-----|---------|
| **P0** | Security / data-loss / correctness — do first |
| **P1** | High-value foundation or user-visible quality |
| **P2** | Professional polish / developer experience |
| **P3** | Nice-to-have / future |

### Effort legend

`S` = hours, `M` = 1–2 days, `L` = 3–5 days, `XL` = 1+ week.

---

## 2. Current-State Assessment

### 2.1 Code map (by line ranges)

| Area | Lines | Notes |
|------|-------|-------|
| Document head / CDN deps | 1–80 | 30+ external CSS/JS tags, no SRI |
| CSS (theming, layout) | 81–390 | ~310 lines inline `<style>` |
| Markup (menu, modals, panels) | 392–748 | Inline `onclick` handlers throughout |
| App core / state | 750–960 | IIFE exposing global `App` |
| Editor actions (find/replace, base64) | 973–1136 | CodeMirror 5 APIs |
| File open/save/data-url | 1138–1299 | File System Access API partially used |
| Tabs & views | 1301–1420 | `innerHTML` template rendering |
| Hex editor | 1497–1599 | Full-viewport `innerHTML` re-render |
| Image editor/encoder | 1601–1779 | BMP encoder, canvas filters |
| UI (tabs, status, toast, theme) | 1781–1861 | |
| Storage (localStorage) | 1867–1920 | Byte arrays serialized as JSON number lists |

### 2.2 Key risk findings

| ID | Finding | Severity | Location |
|----|---------|----------|----------|
| R1 | No Subresource Integrity on any CDN resource — a compromised CDN yields arbitrary code execution | Critical | 15–79 |
| R2 | HTML preview iframe uses `sandbox="allow-scripts allow-same-origin"` — combination defeats the sandbox | Critical | 639 |
| R3 | Markdown preview falls back to `innerHTML` with unsanitized HTML if DOMPurify is unavailable | Critical | 1427–1430 |
| R4 | `localStorage` stores byte arrays as JSON number lists — huge strings, slow, quota-prone | High | 1884–1887 |
| R5 | No `beforeunload` guard for unsaved tabs (autosave only) | High | 896–901 |
| R6 | Hex editor rebuilds entire viewport `innerHTML` on every keystroke/scroll | Medium | 1507–1559 |
| R7 | No CSP; inline handlers block a strict policy | Medium | throughout |
| R8 | No focus trap / `role="dialog"` / Escape handling on modals | Medium | 1834–1839 |
| R9 | `shareDataUrl` builds a `data:text/html` payload containing executable JS | Medium | 1467–1492 |
| R10 | CodeMirror 5 is end-of-life (last release 2023) | Medium | 14–79 |
| R11 | Duplicated mode/extension definitions between JS and markup | Low | 765–800, 454–502, 678–731 |
| R12 | Static status ("Ready", "UTF-8") that does not reflect real state | Low | 580–589 |

---

## 3. Workstreams & Upgrade Items

### WS-1 — Security & Privacy (P0)

| ID | Item | Priority | Effort |
|----|------|----------|--------|
| 1.1 | Add SRI (`integrity` + `crossorigin`) to all CDN `<script>`/`<link>` tags | P0 | S |
| 1.2 | Fix HTML preview iframe sandbox: drop `allow-same-origin`, or isolate via blob/origin proxy | P0 | S |
| 1.3 | Remove unsanitized Markdown fallback; fail closed and warn | P0 | S |
| 1.4 | Add a Content-Security-Policy (requires removing inline handlers — see WS-2) | P0 | M |
| 1.5 | Document and harden `shareDataUrl` (executable data-URL); add explicit user consent/caveat | P1 | S |
| 1.6 | Move binary/large content to IndexedDB; keep secrets out of localStorage | P1 | M |
| 1.7 | ~~Vendor/self-host dependencies~~ — optional single-file offline pass (inline-vendor libs), see Phase 4 | P3 | L |

**Acceptance criteria**
- Every external resource carries a valid `integrity` attribute and `crossorigin="anonymous"`.
- Opening arbitrary HTML in preview cannot access parent `window`/cookies.
- Markdown preview never injects unsanitized HTML under any condition.
- A meaningful CSP is in effect (blocks `object-src`, `base-uri`, `form-action`, and `eval`).

### WS-2 — Architecture & Maintainability (P1) — **single-file, no build**

**Constraint:** JustEdit Pro must ship as **one HTML file** and must not require a build step
or a modular split. Architecture work therefore happens *in place*.

| ID | Item | Priority | Effort |
|----|------|----------|--------|
| 2.1 | ~~Split into `index.html` + ES modules~~ — **out of scope (single-file requirement)** | — | — |
| 2.2 | Remove all inline `onclick`/`oninput`/`onchange` handlers; use delegated listeners (`data-action`) | P1 | M |
| 2.3 | ~~Build tooling (Vite/esbuild)~~ — **out of scope (no build step)** | — | — |
| 2.4 | ~~`package.json` / npm scripts~~ — replaced by an optional, build-free self-check script | P3 | S |
| 2.5 | Single source of truth for language modes / extensions; generate the two `<select>` lists | P2 | S |
| 2.6 | Introduce a small state store (e.g. observable pattern) instead of ad-hoc global mutation | P2 | M |
| 2.7 | Add JSDoc annotations and run `node --check` on the extracted script as a build-free syntax gate | P2 | S |

**Acceptance criteria**
- No inline event attributes remain; the file stays a single deployable HTML document.
- App functions under a meaningful CSP without a bundler or dev server.
- Language/mode lists can only be edited in one place.

### WS-3 — Performance & Storage (P1)

| ID | Item | Priority | Effort |
|----|------|----------|--------|
| 3.1 | IndexedDB persistence for tabs/bytes (Blob storage), localStorage only for prefs | P1 | M |
| 3.2 | Incremental hex rendering — update only changed cells; recycle DOM nodes | P1 | M |
| 3.3 | Debounce/serialize large content operations; avoid full `getValue()`/`setValue()` churn | P2 | S |
| 3.4 | Lazy-load language modes on demand instead of 30+ eager script tags | P2 | M |
| 3.5 | Virtualize or cap tab preview rendering for very large files | P3 | M |

**Acceptance criteria**
- Editing/persisting a 5 MB tab does not freeze the UI or exceed storage quotas silently.
- Hex scrolling/editing at 60 fps on a mid-range mobile device.
- Initial load requests only the modes actually needed.

### WS-4 — Editor Engine Modernization (P1)

| ID | Item | Priority | Effort |
|----|------|----------|--------|
| 4.1 | Migrate CodeMirror 5 → CodeMirror 6 (or Monaco) | P1 | XL |
| 4.2 | Persist editor preferences: font size, word wrap, tab size, line numbers, theme | P1 | S |
| 4.3 | Find/Replace: highlight all matches, show count/index, search history | P1 | M |
| 4.4 | Add command palette (Ctrl/Cmd+Shift+P) + discoverable shortcut help | P2 | M |
| 4.5 | Autocomplete, auto-indent-on-enter, multi-cursor, bracket-colorization | P2 | L |
| 4.6 | Undo/redo toolbar state + "reopen closed tab" | P2 | S |
| 4.7 | Go-to-line / go-to-symbol | P3 | S |

**Acceptance criteria**
- Editor rendering is the CM6/current-generation engine.
- Editor preferences survive reloads.
- All matches are visually highlighted with a match counter.

### WS-5 — Feature Expansion (P2)

| ID | Item | Priority | Effort |
|----|------|----------|--------|
| 5.1 | Folder open + file-tree sidebar (File System Access API) | P1 | L |
| 5.2 | Save back to original file handle (not just download) | P1 | M |
| 5.3 | Hex editor: search, insert/delete bytes, range selection, copy/paste, undo | P2 | L |
| 5.4 | Image editor: crop, resize, richer filters, metadata/EXIF panel | P2 | L |
| 5.5 | Real encoding + EOL controls (UTF-8/UTF-16, LF/CRLF) with status-bar reflection | P2 | M |
| 5.6 | Recent files list + stronger session restore | P2 | S |
| 5.7 | Custom theme editor / CSS-variable theming | P3 | M |
| 5.8 | Export/print (PDF, formatted HTML) | P3 | M |

### WS-6 — UX, Accessibility & i18n (P2)

| ID | Item | Priority | Effort |
|----|------|----------|--------|
| 6.1 | Modal focus trap, `role="dialog"`, `aria-modal`, Escape-to-close | P1 | S |
| 6.2 | `aria-label` on all icon-only buttons; keyboard-operable menu | P1 | S |
| 6.3 | Dynamic status bar (real encoding/EOL/language; live "saved" state) | P2 | S |
| 6.4 | Reflected dirty state in `document.title`; full path in breadcrumbs | P2 | S |
| 6.5 | WCAG AA contrast audit across all four themes | P2 | M |
| 6.6 | Loading/skeleton and disabled states for async actions | P3 | S |
| 6.7 | Internationalization (i18n) framework + initial locales | P3 | L |

### WS-7 — Reliability & Offline (P1) — **single-file adaptations**

| ID | Item | Priority | Effort |
|----|------|----------|--------|
| 7.1 | ~~PWA service worker~~ — **not possible in a pure single file**; replace with inline-vendored assets for offline (Phase 4) | — | — |
| 7.2 | `beforeunload` guard when tabs are dirty | P1 | S |
| 7.3 | Global error handler + user-facing error reporting (optional telemetry, opt-in) | P2 | S |
| 7.4 | File Handling API so the OS can open files into the app | P2 | M |
| 7.5 | Crash/restore recovery snapshot independent of debounce window | P2 | M |

### WS-8 — Tooling, Docs & DevOps (P2) — **build-free where possible**

| ID | Item | Priority | Effort |
|----|------|----------|--------|
| 8.1 | Prettier for formatting + a `node --check` syntax gate (ESLint optional, run ad hoc) | P2 | S |
| 8.2 | Test suite: unit (Vitest) + e2e smoke (Playwright) — runs against the single file | P2 | L |
| 8.3 | `README.md`, user guide, `CONTRIBUTING.md`, `CHANGELOG.md`, `LICENSE` | P2 | S |
| 8.4 | CI that runs the syntax gate + tests and publishes the single HTML file | P2 | M |
| 8.5 | Automated dependency/security scanning (Dependabot, `npm audit`) | P3 | S |

---

## 4. Delivery Phases

### Phase 0 — Quick Wins (done)
Low-risk, high-value fixes applied directly to the single-file build.
- 1.1 SRI on all CDN resources
- 1.2 Iframe sandbox fix
- 1.3 Remove unsafe Markdown fallback
- 6.1 Escape-to-close modals/menus
- 7.2 `beforeunload` guard for dirty tabs
- 2.5 Single source of truth for mode/extension lists
- 6.2 `aria-label` additions on icon-only controls

**Exit criteria:** all hashes validate, no XSS via Markdown/preview, no data-loss on accidental
navigation, lists generated from one source.

### Phase 1 — Foundation (single-file) — **done**
- 2.2 Remove all inline handlers → delegated `data-action` listeners
- 1.4 Add a meaningful CSP (meta)
- WS-3 storage/performance (3.1–3.2): IndexedDB + incremental hex rendering
- 1.4 storage schema versioning + migration
- 7.3 global error handler

**Exit criteria:** no inline handlers, meaningful CSP active, IndexedDB-backed with v1→v2
migration, smooth hex rendering on large files — all within one HTML file.

### Phase 2 — Editor Modernization — **done (CM5 retained)**
- 4.2 Persisted editor preferences (font size, tab size, word wrap, line numbers, active line,
  auto-close brackets, code folding, highlight matches) with an **Editor** section in the menu.
- 4.3 Find/Replace: highlight all matches, live match counter ("n of total"), current-match
  emphasis, silent invalid-regex handling, highlights cleared on close/tab-switch.
- 6.3/6.4 Dynamic status bar: language, clickable EOL (LF/CRLF) toggle with byte-accurate
  conversion, encoding, and Saved/Unsaved state.
- Decision: **CodeMirror 5 retained** (single-file constraint); no engine migration.

**Exit criteria:** modern engine, persisted preferences, improved search UX.

### Phase 3 — Power Features — **done**
- 5.1 Folder open + read-only file-tree sidebar (File System Access API), lazy expansion.
- 5.2 Save-back-to-original-handle (Ctrl/Cmd+S) with image-edit baking.
- 5.6 Persisted folder/file handles in IndexedDB + Recent Files menu + reconnect.
- 5.3 Hex search (hex/text), match highlighting, next/prev, insert/delete bytes.
- 5.5 Encoding detection + conversion: UTF-8, UTF-16 LE/BE (with/without BOM).
- 5.4 Image crop (drag handles), resize (aspect lock), and metadata readout.
- 5.7 Theme editor: edit all theme variables with live preview, save/load/delete custom themes.
- 4.4 Command palette (Ctrl/Cmd+Shift+P) with all commands + recent files.

### Phase 4 — Single-file Offline & Ecosystem
- Inline-vendor CodeMirror + modes + libs + fonts for a zero-network single file (~2–3 MB)
- WS-8 tests/CI/docs
- 5.4 image editor expansion; 5.7 theme editor; 6.7 i18n

---

## 5. Testing Strategy

| Layer | Tool | Coverage target |
|-------|------|-----------------|
| Unit | Vitest | Storage codecs, mode mapping, base64/data-url, image filter builder, hex math |
| Integration | Vitest + jsdom | Tab lifecycle, find/replace, theme switching |
| E2E | Playwright | Open/edit/save, drag-drop, hex editing, preview sandbox isolation, mobile viewport |
| Security | Manual + automated scan | SRI validation, CSP violations, sandbox escape attempts |

Regression gates for every phase:
1. `npm run lint` passes.
2. `npm test` (unit + integration) passes.
3. E2E smoke passes for core flows.
4. Lighthouse PWA + accessibility budgets met (target ≥ 90).

---

## 6. Compatibility & Migration Notes

- **Persistence migration:** moving from localStorage (v1 schema `je_pro_*`) to IndexedDB must
  include a one-time migration and a versioned schema; retain read compatibility with v1 data.
- **CodeMirror migration:** map the current `mode` strings to CM6 language extensions; keep a
  fallback to plain text for unmapped modes. Preserve the mode dropdown values where possible.
- **SRI maintenance:** any dependency version bump requires recomputing hashes; automate this in
  the build pipeline rather than hand-editing.

---

## 7. Success Metrics

| Metric | Baseline | Target |
|--------|----------|--------|
| External resources with SRI | 0% | 100% |
| Unsafe HTML injection paths | 2 | 0 |
| Time-to-interactive (cold) | measure | < 1.5 s on mid-tier mobile |
| Offline availability | none | full editor + assets |
| Automated test coverage | 0 | ≥ 60% on core modules |
| Accessibility (Lighthouse) | measure | ≥ 90 |

---

## 8. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| CM5→CM6 migration breaks features | High | Feature-flag/parallel integration; keep CM5 fallback until parity |
| Storage migration corrupts sessions | High | Versioned schema + backup before migrate + rollback path |
| CSP blocks libraries that use `eval`/inline | Medium | Audit deps; prefer CSP-safe builds; allowlist only what is required |
| Scope creep in single-file quick-wins pass | Low | Keep quick wins strictly additive/low-risk; defer structural work to Phase 1 |

---

*Change log:*
- *initial specification.*
- *Single-file constraint locked in; module split, bundler, and service-worker PWA marked out of scope.*
- *Phase 0 (quick wins) delivered: SRI on all CDN assets, iframe sandbox hardening, safe Markdown
  fallback, Escape-to-close, dirty-tab guard, generated mode/extension lists, aria-labels.*
- *Phase 1 delivered: 61 inline handlers → delegated `data-action` listeners; meaningful CSP;
  IndexedDB binary storage with v1→v2 migration and corrupt-recovery; incremental hex rendering;
  global error handler; share-link consent.*
- *Bug fixes: missing `defineSimpleMode` addon (rust/dockerfile modes), stale `tab.bytes` after
  programmatic `setValue` (Encode/Decode Base64 and Toggle-Hex), duplicate `data-idx` attributes.*
- *Phase 2 delivered (CodeMirror 5 retained): persisted editor preferences with an Editor menu
  section; find/replace match highlighting and live counter; dynamic status bar with language,
  clickable LF/CRLF EOL conversion, encoding, and Saved/Unsaved state.*
- *Filenames: breadcrumb + `document.title` (with dirty marker) update from the live tab name;
  untitled tabs named `new-N` reusing the lowest free number; added/updated editor prefs to
  `Storage`.*
- *Phase 3 (part 1) delivered: read-only folder/file-tree sidebar with lazy expansion; open files
  from the tree; save back to the original file handle (Ctrl/Cmd+S); Image-edits baked on save;
  IndexedDB `handles` store (DB v2) persisting the folder and recent files; Recent Files menu and
  folder reconnect after reload. Prevented stale-buffer writes by refreshing text tabs from the
  live editor before saving.*
- *Phase 3 (part 2) delivered: hex toolbar with byte/text search and match highlighting,
  next/previous navigation, and insert/delete byte operations with selection ranges. Hex view
  re-laid-out as a flex column (toolbar + horizontally scrolling body) and verified on a 390×844
  mobile viewport. Fixed a mobile stacking bug where the dropdown menu was trapped under the
  sidebar by removing the breadcrumb stacking context.*
- *Phase 3 (part 3) delivered: command palette (Ctrl/Cmd+Shift+P) covering all commands + recent
  files; theme editor with live preview and save/load/delete of custom themes (persisted); text
  encoding detection and conversion for UTF-8 / UTF-16 LE / UTF-16 BE (BOM aware) wired into the
  status bar, save, and hex round-trip; image crop with pointer-drag handles, aspect-locked
  resize, and a dimensions/format/size metadata readout. Verified all four on a 390×844 viewport
  and added regression coverage. Fixed hex→text decoding to honor the tab encoding.*
- *Hardening pass: theme-editor live preview now reverts on close (no leaked inline
  overrides); hex row geometry scales with the editor font size (em-based columns +
  `--hex-row-height`, virtualization kept in sync); saving image adjustments to a format
  that cannot store them now warns before writing the unedited original; editor shortcuts
  are gated to editor focus so they no longer hijack typing (Ctrl/Cmd+S stays global);
  Content-Security-Policy tightened from a blanket `https:` to the exact CDN/font hosts;
  dropped `user-scalable=no`/`maximum-scale` so zoom is available again (inputs stay 16px);
  per-keystroke tab refresh is now incremental instead of rebuilding the whole tab bar and
  re-scanning icons; and added CodeMirror addons — comment (Ctrl/Cmd+/), closetag +
  matchtags (with xml-fold) and continuelist. All verified via the CDP harness with zero
  exceptions, plus a full regression pass on a clean profile.*
