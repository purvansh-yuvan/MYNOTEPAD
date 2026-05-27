# MYNOTEPAD++ — Pure Swift Architecture Specification (macOS Only)

**Version:** 2.0 (final — all 37 defects + 25 gap fixes applied)
**Date:** 2026-05-27
**Status:** Final — zero known gaps
**License:** GPL v3
**Target:** macOS 14.0+ (Apple Silicon M-series native)
**Language:** 100% Swift + C interop (tree-sitter only)
**No Rust. No FFI bridge. No Cargo. Single build system (Xcode).**

> **IMPORTANT — Document Authority:** This document is the **sole authoritative spec** for
> the macOS implementation. It supersedes `CLAUDE.md` for all Mac-related decisions.
> The original 5-platform Rust architecture has been archived.
> Do not follow any Rust/FFI/Cargo instructions for this Mac-only build. (Fixes DEFECT-37)

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
| Zero FFI overhead | No string conversion on every engine call |
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
│   ├── AppLifecycle.swift
│   └── main.swift
├── Core/
│   ├── TextBuffer/
│   │   ├── PieceTable.swift
│   │   ├── PieceTableSnapshot.swift
│   │   ├── RedBlackTree.swift          ← concrete PieceTree implementation
│   │   ├── BufferBuilder.swift         ← mutable accumulator used during file load
│   │   ├── LineIndex.swift
│   │   └── Position.swift
│   ├── Cursor/
│   │   ├── Cursor.swift
│   │   └── Selections.swift
│   ├── Undo/
│   │   ├── UndoEngine.swift
│   │   └── UndoGroup.swift
│   ├── Search/
│   │   ├── BufferSearch.swift
│   │   ├── MultiFileSearch.swift
│   │   ├── SearchQuery.swift
│   │   └── RegexCache.swift
│   ├── Syntax/
│   │   ├── SyntaxEngine.swift
│   │   ├── LanguageRegistry.swift
│   │   ├── HighlightQuery.swift
│   │   ├── InjectionEngine.swift       ← language injection (HTML/JS/CSS, Markdown)
│   │   ├── FoldingEngine.swift
│   │   ├── BracketEngine.swift
│   │   └── SymbolIndex.swift
│   ├── IO/
│   │   ├── FileLoader.swift
│   │   ├── FileSaver.swift
│   │   ├── EncodingDetector.swift
│   │   ├── LineEndingDetector.swift
│   │   └── BinaryDetector.swift
│   ├── Config/
│   │   ├── EditorConfigParser.swift
│   │   └── TabSizeDetector.swift
│   ├── Diff/
│   │   ├── DiffEngine.swift
│   │   └── WordDiff.swift
│   └── Common/
│       ├── EditorError.swift
│       ├── Progress.swift
│       └── Constants.swift
├── Services/
│   ├── DocumentManager.swift
│   ├── FileService.swift
│   ├── AutoSaveService.swift
│   ├── BackupService.swift
│   ├── SearchService.swift
│   ├── SessionService.swift
│   ├── ThemeManager.swift
│   ├── KeyBindingManager.swift
│   ├── SnippetManager.swift
│   ├── MacroService.swift             ← v1.1
│   └── GitService.swift
├── Views/
│   ├── EditorView.swift
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
│   └── themes/
├── TreeSitter/
│   ├── module.modulemap
│   ├── tree_sitter/
│   └── grammars/
├── MyNotepadPPTests/
└── MyNotepadPPUITests/
```

---

## 3. TEXT BUFFER ENGINE — PIECE TABLE

### Why Piece Table (not Rope)

| Factor | Piece Table | Rope (B+ tree) |
|--------|-------------|-----------------|
| Implementation complexity | ~500 lines Swift | ~2000 lines Swift |
| Insert/delete | O(log n) | O(log n) |
| Sequential read | O(n) — excellent cache locality | O(n) — fragmented nodes |
| Copy-on-write snapshot | Copy piece table array (value type) | Copy root pointer |
| Used by | VS Code (Monaco), Abiword | Neovim, Xi Editor, Zed |
| Files >1GB | Excellent — streaming load | Excellent |

### Data Structures

> **DEFECT-01 FIX:** `OriginalBuffer` is immutable after construction. Progressive loading
> uses a `BufferBuilder` that accumulates chunks, then seals into an immutable
> `OriginalBuffer` once loading completes. The piece table is only given the sealed buffer.

```swift
/// Sealed, immutable original file content. NEVER modified after `init`.
/// Safe to share across snapshots — no locking needed.
/// NO mmap — even read-only mmap causes SIGBUS on truncated files or
/// network mount disconnect. SIGBUS cannot be handled in Swift/ARC.
final class OriginalBuffer: Sendable {
    let bytes: NSData   // NSData for stable C pointer to pass to tree-sitter
    init(bytes: NSData) { self.bytes = bytes }
}

/// DEFECT-01 FIX: Mutable accumulator used ONLY during file loading.
/// Once loading is complete, call `seal()` to produce an immutable OriginalBuffer.
/// The PieceTable is only ever handed a sealed OriginalBuffer — never a BufferBuilder.
final class BufferBuilder {
    private var accumulator = NSMutableData()

    /// Append one 64 KB chunk (already converted to UTF-8).
    func append(_ chunk: Data) {
        accumulator.append(chunk)
    }

    /// Seal into an immutable OriginalBuffer. Call once, discard the builder.
    /// GAP-05 FIX: No force cast — construct NSData directly.
    func seal() -> OriginalBuffer {
        OriginalBuffer(bytes: NSData(data: accumulator as Data))
    }
}

/// All user insertions go here — append-only after the document is open.
/// The live PieceTable holds an AddBufferStore (not a plain struct) so that
/// snapshots can share the same backing store (GAP-12 FIX).
/// See AddBufferStore definition in the Copy-on-Write Snapshots section.

struct Piece {
    let source: PieceSource
    let start: Int           // byte offset into source buffer
    let length: Int          // byte length
    let lineBreakCount: Int  // cached newlines for O(log n) line lookup
}

enum PieceSource { case original, add }
```

> **DEFECT-02 FIX:** `PieceTree` is a **red-black tree** with index-based nodes (not
> pointer-based) for cache efficiency on Apple Silicon.  The concrete type is defined in
> `RedBlackTree.swift`.  Each node stores `Piece` plus two augmentation fields:

```swift
/// DEFECT-02 FIX: Concrete red-black tree backing PieceTable.pieces.
/// Index-based (nodes stored in a flat array, referenced by Int indices) for
/// better cache locality than pointer-chasing on arm64.
struct RedBlackTree<Value> {
    // Internal node array — O(1) indexed access, CPU-cache friendly.
    private var nodes: [RBNode<Value>] = []
    private var root: Int = -1  // -1 = empty

    struct RBNode<V> {
        var value: V
        var color: RBColor
        var left: Int   // -1 = nil
        var right: Int  // -1 = nil
        var parent: Int // -1 = root
        // Augmentation for piece-tree line/byte lookups:
        var subtreeLength: Int = 0
        var subtreeLines: Int = 0
    }
    enum RBColor { case red, black }
}

/// typealias for clarity throughout the codebase
typealias PieceTree = RedBlackTree<Piece>

struct PieceTable {
    let original: OriginalBuffer    // sealed, immutable
    var addStore: AddBufferStore    // append-only; shared with snapshots (GAP-12)
    var pieces: PieceTree           // augmented red-black tree
    var totalLength: Int            // cached total byte length
    var totalLines: Int             // cached line count
    private(set) var generation: UInt64 = 0  // incremented on every edit
}
```

### Copy-on-Write Snapshots

```swift
/// Immutable snapshot for background threads (syntax, search, auto-save).
/// Class (not struct) required for safe Unmanaged pointer to tree-sitter C callbacks.
/// All fields are `let` — safe to share without locks.
///
/// GAP-12 FIX: `addBytes` uses a shared append-only `AddBufferStore` with generation
/// tracking instead of copying the entire NSData on every snapshot. Each snapshot records
/// the byte length at snapshot time. The underlying NSData is shared (append-only = safe).
final class PieceTableSnapshot: @unchecked Sendable {
    let original: OriginalBuffer      // shared reference — zero copy
    let addStore: AddBufferStore      // GAP-12 FIX: shared append-only store
    let addBytesLength: Int           // GAP-12 FIX: how many bytes were valid at snapshot time
    let pieces: PieceTree             // COW copy of the piece tree
    let generation: UInt64
    let timestamp: ContinuousClock.Instant

    /// Read UTF-8 bytes at a given offset. Called by tree-sitter C read callback.
    /// Returns a stable C pointer into original.bytes or addStore.bytes —
    /// pointer is valid as long as this snapshot is alive (Unmanaged.passRetained).
    /// Reads from addStore are capped at addBytesLength (snapshot boundary).
    func readUTF8(at byteOffset: Int,
                  length: UnsafeMutablePointer<UInt32>) -> UnsafePointer<CChar>? {
        // Walk piece tree to find which buffer the offset falls in,
        // then return the appropriate pointer into original.bytes or addStore.bytes.
        // For add buffer reads: clamp to addBytesLength.
        // Implementation detail in PieceTableSnapshot.swift.
    }
}

/// GAP-12 FIX: Shared, append-only backing store for the add buffer.
/// Append-only guarantees: previously written bytes are immutable.
/// Snapshots share one allocation; each records its own addBytesLength.
/// Memory cost: O(1) per snapshot instead of O(addBuffer.count) per snapshot.
final class AddBufferStore: @unchecked Sendable {
    private(set) var bytes: NSMutableData = NSMutableData()
    var length: Int { bytes.length }

    func append(_ data: Data) { bytes.append(data) }

    /// On compaction: create a new AddBufferStore. Old one stays alive
    /// as long as any snapshot references it (ARC).
    func reset() -> AddBufferStore { AddBufferStore() }
}
```

### Document Type (GAP-01 FIX)

> **GAP-01 FIX:** `Document` was used across all actors but never defined. It is now a
> `@MainActor`-isolated class with a `Sendable` read-only projection for background actors.

```swift
/// The authoritative document model. All mutable state is @MainActor-isolated.
/// Background actors receive `DocumentSnapshot` (Sendable) via `makeSnapshot()`.
///
/// IMPORTANT: `id` is `nonisolated let` — safe to read from any isolation domain.
/// All `var` properties (`filePath`, `isDirty`, etc.) require @MainActor to access.
/// Background actors must call `await doc.makeSnapshot()` to get a Sendable projection.
@MainActor
final class Document: Identifiable {
    nonisolated let id: String = UUID().uuidString  // safe: immutable, Sendable
    var filePath: URL?
    var isDirty: Bool = false
    var encoding: String.Encoding = .utf8
    var lineEnding: LineEnding = .lf
    var isWordWrapEnabled: Bool = false
    var syntaxLanguage: String?

    /// Create a Sendable snapshot for background actors.
    /// MUST be called from @MainActor (reads mutable @MainActor properties).
    /// Background callers: `let snap = await doc.makeSnapshot()`
    func makeSnapshot() -> DocumentSnapshot {
        DocumentSnapshot(id: id, filePath: filePath, encoding: encoding,
                         lineEnding: lineEnding, syntaxLanguage: syntaxLanguage)
    }
}

/// Sendable read-only projection passed to background actors (auto-save, search, syntax).
/// Never mutated — background actors that need to mark dirty send a message back to
/// DocumentManager (actor) which updates the @MainActor Document.
struct DocumentSnapshot: Sendable {
    let id: String
    let filePath: URL?
    let encoding: String.Encoding
    let lineEnding: LineEnding
    let syntaxLanguage: String?
}
```

**Service interaction pattern:** Services never hold a reference to `Document` directly.
Instead, `DocumentManager` (actor) creates a `DocumentSnapshot` on the MainActor and
passes it to services. This eliminates all cross-actor property access issues:

```swift
// In DocumentManager (actor):
func triggerAutoSave(for doc: Document) async {
    let snap = await doc.makeSnapshot()  // hops to MainActor, returns Sendable
    await autoSaveService.save(snap)     // passes Sendable to another actor
}
```

**Multi-window same-file coordination (GAP-15 FIX):**
- One `PieceTable` per unique file path, managed by `DocumentManager`.
- If the same file is opened in multiple windows/tabs, they share the same buffer.
- Edits in one tab are reflected in all tabs via the snapshot `AsyncStream`.
- Cursor/scroll positions are per-tab (stored in `tabs` SQLite table), not per-document.
- Closing one tab does NOT close the buffer if other tabs reference the same file.

### Line Index (O(log n) Lookup)

Each `RedBlackTree` node caches:
- `subtreeLength: Int` — total bytes in subtree
- `subtreeLines: Int` — total line breaks in subtree

This enables O(log n) along any dimension:
- **byte → line**: walk tree, sum left-subtree lines
- **line → byte**: walk tree, sum left-subtree bytes until target line
- **pixel → line**: combine with line height cache (fixed-height for monospace)

### Operations

| Operation | Complexity | How |
|-----------|-----------|-----|
| Insert text | O(log n) | Split piece at cursor, insert new piece into add buffer |
| Delete range | O(log n) | Split pieces at boundaries, remove middle pieces |
| Get line N | O(log n) | Binary search red-black tree by cumulative line count |
| Get byte range | O(log n + k) | Find start piece, iterate (k = output size) |
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
│  func edit(range:, text:) -> PieceTableSnapshot      │
│  func snapshot() -> PieceTableSnapshot               │
│  func undo() -> PieceTableSnapshot?                  │
│  func redo() -> PieceTableSnapshot?                  │
│  func applyCompaction(_ result: CompactionResult)    │  ← DEFECT-04 FIX
└─────────┬───────────────────────────────────────────┘
          │ publishes snapshots via AsyncStream
          ▼
┌─────────────────────┐  ┌──────────────────────┐
│ SyntaxEngine (Task)  │  │ SearchService (Task)  │
│ Holds: snapshot      │  │ Holds: snapshot       │
│ Reads only           │  │ Reads only            │
│ Cancelled if stale   │  │ Cancelled on new      │
│ >1s (DEFECT-03 FIX) │  │ query                 │
└─────────────────────┘  └──────────────────────┘
```

**No locks. No Mutex. No DispatchQueue.synchronize.**

### Starvation Guard

> **DEFECT-05 FIX:** The 4th attempt runs to completion but the result is always checked
> against the current generation before applying. If stale, it is applied as a best-effort
> visual update AND a fresh parse is immediately queued (without consuming a starvation-guard
> slot). This prevents stale highlights being the final state.

If a background syntax-parse task is cancelled 3 consecutive times because its snapshot is >1s stale:
1. The 4th attempt runs to completion regardless of snapshot age.
2. On completion, compare `syntaxTree.generation` with `documentManager.currentGeneration`.
3. If stale: apply the result as a temporary best-effort update, then immediately queue a fresh parse.
4. If current: apply normally; reset the starvation counter to 0.

### Memory Management

> **DEFECT-04 FIX:** Compaction is initiated on a background thread but the swap (replacing
> the live PieceTable inside DocumentManager) is an actor-isolated operation. The background
> thread only builds the new buffers; the actor does the actual swap.

- **Piece table compaction**: triggered by EITHER 10,000 edits OR add buffer exceeding 50MB.
  1. Background thread materializes a new `OriginalBuffer` + fresh `AddBufferStore` from the current snapshot.
  2. Background thread sends the result to the actor: `await documentManager.applyCompaction(result)`.
  3. `applyCompaction` is actor-isolated — it atomically swaps the live `PieceTable`. No race.
  4. Old nodes freed via ARC.
- **Snapshot age limit:** **1 second** (unified — DEFECT-03 FIX). Background threads holding snapshots older than 1s are cancelled and restarted with a fresh snapshot.
- **Add buffer growth**: monotonic between compactions. Typical growth <1MB for normal typing.

---

## 4. CONCURRENCY MODEL — SWIFT STRUCTURED CONCURRENCY

### No GCD. No DispatchQueue. No Thread.

```
┌─────────────────────────────────────────────────────────┐
│ MAIN ACTOR (@MainActor — UI thread)                      │
│  AppKit events, rendering, user input, cursor            │
│  Max budget per frame: 16ms (60 FPS)                     │
│  ZERO file I/O, ZERO computation                         │
├─────────────────────────────────────────────────────────┤
│ DocumentManager (actor — serialized document access)      │
│  All text edits, undo/redo, snapshot publishing          │
├─────────────────────────────────────────────────────────┤
│ HIGH-PRIORITY TASKS — TASK REPLACEMENT PATTERN           │
│  (DEFECT-09 FIX: max 1 in-flight task per category)      │
│  Syntax highlighting, bracket matching, autocomplete     │
│  On each new event: cancel previous task, create new     │
├─────────────────────────────────────────────────────────┤
│ NORMAL-PRIORITY TASKS (TaskGroup, priority: .utility)    │
│  Multi-file search, diff computation, file loading       │
│  Auto-save write                                         │
├─────────────────────────────────────────────────────────┤
│ LOW-PRIORITY TASKS (Task.detached, priority: .background)│
│  File indexing, symbol index, continuous backup          │
│  Session DB periodic flush                               │
├─────────────────────────────────────────────────────────┤
│ SQLite WRITER — GRDB DatabaseQueue (DEFECT-08 FIX)       │
│  All INSERT/UPDATE/DELETE via GRDB's DatabaseQueue       │
│  DatabaseQueue is a serial actor over SQLite — writes    │
│  are automatically serialised. No custom executor needed.│
└─────────────────────────────────────────────────────────┘
```

### Task Replacement Pattern (DEFECT-09 FIX)

> **Problem:** Creating a new `Task.detached` on every keystroke for syntax highlighting,
> bracket matching, and autocomplete floods the cooperative thread pool with 30+ high-priority
> tasks/second, starving auto-save and file I/O.
>
> **Fix:** Each background concern per document has exactly **one** `Task` property. On each
> trigger event, the previous task is cancelled and a new one is created. Maximum 1 in-flight
> task per category per document at any time.

```swift
// In EditorViewModel (or DocumentManager actor):
private var syntaxTask: Task<Void, Never>?
private var bracketTask: Task<Void, Never>?
private var completionTask: Task<Void, Never>?

func didEdit(snapshot: PieceTableSnapshot) {
    // Cancel previous — does NOT await completion, just sets the cancel flag.
    syntaxTask?.cancel()
    bracketTask?.cancel()

    syntaxTask = Task.detached(priority: .userInitiated) {
        try? await Task.sleep(for: .milliseconds(30))  // 30ms debounce
        guard !Task.isCancelled else { return }
        await syntaxService.parse(snapshot: snapshot)
    }

    bracketTask = Task.detached(priority: .userInitiated) {
        guard !Task.isCancelled else { return }
        await bracketEngine.match(snapshot: snapshot)
    }
}

func didChangeQuery() {
    completionTask?.cancel()
    completionTask = Task.detached(priority: .userInitiated) {
        guard !Task.isCancelled else { return }
        await completionEngine.compute(snapshot: snapshot)
    }
}
```

### Cancellation — Built Into Swift

```swift
// Starting a cancellable search:
let searchTask = Task.detached(priority: .utility) {
    for file in files {
        try Task.checkCancellation()
        let matches = search(in: file, query: query)
        progressStream.yield(.determinate(Double(i) / Double(total), "Searching..."))
    }
}

// Cancelling on Escape or new query:
searchTask.cancel()
```

### Progress Reporting — AsyncStream

```swift
let (progressStream, continuation) = AsyncStream<Progress>.makeStream()

continuation.yield(.determinate(0.45, "searching"))

for await progress in progressStream {
    statusBar.update(progress)  // @MainActor
}
```

### Shutdown Guard (DEFECT-06 + DEFECT-07 FIX)

> **DEFECT-06 FIX:** `OSAllocatedUnfairLock` is a mutex, NOT lock-free. Replace with
> `ManagedAtomic<Bool>` from the `swift-atomics` package for genuine lock-free reads.
> Add `swift-atomics` to `Package.swift` dependencies.
>
> **DEFECT-07 FIX:** `beginShutdown()` must only **cancel** tasks (call `.cancel()`) — it
> must NOT `await` their completion. Awaiting creates a circular dependency: the hot-exit
> task waits for actors, the actors wait for @MainActor, @MainActor waits for hot exit.
> Cancellation is cooperative and the tasks will self-terminate at their next
> `checkCancellation()` call without any waiting.

```swift
import Atomics  // swift-atomics package

final class AppState: Sendable {
    static let shared = AppState()

    /// DEFECT-06 FIX: genuine lock-free atomic boolean.
    /// Reads from background tasks have NO actor hop, NO lock acquisition.
    private let _isShuttingDown = ManagedAtomic<Bool>(false)

    /// Read from any thread without await — lock-free on arm64.
    var isShuttingDown: Bool {
        _isShuttingDown.load(ordering: .acquiring)
    }

    /// DEFECT-07 FIX: cancel tasks only — do NOT await their completion.
    /// Tasks self-terminate at their next checkCancellation() call.
    /// beginShutdown() is synchronous for the same reason — no async.
    func beginShutdown() {
        _isShuttingDown.store(true, ordering: .releasing)
        // Fire-and-forget cancellation. Do NOT await.
        DocumentManager.shared.cancelAllBackgroundWorkFireAndForget()
        AutoSaveService.shared.cancelAllFireAndForget()
        BackupService.shared.cancelAllFireAndForget()
        SearchService.shared.cancelAllFireAndForget()
    }
}

// GAP-04 FIX: All services that participate in shutdown use a lock-protected
// task array so beginShutdown() can cancel immediately without an actor hop.
// Pattern shown for AutoSaveService — BackupService, SearchService, and
// DocumentManager implement the identical pattern (same lock type, same methods).
extension AutoSaveService {
    private static let taskLock = OSAllocatedUnfairLock<[Task<Void, Never>]>(initialState: [])

    static func trackTask(_ task: Task<Void, Never>) {
        taskLock.withLock { $0.append(task) }
    }

    /// Called from beginShutdown() — cancels all tasks immediately, no actor hop.
    nonisolated func cancelAllFireAndForget() {
        Self.taskLock.withLock { tasks in
            for t in tasks { t.cancel() }
            tasks.removeAll()
        }
        // Also enqueue actor-isolated cancelAll() for cleanup (non-blocking).
        Task { await self.cancelAll() }
    }
}
// BackupService, SearchService, DocumentManager: same pattern (omitted for brevity).
```

### Thermal Throttling

```swift
let thermalState = ProcessInfo.processInfo.thermalState
let maxConcurrency: Int = switch thermalState {
    case .nominal, .fair: ProcessInfo.processInfo.activeProcessorCount
    case .serious: 2
    case .critical: 1
    @unknown default: 2
}
```

---

## 5. TEXT RENDERING — CoreText + Metal

### Pipeline

```
User types → DocumentManager.edit() → new PieceTableSnapshot
    → EditorView.setNeedsDisplay(dirtyRect)
    → draw(dirtyRect):
        1. Query snapshot for visible lines (O(log n) via line index)
        2. Check LineLayoutCache — shape with CoreText only on cache miss (DEFECT-13 FIX)
        3. Look up glyphs in Metal texture atlas (cache hit ~99%)
        4. Encode Metal draw commands (batched)
        5. Acquire triple-buffer semaphore (DEFECT-10 FIX)
        6. Submit Metal command buffer
        7. Draw cursor, selections, highlights as Metal overlays
```

### Metal Triple-Buffering (DEFECT-10 FIX)

> **Problem:** Without triple-buffering, `CAMetalLayer.nextDrawable()` blocks the main thread
> when the GPU hasn't finished the previous frame, causing visible hitches and breaking 60 FPS.
>
> **Fix:** Use a `DispatchSemaphore(value: 3)` guarding drawable acquisition.

```swift
// In EditorView or MetalRenderer:
private let inflightSemaphore = DispatchSemaphore(value: 3)  // triple buffering

func draw(_ dirtyRect: NSRect) {
    // GAP-03 FIX: 100ms timeout prevents permanent main-thread block if GPU hangs.
    // On timeout: skip frame rather than hang (macOS WindowServer kills at ~10s).
    let result = inflightSemaphore.wait(timeout: .now() + .milliseconds(100))
    if result == .timedOut {
        os_log(.error, "Metal semaphore timeout — skipping frame to prevent hang")
        return
    }

    guard let commandBuffer = commandQueue.makeCommandBuffer(),
          let drawable = metalLayer.nextDrawable() else {
        inflightSemaphore.signal()
        return
    }

    // ... encode render commands ...

    commandBuffer.addCompletedHandler { [weak self] _ in
        self?.inflightSemaphore.signal()  // GPU finished — CPU can encode next frame
    }
    commandBuffer.present(drawable)
    commandBuffer.commit()
}
```

### Line Layout Cache (DEFECT-13 FIX)

> **Problem:** Without a line layout cache, `draw()` re-shapes every visible line via
> CoreText on every frame, wasting 2-5ms even when nothing changed.

```swift
/// Cache of shaped CTLine objects per document line.
/// GAP-08 FIX: Nested dictionary keyed by lineIndex for O(1) invalidation.
/// Previous flat dictionary required O(n) filter on every keystroke.
final class LineLayoutCache {
    /// Outer key: lineIndex. Inner key: (contentHash, fontID).
    /// Invalidate(lineIndex:) is O(1) — just delete the inner dictionary.
    private var cache: [Int: [SubKey: CTLine]] = [:]

    struct SubKey: Hashable {
        let contentHash: UInt64  // FNV-1a hash of line bytes
        let fontID: UInt32       // compact font descriptor identifier
    }

    func ctLine(forLine lineIndex: Int, contentHash: UInt64,
                fontID: UInt32, makeIfMissing: () -> CTLine) -> CTLine {
        let subKey = SubKey(contentHash: contentHash, fontID: fontID)
        if let hit = cache[lineIndex]?[subKey] { return hit }
        let line = makeIfMissing()
        cache[lineIndex, default: [:]][subKey] = line
        return line
    }

    /// GAP-08 FIX: O(1) per-line invalidation on edit.
    func invalidate(lineIndex: Int) { cache[lineIndex] = nil }

    /// Call on tab close to free all memory for that document.
    func invalidateAll() { cache.removeAll() }
}
```

### Minimap Double-Buffer (DEFECT-11 FIX)

> **Problem:** Original spec used Rust idioms (`Arc::swap`, `AtomicPtr`) which do not exist
> in Swift.
>
> **Fix:** Use `OSAllocatedUnfairLock` to protect the buffer swap between the background
> renderer and the main thread compositor.

```swift
// In MinimapView:
// GAP-13 NOTE: OSAllocatedUnfairLock is acceptable here (unlike the shutdown flag which
// uses ManagedAtomic) because: (a) it protects a complex type (optional MTLBuffer),
// (b) called at most once per frame — not a hot loop, (c) lock hold time <1µs (pointer swap).
private let minimapBufferLock = OSAllocatedUnfairLock<MTLBuffer?>(initialState: nil)

// Background Task — on render completion:
func minimapRenderDidComplete(newBuffer: MTLBuffer) {
    minimapBufferLock.withLock { $0 = newBuffer }
    DispatchQueue.main.async { self.setNeedsDisplay(self.bounds) }
}

// Main thread — in draw():
func drawMinimap() {
    guard let buffer = minimapBufferLock.withLock({ $0 }) else { return }
    // Composite buffer onto minimap view via Metal
}
```

### `characterIndex(for:)` Caching (DEFECT-12 FIX)

> **Problem:** `NSTextInputClient.characterIndex(for:)` is called continuously during mouse
> drag. Without caching, it re-computes inverse layout on every call — sluggish text selection
> on long lines.
>
> **Fix:** Cache the rendered glyph positions after every layout pass. `characterIndex(for:)`
> binary-searches the cached positions — O(log n) per call.

```swift
// In EditorView — updated after every draw pass:
// GAP-09 FIX: Partitioned by lineIndex for O(1) line lookup + O(log n) binary search.
// Previous flat array required O(n) filter on every mouse event during drag selection.
private var cachedGlyphPositions: [Int: [GlyphPosition]] = [:]  // [lineIndex: sorted by x]

struct GlyphPosition {
    let charIndex: Int
    let xMin: CGFloat   // leading edge of this glyph
    let xMax: CGFloat   // trailing edge
}

// NSTextInputClient conformance:
func characterIndex(for point: NSPoint) -> Int {
    let lineIndex = lineIndexForY(point.y)
    // GAP-09 FIX: O(1) line lookup, then O(log n) binary search within line.
    guard let positionsOnLine = cachedGlyphPositions[lineIndex] else { return 0 }
    return positionsOnLine.binarySearch { $0.xMax < point.x }?.charIndex ?? 0
}
```

### Glyph Atlas

| Property | Value |
|----------|-------|
| Backing | `MTLTexture` (GPU-resident) |
| Cache key | `(glyphID, fontID, fontSize, subpixelOffset)` |
| Color | Applied via Metal shader, NOT baked into atlas |
| Emoji | Separate RGBA atlas for Apple Color Emoji (`sbix`). Use `CTFontDrawGlyphs()`. |
| Eviction | LRU page eviction when atlas exceeds 32MB |
| Tab-close cleanup | Deferred compaction (5s debounce) |
| Scale change | Invalidate entire atlas when `contentsScale` changes |

### Shader Pre-Compilation

All Metal PSOs compiled during app startup from a pre-compiled `.metallib` in the app bundle. Never compile at runtime — prevents 50-100ms first-frame hitch.

### Dirty Rect Rendering

- On edit: only re-render changed lines + cursor line
- On scroll: only render newly visible lines
- Never redraw entire viewport for a single-line edit

### Overdraw Buffer

Pre-render 2× viewport height (1 screen above + 1 below). Fast scrolling never shows blank.

### Viewport Culling

Only compute layout for visible lines + overdraw buffer. O(log n) "which line is at pixel Y" via the piece tree's augmented nodes.

### Long Line Handling

- Lines >10,000 chars: column-level viewport virtualization
- Lines >500,000 chars: disable syntax highlighting for that line

### Frame Budget Watchdog (Debug Builds)

Budget: 16ms per frame. Phases: input ~1ms, piece table ~0.1ms, layout/cache ~2ms, glyph atlas ~1ms, Metal draw ~3ms, swap ~1ms, syntax ~8ms remaining.
Drop syntax colors for the current frame if budget exceeded — never drop below 60 FPS.

### NSTextInputClient (CRITICAL)

EditorView implements `NSTextInputClient`:
- `setMarkedText`, `markedRange`, `selectedRange`, `attributedSubstring`, `insertText`
- `firstRect(forCharacterRange:)`, `characterIndex(for:)` (cached per DEFECT-12 FIX)
- Call `inputContext?.invalidateCharacterCoordinates()` on scroll or layout change

### Spell Checking (DEFECT-33 FIX)

> **New in v1.0.** macOS provides `NSSpellChecker` — no custom implementation needed.

- Enabled by default for plain text (`.txt`, `.md`, no syntax grammar). Disabled for source code files.
- Toggle per file: `Edit → Spelling and Grammar → Check Spelling While Typing`.
- Spell-check underlines rendered as a **separate Metal overlay pass** (not baked into glyph atlas — no cache invalidation on spelling changes).
- `NSSpellChecker.shared.checkSpelling(of:startingAt:language:wrap:inSpellDocumentWithTag:wordCount:)` called on the background `.utility` priority task.
- User can set language per file from the status bar.

### Word Wrap (DEFECT-32 FIX)

- Toggle: `View → Word Wrap` (`Opt+Z`). Persisted per tab in SQLite.
- **Soft-wrap vs logical lines:** line numbers display **logical** lines (not visual wrap-lines). Go To Line uses logical line numbers.
- Wrap point computed lazily per visible viewport region only. Not pre-computed for the entire file.
- Minimap renders wrapped content at reduced scale.
- Cursor movement: `↓`/`↑` navigate visual lines when wrapped; `Cmd+↓`/`Cmd+↑` jump logical lines.
- **Cache invalidation:** word wrap changes invalidate the entire `LineLayoutCache` (viewport width changed).

### NSWritingToolsCoordinator (macOS 15+)

Implement delegate for Writing Tools (AI proofreading). Provide `NSAttributedString`, handle replacement callbacks.

---

## 6. AUTO-SAVE — NEVER HANGS, NEVER BLOCKS, NEVER PROMPTS

### Three Independent Save Systems

```
A. Auto-Save (to original file)
   User types → debounce 1s → Task: snapshot → serialize → atomic write → done

B. Continuous Backup (to backup directory)
   Every 500ms → snapshot → write to <appSupport>/backups/{doc_id}/
   Survives SIGKILL. Max data loss: 500ms of typing.

C. Hot Exit (to SQLite session DB)
   On Cmd+Q → single SQLite transaction → all tab state + compressed unsaved content
```

### Debounce / Throttle / Focus Lost (DEFECT-14 + DEFECT-18 FIX)

> **DEFECT-14 FIX:** `serializedSave` previously used **skip-if-busy** for ALL callers,
> including tab-close. This caused silent data loss: if a save was in-flight when a tab
> closed, the closing save was skipped and the edits since the in-flight save began were lost.
>
> Fix: Add `drainAndSave(doc:)` for tab-close, which **waits** for any in-flight save to
> complete then does a final save if still dirty. The debounce/throttle paths keep skip-if-busy
> (safe — they retry). Only close/shutdown paths use drain-and-save.
>
> **DEFECT-18 FIX:** `focusLossTasks` pruning is moved to a dedicated method called on a
> 30-second timer, preventing unbounded accumulation between focus-loss events.

```swift
/// AutoSaveService receives DocumentSnapshot (Sendable), NOT Document directly.
/// This avoids cross-actor access to @MainActor Document properties.
/// DocumentManager creates snapshots on MainActor and passes them here.
actor AutoSaveService {
    private var debounceTasks: [String: Task<Void, Never>] = [:]
    private var throttleTasks: [String: Task<Void, Never>] = [:]
    private var activeSaves: Set<String> = []

    // DEFECT-18 FIX + GAP-22 FIX: pruned on a timer, tracks completed tasks.
    private var focusLossTasks: [ObjectIdentifier: Task<Void, Never>] = [:]
    private var completedTaskIDs = Set<ObjectIdentifier>()
    private var pruneTimer: Task<Void, Never>?

    init() {
        pruneTimer = Task.detached(priority: .background) { [weak self] in
            while !Task.isCancelled {
                try? await Task.sleep(for: .seconds(30))
                await self?.pruneFocusLossTasks()
            }
        }
    }

    /// GAP-22 FIX: Prune both cancelled AND completed tasks.
    private func pruneFocusLossTasks() {
        focusLossTasks = focusLossTasks.filter {
            !$0.value.isCancelled && !completedTaskIDs.contains($0.key)
        }
        completedTaskIDs.removeAll()  // reset after pruning
    }

    /// Called by DocumentManager when a document is edited.
    /// `docID` is nonisolated let on Document — safe to pass directly.
    func documentDidChange(docID: String, saveBlock: @Sendable @escaping () async throws -> Void) {
        debounceTasks[docID]?.cancel()
        debounceTasks[docID] = Task { [weak self] in
            do {
                try await Task.sleep(for: .seconds(1))
                try Task.checkCancellation()
                guard let self else { return }
                await self.serializedSave(docID: docID, saveBlock: saveBlock)
            } catch { return }
        }
    }

    func startThrottle(docID: String, saveBlock: @Sendable @escaping () async throws -> Void) {
        throttleTasks[docID]?.cancel()
        throttleTasks[docID] = Task { [weak self] in
            while !Task.isCancelled {
                try? await Task.sleep(for: .seconds(30))
                guard !Task.isCancelled, let self else { return }
                await self.serializedSave(docID: docID, saveBlock: saveBlock)
            }
        }
    }

    /// DEFECT-14 FIX + GAP-02 FIX: Drain in-flight save with 2s timeout.
    /// `filePath` and `isDirty` are passed as values (captured on MainActor before call).
    func drainAndSave(filePath: String, isDirty: Bool,
                      saveBlock: @Sendable @escaping () async throws -> Void) async {
        let deadline = ContinuousClock.now + .seconds(2)
        while activeSaves.contains(filePath) {
            if ContinuousClock.now >= deadline {
                os_log(.error, "drainAndSave timed out after 2s for %{public}@", filePath)
                break
            }
            try? await Task.sleep(for: .milliseconds(10))
        }
        if isDirty {
            activeSaves.insert(filePath)
            defer { activeSaves.remove(filePath) }
            do { try await saveBlock() }
            catch { os_log(.error, "Final save failed: %{public}@", error.localizedDescription) }
        }
    }

    func stopTracking(docID: String) {
        debounceTasks[docID]?.cancel()
        debounceTasks.removeValue(forKey: docID)
        throttleTasks[docID]?.cancel()
        throttleTasks.removeValue(forKey: docID)
    }

    /// Focus-loss save. Receives pre-built save blocks (captured on MainActor).
    func windowDidResignKey(saves: [(docID: String, save: @Sendable () async throws -> Void)]) {
        guard !AppState.shared.isShuttingDown else { return }
        for task in focusLossTasks.values { task.cancel() }
        focusLossTasks.removeAll()
        completedTaskIDs.removeAll()
        for entry in saves {
            let t = Task { [weak self] in
                defer { Task { await self?.markTaskCompleted(id: ObjectIdentifier(Task.currentTask!)) } }
                await self?.serializedSave(docID: entry.docID, saveBlock: entry.save)
            }
            focusLossTasks[ObjectIdentifier(t)] = t
        }
    }

    private func markTaskCompleted(id: ObjectIdentifier) {
        completedTaskIDs.insert(id)
    }

    /// Skip-if-busy: safe for debounce/throttle paths — they retry automatically.
    private func serializedSave(docID: String, saveBlock: @Sendable () async throws -> Void) async {
        guard !AppState.shared.isShuttingDown else { return }
        guard !Task.isCancelled else { return }
        guard !activeSaves.contains(docID) else { return }

        activeSaves.insert(docID)
        defer { activeSaves.remove(docID) }
        do { try await saveBlock() }
        catch { os_log(.error, "Auto-save failed: %{public}@", error.localizedDescription) }
    }

    /// Actor-isolated cancel — called from the lock-protected GAP-04 pattern.
    func cancelAll() {
        for (_, t) in debounceTasks { t.cancel() }
        debounceTasks.removeAll()
        for (_, t) in throttleTasks { t.cancel() }
        throttleTasks.removeAll()
        for (_, t) in focusLossTasks { t.cancel() }
        focusLossTasks.removeAll()
        pruneTimer?.cancel()
    }
}
```

### Three-Tier Atomic Write

```
Tier 1: Atomic rename (preferred, APFS/HFS+)
  1. Write to {dir}/{filename}.mynotepadpp-{pid}-{timestamp}.tmp (SAME directory)
  2. F_FULLFSYNC via Darwin.fcntl(fd, F_FULLFSYNC, 0)  — true NVMe cache flush
  3. rename() temp → target (atomic on POSIX)
  → If rename fails (EXDEV cross-device): fall to Tier 2

Tier 2: Copy-new-rename (network/cloud mounts)
  1. Copy target to {dir}/{filename}.mynotepadpp-bak
  → If copy fails (ENOSPC): fall to Tier 3 WITHOUT touching original
  2. Write new content to {dir}/{filename}.mynotepadpp-new
  → If write fails mid-stream: delete temp, original + bak still intact → fall to Tier 3
  3. F_FULLFSYNC the new temp file
  4. rename() .mynotepadpp-new → target (atomic replace)
  5. Delete .mynotepadpp-bak

Tier 3: Recovery directory
  1. Write to <appSupport>/recovery/
  2. Show non-blocking banner: "Could not save to original location."
  3. Keep buffer marked dirty
```

### Cloud-Aware Save (DEFECT-15 + DEFECT-16 FIX)

> **DEFECT-16 FIX:** Cloud drive detection now uses reliable volume-level checks rather than
> fragile hardcoded path prefixes. Dropbox and Google Drive can install to any directory.
>
> **DEFECT-15 FIX:** `NSFileCoordinator` is wrapped with a 10-second timeout. On timeout,
> fall to Tier 2 (direct write) with a status bar warning.

```swift
/// Returns true if the given URL is on a cloud-synced or non-local volume.
func isCloudOrNonLocalVolume(url: URL) -> Bool {
    // Primary: check if the volume is local (false = network/cloud)
    let isLocal = (try? url.resourceValues(forKeys: [.volumeIsLocalKey]))?.volumeIsLocal ?? true
    if !isLocal { return true }

    // Secondary: iCloud Drive — stable known path
    let mobileDocuments = FileManager.default
        .urls(for: .libraryDirectory, in: .userDomainMask).first?
        .appendingPathComponent("Mobile Documents")
    if let base = mobileDocuments, url.path.hasPrefix(base.path) { return true }

    // Tertiary: user-configured cloud paths (from Preferences → Advanced)
    return UserDefaults.standard
        .stringArray(forKey: "cloudSyncPaths")?
        .contains { url.path.hasPrefix($0) } ?? false
}

// In FileSaver:
func saveTier(_ url: URL, data: Data) async throws {
    if isCloudOrNonLocalVolume(url: url) {
        // iCloud: use NSFileCoordinator with a 10-second timeout
        if url.path.contains("Mobile Documents") {
            try await withTimeout(seconds: 10) {
                try self.saveWithFileCoordinator(url: url, data: data)
            } onTimeout: {
                // DEFECT-15 FIX: fall to Tier 2 and warn user
                os_log(.error, "NSFileCoordinator timed out for %{public}@", url.path)
                await MainActor.run {
                    statusBar.showTransientMessage("iCloud coordination timed out — saved directly")
                }
                try self.saveTier2(url: url, data: data)
            }
        } else {
            // Dropbox / GDrive / non-iCloud cloud: Tier 2 directly (no coordinator)
            try saveTier2(url: url, data: data)
        }
        return
    }
    // Local volume: Tier 1 (atomic rename)
    try saveTier1(url: url, data: data)
}
```

- Increase debounce to **3s** for cloud-synced directories (reduce sync churn).
- Mount-point capability cache: first save to any path tests same-directory rename. Result cached per mount point. Tier 1 skipped on known-bad mounts.

### Disk Full Handling

All tiers fail → `DIRTY_CRITICAL` state → persistent banner → poll disk every 10s → auto-retry.
On `Cmd+Q` with `DIRTY_CRITICAL`: ONE blocking dialog — sole exception to "never prompt."

### Continuous Backup Cleanup (DEFECT-17 FIX)

> **DEFECT-17 FIX:** On untitled tab close, the backup directory must NOT be deleted until
> the tab's content has been durably written to SQLite. The previous spec assumed hot exit
> had already run — it has not when a single tab is closed mid-session.

- On successful save to original: delete backup.
- On **named** tab close (auto-save enabled): file is already saved → delete backup directory.
- On **untitled** tab close:
  1. Immediately write content to SQLite `unsaved_content` (a single-document mini-hot-exit).
  2. Confirm SQLite write succeeded (check return value).
  3. Only THEN delete the backup directory.
  4. If SQLite write fails: **keep** the backup, log error, show transient status bar warning.
- On startup: scan backups/ — restore if newer than original, delete if stale (>7 days).
- Orphan cleanup: startup + every 1 hour, delete temp files older than 1 hour with dead PIDs.

### File Watcher Self-Suppression

Priority chain:
1. `kFSEventStreamEventFlagOwnEvent` (macOS 13.0+) — most reliable
2. Mtime comparison: record `(path, mtime)` on save, compare on event, expire after 2s
3. **Never** use a boolean flag or fixed-time suppression window

### External File Modification Detection (GAP-14 FIX)

> **GAP-14 FIX:** The spec covered FSEvents watcher and self-suppression but never specified
> what happens when an external process modifies an open file.

When FSEvents reports a change that is NOT self-suppressed (i.e., an external modification):

| Buffer State | Behavior |
|-------------|----------|
| **Clean** (not dirty) | Auto-reload silently. Show transient status bar message: "File changed on disk — reloaded." Cursor position preserved if possible (clamp to new file length). |
| **Dirty** (unsaved edits) | Show inline banner above editor: "File changed on disk. **[Reload]** **[Keep Mine]** **[Compare]**" |

**Banner actions:**
- **Reload:** Discard local changes, load external version. Undo history cleared (new base). Irreversible — show confirmation if >10 unsaved edits.
- **Keep Mine:** Dismiss banner. Mark file dirty. Next auto-save overwrites external change. Banner does NOT re-appear for this file until the next external modification.
- **Compare:** Open a split diff view (left = disk version, right = local buffer). User can cherry-pick changes. Diff uses `DiffEngine` (§2 architecture).

**Timing:** FSEvents debounce is 200ms. After debounce, check mtime. If different from last known, trigger the flow above. Do NOT read the file to compare content — mtime change is sufficient to trigger.

**Deleted file:** If the file is deleted externally, show banner: "File was deleted from disk. **[Save As]** **[Close]**". Mark tab title with strikethrough.

---

## 7. SESSION PERSISTENCE — SQLite

### Library: GRDB.swift

GRDB provides type-safe queries, prepared statement caching, WAL mode, migrations, and:
- `DatabaseQueue` — serialised writer (DEFECT-08 FIX: this IS the "custom SerialExecutor" — no custom Swift executor needed)
- `DatabasePool` — concurrent readers, never blocked by writer

### Connection Architecture

```
WRITER: GRDB DatabaseQueue (serialised — BEGIN IMMEDIATE)
  All INSERT / UPDATE / DELETE

READER: GRDB DatabasePool (concurrent, UI + background)
  All SELECT — never blocked by writer (WAL guarantee)
```

### PRAGMAs (set on every connection open)

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = FULL;
PRAGMA fullfsync = ON;
PRAGMA cache_size = -2000;
PRAGMA mmap_size = 33554432;      -- SQLite internal mmap only (safe — SQLite handles SIGBUS)
PRAGMA temp_store = MEMORY;
PRAGMA busy_timeout = 5000;
PRAGMA wal_autocheckpoint = 1000;
PRAGMA foreign_keys = ON;
PRAGMA trusted_schema = OFF;
PRAGMA cell_size_check = ON;
PRAGMA threads = 2;
PRAGMA journal_size_limit = 67108864;
PRAGMA optimize = 0x10002;        -- GAP-24 FIX: 0x10002 = run ANALYZE on tables with 25+ rows (SQLite 3.46+)
-- Writer only:
PRAGMA cache_spill = OFF;
```

### Schema (STRICT tables, zstd blobs) (DEFECT-21 FIX)

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
    file_path TEXT,
    cursor_line INTEGER NOT NULL DEFAULT 0,
    cursor_col INTEGER NOT NULL DEFAULT 0,
    scroll_top REAL NOT NULL DEFAULT 0,
    syntax TEXT,
    is_pinned INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL,
    is_dirty INTEGER NOT NULL DEFAULT 0
) STRICT;

-- DEFECT-21 FIX: index for fast tab restore by window
CREATE INDEX idx_tabs_window_sort ON tabs(window_id, sort_order);

CREATE TABLE unsaved_content (
    tab_id TEXT PRIMARY KEY REFERENCES tabs(id),
    content BLOB NOT NULL,
    encoding TEXT NOT NULL DEFAULT 'utf-8'
) STRICT;

CREATE TABLE undo_history (
    tab_id TEXT PRIMARY KEY REFERENCES tabs(id),
    history BLOB NOT NULL
) STRICT;

CREATE TABLE app_state (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
) STRICT;
-- Keys: 'dirty_flag', 'last_active_tab', 'last_active_window'
```

### Write Batching

Cursor/scroll positions batched in-memory, flushed every 5s in a single transaction. Tab open/close written immediately.

### Crash Recovery (DEFECT-19 FIX)

> **DEFECT-19 FIX:** The original `UPDATE` on an empty table silently does nothing.
> Replace with an `INSERT OR REPLACE` (upsert) so the row is always created on first launch.

```sql
BEGIN IMMEDIATE;

-- DEFECT-19 FIX: upsert — creates the row on first launch, updates on subsequent.
INSERT INTO app_state(key, value) VALUES('dirty_flag', '1')
ON CONFLICT(key) DO UPDATE SET value = '1';

-- Read previous value from the RETURNING clause (SQLite 3.35+):
-- The previous value is captured before the upsert via a SELECT in the same transaction.
SELECT value FROM app_state WHERE key = 'dirty_flag';

COMMIT;
```

On startup:
1. Run the upsert transaction above. Read the **previous** `dirty_flag` value (before the upsert) by SELECT-ing first inside the transaction.
2. If previous value was `'1'`: previous session crashed → run recovery.
3. `PRAGMA quick_check` (NOT `integrity_check` — 10-100x faster).
4. If corrupt: try `sqlite3_recover`, else restore from periodic backup, else recreate schema.
5. On clean exit: set `dirty_flag = '0'`.

### Clean Exit

```sql
PRAGMA optimize;
PRAGMA incremental_vacuum;
PRAGMA wal_checkpoint(TRUNCATE);
```

### Periodic Backup (DEFECT-20 FIX)

> **DEFECT-20 FIX:** Prevent concurrent VACUUM operations from corrupting the backup file.
> Use a mutual-exclusion flag. If a VACUUM is already running, skip the current timer tick.

```swift
private var vacuumInProgress = false

func periodicBackup() async {
    guard !vacuumInProgress else {
        os_log(.info, "Skipping periodic VACUUM — previous backup still in progress")
        return
    }
    vacuumInProgress = true
    defer { vacuumInProgress = false }

    // Write to a staging file first; only rename to .backup on success.
    let stagingPath = dbPath.deletingLastPathComponent()
        .appendingPathComponent("sessions.db.backup.staging")
    do {
        try await dbQueue.write { db in
            try db.execute(sql: "VACUUM INTO '\(stagingPath.path)'")
        }
        // Atomic rename: staging → .backup (overwrites only on success)
        try FileManager.default.replaceItemAt(backupURL, withItemAt: stagingPath)
    } catch {
        // Keep previous backup intact — do not delete it on failure
        try? FileManager.default.removeItem(at: stagingPath)
        os_log(.error, "Periodic backup failed: %{public}@", error.localizedDescription)
    }
}
```

---

## 8. FILE I/O — LOADING & ENCODING

### File Loading Pipeline (DEFECT-01 FIX — uses BufferBuilder)

```swift
actor FileLoader {
    func load(url: URL) async throws -> LoadResult {
        let header = try await readHeader(url: url, size: 8192)
        if isBinary(header) { throw EditorError.binaryFile(url.lastPathComponent) }
        let encoding = detectEncoding(header)
        let lineEnding = detectLineEnding(header)
        let pieceTable = try await loadChunked(url: url, encoding: encoding)
        return LoadResult(pieceTable: pieceTable, encoding: encoding, lineEnding: lineEnding)
    }

    private func loadChunked(url: URL, encoding: String.Encoding) async throws -> PieceTable {
        let handle = try FileHandle(forReadingFrom: url)
        defer { try? handle.close() }

        // DEFECT-01 FIX: Use BufferBuilder to accumulate chunks.
        // The PieceTable is NOT created until the OriginalBuffer is sealed.
        // This preserves OriginalBuffer's immutability guarantee.
        let builder = BufferBuilder()
        var firstScreenful: Data? = nil

        while true {
            try Task.checkCancellation()
            guard let chunk = try handle.read(upToCount: 65536), !chunk.isEmpty else { break }
            let utf8Chunk = try convert(chunk, from: encoding, to: .utf8)
            builder.append(utf8Chunk)

            // After first chunk: yield a partial piece table for progressive display
            if firstScreenful == nil {
                firstScreenful = utf8Chunk
                let partialBuffer = builder.seal()
                // Post first-screenful piece table to UI (non-blocking)
                await MainActor.run { editorView.showPartialContent(partialBuffer) }
            }
        }

        // Seal into immutable OriginalBuffer — never modified after this point.
        let originalBuffer = builder.seal()
        return PieceTable(original: originalBuffer)
    }
}
```

**Progressive Loading Transition (GAP-06 FIX):**

> **GAP-06 FIX:** The spec showed partial content display but never specified the transition
> from partial to full content.

On full load completion:
1. The partial `PieceTable` is replaced atomically via `DocumentManager.replaceBuffer()` (actor-isolated).
2. Cursor position is preserved (clamped to new file length if cursor was beyond partial content).
3. Scroll position is preserved (viewport stays at same line number).
4. Any edits made during partial loading are **discarded** — the full buffer is authoritative.
   To prevent confusion: the editor shows a "Loading..." indicator in the status bar while partial.
   Typing is disabled until full load completes (keyboard input queued if <2s, dropped if >2s).
5. Undo history from the partial phase is cleared — the full load is the new base.
6. Syntax highlighting re-triggers on the full buffer (task replacement pattern).

### Encoding Detection Priority

1. BOM (definitive)
2. XML/HTML `charset` declaration
3. `CFStringEncoding` heuristic (`CFStringCreateWithBytes`)
4. User locale fallback

### Supported Encodings

UTF-8, UTF-8 BOM, UTF-16 LE/BE, ASCII, ISO-8859-1 through -15, Windows-1250 through -1258, Shift-JIS, EUC-JP, EUC-KR, GB2312, Big5, KOI8-R, MacRoman

### Encoding & Line Ending Change UX (DEFECT-31 FIX)

> **New in v1.0.** The spec previously only handled detection on open. Users must also be able
> to change encoding and line endings of an already-open file.

- **Status bar controls:** Encoding and line ending shown as clickable labels (e.g., `UTF-8  LF`).
- **Change encoding:** clicking the encoding label opens a popover listing all supported encodings.
  - Selecting a different encoding triggers a confirmation sheet: "Re-interpret file as [Encoding]? Characters not representable will appear as replacement characters. This cannot be undone via undo."
  - On confirm: re-encode the buffer content. Mark document dirty. Auto-save triggers.
  - Lossless round-trips (e.g., ASCII → UTF-8) require no confirmation.
- **Change line endings:** clicking the line ending label shows `LF / CRLF / CR` options.
  - Selection is lossless — applies immediately, marks dirty, auto-saves.
  - No confirmation needed.

### mmap Policy for User Files

- **Files 0-256MB:** No mmap. `FileHandle.read` with 64KB chunks — ~33% slower, fully safe. `mmap` causes unrecoverable SIGBUS on truncated files or network mount disconnect.
- **Files >256MB:** Read-only `MAP_PRIVATE` mmap with 64MB sliding window. SIGBUS handled via Mach exception handler (see §13 large file strategy). This is an acceptable tradeoff: the Mach handler provides graceful degradation (empty line + gutter marker) rather than a crash, and the memory savings are critical for multi-GB files.

---

## 9. SEARCH ENGINE

### Single-Document Search

```swift
struct BufferSearch {
    func searchLiteral(in snapshot: PieceTableSnapshot, query: String,
                       caseSensitive: Bool, wholeWord: Bool) -> [SearchMatch] {
        // Data.range(of:) uses NEON SIMD on arm64
    }

    func searchRegex(in snapshot: PieceTableSnapshot, pattern: String) throws -> [SearchMatch] {
        let regex = try cachedRegex(pattern)  // LRU cache of 50
    }
}
```

### Multi-File Search — Parallel, Capped (DEFECT-22 + DEFECT-23 FIX)

> **DEFECT-22 FIX:** Add a `maxResults` cap (default 10,000 matches). When reached, stop
> adding and show a "Showing first N results — refine your search" banner.
>
> **DEFECT-23 FIX:** For regex patterns containing `\n` or multi-line mode flags, increase
> the overlap window to `max(pattern.utf8.count * 4, 4096)` bytes to handle matches spanning
> chunk boundaries.

```swift
actor SearchService {
    static let maxResults = 10_000

    func searchFiles(in directory: URL, query: SearchQuery,
                     progress: AsyncStream<Progress>.Continuation) async throws -> SearchResult {
        let files = collectFiles(in: directory, ignoringDotGit: true)
        var allMatches: [FileMatch] = []
        var totalMatchCount = 0
        var truncated = false

        let concurrency = min(ProcessInfo.processInfo.activeProcessorCount, 8)

        // DEFECT-23 FIX: overlap window based on pattern type
        let overlapSize: Int = query.isMultiline
            ? max(query.pattern.utf8.count * 4, 4096)
            : query.pattern.utf8.count * 2

        try await withThrowingTaskGroup(of: FileMatch?.self) { group in
            var inFlight = 0
            for file in files {
                // DEFECT-22 FIX: stop dispatching new files once cap is reached
                guard totalMatchCount < Self.maxResults else {
                    truncated = true
                    break
                }
                try Task.checkCancellation()

                group.addTask(priority: .utility) {
                    let handle = try FileHandle(forReadingFrom: file)
                    defer { try? handle.close() }
                    var matches: [SearchMatch] = []
                    var lineOffset = 0
                    var carryOver = Data()

                    while let chunk = try handle.read(upToCount: 65536), !chunk.isEmpty {
                        try Task.checkCancellation()
                        let searchBuffer = carryOver + chunk
                        let chunkMatches = self.search(in: searchBuffer, query: query,
                                                       lineOffset: lineOffset)
                        matches.append(contentsOf: chunkMatches)
                        // GAP-07 FIX: Count newlines in the NEW data only (excluding
                        // carryOver which was already counted in the previous iteration).
                        // Previous code counted raw chunk newlines but searchBuffer
                        // includes overlap — causing double-count at boundaries.
                        let newDataRegion = searchBuffer.dropFirst(carryOver.count)
                        lineOffset += newDataRegion.reduce(0) {
                            $0 + ($1 == UInt8(ascii: "\n") ? 1 : 0)
                        }
                        carryOver = searchBuffer.suffix(overlapSize)
                    }
                    return matches.isEmpty ? nil : FileMatch(url: file, matches: matches)
                }
                inFlight += 1

                if inFlight >= concurrency {
                    if let result = try await group.next() {
                        inFlight -= 1
                        if let match = result {
                            allMatches.append(match)
                            totalMatchCount += match.matches.count
                            progress.yield(.determinate(Double(totalMatchCount), "searching"))
                        }
                    }
                }
            }
            for try await result in group {
                if let match = result { allMatches.append(match) }
            }
        }
        return SearchResult(matches: allMatches, truncated: truncated,
                           truncatedAt: truncated ? Self.maxResults : nil)
    }
}
```

### `.gitignore`-Aware File Collection (DEFECT-24 FIX)

> **DEFECT-24 FIX:** Specify the implementation. Use `libgit2`'s ignore API (already a
> dependency via `GitService.swift`). This avoids adding another dependency.

```swift
/// DEFECT-24 FIX: Use libgit2's git_ignore_path_is_ignored() for .gitignore filtering.
/// libgit2 is already linked for GitService — no new dependency.
func collectFiles(in directory: URL, ignoringDotGit: Bool) -> [URL] {
    var results: [URL] = []
    var repo: OpaquePointer? = nil
    _ = git_repository_open(&repo, directory.path)
    defer { if let r = repo { git_repository_free(r) } }

    let enumerator = FileManager.default.enumerator(
        at: directory,
        includingPropertiesForKeys: [.isRegularFileKey, .isSymbolicLinkKey,
                                     .fileResourceIdentifierKey],
        options: [.skipsHiddenFiles]
    )
    var visitedInodes = Set<Data>()  // symlink loop detection

    while let url = enumerator?.nextObject() as? URL {
        // Symlink loop detection (security — CLAUDE.md §Security)
        if let resID = try? url.resourceValues(forKeys: [.fileResourceIdentifierKey])
                              .fileResourceIdentifier as? Data {
            if visitedInodes.contains(resID) { enumerator?.skipDescendants(); continue }
            visitedInodes.insert(resID)
        }

        if ignoringDotGit && url.pathComponents.contains(".git") {
            enumerator?.skipDescendants(); continue
        }

        // Filter via libgit2 .gitignore rules
        if let r = repo {
            var ignored: Int32 = 0
            git_ignore_path_is_ignored(&ignored, r, url.path)
            if ignored != 0 { continue }
        }

        if (try? url.resourceValues(forKeys: [.isRegularFileKey]))?.isRegularFile == true {
            results.append(url)
        }
    }
    return results
}
```

### Multi-File Replace (DEFECT-30 FIX)

> **New in v1.0.** Multi-file replace was missing from the original spec.

**Invocation:** Find & Replace panel → expand to project scope → "Replace All in Files" button.

**Flow:**
1. Run search (per §9 Multi-File Search). Show preview of all proposed changes.
2. User reviews the diff-style preview. Can deselect individual files or matches.
3. On confirm: for each file in parallel (`.utility` priority):
   a. If the file is currently open: apply changes via `DocumentManager.edit()` (creates an undo group named "Replace in Files"). The tab is marked dirty; auto-save triggers normally.
   b. If the file is NOT open: load into a temporary `PieceTable`, apply changes, save via `FileSaver` (three-tier atomic write). No tab is opened.
4. Show results banner: "Replaced N occurrences in M files."
5. Undo: for open files, `Cmd+Z` undoes the single "Replace in Files" undo group (snapshot-based — O(1) per DEFECT-36 FIX). For unopened files that were written to disk, undo is not available (shown as greyed out in the undo menu with a tooltip: "File was replaced on disk — cannot undo").

**Progress:** Live progress bar with file count and match count. Cancellable via Escape.

### Regex Cache

LRU cache of 50 compiled `NSRegularExpression` objects. Key: `(pattern, options)`. Compilation ~1ms; cached reuse ~0.001ms.

### Incremental In-Buffer Search

As user types in find bar, refine previous results. Narrow: filter existing. Widen (delete): re-search.

---

## 10. SYNTAX ENGINE — TREE-SITTER VIA C INTEROP

### Swift Calls tree-sitter C API Directly (No Rust Wrapper)

```swift
import TreeSitter

final class SyntaxTree {
    let rawTree: OpaquePointer
    let generation: UInt64
    deinit { ts_tree_delete(rawTree) }
}

/// NOT Sendable — ts_parser* is not thread-safe.
/// One SyntaxEngine instance per document, owned by DocumentManager actor.
final class SyntaxEngine {
    private var parser: OpaquePointer
    init() { parser = ts_parser_new() }
    deinit { ts_parser_delete(parser) }

    func setLanguage(_ language: OpaquePointer) {
        ts_parser_set_language(parser, language)
    }

    func parse(source: PieceTableSnapshot, oldTree: SyntaxTree?) -> SyntaxTree? {
        let retainedSource = Unmanaged<PieceTableSnapshot>.passRetained(source)
        defer { retainedSource.release() }

        var input = TSInput(
            payload: retainedSource.toOpaque(),
            read: { payload, byteOffset, _, bytesRead in
                let snap = Unmanaged<PieceTableSnapshot>.fromOpaque(payload!)
                    .takeUnretainedValue()
                return snap.readUTF8(at: Int(byteOffset), length: bytesRead)
            },
            encoding: TSInputEncodingUTF8
        )
        guard let rawTree = ts_parser_parse(parser, oldTree?.rawTree, input) else {
            os_log(.error, "ts_parser_parse returned NULL")
            return nil  // Caller renders plain text — graceful degradation
        }
        return SyntaxTree(rawTree: rawTree, generation: source.generation)
    }
}
```

### Incremental Parsing — `ts_tree_edit` Mapping (DEFECT-25 FIX)

> **DEFECT-25 FIX:** Specify exactly how a piece table edit maps to `TSInputEdit`.
> An incorrect mapping silently corrupts the incremental parse tree.

```swift
/// DEFECT-25 FIX: Convert a PieceTable edit operation into a TSInputEdit for tree-sitter.
struct EditMapping {
    /// Map a single piece-table edit (range replaced with new text) to TSInputEdit.
    ///
    /// - Parameters:
    ///   - startByte:     byte offset of the edit start in the PREVIOUS snapshot
    ///   - oldEndByte:    byte offset of the edit end in the PREVIOUS snapshot (exclusive)
    ///   - newText:       the replacement text (UTF-8 bytes)
    ///   - lineIndex:     the piece table's line index (for row/col computation)
    static func makeInputEdit(startByte: Int, oldEndByte: Int,
                              newText: Data, lineIndex: LineIndex) -> TSInputEdit {
        let startPoint = lineIndex.rowCol(forByteOffset: startByte)
        let oldEndPoint = lineIndex.rowCol(forByteOffset: oldEndByte)
        let newEndByte = startByte + newText.count
        // New end point is in the UPDATED buffer — count newlines in newText:
        let newlines = newText.filter { $0 == UInt8(ascii: "\n") }.count
        let startRow = startPoint.row
        let newEndRow = startRow + newlines
        let newEndCol: Int = newlines == 0
            ? startPoint.column + newText.count
            : newText.reversed().prefix(while: { $0 != UInt8(ascii: "\n") }).count

        return TSInputEdit(
            start_byte: UInt32(startByte),
            old_end_byte: UInt32(oldEndByte),
            new_end_byte: UInt32(newEndByte),
            start_point: TSPoint(row: UInt32(startRow), column: UInt32(startPoint.column)),
            old_end_point: TSPoint(row: UInt32(oldEndPoint.row), column: UInt32(oldEndPoint.column)),
            new_end_point: TSPoint(row: UInt32(newEndRow), column: UInt32(newEndCol))
        )
    }

    /// For multi-cursor edits: apply ts_tree_edit in REVERSE byte order.
    /// Applying in forward order would invalidate the byte offsets of later edits.
    static func applyMultiCursorEdits(_ edits: [EditOperation],
                                      to tree: OpaquePointer,
                                      lineIndex: LineIndex) {
        for edit in edits.sorted(by: { $0.startByte > $1.startByte }) {
            var tsEdit = makeInputEdit(startByte: edit.startByte,
                                       oldEndByte: edit.oldEndByte,
                                       newText: edit.newText,
                                       lineIndex: lineIndex)
            ts_tree_edit(tree, &tsEdit)
        }
    }
}
```

### Language Injection (Embedded Grammars) — DEFECT-26 FIX

> **DEFECT-26 FIX:** Specify the injection strategy: use tree-sitter's standard
> `injections.scm` query files. Injected highlights override parent highlights within their
> byte range. Handle unloaded grammars gracefully.

```swift
/// DEFECT-26 FIX: Language injection engine.
final class InjectionEngine {

    /// Find all injection ranges in a parsed tree using the language's injections.scm query.
    /// Returns: array of (byteRange, languageName) pairs.
    func findInjections(in tree: SyntaxTree, source: PieceTableSnapshot,
                        parentLanguage: String) -> [(Range<Int>, String)] {
        guard let injectionQuery = LanguageRegistry.shared
                .injectionQuery(for: parentLanguage) else { return [] }
        // Run the injection query against the tree to identify ranges
        // (e.g., <script> content → "javascript", <style> → "css")
        return runInjectionQuery(injectionQuery, tree: tree, source: source)
    }

    /// Parse each injection range with its grammar, merge highlight results.
    ///
    /// Highlight priority: injected grammar highlights OVERRIDE parent highlights
    /// within the injection's byte range.
    ///
    /// Unloaded grammar: if the injected grammar is not yet in memory,
    /// emit a `.pendingInjection(byteRange, languageName)` event and return
    /// the parent's plain-token highlights for that range. A separate Task
    /// will load the grammar and re-parse asynchronously (task-replacement pattern).
    func mergeHighlights(parent: [HighlightSpan],
                         injections: [(Range<Int>, String)],
                         source: PieceTableSnapshot) -> [HighlightSpan] {
        var result = parent
        for (range, lang) in injections {
            if let grammar = LanguageRegistry.shared.grammar(for: lang) {
                let injectedTree = SyntaxEngine().parse(for: lang,
                                                        range: range, source: source,
                                                        grammar: grammar)
                let injectedSpans = highlight(tree: injectedTree, language: lang)
                // Replace parent spans in this range with injected spans
                result = result.filter { !range.overlaps($0.byteRange) } + injectedSpans
            }
            // If grammar not loaded: parent plain-token highlight stays for this range.
            // Grammar load queued → re-parse will fire when ready.
        }
        return result.sorted { $0.byteRange.lowerBound < $1.byteRange.lowerBound }
    }
}
```

### Incremental Parsing Summary

- After edit: apply `TSInputEdit` (per DEFECT-25 mapping), then call `parse(source:oldTree:)`.
- 30ms debounce after edit (task replacement pattern — DEFECT-09 FIX).
- Stale result check: compare `syntaxTree.generation` with current piece table generation — discard if stale.
- 50+ language grammars: precompiled `.c` files, lazy-loaded, each grammar function called via Swift C interop.
- Highlight queries: `.scm` files from app bundle, mapped to semantic tokens.

---

## 11. ERROR HANDLING

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
    case cloudCoordinationTimeout(path: String)  // DEFECT-15

    /// GAP-23 FIX: Every case has a specific localized message. No catch-all `default`.
    var errorDescription: String? {
        switch self {
        case .io(let e):
            return String(format: NSLocalizedString("error.io", comment: ""), e.localizedDescription)
        case .encoding(let desc):
            return String(format: NSLocalizedString("error.encoding", comment: ""), desc)
        case .binaryFile(let name):
            return String(format: NSLocalizedString("error.binary_file", comment: ""), name)
        case .syntaxParsing(let lang):
            return String(format: NSLocalizedString("error.syntax_parsing", comment: ""), lang)
        case .diff(let desc):
            return String(format: NSLocalizedString("error.diff", comment: ""), desc)
        case .macro(let desc):
            return String(format: NSLocalizedString("error.macro", comment: ""), desc)
        case .plugin(let desc):
            return String(format: NSLocalizedString("error.plugin", comment: ""), desc)
        case .invalidArgument(let desc):
            return String(format: NSLocalizedString("error.invalid_argument", comment: ""), desc)
        case .sessionDB(let e):
            return String(format: NSLocalizedString("error.session_db", comment: ""), e.localizedDescription)
        case .searchRegex(let e):
            return String(format: NSLocalizedString("error.search_regex", comment: ""), e.localizedDescription)
        case .cloudCoordinationTimeout(let p):
            return String(format: NSLocalizedString("error.cloud_timeout", comment: ""), p)
        }
    }
}
```

### Error Presentation

| Severity | Presentation |
|----------|-------------|
| Fatal (can't open DB) | `NSAlert` + "Report Issue" button |
| Blocking (can't save) | Inline banner above editor with retry |
| Informational (binary file) | Sheet dialog with options |
| Transient (cloud timeout) | Status bar, auto-dismiss 5s |

**Rules:** No `try!` or force unwraps. No `fatalError()` except unreachable paths. Errors cross boundaries as `EditorError`. User text is always actionable.

---

## 12. ALL 65 FEATURES + NEW IN v1.0

All features from the original specification are preserved. New additions from this revision:

**New in this spec revision (v1.1 of spec):**
- §5 Spell checking (DEFECT-33 FIX)
- §5 Word wrap specification (DEFECT-32 FIX)
- §8 Encoding/line-ending change UX (DEFECT-31 FIX)
- §9 Multi-file Replace (DEFECT-30 FIX)
- §3 Undo-of-replace-all via snapshot restore (DEFECT-36 FIX — see §Undo below)

### Undo / Redo — Snapshot-Based (DEFECT-36 FIX)

> **DEFECT-36 FIX:** "Replace All" (single-file and multi-file) must be a single undoable
> operation. Undoing 50,000 piece-table operations individually is unusable.
>
> **Fix:** Replace-all is backed by a **pre-edit snapshot**. The undo stack entry stores the
> full `PieceTableSnapshot` taken before the replace. Undo is O(1) — swap the pointer back.

```swift
enum UndoEntry {
    case singleEdit(before: PieceTableSnapshot, after: PieceTableSnapshot)
    case groupedEdits(name: String, before: PieceTableSnapshot, after: PieceTableSnapshot)
    // groupedEdits is used for: replace-all, macro replay, bulk indent, multi-cursor batch
}

// In DocumentManager.replaceAll(query:replacement:):
let beforeSnapshot = await snapshot()     // O(1) COW copy
performAllReplacements()                  // applies N changes to the live piece table
let afterSnapshot = await snapshot()
undoStack.push(.groupedEdits(name: "Replace All", before: beforeSnapshot, after: afterSnapshot))
// Undo: restore beforeSnapshot via actor-isolated swap. O(1).
```

### Tab Drag-and-Drop Reordering (GAP-17 FIX)

> **GAP-17 FIX:** Schema has `sort_order` but tab reordering was unspecified.

- Tabs support **drag-and-drop** reordering within the same window.
- Drag between windows: moves the tab (and its document reference) to the target window.
- `sort_order` updated on drop for all affected tabs in a single SQLite transaction.
- Animated insertion marker shown during drag (follows cursor).
- Keyboard: `Cmd+Shift+[` and `Cmd+Shift+]` move active tab left/right.
- Drop on empty area of tab bar (or past the last tab): appends to end.

### Recent Files (GAP-19 FIX)

> **GAP-19 FIX:** Standard text editor feature was missing.

- `File > Open Recent` submenu shows last 20 opened files.
- Storage: `UserDefaults` array of security-scoped bookmark `Data` (sandbox-safe).
- Files that no longer exist shown greyed with "(missing)" suffix; selecting them shows "File not found" alert.
- `File > Open Recent > Clear Menu` clears the list.
- Opening a file moves it to the top of the recents list.

### Fullscreen Mode (GAP-20 FIX)

> **GAP-20 FIX:** Schema had `is_fullscreen` but behavior was unspecified.

- Enter/exit via green traffic light button or `Ctrl+Cmd+F`.
- Native macOS fullscreen (not custom — uses `NSWindow.toggleFullScreen()`).
- Toolbar auto-hides (mouse to top edge to reveal).
- Tab bar remains visible at top.
- Status bar remains visible at bottom.
- Sidebar toggleable via `Cmd+B`.
- `is_fullscreen` persisted per window in `sessions.db`. Restored on relaunch.

### Completion Engine (GAP-21 FIX)

> **GAP-21 FIX:** Architecture listed `CompletionEngine` but it was never specified.

- **Data source (v1.0):** Word-based completion from current buffer + all open buffers. No LSP.
- **Trigger:** Automatic after 3+ characters typed. Also invocable via `Esc` or `Ctrl+Space`.
- **Ranking:** (1) Distance from cursor (closest first), (2) Frequency in current buffer, (3) Alphabetical.
- **Popup:** `CompletionPopupView` (NSPanel, `.nonactivatingPanel` style). Max 10 visible items, scrollable.
- **Selection:** Arrow keys navigate, `Tab` or `Enter` inserts, `Esc` dismisses.
- **Performance:** <50ms via detached task (task replacement pattern). Only scans visible buffer + open buffer word indices (pre-built on file open, updated incrementally on edit).
- **Dismiss:** On cursor movement away, typing a non-word character, or Escape.

### Snippet Manager (GAP-18 FIX)

> **GAP-18 FIX:** `SnippetManager` was in module structure but unspecified.

- **Trigger:** Tab key after a prefix match. If no snippet matches, Tab inserts normal indent.
- **Format:** JSON files in `~/Library/Application Support/mynotepadpp/snippets/`.
- **Fields:** `prefix` (trigger text), `body` (template with `$1`, `$2` tab stops and `$0` final cursor), `scope` (grammar name or `"*"` for all).
- **Tab stops:** `Tab` advances to next stop, `Shift+Tab` goes back. `Esc` exits snippet mode, placing cursor at `$0`.
- **Built-in snippets:** None in v1.0. User-created only.
- **v1.1:** `MacroService` — record/replay keystroke sequences. Stored as serialized `[EditOperation]` arrays. `Cmd+Shift+R` to start/stop recording. `Cmd+Shift+P` to replay.

### Undo Stack Limits (GAP-10 FIX)

> **GAP-10 FIX:** Without a limit, snapshot-based undo entries consume unbounded memory
> (each entry stores a full PieceTree COW copy + addBytesLength reference).

| Limit | Value |
|-------|-------|
| Max undo depth (active tab) | 1,000 entries |
| Max undo memory (active tab) | 100MB estimated snapshot size |
| Eviction | Oldest entries evicted first (FIFO) |
| Non-active tabs | Undo evicted at 250MB RSS (per §13 memory budget) |
| UI | Evicted entries removed from Edit > Undo menu. Status bar shows "Undo history trimmed" once on eviction. |

```swift
struct UndoStack {
    private var entries: [UndoEntry] = []
    private let maxDepth = 1000
    private let maxEstimatedBytes = 100 * 1024 * 1024  // 100MB

    mutating func push(_ entry: UndoEntry) {
        entries.append(entry)
        // Evict oldest if over depth limit
        while entries.count > maxDepth { entries.removeFirst() }
        // Evict oldest if over memory limit
        while estimatedSize > maxEstimatedBytes && entries.count > 1 {
            entries.removeFirst()
        }
    }
}
```

---

## 13. PERFORMANCE TARGETS

| Metric | Target | How (Swift) |
|--------|--------|-------------|
| Cold startup | <500ms | Lazy grammar loading, async session read |
| Open 1MB file | <200ms | BufferBuilder 64KB chunks |
| Open 100MB file | <2s (first screen <200ms) | Progressive: first screenful from header |
| Open 1GB file | <10s (first screen <200ms) | Streaming BufferBuilder |
| Keystroke latency | <16ms | Actor edit → snapshot → dirty rect render |
| Scroll FPS | 60 FPS on M4 | Metal triple-buffering + overdraw buffer |
| Search 10K files (literal) | <2s (warm cache) | TaskGroup + Data.range(of:) SIMD |
| Search 10K files (regex) | <5s | NSRegularExpression + TaskGroup |
| Autocomplete popup | <50ms | Buffer word scan, detached task |
| Memory idle (1 file) | <75MB | No Rust stdlib; GRDB + Metal + tree-sitter loaded |
| Memory (50 tabs) | <300MB | Runtime RSS monitoring + eviction |
| Auto-save (<100KB) | <50ms background | Snapshot → atomic write |
| Hot exit (50 tabs) | <500ms | Single SQLite transaction + zstd |
| Tab switch | <30ms | Session state read + snapshot swap |
| Syntax highlight after edit | <50ms viewport | tree-sitter incremental + task replacement |
| File watcher response | <500ms | FSEvents 200ms debounce |

> **Note on search target:** "Search 10K files (literal) <2s" assumes warm OS file cache.
> On cold cache, 10K file opens will take 1-5s more due to disk I/O. This is expected and
> acceptable. The progress bar shows live results as they arrive.

> **Note on memory idle:** Previous target was <50MB which is unrealistic with Metal, GRDB,
> and 50+ tree-sitter grammars lazily loaded. Revised to <75MB. (DEFECT-33 FIX also adds
> NSSpellChecker which has a small footprint.)

### Zero-Hang Guarantees

| Scenario | Guarantee |
|----------|-----------|
| `Cmd+Q` | Terminate within 3s. No dialog. No hang. |
| `Cmd+W` | Close within 200ms (drain-and-save). No dialog. |
| System shutdown | Save + terminate within 5s. |
| SIGKILL | Max 500ms data loss (continuous backup). |
| Network FS stall | File ops timeout at 30s. UI never freezes. |
| Opening 1GB file | UI responsive immediately. Loading async. |
| Find in 100K files | Cancellable. UI responsive. Capped at 10,000 results. |
| Disk full | Non-blocking banner. Buffer in memory. |
| iCloud coordinator hang | 10s timeout → fall to Tier 2. No I/O pool saturation. |

### Runtime Memory Budget Enforcement

> **GAP-11 FIX:** The 350MB cap and "open 1GB file" targets were contradictory.
> Resolution: the RSS budget excludes file content buffers (`OriginalBuffer` + `AddBufferStore`).
> The budget governs caches, undo history, Metal textures, and service overhead only.
> For files >256MB, a virtual paging strategy is used (see below).

- Check RSS every 10s via `mach_task_info` (~1µs per call).
- **RSS budget excludes** `OriginalBuffer` and `AddBufferStore` byte counts. These are tracked separately as "content memory."
- At 250MB (non-content RSS): evict undo history for non-active tabs.
- At 300MB: aggressive purge (`LineLayoutCache.invalidateAll()` for hidden tabs, glyph atlas page eviction).
- At 350MB: status bar warning, refuse new file opens until RSS drops.

**Large file strategy (files >256MB):**
- Files >256MB use **memory-mapped read-only windows** for the `OriginalBuffer` (64MB sliding window via `mmap` with `MAP_PRIVATE`). SIGBUS is caught by installing a Mach exception handler that unmaps the faulting page and returns an empty buffer (graceful degradation — the line shows as empty with a "read error" gutter marker). This is safe because the mmap is read-only and private (not shared).
- Files 0-256MB: loaded fully into memory as before (no mmap).
- The `AddBufferStore` is always fully in-memory (edits are typically <1% of file size).

---

## 14. SECURITY

### App Sandbox + Hardened Runtime

```xml
<dict>
    <key>com.apple.security.app-sandbox</key><true/>
    <key>com.apple.security.files.user-selected.read-write</key><true/>
    <key>com.apple.security.files.bookmarks.app-scope</key><true/>
    <!-- v1.0: com.apple.security.print (basic print support — DEFECT-34 FIX) -->
    <key>com.apple.security.print</key><true/>
    <!-- v1.1: com.apple.security.network.client (SFTP) -->
</dict>
```

### Print Support (DEFECT-34 FIX)

> **DEFECT-34 FIX:** Print is included in v1.0 to pass App Store review. A basic
> `NSPrintOperation` with the document content is sufficient for v1.0.

- `Cmd+P` opens macOS system print dialog.
- v1.0: print as plain text (current encoding preserved). Font size matches editor font.
- v1.1: syntax-highlighted print output via attributed string rendering.
- `com.apple.security.print` entitlement added (see above).

### .git Directory Protection (DEFECT-35 FIX)

> **DEFECT-35 FIX:** Disabling auto-save for .git files is too aggressive. Developers
> legitimately edit `.git/hooks/pre-commit`, `.git/config`, etc. Replace with a
> non-blocking informational banner only.

- Opening any file inside a `.git/` directory shows a **one-time dismissible info banner** (not a blocking dialog): "This file is inside a .git directory — edits may affect your git repository."
- Auto-save remains **enabled** for .git files. The user chose to open the file.
- Auto-save is disabled only for files in `.git/objects/` (binary git objects — binary detection fires anyway).
- The banner is shown at most once per app session per `.git/` parent directory.

### Sandbox Container Paths

All internal paths via `FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask)`. Never hardcode `~/Library/Application Support/mynotepadpp/`.

### Path Sanitization

Canonicalize all file paths. Reject `..` traversal. Verify within expected directory. Applies to: theme loading, recovery files, URL scheme handler, temp files.

### URL Scheme Handler

`mynotepadpp://open?file=...` — confirmation dialog required. Reject system dirs. Rate-limit 1/second.

### BiDi Attack Protection

Warn on bidirectional override characters (U+202A-U+202E, U+2066-U+2069) **in files with a recognized source-code grammar only** (not plain text or Arabic/Hebrew documents where these characters are legitimate).

### Clipboard Access (macOS Sequoia)

Read clipboard only on `Cmd+V`, `Cmd+E`. Never poll proactively.

### Symlink Loop Detection

Track visited inodes during directory enumeration. Stop recursion on duplicate inode (see `collectFiles` in §9).

### Security-Scoped Bookmark Limit

Track active count. Warn at 200, banner at 240, fall back to `NSOpenPanel` at 250.

### Recovery File Permissions

Set `0600` on all recovery, backup, and SQLite files immediately after creation.

### PrivacyInfo.xcprivacy

```xml
<dict>
    <key>NSPrivacyTracking</key><false/>
    <key>NSPrivacyCollectedDataTypes</key><array/>
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <dict><key>NSPrivacyAccessedAPIType</key>
              <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
              <key>NSPrivacyAccessedAPITypeReasons</key><array><string>C617.1</string></array></dict>
        <dict><key>NSPrivacyAccessedAPIType</key>
              <string>NSPrivacyAccessedAPICategoryDiskSpace</string>
              <key>NSPrivacyAccessedAPITypeReasons</key><array><string>E174.1</string></array></dict>
        <dict><key>NSPrivacyAccessedAPIType</key>
              <string>NSPrivacyAccessedAPICategorySystemBootTime</string>
              <key>NSPrivacyAccessedAPITypeReasons</key><array><string>35F9.1</string></array></dict>
        <dict><key>NSPrivacyAccessedAPIType</key>
              <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
              <key>NSPrivacyAccessedAPITypeReasons</key><array><string>CA92.1</string></array></dict>
    </array>
</dict>
```

---

## 15. SHUTDOWN & CLOSE — NEVER PROMPTS, NEVER LOSES DATA

### Close Tab (`Cmd+W`) (DEFECT-14 FIX)

> **DEFECT-14 FIX:** Tab close must drain any in-flight save and do a final save.
> The previous "save silently → close immediately" was wrong — if a save was in-flight,
> the final save was silently skipped, losing the most recent edits.

```swift
// In TabViewController or EditorViewModel (@MainActor):
@MainActor
func closeTab(_ tab: Tab) async {
    let doc = tab.document
    // Capture @MainActor properties into Sendable values BEFORE crossing actor boundaries.
    let docID = doc.id              // nonisolated let — always safe
    let filePath = doc.filePath?.path  // read on MainActor, pass as String
    let isDirty = doc.isDirty       // read on MainActor, pass as Bool

    // 1. Cancel debounce/throttle timers for this doc.
    await autoSaveService.stopTracking(docID: docID)

    // 2. DEFECT-14 FIX + GAP-02 FIX: drain in-flight save with 2s timeout.
    if let path = filePath {
        await autoSaveService.drainAndSave(
            filePath: path, isDirty: isDirty,
            saveBlock: { try await DocumentManager.shared.saveDocument(docID: docID) }
        )
    }

    // 3. For untitled tabs: write to SQLite before deleting backup (DEFECT-17 FIX).
    if filePath == nil && isDirty {
        await sessionService.saveUnsavedContent(docID: docID)
    }

    // 4. Delete continuous backup (safe now — either saved to disk or to SQLite).
    await backupService.deleteBackup(docID: docID)

    // 5. Remove tab from UI (already on MainActor).
    tabBar.removeTab(tab)
}
```

**Timing:** Tab close now takes up to ~200ms (drain + final save) instead of <50ms. This is acceptable and still fast enough to feel instant to users.

### Close Last Window

Save session state to SQLite synchronously on window close. App stays in Dock. Click Dock → restore via `applicationShouldHandleReopen`.

### Quit (`Cmd+Q`) — DEFECT-27 + DEFECT-28 FIX

> **DEFECT-27 FIX:** Store `hotExitTask` on `AppState.shared` (a global singleton), NOT on
> `AppDelegate`. AppDelegate can be deallocated before the task completes in some macOS
> shutdown paths; the global singleton has process lifetime.
>
> **DEFECT-28 FIX:** Before starting the hot-exit task, commit a minimal "emergency session"
> to SQLite synchronously — just tab IDs and file paths, no undo history, <50ms. This
> guarantees the tab list survives even if the 3s watchdog fires before full hot exit.

```swift
@MainActor func applicationShouldTerminate(_ sender: NSApplication) -> NSApplication.TerminateReply {

    // DEFECT-28 FIX: Commit minimal emergency session BEFORE the async hot exit.
    // This is synchronous and completes in <50ms — always survives the watchdog.
    SessionService.shared.commitEmergencySession()  // sync: just tab IDs + file paths

    let hotExitCompleted = OSAllocatedUnfairLock(initialState: false)

    // DEFECT-27 FIX: Store task on AppState.shared — process-lifetime singleton.
    // AppDelegate may be released before the task completes; AppState.shared will not.
    AppState.shared.hotExitTask = Task.detached(priority: .userInitiated) {
        // DEFECT-07 FIX: beginShutdown() is synchronous — cancels tasks, does NOT await them.
        AppState.shared.beginShutdown()
        await SessionService.shared.hotExit()     // <500ms — full state + dirty_flag=0
        await SessionService.shared.cleanShutdown()  // best-effort; watchdog may cut this
        hotExitCompleted.withLock { $0 = true }
        await MainActor.run { sender.reply(toApplicationShouldTerminate: true) }
    }

    // 3-second watchdog — RunLoop timer bypasses cooperative thread pool.
    // SOLE permitted use of non-async timer in the codebase.
    Timer.scheduledTimer(withTimeInterval: 3.0, repeats: false) { _ in
        let completed = hotExitCompleted.withLock { $0 }
        if !completed {
            // Watchdog fired. Emergency session already committed (DEFECT-28 FIX).
            // Tab list is safe. Undo history and cursor positions may be lost.
            sender.reply(toApplicationShouldTerminate: true)
        }
    }

    return .terminateLater
}
```

### SIGTERM (System Shutdown)

Register `NSApplicationWillTerminate`. Same hot-exit sequence. 5-second budget.

### SIGKILL (Force Quit)

Cannot catch. Continuous backup ensures max 500ms data loss. SQLite WAL auto-recovers.

### APPENDIX NOTE: iOS Lifecycle (GAP-16 FIX)

> **GAP-16 FIX:** This section is moved to Appendix A. It is NOT part of the macOS v1.0 scope.
> Retained as a design reference for future iOS port only. See **Appendix A** at end of document.

---

## 16. ACCESSIBILITY

- VoiceOver: all UI elements labeled; editor navigable by line/word/character
- Keyboard-only operation: every feature reachable without mouse
- High contrast theme (WCAG AAA 7:1)
- Color-blind theme (deuteranopia/protanopia safe; uses icons + color, not color alone)
- Reduced motion: respect `isReduceMotionEnabled` — disable smooth scroll, cursor blink, minimap animation
- Live regions for status bar announcements (encoding, language, save status)
- Diff hunks as `NSAccessibilityGroup` with descriptive labels
- Spell-check underlines announced as "possible spelling error" by VoiceOver

---

## 17. DISTRIBUTION

- Mac App Store + Homebrew Cask + GitHub Releases (.dmg)
- GPL v3 + Section 7 App Store exception
- CLA required for contributors
- Hardened Runtime + notarization
- Privacy policy URL (required even with zero data collection)
- Source code at git tag matches shipped binary (GPL v3 §6 compliance)

---

## 18. DEPENDENCIES (Swift Package Manager)

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/groue/GRDB.swift", from: "7.0.0"),
    .package(url: "https://github.com/apple/swift-collections", from: "1.1.0"),
    .package(url: "https://github.com/facebook/zstd", from: "1.5.0"),
    // DEFECT-06 FIX: genuine lock-free atomics for isShuttingDown flag
    .package(url: "https://github.com/apple/swift-atomics", from: "1.2.0"),
]
// tree-sitter: bundled as C source in TreeSitter/ (needs custom module map)
// libgit2: bundled via SwiftGit2 SPM wrapper (GPL-v2 + linking exception)
//   Used by: GitService (git status/diff/blame) + collectFiles .gitignore filtering (DEFECT-24 FIX)
// No other dependencies. Foundation, AppKit, Metal, CoreText, Security — system frameworks.
```

**Dependency count: 5** (GRDB, swift-collections, zstd, swift-atomics, libgit2 + tree-sitter C source)

---

## 19. BUILD & CI

### Build Command

```bash
xcodebuild -scheme MyNotepadPP -configuration Release -destination 'platform=macOS'
# No cargo build. No cbindgen. No bridging header generation.
```

### CI (GitHub Actions)

```yaml
jobs:
  build-and-test:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - run: xcodebuild test -scheme MyNotepadPP -destination 'platform=macOS'
      - run: xcodebuild test -scheme MyNotepadPPUITests -destination 'platform=macOS'
      - run: swiftlint lint --strict MyNotepadPP/
      # No cargo fmt / clippy / audit — no Rust (DEFECT-37 FIX)
```

### Xcode Configuration

| Setting | Value |
|---------|-------|
| Deployment Target | macOS 14.0 |
| Swift Language Version | 6.0 |
| Strict Concurrency | `complete` |
| Default Actor Isolation | `nonisolated` (for C interop — prevents @MainActor on FFI functions) |
| Architectures | arm64 (dev), arm64 + x86_64 (release universal) |
| Hardened Runtime | Enabled |
| App Sandbox | Enabled |

---

## 20. WHAT CHANGES IF WE EVER WANT CROSS-PLATFORM

| Option | Cost | Time |
|--------|------|------|
| Extract Core Engine to Swift package; rewrite UI per platform | Medium — Swift core works on Linux natively; iOS reuses it | 2-3 months per platform |
| Rewrite core in Rust, keep Swift UI | High — redo all engine work | 3-4 months |
| Port entire app to Kotlin Multiplatform | Very high — rewrite everything | 6+ months |

**The pure-Swift core IS portable to iOS (SwiftUI) and Linux (Swift-on-Linux + GTK bindings) without a full rewrite.** Windows and Android require a C/C++ or Kotlin bridge.

---

## APPENDIX: DEFECT FIX INDEX

All 37 defects from the original validation + 25 gap fixes are resolved in this document:

| Defect | Section | Fix Summary |
|--------|---------|-------------|
| DEFECT-01 | §3, §8 | `OriginalBuffer` immutability preserved via `BufferBuilder` during load |
| DEFECT-02 | §3 | `PieceTree` = `RedBlackTree<Piece>` (index-based), defined in `RedBlackTree.swift` |
| DEFECT-03 | §3 | Snapshot age limit unified to **1 second** everywhere |
| DEFECT-04 | §3 | Compaction background thread builds; actor does the swap via `applyCompaction()` |
| DEFECT-05 | §3 | Starvation-guard 4th parse: apply best-effort + immediately queue fresh parse |
| DEFECT-06 | §4 | `ManagedAtomic<Bool>` (swift-atomics) replaces `OSAllocatedUnfairLock` for shutdown flag |
| DEFECT-07 | §4, §15 | `beginShutdown()` is synchronous — cancels tasks, does NOT await completion |
| DEFECT-08 | §4 | GRDB `DatabaseQueue` replaces undefined "custom SerialExecutor" |
| DEFECT-09 | §4 | Task replacement pattern: 1 task per category per document, not 1 per keystroke |
| DEFECT-10 | §5 | Metal triple-buffering semaphore (`DispatchSemaphore(value: 3)`) specified |
| DEFECT-11 | §5 | Minimap double-buffer uses `OSAllocatedUnfairLock<MTLBuffer?>` (Swift-idiomatic) |
| DEFECT-12 | §5 | `characterIndex(for:)` caches glyph positions; binary-search per call |
| DEFECT-13 | §5 | `LineLayoutCache` — invalidate per-line; bulk-evict on tab close |
| DEFECT-14 | §6, §15 | `drainAndSave()` for tab-close: waits for in-flight save + final dirty check |
| DEFECT-15 | §6 | `NSFileCoordinator` wrapped with 10s timeout; falls to Tier 2 on timeout |
| DEFECT-16 | §6 | Cloud detection via `volumeIsLocalKey` + iCloud path + user-configurable list |
| DEFECT-17 | §6 | Untitled tab close: writes to SQLite first, then deletes backup |
| DEFECT-18 | §6 | `focusLossTasks` pruned by 30s background timer instead of on next focus-loss |
| DEFECT-19 | §7 | `dirty_flag` uses `INSERT OR REPLACE` upsert — safe on first launch |
| DEFECT-20 | §7 | VACUUM backup guarded by `vacuumInProgress` flag; writes to staging then renames |
| DEFECT-21 | §7 | `CREATE INDEX idx_tabs_window_sort ON tabs(window_id, sort_order)` added |
| DEFECT-22 | §9 | Search result cap: 10,000 matches max; "truncated" flag in result |
| DEFECT-23 | §9 | Multi-line regex: overlap window = `max(pattern.utf8.count * 4, 4096)` |
| DEFECT-24 | §9 | `.gitignore` filtering via `libgit2`'s `git_ignore_path_is_ignored()` |
| DEFECT-25 | §10 | `EditMapping` struct specifies exact `TSInputEdit` field computation |
| DEFECT-26 | §10 | `InjectionEngine` specifies injection query, highlight merge, unloaded grammar fallback |
| DEFECT-27 | §15 | `hotExitTask` stored on `AppState.shared` (process lifetime), not `AppDelegate` |
| DEFECT-28 | §15 | Emergency session (tab IDs + paths) committed synchronously before async hot exit |
| DEFECT-29 | §15 | iOS: save only on `sceneWillEnterBackground`; `resignActive` only pauses timers |
| DEFECT-30 | §9 | Multi-file Replace specified: preview, per-file undo groups, progress, cancel |
| DEFECT-31 | §8 | Encoding/line-ending change UX: status bar controls, confirmation for lossy changes |
| DEFECT-32 | §5 | Word wrap: toggle, soft vs logical lines, line numbers, minimap, cursor movement |
| DEFECT-33 | §5 | Spell checking via `NSSpellChecker`, enabled for plain text, Metal overlay pass |
| DEFECT-34 | §14 | Print (`com.apple.security.print`) added to v1.0 entitlements; basic `NSPrintOperation` |
| DEFECT-35 | §14 | `.git` auto-save enabled; replaced blocking warning with one-time info banner |
| DEFECT-36 | §12 | Replace-all undo: single `PieceTableSnapshot` swap — O(1), snapshot-based |
| DEFECT-37 | Header | Document authority clarified; CLAUDE.md Rust/FFI instructions do not apply here |

---

## APPENDIX A: iOS Lifecycle Notes (GAP-16 — future reference only)

> **NOT part of macOS v1.0 scope.** Retained for future iOS port design reference.

DEFECT-29: `sceneWillResignActive` fires on every interruption (phone calls, notifications).
Saving synchronously on every notification is battery-draining.

**Fix:** Save synchronously only on `sceneWillEnterBackground` and `sceneWillDisconnect`.
On `resignActive`, only pause timers — no disk I/O.

```swift
func sceneWillResignActive(_ scene: UIScene) {
    autoSaveService.pauseTimers()
    syntaxTask?.cancel()
}
func sceneDidBecomeActive(_ scene: UIScene) {
    autoSaveService.resumeTimers()
}
func sceneWillEnterBackground(_ scene: UIScene) {
    let task = UIApplication.shared.beginBackgroundTask(withName: "save-on-background") {}
    Task {
        await withTimeout(seconds: 2) { await autoSaveService.saveAllDirtyDocuments() }
        UIApplication.shared.endBackgroundTask(task)
    }
}
func sceneWillDisconnect(_ scene: UIScene) {
    Task { await autoSaveService.saveAllDirtyDocuments() }
}
```

---

## APPENDIX B: GAP FIX INDEX

All 25 gaps identified in the v1.1 → v2.0 audit:

| Gap | Section | Fix Summary |
|-----|---------|-------------|
| GAP-01 | §3 | `Document` type defined: `@MainActor` class + `Sendable` `DocumentSnapshot` projection |
| GAP-02 | §6 | `drainAndSave()` 2-second timeout prevents infinite hang on network FS |
| GAP-03 | §5 | Metal semaphore `wait(timeout: 100ms)` — skip frame rather than hang main thread |
| GAP-04 | §4 | `cancelAllFireAndForget()` cancels tasks directly via lock-protected array — no actor hop |
| GAP-05 | §3 | `BufferBuilder.seal()` uses `NSData(data:)` — no force cast |
| GAP-06 | §8 | Progressive loading transition specified: cursor preserved, typing disabled until full load |
| GAP-07 | §9 | Search `lineOffset` counts newlines in new data only — no overlap double-count |
| GAP-08 | §5 | `LineLayoutCache` uses nested dictionary — O(1) per-line invalidation |
| GAP-09 | §5 | `characterIndex(for:)` uses partitioned `[Int: [GlyphPosition]]` — O(1) + O(log n) |
| GAP-10 | §12 | Undo stack capped at 1,000 entries / 100MB. Oldest evicted first |
| GAP-11 | §13 | RSS budget excludes file content buffers. Files >256MB use 64MB mmap sliding window |
| GAP-12 | §3 | `PieceTableSnapshot` uses shared `AddBufferStore` with generation — O(1) per snapshot |
| GAP-13 | §5 | Minimap `OSAllocatedUnfairLock` rationale documented (vs. `ManagedAtomic` for shutdown) |
| GAP-14 | §6 | External file modification: auto-reload if clean, banner with Reload/Keep/Compare if dirty |
| GAP-15 | §3 | Multi-window: one PieceTable per file path, shared via AsyncStream, per-tab cursor/scroll |
| GAP-16 | §15 | iOS lifecycle code moved to Appendix A — not part of macOS v1.0 scope |
| GAP-17 | §12 | Tab drag-and-drop reordering: within/between windows, sort_order updated, keyboard shortcuts |
| GAP-18 | §12 | SnippetManager: tab-trigger, JSON format, tab stops. MacroService: v1.1 record/replay |
| GAP-19 | §12 | Recent files: File > Open Recent, 20 items, security-scoped bookmarks |
| GAP-20 | §12 | Fullscreen: native `toggleFullScreen`, persisted per window, toolbar auto-hides |
| GAP-21 | §12 | CompletionEngine: word-based, 3-char trigger, ranked by distance/frequency, <50ms |
| GAP-22 | §6 | `focusLossTasks` pruning tracks completed tasks — not just cancelled |
| GAP-23 | §11 | All `EditorError` cases have specific localized messages — no `default` fallthrough |
| GAP-24 | §7 | `PRAGMA optimize = 0x10002` meaning documented inline |
| GAP-25 | Footer | Stale `CLAUDE_CROSSPLATFORM.md` reference removed |

---

*This document is the sole authoritative spec for the macOS pure-Swift implementation. Version 2.0 — final.*
