# MYNOTEPAD++ — Implementation Task Plan

**Spec Reference:** `docs/FUNCTIONAL_SPECIFICATION_SWIFT.md` v2.0
**Target:** macOS 14.0+ · Apple Silicon · Swift 6 · Xcode
**Total Phases:** 16 · **Estimated Tasks:** 180+

> Each task has a verification checkpoint. A phase is not complete until ALL
> checkpoints pass. Tasks within a phase are ordered — do not skip ahead.

---

## PHASE 1: Project Scaffolding & Build System

**Goal:** Empty Xcode project that builds, runs, shows a blank window, and has all SPM
dependencies resolving. CI pipeline green.

| # | Task | Verification |
|---|------|-------------|
| 1.1 | Create Xcode project `MyNotepadPP` (macOS App, Swift, AppKit lifecycle) | Project opens in Xcode without errors |
| 1.2 | Set deployment target macOS 14.0, Swift 6.0, `SWIFT_STRICT_CONCURRENCY = complete`, `SWIFT_DEFAULT_ACTOR_ISOLATION = nonisolated` | Build Settings show correct values |
| 1.3 | Enable Hardened Runtime + App Sandbox in target signing | Entitlements file exists with sandbox keys |
| 1.4 | Create `MyNotepadPP.entitlements` with: `app-sandbox`, `files.user-selected.read-write`, `files.bookmarks.app-scope`, `print` | Entitlements XML matches spec §14 |
| 1.5 | Add `PrivacyInfo.xcprivacy` with FileTimestamp, DiskSpace, SystemBootTime, UserDefaults reasons | File matches spec §14 exactly |
| 1.6 | Add SPM dependencies: GRDB.swift 7.0+, swift-collections 1.1+, zstd 1.5+, swift-atomics 1.2+ | `swift package resolve` succeeds |
| 1.7 | Create directory structure matching spec §2 module layout (App/, Core/, Services/, Views/, ViewModels/, Resources/, TreeSitter/) | All directories exist, empty `.swift` placeholder files compile |
| 1.8 | Add tree-sitter C source to `TreeSitter/` directory with `module.modulemap` | `import TreeSitter` compiles in a test file |
| 1.9 | Add `MainMenu.xib` with minimal menu bar (File, Edit, View, Find, Window, Help) | App launches, shows menu bar |
| 1.10 | Create `AppDelegate.swift` with `@MainActor` annotation, blank `NSWindow` | App launches, blank window visible |
| 1.11 | Create `Localizable.strings` with placeholder entries for all error keys from spec §11 | `NSLocalizedString("error.io", comment: "")` resolves |
| 1.12 | Add `Assets.xcassets` with app icon placeholder | App shows icon in Dock |
| 1.13 | Set up GitHub Actions CI: `macos-14` runner, `xcodebuild test`, SwiftLint | CI builds and passes (no tests yet, but compiles) |
| 1.14 | Add `.swiftlint.yml` configuration | `swiftlint lint` passes on empty project |
| 1.15 | Add libgit2 via SwiftGit2 SPM wrapper — needed by GitService + .gitignore filtering (DEFECT-24) | `import SwiftGit2` compiles |
| 1.16 | Add `LICENSE` file — GPL v3 with Section 7 App Store exception text | License file in repo root |
| 1.17 | Create privacy policy URL placeholder (required even with zero data collection) | URL accessible, linked in App Store metadata |
| 1.18 | Set up CLA for contributors (CLA Assistant or DCO sign-off) | CLA bot configured on repo |
| 1.19 | Configure release build architectures — arm64 (dev), arm64 + x86_64 universal (release) | `xcodebuild -configuration Release` produces universal binary |
| 1.20 | Tag: `scaffold-complete`. Verify: `xcodebuild -scheme MyNotepadPP -configuration Release` succeeds | Clean release build, app launches |

---

## PHASE 2: Core Text Buffer Engine

**Goal:** Piece table with red-black tree, snapshots, line index — all with comprehensive
unit tests. No UI yet. Pure data structures.

| # | Task | Verification |
|---|------|-------------|
| 2.1 | Implement `RedBlackTree<Value>` in `Core/TextBuffer/RedBlackTree.swift` — index-based nodes (flat array), insert, delete, rebalance, in-order traversal | Unit test: 10,000 random insertions + deletions, tree stays balanced (black-height invariant) |
| 2.2 | Add augmentation fields: `subtreeLength`, `subtreeLines` with propagation on insert/delete/rotate | Unit test: after every mutation, walk tree and verify augmentation matches actual subtree sums |
| 2.3 | Implement `Piece`, `PieceSource` enum, `OriginalBuffer` (immutable `Sendable` class) | Compiles with `SWIFT_STRICT_CONCURRENCY = complete` |
| 2.4 | Implement `BufferBuilder` — append chunks, `seal()` to produce `OriginalBuffer` (no force cast) | Unit test: build from 100 chunks, seal, verify byte-exact match |
| 2.5 | Implement `AddBufferStore` — append-only, shared across snapshots, `length` tracking | Unit test: append 1000 times, verify length, verify old snapshot's `addBytesLength` unchanged |
| 2.6 | Implement `PieceTable` struct — `original`, `addStore`, `pieces` (PieceTree), `totalLength`, `totalLines`, `generation` counter | Unit test: construct from OriginalBuffer, verify initial state |
| 2.7 | Implement `PieceTable.insert(at:text:)` — split piece, insert into add buffer, update tree | Unit test: insert at start, middle, end of 1MB buffer; verify content via `getText()` |
| 2.8 | Implement `PieceTable.delete(range:)` — split at boundaries, remove middle pieces | Unit test: delete ranges, verify content, verify line count updates |
| 2.9 | Implement `PieceTable.getText(range:)` — walk tree, concatenate piece bytes | Unit test: random ranges, compare with naive string implementation |
| 2.10 | Implement `PieceTable.getLine(_ n:)` — O(log n) via augmented tree | Unit test: load 100K-line file, verify every line matches naive split |
| 2.11 | Implement `LineIndex` helper — `rowCol(forByteOffset:)`, `byteOffset(forRow:col:)` | Unit test: round-trip byte→row/col→byte for all positions in a multi-line buffer |
| 2.12 | Implement `Position.swift` — `struct Position { var line: Int; var column: Int }` | Compiles, used by LineIndex |
| 2.13 | Implement `PieceTableSnapshot` — immutable class, `@unchecked Sendable`, captures `addBytesLength` from `AddBufferStore` | Unit test: take snapshot, mutate original, snapshot unchanged |
| 2.14 | Implement `PieceTableSnapshot.readUTF8(at:length:)` — C pointer for tree-sitter callback | Unit test: read bytes at various offsets, verify against known content |
| 2.15 | Implement piece table compaction — materialize new `OriginalBuffer` + fresh `AddBufferStore` from current state | Unit test: 15,000 edits → compact → verify content identical, generation reset |
| 2.16 | Performance test: 100,000 sequential inserts < 500ms, 100,000 random inserts < 2s | `XCTestCase.measure {}` passes thresholds |
| 2.17 | Performance test: `getLine()` on 1M-line buffer < 1µs per call | Measure block passes |
| 2.18 | Tag: `core-buffer-complete` | All 15+ unit tests pass, zero warnings |

---

## PHASE 3: Concurrency Infrastructure & App State

**Goal:** Actor-based service skeleton, shutdown guard, task replacement pattern —
all compiling under strict concurrency with zero warnings.

| # | Task | Verification |
|---|------|-------------|
| 3.1 | Implement `AppState` singleton — `ManagedAtomic<Bool>` shutdown flag, `beginShutdown()` synchronous cancel | Unit test: set flag, read from background Task, verify `true` |
| 3.2 | Implement `DocumentManager` actor — holds `PieceTable`, publishes snapshots via `AsyncStream` | Unit test: edit → receive snapshot on stream |
| 3.3 | Implement `DocumentManager.edit(range:text:)` — returns new `PieceTableSnapshot`, increments generation | Unit test: edit, verify snapshot generation matches |
| 3.4 | Implement `DocumentManager.undo()` / `redo()` — snapshot-based swap | Unit test: edit → undo → verify original content restored |
| 3.5 | Implement `DocumentManager.applyCompaction()` — actor-isolated buffer swap | Unit test: compact, verify new OriginalBuffer, empty AddBufferStore |
| 3.6 | Implement `Document` class (`@MainActor`) with `nonisolated let id`, `makeSnapshot()` method | Compiles under strict concurrency, `DocumentSnapshot` is `Sendable` |
| 3.7 | Implement `UndoStack` with 1,000 entry / 100MB cap, FIFO eviction | Unit test: push 1,500 entries, verify oldest 500 evicted |
| 3.8 | Implement `UndoEntry` enum — `singleEdit`, `groupedEdits` | Compiles, snapshot references correct |
| 3.9 | Implement task replacement pattern helper — `TaskSlot<T>` that cancels previous on new assignment | Unit test: assign 100 tasks rapidly, verify only last runs to completion |
| 3.10 | Implement starvation guard — counter per document, 4th attempt runs to completion | Unit test: cancel 3 times, verify 4th completes, counter resets |
| 3.11 | Implement thermal throttling — read `ProcessInfo.thermalState`, adjust `maxConcurrency` | Manual: verify value changes when thermal state simulated |
| 3.12 | Implement `EditorError` enum with all 11 cases + localized descriptions (GAP-23) | Unit test: every case produces non-empty localized string |
| 3.13 | Implement `Progress` enum — `indeterminate`, `determinate(Double, String)` | Compiles, used by services |
| 3.14 | Implement `Constants.swift` — all magic numbers from spec (debounce times, caps, thresholds) | All constants referenced by name, not literal |
| 3.15 | Implement multi-window same-file buffer sharing (GAP-15) — one `PieceTable` per unique file path, shared via `AsyncStream`, per-tab cursor/scroll positions, reference counting for buffer lifecycle | Unit test: open same file in 2 tabs, edit in one, verify snapshot arrives in the other; close one tab, buffer stays alive |
| 3.16 | Tag: `concurrency-infra-complete` | Zero concurrency warnings, all tests pass |

---

## PHASE 4: File I/O — Loading, Saving, Encoding

**Goal:** Open any text file, detect encoding, progressive load for large files,
three-tier atomic save. All tested with real files.

| # | Task | Verification |
|---|------|-------------|
| 4.1 | Implement `EncodingDetector` — BOM, XML charset, CFStringEncoding heuristic, locale fallback | Unit test: detect UTF-8, UTF-16 LE/BE, Shift-JIS, ISO-8859-1 from sample files |
| 4.2 | Implement `LineEndingDetector` — detect LF, CRLF, CR from first 8KB | Unit test: all 3 line ending types detected correctly |
| 4.3 | Implement `BinaryDetector` — check for null bytes in header | Unit test: PNG = binary, Swift file = text |
| 4.4 | Implement `FileLoader` actor — `load(url:)` → `LoadResult(pieceTable, encoding, lineEnding)` | Unit test: load a 1KB UTF-8 file, verify content |
| 4.5 | Implement `FileLoader.loadChunked()` — 64KB chunks via `FileHandle.read`, `BufferBuilder` | Unit test: load 10MB file, verify byte-exact match |
| 4.6 | Implement progressive loading — first chunk → partial display, typing disabled until complete (GAP-06) | Unit test: load 50MB file, verify partial callback fires before completion |
| 4.7 | Implement encoding conversion — `convert(chunk:from:to:)` for all supported encodings | Unit test: round-trip UTF-8↔UTF-16↔Shift-JIS |
| 4.8 | Implement `FileSaver` — three-tier atomic write (rename → copy-new-rename → recovery) | Unit test: save to local, verify Tier 1 atomic rename |
| 4.9 | Implement `F_FULLFSYNC` via `Darwin.fcntl(fd, F_FULLFSYNC, 0)` in Tier 1 | Verify: save, check file content matches |
| 4.10 | Implement Tier 2 — copy-bak → write-new → rename → delete-bak | Unit test: simulate EXDEV, verify Tier 2 path taken |
| 4.11 | Implement Tier 3 — write to `<appSupport>/recovery/` + banner flag | Unit test: simulate ENOSPC, verify recovery file created |
| 4.12 | Implement `isCloudOrNonLocalVolume()` — volumeIsLocalKey + iCloud path + user config | Unit test: local path → false, iCloud path → true |
| 4.13 | Implement `NSFileCoordinator` wrapper with 10s timeout for iCloud (DEFECT-15) | Unit test: mock coordinator, verify timeout fires |
| 4.14 | Implement mount-point capability cache — test rename on first save, cache result | Unit test: first save tests, second save uses cache |
| 4.15 | Implement encoding/line-ending change UX data flow — re-encode buffer, mark dirty (DEFECT-31) | Unit test: change encoding, verify buffer re-encoded |
| 4.16 | Implement all 20+ encoding conversions — UTF-8, UTF-8 BOM, UTF-16 LE/BE, ASCII, ISO-8859-1 through -15, Windows-1250 through -1258, Shift-JIS, EUC-JP, EUC-KR, GB2312, Big5, KOI8-R, MacRoman | Unit test: load + save round-trip for every encoding |
| 4.17 | Implement 30s timeout for file operations on network filesystems — wrap FileHandle.read/write with `withTimeout(seconds: 30)`, show non-blocking banner on timeout | Unit test: simulate slow read, verify timeout fires |
| 4.18 | Implement mmap sliding window for files >256MB (GAP-11) — 64MB `MAP_PRIVATE` read-only window, `AddBufferStore` stays in-memory | Unit test: open 500MB file, verify only 64MB mapped at a time |
| 4.19 | Implement Mach exception handler for SIGBUS on mmap — unmaps faulting page, returns empty buffer, shows "read error" gutter marker | Unit test: truncate mmapped file, verify graceful degradation (no crash) |
| 4.20 | Performance test: load 100MB file < 2s, first screen < 200ms | Measure block passes |
| 4.21 | Performance test: save 100MB file < 50ms (snapshot → atomic write) | Measure block passes |
| 4.22 | Tag: `file-io-complete` | All tests pass, all encodings work |

---

## PHASE 5: SQLite Session Persistence

**Goal:** Session DB with schema, crash recovery, periodic backup, write batching —
all tested including simulated crash scenarios.

| # | Task | Verification |
|---|------|-------------|
| 5.1 | Create `SessionService` actor — open GRDB `DatabaseQueue` + `DatabasePool` | DB file created at `<appSupport>/sessions/sessions.db` |
| 5.2 | Set all PRAGMAs from spec §7 on every connection open | Verify: `PRAGMA journal_mode` returns `wal` |
| 5.3 | Create schema — `windows`, `tabs`, `unsaved_content`, `undo_history`, `app_state` (all STRICT) | Verify: `sqlite3 sessions.db ".schema"` matches spec |
| 5.4 | Create index `idx_tabs_window_sort ON tabs(window_id, sort_order)` | Verify: index exists in schema |
| 5.5 | Implement `dirty_flag` upsert — `INSERT OR REPLACE` on startup (DEFECT-19) | Unit test: first launch creates row, second launch detects previous value |
| 5.6 | Implement crash recovery — check previous `dirty_flag`, `PRAGMA quick_check`, `sqlite3_recover` fallback | Unit test: set dirty_flag=1 manually, restart, verify recovery triggers |
| 5.7 | Implement clean exit — `PRAGMA optimize`, `incremental_vacuum`, `wal_checkpoint(TRUNCATE)`, set dirty_flag=0 | Unit test: clean exit, verify dirty_flag=0 |
| 5.8 | Implement tab state save — insert/update tab on open/close, immediate write | Unit test: open tab, close app, reopen, verify tab restored |
| 5.9 | Implement cursor/scroll batching — in-memory accumulator, flush every 5s in single transaction | Unit test: 100 cursor updates → 1 transaction |
| 5.10 | Implement `unsaved_content` — store compressed (zstd) buffer for dirty untitled tabs (DEFECT-17) | Unit test: close untitled dirty tab, verify content in DB |
| 5.11 | Implement `commitEmergencySession()` — synchronous, <50ms, just tab IDs + file paths (DEFECT-28) | Unit test: measure < 50ms for 50 tabs |
| 5.12 | Implement `hotExit()` — full state + dirty buffers in single `BEGIN IMMEDIATE` transaction | Unit test: 50 tabs with dirty content, < 500ms |
| 5.13 | Implement periodic backup — `VACUUM INTO` staging → rename (DEFECT-20), skip if in-progress | Unit test: trigger backup, verify .backup file exists |
| 5.14 | Implement window state save/restore — position, size, fullscreen flag (GAP-20) | Unit test: save window state, restore, verify coordinates |
| 5.15 | Tag: `session-db-complete` | All tests pass, crash recovery works |

---

## PHASE 6: Auto-Save & Backup Systems

**Goal:** Three independent save systems running in parallel — debounce, continuous
backup, hot exit. Zero data loss. Zero hangs.

| # | Task | Verification |
|---|------|-------------|
| 6.1 | Implement `AutoSaveService` actor — debounce tasks, throttle tasks, active saves tracking | Compiles under strict concurrency |
| 6.2 | Implement `documentDidChange()` — 1s debounce, skip-if-busy | Unit test: rapid edits, only 1 save fires after 1s quiet |
| 6.3 | Implement `startThrottle()` — 30s periodic save while document is dirty | Unit test: dirty doc, verify save every 30s |
| 6.4 | Implement `drainAndSave()` — wait for in-flight + final save, 2s timeout (GAP-02) | Unit test: simulate stuck save, verify timeout at 2s |
| 6.5 | Implement `stopTracking()` — cancel debounce + throttle for closed doc | Unit test: close doc, verify no further saves fire |
| 6.6 | Implement `windowDidResignKey()` — save all dirty docs, GAP-22 completion tracking | Unit test: resign key, verify all dirty docs saved |
| 6.7 | Implement `cancelAllFireAndForget()` — lock-protected immediate cancellation (GAP-04) | Unit test: cancel during active save, verify cancelled |
| 6.8 | Implement cloud-aware debounce — 3s for cloud paths instead of 1s | Unit test: iCloud path → 3s debounce |
| 6.9 | Implement `BackupService` actor — continuous backup every 500ms to `<appSupport>/backups/{doc_id}/` | Unit test: dirty doc, verify backup file created within 1s |
| 6.10 | Implement backup cleanup — on save: delete backup; on untitled close: SQLite first then delete (DEFECT-17) | Unit test: save → backup deleted; untitled close → SQLite → backup deleted |
| 6.11 | Implement startup backup scan — scan `backups/` directory, restore if backup is newer than original file, delete if stale (>7 days) | Unit test: create stale backup, launch, verify deleted; create newer backup, launch, verify restored |
| 6.12 | Implement orphan cleanup — startup + hourly, delete temp files >1hr with dead PIDs | Unit test: create orphan with dead PID, trigger cleanup, verify deleted |
| 6.12 | Implement recovery file permissions — `chmod 0600` immediately after creation | Unit test: create recovery file, verify permissions |
| 6.13 | Implement `DIRTY_CRITICAL` state — all tiers fail → persistent banner → poll every 10s | Unit test: simulate disk full, verify banner flag set |
| 6.14 | Implement file watcher — FSEvents with `kFSEventStreamEventFlagOwnEvent` self-suppression | Unit test: save file, verify no external change notification |
| 6.15 | Implement external file modification detection (GAP-14) — auto-reload if clean, banner if dirty | Unit test: modify file externally, verify correct behavior for clean vs dirty buffer |
| 6.16 | Implement deleted file detection — banner with Save As / Close options | Unit test: delete file externally, verify banner flag |
| 6.17 | Integration test: rapid typing (10 chars/s for 60s) → quit → relaunch → verify all content restored | Pass with zero data loss |
| 6.18 | Integration test: SIGKILL during typing → verify continuous backup has < 500ms data loss | Backup timestamp within 500ms of kill |
| 6.19 | Tag: `autosave-complete` | All tests pass, zero data loss scenarios |

---

## PHASE 7: Metal Rendering Engine

**Goal:** Custom `NSView` with CoreText text shaping + Metal rendering. Scrollable,
60 FPS, dirty-rect only. No syntax colors yet (plain white text).

| # | Task | Verification |
|---|------|-------------|
| 7.1 | Create `EditorView` — `NSView` subclass, `@MainActor`, `CAMetalLayer` backing | View appears in window, grey background |
| 7.2 | Set up Metal device, command queue, `CAMetalLayer` configuration | `MTLCreateSystemDefaultDevice()` succeeds |
| 7.3 | Compile Metal shaders from `.metal` source → `.metallib` at build time, load at startup | PSOs created in `viewDidMoveToWindow`, zero runtime compilation |
| 7.4 | Implement triple-buffer semaphore — `DispatchSemaphore(value: 3)`, 100ms timeout (GAP-03) | Frame renders without blocking; timeout path tested |
| 7.5 | Implement glyph atlas — `MTLTexture`, `(glyphID, fontID, fontSize, subpixelOffset)` key, 32MB LRU | Glyphs render on screen |
| 7.6 | Implement separate RGBA atlas for Apple Color Emoji (`CTFontDrawGlyphs`) | Emoji render correctly |
| 7.7 | Implement `LineLayoutCache` — nested dictionary, O(1) invalidation per line (GAP-08) | Unit test: invalidate line, verify only that line re-shaped |
| 7.8 | Implement `draw(_ dirtyRect:)` — query snapshot for visible lines, shape with CoreText on cache miss, encode Metal draw commands | Text renders on screen |
| 7.9 | Implement dirty rect rendering — only re-render changed lines + cursor line on edit | Profile: single-char edit does NOT redraw entire viewport |
| 7.10 | Implement overdraw buffer — pre-render 2x viewport height | Fast scroll never shows blank |
| 7.11 | Implement viewport culling — only compute layout for visible lines + overdraw | Profile: 1M-line file, only ~100 lines laid out per frame |
| 7.12 | Implement scrolling — `NSScroller`, pixel-level smooth scroll, `scrollWheel(with:)` | Scroll works smoothly |
| 7.13 | Implement line numbers gutter — draw logical line numbers left of text | Line numbers visible, correct |
| 7.14 | Implement cursor rendering — blinking cursor as Metal overlay, `NSTimer` for blink | Cursor visible, blinks |
| 7.15 | Implement selection rendering — highlight range as Metal overlay (semi-transparent rect) | Click + drag selects text, highlight visible |
| 7.16 | Implement long line handling — column-level virtualization for lines >10,000 chars | Load file with 50K-char line, renders without lag |
| 7.17 | Implement glyph atlas scale change — invalidate on `contentsScale` change | Move window to Retina ↔ non-Retina, glyphs re-render correctly |
| 7.18 | Implement frame budget watchdog (debug builds) — 16ms budget, drop syntax colors if exceeded, never drop below 60 FPS | In debug: simulate heavy frame, verify syntax colors dropped + FPS maintained |
| 7.19 | Implement glyph atlas tab-close cleanup — deferred compaction with 5s debounce after tab close | Unit test: close tab, verify compaction fires after 5s, not immediately |
| 7.20 | Performance test: scroll 60 FPS on 1M-line file (Instruments Time Profiler) | Sustained 60 FPS |
| 7.21 | Performance test: cold startup to first frame < 500ms | Measure from launch to `draw()` call |
| 7.22 | Tag: `metal-render-complete` | Text renders, scrolls at 60 FPS |

---

## PHASE 8: Text Input & Editing

**Goal:** Full keyboard input, IME support, cursor movement, selection, clipboard,
multi-cursor. Connected to DocumentManager actor.

| # | Task | Verification |
|---|------|-------------|
| 8.1 | Implement `NSTextInputClient` on `EditorView` — `insertText`, `setMarkedText`, `markedRange`, `selectedRange` | Type ASCII text, appears on screen |
| 8.2 | Implement `attributedSubstring(forProposedRange:actualRange:)` | IME candidate window shows correct context |
| 8.3 | Implement `firstRect(forCharacterRange:)` — map character position to screen rect | IME candidate window positioned correctly |
| 8.4 | Implement `characterIndex(for:)` — partitioned glyph cache, O(log n) binary search (GAP-09) | Mouse click positions cursor correctly |
| 8.5 | Implement `inputContext?.invalidateCharacterCoordinates()` on scroll/layout change | IME stays positioned after scroll |
| 8.6 | Implement CJK IME — marked text rendering (underline), composition commit | Type Chinese/Japanese/Korean, correct composing + commit |
| 8.7 | Implement emoji picker — `Ctrl+Cmd+Space` → `NSTextInputClient` handles it | Emoji picker works, emoji inserted |
| 8.8 | Implement cursor movement — arrow keys (char/line), `Opt+arrow` (word), `Cmd+arrow` (line start/end, doc start/end) | All navigation shortcuts work |
| 8.9 | Implement selection — `Shift+arrow` variants, `Cmd+A` (select all), double-click (word), triple-click (line) | All selection modes work |
| 8.10 | Implement clipboard — `Cmd+C` copy, `Cmd+V` paste, `Cmd+X` cut (read clipboard only on explicit action) | Copy/paste works, clipboard not polled |
| 8.11 | Implement `Cmd+Z` undo / `Cmd+Shift+Z` redo — connected to `UndoStack` via `DocumentManager` | Undo/redo works for single edits |
| 8.12 | Implement multi-cursor — `Cmd+click` to add cursor, type at all cursors simultaneously | Multiple cursors type in parallel |
| 8.13 | Implement multi-cursor selection — `Cmd+D` to select next occurrence | Select next occurrence works |
| 8.14 | Implement indent/outdent — `Tab` / `Shift+Tab` on selection | Block indent/outdent works |
| 8.15 | Implement auto-indent — maintain indent level on Enter | New line matches previous indent |
| 8.16 | Implement bracket auto-close — `(`, `[`, `{`, `"`, `'` | Type `(`, get `()` with cursor inside |
| 8.17 | Implement system text replacement — `NSTextInputClient` handles it | macOS text replacements work (e.g., `omw` → `On my way!`) |
| 8.18 | Connect editing to `DocumentManager.edit()` → snapshot → `EditorView.setNeedsDisplay(dirtyRect)` | Type → text appears with minimal latency |
| 8.19 | Performance test: keystroke latency < 16ms (type → pixel on screen) | Instruments shows < 16ms per keystroke |
| 8.20 | Tag: `text-input-complete` | Full editing, IME, multi-cursor all work |

---

## PHASE 9: Syntax Highlighting — Tree-sitter

**Goal:** Incremental syntax highlighting for 50+ languages via tree-sitter C interop.
Highlight queries, language injection, starvation guard.

| # | Task | Verification |
|---|------|-------------|
| 9.1 | Implement `SyntaxEngine` — `ts_parser_new()`, `ts_parser_delete()`, `setLanguage()` | Parser creates/destroys without leak |
| 9.2 | Implement `SyntaxTree` — wraps `OpaquePointer`, `deinit` calls `ts_tree_delete()` | No memory leaks in Instruments |
| 9.3 | Implement `SyntaxEngine.parse(source:oldTree:)` — `TSInput` with `PieceTableSnapshot.readUTF8` callback | Parse a Swift file, tree is non-null |
| 9.4 | Implement `EditMapping.makeInputEdit()` — exact `TSInputEdit` field computation (DEFECT-25) | Unit test: insert/delete → verify `start_byte`, `old_end_byte`, `new_end_byte`, all `TSPoint` fields |
| 9.5 | Implement `EditMapping.applyMultiCursorEdits()` — reverse byte order (DEFECT-25) | Unit test: 3 cursor edits, verify tree-sitter gets correct offsets |
| 9.6 | Implement incremental parsing — `ts_tree_edit()` → `ts_parser_parse()` with old tree | Unit test: edit in middle of file, verify only affected nodes re-parsed |
| 9.7 | Implement `LanguageRegistry` — lazy-load grammar `.c` files, cache `OpaquePointer` per language | Load Swift grammar, verify pointer non-null |
| 9.8 | Implement `HighlightQuery` — load `.scm` highlight files, run `ts_query_new()`, map captures to semantic tokens | Parse Swift, get `keyword`, `string`, `comment` tokens |
| 9.9 | Implement highlight → Metal color mapping — theme colors applied via shader uniform, not baked into atlas | Text renders with syntax colors |
| 9.10 | Implement `InjectionEngine` — `injections.scm` queries, `mergeHighlights()` (DEFECT-26) | HTML file: `<script>` content highlighted as JS |
| 9.11 | Implement unloaded grammar fallback — plain-token highlight + async grammar load | Open HTML with rare injection, verify parent tokens shown, then injection loads |
| 9.12 | Implement starvation guard — 3 cancels → 4th runs to completion, stale check, re-queue if stale (DEFECT-05) | Unit test: simulate 4 rapid edits, verify 4th parse completes |
| 9.13 | Implement syntax task replacement — `syntaxTask?.cancel()` + new task on each edit (DEFECT-09) | Profile: only 1 syntax task per edit, not N |
| 9.14 | Implement `BracketEngine` — bracket matching via tree-sitter node lookup | Cursor on `(`, matching `)` highlighted |
| 9.15 | Implement `FoldingEngine` — foldable regions from tree-sitter node types | Fold markers appear for functions/classes |
| 9.16 | Implement `SymbolIndex` — build from tree-sitter nodes, support Go To Symbol | Symbol list populated for a Swift file |
| 9.17 | Bundle grammars for top 15 languages: Swift, C, C++, Python, JavaScript, TypeScript, HTML, CSS, JSON, YAML, Markdown, Rust, Go, Java, Ruby | All 15 parse and highlight correctly |
| 9.18 | Bundle remaining 35+ grammars (Bash, SQL, Lua, PHP, etc.) | All grammars load without crash |
| 9.19 | Performance test: syntax highlight after edit < 50ms viewport | Instruments measure on 10K-line Swift file |
| 9.20 | Disable syntax highlighting for lines > 500,000 chars | Open file with mega-line, no crash or hang |
| 9.21 | Tag: `syntax-complete` | 50+ languages highlight, incremental, injection works |

---

## PHASE 10: Search Engine

**Goal:** Single-document search (literal + regex), multi-file search (parallel, capped),
find & replace, multi-file replace with preview.

| # | Task | Verification |
|---|------|-------------|
| 10.1 | Implement `BufferSearch.searchLiteral()` — `Data.range(of:)`, case sensitive/insensitive, whole word | Unit test: find all occurrences of "func" in a Swift file |
| 10.2 | Implement `BufferSearch.searchRegex()` — `NSRegularExpression` with `RegexCache` (LRU 50) | Unit test: regex `\bfunc\s+\w+` finds all function names |
| 10.3 | Implement `SearchQuery` struct — pattern, isRegex, caseSensitive, wholeWord, isMultiline | Compiles, all fields |
| 10.4 | Implement `SearchService` actor — multi-file search with `TaskGroup`, concurrency cap | Unit test: search 100 files, results returned |
| 10.5 | Implement 10,000 match cap — `truncated` flag, stop dispatching new files (DEFECT-22) | Unit test: search for `e` in large corpus, verify truncated at 10K |
| 10.6 | Implement overlap window for multi-line regex — `max(pattern.utf8.count * 4, 4096)` (DEFECT-23) | Unit test: regex spanning chunk boundary, match found |
| 10.7 | Implement line offset counting — newlines in new data only, not overlap (GAP-07) | Unit test: verify correct line numbers at chunk boundaries |
| 10.8 | Implement `.gitignore`-aware file collection — `git_ignore_path_is_ignored()` (DEFECT-24) | Unit test: file in .gitignore excluded from results |
| 10.9 | Implement symlink loop detection in file enumeration — track visited inodes | Unit test: create symlink loop, verify no infinite recursion |
| 10.10 | Implement cancellation — `Escape` cancels search, `Task.checkCancellation()` in loop | Press Escape mid-search, verify stops |
| 10.11 | Implement progress reporting — `AsyncStream<Progress>`, live match count | Status bar shows live progress during search |
| 10.12 | Implement incremental in-buffer search — narrow on type, widen on delete | Type in find bar, results update live |
| 10.13 | Implement Find & Replace — single file, `Cmd+H`, replace one / replace all | Replace all works, undo restores (DEFECT-36) |
| 10.14 | Implement multi-file replace — preview sheet, per-file undo groups, progress, cancel (DEFECT-30) | Replace in 10 files, verify preview, undo for open files |
| 10.15 | Performance test: literal search 10K files < 2s (warm cache) | Measure block passes |
| 10.16 | Tag: `search-complete` | All search/replace modes work |

---

## PHASE 11: UI Views

**Goal:** All UI chrome — tabs, sidebar, find bar, status bar, command palette,
minimap, preferences, split view. Connected to services.

| # | Task | Verification |
|---|------|-------------|
| 11.1 | Implement `TabBarView` — tab strip, active tab highlight, close button, dirty indicator | Tabs render, click to switch |
| 11.2 | Implement tab drag-and-drop reordering — within window, animated insertion marker (GAP-17) | Drag tab, verify reorder + sort_order updated |
| 11.3 | Implement tab drag between windows — move tab + document reference | Drag tab to another window, verify moved |
| 11.4 | Implement tab keyboard shortcuts — `Cmd+Shift+[` / `]` to move, `Cmd+1-9` to select | All shortcuts work |
| 11.5 | Implement tab context menu — Close, Close Others, Close to the Right, Pin, Unpin | All menu items work |
| 11.6 | Implement `StatusBarView` — encoding, line ending, cursor position, language, save status | Status bar shows all fields |
| 11.7 | Implement encoding/line ending change from status bar — clickable labels, popover (DEFECT-31) | Click encoding → popover → select → buffer re-encoded |
| 11.8 | Implement `FindBarView` — search field, replace field, match count, navigation arrows | Find bar appears on `Cmd+F`, functional |
| 11.9 | Implement `SearchResultsPanel` — multi-file search results tree view | Results tree shows file → matches hierarchy |
| 11.10 | Implement `SidebarView` — file tree, `Cmd+B` to toggle | Sidebar shows directory tree |
| 11.11 | Implement `CommandPaletteView` — `Cmd+Shift+P`, fuzzy search commands | Palette opens, commands searchable |
| 11.12 | Implement `GotoAnythingView` — `Cmd+P` for file, `Cmd+G` for line | Go to file / line works |
| 11.13 | Implement `MinimapView` — double-buffer with `OSAllocatedUnfairLock` (DEFECT-11, GAP-13) | Minimap renders, click to scroll |
| 11.14 | Implement `CompletionPopupView` — `NSPanel`, `.nonactivatingPanel`, max 10 items (GAP-21) | Popup appears after 3 chars, arrow keys navigate |
| 11.15 | Implement `CompletionEngine` — word-based completion, ranked by distance/frequency (GAP-21) | Completions relevant, < 50ms |
| 11.16 | Implement `SplitContainerView` — horizontal/vertical split, `Cmd+\` | Split editor, edit in both panes |
| 11.17 | Implement `PreferencesView` — font, theme, tab size, auto-save toggle, cloud paths | Preferences window opens, changes apply |
| 11.18 | Implement fullscreen — `Ctrl+Cmd+F`, native `toggleFullScreen`, state persisted (GAP-20) | Enter/exit fullscreen, state restored on relaunch |
| 11.19 | Implement recent files — `File > Open Recent`, 20 items, security-scoped bookmarks (GAP-19) | Open file → appears in recents → relaunch → still there |
| 11.20 | Implement `Cmd+N` new file, `Cmd+O` open file (NSOpenPanel), `Cmd+S` save, `Cmd+Shift+S` save as | All file menu items work |
| 11.21 | Tag: `ui-views-complete` | All views functional, connected to services |

---

## PHASE 12: Themes & Visual Polish

**Goal:** Theme system, dark/light mode, font management, cursor customization.

| # | Task | Verification |
|---|------|-------------|
| 12.1 | Implement `ThemeManager` actor — load `.json` theme files from bundle + `<appSupport>/themes/` | Default theme loads |
| 12.2 | Create default dark theme + default light theme (matches Xcode color scheme) | Both themes render correctly |
| 12.3 | Implement high-contrast theme — WCAG AAA 7:1 ratio | Color contrast checker passes |
| 12.4 | Implement color-blind theme — deuteranopia/protanopia safe, icons + color not color alone | Visually verified |
| 12.5 | Implement theme switching — `Preferences > Theme`, apply to all open editors | Switch theme, all editors update |
| 12.6 | Implement system appearance tracking — auto-switch light/dark with macOS | Change macOS appearance, app follows |
| 12.7 | Implement font selection — monospace fonts, size slider, `Cmd++` / `Cmd+-` zoom | Font change applies, zoom works |
| 12.8 | Implement cursor style — line/block/underline option in Preferences | Cursor style changes |
| 12.9 | Implement current line highlight — subtle background color on active line | Active line highlighted |
| 12.10 | Implement indent guides — vertical lines at indent levels | Guides visible in indented code |
| 12.11 | Implement whitespace rendering toggle — show spaces/tabs/newlines as symbols | Toggle works |
| 12.12 | Tag: `themes-complete` | All visual features work |

---

## PHASE 13: Advanced Features

**Goal:** Word wrap, spell check, snippets, printing, diff, EditorConfig, git integration.

| # | Task | Verification |
|---|------|-------------|
| 13.1 | Implement word wrap — `Opt+Z` toggle, soft wrap, logical line numbers (DEFECT-32) | Toggle wrap, line numbers stay logical |
| 13.2 | Implement word wrap cursor movement — `↓`/`↑` = visual lines, `Cmd+↓`/`Cmd+↑` = logical | Navigation correct in wrapped mode |
| 13.3 | Implement spell checking — `NSSpellChecker`, enabled for plain text, Metal overlay (DEFECT-33) | Misspelled words underlined in .txt file |
| 13.4 | Implement spell check per-file language — status bar control | Change language, spell check updates |
| 13.5 | Implement `SnippetManager` — tab-trigger, JSON format, tab stops $1/$2/$0 (GAP-18) | Create snippet, trigger with Tab, navigate stops |
| 13.6 | Implement print — `Cmd+P`, `NSPrintOperation`, plain text v1.0 (DEFECT-34) | Print dialog opens, output correct |
| 13.7 | Implement `DiffEngine` — histogram + Myers + patience algorithms | Unit test: diff two strings, verify hunks |
| 13.8 | Implement `WordDiff` — inline word-level diff | Unit test: word-level changes identified |
| 13.9 | Implement diff view — split left/right, hunk navigation | Open diff between two files, hunks navigable |
| 13.10 | Implement `EditorConfigParser` — read `.editorconfig`, apply indent_style, indent_size, charset, end_of_line, trim_trailing_whitespace | Unit test: parse `.editorconfig`, verify settings |
| 13.11 | Implement `TabSizeDetector` — heuristic from file content if no EditorConfig | Unit test: detect 2-space vs 4-space vs tab |
| 13.12 | Implement `GitService` — git status, diff, blame via libgit2 | Sidebar shows git status indicators |
| 13.13 | Implement .git directory protection — info banner, auto-save enabled except `.git/objects/` (DEFECT-35) | Open `.git/config`, see banner once |
| 13.14 | Implement BiDi attack protection — warn on U+202A-U+202E, U+2066-U+2069 override chars ONLY in files with a recognized source-code grammar (not plain text or Arabic/Hebrew where BiDi is legitimate) | Open Swift file with U+202E → warning; open .txt with U+202E → no warning |
| 13.15 | Implement `NSWritingToolsCoordinator` (macOS 15+) — delegate, `NSAttributedString`, replacements | Writing Tools invocable on macOS 15+ |
| 13.16 | **v1.1 ONLY (not in v1.0 scope):** Implement `MacroService` — `Cmd+Shift+R` record/stop, `Cmd+Shift+P` replay, stored as serialized `[EditOperation]` arrays | Record macro, replay, verify identical edits |
| 13.17 | Tag: `advanced-features-complete` | All features work |

---

## PHASE 14: Shutdown & Lifecycle

**Goal:** Complete shutdown sequence — tab close, window close, quit, SIGTERM.
Zero data loss, zero prompts, zero hangs.

| # | Task | Verification |
|---|------|-------------|
| 14.1 | Implement `closeTab()` — stop tracking → drainAndSave → SQLite for untitled → delete backup → remove UI | Tab closes within 200ms |
| 14.2 | Implement close last window — save session to SQLite, app stays in Dock | Close window → click Dock → window restores |
| 14.3 | Implement `applicationShouldHandleReopen` — restore session from SQLite | Click Dock icon with no window → previous session restored |
| 14.4 | Implement `applicationShouldTerminate` — emergency session → hot exit task → 3s watchdog (DEFECT-27/28) | `Cmd+Q` → clean exit within 3s |
| 14.5 | Implement `NSApplicationWillTerminate` — same hot-exit for system shutdown | System shutdown → data saved |
| 14.6 | Implement DIRTY_CRITICAL quit dialog — sole exception to "never prompt" | Simulate disk full + quit → dialog appears |
| 14.7 | Integration test: `Cmd+Q` with 50 dirty tabs → relaunch → all content restored | Pass with zero data loss |
| 14.8 | Integration test: `Cmd+W` during active auto-save → no data loss | Edit saved correctly |
| 14.9 | Integration test: system shutdown signal → all tabs restored on relaunch | Pass |
| 14.10 | Tag: `lifecycle-complete` | All shutdown paths tested |

---

## PHASE 15: Accessibility

**Goal:** Full VoiceOver support, keyboard-only operation, WCAG compliance.

| # | Task | Verification |
|---|------|-------------|
| 15.1 | Add `accessibilityLabel` / `.accessibility()` to ALL UI elements | VoiceOver reads every element |
| 15.2 | Implement editor VoiceOver navigation — by line, word, character | Navigate code with VoiceOver |
| 15.3 | Implement live regions for status bar — save status, encoding, language changes announced | VoiceOver announces status changes |
| 15.4 | Implement `isReduceMotionEnabled` — disable smooth scroll, cursor blink, minimap animation | Enable reduce motion → animations stop |
| 15.5 | Implement keyboard-only operation — every feature reachable without mouse | Tab through every UI element |
| 15.6 | Implement diff hunks as `NSAccessibilityGroup` | VoiceOver describes diff hunks |
| 15.7 | Implement spell-check underline announcement — "possible spelling error" | VoiceOver announces on underlined word |
| 15.8 | Manual test: complete VoiceOver walkthrough of all features | All features accessible |
| 15.9 | Tag: `accessibility-complete` | VoiceOver audit passes |

---

## PHASE 16: Security Hardening

**Goal:** All security rules from spec §14 enforced. URL scheme handler,
path sanitization, bookmark limits.

| # | Task | Verification |
|---|------|-------------|
| 16.1 | Implement path sanitization — `URL.resolvingSymlinksInPath()`, reject `..` traversal | Unit test: `../../../etc/passwd` rejected |
| 16.2 | Implement URL scheme handler — `mynotepadpp://open?file=...` + confirmation dialog | Unit test: scheme opens dialog, system dirs rejected |
| 16.3 | Implement URL scheme rate limiting — 1 request/second | Rapid requests throttled |
| 16.4 | Implement security-scoped bookmark tracking — warn at 200, banner at 240, NSOpenPanel at 250 | Simulate 250 bookmarks, fallback triggers |
| 16.5 | Implement sandbox container paths — all internal paths via `FileManager.default.urls(for:in:)` | Grep: no hardcoded `~/Library` paths |
| 16.6 | Implement clipboard read restriction — only on `Cmd+V`, `Cmd+E` | Clipboard not read on app launch or focus |
| 16.7 | Verify: no `com.apple.security.cs.allow-jit` in entitlements | Entitlements audit passes |
| 16.8 | Verify: no `com.apple.security.network.client` in v1.0 entitlements | Entitlements audit passes |
| 16.9 | Tag: `security-complete` | All security rules verified |

---

## PHASE 17: Performance Optimization & Profiling

**Goal:** Hit every performance target from spec §13. Profile with Instruments.

| # | Task | Verification |
|---|------|-------------|
| 17.1 | Profile cold startup — target < 500ms to first frame | Instruments Time Profiler |
| 17.2 | Profile keystroke latency — target < 16ms | Signpost timing in edit path |
| 17.3 | Profile scroll FPS — target 60 FPS sustained on 1M-line file | Instruments Core Animation FPS |
| 17.4 | Profile memory idle (1 file) — target < 75MB RSS | `mach_task_info` measurement |
| 17.5 | Profile memory (50 tabs) — target < 300MB non-content RSS | Memory graph in Instruments |
| 17.6 | Profile tab switch — target < 30ms | Signpost timing |
| 17.7 | Profile hot exit (50 tabs) — target < 500ms | Signpost timing |
| 17.8 | Profile auto-save (100KB file) — target < 50ms | Signpost timing |
| 17.9 | Profile search 10K files literal — target < 2s warm cache | Measure block |
| 17.10 | Implement runtime RSS monitoring — `mach_task_info` every 10s, eviction at 250/300/350MB | Simulate high memory, verify eviction triggers |
| 17.11 | Profile open 1GB file — target < 10s total, < 200ms first screen | Measure block |
| 17.12 | Optimize any metric that fails targets | All targets met |
| 17.13 | Tag: `performance-complete` | All §13 targets verified |

---

## PHASE 18: Testing & QA

**Goal:** Comprehensive test suite — unit tests, UI tests, manual QA checklist.

| # | Task | Verification |
|---|------|-------------|
| 18.1 | Write unit tests for all Core/ modules — PieceTable, RedBlackTree, UndoStack, SearchEngine, SyntaxEngine | > 80% code coverage on Core/ |
| 18.2 | Write unit tests for all Services/ — DocumentManager, AutoSaveService, SessionService, FileLoader, FileSaver | > 70% code coverage on Services/ |
| 18.3 | Write XCUITest: open file, type text, save, close, reopen, verify content | UI test passes |
| 18.4 | Write XCUITest: find & replace, undo, verify | UI test passes |
| 18.5 | Write XCUITest: tab operations — new, close, switch, reorder | UI test passes |
| 18.6 | Write XCUITest: preferences — change font, theme, verify applied | UI test passes |
| 18.7 | Write crash recovery test — corrupt dirty_flag, launch, verify recovery | Test passes |
| 18.8 | Write encoding round-trip tests — all 20+ encodings | All encodings survive save → close → reopen |
| 18.9 | Manual QA: VoiceOver full walkthrough | Pass |
| 18.10 | Manual QA: 60 FPS scroll on 100MB file (Instruments) | Pass |
| 18.11 | Manual QA: cold start < 500ms | Pass |
| 18.12 | Manual QA: crash recovery — force quit during editing → relaunch | Content restored |
| 18.13 | Manual QA: iCloud file save/load | Works |
| 18.14 | Manual QA: external USB drive file save/load | Works |
| 18.15 | Tag: `qa-complete` | All tests pass, all manual checks pass |

---

## PHASE 19: Distribution & Release

**Goal:** Signed, notarized `.dmg`, Homebrew cask formula, App Store submission ready.

| # | Task | Verification |
|---|------|-------------|
| 19.1 | Configure code signing with Developer ID certificate | `codesign --verify` passes |
| 19.2 | Submit for notarization — `xcrun notarytool submit` | Notarization succeeds |
| 19.3 | Create `.dmg` with app + Applications alias | DMG opens, drag-to-install works |
| 19.4 | Create Homebrew cask formula — `mynotepadpp.rb` | `brew install --cask mynotepadpp` works |
| 19.5 | Prepare App Store submission — screenshots, description, privacy policy URL | Ready for Xcode Organizer upload |
| 19.6 | Create GitHub Release — tag `macos-v1.0.0`, attach `.dmg`, write release notes | Release visible on GitHub |
| 19.7 | Verify: git tag matches shipped binary (GPL v3 §6 compliance) | Binary hash matches tagged source build |
| 19.8 | Tag: `v1.0.0-release` | App shipped |

---

## PHASE DEPENDENCY MAP

```
Phase 1  ──→ Phase 2  ──→ Phase 3  ──→ Phase 4  ──→ Phase 5
(scaffold)   (buffer)     (actors)     (file I/O)   (SQLite)
                                            │
                                            ▼
Phase 7  ◄── Phase 3          Phase 6  ◄── Phase 4 + 5
(Metal)                       (auto-save)
    │                              │
    ▼                              ▼
Phase 8  ──→ Phase 9          Phase 14
(input)     (syntax)          (shutdown)
    │           │
    ▼           ▼
Phase 10     Phase 11 ──→ Phase 12 ──→ Phase 13
(search)     (UI views)   (themes)     (advanced)
                                           │
                                           ▼
                            Phase 15 ──→ Phase 16 ──→ Phase 17 ──→ Phase 18 ──→ Phase 19
                            (a11y)      (security)   (perf)       (QA)         (release)
```

**Critical path:** 1 → 2 → 3 → 4 → 7 → 8 → 9 → 11 → 18 → 19

**Can be parallelized:**
- Phase 5 (SQLite) + Phase 7 (Metal) after Phase 3
- Phase 6 (auto-save) after Phase 4 + 5
- Phase 9 (syntax) + Phase 10 (search) after Phase 8
- Phase 12 (themes) + Phase 13 (advanced) after Phase 11
- Phase 15 (a11y) + Phase 16 (security) after Phase 13

---

## TASK STATISTICS

| Phase | Tasks | Category |
|-------|-------|----------|
| 1. Scaffolding | 20 | Setup |
| 2. Text Buffer | 18 | Core |
| 3. Concurrency | 16 | Core |
| 4. File I/O | 22 | Core |
| 5. SQLite | 15 | Core |
| 6. Auto-Save | 20 | Core |
| 7. Metal Render | 22 | UI |
| 8. Text Input | 20 | UI |
| 9. Syntax | 21 | Core |
| 10. Search | 16 | Core |
| 11. UI Views | 21 | UI |
| 12. Themes | 12 | UI |
| 13. Advanced | 17 | Feature |
| 14. Shutdown | 10 | Core |
| 15. Accessibility | 9 | QA |
| 16. Security | 9 | QA |
| 17. Performance | 13 | QA |
| 18. Testing | 15 | QA |
| 19. Distribution | 8 | Release |
| **TOTAL** | **304** | |

---

*Every task traces to a section in `FUNCTIONAL_SPECIFICATION_SWIFT.md` v2.0.*
*No task is speculative. No feature is missing. This is the complete build plan.*
