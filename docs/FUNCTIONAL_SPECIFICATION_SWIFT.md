# MYNOTEPAD++ — Pure Swift Architecture Specification (macOS Only)

**Version:** 1.0  
**Date:** 2026-05-03  
**Status:** Draft  
**License:** GPL v3  
**Target:** macOS 14.0+ (Apple Silicon M-series native)  
**Language:** 100% Swift + C interop (tree-sitter only)  
**No Rust. No FFI bridge. No Cargo. Single build system (Xcode).**

---

## 1. WHY PURE SWIFT

The original architecture used Rust core + Swift UI for cross-platform reuse across 5 platforms. This document redesigns for **Mac-only** deployment, eliminating Rust entirely.

### What we eliminate

| Removed | Why |
|---------|-----|
| Rust core (`core/` crate) | No cross-platform need. Swift handles all logic natively. |
| C FFI bridge (`core/ffi/`) | No foreign language to bridge. Swift calls Swift directly. |
| `cbindgen` / C headers | No C headers needed. No bridging header. |
| Cargo build system | Xcode is the sole build system. |
| `mimalloc` allocator | macOS `libmalloc` is already SIMD-optimized for arm64 Apple Silicon. |
| `ropey` rope crate | Replaced by Swift piece table (simpler, proven by VS Code's Monaco engine). |
| `memchr` / `regex-automata` | Replaced by Foundation `Data.range(of:)` + `NSRegularExpression` (ICU engine, tuned for Apple hardware). |
| `rusqlite` | Replaced by `GRDB.swift` (or raw `sqlite3` C API built into macOS). |
| `encoding_rs` | Replaced by `CFStringEncoding` + `String.Encoding` (same ICU engine underneath). |
| Rayon thread pools | Replaced by Swift structured concurrency (`TaskGroup`, `async/await`). |

### What we gain

| Gain | Impact |
|------|--------|
| Single language | One debugger (LLDB), one profiler (Instruments), one mental model |
| Zero FFI overhead | No string conversion (`*const u8, usize` ↔ `String`) on every engine call |
| Native Instruments profiling | Full source-level Time Profiler, Allocations, Leaks — all Swift-native |
| Faster builds | No Rust release compilation (saves 2-5 minutes per build) |
| Smaller binary | No Rust stdlib linked (~5-10MB savings) |
| Xcode-native debugging | Breakpoints, memory graph, view hierarchy — everything works |
| Swift Package Manager | All dependencies via SPM |

### What stays identical

| Unchanged | Why |
|-----------|-----|
| CoreText + Metal rendering | Always was Swift/ObjC. No change. |
| NSTextInputClient (IME/emoji) | Always was AppKit. No change. |
| FSEvents file watcher | Always was macOS-native. No change. |
| App Sandbox + Hardened Runtime | Always was Xcode config. No change. |
| VoiceOver accessibility | Always was AppKit. No change. |
| All 65 user-facing features | Architecture changed, features identical. |
| All keyboard shortcuts | No user-facing change. |
| All performance targets | Same or better (no FFI overhead). |

---

## 2. ARCHITECTURE OVERVIEW

```
┌───────────────────────────────────────────────────────────┐
│                    MYNOTEPAD++ (macOS)                      │
├───────────────────────────────────────────────────────────┤
│                                                            │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              UI Layer (AppKit)                        │  │
│  │  EditorView (NSView + CoreText + Metal)              │  │
│  │  TabBarView · SidebarView · FindBarView              │  │
│  │  CommandPaletteView · MinimapView · StatusBar        │  │
│  │  PreferencesWindow · GotoAnythingView                │  │
│  └──────────────────────┬──────────────────────────────┘  │
│                         │ direct Swift calls               │
│  ┌──────────────────────▼──────────────────────────────┐  │
│  │           Service Layer (Swift Actors)                │  │
│  │  DocumentManager (actor) — single document writer    │  │
│  │  FileService · AutoSaveService · BackupService       │  │
│  │  SearchService · ThemeManager · SessionService       │  │
│  │  KeyBindingManager · SyntaxService                   │  │
│  └──────────────────────┬──────────────────────────────┘  │
│                         │ direct Swift calls               │
│  ┌──────────────────────▼──────────────────────────────┐  │
│  │           Core Engine (pure Swift, no UI deps)       │  │
│  │  TextBuffer (piece table + copy-on-write snapshots)  │  │
│  │  SyntaxEngine (tree-sitter via C interop)            │  │
│  │  SearchEngine (Foundation regex + byte scan)         │  │
│  │  UndoEngine (snapshot stack)                         │  │
│  │  FoldingEngine · CompletionEngine                    │  │
│  │  EncodingDetector · EditorConfigParser               │  │
│  │  DiffEngine (histogram + Myers + patience)           │  │
│  └──────────────────────┬──────────────────────────────┘  │
│                         │ C interop (tree-sitter only)     │
│  ┌──────────────────────▼──────────────────────────────┐  │
│  │           C Libraries (bundled, no Rust)              │  │
│  │  tree-sitter (parser generator)                      │  │
│  │  tree-sitter-{language} grammars (50+)               │  │
│  │  sqlite3 (built into macOS, or bundled modern ver)   │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                            │
└───────────────────────────────────────────────────────────┘
```

### Module Structure (Xcode Project)

```
MyNotepadPP/
├── App/
│   ├── AppDelegate.swift
│   ├── AppLifecycle.swift          ← applicationShouldHandleReopen, openFiles, etc.
│   └── main.swift
├── Core/
│   ├── TextBuffer/
│   │   ├── PieceTable.swift        ← text storage engine
│   │   ├── PieceTableSnapshot.swift ← immutable COW snapshot for background threads
│   │   ├── LineIndex.swift         ← byte↔line O(log n) lookup
│   │   └── Position.swift          ← line, column, byte offset
│   ├── Cursor/
│   │   ├── Cursor.swift
│   │   └── Selections.swift        ← multi-selection container
│   ├── Undo/
│   │   ├── UndoEngine.swift        ← snapshot-based undo/redo stack
│   │   └── UndoGroup.swift
│   ├── Search/
│   │   ├── BufferSearch.swift       ← single-document search
│   │   ├── MultiFileSearch.swift    ← parallel project search
│   │   ├── SearchQuery.swift
│   │   └── RegexCache.swift         ← LRU cache of 50 compiled NSRegularExpression
│   ├── Syntax/
│   │   ├── SyntaxEngine.swift       ← tree-sitter wrapper
│   │   ├── LanguageRegistry.swift   ← extension→grammar mapping
│   │   ├── HighlightQuery.swift
│   │   ├── FoldingEngine.swift
│   │   ├── BracketEngine.swift
│   │   └── SymbolIndex.swift
│   ├── IO/
│   │   ├── FileLoader.swift         ← chunked loading, encoding detection
│   │   ├── FileSaver.swift          ← three-tier atomic save
│   │   ├── EncodingDetector.swift
│   │   ├── LineEndingDetector.swift
│   │   └── BinaryDetector.swift
│   ├── Config/
│   │   ├── EditorConfigParser.swift
│   │   └── TabSizeDetector.swift
│   ├── Diff/
│   │   ├── DiffEngine.swift         ← histogram/Myers/patience algorithms
│   │   └── WordDiff.swift
│   └── Common/
│       ├── EditorError.swift        ← unified error enum
│       ├── Progress.swift           ← AsyncStream-based progress
│       └── Constants.swift
├── Services/
│   ├── DocumentManager.swift        ← actor: single document writer
│   ├── FileService.swift            ← open/save/revert
│   ├── AutoSaveService.swift        ← debounce/throttle/focus-lost
│   ├── BackupService.swift          ← continuous 500ms backup
│   ├── SearchService.swift          ← coordinates search across files
│   ├── SessionService.swift         ← GRDB.swift SQLite session DB
│   ├── ThemeManager.swift           ← theme loading, dark/light switching
│   ├── KeyBindingManager.swift      ← chord support, context-aware
│   ├── SnippetManager.swift
│   ├── MacroService.swift           ← v1.1
│   └── GitService.swift             ← git status via libgit2 C library (GPL-v2 with linking exception — compatible with GPL-v3; NOT Process — App Sandbox blocks /usr/bin/git)
├── Views/
│   ├── EditorView.swift             ← NSView + CoreText + Metal
│   ├── TabBarView.swift
│   ├── SidebarView.swift
│   ├── FindBarView.swift
│   ├── SearchResultsPanel.swift
│   ├── CommandPaletteView.swift
│   ├── GotoAnythingView.swift
│   ├── MinimapView.swift
│   ├── CompletionPopupView.swift
│   ├── SplitContainerView.swift
│   ├── StatusBarView.swift
│   └── PreferencesView.swift
├── ViewModels/
│   ├── EditorViewModel.swift
│   ├── TabViewModel.swift
│   └── SidebarViewModel.swift
├── Resources/
│   ├── Assets.xcassets/
│   ├── MainMenu.xib
│   ├── Localizable.strings
│   ├── PrivacyInfo.xcprivacy
│   └── themes/                      ← 10 bundled JSON themes
├── TreeSitter/                      ← C module for tree-sitter
│   ├── module.modulemap
│   ├── tree_sitter/                 ← tree-sitter C headers + source
│   └── grammars/                    ← 50+ precompiled grammar .c files
├── MyNotepadPPTests/
│   └── *.swift                      ← XCTest unit tests
└── MyNotepadPPUITests/
    └── *.swift                      ← XCUITest UI automation
```

---

## 3. TEXT BUFFER ENGINE — PIECE TABLE

### Why Piece Table (not Rope)

| Factor | Piece Table | Rope (B+ tree) |
|--------|-------------|-----------------|
| Implementation complexity | ~500 lines Swift | ~2000 lines Swift |
| Insert/delete | O(log n) with balanced tree of pieces | O(log n) |
| Sequential read | O(n) — excellent cache locality | O(n) — fragmented nodes hurt cache |
| Memory overhead | Very low (pointers into two buffers) | Higher (node metadata per ~512 chars) |
| Copy-on-write snapshot | Copy piece table array (value type) | Copy root pointer (Arc) |
| Used by | VS Code (Monaco), Abiword | Neovim, Xi Editor, Zed |
| Swift ecosystem | Easy to build with Swift value types | No mature Swift library exists |
| Files >1GB | Excellent (streamed via FileHandle in 64KB chunks, progressive loading) | Excellent |
| Proven at scale | VS Code — most popular editor in the world | Proven but harder to implement |

### Data Structures

```swift
/// The original file content — read-only, never modified after load.
/// NO mmap — even read-only mmap causes unrecoverable SIGBUS if the file is
/// truncated externally or a network mount disconnects. SIGBUS cannot be safely
/// handled in Swift/ARC (signal handlers cannot call ARC retain/release).
/// Data is loaded via FileHandle.read in 64KB chunks — only ~33% slower than mmap,
/// fully safe, and works correctly with App Sandbox.
final class OriginalBuffer: Sendable {
    let data: Data
    init(data: Data) { self.data = data }
}

/// All insertions go here — append-only, never modified after append.
/// Grows as the user types. Typically small relative to file size.
struct AddBuffer {
    var data: Data  // starts empty, grows with edits
}

/// A piece points into either the original buffer or the add buffer.
struct Piece {
    let source: PieceSource    // .original or .add
    let start: Int             // byte offset into source buffer
    let length: Int            // byte length of this piece
    let lineBreakCount: Int    // cached newline count for O(log n) line lookup
}

enum PieceSource {
    case original
    case add
}

/// The piece table: an ordered collection of pieces that together
/// represent the full document content.
/// Uses a balanced binary tree (red-black or B-tree) for O(log n) operations.
struct PieceTable {
    let original: OriginalBuffer
    var add: AddBuffer
    var pieces: PieceTree       // balanced tree of Piece nodes
    var totalLength: Int        // cached total byte length
    var totalLines: Int         // cached total line count
    private var generation: UInt64  // incremented on every edit (for stale snapshot detection)
}
```

### Copy-on-Write Snapshots

```swift
/// An immutable snapshot of the document at a point in time.
/// Background threads (syntax parsing, search, auto-save) hold snapshots.
/// The main thread continues editing — snapshots are never invalidated.
///
/// Swift's value semantics + copy-on-write on the piece array means:
/// - Taking a snapshot is O(1) — just copy the struct (reference counted backing)
/// - Editing after snapshot creates a new copy of modified nodes only (COW)
/// - Old snapshot remains valid until dropped
/// - No locks, no contention, no Arc — pure Swift value types
/// NOTE: This is a CLASS (reference type), not a struct.
/// Required for safe Unmanaged pointer passing to tree-sitter C callbacks.
/// Immutable after creation — all fields are `let`. Sendable via immutability.
final class PieceTableSnapshot: @unchecked Sendable {
    let original: OriginalBuffer      // shared (immutable, reference counted)
    let add: Data                     // snapshot of add buffer at this point
    let pieces: PieceTree             // COW copy of piece tree
    let generation: UInt64            // which generation this snapshot represents
    let timestamp: ContinuousClock.Instant  // for stale snapshot detection
}
```

### Line Index (O(log n) Lookup)

The piece tree is an **augmented balanced binary tree** where each node caches:
- `subtreeLength: Int` — total bytes in subtree
- `subtreeLines: Int` — total line breaks in subtree

This enables O(log n) lookup along any dimension:
- **byte → line**: walk tree, summing left subtree lines
- **line → byte**: walk tree, summing left subtree bytes until target line reached
- **pixel → line**: combine with line height cache (fixed-height for monospace)

### Operations

| Operation | Complexity | How |
|-----------|-----------|-----|
| Insert text | O(log n) | Split piece at cursor, insert new piece pointing into add buffer |
| Delete range | O(log n) | Split pieces at range boundaries, remove middle pieces |
| Get line N | O(log n) | Binary search piece tree by cumulative line count |
| Get byte range | O(log n + k) | Find start piece, iterate until range covered (k = output size) |
| Snapshot | O(1) | Copy struct (Swift COW — actual copy deferred until mutation) |
| Line count | O(1) | Cached in root node |
| Byte count | O(1) | Cached in root node |

### Thread Safety Model

```
┌─────────────────────────────────────────────────────┐
│  DocumentManager (Swift actor — SOLE writer)         │
│                                                      │
│  Receives edit commands from UI → applies to          │
│  PieceTable → publishes new snapshot                 │
│                                                      │
│  ONLY the DocumentManager mutates the PieceTable.    │
│  All other threads work with immutable snapshots.     │
│                                                      │
│  func edit(range:, text:) -> PieceTableSnapshot      │
│  func snapshot() -> PieceTableSnapshot               │
│  func undo() -> PieceTableSnapshot?                  │
│  func redo() -> PieceTableSnapshot?                  │
└─────────┬───────────────────────────────────────────┘
          │ publishes snapshots via AsyncStream
          ▼
┌─────────────────────┐  ┌──────────────────────┐
│ SyntaxEngine (Task)  │  │ SearchService (Task)  │
│ Holds: snapshot      │  │ Holds: snapshot       │
│ Reads only           │  │ Reads only            │
│ Cancelled on new     │  │ Cancelled on new      │
│ edit if stale >500ms │  │ query                 │
└─────────────────────┘  └──────────────────────┘

┌─────────────────────┐  ┌──────────────────────┐
│ AutoSaveService      │  │ MinimapRenderer       │
│ Holds: snapshot      │  │ Holds: snapshot       │
│ Writes to disk       │  │ Renders bitmap        │
└─────────────────────┘  └──────────────────────┘
```

**No locks. No Mutex. No DispatchQueue.synchronize.** The actor model + immutable snapshots eliminate all contention.

### Starvation Guard

If a background task (especially syntax parsing) is cancelled 3 consecutive times because its snapshot is >500ms stale, the 4th attempt runs to completion regardless of age. This prevents rapid typing from starving syntax highlighting indefinitely.

### Memory Management

- **Piece table compaction**: triggered by EITHER 10,000 edits OR add buffer exceeding 50MB (whichever comes first). Compaction materializes the piece table into a new `OriginalBuffer` + empty `AddBuffer`. Runs on background thread. The compacted version becomes the new document; old pieces freed via ARC. The 50MB add buffer trigger prevents large paste operations (e.g., 10,000 x 10KB = 100MB) from exceeding the memory budget before the edit count threshold.
- **Snapshot age limit**: background threads holding snapshots older than 2 seconds are cancelled and restarted with a fresh snapshot (prevents unbounded memory from accumulated add buffer snapshots).
- **Add buffer growth**: the add buffer grows monotonically. Compaction resets it. Between compactions, typical growth is <1MB for normal typing. Large paste operations trigger the 50MB size-based compaction.

---

## 4. CONCURRENCY MODEL — SWIFT STRUCTURED CONCURRENCY

### No GCD. No DispatchQueue. No Thread.

The entire concurrency model uses Swift 6 structured concurrency:

```
┌─────────────────────────────────────────────────────────┐
│ MAIN ACTOR (@MainActor — UI thread)                      │
│  AppKit events, rendering, user input, cursor            │
│  Max budget per frame: 16ms (60 FPS)                     │
│  ZERO file I/O, ZERO computation                         │
├─────────────────────────────────────────────────────────┤
│ DocumentManager (actor — serialized document access)      │
│  All text edits, undo/redo, snapshot publishing          │
│  Single writer per document. Multiple views send          │
│  commands; actor serializes them automatically.           │
├─────────────────────────────────────────────────────────┤
│ HIGH-PRIORITY TASKS (Task.detached, priority: .userInit) │
│  Syntax highlighting for visible viewport                │
│  Bracket matching at cursor                              │
│  Autocomplete computation                                │
│  Created via: Task.detached(priority: .userInitiated)    │
├─────────────────────────────────────────────────────────┤
│ NORMAL-PRIORITY TASKS (TaskGroup, priority: .utility)    │
│  Multi-file search (parallel via withTaskGroup)          │
│  Diff computation                                        │
│  File loading (chunked)                                  │
│  Auto-save write                                         │
├─────────────────────────────────────────────────────────┤
│ LOW-PRIORITY TASKS (Task.detached, priority: .background)│
│  Background file indexing                                │
│  Symbol index building                                   │
│  Continuous backup                                       │
│  Session DB periodic flush                               │
├─────────────────────────────────────────────────────────┤
│ SQLite WRITER (dedicated serial executor)                 │
│  All INSERT/UPDATE/DELETE to session DB                   │
│  Uses a custom SerialExecutor to guarantee single-writer  │
│  Never contends with UI or file I/O                      │
└─────────────────────────────────────────────────────────┘
```

### Cancellation — Built Into Swift

```swift
// No manual CancelToken needed. Swift Tasks have native cancellation.

// Starting a cancellable search:
let searchTask = Task.detached(priority: .utility) {
    for file in files {
        try Task.checkCancellation()  // throws if cancelled
        let matches = search(in: file, query: query)
        progressStream.yield(.determinate(Double(i) / Double(total), "Searching..."))
    }
}

// Cancelling (from UI — user pressed Escape or typed new query):
searchTask.cancel()  // immediate, cooperative cancellation
```

Every long-running operation uses `Task.checkCancellation()` between iterations. No custom `AtomicBool` needed.

### Progress Reporting — AsyncStream

```swift
// Core reports progress via AsyncStream (native Swift async sequence):
let (progressStream, continuation) = AsyncStream<Progress>.makeStream()

// Background task yields progress:
continuation.yield(.determinate(0.45, "searching"))

// UI consumes progress on MainActor:
for await progress in progressStream {
    statusBar.update(progress)  // runs on @MainActor
}
```

### Shutdown Guard

```swift
/// Global shutdown flag. Checked by all services before starting new work.
/// Uses Swift's native actor isolation — no raw atomics needed.
/// Shutdown flag uses OSAtomicBool (nonisolated) — NOT @MainActor.
/// Background tasks check this WITHOUT hopping to MainActor,
/// which would flood the main actor's mailbox during shutdown.
final class AppState: Sendable {
    static let shared = AppState()

    /// Lock-free atomic boolean. Read from any thread without actor hop.
    private let _isShuttingDown = OSAllocatedUnfairLock(initialState: false)

    var isShuttingDown: Bool {
        _isShuttingDown.withLock { $0 }
    }

    /// Cancel all background work. This is an async function so the caller
    /// can await completion before starting hot exit.
    func beginShutdown() async {
        _isShuttingDown.withLock { $0 = true }
        // Cancel all in-flight tasks — await each to ensure cancellation propagates
        // before hot exit begins. Each cancelAll() is fast (<10ms — just sets flags).
        await DocumentManager.shared.cancelAllBackgroundWork()
        await AutoSaveService.shared.cancelAll()
        await BackupService.shared.cancelAll()
        await SearchService.shared.cancelAll()
    }
}
```

**Key design decisions:**
- `isShuttingDown` is `nonisolated` — background tasks read it via `AppState.shared.isShuttingDown` with NO actor hop (no `await`). This prevents flooding the main actor mailbox during shutdown.
- Uses `OSAllocatedUnfairLock` (macOS 14+) for lock-free atomic read/write — the most efficient synchronization primitive on Apple Silicon.

### Thermal Throttling

```swift
// Reduce concurrency under thermal pressure
let thermalState = ProcessInfo.processInfo.thermalState
let maxConcurrency: Int = switch thermalState {
    case .nominal, .fair: ProcessInfo.processInfo.activeProcessorCount
    case .serious: 2
    case .critical: 1
    @unknown default: 2
}

// Used in multi-file search:
await withTaskGroup(of: [Match].self) { group in
    for file in files.prefix(maxConcurrency) {
        group.addTask { search(in: file, query: query) }
    }
    // ... throttled parallelism
}
```

---

## 5. TEXT RENDERING — CoreText + Metal

**Unchanged from the original architecture.** This was always pure Swift/ObjC.

### Pipeline

```
User types → DocumentManager.edit() → new PieceTableSnapshot
    → EditorView.setNeedsDisplay(dirtyRect)
    → draw(dirtyRect):
        1. Query snapshot for visible lines (O(log n) via line index)
        2. Shape text with CoreText (CTLine per visible line)
        3. Look up glyphs in Metal texture atlas (cache hit ~99%)
        4. Draw via Metal command buffer (batched draw calls)
        5. Draw cursor, selections, highlights as Metal overlays
```

### Glyph Atlas

| Property | Value |
|----------|-------|
| Backing | `MTLTexture` (GPU-resident) |
| Cache key | `(glyphID, fontID, fontSize, subpixelOffset)` |
| Color | Applied via Metal shader, NOT baked into atlas |
| Emoji | Separate RGBA atlas for Apple Color Emoji (`sbix` format). Use `CTFontDrawGlyphs()`. |
| Eviction | LRU page eviction when atlas exceeds 32MB |
| Tab-close cleanup | Deferred compaction (5s debounce) — evict pages with no live references |
| Scale change | Invalidate entire atlas when `contentsScale` changes (Retina ↔ non-Retina) |

### Shader Pre-Compilation

All Metal Pipeline State Objects (PSOs) compiled during app startup from a pre-compiled `.metallib` bundled in the app. Never compile shaders at runtime — prevents 50-100ms first-frame hitch.

### Dirty Rect Rendering

- On edit: only re-render changed lines + cursor line
- On scroll: only render newly visible lines
- Never redraw entire viewport for a single-line edit

### Overdraw Buffer

Pre-render 2x viewport height (1 screen above + 1 below). Fast scrolling never shows blank.

### Viewport Culling

Only compute layout for visible lines + overdraw buffer. A 1GB file with 20M lines does NOT compute layout for all lines. The piece table's augmented tree provides O(log n) "which line is at pixel Y" lookup.

### Long Line Handling

- Lines >10,000 chars: column-level viewport virtualization — only compute layout for visible columns +/- overdraw
- Lines >500,000 chars: disable syntax highlighting for that line

### Frame Budget Watchdog (Debug Builds)

Budget: 16ms per frame. Breakdown: input ~1ms, piece table edit ~0.1ms, layout ~2ms, glyph lookup ~1ms, Metal draw ~3ms, swap ~1ms, syntax query ~8ms remaining.

If any frame exceeds 14ms, log which phase was slow. Drop syntax colors for the current frame rather than dropping below 60 FPS.

### NSTextInputClient (CRITICAL)

EditorView implements `NSTextInputClient` for:
- IME composition (CJK input methods)
- Emoji picker (`Cmd+Ctrl+Space`)
- Dictation
- System text replacement
- Character palette

Methods: `setMarkedText`, `markedRange`, `selectedRange`, `attributedSubstring`, `insertText`, `firstRect(forCharacterRange:)`, `characterIndex(for:)`.

Call `inputContext?.invalidateCharacterCoordinates()` on scroll or layout change.

### NSWritingToolsCoordinator (macOS 15+)

Implement delegate for Writing Tools integration (AI proofreading). Provide text context as `NSAttributedString`, handle replacement callbacks.

---

## 6. AUTO-SAVE — NEVER HANGS, NEVER BLOCKS, NEVER PROMPTS

### Three Independent Save Systems

```
A. Auto-Save (to original file)
   User types → debounce 1s → Task: snapshot → serialize → atomic write → done

B. Continuous Backup (to backup directory)
   Every 500ms → snapshot → write to ~/Library/Application Support/mynotepadpp/backups/{doc_id}/
   Survives SIGKILL. Max data loss: 500ms of typing.

C. Hot Exit (to SQLite session DB)
   On Cmd+Q → single SQLite transaction → all tab state + compressed unsaved content
```

### Debounce / Throttle / Focus Lost

```swift
actor AutoSaveService {
    /// Per-document debounce tasks. Key = document ID.
    private var debounceTasks: [String: Task<Void, Never>] = [:]
    /// Per-document throttle tasks. Key = document ID.
    private var throttleTasks: [String: Task<Void, Never>] = [:]
    /// Per-file save lock: prevents concurrent saves to the same path.
    private var activeSaves: Set<String> = []

    /// Called on every keystroke — resets 1s debounce FOR THIS DOCUMENT ONLY.
    /// Other documents' debounce timers are unaffected.
    func documentDidChange(_ doc: Document) {
        let docID = doc.id
        debounceTasks[docID]?.cancel()
        debounceTasks[docID] = Task { [weak self, weak doc] in
            do {
                try await Task.sleep(for: .seconds(1))
                try Task.checkCancellation()
                guard let self, let doc else { return }
                await self.serializedSave(doc)
            } catch {
                return
            }
        }
    }

    /// 30s throttle FOR THIS DOCUMENT — fires regardless of typing activity.
    func startThrottle(_ doc: Document) {
        let docID = doc.id
        throttleTasks[docID]?.cancel()
        throttleTasks[docID] = Task { [weak self, weak doc] in
            while !Task.isCancelled {
                try? await Task.sleep(for: .seconds(30))
                guard !Task.isCancelled, let self, let doc else { return }
                await self.serializedSave(doc)
            }
        }
    }

    /// MUST be called when a tab is closed — cancels THIS document's tasks only.
    func stopTracking(_ doc: Document) {
        let docID = doc.id
        debounceTasks[docID]?.cancel()
        debounceTasks.removeValue(forKey: docID)
        throttleTasks[docID]?.cancel()
        throttleTasks.removeValue(forKey: docID)
    }

    /// Tracked focus-loss save tasks (so cancelAll can reach them).
    /// Pruned of completed tasks on each new focus-loss event to prevent unbounded growth.
    private var focusLossTasks: [Task<Void, Never>] = []

    /// Cancel all in-flight saves for ALL documents (called from AppState.beginShutdown)
    func cancelAll() {
        for (_, task) in debounceTasks { task.cancel() }
        debounceTasks.removeAll()
        for (_, task) in throttleTasks { task.cancel() }
        throttleTasks.removeAll()
        for task in focusLossTasks { task.cancel() }
        focusLossTasks.removeAll()
    }

    /// Immediate save on window focus lost — Tasks are tracked for cancellation.
    func windowDidResignKey(dirtyDocuments: [Document]) {
        guard !AppState.shared.isShuttingDown else { return }
        // Cancel any still-in-flight tasks from a previous focus-loss, THEN clear.
        for task in focusLossTasks { task.cancel() }
        focusLossTasks.removeAll()
        for doc in dirtyDocuments {
            let task = Task { [weak self, weak doc] in
                guard let self, let doc else { return }
                await self.serializedSave(doc)
            }
            focusLossTasks.append(task)
        }
    }

    /// Serialized save: only ONE save per file path at a time.
    /// If a save is already in-flight for this path, SKIP (don't wait/queue).
    /// The debounce/throttle timer will retry on the next cycle.
    /// This eliminates all waiter complexity, continuation leaks, and deadlock risks.
    func serializedSave(_ doc: Document) async {
        guard let path = doc.filePath else { return }
        guard !AppState.shared.isShuttingDown else { return }
        guard !Task.isCancelled else { return }
        guard !activeSaves.contains(path) else { return }

        activeSaves.insert(path)
        defer { activeSaves.remove(path) }  // ALWAYS removes — even on CancellationError
        do {
            try await save(doc)
        } catch {
            // Catches ALL errors including CancellationError.
            // `defer` ensures activeSaves is cleaned up regardless.
            os_log(.error, "Save failed for %{public}@: %{public}@", path, error.localizedDescription)
        }
    }
}
```

**Key design decisions:**
- `serializedSave` uses **skip-if-busy** instead of wait/queue — the 1s debounce and 30s throttle guarantee a retry within seconds. No waiter accumulation, no continuation leak, no deadlock.
- `save()` is now `throws` — errors don't leave `activeSaves` in corrupt state (the `do/catch` ensures `activeSaves.remove` always runs).
- `stopTracking` cancels the throttle task on tab close (prevents orphaned Task loop)
- `cancelAll` for shutdown

All three (debounce, throttle, focus-lost) are OR'd — whichever fires first wins. After a save, all timers reset.

### Three-Tier Atomic Write

```
Tier 1: Atomic rename (preferred)
  1. Write to {dir}/{filename}.mynotepadpp-{pid}-{timestamp}.tmp
  2. fsync() (F_FULLFSYNC on macOS for true NVMe cache flush)
  3. rename() temp → target (atomic on APFS/HFS+)
  → If rename fails (cross-device mount): fall to Tier 2

Tier 2: Direct overwrite with backup
  1. Copy target to {dir}/{filename}.mynotepadpp-bak
  → If copy fails (ENOSPC): fall to Tier 3 WITHOUT touching original
  2. Write new content to a SECOND temp file {dir}/{filename}.mynotepadpp-new
  → If write fails (ENOSPC mid-write): delete the temp, original + backup still intact → fall to Tier 3
  3. fsync() the new temp file
  4. rename() {filename}.mynotepadpp-new → target (atomic replace)
  5. Delete backup (.mynotepadpp-bak)
  → NOTE: we NEVER truncate the original directly. Step 2-4 use a temp + rename,
    so a failure at any step leaves the original intact. The backup is a safety net
    only for the rename step (which can't partially fail on APFS).

Tier 3: Recovery directory write
  1. Write to ~/Library/Application Support/mynotepadpp/recovery/
  2. Show non-blocking banner: "Could not save to original location."
  3. Keep buffer marked dirty
```

### Cloud-Aware Save

- Detect iCloud Drive (`~/Library/Mobile Documents/`), Dropbox, Google Drive, OneDrive
- Cloud dirs: use Tier 2 (direct overwrite) — cloud sync handles in-place writes better
- macOS iCloud: use `NSFileCoordinator` for all writes
- Increase debounce to 3s for cloud dirs (reduce sync churn)

### Disk Full Handling

If all tiers fail → keep buffer in memory → set `DIRTY_CRITICAL` state → show persistent banner → poll disk space every 10s → auto-retry when space appears.

On `Cmd+Q` with `DIRTY_CRITICAL`: show ONE blocking dialog: "Cannot save N files (disk full). Quit anyway?" — sole exception to "never prompt."

### File Watcher Self-Suppression

Unified strategy (priority chain):
1. `kFSEventStreamEventFlagOwnEvent` (macOS 13.0+) — most reliable
2. Mtime comparison fallback: record `(path, mtime)` on save, compare on event, expire after 2s
3. Never use a boolean flag or fixed-time suppression window

### Continuous Backup Cleanup

- On successful save to original: delete backup
- On named tab close (auto-saved): delete backup directory immediately
- On untitled tab close: hot exit saves to SQLite, then delete backup directory
- On startup: scan backups/ — restore if newer than original, delete if stale (>7 days)
- Orphan cleanup: startup + every 1 hour, delete temp files older than 1 hour with dead PIDs

---

## 7. SESSION PERSISTENCE — SQLite

### Library: GRDB.swift (Swift SQLite wrapper)

Or raw `sqlite3` C API (built into macOS). GRDB provides:
- Type-safe queries with `Codable` records
- Prepared statement caching (`prepareStatement` reuse)
- WAL mode, migrations, connection pooling
- `DatabaseQueue` (serialized writer) + `DatabasePool` (concurrent readers)

### Connection Architecture

```
WRITER (DatabaseQueue — serialized, dedicated)
  All INSERT/UPDATE/DELETE
  Uses BEGIN IMMEDIATE

READER (DatabasePool — concurrent, UI + background)
  All SELECT queries
  Never blocked by writer (WAL guarantee)
```

### PRAGMAs (set on every connection open)

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = FULL;
PRAGMA fullfsync = ON;
PRAGMA cache_size = -2000;                -- 2MB
PRAGMA mmap_size = 33554432;              -- 32MB (SQLite internal mmap — safe, SQLite handles SIGBUS)
PRAGMA temp_store = MEMORY;
PRAGMA busy_timeout = 5000;
PRAGMA wal_autocheckpoint = 1000;
PRAGMA foreign_keys = ON;
PRAGMA trusted_schema = OFF;
PRAGMA cell_size_check = ON;
PRAGMA threads = 2;
PRAGMA journal_size_limit = 67108864;     -- 64MB WAL cap
PRAGMA optimize = 0x10002;

-- Writer only:
PRAGMA cache_spill = OFF;
```

### Schema (STRICT tables, zstd compression for blobs)

```sql
CREATE TABLE windows (
    id TEXT PRIMARY KEY,
    x REAL NOT NULL, y REAL NOT NULL,
    width REAL NOT NULL, height REAL NOT NULL,
    is_fullscreen INTEGER NOT NULL DEFAULT 0
) STRICT;

CREATE TABLE tabs (
    id TEXT PRIMARY KEY,
    window_id TEXT NOT NULL REFERENCES windows(id),
    file_path TEXT,                      -- NULL for untitled
    cursor_line INTEGER NOT NULL DEFAULT 0,
    cursor_col INTEGER NOT NULL DEFAULT 0,
    scroll_top REAL NOT NULL DEFAULT 0,
    syntax TEXT,
    is_pinned INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL,
    is_dirty INTEGER NOT NULL DEFAULT 0
) STRICT;

CREATE TABLE unsaved_content (
    tab_id TEXT PRIMARY KEY REFERENCES tabs(id),
    content BLOB NOT NULL,              -- zstd compressed
    encoding TEXT NOT NULL DEFAULT 'utf-8'
) STRICT;

CREATE TABLE undo_history (
    tab_id TEXT PRIMARY KEY REFERENCES tabs(id),
    history BLOB NOT NULL               -- zstd compressed, last 1000 ops
) STRICT;

CREATE TABLE app_state (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
) STRICT;
-- Keys: 'dirty_flag', 'last_active_tab', 'last_active_window'
```

### Write Batching

Cursor/scroll positions batched in-memory (`Dictionary<TabID, TabState>`), flushed to SQLite every 5 seconds in a single transaction. Tab open/close written immediately.

### Crash Recovery

**Strict ordering — all in a single `BEGIN IMMEDIATE` transaction:**
```sql
BEGIN IMMEDIATE;
-- Step 1: READ the old dirty_flag value FIRST
SELECT value FROM app_state WHERE key = 'dirty_flag';  -- returns '0' (clean) or '1' (crashed)
-- Step 2: IMMEDIATELY overwrite to '1' (marks this session as dirty)
UPDATE app_state SET value = '1' WHERE key = 'dirty_flag';
COMMIT;
-- fsync via PRAGMA fullfsync = ON (already set)
```
3. If old value was `1` → previous session crashed → run recovery:
4. `PRAGMA quick_check` (NOT `integrity_check` — 10-100x faster)
5. If corrupt: try `sqlite3_recover`, else restore from backup, else recreate
6. On clean exit: set `dirty_flag = 0` (in shutdown sequence)

**Why single transaction:** prevents a crash between read and write from losing the flag state. `BEGIN IMMEDIATE` ensures no concurrent writer interferes.

### Clean Exit

```sql
PRAGMA optimize;
PRAGMA incremental_vacuum;
PRAGMA wal_checkpoint(TRUNCATE);
```

### Periodic Backup

Every 10 minutes: `VACUUM INTO 'sessions.db.backup'`.

---

## 8. FILE I/O — LOADING & ENCODING

### File Loading Pipeline

```swift
actor FileLoader {
    func load(url: URL) async throws -> LoadResult {
        // 1. Read first 8KB for detection
        let header = try await readHeader(url: url, size: 8192)

        // 2. Binary detection (null bytes >0.1% or magic bytes)
        if isBinary(header) {
            throw EditorError.binaryFile(url.lastPathComponent)
        }

        // 3. Encoding detection (BOM → charset → heuristic → locale)
        let encoding = detectEncoding(header)

        // 4. Line ending detection (LF/CRLF/CR)
        let lineEnding = detectLineEnding(header)

        // 5. Chunked loading (64KB chunks → build piece table)
        let pieceTable = try await loadChunked(url: url, encoding: encoding)

        return LoadResult(
            pieceTable: pieceTable,
            encoding: encoding,
            lineEnding: lineEnding
        )
    }

    private func loadChunked(url: URL, encoding: String.Encoding) async throws -> PieceTable {
        let handle = try FileHandle(forReadingFrom: url)
        defer { try? handle.close() }

        // PROGRESSIVE LOADING: build piece table chunk-by-chunk.
        // NEVER accumulate the entire file into one Data allocation (OOM on 1GB files).
        // Each 64KB chunk is converted to UTF-8 and appended to the piece table's add buffer.
        // The piece table materializes as we go — first screenful available within 200ms.
        var pieceTable = PieceTable.empty()
        while true {
            try Task.checkCancellation()
            guard let chunk = try handle.read(upToCount: 65536), !chunk.isEmpty else { break }
            let utf8Chunk = try convert(chunk, from: encoding, to: .utf8)
            pieceTable.appendToOriginal(utf8Chunk)  // O(1) — extends OriginalBuffer, adds piece
        }
        // After loading, the OriginalBuffer contains the full file as a contiguous NSData
        // (built incrementally via append). No single 1GB allocation — the OS pages it in.
        return pieceTable
    }
}
```

### Encoding Detection Priority

1. BOM (Byte Order Mark) — definitive
2. XML/HTML charset declaration — `<?xml encoding="..."?>`, `<meta charset="...">`
3. `CFStringEncoding` heuristic (`CFStringCreateWithBytes` with `isExternalRepresentation: true`)
4. User locale fallback

### Supported Encodings

Via `CFStringEncoding` (all available on macOS):
UTF-8, UTF-8 BOM, UTF-16 LE, UTF-16 BE, ASCII, ISO-8859-1 through -15, Windows-1250 through -1258, Shift-JIS, EUC-JP, EUC-KR, GB2312, Big5, KOI8-R, MacRoman

### No mmap for User Files

`mmap` causes unrecoverable SIGBUS on truncated files or network filesystem disconnects. SIGBUS cannot be safely handled in Swift/ARC — signal handlers cannot call ARC retain/release or Swift runtime functions. **No mmap for any user file, regardless of size or filesystem.** `FileHandle.read` with 64KB chunks is only ~33% slower and fully safe.

---

## 9. SEARCH ENGINE

### Single-Document Search

```swift
struct BufferSearch {
    /// Literal search using Data.range(of:) — Apple's SIMD-optimized byte scan
    func searchLiteral(in snapshot: PieceTableSnapshot, query: String,
                       caseSensitive: Bool, wholeWord: Bool) -> [SearchMatch] {
        let data = snapshot.materializedUTF8  // or iterate pieces
        // Data.range(of:) uses NEON SIMD on arm64
        // ...
    }

    /// Regex search using NSRegularExpression (ICU engine)
    func searchRegex(in snapshot: PieceTableSnapshot, pattern: String) throws -> [SearchMatch] {
        let regex = try cachedRegex(pattern)  // LRU cache of 50
        // ...
    }
}
```

### Multi-File Search (Parallel)

```swift
actor SearchService {
    func searchFiles(in directory: URL, query: SearchQuery,
                     progress: AsyncStream<Progress>.Continuation) async throws -> [FileMatch] {
        let files = collectFiles(in: directory, respecting: ".gitignore")

        return try await withThrowingTaskGroup(of: FileMatch?.self) { group in
            var results: [FileMatch] = []
            var inFlight = 0
            let concurrency = min(ProcessInfo.processInfo.activeProcessorCount, 8)

            for file in files {
                try Task.checkCancellation()
                group.addTask(priority: .utility) {
                    // No [weak self] needed — actor is alive for entire withThrowingTaskGroup scope
                    //
                    // STREAMING SEARCH: read file in 64KB chunks, search each chunk with
                    // overlap window (query.length bytes carried over between chunks to catch
                    // matches spanning chunk boundaries). Peak memory per task: ~128KB, not 10MB.
                    let handle = try FileHandle(forReadingFrom: file)
                    defer { try? handle.close() }
                    var matches: [SearchMatch] = []
                    var lineOffset = 0
                    var carryOver = Data()  // overlap from previous chunk

                    while let chunk = try handle.read(upToCount: 65536), !chunk.isEmpty {
                        try Task.checkCancellation()
                        var searchBuffer = carryOver + chunk
                        let chunkMatches = self.search(in: searchBuffer, query: query,
                                                        lineOffset: lineOffset)
                        matches.append(contentsOf: chunkMatches)
                        // Carry over last N bytes for boundary-spanning matches
                        let overlapSize = min(query.pattern.utf8.count * 2, searchBuffer.count)
                        carryOver = searchBuffer.suffix(overlapSize)
                        lineOffset += chunk.withUnsafeBytes { buf in
                            buf.reduce(0) { $0 + ($1 == UInt8(ascii: "\n") ? 1 : 0) }
                        }
                    }
                    return matches.isEmpty ? nil : FileMatch(url: file, matches: matches)
                }
                inFlight += 1

                // Throttle: when at capacity, wait for one task to finish before adding more
                if inFlight >= concurrency {
                    if let result = try await group.next() {
                        inFlight -= 1
                        if let match = result {
                            results.append(match)
                            progress.yield(.determinate(Double(results.count), "searching"))
                        }
                    }
                }
            }  // end for file in files

            // Drain all remaining in-flight tasks
            for try await result in group {
                if let match = result { results.append(match) }
            }
            return results
        }
    }
}
```

### Regex Cache

LRU cache of 50 compiled `NSRegularExpression` objects. Key: `(pattern, options)`. `NSRegularExpression` compilation is expensive (~1ms); cached reuse is ~0.001ms.

### Incremental In-Buffer Search

As user types in find bar, refine previous results rather than re-scanning entire document. If new character narrows results, filter existing matches. If character deleted, re-search.

---

## 10. SYNTAX ENGINE — TREE-SITTER VIA C INTEROP

### No Rust Wrapper. Swift calls tree-sitter C API directly.

```swift
// module.modulemap exposes tree-sitter C headers to Swift
import TreeSitter

/// Wraps a tree-sitter tree pointer with automatic memory management.
/// ts_tree_delete is called on deinit — no manual free needed by callers.
final class SyntaxTree {
    let rawTree: OpaquePointer  // ts_tree*
    let generation: UInt64       // piece table generation this was parsed from

    init(rawTree: OpaquePointer, generation: UInt64) {
        self.rawTree = rawTree
        self.generation = generation
    }

    deinit {
        ts_tree_delete(rawTree)  // CRITICAL: prevents tree memory leak
    }
}

/// SyntaxEngine wraps a tree-sitter parser. tree-sitter's C parser is NOT thread-safe —
/// concurrent calls to ts_parser_parse on the same ts_parser* cause UB.
/// This class is therefore NOT Sendable. Each background Task that needs to parse
/// MUST use its own SyntaxEngine instance (one per document, owned by DocumentManager).
/// The DocumentManager actor serializes access — only one parse Task per document at a time.
final class SyntaxEngine {
    private var parser: OpaquePointer  // ts_parser — NOT thread-safe

    init() {
        parser = ts_parser_new()
    }

    deinit {
        ts_parser_delete(parser)
    }

    func setLanguage(_ language: OpaquePointer) {
        ts_parser_set_language(parser, language)
    }

    /// Parse a snapshot into a syntax tree.
    /// SAFETY:
    /// 1. snapshot is heap-allocated (class-backed reference) and retained
    ///    for the entire duration of ts_parser_parse via Unmanaged.passRetained.
    /// 2. ts_parser_parse is SYNCHRONOUS — it calls the read callback inline
    ///    during its execution and returns only when parsing is complete.
    ///    The TSInput struct and its callback are NOT stored by tree-sitter
    ///    after ts_parser_parse returns. This guarantees the retained reference
    ///    is alive for all callback invocations.
    /// 3. The retained reference is released immediately after parsing completes.
    func parse(source: PieceTableSnapshot, oldTree: SyntaxTree?) -> SyntaxTree? {
        // Retain snapshot for the duration of the C parse call.
        // This prevents ARC from deallocating it while the C callback is in-flight.
        // Use concrete type Unmanaged<PieceTableSnapshot> — NO AnyObject erasure.
        // PieceTableSnapshot is a final class, so Unmanaged works directly.
        let retainedSource = Unmanaged<PieceTableSnapshot>.passRetained(source)
        defer { retainedSource.release() }  // balanced release after parse completes

        var input = TSInput(
            payload: retainedSource.toOpaque(),
            read: { payload, byteOffset, position, bytesRead in
                // Direct type recovery — no fragile `as?` cast needed.
                let snapshot = Unmanaged<PieceTableSnapshot>.fromOpaque(payload!)
                    .takeUnretainedValue()
                // POINTER LIFETIME: PieceTableSnapshot stores its data as an `NSData`
                // property (not Swift `Data`). NSData.bytes returns a pointer that is stable
                // for the lifetime of the NSData object. The snapshot is retained by
                // Unmanaged.passRetained for the entire parse → NSData stays alive →
                // pointer is valid across all callback invocations within this parse.
                //
                // Implementation:
                //   class PieceTableSnapshot {
                //       let originalNSData: NSData  // stored as NSData, not Data
                //       func readUTF8(at offset: Int, length: UnsafeMutablePointer<UInt32>)
                //           -> UnsafePointer<CChar>? {
                //           let available = min(Int(length.pointee), originalNSData.length - offset)
                //           guard available > 0 else { length.pointee = 0; return nil }
                //           length.pointee = UInt32(available)
                //           return originalNSData.bytes.advanced(by: offset)
                //               .assumingMemoryBound(to: CChar.self)
                //       }
                //   }
                //
                // NEVER use Swift `Data` for this — `Data` is a value type and
                // `.withUnsafeBytes` pointer is only valid inside the closure.
                return snapshot.readUTF8(at: Int(byteOffset), length: bytesRead)
            },
            encoding: TSInputEncodingUTF8
        )
        guard let rawTree = ts_parser_parse(parser, oldTree?.rawTree, input) else {
            // ts_parser_parse returns NULL on OOM, missing language, or reentrant call.
            // Return nil — caller renders without syntax highlighting (graceful degradation).
            os_log(.error, "ts_parser_parse returned NULL — language may not be set or OOM")
            return nil
        }
        return SyntaxTree(rawTree: rawTree, generation: source.generation)
    }
}
```

**IMPORTANT**: `PieceTableSnapshot` must be a `class` (reference type), NOT a `struct`, to work safely with `Unmanaged`. If it were a struct, `passRetained` would box it implicitly, but the `takeUnretainedValue` cast back would fail. The snapshot is designed as a reference-counted class wrapping immutable data.

### Incremental Parsing

- After edit: call `ts_tree_edit()` on the old `SyntaxTree.rawTree` with change range, then `parse(source:oldTree:)`
- Tree-sitter incrementally updates only affected nodes
- Old `SyntaxTree` is automatically freed when the last reference drops (ARC via `deinit`)
- 30ms debounce after edit before re-parse
- Parse on high-priority detached Task
- Revision tracking: each `SyntaxTree` carries the piece table generation number — discard stale results

### 50+ Language Grammars

Precompiled as `.c` files bundled in the app. Each grammar is a C function (`tree_sitter_rust()`, `tree_sitter_python()`, etc.) called via Swift C interop. Lazy-loaded on first use.

### Highlight Queries

Each language has a `.scm` highlight query file (standard tree-sitter format). Loaded from app bundle resources. Maps tree-sitter node types to semantic tokens (`keyword`, `string`, `comment`, `function`, etc.).

### Embedded Languages

HTML: detect `<script>` → JS grammar, `<style>` → CSS grammar via tree-sitter language injection.
Markdown: detect fenced code blocks → appropriate grammar.

---

## 11. ERROR HANDLING

### Unified Error Type

```swift
enum EditorError: LocalizedError {
    case io(underlying: Error)
    case encoding(description: String)
    case binaryFile(String)
    case syntaxParsing(String)
    case diff(String)
    case macro(String)
    case plugin(String)
    case invalidArgument(String)
    case sessionDB(underlying: Error)
    case searchRegex(underlying: Error)

    /// User-facing descriptions use NSLocalizedString — never hardcoded English.
    /// All strings are in Localizable.strings for i18n compliance.
    var errorDescription: String? {
        switch self {
        case .io(let error):
            return String(format: NSLocalizedString("error.io", comment: ""), error.localizedDescription)
        case .encoding(let desc):
            return String(format: NSLocalizedString("error.encoding", comment: ""), desc)
        case .binaryFile(let name):
            return String(format: NSLocalizedString("error.binary_file", comment: ""), name)
        case .syntaxParsing(let msg):
            return String(format: NSLocalizedString("error.syntax", comment: ""), msg)
        // ... all cases use NSLocalizedString for i18n
        default:
            return NSLocalizedString("error.unknown", comment: "")
        }
    }
}
```

### Error Presentation

| Severity | Presentation |
|----------|-------------|
| Fatal (can't open DB) | `NSAlert` with details + "Report Issue" button |
| Blocking (can't save) | Inline banner above editor with retry button |
| Informational (binary file) | Sheet dialog with options |
| Transient (network timeout) | Status bar message, auto-dismiss 5s |

### Rules

- No `try!` or force unwraps in production code — always `try` with `do/catch` or `try?`
- No `fatalError()` except in truly unreachable code paths (mark with `// UNREACHABLE:` comment)
- Errors cross service boundaries as `EditorError` — never raw `NSError` or `String`
- User-facing text is always actionable ("Could not save: disk full") not technical ("ENOSPC")

---

## 12. ALL 65 FEATURES — UNCHANGED

Every feature from the original specification is preserved identically. The architecture change (Rust→Swift) affects only the engine internals, not user-facing behavior.

Refer to `docs/FUNCTIONAL_SPECIFICATION.md` sections 3.1 and 4.1-4.55 for complete feature specifications including:
- All 65 v1.0 features (multi-tab, split view, multi-cursor, find/replace, auto-save, syntax highlighting 50+ languages, code folding, autocomplete, Go to Definition, Command Palette, Goto Anything, minimap, sidebar, distraction-free mode, git gutter, snippets, macros, etc.)
- All keyboard shortcuts (section 4.4)
- All menu bar items (section 4.4.8)
- All file handling edge cases (section 4.49)
- All save behavior details (section 4.48)

**No user-facing feature changes. No shortcut changes. No behavior changes.**

---

## 13. PERFORMANCE TARGETS

### Core Metrics (same as original — no regression allowed)

| Metric | Target | How (Swift) |
|--------|--------|-------------|
| Cold startup | <500ms | Lazy grammar loading, async session read |
| Open 1MB file | <200ms | FileHandle 64KB chunks → piece table |
| Open 100MB file | <2s (first screen <200ms) | Progressive: first screen from header, rest async |
| Open 1GB file | <10s (first screen <200ms) | Streaming piece table construction |
| Keystroke latency | <16ms | Actor edit → snapshot → dirty rect render |
| Scroll FPS | 60 FPS on M4 | Metal glyph atlas + overdraw buffer |
| Search 10K files (literal) | <2s | TaskGroup parallel + Data.range(of:) SIMD |
| Search 10K files (regex) | <5s | NSRegularExpression + TaskGroup |
| Autocomplete popup | <50ms | Buffer word scan on detached Task |
| Memory idle (1 file) | <50MB | No Rust stdlib overhead |
| Memory (50 tabs) | <300MB | Runtime RSS monitoring + eviction |
| Auto-save (<100KB) | <50ms background | Piece table snapshot → atomic write |
| Hot exit (50 tabs) | <500ms | Single SQLite transaction + zstd |
| Tab switch | <30ms | Session state read + snapshot swap |
| Syntax highlight after edit | <50ms viewport | tree-sitter incremental parse |
| File watcher response | <500ms | FSEvents with 200ms debounce |

### Zero-Hang Guarantees

| Scenario | Guarantee |
|----------|-----------|
| `Cmd+Q` | Terminate within 3s. No dialog. No hang. |
| `Cmd+W` | Close within 50ms. No dialog (auto-save on). |
| System shutdown | Save + terminate within 5s. |
| SIGKILL | Max 500ms data loss (continuous backup). |
| Network FS stall | File ops timeout at 30s. UI never freezes. |
| Opening 1GB file | UI responsive immediately. Loading async. |
| Find in 100K files | Cancellable. UI responsive. |
| Disk full | Non-blocking banner. Buffer in memory. |

### Runtime Memory Budget Enforcement

- Check RSS every 10s via `mach_task_info`
- At 250MB: evict undo history for non-active tabs, drop syntax trees for hidden tabs
- At 300MB: aggressive cache purge (shaped text, glyph atlas, line layout)
- At 350MB: status bar warning, refuse new file opens until RSS drops

### App Nap Prevention

```swift
ProcessInfo.processInfo.disableSuddenTermination()  // while documents dirty
ProcessInfo.processInfo.beginActivity(
    options: [.userInitiated],
    reason: "unsaved documents"
)  // prevents App Nap throttling auto-save timers
```

---

## 14. SECURITY

### App Sandbox + Hardened Runtime

```xml
<!-- MyNotepadPP.entitlements -->
<dict>
    <key>com.apple.security.app-sandbox</key><true/>
    <key>com.apple.security.files.user-selected.read-write</key><true/>
    <key>com.apple.security.files.bookmarks.app-scope</key><true/>
    <!-- v1.1: com.apple.security.network.client (SFTP) -->
    <!-- v1.1: com.apple.security.print (Print support) -->
</dict>
```

### Sandbox Container Paths

All app-internal data (backups, recovery, sessions, plugins, themes, macros, snippets) MUST use `FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask)` to get the sandbox-correct path. **Never hardcode** `~/Library/Application Support/mynotepadpp/` — the sandbox may relocate the container. The path references in this document (e.g., `~/Library/Application Support/mynotepadpp/backups/`) are logical descriptions; implementation MUST use `FileManager` API.

### Path Sanitization

Canonicalize ALL file paths before use. Reject `..` traversal. Verify resolved path within expected directory. Applies to: theme loading, recovery files, URL scheme handler, temp files.

### URL Scheme Handler

`mynotepadpp://open?file=...` — confirmation dialog required. Reject system dirs. Rate-limit 1/second.

### BiDi Attack Protection

Warn on bidirectional override characters (U+202A-U+202E, U+2066-U+2069) in source files.

### Clipboard Access (macOS Sequoia)

Only read clipboard on explicit user action (Cmd+V, Cmd+E). Never poll proactively.

### Symlink Loop Detection

Track visited inodes during directory enumeration. Stop recursion on duplicate inode.

### Security-Scoped Bookmark Limit

Track active count. Warn at 200, persistent banner at 240, fall back to NSOpenPanel at 250.

### .git Directory Protection

Show warning when opening files inside `.git/`. Disable auto-save for .git files.

### Recovery File Permissions

Set file permissions to `0600` on all recovery, backup, and SQLite files.

### PrivacyInfo.xcprivacy

```xml
<dict>
    <key>NSPrivacyTracking</key><false/>
    <key>NSPrivacyCollectedDataTypes</key><array/>
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array><string>C617.1</string></array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryDiskSpace</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array><string>E174.1</string></array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategorySystemBootTime</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array><string>35F9.1</string></array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array><string>CA92.1</string></array>
        </dict>
    </array>
</dict>
```
NOTE: If `mach_task_info()` for RSS monitoring (Section 13) triggers Apple's binary scanner, add `NSPrivacyAccessedAPICategorySystemUsageStat` entry. As of 2026, Apple has not formally categorized `task_info()` — monitor WWDC announcements and submission feedback.

---

## 15. SHUTDOWN & CLOSE — NEVER PROMPTS, NEVER LOSES DATA

### Close Tab (`Cmd+W`)

Auto-save on → save silently → close instantly (<50ms). No prompt.

### Close Last Window

Save session to SQLite immediately. App stays in Dock. Click Dock → restore from SQLite via `applicationShouldHandleReopen`.

### Quit (`Cmd+Q`)

```swift
@MainActor func applicationShouldTerminate(_ sender: NSApplication) -> NSApplication.TerminateReply {
    // Shared flag: set to true when hot exit completes successfully.
    let hotExitCompleted = OSAllocatedUnfairLock(initialState: false)

    // Hot exit runs DETACHED (not @MainActor) to avoid deadlock:
    // beginShutdown() awaits service actors which may have tasks awaiting MainActor.
    // If hot exit ran ON MainActor, those tasks could never resume → deadlock.
    // Store Task in AppDelegate property to prevent ARC deallocation.
    // `_ = task` does NOT retain — the property does.
    self.hotExitTask = Task.detached(priority: .userInitiated) {
        await AppState.shared.beginShutdown()  // awaits all cancellations
        await SessionService.shared.hotExit()  // <500ms — saves ALL critical data + dirty_flag=0
        // cleanShutdown (PRAGMA optimize, incremental_vacuum, wal_checkpoint) is BEST-EFFORT.
        // If the watchdog fires at 3s, it's OK to skip this — WAL auto-recovers on next launch.
        // Critical data is already committed by hotExit.
        await SessionService.shared.cleanShutdown()
        hotExitCompleted.withLock { $0 = true }
        await MainActor.run { sender.reply(toApplicationShouldTerminate: true) }
    }
    // 3-second watchdog: uses RunLoop Timer (not async/await — INTENTIONAL exception).
    // The watchdog MUST bypass Swift's cooperative thread pool and actor system entirely,
    // because those may be the source of the hang. Timer fires on the NSRunLoop which
    // continues spinning during .terminateLater. This is the SOLE permitted use of
    // non-async timer in the entire codebase.
    Timer.scheduledTimer(withTimeInterval: 3.0, repeats: false) { _ in
        let completed = hotExitCompleted.withLock { $0 }
        if !completed {
            sender.reply(toApplicationShouldTerminate: true)
        }
    }

    return .terminateLater
}
```

### SIGTERM (System Shutdown)

Register `NSApplicationWillTerminate`. Same hot-exit sequence. 5-second budget.

### SIGKILL (Force Quit)

Cannot catch. Continuous backup ensures max 500ms data loss. SQLite WAL auto-recovers on next launch.

---

## 16. ACCESSIBILITY

Unchanged from original spec. All requirements apply:

- VoiceOver: all UI elements labeled, editor navigable by line/word/character
- Keyboard-only operation: every feature reachable without mouse
- High contrast theme (WCAG AAA 7:1)
- Color-blind theme (deuteranopia/protanopia safe)
- Reduced motion: respect `isReduceMotionEnabled`
- Live regions for status bar announcements
- Diff hunks as `NSAccessibilityGroup` with descriptive labels

---

## 17. DISTRIBUTION

Identical to original spec:
- Mac App Store + Homebrew Cask + GitHub Releases (.dmg)
- GPL v3 + Section 7 App Store exception
- CLA required for contributors
- Hardened Runtime + notarization
- Privacy policy URL
- Source code at git tag matches shipped binary

---

## 18. DEPENDENCIES (Swift Package Manager)

```swift
// Package.swift (or Xcode SPM integration)
dependencies: [
    .package(url: "https://github.com/groue/GRDB.swift", from: "7.0.0"),   // SQLite
    .package(url: "https://github.com/apple/swift-collections", from: "1.1.0"),  // OrderedDictionary, Deque
    .package(url: "https://github.com/facebook/zstd", from: "1.5.0"),       // zstd compression (C lib)
]
// tree-sitter: bundled as C source in TreeSitter/ directory (not SPM — needs custom module map)
// libgit2: bundled as C source or via SPM wrapper (SwiftGit2) for git status/diff/blame
//   License: GPL-v2 with linking exception (explicitly permits use in any program regardless of license)
//   App Sandbox blocks Process("/usr/bin/git") — libgit2 reads .git directly, no process spawn
// No other dependencies. Foundation, AppKit, Metal, CoreText, Security — all system frameworks.
```

### Dependency Count: 4 (GRDB, swift-collections, zstd, libgit2 + tree-sitter C source)

Compare to Rust architecture: ~15 crates (ropey, memchr, regex-automata, rayon, ignore, encoding_rs, rusqlite, thiserror, tracing, mimalloc, simdutf8, bytecount, dashmap, tree-sitter, zstd).

---

## 19. BUILD & CI

### Build Command

```bash
# Single command builds everything:
xcodebuild -scheme MyNotepadPP -configuration Release -destination 'platform=macOS'

# No cargo build. No cbindgen. No bridging header generation.
```

### CI (GitHub Actions)

```yaml
jobs:
  build-and-test:
    runs-on: macos-14  # Sonoma + Apple Silicon
    steps:
      - uses: actions/checkout@v4
      - run: xcodebuild test -scheme MyNotepadPP -destination 'platform=macOS'
      - run: xcodebuild test -scheme MyNotepadPPUITests -destination 'platform=macOS'
      - run: swiftlint lint --strict MyNotepadPP/
      # No cargo fmt, clippy, audit — no Rust
```

### Xcode Configuration

| Setting | Value |
|---------|-------|
| Deployment Target | macOS 14.0 |
| Swift Language Version | 6.0 |
| Strict Concurrency | `complete` |
| Default Actor Isolation | `nonisolated` (for C interop) |
| Architectures | arm64 (dev), arm64 + x86_64 (release universal) |
| Hardened Runtime | Enabled |
| App Sandbox | Enabled |

---

## 20. WHAT CHANGES IF WE EVER WANT CROSS-PLATFORM

This is the one-way-door risk. If cross-platform is ever needed:

| Option | Cost | Time |
|--------|------|------|
| Extract Core Engine to a Swift package, rewrite UI per platform | Medium (Swift core works on Linux via Swift-on-Linux; iOS natively) | 2-3 months per platform |
| Rewrite core in Rust, keep Swift UI | High (redo all engine work in different language) | 3-4 months |
| Port entire app to Kotlin Multiplatform | Very high (rewrite everything) | 6+ months |

**The pure-Swift core IS portable to iOS (SwiftUI) and Linux (Swift-on-Linux + GTK bindings) without a rewrite.** Windows and Android would require a C/C++ or Kotlin bridge. This is a reasonable trade-off for Mac-only v1.0.

---

*This document supersedes the Rust+Swift architecture for Mac-only deployment. The original `FUNCTIONAL_SPECIFICATION.md` and `CLAUDE.md` remain as reference for the cross-platform design. All user-facing features, shortcuts, behaviors, and performance targets are identical between the two architectures.*
