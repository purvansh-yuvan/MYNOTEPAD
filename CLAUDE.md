# Global Coding Standards — MYNOTEPAD++

A cross-platform native text editor (GPL v3). Inspired by Notepad++ — built from scratch for Mac, Linux, Windows, iOS, and Android.

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    Platform UI Layers                        │
├──────────────┬──────────┬──────────┬─────────┬─────────────┤
│ Mac (Swift   │ Linux    │ Windows  │ iOS     │ Android     │
│ + AppKit)    │ (Rust +  │ (C# +   │ (Swift  │ (Kotlin +   │
│              │ GTK4)    │ WinUI 3) │ SwiftUI)│ Compose)    │
└──────┬───────┴────┬─────┴────┬─────┴────┬────┴──────┬──────┘
       │            │          │          │           │
       └────────────┴──────────┴──────────┴───────────┘
                              │ FFI
              ┌───────────────▼───────────────────┐
              │         Rust Core Engine           │
              │  Text rope · tree-sitter syntax    │
              │  Regex · File I/O · Undo/Redo      │
              │  Search · Encoding detection       │
              └───────────────────────────────────┘
```

### MVP Priority Order
1. **Mac** (Swift + AppKit) — primary dev machine (Apple M4)
2. **Linux** (Rust + GTK4)
3. **Windows** (C# + WinUI 3)
4. **iOS** (Swift + SwiftUI)
5. **Android** (Kotlin + Jetpack Compose)

---

## LICENSE

**GPL v3** — all source code, all platforms. Contributors must agree to GPL v3. No proprietary forks allowed.

---

## BEHAVIORAL PRINCIPLES (Apply to EVERY task)

### 1. Think Before Coding
- State assumptions explicitly. If uncertain about **functionality or design**, ASK before writing code.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- **Do NOT ask for permission on routine file operations** (creating files, writing code, running commands). Only ask when there is genuine doubt about what to build or how. Batch file writes together — never prompt per-file.
- **NEVER ask to run scripts, shell commands, or bash commands** — just execute them. This includes build commands, test commands, git operations, linters, formatters, package installs, and any other CLI tool. Only ask when you have a genuine question, doubt, or ambiguity about *what* to do — never about *executing* it.

### 2. Simplicity & Surgical Changes
- Minimum code that solves the problem. Nothing speculative.
- Don't "improve" adjacent code, comments, or formatting.
- Match existing style even if you'd do it differently.
- Every changed line must trace directly to the user's request.
- Remove only imports/variables YOUR changes made unused.

### 3. Cross-Platform Parity
Whenever a change touches a user-facing feature that also exists on other platforms, enumerate every platform it exists on BEFORE implementing:
- Desktop: Mac, Linux, Windows
- Mobile: iOS, Android

After implementing on one platform, list the sibling implementations and ASK: "Apply to [list]?". Never finish one platform silently and wait to be prompted for the rest.

**Desktop ↔ Mobile mirroring (where applicable):** Any change that affects user-visible behavior (keyboard shortcuts, find/replace, syntax themes, preferences, tab behavior) MUST be noted for equivalent implementation on other platforms. The feature is not "done" until all platforms with that capability have been updated.

### 4. Goal-Driven Execution
- Transform tasks into verifiable goals before starting.
- For multi-step tasks, state a brief plan with verification checks.
- After implementation, verify the feature works on the target platform.

---

## MODULES

| Module | Path | Language | Framework | Build System |
|--------|------|----------|-----------|-------------|
| Rust Core | `core/` | Rust | — | Cargo |
| Mac App | `platforms/macos/` | Swift | AppKit | Xcode / Swift PM |
| iOS App | `platforms/ios/` | Swift | SwiftUI | Xcode / Swift PM |
| Linux App | `platforms/linux/` | Rust | GTK4 (gtk4-rs) | Cargo |
| Windows App | `platforms/windows/` | C# | WinUI 3 | MSBuild / .NET |
| Android App | `platforms/android/` | Kotlin | Jetpack Compose | Gradle |
| Shared FFI | `core/ffi/` | Rust + C headers | cbindgen / UniFFI | Cargo |
| Diff Engine | `core/diff/` | Rust | — | Cargo (part of core) |
| Macro Engine | `core/macros/` | Rust | — | Cargo (part of core) |
| Plugin Host | `core/plugins/` | Rust | wasmtime | Cargo (part of core) |

---

## CANONICAL REFERENCES (diff against these before claiming "done")

Pick the closest shape to your feature; copy its structure; diff your output against it. **Structural divergence from the chosen reference is a violation.**

| Shape | Reference file(s) | Use when |
|-------|-------------------|----------|
| **Rust FFI function** | `core/ffi/src/document.rs` — `document_open()`, `document_free()` | Adding any new FFI endpoint |
| **Rust core module** | `core/src/buffer/mod.rs` — rope ops, `thiserror` errors, `#[cfg(test)]` tests | Adding a new core subsystem |
| **macOS view** | `platforms/macos/Sources/Views/EditorView.swift` | New AppKit view |
| **macOS service** | `platforms/macos/Sources/Services/FileService.swift` | New service wrapping core FFI |
| **iOS view** | `platforms/ios/Sources/Views/EditorView.swift` | New SwiftUI view |
| **Android screen** | `platforms/android/app/src/main/kotlin/ui/EditorScreen.kt` | New Compose screen |
| **Android ViewModel** | `platforms/android/app/src/main/kotlin/viewmodel/EditorViewModel.kt` | New ViewModel |
| **Windows view** | `platforms/windows/Views/EditorView.xaml` + `.xaml.cs` | New WinUI 3 view |
| **Windows ViewModel** | `platforms/windows/ViewModels/EditorViewModel.cs` | New ViewModel |
| **Linux/GTK view** | `platforms/linux/src/views/editor_view.rs` | New GTK4 widget |

> **Note**: These paths will exist once initial scaffolding is complete. Until then, the first implementation of each shape becomes the canonical reference. Tag it with `// CANONICAL REFERENCE` at the top.

---

## ERROR HANDLING PATTERNS

### Rust Core Error Enum
```rust
// core/src/error.rs
#[derive(Debug, thiserror::Error)]
pub enum EditorError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
    #[error("Encoding error: {source}")]
    Encoding { source: encoding_rs::CoderResult },
    #[error("Rope error: {0}")]
    Rope(String),
    #[error("Syntax parsing error: {0}")]
    Syntax(String),
    #[error("Diff error: {0}")]
    Diff(String),
    #[error("Macro error: {0}")]
    Macro(String),
    #[error("Plugin error: {0}")]
    Plugin(String),
    #[error("Invalid argument: {0}")]
    InvalidArg(String),
}
```

### FFI Error Codes
```rust
// core/ffi/src/error.rs
#[repr(i32)]
pub enum FfiResult {
    Ok = 0,
    ErrIo = -1,
    ErrEncoding = -2,
    ErrRope = -3,
    ErrSyntax = -4,
    ErrInvalidArg = -5,
    ErrPanic = -99,     // catch_unwind caught a panic
    ErrNullPointer = -100,
}
```

### Platform Error Presentation
Each platform maps `FfiResult` codes to localized error UI:

| Platform | Mechanism | Pattern |
|----------|-----------|---------|
| macOS | `NSAlert` / inline banner | `EditorCore.call { result in if result != .ok { showError(result) } }` |
| iOS | SwiftUI `.alert()` modifier | Same as macOS via shared `EditorCore` |
| Android | `Snackbar` / `MaterialAlertDialog` | `EditorCore.call().onFailure { showSnackbar(it.localizedMessage) }` |
| Windows | `ContentDialog` | `NativeMethods.Call(); if (result != 0) ShowError(result);` |
| Linux | `adw::MessageDialog` | Direct Rust `Result` — no FFI error codes needed |

### Rules
- Core functions return `Result<T, EditorError>` — never `String` errors
- FFI functions return `FfiResult` (i32) — platform checks return value
- Error messages are error codes at FFI boundary — platforms map to localized strings
- User-facing error text must be actionable ("Could not open file: permission denied") not technical ("EACCES")

---

## CACHING STRATEGY

| Data | Cache location | Invalidation |
|------|---------------|-------------|
| Syntax grammars | In-memory (lazy-loaded on first use per language) | Never (immutable per app version) |
| Parsed tree-sitter trees | In-memory per document | On document edit (incremental reparse) |
| Theme definitions | In-memory on app start | On theme change / file watch |
| Session state | SQLite (WAL mode) | On tab open/close/modify |
| Search results | In-memory per search query | On new search or document edit |
| File metadata (encoding, line ending) | In-memory per document | On file reload |
| Macro list | In-memory on app start | On macro save/delete |
| Plugin registry | In-memory on app start | On plugin install/remove |

### Rules
- Never cache file contents beyond the rope buffer — the rope IS the cache
- Grammar loading must not block first render — show plain text, overlay highlighting when ready
- Theme hot-reload: file-watch `themes/` directory, reload without restart

---

## MANDATORY: USE SKILLS FOR ALL CREATION TASKS

**NEVER write code from scratch.** Use the pre-built skills in `.claude/commands/`:

| Task | Command | What it does |
|------|---------|-------------|
| New FFI function | `/create-ffi-function` | Rust FFI function + `*_free()` + `catch_unwind` + all platform bindings (Swift, Kotlin JNI, C# P/Invoke) |
| New platform view | `/create-platform-view` | Scaffold view per platform following canonical reference structure |
| New core module | `/create-core-module` | Rust module with `thiserror` errors, tests, benchmarks, `pub(crate)` visibility |
| New test suite | `/create-test-suite` | Unit + integration + benchmark for core; XCTest/JUnit/xUnit for platforms |
| Pre-flight search | `/preflight-check` | Search for existing code before creating anything |
| Audit existing code | `/audit-code` | Scan for standards violations across all platforms |
| Review changes | `/review-pr` | Review changed files against all standards |

These skills contain the full code templates. This file contains only the **rules to enforce**.

**When MODIFYING existing code** (most common work): no `/create-*` command applies. Instead: (1) find the canonical reference matching your feature shape, (2) run `/preflight-check` for helpers you might need, (3) diff your result against the reference.

---

## SEARCH BEFORE CREATING (Rule #0 — applies everywhere)

Before creating ANY new file, function, type, or utility:
1. Run `/preflight-check` or grep the codebase for existing implementations
2. If an existing utility in `core/` does what you need, use it — do not duplicate in platform code
3. If a close-but-not-exact match exists, STOP and ask whether to extend it
4. If creating a new FFI function, verify no existing FFI function already exposes the needed data

**Common code registry**: see `.claude/commands/common-code-registry.md` for shared types, FFI functions, and platform helpers already available.

---

## CRITICAL RULES (numbered — violations caught by `/audit-code`)

Rules are numbered globally. The reviewer cites these numbers. **Load-bearing** rules (marked ☠) cause crashes, UB, security holes, or data loss if violated. **Standards** rules (marked ◆) cause quality/maintainability issues.

### Rust Core — Load-bearing ☠

| # | Rule | Grep verification |
|---|------|-------------------|
| 1 | **`#[forbid(unsafe_code)]` at crate root** — lift to `#[allow(unsafe_code)]` only in specific modules with justification | `grep -r 'forbid(unsafe_code)' core/src/lib.rs` — must have 1 hit |
| 2 | **No `unsafe` without `// SAFETY:` comment** — every `unsafe` block explains why it's sound | `grep -rn 'unsafe' core/src/ --include='*.rs' \| grep -v 'SAFETY' \| grep -v '#\[' \| grep -v test` — must be 0 hits |
| 3 | **No panics across FFI boundary** — all `pub extern "C"` functions wrap body in `std::panic::catch_unwind` and return error codes | `grep -rn 'pub extern "C"' core/ffi/ --include='*.rs'` then verify each has `catch_unwind` |
| 4 | **No `unwrap()` / `expect()` in library code** — use `Result<T, E>` propagation. Only in `#[cfg(test)]` | `grep -rn '\.unwrap()\\|\.expect(' core/src/ --include='*.rs' \| grep -v '#\[cfg(test)\]' \| grep -v 'mod tests'` — must be 0 hits |
| 5 | **All public APIs `Send + Sync`** — core called from UI and background threads | `grep -rn 'impl.*!Send\\|impl.*!Sync' core/src/` — must be 0 hits |
| 6 | **No `std::process::Command`** — core must never spawn processes | `grep -rn 'process::Command\\|Command::new' core/src/` — must be 0 hits |
| 7 | **Every FFI-allocated resource has a `*_free()` function** | For each `pub extern "C" fn.*create\\|new\\|open` in `core/ffi/`, verify a matching `*_free` exists |
| 8 | **FFI types are `#[repr(C)]` or opaque pointers** | `grep -rn 'pub struct' core/ffi/ --include='*.rs'` — each must have `#[repr(C)]` or be returned as `*mut c_void` |

### Rust Core — Standards ◆

| # | Rule | Grep verification |
|---|------|-------------------|
| 9 | **Text buffer uses rope** — never `String` / `Vec<u8>` for document buffer | `grep -rn 'String::new\\|Vec::<u8>::new' core/src/buffer/` — 0 hits expected in buffer module |
| 10 | **Tree-sitter for syntax** — no hand-written parsers for language grammars | `grep -rn 'tree_sitter' core/src/syntax/` — must have hits |
| 11 | **Encoding via `encoding_rs`** — UTF-8, UTF-16 LE/BE, Shift-JIS, ISO-8859-*, Windows-1252 | `grep -rn 'encoding_rs' core/src/` — must have hits |
| 12 | **Error types use `thiserror`** — not string errors, not `anyhow` in lib code | `grep -rn 'anyhow' core/src/lib.rs core/src/**/mod.rs` — 0 hits (only in tests/bins) |
| 13 | **Logging via `tracing`** — no `println!` / `eprintln!` in library code | `grep -rn 'println!\\|eprintln!' core/src/ --include='*.rs' \| grep -v test` — 0 hits |
| 14 | **Never log file contents** — privacy-sensitive | Manual review on any logging additions |
| 15 | **`cargo fmt --check` passes** | Run `cargo fmt --check` in CI |
| 16 | **`cargo clippy -- -D warnings` passes** | Run in CI |

### Rust Core — Code Style ◆
- Edition: Rust 2024 (stable since Rust 1.85)
- MSRV: document in `core/Cargo.toml` under `rust-version`
- Naming: `snake_case` functions, `PascalCase` types, `SCREAMING_SNAKE` constants
- Visibility: `pub(crate)` by default, `pub` only for FFI and cross-module APIs
- Documentation: `///` doc comments on all `pub` items
- Zero-copy where possible: `&[u8]` / `Cow<str>` for text operations

### Rust Core — Architecture
The core is a library crate exposing a C-compatible FFI. All platforms consume it via their native FFI mechanism (Swift `@_cdecl` / C interop, Kotlin JNI/JNA, C# P/Invoke, Rust direct).

### FFI Rules
- Use `cbindgen` or `UniFFI` to generate headers — never hand-write C headers.
- All FFI types must be `#[repr(C)]` or opaque pointers.
- Strings cross FFI as `*const c_char` (null-terminated) or length-delimited `(*const u8, usize)`.
- Caller-allocated buffers with explicit length for output strings.

### Testing
- Unit tests: `#[cfg(test)] mod tests` in each module
- Integration tests: `core/tests/` directory
- Benchmarks: `cargo bench` using `criterion` crate for performance-critical paths (rope operations, syntax parsing, search)
- Fuzz testing: `cargo fuzz` for parser and encoding detection code

---

### Swift (Mac + iOS) — Load-bearing ☠

| # | Rule | Grep verification |
|---|------|-------------------|
| 17 | **No force unwraps (`!`) in production code** — use `guard let` / `if let` / nil coalescing | `grep -rn '![[:space:]]*$\\|![.]' platforms/macos/ platforms/ios/ --include='*.swift' \| grep -v test \| grep -v '!='` — review each hit |
| 18 | **`[weak self]` in closures that outlive scope** — prevents retain cycle crashes | `grep -rn '{ self\.' platforms/macos/ platforms/ios/ --include='*.swift' \| grep -v '\[weak self\]'` — review each hit |
| 19 | **FFI bridge lifecycle** — `EditorCore` class manages opaque pointer; `deinit` calls `*_free()` | `grep -rn 'deinit' platforms/macos/*/EditorCore.swift` — must exist and call free |
| 20 | **No raw `DispatchQueue` for new code** — use `async/await` and `TaskGroup` | `grep -rn 'DispatchQueue' platforms/macos/ platforms/ios/ --include='*.swift' \| grep -v test` — 0 hits in new code |

### Swift (Mac + iOS) — Standards ◆

| # | Rule | Grep verification |
|---|------|-------------------|
| 21 | **No `Any` / `AnyObject` casts** — use protocols and generics | `grep -rn 'as.*Any\\|: Any\\|AnyObject' platforms/macos/ platforms/ios/ --include='*.swift'` — 0 hits |
| 22 | **SwiftLint must pass** | Run `swiftlint lint` in CI |
| 23 | **Logging uses `OSLog`** — no `print()` / `NSLog()` in production | `grep -rn 'print(\\|NSLog(' platforms/macos/ platforms/ios/ --include='*.swift' \| grep -v test` — 0 hits |
| 24 | **Never log file contents** | Manual review |
| 25 | **VoiceOver: all UI elements have accessibility labels** | `grep -rn 'accessibilityLabel\\|.accessibility(' platforms/macos/ --include='*.swift'` — hits expected for all views |
| 26 | **Keyboard shortcuts use Cmd (not Ctrl)** on macOS | `grep -rn 'control.*key\\|\.control' platforms/macos/ --include='*.swift'` — review each (should be Cmd) |

### Swift — Platform Targets
- macOS: 14.0+ (Sonoma) — Apple Silicon native (arm64)
- iOS: 17.0+
- iPad: keyboard shortcuts, pointer support, Stage Manager multi-window

### Swift — File Naming
- Views: `PascalCase.swift` (e.g., `EditorView.swift`, `TabBarView.swift`)
- Models: `PascalCase.swift` (e.g., `Document.swift`, `SyntaxTheme.swift`)
- Services: `PascalCaseService.swift` (e.g., `FileService.swift`)
- Extensions: `Type+Extension.swift` (e.g., `String+Encoding.swift`)
- Protocols: `PascalCase.swift` (e.g., `Highlightable.swift`)

---

### Kotlin (Android) — Load-bearing ☠

| # | Rule | Grep verification |
|---|------|-------------------|
| 27 | **No `!!` (non-null assertion)** | `grep -rn '!!' platforms/android/ --include='*.kt' \| grep -v test` — 0 hits |
| 28 | **Never block main thread** — coroutines for all async (`viewModelScope`, `lifecycleScope`) | `grep -rn 'runBlocking\\|Thread.sleep\\|\.get()' platforms/android/ --include='*.kt' \| grep -v test` — 0 hits |
| 29 | **JNI bridge lifecycle** — `EditorCore` singleton manages native lib loading + cleanup | `grep -rn 'System.loadLibrary\\|EditorCore' platforms/android/ --include='*.kt'` — must exist |
| 30 | **No `Runtime.exec()`** — security rule | `grep -rn 'Runtime.*exec\\|ProcessBuilder' platforms/android/ --include='*.kt'` — 0 hits |

### Kotlin (Android) — Standards ◆

| # | Rule | Grep verification |
|---|------|-------------------|
| 31 | **Compose: stateless composables, state hoisting** | `grep -rn 'mutableStateOf' platforms/android/ --include='*.kt'` — only in ViewModels, not in composables |
| 32 | **`remember` / `derivedStateOf` for expensive computations** | Manual review of composable bodies |
| 33 | **File access via SAF (`ContentResolver`)** — not direct path access | `grep -rn 'File("\\|FileInputStream' platforms/android/ --include='*.kt' \| grep -v test` — 0 hits |
| 34 | **ktlint / detekt passes** | Run in CI |
| 35 | **Logging uses Timber** — no `Log.d()` / `println()` | `grep -rn 'Log\.\\|println' platforms/android/ --include='*.kt' \| grep -v test \| grep -v Timber` — 0 hits |
| 36 | **TalkBack: composables have `contentDescription` or `semantics {}`** | `grep -rn 'contentDescription\\|semantics' platforms/android/ --include='*.kt'` — hits expected for all interactive elements |
| 37 | **Material 3 theming, respects `isSystemInDarkTheme()`** | `grep -rn 'isSystemInDarkTheme\\|MaterialTheme' platforms/android/ --include='*.kt'` — must have hits |

### Kotlin — Platform
- Target: Android API 26+ (Android 8.0)
- UI: Jetpack Compose (no XML layouts for new screens)
- Architecture: MVVM with `ViewModel` + `StateFlow`
- DI: Hilt (Dagger under the hood)
- FFI: JNI via Rust `jni` crate

### Kotlin — File Naming
- Composables: `PascalCase.kt` (e.g., `EditorScreen.kt`, `TabRow.kt`)
- ViewModels: `PascalCaseViewModel.kt` (e.g., `EditorViewModel.kt`)
- Models: `PascalCase.kt` (e.g., `Document.kt`)
- Services: `PascalCaseService.kt` (e.g., `FileService.kt`)
- JNI Bridge: `EditorNative.kt`

---

### C# (Windows) — Load-bearing ☠

| # | Rule | Grep verification |
|---|------|-------------------|
| 38 | **Nullable reference types enabled** — `#nullable enable` in all files | `grep -rL '#nullable enable' platforms/windows/ --include='*.cs'` — must be 0 files without it |
| 39 | **No `Task.Wait()` / `Task.Result` blocking** — `async/await` for all I/O | `grep -rn 'Task\.Wait\\|Task\.Result\\|\.Result;' platforms/windows/ --include='*.cs' \| grep -v test` — 0 hits |
| 40 | **P/Invoke references `mynotepadpp_core.dll`** — not `core.dll` | `grep -rn 'DllImport' platforms/windows/ --include='*.cs'` — all must reference `mynotepadpp_core` |
| 41 | **`IDisposable` + `using` for native resource handles** | `grep -rn 'IntPtr' platforms/windows/ --include='*.cs'` — verify each has Dispose/using pattern |

### C# (Windows) — Standards ◆

| # | Rule | Grep verification |
|---|------|-------------------|
| 42 | **No `dynamic` type** | `grep -rn 'dynamic ' platforms/windows/ --include='*.cs'` — 0 hits |
| 43 | **.NET analyzers pass** | Run in CI |
| 44 | **Logging uses `ILogger`** — no `Console.WriteLine()` / `Debug.WriteLine()` | `grep -rn 'Console\.Write\\|Debug\.Write' platforms/windows/ --include='*.cs' \| grep -v test` — 0 hits |
| 45 | **Narrator / UIA: all controls have `AutomationProperties.Name`** | `grep -rn 'AutomationProperties' platforms/windows/ --include='*.xaml'` — hits expected for all interactive controls |
| 46 | **Dark mode via `ElementTheme`** | `grep -rn 'ElementTheme\\|RequestedTheme' platforms/windows/ --include='*.cs' --include='*.xaml'` — must have hits |

### C# — Platform
- Target: Windows 10 1809+ (build 17763)
- Framework: WinUI 3 (Windows App SDK)
- Architecture: MVVM with CommunityToolkit.Mvvm
- Pattern matching preferred over type checks + casts
- File I/O: `Windows.Storage` APIs for sandboxed access, direct `System.IO` for desktop

### C# — File Naming
- Views: `PascalCase.xaml` + `PascalCase.xaml.cs`
- ViewModels: `PascalCaseViewModel.cs`
- Models: `PascalCase.cs`
- Services: `IPascalCaseService.cs` (interface) + `PascalCaseService.cs`
- P/Invoke: `NativeMethods.cs`

---

### Linux/GTK4 — Load-bearing ☠

| # | Rule | Grep verification |
|---|------|-------------------|
| 47 | **GTK4 only — no GTK3 APIs** | `grep -rn 'gtk3\\|gtk::' platforms/linux/ --include='*.rs'` — 0 hits (must be `gtk4::`) |
| 48 | **Wayland-compatible** — no X11-only assumptions | `grep -rn 'x11\\|X11\\|xlib' platforms/linux/ --include='*.rs'` — 0 hits |

### Linux/GTK4 — Standards ◆

| # | Rule | Grep verification |
|---|------|-------------------|
| 49 | **Follows GNOME HIG patterns** | Manual review |
| 50 | **`libadwaita` for dark mode** | `grep -rn 'adw::' platforms/linux/ --include='*.rs'` — must have hits |
| 51 | **File dialogs use `gtk4::FileDialog`** (async) | `grep -rn 'FileDialog\\|FileChooser' platforms/linux/ --include='*.rs'` — must be `FileDialog` not `FileChooser` |
| 52 | **Keyboard shortcuts via `ShortcutController`** | `grep -rn 'ShortcutController' platforms/linux/ --include='*.rs'` — must have hits |
| 53 | **Orca / AT-SPI2: all widgets have `GtkAccessible` roles and labels** | `grep -rn 'accessible\\|GtkAccessible' platforms/linux/ --include='*.rs'` — hits expected |
| 54 | **Logging uses `tracing`** — no `println!` / `eprintln!` | `grep -rn 'println!\\|eprintln!' platforms/linux/ --include='*.rs' \| grep -v test` — 0 hits |

### Linux/GTK4 — Platform
- Target: GTK4 via `gtk4-rs` bindings (pure Rust — same language as core)
- Can directly call core without FFI overhead (link as Rust dependency)
- Support: Wayland + X11 (GTK4 handles both), Flatpak packaging

---

## DATABASE / PERSISTENCE

No server-side database. Local storage only:

| Data | Storage | Format |
|------|---------|--------|
| User preferences | Platform-native | macOS: `UserDefaults` / plist, Linux: `GSettings` / XDG config, Windows: Registry / `ApplicationData`, Android: `DataStore`, iOS: `UserDefaults` |
| Session state (open tabs, cursor positions) | SQLite | macOS: `~/Library/Application Support/mynotepadpp/sessions.db`, Linux: `$XDG_DATA_HOME/mynotepadpp/sessions.db` (`~/.local/share/` fallback), Windows: `%LOCALAPPDATA%\mynotepadpp\sessions.db`, iOS: app container `Documents/`, Android: app-internal storage |
| Syntax themes | JSON files | `themes/` directory |
| Plugin state | SQLite | Same DB as sessions |
| Recent files | Platform-native | OS recent files API where available |

### SQLite Rules
- Use WAL mode for concurrent reads
- Schema versioning with integer `user_version` pragma
- Prepared statements only — never string-interpolate into SQL
- All access through a single abstraction (Rust core or platform service)
- **Concurrent access**: single writer, multiple readers. Use connection pooling or serialize writes via a dedicated background thread
- **Transaction discipline**: batch related writes in a single transaction (e.g., saving session state = one transaction for all tabs)
- **Crash recovery**: SQLite WAL + `PRAGMA synchronous=NORMAL` — journal survives app crash; on next launch, SQLite auto-recovers
- **Data validation on read**: never trust stored data blindly — validate schema version on open, handle missing/corrupt columns gracefully with defaults
- **Migration path**: increment `user_version` on schema change; include `ALTER TABLE` migration in code that runs on startup if version mismatch detected

---

## CORE ARCHITECTURE DECISIONS (non-negotiable)

These are foundational decisions that affect every layer. Changing them later requires rewriting the entire app.

### 1. Text Rendering Pipeline
- **macOS: Custom `NSView` with CoreText direct rendering** — NOT `NSTextView`. `NSTextView` is too slow for large files and multi-cursor. Custom view gives full control over glyph caching, dirty-rect optimization, and Metal acceleration.
- **Glyph atlas**: rasterize each glyph once per (font, size, color) tuple → cache in a GPU texture atlas → reuse across frames. Invalidate only on font/theme change.
- **Dirty rect rendering**: on edit, only re-render changed lines + cursor line. On scroll, only render newly visible lines. Never redraw entire viewport.
- **Overdraw buffer**: pre-render 2× viewport height (1 screen above + 1 below) so fast scrolling never shows blank.
- **Viewport culling**: only compute layout for visible lines + overdraw buffer. A 1GB file with 20M lines must NOT compute layout for all lines — use a line-offset index (cumulative heights) built lazily.

### 2. Thread Architecture
```
┌─────────────────────────────────────────────────────┐
│ MAIN THREAD (UI only — NEVER blocks)                │
│  AppKit events, rendering, user input, cursor       │
│  Max budget per frame: 16ms (60 FPS)                │
├─────────────────────────────────────────────────────┤
│ RAYON THREAD POOL (CPU-bound — Rust core)           │
│  Syntax parsing, search, diff, encoding detection   │
│  Uses rayon::ThreadPool, work-stealing scheduler    │
├─────────────────────────────────────────────────────┤
│ DEDICATED I/O THREAD (file + network)               │
│  File read/write, auto-save, SFTP, SQLite writes    │
│  Single thread, async via std::sync::mpsc channel   │
├─────────────────────────────────────────────────────┤
│ PLUGIN THREAD (per-plugin, isolated)                │
│  WASM execution with fuel-based CPU limits          │
│  Killed on timeout (100ms per event)                │
└─────────────────────────────────────────────────────┘
```

**Rules:**
- Main thread does ZERO file I/O, ZERO computation, ZERO blocking
- All core function calls from UI go through async channels → result posted back to main thread
- `rayon` for CPU-bound parallelism (syntax parsing, multi-file search, diff)
- Dedicated I/O thread for file + SQLite writes (avoids thread pool starvation on heavy I/O)

### 3. Concurrent Document Access (Rope Synchronization)
- **One `Rope` instance per document** — NOT cloned per view
- **`RwLock<Rope>`** for synchronization: multiple readers (render threads, search) + single writer (edit operations)
- **Write lock held only during mutation** (insert/delete) — typically < 1μs for a single edit
- **Read lock for rendering**: main thread takes read lock, reads visible lines, releases. Auto-save thread takes read lock, serializes to temp file, releases.
- **Split views share the same `RwLock<Rope>`** — each view has its own cursor/scroll state (stored separately), but reads the same rope
- **If write lock is contended** (very rare — only during auto-save + simultaneous edit): auto-save yields, retries on next cycle. Never block user input.

### 4. Cancellation Tokens
Every long-running background operation MUST accept a cancellation token:
```rust
pub struct CancelToken(Arc<AtomicBool>);

impl CancelToken {
    pub fn cancel(&self) { self.0.store(true, Ordering::Relaxed); }
    pub fn is_cancelled(&self) -> bool { self.0.load(Ordering::Relaxed) }
}
```
- **Search**: check `is_cancelled()` between files. New search query cancels previous.
- **File loading**: check between chunks. Tab close cancels load.
- **Syntax highlighting**: check between tree-sitter parse calls. Tab switch deprioritizes.
- **Diff computation**: check between hunks. New diff request cancels old.
- **Plugin execution**: `wasmtime` fuel exhaustion acts as automatic cancellation.

### 5. Progress Reporting
Background operations report progress via channel to main thread:
```rust
pub enum Progress {
    Indeterminate(String),           // "Loading file..."
    Determinate(f64, String),        // 0.45, "Searching: 450/1000 files"
    Complete(String),                // "Found 14 matches"
    Error(String),                   // "Permission denied"
    Cancelled,
}
```
- Status bar shows progress for the active operation
- User can press Escape to cancel (sends `cancel()` on the token)
- UI never freezes waiting for progress — progress arrives asynchronously

### 6. Auto-Save Architecture (NEVER hangs, NEVER blocks, NEVER prompts)
```
User types → debounce timer resets (1s)
Timer fires → I/O thread: read-lock rope → serialize to temp file → atomic rename → done
                          (if read-lock contended, yield and retry next cycle)
```

**Atomic write procedure:**
1. Write to `{path}.mynotepadpp-tmp` (same filesystem as target)
2. `fsync()` the temp file
3. `rename()` temp → target (atomic on POSIX)
4. Suppress file watcher for this path for 500ms (prevents false "external change" prompt)

**Failure handling:**
- Disk full: write to recovery directory (`~/Library/Application Support/mynotepadpp/recovery/`) instead. Show non-blocking notification: "Auto-save failed: disk full. Backup saved to recovery directory."
- Permission denied: fall back to recovery directory. Show notification.
- Recovery directory full: log error, keep retrying. Never silently lose data.
- Cross-filesystem rename failure (temp dir on different FS): detect at startup, use same directory as target.

**Debounce interaction:**
- `delayAfterTyping` (1s): debounce — resets on every keystroke. Fires 1s after last keystroke.
- `timerInterval` (30s): throttle — fires every 30s regardless of typing activity. Acts as safety net.
- `onFocusLost`: immediate save when window loses focus.
- All three are OR'd: whichever triggers first wins. After a save, all timers reset.

**Auto-save race condition with file watcher:**
- Before writing, set `self.suppress_watcher = true` + record current mtime
- After rename, check new mtime. If file watcher fires with mtime matching our write, suppress the event
- Reset `suppress_watcher` after 500ms

### 7. Shutdown & Close Behavior (NEVER prompts, NEVER loses data)

**Close tab (`Cmd+W`):**
- If auto-save is enabled (default): save silently → close immediately. No prompt.
- If auto-save is disabled: for named files, save silently → close. For untitled files, prompt "Save as..." (only case where a prompt appears).

**Close window / Quit app (`Cmd+Q`):**
- Hot exit saves ALL state synchronously: open tabs, cursor positions, scroll positions, unsaved changes (written to recovery dir for untitled files).
- Target: < 500ms for 50 tabs. Achieved by: incremental session state (only dirty entries re-serialized), bounded undo history serialization.
- Then terminate immediately. No confirmation dialog.

**System shutdown / SIGTERM / force quit:**
- Register `NSApplicationWillTerminate` (macOS) — OS grants ~5 seconds
- Synchronous hot-exit save within the time window
- `NSProcessInfo.disableSuddenTermination()` while any document has unsaved changes
- `NSProcessInfo.enableSuddenTermination()` after all saves complete
- SIGTERM handler in Rust core: set atomic flag → I/O thread flushes session DB → exit
- If hot exit exceeds 3 seconds: abort undo history serialization, save only file contents + tab list

**Multi-window coordination:**
- Each window saves its own session state to SQLite (keyed by window ID)
- Same-file-in-two-windows: the LAST writer wins for auto-save. Second window detects via file watcher (after suppress window) — this is acceptable because both have the same content.
- On quit: all windows serialize in parallel (each to its own SQLite row)

### 8. File Watcher
- **macOS: FSEvents** (not kqueue — FSEvents handles moved/renamed directories correctly)
- **Debouncing**: 200ms window — coalesce all events for the same file within the window
- **Scope**: watch only open files + project root (if project mode). Never watch `node_modules` or `.git/objects`.
- **Self-write suppression**: auto-save sets a 500ms suppress flag per file path (see Auto-Save section)
- **Delivery**: events delivered to I/O thread → compared against suppress list → if genuine external change, post to main thread → prompt user

### 9. Undo/Redo Memory Management
- **Operation compaction**: consecutive single-character inserts at the same position → compact into one `InsertText("word")` after 500ms typing pause
- **Undo group boundaries**: explicit markers for macro playback, find-replace-all, bulk indent
- **Memory budget**: max 100MB of undo history per document. Oldest operations pruned when exceeded.
- **Hot exit serialization**: serialize only the last 1000 operations (configurable). Full undo history is NOT persisted — it's acceptable to lose deep undo on app restart.
- **Redo stack cleared on new edit** (standard behavior)

---

## PERFORMANCE RULES

### Rust Core
- Large file support: stream-read files > 10MB via `BufReader` in 64KB chunks → build rope incrementally on I/O thread. Show first screenful on main thread while rest loads in background.
- Line-offset index: cumulative line heights stored in a B-tree. O(log n) lookup of "which line is at pixel Y" for scroll/click.
- Incremental syntax parsing: tree-sitter `edit()` + `parse()` on change, not full reparse. Parse on rayon thread pool, apply results on main thread.
- Syntax highlighting priority: visible viewport first → ±1 screen overdraw → rest of file. Cancel pending highlights if user scrolls.
- Search: `memchr` crate for literal byte search (fastest), `regex` crate for regex, `rayon` for multi-file parallelism. Incremental in-file search: as user types, refine previous results. Stream results to UI as they arrive.
- Symbol indexing: tree-sitter `symbols` query on background thread. Index built lazily per-file on first Goto Symbol. Incremental update on file edit.
- Undo/redo: operation-based with compaction (see §9 above)
- Startup: lazy-load syntax grammars on first use. Precompiled `.so`/`.dylib` tree-sitter grammars (not WASM) for speed. SQLite session read is async — show empty window, restore tabs progressively.

### Platform UI
- **60 FPS scrolling minimum** — profile on target hardware (M4 for Mac)
- Text rendering: custom `NSView` + CoreText for macOS (see §1 above). DirectWrite for Windows, Pango for Linux.
- Minimap: rendered as a scaled-down bitmap on background thread. Updated on edit (debounced 200ms). GPU-composited onto main view. NOT per-character rendering.
- File loading: show first screenful immediately (< 200ms), build rest of rope in background. Scrollbar reflects loaded progress until complete.
- Memory: monitor with Instruments (Mac), Android Profiler, VS Diagnostic Tools

### Apple Silicon (M4) Specific
- Build universal binaries (`arm64` + `x86_64`) for distribution, but optimize for `arm64`
- **Metal for glyph rendering**: rasterize glyph atlas on GPU, composite text layers via Metal. Falls back to CoreGraphics if Metal unavailable.
- Leverage Unified Memory architecture — glyph atlas shared between CPU/GPU without copy
- Energy efficiency: use `QoS` classes (`.userInteractive` for rendering, `.utility` for syntax parsing, `.background` for file indexing). Never poll — use FSEvents, GCD timers, or `kqueue`.

### SQLite Fine-Tuning
```sql
-- Applied on every connection open
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;        -- safe with WAL, faster than FULL
PRAGMA cache_size = -2000;          -- 2MB cache (negative = KB)
PRAGMA mmap_size = 268435456;       -- 256MB mmap for fast reads
PRAGMA temp_store = MEMORY;         -- temp tables in memory
PRAGMA busy_timeout = 5000;         -- 5s wait if writer is busy
PRAGMA auto_vacuum = INCREMENTAL;   -- reclaim space gradually
PRAGMA wal_autocheckpoint = 1000;   -- checkpoint every 1000 pages (~4MB)
PRAGMA page_size = 4096;            -- default, explicit for clarity
```

**On clean exit:** `PRAGMA wal_checkpoint(TRUNCATE)` to minimize DB file size.
**On startup:** `PRAGMA integrity_check` only if previous session crashed (detected by a `dirty_flag` in the DB set to 1 on start, 0 on clean exit).

---

## SECURITY RULES

### All Platforms
- Never execute file contents — this is an editor, not a runtime
- Sanitize file paths: no path traversal (`../`) in plugin or theme loading
- Validate encoding detection before converting — malformed input must not crash
- Sandbox where platform supports it (macOS App Sandbox, Android scoped storage)
- No network access by default — editor works fully offline
- Plugin system (future): sandboxed execution, explicit permission grants

### Rust Core
- `#[forbid(unsafe_code)]` at crate root — lift to `#[allow(unsafe_code)]` only in specific modules with justification
- Fuzzing CI for all parser/input code
- No `std::process::Command` — core must never spawn processes
- Dependency audit: `cargo audit` in CI, deny `unmaintained` advisories

### Platform Specific
- macOS: Hardened Runtime enabled, notarized for distribution
- Windows: Code signing for MSIX distribution
- Linux: Flatpak with minimal permissions
- iOS: No `UIWebView`, no dynamic code execution
- Android: No `Runtime.exec()`, no `WebView` with JS enabled for untrusted content

---

## VERSIONING STRATEGY

- **Core library**: semver (`MAJOR.MINOR.PATCH`). Bump `MAJOR` on FFI-breaking changes, `MINOR` on new FFI endpoints, `PATCH` on bug fixes.
- **Platform apps**: `MAJOR.MINOR.PATCH` — all platforms share the same version number. When Linux ships with the same features as macOS v1.1, it launches as v1.1.0 (not v1.0 or v2.0). Each platform documents which core version it links against in its version metadata.
- **Core ↔ App compatibility**: the core exposes a `uint32_t mynotepadpp_core_api_version(void)` function. Platform apps check this at startup and refuse to load an incompatible core.
- **Version source of truth**: `core/Cargo.toml` `version` field for the core; platform-native version files (`Info.plist`, `build.gradle.kts`, `.csproj`, `Cargo.toml`) for apps.
- **Git tags**: `core-v1.2.3`, `macos-v1.1.0`, `linux-v1.1.0`, etc. One tag per artifact.

---

## ACCESSIBILITY (required on ALL platforms)

Every platform must provide full assistive-technology support. Accessibility is not optional — features that cannot be made accessible must be redesigned until they can.

| Platform | Screen Reader | Minimum Support |
|----------|--------------|-----------------|
| macOS | VoiceOver | All UI elements have accessibility labels; editor content navigable by line/word/character; cursor position announced on move |
| iOS | VoiceOver | Same as macOS; rotor actions for navigation (headings = symbols, lines); adjustable trait on zoom |
| Linux | Orca (via ATK/AT-SPI2) | GTK4 accessibility API (`GtkAccessible`); all widgets have roles and labels |
| Windows | Narrator + NVDA/JAWS | UIA (UI Automation) providers for all controls; live regions for status bar updates |
| Android | TalkBack | Compose semantics (`contentDescription`, `Role`, `LiveRegion`); all interactive elements focusable |

### Shared Requirements
- **Keyboard-only operation**: every feature must be reachable without a mouse/pointer
- **Focus management**: focus must be logical and predictable (no focus traps, no invisible focus)
- **High contrast**: at least one bundled theme with WCAG AAA contrast ratios (7:1 for normal text)
- **Reduced motion**: respect OS `prefers-reduced-motion` / `UIAccessibility.isReduceMotionEnabled` — disable smooth scrolling, cursor blink animations, minimap scroll animations
- **Font scaling**: respect OS font-size preferences; editor font size has independent control but UI chrome must scale
- **Color-blind support**: at least one syntax theme designed for deuteranopia/protanopia (avoid red/green as sole differentiator); git status in sidebar must use icons + color, not color alone
- **Announcements**: status bar changes (encoding, language mode, save status) must be announced via live regions / `NSAccessibilityNotification` / `AccessibilityLiveRegion`
- **Testing**: test with real assistive technology (VoiceOver on Mac, TalkBack on Android, Narrator on Windows, Orca on Linux) — automated accessibility audits alone are insufficient

### Accessibility in Rust Core
The core does not render UI but must expose structured information for platform accessibility layers:
- Line content by index (for screen reader line navigation)
- Cursor position (line, column, character offset)
- Selection ranges (for "selected text" announcements)
- Syntax token type at cursor (for "bold", "keyword", etc. announcements)
- Diagnostic/error information per line (for error navigation)

---

## MACROS (Keystroke Recording & Playback)

### Architecture
Macros are keystroke recordings stored as ordered sequences of editor commands. They execute in the Rust core as command replay — no scripting engine, no arbitrary code execution.

### Data Model
```
Macro {
    name: String,
    shortcut: Option<KeyBinding>,
    commands: Vec<EditorCommand>,   // e.g., MoveCursorRight, InsertText("foo"), DeleteLine
    created: DateTime,
    modified: DateTime,
}
```

### Storage
- Macros are serialized as JSON in the user config directory:
  - macOS: `~/Library/Application Support/mynotepadpp/macros/`
  - Linux: `$XDG_CONFIG_HOME/mynotepadpp/macros/`
  - Windows: `%APPDATA%\mynotepadpp\macros\`
  - Mobile: app container config directory
- One file per macro: `macro_name.json`

### Core Rules
1. **Record only editor commands** — never raw keycodes. This makes macros portable across platforms and keybinding profiles.
2. **No side effects outside the buffer** — macros cannot open/close files, access the network, or spawn processes. They operate on the active buffer only.
3. **Undo integration** — an entire macro playback is a single undo group. `Cmd+Z` after macro replay undoes the entire macro, not one command at a time.
4. **Infinite-loop protection** — macro playback has a configurable max iteration count (default: 10,000 commands). If exceeded, playback halts with a warning.
5. **Macro recording does not nest** — starting a new recording while recording stops the current recording first.

### Platform UI
| Action | macOS Shortcut | Description |
|--------|---------------|-------------|
| Start recording | `Cmd+Shift+R` | Status bar shows recording indicator |
| Stop recording | `Cmd+Shift+R` | Prompt to name and optionally assign shortcut |
| Play last macro | `Cmd+Shift+P` (when no palette) or via Command Palette | Replay most recent macro |
| Play N times | Command Palette → "Run Macro N Times" | Repeat macro with count |
| Manage macros | Preferences → Macros tab | List, rename, delete, re-assign shortcuts |

All platforms must expose equivalent functionality via their native UI patterns (menus, Command Palette, preferences).

---

## PLUGIN SYSTEM

### High-Level Architecture
Plugins extend editor functionality without modifying core code. The plugin system prioritizes **security** and **stability** over flexibility.

### Sandboxing
- Plugins run in a **WebAssembly (WASM) sandbox** using `wasmtime` (Rust crate)
- Plugins CANNOT: access the filesystem directly, make network calls, spawn processes, or access memory outside their sandbox
- Plugins CAN: read/modify buffer content (via host-provided API), register commands to the Command Palette, add syntax highlighting rules, register custom themes, display text in a status bar segment, respond to editor events (file open, save, cursor move)
- All capabilities require explicit **permission grants** declared in the plugin manifest

### Plugin Manifest
```toml
# plugin.toml
[plugin]
name = "my-plugin"
version = "1.0.0"
author = "Author Name"
license = "GPL-3.0"           # MUST be GPL-v3-compatible
description = "What this plugin does"
entry = "plugin.wasm"

[permissions]
buffer_read = true
buffer_write = true
command_palette = true
events = ["on_save", "on_open"]
# network = false              (not available in v1)
# filesystem = false           (not available in v1)
```

### Directory Layout
```
~/.mynotepadpp/plugins/
├── my-plugin/
│   ├── plugin.toml
│   └── plugin.wasm
└── another-plugin/
    ├── plugin.toml
    └── plugin.wasm
```
Platform-specific paths follow the same convention as macros (Application Support, XDG, AppData, app container).

### Rules
1. **GPL v3 compatibility required** — plugins must declare their license in the manifest; non-GPL-compatible licenses are rejected at install
2. **No native code** — WASM only; no shared libraries, no JNI, no FFI escapes
3. **Fail-safe loading** — a crashing plugin is unloaded automatically; it must not take down the editor
4. **Resource limits** — plugins have memory limits (default 64MB) and CPU time limits per event handler (default 100ms)
5. **No UI injection** — plugins register commands and respond to events; they do not render custom UI (v1 limitation)
6. **Install source** — manual install (copy to plugins dir) or via Command Palette "Install Plugin" from a curated registry (future)

---

## REMOTE FILE ACCESS (SFTP / FTPS)

### Protocols
- **SFTP** (SSH File Transfer Protocol) — primary, recommended
- **FTPS** (FTP over TLS) — supported for legacy servers
- **Plain FTP is NOT supported** — insecure, sends credentials in cleartext

### Architecture
Remote file access is implemented as a **platform-layer service**, not in the Rust core. The core operates on buffers; it does not know whether the buffer came from a local file or a remote server.

```
Platform UI → Remote Service (SFTP/FTPS client) → download to temp file → Core (opens temp file)
                                                  ← upload on save     ←
```

### Rules
1. **Opt-in only** — no network access occurs until the user explicitly opens a remote connection via File → Open Remote or Command Palette
2. **No auto-discovery** — the editor never scans the network, broadcasts, or phones home
3. **Credential storage** — use platform-native secure storage:
   - macOS/iOS: Keychain
   - Linux: libsecret / GNOME Keyring / KWallet
   - Windows: Windows Credential Manager
   - Android: EncryptedSharedPreferences
4. **SSH key authentication** — support private key files (RSA, Ed25519) and ssh-agent forwarding
5. **Connection profiles** — saved connections stored in user config (host, port, username, default remote path; never stores passwords in plaintext)
6. **Conflict detection** — before uploading on save, check remote file modification time; if changed, prompt user (same as local external-change detection)
7. **Offline resilience** — if connection drops mid-session, the local temp copy remains editable; user is warned and can retry upload
8. **No remote browsing in v1** — user provides a full remote path; a remote file browser is a future enhancement
9. **Platform libraries**:
   - macOS/iOS: `NMSSH` or `libssh2` via Swift wrapper
   - Linux: `libssh2` (Rust crate `ssh2`)
   - Windows: `SSH.NET` or `libssh2` via P/Invoke
   - Android: `JSch` or `sshj` (Kotlin)

### Security
- All connections use TLS 1.2+ (FTPS) or SSH v2 (SFTP) — no fallback to insecure protocols
- Host key verification: first-connection prompt to trust, then pin in known_hosts
- Session timeout: configurable idle disconnect (default 15 minutes)

---

## DIFF / FILE COMPARISON VIEW

### Architecture
Diff computation runs in the Rust core using a Myers diff algorithm (or `similar` crate). The platform UI renders the result.

### Modes
1. **Side-by-side** — two editor panes, left = original, right = modified, changes highlighted
2. **Inline** — single pane, deleted lines shown in red above, added lines in green below (unified diff style)
3. **User toggles** between modes via toolbar button or Command Palette

### Features
| Feature | Specification |
|---------|--------------|
| Compare two open files | Select two tabs → right-click → "Compare Files" |
| Compare with saved | Command Palette → "Compare with Saved Version" (diff working buffer vs. on-disk) |
| Compare with clipboard | Command Palette → "Compare with Clipboard" |
| Navigate changes | `F7` / `Shift+F7` to jump between diff hunks |
| Merge direction | Click gutter arrow to copy a hunk from left→right or right→left |
| Syntax highlighting | Both sides retain full syntax highlighting |
| Ignore whitespace | Toggle to ignore whitespace-only changes |
| Ignore case | Toggle for case-insensitive comparison |
| Line-level + word-level | Highlight changed words within changed lines (word-level diff) |
| Large file support | Stream-diff for files > 10MB; show first N differences with "load more" |

### Core API
```rust
pub fn compute_diff(old: &Rope, new: &Rope, options: DiffOptions) -> Vec<DiffHunk>;

pub struct DiffHunk {
    pub kind: DiffKind,          // Added, Removed, Modified
    pub old_range: Range<usize>, // line range in old
    pub new_range: Range<usize>, // line range in new
    pub word_diffs: Vec<WordDiff>, // intra-line changes
}
```

### Platform UI
- Diff gutter: colored bars (green = added, red = removed, blue = modified) in the gutter area
- Synchronized scrolling: both panes scroll together (with toggle to unlock)
- Dark mode: diff colors must be visible in both light and dark themes (use background tint, not text color alone — accessibility)

---

## INTERNATIONALIZATION (i18n)

### Rules (numbered for reviewer citation)

| # | Rule | Grep verification |
|---|------|-------------------|
| 55 | **No hardcoded UI strings in platform code** — all user-visible text in resource bundles | macOS: `grep -rn 'NSLocalizedString\|\.strings' platforms/macos/` — expect hits; `grep -rn '"[A-Z][a-z].*"' platforms/macos/ --include='*.swift' \| grep -v test \| grep -v '//'` — review for hardcoded English |
| 56 | **No hardcoded strings in core** — error messages are error codes; platform maps to localized strings | `grep -rn 'EditorError' core/src/error.rs` — must use enum variants not string messages for user display |
| 57 | **Resource bundles exist per platform** | macOS: `Localizable.strings`, Android: `strings.xml`, Windows: `.resx`, Linux: `.po` |
| 58 | **Date/time uses platform-native formatters** — no manual date string construction | `grep -rn 'DateFormatter\|DateTimeFormatter\|CultureInfo\|chrono.*format' platforms/` — expect hits; `grep -rn 'strftime\|manual.*date\|"yyyy' platforms/` — expect 0 |

### Strategy
- **UI strings**: externalized into per-platform resource bundles (`.strings` on Apple, `strings.xml` on Android, `.resx` on Windows, gettext `.po` on Linux)
- **Default language**: English (en)
- **Initial supported languages**: English only for v1.0; architecture must support adding languages without code changes
- **RTL text editing**: the core rope and rendering must handle bidirectional text (Unicode BiDi algorithm). Full RTL UI layout is deferred to a future release.
- **Date/time formatting**: use platform-native formatters (`DateFormatter` on Apple, `DateTimeFormatter` on Android/Kotlin, `CultureInfo` on .NET, `chrono` with locale on Linux)

---

## DARK MODE (required on ALL platforms)

Every platform must respect system appearance preference and support manual override:

| Platform | Mechanism |
|----------|-----------|
| macOS | `NSAppearance` / system preference |
| iOS | `UIUserInterfaceStyle` / `preferredColorScheme` |
| Linux | `libadwaita` `StyleManager` / `prefers-color-scheme` |
| Windows | `ElementTheme` / `Application.RequestedTheme` |
| Android | Material 3 `isSystemInDarkTheme()` |

Syntax highlighting themes must provide both light and dark variants.

---

## LOGGING

| Platform | Framework |
|----------|-----------|
| Rust Core | `tracing` crate with `tracing-subscriber` |
| macOS/iOS | `os_log` (unified logging) via `OSLog` |
| Android | `android.util.Log` / Timber |
| Windows | `ILogger` / Debug output |
| Linux | `tracing` (same as core, since it's Rust) |

- Log levels: `error` (user-visible failures), `warn` (recoverable issues), `info` (operations), `debug` (development only)
- NEVER log file contents — privacy-sensitive
- Performance: logging must not block the UI thread

---

## TESTING

### Testing Strategy — macOS First (v1.0/v1.1)

**Docker CANNOT be used for macOS GUI testing.** Docker on Mac runs Linux containers — no AppKit, no VoiceOver, no macOS display server. macOS native testing requires a real macOS environment.

#### Test Layers

```
┌─────────────────────────────────────────────────────────┐
│ Layer 4: Manual QA (VoiceOver, visual, performance)     │  ← Your Mac (M4)
├─────────────────────────────────────────────────────────┤
│ Layer 3: XCUITest (UI automation)                       │  ← Your Mac + CI (GitHub Actions macos-14)
├─────────────────────────────────────────────────────────┤
│ Layer 2: XCTest (Swift unit tests)                      │  ← Your Mac + CI
├─────────────────────────────────────────────────────────┤
│ Layer 1: cargo test + cargo bench + cargo fuzz          │  ← Your Mac + CI + Docker Linux (core only)
└─────────────────────────────────────────────────────────┘
```

#### Layer 1: Rust Core (can run ANYWHERE — including Docker)
```bash
# Local (your Mac)
cd core && cargo test                    # Unit + integration tests
cd core && cargo test --release          # Verify no debug-only behavior
cd core && cargo bench                   # Performance benchmarks (criterion)
cd core && cargo fuzz run fuzz_encoding -- -max_total_time=60  # Fuzz testing

# Docker (for CI / cross-platform core validation)
docker run --rm -v $(pwd)/core:/workspace -w /workspace rust:latest cargo test
docker run --rm -v $(pwd)/core:/workspace -w /workspace rust:latest cargo clippy -- -D warnings
docker run --rm -v $(pwd)/core:/workspace -w /workspace rust:latest cargo fmt --check
```

**Docker IS useful for**: running Rust core tests in a clean Linux environment, testing cross-compilation, running `cargo audit`, fuzzing in CI.

#### Layer 2: Swift Unit Tests (macOS only)
```bash
# Run from Xcode or command line
cd platforms/macos
xcodebuild test -scheme MyNotepadPP -destination 'platform=macOS'

# Specific test class
xcodebuild test -scheme MyNotepadPP -destination 'platform=macOS' -only-testing:MyNotepadPPTests/EditorCoreTests
```

What to test:
- `EditorCore` FFI bridge: open/close/read/write via Rust core
- `ThemeManager`: loading themes, dark/light switching
- `FileService`: encoding detection, line ending handling
- `KeyBindingManager`: shortcut resolution, chord sequences
- Macro serialization/deserialization
- Error mapping: `FfiResult` codes → Swift errors

#### Layer 3: XCUITest UI Automation (macOS only)
```bash
xcodebuild test -scheme MyNotepadPPUITests -destination 'platform=macOS'
```

What to test:
- Tab creation, switching, closing, reordering
- Split view creation and navigation
- Find & Replace: text search, regex, replace all
- Command Palette: open, fuzzy search, execute command
- Multi-cursor: add cursor, type at all, undo
- File open/save dialog flow
- Keyboard shortcuts (Cmd+S, Cmd+N, Cmd+W, etc.)
- Dark mode toggle
- Encoding/line-ending switching via status bar
- Auto-save behavior (modify, wait, verify saved)

#### Layer 4: Manual Testing (macOS only — cannot be automated)
| Test | How | Frequency |
|------|-----|-----------|
| VoiceOver navigation | Turn on VoiceOver (Cmd+F5), navigate editor by line/word/character, verify announcements | Every UI change |
| Performance (60 FPS) | Open 100MB file, scroll rapidly, check with Instruments → Time Profiler | Every rendering change |
| Memory (< 50MB idle) | Open 1 file, check Activity Monitor | Weekly |
| Cold startup (< 500ms) | Quit app, `time open MyNotepadPP.app`, measure to first frame | Every startup-path change |
| Large file (1GB) | Open a 1GB log file, verify progressive loading, no crash | After buffer changes |
| Crash recovery | Force-quit during editing, relaunch, verify hot exit restoration | After session code changes |
| Finder integration | Drag file onto app icon, Open With context menu, Spotlight | After file handling changes |

#### CI Setup (GitHub Actions)
```yaml
# .github/workflows/ci.yml
jobs:
  rust-core:
    runs-on: ubuntu-latest  # or Docker
    steps:
      - uses: actions/checkout@v4
      - run: cd core && cargo fmt --check
      - run: cd core && cargo clippy -- -D warnings
      - run: cd core && cargo test
      - run: cd core && cargo test --release
      - run: cd core && cargo audit

  macos-app:
    runs-on: macos-14  # Sonoma + Apple Silicon runner
    steps:
      - uses: actions/checkout@v4
      - run: cd core && cargo build --release --target aarch64-apple-darwin
      - run: cd platforms/macos && xcodebuild test -scheme MyNotepadPP -destination 'platform=macOS'
      - run: cd platforms/macos && xcodebuild test -scheme MyNotepadPPUITests -destination 'platform=macOS'
      - run: swiftlint lint platforms/macos/
```

### Rust Core
- `cargo test` — all modules, run in CI
- `cargo test --release` — verify no debug-only behavior
- Benchmarks: `cargo bench` for rope operations, search, syntax parsing
- Fuzz: `cargo fuzz` targets for file loading, encoding detection, search input

### Swift (Mac + iOS)
- XCTest for unit tests
- XCUITest for UI automation
- Test on both Apple Silicon and (if CI supports) Intel via Rosetta

### Kotlin (Android)
- JUnit 5 for unit tests
- Compose UI tests (`createComposeRule`)
- Instrumented tests on API 26 and latest

### C# (Windows)
- xUnit for unit tests
- WinUI test infrastructure for UI tests

### Linux (Rust + GTK)
- `cargo test` (shared with core tests where applicable)
- GTK widget tests where feasible

---

## FILE NAMING (Summary)

| Layer | Convention |
|-------|-----------|
| Rust | `snake_case.rs` |
| Swift | `PascalCase.swift` |
| Kotlin | `PascalCase.kt` |
| C# | `PascalCase.cs` / `PascalCase.xaml` |
| Config | `snake_case.toml` / `snake_case.json` |
| Docs | `SCREAMING_CASE.md` (e.g., `README.md`, `CONTRIBUTING.md`) |

---

## BUILD & CI

### macOS Build Setup (PRIMARY — must work first)

#### Rust Core → Static Library for macOS
```bash
# Build Rust core as static library for Apple Silicon
cargo build --release --target aarch64-apple-darwin
# Output: target/aarch64-apple-darwin/release/libmynotepadpp_core.a

# Generate C headers from Rust FFI
cargo install cbindgen
cbindgen --config core/ffi/cbindgen.toml --crate mynotepadpp-core-ffi --output core/ffi/include/mynotepadpp_core.h
```

#### Xcode Project Structure
```
platforms/macos/
├── MyNotepadPP.xcodeproj/
├── MyNotepadPP/
│   ├── App/
│   │   ├── AppDelegate.swift
│   │   └── main.swift
│   ├── Core/
│   │   ├── EditorCore.swift          ← FFI bridge (opaque pointer lifecycle)
│   │   ├── EditorError.swift         ← Maps FfiResult to Swift errors
│   │   └── BridgingHeader.h          ← #include "mynotepadpp_core.h"
│   ├── Views/
│   │   ├── EditorView.swift          ← CANONICAL REFERENCE for views
│   │   ├── TabBarView.swift
│   │   └── SidebarView.swift
│   ├── Services/
│   │   ├── FileService.swift         ← CANONICAL REFERENCE for services
│   │   └── ThemeManager.swift
│   ├── ViewModels/
│   └── Resources/
│       ├── Assets.xcassets/
│       ├── MainMenu.xib
│       └── Localizable.strings       ← i18n resource bundle
├── MyNotepadPPTests/
│   └── *.swift                        ← XCTest unit tests
├── MyNotepadPPUITests/
│   └── *.swift                        ← XCUITest UI automation
└── MyNotepadPP.entitlements           ← Hardened Runtime + App Sandbox
```

#### Xcode Configuration Requirements
| Setting | Value | Why |
|---------|-------|-----|
| Deployment Target | macOS 14.0 | Sonoma minimum |
| Architectures | arm64 (dev), arm64 + x86_64 (release) | Apple Silicon primary |
| Hardened Runtime | Enabled | Required for notarization |
| App Sandbox | Enabled | Security — file access via Open/Save panels |
| Signing | Development cert (local), Distribution cert (release) | Notarization |
| Other Linker Flags | `-lmynotepadpp_core` | Link Rust static library |
| Header Search Paths | `$(PROJECT_DIR)/../../core/ffi/include` | cbindgen output |
| Library Search Paths | `$(PROJECT_DIR)/../../target/aarch64-apple-darwin/release` | Rust build output |
| Bridging Header | `MyNotepadPP/Core/BridgingHeader.h` | Expose C FFI to Swift |

#### Build Phases (Xcode)
1. **Run Script — Build Rust Core**: `cd ../../core && cargo build --release --target aarch64-apple-darwin`
2. **Run Script — Generate Headers**: `cd ../../core/ffi && cbindgen --config cbindgen.toml ...`
3. **Compile Sources** (Swift)
4. **Link Binary** with `libmynotepadpp_core.a`
5. **Copy Bundle Resources** (themes, assets)

#### Entitlements (`MyNotepadPP.entitlements`)
```xml
<dict>
    <key>com.apple.security.app-sandbox</key><true/>
    <key>com.apple.security.files.user-selected.read-write</key><true/>
    <key>com.apple.security.files.bookmarks.app-scope</key><true/>
    <key>com.apple.security.network.client</key><true/>  <!-- SFTP only, opt-in -->
</dict>
```

### Build Commands (All Platforms)

| Platform | Command | Artifact |
|----------|---------|----------|
| Rust Core | `cargo build --release` | `libmynotepadpp_core.a` (macOS/iOS) / `libmynotepadpp_core.so` (Linux/Android) / `mynotepadpp_core.dll` (Windows) / `libmynotepadpp_core.dylib` (macOS dev) |
| Mac | `xcodebuild -scheme MyNotepadPP -configuration Release` | `.app` bundle |
| iOS | `xcodebuild -scheme MyNotepadPP-iOS -destination 'platform=iOS'` | `.ipa` |
| Linux | `cargo build --release -p mynotepadpp-gtk` | Binary + `.desktop` + Flatpak |
| Windows (x64) | `dotnet build -c Release -r win-x64` | MSIX package |
| Android | `./gradlew assembleRelease` | `.apk` / `.aab` |

### Packaging — Library Placement in App Bundles

| Platform | Core library location | Format |
|----------|----------------------|--------|
| macOS | `MyNotepadPP.app/Contents/Frameworks/libmynotepadpp_core.dylib` (or linked statically into binary) | `.a` (static, preferred) or `.dylib` |
| iOS | Embedded in app binary (static link) | `.a` |
| Linux | Same binary (Rust crate dep, no FFI) | N/A |
| Windows | `MyNotepadPP/mynotepadpp_core.dll` alongside `.exe` | `.dll` (x64) |
| Android | `app/src/main/jniLibs/{abi}/libmynotepadpp_core.so` | `.so` per ABI (arm64-v8a, armeabi-v7a, x86_64) |

### CI Requirements
- All platforms build on every PR (use GitHub Actions matrix)
- `cargo clippy`, `cargo fmt --check`, `cargo test`, `cargo audit` gate merges
- SwiftLint, ktlint/detekt, .NET analyzers gate respective platform code
- Cross-compile Rust core for all targets in CI
- macOS CI runner: `macos-14` (Sonoma) with Xcode 15+

---

## DISTRIBUTION

| Platform | Channel | Cost |
|----------|---------|------|
| macOS | GitHub Releases (`.dmg`) + Homebrew Cask | Free (notarization requires Apple Developer $99/yr) |
| Linux | Flathub + GitHub Releases (AppImage) | Free |
| Windows | GitHub Releases (`.msix`) + winget | Free |
| iOS | App Store (free) + TestFlight | Apple Developer $99/yr |
| Android | Google Play (free) + F-Droid + GitHub APK | Google Play $25 one-time |

---

## CONTRIBUTING

- All contributions must be GPL v3 compatible
- DCO (Developer Certificate of Origin) sign-off on commits
- PR template: what, why, which platforms affected, how to test
- One feature = one PR. Don't bundle unrelated changes.
