# Coding Standards — MYNOTEPAD++ (macOS)

> **Authoritative spec:** `docs/FUNCTIONAL_SPECIFICATION_SWIFT.md` (v2.0 — final)
> This file governs the **macOS-only Swift implementation**.

**Stack:** 100% Swift + C interop (tree-sitter only) · macOS 14.0+ · Apple Silicon · Xcode · Swift Package Manager · GPL v3

---

## Architecture overview

```
┌─────────────────────────────────────────────────────────────┐
│                      AppKit / SwiftUI                        │
│  EditorView (NSView + CoreText + Metal)                      │
│  TabBarView · SidebarView · StatusBarView · FindBarView      │
└───────────────────────────┬─────────────────────────────────┘
                            │ Swift calls (no FFI)
┌───────────────────────────▼─────────────────────────────────┐
│                     Core Services (actors)                   │
│  DocumentManager · AutoSaveService · SyntaxService           │
│  SearchService · ThemeManager · SessionService               │
└───────────────────────────┬─────────────────────────────────┘
                            │ Swift struct/class
┌───────────────────────────▼─────────────────────────────────┐
│                   Data Layer                                 │
│  PieceTable (RedBlackTree<Piece>) · LineIndex                │
│  PieceTableSnapshot · GRDB.swift (SQLite) · tree-sitter C   │
└─────────────────────────────────────────────────────────────┘
```

**Build system:** Xcode only. No Cargo, no Makefile, no shell build scripts.

**SPM dependencies (5 packages):**

| Package | Purpose |
|---------|---------|
| `swift-collections` | `OrderedDictionary`, `Deque` |
| `GRDB.swift` | SQLite via `DatabaseQueue` |
| `swift-atomics` | `ManagedAtomic<Bool>` for shutdown flag |
| `swift-syntax` | (future) Swift syntax highlighting |
| tree-sitter C libraries | Syntax parsing (via XCFramework, not SPM) |

---

## BEHAVIORAL PRINCIPLES (apply to every task)

### 1. Think before coding
- State assumptions explicitly. If uncertain about **functionality or design**, ASK before writing code.
- If multiple interpretations exist, present them — do not pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- **Do NOT ask for permission on routine file operations** (creating files, writing code, running commands). Only ask when there is genuine doubt about what to build or how.
- **NEVER ask to run scripts or shell commands** — just execute them. Only ask when you have a genuine question about *what* to do — never about *executing* it.

### 2. Simplicity and surgical changes
- Minimum code that solves the problem. Nothing speculative.
- Do not improve adjacent code, comments, or formatting.
- Match existing style even if you would do it differently.
- Every changed line must trace directly to the user's request.
- Remove only imports/variables YOUR changes made unused.

### 3. Platform parity note
This is a macOS-only project. Cross-platform parity rules from `CLAUDE_CROSSPLATFORM.md` do **not** apply. If a feature touches both macOS and iOS (future), enumerate both before implementing.

### 4. Goal-driven execution
- Transform tasks into verifiable goals before starting.
- For multi-step tasks, state a brief plan with verification checks.
- After implementation, verify the feature works on macOS 14 / Apple Silicon.

---

## MODULES

| Module | Path | Language |
|--------|------|----------|
| App entry | `platforms/macos/Sources/App/` | Swift |
| Views | `platforms/macos/Sources/Views/` | Swift (AppKit + CoreText + Metal) |
| Services | `platforms/macos/Sources/Services/` | Swift actors |
| Data models | `platforms/macos/Sources/Models/` | Swift structs/classes |
| Tree-sitter bridge | `platforms/macos/Sources/Syntax/` | Swift + C interop |
| Resources | `platforms/macos/Resources/` | `.strings`, `.xcassets`, `.metallib`, themes |
| Unit tests | `platforms/macos/Tests/` | XCTest |
| UI tests | `platforms/macos/UITests/` | XCUITest |

---

## CANONICAL REFERENCES

Pick the closest shape; copy its structure; diff your output against it. Structural divergence is a violation.

| Shape | Reference file | Use when |
|-------|---------------|----------|
| AppKit view | `platforms/macos/Sources/Views/EditorView.swift` | New `NSView` subclass |
| Actor service | `platforms/macos/Sources/Services/FileService.swift` | New actor-based service |
| Data model | `platforms/macos/Sources/Models/Document.swift` | New value type |
| Tree-sitter wrapper | `platforms/macos/Sources/Syntax/SyntaxService.swift` | New grammar / highlight pass |

> **Note:** These paths will exist once initial scaffolding is complete. Until then, the first implementation of each shape becomes the canonical reference. Tag it with `// CANONICAL REFERENCE` at the top.

---

## ERROR HANDLING

```swift
// All service functions throw a typed error
enum EditorError: Error {
    case fileNotFound(URL)
    case encodingFailed(String.Encoding)
    case saveFailed(underlying: Error)
    case syntaxLoadFailed(String)
    case searchCancelled
    case sessionCorrupt
}
```

**Rules:**
- Service functions return `throws` — never silent failures.
- UI layer catches and maps to `NSAlert` / SwiftUI `.alert()`.
- Never use `fatalError` in production paths — only in `#if DEBUG` guards.
- Never swallow errors with `try?` unless the failure is genuinely ignorable (e.g., optional cache write).

---

## CONCURRENCY

All concurrency uses **Swift structured concurrency** (Swift 6, `SWIFT_STRICT_CONCURRENCY = complete`).

| Rule | Detail |
|------|--------|
| No `DispatchQueue` in new code | Use `async/await`, `actor`, `TaskGroup` |
| No `Thread` / `pthread` in new code | Same |
| `@MainActor` for all UI code | `NSView`, `NSViewController`, `AppDelegate` |
| `actor` for all services | `DocumentManager`, `AutoSaveService`, `SessionService`, etc. |
| Task replacement pattern | Each document keeps `var syntaxTask: Task<Void, Never>?` — cancel before creating new |
| `ManagedAtomic<Bool>` for shutdown flag | From `swift-atomics`; `Ordering.releasing` store, `.acquiring` load |
| `SWIFT_DEFAULT_ACTOR_ISOLATION = nonisolated` | Prevents C/tree-sitter calls from being implicitly `@MainActor` |

**Task replacement pattern (mandatory for per-document background work):**

```swift
private var syntaxTask: Task<Void, Never>?

func didEdit(snapshot: PieceTableSnapshot) {
    syntaxTask?.cancel()
    syntaxTask = Task.detached(priority: .userInitiated) {
        try? await Task.sleep(for: .milliseconds(30))
        guard !Task.isCancelled else { return }
        await syntaxService.parse(snapshot: snapshot)
    }
}
```

---

## TEXT BUFFER

The text buffer is a **piece table** backed by a **red-black tree** (index-based nodes, not pointers — avoids `UnsafePointer` and is cache-friendly).

```swift
struct RedBlackTree<Value> {
    private var nodes: [RBNode<Value>] = []
    private var root: Int = -1
    struct RBNode<V> {
        var value: V; var color: RBColor
        var left: Int; var right: Int; var parent: Int
        var subtreeLength: Int = 0; var subtreeLines: Int = 0
    }
}
typealias PieceTree = RedBlackTree<Piece>
```

**`PieceTableSnapshot`** — immutable class (not struct; `Unmanaged` pointer safety for tree-sitter C callbacks):

```swift
final class PieceTableSnapshot: @unchecked Sendable {
    let tree: PieceTree          // value-type copy at snapshot time
    let generation: UInt64
    init(tree: PieceTree, generation: UInt64) { self.tree = tree; self.generation = generation }
}
```

---

## RENDERING

- **Custom `NSView`** with CoreText + Metal — never `NSTextView`.
- **`NSTextInputClient`** implemented for IME (CJK), emoji picker, dictation, system text replacement.
- **Metal PSOs** compiled from `.metallib` bundle at startup — never at draw time.
- **Triple-buffer semaphore:** `DispatchSemaphore(value: 3)` — `wait()` before acquiring drawable, `signal()` in command buffer completion handler.
- **Glyph atlas:** LRU, 32MB cap, separate RGBA atlas for color emoji, invalidate on font/scale change.
- **Line layout cache:** `LineLayoutCache` invalidates per changed line — never full-viewport re-layout for single-line edit.
- **Dirty rect rendering:** redraw only changed lines + cursor line. Never full viewport on edit.

---

## AUTO-SAVE

Three independent systems run in parallel:

| System | Trigger | Destination |
|--------|---------|-------------|
| Debounce auto-save | 1s after last keystroke | Original file path |
| Continuous backup | Every 500ms while dirty | `~/Library/Application Support/mynotepadpp/backups/{doc_id}/` |
| Hot exit | On `applicationWillTerminate` | SQLite (`session.db`) |

**Three-tier atomic write:**

1. **Tier 1 (preferred):** write temp → `fsync` → `rename` (atomic POSIX)
2. **Tier 2 (fallback):** copy original to `.bak` → write in place → delete `.bak`
3. **Tier 3 (last resort):** write to recovery directory, show non-blocking warning

**Tab close:** call `drainAndSave(_:)` — waits for any in-flight save, then does a final dirty check before closing. Never skip-if-busy.

---

## SHUTDOWN

```swift
// AppState — synchronous, fire-and-forget. NO await.
func beginShutdown() {
    _isShuttingDown.store(true, ordering: .releasing)
    DocumentManager.shared.cancelAllBackgroundWorkFireAndForget()
    AutoSaveService.shared.cancelAllFireAndForget()
    BackupService.shared.cancelAllFireAndForget()
}
```

Hot exit sequence (async `Task.detached`):
1. Commit emergency session (synchronous: tab IDs + file paths only).
2. Serialize all dirty buffers to SQLite in one `BEGIN IMMEDIATE` transaction.
3. `PRAGMA wal_checkpoint(TRUNCATE)`.
4. `NSApp.reply(toApplicationShouldTerminate: .now)`.

Budget: ≤500ms total. Abort undo history serialization if over 3s.

---

## SQLITE (GRDB.swift)

Use `DatabaseQueue` (serial writer) — never raw `sqlite3_*` calls for session writes.

**PRAGMAs set on every connection open:**

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = FULL;
PRAGMA fullfsync = ON;
PRAGMA cache_size = -2000;
PRAGMA temp_store = MEMORY;
PRAGMA busy_timeout = 5000;
PRAGMA foreign_keys = ON;
PRAGMA trusted_schema = OFF;
PRAGMA cell_size_check = ON;
```

**Schema rules:**
- `STRICT` tables (SQLite 3.37+).
- `CREATE INDEX idx_tabs_window_sort ON tabs(window_id, sort_order)` — required for session restore ordering.
- `dirty_flag` upsert: `INSERT INTO app_state(key,value) VALUES('dirty_flag','1') ON CONFLICT(key) DO UPDATE SET value='1'`.

---

## SYNTAX HIGHLIGHTING

Tree-sitter C library called via Swift C interop (no Rust bridge).

**`EditMapping` struct** — exact `TSInputEdit` field computation:

```swift
struct EditMapping {
    static func makeInputEdit(startByte: Int, oldEndByte: Int,
                              newText: Data, lineIndex: LineIndex) -> TSInputEdit { ... }
    // Multi-cursor: apply edits in REVERSE byte order to keep earlier offsets valid
    static func applyMultiCursorEdits(_ edits: [EditOperation], to tree: OpaquePointer,
                                      lineIndex: LineIndex) {
        for edit in edits.sorted(by: { $0.startByte > $1.startByte }) { ... }
    }
}
```

**Starvation guard:** if `SyntaxService` cancels 3 consecutive parse tasks due to snapshot age expiry (>1s), the 4th attempt runs to completion regardless.

**Language injection:** `InjectionEngine` reads `injections.scm` queries; calls `mergeHighlights()` to combine parent and child grammar highlight ranges.

---

## SEARCH

- Literal: `Data.range(of:options:)` — hardware-accelerated on Apple Silicon.
- Regex: `NSRegularExpression` (ICU engine).
- Multi-file: `TaskGroup` with `gitignore`-aware file enumeration via `git_ignore_path_is_ignored()` (libgit2).
- **Cap:** 10,000 matches max; `SearchResult.truncated = true` flag when exceeded. Show "10,000+ matches" in UI.
- Multi-file replace: preview sheet → per-file undo groups → progress reporting → cancel support.

---

## CRITICAL RULES (violations caught by code review)

### Load-bearing ☠

| # | Rule |
|---|------|
| S-1 | No force unwraps (`!`) in production code — `guard let` / `if let` / nil coalescing only |
| S-2 | `[weak self]` in all closures that outlive scope |
| S-3 | `deinit` on `EditorCore` (if used) calls any required cleanup |
| S-4 | No `DispatchQueue` for new code — `async/await` only |
| S-5 | Metal: `DispatchSemaphore(value: 3)` triple-buffer — never hold drawable across frames |
| S-6 | Tab close uses `drainAndSave()` — never `serializedSave` with skip-if-busy |
| S-7 | `beginShutdown()` is synchronous fire-and-forget — never `async`, never `await` |
| S-8 | `dirty_flag` written via upsert (`ON CONFLICT DO UPDATE`) — never bare `INSERT` |
| S-9 | Multi-cursor tree-sitter edits applied in **reverse** byte order |
| S-10 | Search results capped at 10,000; `SearchResult.truncated` flag set when exceeded |

### Standards ◆

| # | Rule |
|---|------|
| S-11 | No `Any` / `AnyObject` casts — use protocols and generics |
| S-12 | SwiftLint must pass (`swiftlint lint`) |
| S-13 | `OSLog` for all logging — no `print()` / `NSLog()` in production |
| S-14 | Never log file contents (privacy) |
| S-15 | All UI elements have `accessibilityLabel` / `.accessibility()` (VoiceOver) |
| S-16 | Keyboard shortcuts use `Cmd` (not `Ctrl`) on macOS |
| S-17 | No hardcoded UI strings — all text in `Localizable.strings` |
| S-18 | Date/time via `DateFormatter` — never manual string construction |
| S-19 | `SWIFT_STRICT_CONCURRENCY = complete` — no concurrency warnings suppressed |
| S-20 | Metal PSOs compiled from `.metallib` at startup — never compiled inside `draw()` |

---

## FILE NAMING

| Category | Convention | Example |
|----------|-----------|---------|
| Views | `PascalCase.swift` | `EditorView.swift`, `TabBarView.swift` |
| Services | `PascalCaseService.swift` | `FileService.swift`, `AutoSaveService.swift` |
| Models | `PascalCase.swift` | `Document.swift`, `PieceTable.swift` |
| Extensions | `Type+Feature.swift` | `String+Encoding.swift` |
| Protocols | `PascalCase.swift` | `Highlightable.swift` |
| Tests | `PascalCaseTests.swift` | `PieceTableTests.swift` |

---

## TESTING

```
┌──────────────────────────────────────────────────────────┐
│ Layer 3: Manual QA (VoiceOver, 60fps, crash recovery)    │  ← Your Mac (M4)
├──────────────────────────────────────────────────────────┤
│ Layer 2: XCUITest UI automation                          │  ← Your Mac + CI (macos-14)
├──────────────────────────────────────────────────────────┤
│ Layer 1: XCTest Swift unit tests                         │  ← Your Mac + CI
└──────────────────────────────────────────────────────────┘
```

**Run tests:**

```bash
# Unit tests
xcodebuild test -scheme MyNotepadPP -destination 'platform=macOS'

# UI tests
xcodebuild test -scheme MyNotepadPPUITests -destination 'platform=macOS'

# Specific test class
xcodebuild test -scheme MyNotepadPP -destination 'platform=macOS' \
  -only-testing:MyNotepadPPTests/PieceTableTests
```

**CI (GitHub Actions `macos-14` runner):**

```yaml
jobs:
  macos:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - run: swiftlint lint platforms/macos/
      - run: xcodebuild test -scheme MyNotepadPP -destination 'platform=macOS'
      - run: xcodebuild test -scheme MyNotepadPPUITests -destination 'platform=macOS'
```

**Manual tests every UI change:** VoiceOver (Cmd+F5), 60fps scroll on 100MB file (Instruments), cold start <500ms, crash recovery.

---

## PERFORMANCE TARGETS

| Metric | Target |
|--------|--------|
| Scrolling | 60 FPS minimum on M4 |
| Cold startup | < 500ms to first frame |
| Tab close (`Cmd+W`) | < 200ms (accounts for `drainAndSave`) |
| Memory at idle (1 file) | < 75MB RSS |
| Large file (1GB) | Progressive load — first screenful < 200ms |
| Auto-save write | < 50ms for files up to 100MB |

---

## SECURITY RULES

- Hardened Runtime enabled; App Sandbox enabled.
- **No `com.apple.security.cs.allow-jit`** — use tree-sitter's interpreter backend for any WASM plugins, not JIT.
- **No `com.apple.security.network.client`** in v1.0 entitlements (network is v1.1 only).
- Canonicalize all file paths (`URL.resolvingSymlinksInPath()`) before use.
- URL scheme handler (`mynotepadpp://open?file=…`) requires confirmation dialog; reject system directory paths.
- BiDi override character warning in source files (Trojan Source protection).
- Recovery/backup files: `0600` permissions immediately after creation.
- Clipboard read only on explicit user actions (`Cmd+V`, `Cmd+E`, etc.) — never polled.
- Symlink loop detection during directory enumeration (track visited inodes).

---

## ENTITLEMENTS (`MyNotepadPP.entitlements`)

```xml
<dict>
    <key>com.apple.security.app-sandbox</key><true/>
    <key>com.apple.security.files.user-selected.read-write</key><true/>
    <key>com.apple.security.files.bookmarks.app-scope</key><true/>
    <key>com.apple.security.print</key><true/>
    <!-- v1.1 only: <key>com.apple.security.network.client</key><true/> -->
</dict>
```

---

## DISTRIBUTION

| Channel | Format |
|---------|--------|
| GitHub Releases | `.dmg` (primary) |
| Homebrew Cask | `brew install --cask mynotepadpp` |
| Mac App Store | `.pkg` via Xcode Organizer |

Git tag format: `macos-v1.0.0`. Tag must match binary shipped to store exactly (GPL v3 compliance).

---

## USE SKILLS FOR FILE CREATION

Before creating `.docx`, `.pptx`, `.xlsx`, or `.pdf` files, read the appropriate skill:

| File type | Skill path |
|-----------|-----------|
| Word document | `/var/folders/.../skills/docx/SKILL.md` |
| Presentation | `/var/folders/.../skills/pptx/SKILL.md` |
| Spreadsheet | `/var/folders/.../skills/xlsx/SKILL.md` |
| PDF | `/var/folders/.../skills/pdf/SKILL.md` |
