# Global Coding Standards — MYNOTEPAD++

A cross-platform native text editor (GPL v3). Built from scratch for Mac, Linux, Windows, iOS, and Android.

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

| Shaped text output | In-memory per (content, style, font) | On content/font change |
| Line layout | In-memory per line | On line content change |
| Font metrics | In-memory per (font, size) | On font change |
| Compiled regex patterns | LRU cache (50 entries) | LRU eviction |
| Mount point capabilities | In-memory per mount point | On app restart |

### Rules
- Never cache file contents beyond the rope buffer — the rope IS the cache
- Grammar loading must not block first render — show plain text, overlay highlighting when ready
- Theme hot-reload: file-watch `themes/` directory, reload without restart
- **Derived data invalidation**: each derived computation (syntax tree, fold regions, symbol index, bracket pairs) records which rope generation it was computed from. Recompute only when generation advances. This avoids redundant recomputation when multiple operations query the same derived data.

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
- **Strings cross FFI as length-delimited `(*const u8, usize)` ONLY** — never `*const c_char` (null-terminated). Null-terminated strings allow embedded-null path traversal attacks and C string truncation mismatches between Rust/Swift.
- **Rust allocates all output buffers** — never use caller-allocated buffers for variable-length output. Return a `(*const u8, usize)` pair from Rust with a matching `*_free_string()` function. Caller-allocated buffers risk buffer overflow (e.g., encoding conversion expands Shift-JIS to UTF-8 by up to 3x).
- **Handle registry**: FFI layer maintains a `HashSet<usize>` of live handle addresses. Every FFI call validates the pointer is in the live set before dereferencing. Prevents use-after-free and double-free across the FFI boundary.
- **Crash symbolication**: set `split-debuginfo = "packed"` and `force-frame-pointers = true` in `[profile.release]` in `Cargo.toml`. Include Rust `.dSYM` in symbol server for mixed Rust/Swift crash report symbolication.

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
- **NSTextInputClient (CRITICAL)**: the EditorView MUST implement `NSTextInputClient` protocol for: IME composition (CJK input methods), emoji picker (`Cmd+Ctrl+Space`), dictation, system text replacement, and character palette. This includes `markedText` management, `firstRect(forCharacterRange:)` for candidate window positioning, `characterIndex(for:)` for point-to-character mapping. Call `inputContext?.invalidateCharacterCoordinates()` on scroll or layout change. Without this, CJK input, emoji, and dictation are completely broken.
- **NSWritingToolsCoordinator (macOS 15+)**: implement `NSWritingToolsCoordinator.Delegate` for Writing Tools integration (AI proofreading). Without this, the right-click context menu is missing Writing Tools on Sequoia+. Provide text context as `NSAttributedString`, handle replacement callbacks.
- **Text shaping**: use CoreText (macOS/iOS), HarfBuzz (Linux), DirectWrite (Windows) for text shaping — handles ligatures (`->`, `=>`, `!=` in Fira Code), kerning, combining characters, and complex scripts (Arabic, Devanagari, CJK). Variable font axis interpolation supported.
- **Glyph atlas**: rasterize each glyph once per `(glyph_id, font_id, size, subpixel_offset)` tuple → cache in a GPU texture atlas → reuse across frames. Color applied via shader, NOT baked into atlas. Invalidate only on font change. **Emoji/color glyphs**: use a separate RGBA texture atlas for color emoji (`sbix` format). Use `CTFontDrawGlyphs()` for rendering — standard grayscale atlas does NOT work for emoji.
- **Glyph atlas eviction**: LRU page eviction when atlas exceeds 32MB. CJK text can produce thousands of unique glyphs — eviction prevents unbounded growth.
- **Shaped text cache**: cache shaped output per (text_content, style, font) tuple. Segment by word/token boundaries for maximum reuse. LRU cache of 10,000 entries.
- **Line layout cache**: cache each visible line's layout (character positions, advances, wrap points). Invalidate only when that specific line's content changes — never re-layout all 50 visible lines for a single-line edit.
- **Font metric cache**: cache ascent, descent, line height, advance widths per font/size at font load time. CoreText calls for metrics are not free.
- **Dirty rect rendering**: on edit, only re-render changed lines + cursor line. On scroll, only render newly visible lines. Never redraw entire viewport.
- **Overdraw buffer**: pre-render 2× viewport height (1 screen above + 1 below) so fast scrolling never shows blank.
- **Viewport culling**: only compute layout for visible lines + overdraw buffer. A 1GB file with 20M lines must NOT compute layout for all lines — use a line-offset index (cumulative heights) built lazily.

### 2. Thread Architecture
```
┌─────────────────────────────────────────────────────┐
│ MAIN THREAD (UI only — NEVER blocks)                │
│  AppKit events, rendering, user input, cursor       │
│  Max budget per frame: 16ms (60 FPS)                │
│  QoS: .userInteractive                              │
├─────────────────────────────────────────────────────┤
│ RAYON THREAD POOL (CPU-bound — Rust core)           │
│  Syntax parsing, search, diff, encoding detection   │
│  Uses rayon::ThreadPool, work-stealing scheduler    │
│  Pool size: P-core count (6 on M4)                  │
│  Priority lanes: HIGH (visible viewport syntax)     │
│                  NORMAL (search, diff)               │
│                  LOW (background indexing)           │
│  QoS: HIGH → .userInitiated                         │
│        NORMAL → .utility                            │
│        LOW → .background                            │
├─────────────────────────────────────────────────────┤
│ LOCAL I/O THREAD POOL (2 threads)                    │
│  Local file read/write, auto-save                   │
│  30-second timeout per operation                    │
│  QoS: .utility                                      │
├─────────────────────────────────────────────────────┤
│ NETWORK I/O THREAD POOL (2 threads) — v1.1 only     │
│  SFTP/FTPS operations                               │
│  30-second timeout per operation                    │
│  Separate from local I/O — network stalls never     │
│  block local file saves                             │
│  QoS: .utility                                      │
├─────────────────────────────────────────────────────┤
│ SQLITE WRITER THREAD (dedicated, 1 thread)          │
│  All INSERT/UPDATE/DELETE to session DB              │
│  Serialized writes via mpsc channel                 │
│  Never shares with file I/O (no starvation)         │
│  QoS: .utility                                      │
├─────────────────────────────────────────────────────┤
│ PLUGIN THREAD (per-plugin, isolated)                │
│  WASM execution with fuel-based CPU limits          │
│  Killed on timeout (100ms per event)                │
│  QoS: .background                                   │
└─────────────────────────────────────────────────────┘
```

**Rules:**
- Main thread does ZERO file I/O, ZERO computation, ZERO blocking
- All core function calls from UI go through async channels → result posted back to main thread
- `rayon` for CPU-bound parallelism (syntax parsing, multi-file search, diff) — **two separate pools**: HIGH pool (4 threads on M4, `QOS_CLASS_USER_INITIATED` via `pthread_set_qos_class_self_np` in rayon `spawn_handler`) for visible viewport work, LOW pool (`QOS_CLASS_UTILITY`) for background indexing/search. Rayon does NOT automatically set QoS — you MUST configure it manually per thread.
- I/O thread pool (2-4 threads) for file operations — if one thread stalls on network FS, others continue
- SQLite gets its own dedicated thread — never blocked by file I/O stalls
- Every I/O operation has a 30-second hard timeout — stalled operations are abandoned, not retried indefinitely
- **Watchdog**: lightweight check every 5s for stuck I/O operations. If stuck, post warning to UI.
- **Thermal throttling**: check `ProcessInfo.thermalState` — reduce parallelism at `.serious` or `.critical`
- **App Nap prevention**: assert `NSActivityUserInitiated` while documents are dirty (prevents timer throttling)

### 3. Concurrent Document Access (Copy-on-Write Rope)
- **One logical `Rope` per document** — mutations produce new `Arc<Rope>` snapshots via structural sharing (copy-on-write persistent B+ tree)
- **No locks for reads**: background threads (syntax parsing, search, auto-save) hold immutable `Arc<Rope>` snapshots — zero contention with the writer
- **Writer produces new snapshot**: edit operations create a new root node sharing >99% of internal nodes with the previous version. O(log n) per edit.
- **Single document owner (DocumentManager)**: the main thread is the SOLE writer for each document. Multiple views/windows send edit commands to the DocumentManager; it applies them sequentially and broadcasts new `Arc<Rope>` to all views. This eliminates race conditions between multi-window edits.
- **Split views share the same logical document** — each view holds its own cursor/scroll state + the latest snapshot `Arc<Rope>`
- **Snapshot lifecycle**: main thread publishes new snapshots via `Arc::swap`. Background threads hold their snapshot for the duration of their operation, then drop it. Old nodes freed when refcount hits zero. **Snapshot age limit**: if a background thread's snapshot is > 500ms old and the rope has advanced, cancel and re-snapshot (bounds divergence and memory).
- **Large rope deallocation**: if the main thread drops the last `Arc` to a large rope (e.g., closing a tab for a 1GB file), send the `Arc` to a background thread for deallocation to avoid blocking the main thread.
- **Per-file save serialization**: only ONE save operation per file path at a time. If auto-save is in-flight and `Cmd+S` arrives, cancel the auto-save (`CancelToken`) and supersede with the manual save. Use a `DashMap<PathBuf, CancelToken>` to track active saves.
- **Why not `RwLock<Rope>`**: RwLock causes contention when auto-save, search, and syntax parsing all compete for read locks during editing. Copy-on-write eliminates this entirely — no thread ever waits on another.
- **Why not CRDT**: CRDTs add massive complexity for single-user editing and are not justified until collaborative editing becomes a requirement in v3.0.

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
Timer fires → I/O pool: take rope snapshot (Arc clone, instant) → serialize to temp → atomic rename → done
```

**Two independent save systems run in parallel:**

#### A. Auto-Save (to original file location)
Saves the file to its original path. This is what the user sees as "saved."

#### B. Continuous Backup (to backup directory)
Independent safety net — writes every 500ms to `~/Library/Application Support/mynotepadpp/backups/{doc_id}/`. Survives SIGKILL, force quit, power loss. Max data loss: 500ms of typing.

**Three-tier atomic write procedure (with fallback):**
```
Tier 1: Atomic rename (preferred)
  1. Write to {dir}/{filename}.mynotepadpp-{pid}-{timestamp}.tmp (SAME directory)
  2. fsync() the temp file (F_FULLFSYNC on macOS, fdatasync on Linux)
  3. rename() temp → target (atomic on POSIX)
  4. Suppress file watcher for 500ms
  → If rename fails (EXDEV cross-device, NFS, SMB): fall to Tier 2

Tier 2: Direct overwrite with backup
  1. Copy target to {dir}/{filename}.mynotepadpp-bak
  2. Truncate + write target directly
  3. fsync() target
  4. Delete backup
  → If write fails (EACCES, EROFS, ENOSPC): fall to Tier 3

Tier 3: Recovery directory write
  1. Write to recovery directory
  2. Show non-blocking notification: "Could not save to original location."
  3. Keep buffer marked dirty
```

**Mount detection at startup:** On first save to any path, test same-directory rename. Cache result per mount point. Skip Tier 1 for known-bad mounts (NFS, SMB).

**fsync timeout:** 10 seconds. On timeout, log warning and proceed — rename provides atomicity (app crash safe), fsync provides durability (power loss safe). Skipping fsync in degraded mode is acceptable.

**Cloud-aware save behavior:**
- Detect iCloud Drive (`~/Library/Mobile Documents/`), Dropbox, Google Drive, OneDrive directories
- For cloud-synced dirs: use Tier 2 (direct overwrite) instead of Tier 1 (Dropbox handles in-place writes better than delete+create)
- macOS iCloud: use `NSFileCoordinator` for all writes (required for correct iCloud behavior)
- Increase debounce to 3s for cloud dirs (reduce sync churn)

**Failure handling — disk full:**
- Attempt recovery directory → if also full → attempt `/tmp` → if also full:
- Keep buffer in memory. Set state to `DIRTY_CRITICAL`.
- Show persistent banner: "Disk full. Changes held in memory. Free space to save."
- Poll disk space every 10s. Auto-retry when space appears.
- On Cmd+Q with `DIRTY_CRITICAL`: show ONE blocking dialog: "Cannot save N files (disk full). Quit anyway?" — this is the SOLE exception to "never prompt."

**Debounce interaction:**
- `delayAfterTyping` (1s): debounce — resets on every keystroke. Fires 1s after last keystroke.
- `timerInterval` (30s): throttle — fires every 30s regardless of typing activity. Acts as safety net.
- `onFocusLost`: immediate save when window loses focus.
- All three are OR'd: whichever triggers first wins. After a save, all timers reset.

**Continuous backup cleanup:**
- On successful save to original: delete corresponding backup file
- On startup: scan backups/ — restore if newer than original, delete if stale (>7 days)
- Temp file naming: `{filename}.mynotepadpp-{pid}-{timestamp}.tmp` — orphans detectable by dead PID
- Cleanup on startup + every 1 hour: delete orphaned temp files older than 1 hour

**Auto-save race condition with file watcher:**
- **Use `kFSEventStreamEventFlagOwnEvent`** (macOS API) to detect events caused by our own process — most reliable suppression method.
- Fallback (if flag unavailable): store the mtime we wrote. On file watcher event, compare current mtime to our last write mtime. If they match, suppress. If they differ, it was an external change — do NOT suppress.
- **Do NOT use a boolean flag** — a boolean flag can incorrectly suppress genuine external changes that arrive during the 500ms window.

### 7. Shutdown & Close Behavior (NEVER prompts, NEVER loses data)

**Close tab (`Cmd+W`):**
- If auto-save is enabled (default): save silently → close immediately. No prompt. < 50ms.
- If auto-save is disabled: for named files, save silently → close. For untitled files, prompt "Save as..." (only case where a prompt appears).

**Close window (not last window):**
- Save this window's session state to SQLite **synchronously on window close** — don't wait for app quit.
- If auto-save enabled: save all dirty tabs silently → close immediately.

**Close last window (macOS: app stays running):**
- Save session state to SQLite immediately (same as above).
- App remains in Dock with no windows (standard macOS behavior).
- On reopen (click Dock): restore from SQLite.

**Quit app (`Cmd+Q`):**
- Set `isShuttingDown = true` guard flag (rejects new file opens during shutdown).
- Hot exit saves ALL state in ONE `BEGIN IMMEDIATE` SQLite transaction:
  - Tab states: ~5ms for 50 UPSERTs (prepared cached statements)
  - Unsaved content for untitled files: zstd-compressed, ~50ms
  - Undo history (last 1000 ops, zstd-compressed): ~100ms
  - Window geometry: <1ms
  - COMMIT + fsync: ~30ms
  - WAL checkpoint(TRUNCATE): ~20ms
  - **Total: ~200-250ms** (well within 500ms budget)
- For named files with unsaved changes: delta-based — store edit operations (few KB), not full buffer (avoids serializing 1GB files)
- Then terminate immediately. **No confirmation dialog. No waiting.**
- If drag-and-drop arrives during shutdown: queue file path for next launch.

**System shutdown / SIGTERM:**
- Register `NSApplicationWillTerminate` (macOS) — OS grants ~5 seconds
- Same hot-exit sequence as Cmd+Q
- `NSProcessInfo.disableSuddenTermination()` while any document has unsaved changes
- `NSProcessInfo.enableSuddenTermination()` after all saves complete
- SIGTERM handler in Rust core: set atomic flag → SQLite writer thread flushes → exit
- If hot exit exceeds 3 seconds: abort undo history serialization, save only file contents + tab list

**SIGKILL / Force Quit (cannot catch):**
- Continuous backup system (§6B) ensures max 500ms data loss
- SQLite WAL auto-recovers committed transactions on next open
- On next launch: detect crash via `dirty_flag`, scan backups/, offer recovery

**Background save + Cmd+Q race condition:**
- State machine per document: `Clean | Dirty | Saving { cancel } | HotExiting`
- Hot exit ALWAYS wins: if auto-save is in progress, cancel it via CancelToken, take over
- Hot exit writes to backup dir (not original file) — no conflict with in-progress temp file

**iOS lifecycle (jetsam can kill without warning):**
- Save ALL dirty buffers on `sceneWillResignActive` (synchronous, 2s timeout)
- `beginBackgroundTask` in `sceneDidEnterBackground` for extended save (25s)
- On `didReceiveMemoryWarning`: flush all dirty buffers immediately, then release caches

**Multi-window coordination:**
- Each window saves its own session state to SQLite (keyed by window ID) on `windowWillClose`
- Same-file-in-two-windows: shared rope buffer via `Arc<Rope>` (same process). No conflict.
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

### Memory Allocator
- Use **mimalloc** as global allocator: `#[global_allocator] static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;`
- rust-analyzer saved 15s switching to mimalloc. 5x faster than glibc malloc under heavy multithreaded workloads.
- Use **bumpalo** arena allocator for per-parse-pass allocations (tree-sitter integration, per-search-query results). Allocate into arena, drop entire arena when done.
- Benchmark against system allocator on Apple Silicon (macOS libmalloc is already good on arm64) — keep mimalloc only if measurably better.

### SIMD Optimizations
- **UTF-8 validation**: `simdutf8` crate — 10x faster than scalar (0.45 cycles/byte)
- **Line counting**: `bytecount` crate — SIMD-accelerated newline counting for building line-offset index
- **Encoding detection**: vectorized null-byte scan (detects UTF-16 and binary instantly)
- **Multi-pattern search**: Teddy algorithm (from Hyperscan) via `regex-automata` — SIMD-packed 16-byte comparison for multi-literal matching
- **Rope SIMD**: enable `simd` feature flag on ropey crate (if used) for improved rope operations
- All SIMD paths must have scalar fallbacks for architectures without SIMD support

### Rust Core
- **File loading pipeline**:
  1. Read first 8KB: detect binary (null bytes > 0.1%), BOM, encoding
  2. For ALL files: `BufReader` in 64KB chunks → build rope on I/O pool thread (visible viewport first for files > 1MB)
  3. **NO mmap for user files** — mmap causes unrecoverable SIGBUS on truncated files, network filesystem disconnects, and has no error handling path. `BufReader` is only ~33% slower and fully safe. (mmap may be used for read-only internal data like grammar files within the app bundle)
  4. Show first screenful on main thread while rest loads in background
  5. Encoding detection order: BOM → XML/HTML charset → `encoding_rs` statistical → user locale fallback
  6. Line ending detection during first chunk (LF/CRLF/CR). Preserve or normalize based on preference.
  7. Memory budget: max 3 concurrent file loads during session restore
- **Long line handling**: lines > 10,000 chars get column-level viewport virtualization — only compute layout for visible columns ± overdraw. Syntax highlighting chunked to visible range + 500 chars. Word wrap computed lazily per viewport region. Lines > 500,000 chars: disable syntax highlighting for that line.
- **Line-offset index**: cumulative line heights stored in the rope's own tree structure via summary annotations (augmented B+ tree pattern). O(log n) lookup along any dimension (byte→line, line→byte, pixel→line) from a single data structure.
- **Incremental syntax parsing**: tree-sitter `edit()` + `parse()` on change, not full reparse. Parse on rayon pool (HIGH priority lane). 30ms debounce after edit. Each parse result carries a revision number matching its rope snapshot — discard stale results. Cancel-and-restart on new edit during parse.
- **Embedded language parsing**: tree-sitter language injection for HTML+JS+CSS, Markdown+code blocks. Orchestration: identify language ranges from parent grammar, parse each range with appropriate sub-grammar.
- **Syntax highlighting priority**: visible viewport first → ±1 screen overdraw → rest of file. Cancel pending highlights if user scrolls.
- **Search**: `memchr` crate for literal byte search, `regex-automata` (not `regex`) for regex with lazy DFA construction, `rayon` for multi-file parallelism. Use `ignore` crate for `.gitignore`-aware file walking. Incremental in-file search: as user types, refine previous results. Stream results to UI batched per-file with 16ms flush timer. Cache compiled regex patterns (LRU, last 50). For unopened files: mmap for zero-copy scanning.
- **Binary file detection**: scan first 8KB for null bytes (>0.1% = binary) + magic byte signatures (PNG, JPEG, PDF, ZIP, ELF, Mach-O). Show warning, don't attempt syntax highlight.
- Symbol indexing: tree-sitter `symbols` query on background thread. Index built lazily per-file on first Goto Symbol. Incremental update on file edit.
- Undo/redo: with copy-on-write rope, undo can be a **stack of rope snapshots** (each shares >99% of nodes via `Arc`). Undo is O(1) — swap pointer. Keep operation grouping for macro playback / replace-all.
- Startup: lazy-load syntax grammars on first use. Precompiled `.so`/`.dylib` tree-sitter grammars (not WASM) for speed. SQLite session read is async — show empty window, restore active tab first (<200ms), then remaining tabs progressively.
- **File watcher batching**: 200ms debounce + batch size cap. If `git checkout` changes 1000 files, coalesce into single "project changed" notification rather than 1000 individual events.

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
- Energy efficiency: use `QoS` classes (`.userInteractive` for rendering, `.userInitiated` for visible viewport syntax highlighting, `.utility` for search/auto-save, `.background` for file indexing/plugins). Never poll — use FSEvents, GCD timers, or `kqueue`.
- Thermal throttling: check `ProcessInfo.thermalState` — reduce rayon parallelism at `.serious` or `.critical`
- App Nap: assert `ProcessInfo.beginActivity(options: [.userInitiated])` while documents are dirty. Release assertion when all documents are saved.

### SQLite Fine-Tuning

**Minimum SQLite version:** 3.38.0+ (bundled via `rusqlite` with `bundled` + `modern_sqlite` features)

**Connection architecture:**
```
WRITE CONNECTION (1, dedicated SQLite writer thread)
  All INSERT/UPDATE/DELETE
  Uses BEGIN IMMEDIATE (avoids lock-upgrade failures)
  cache_spill = OFF (prevents reader blocking)

READ CONNECTION POOL (2-3, UI thread + background)
  All SELECT queries
  Opened with SQLITE_OPEN_READONLY flag
  Never blocked by writer (WAL guarantee)
```

**Write batching:** Cursor/scroll position updates batched in-memory (`HashMap<TabId, State>`), flushed to SQLite every **5 seconds** in a single transaction. Tab open/close written immediately.

**Prepared statement caching:** Use `prepare_cached()` exclusively. Cache capacity: 32 statements.

```sql
-- Set at DB creation (persistent, cannot change in WAL mode):
-- PRAGMA page_size = 4096;
-- PRAGMA auto_vacuum = INCREMENTAL;
-- PRAGMA application_id = 0x4D4E5050;    -- 'MNPP' — file identification

-- Set on EVERY connection open:
PRAGMA journal_mode = WAL;
PRAGMA synchronous = FULL;                -- NORMAL is unsafe on macOS (fsync is no-op for durability)
PRAGMA fullfsync = ON;                    -- force F_FULLFSYNC on macOS (true NVMe cache flush)
PRAGMA cache_size = -2000;                -- 2MB page cache
PRAGMA mmap_size = 33554432;              -- 32MB (not 256MB — sufficient for session DB)
PRAGMA temp_store = MEMORY;               -- temp tables in memory
PRAGMA busy_timeout = 5000;               -- 5s retry on lock
PRAGMA wal_autocheckpoint = 1000;         -- checkpoint every ~4MB of WAL
PRAGMA foreign_keys = ON;                 -- enforce FK constraints
PRAGMA trusted_schema = OFF;              -- security hardening
PRAGMA cell_size_check = ON;              -- early corruption detection
PRAGMA threads = 2;                       -- minor query parallelism
PRAGMA journal_size_limit = 67108864;     -- 64MB WAL size cap (prevents bloat)
PRAGMA optimize = 0x10002;                -- register for auto-optimize

-- Write connection only:
PRAGMA cache_spill = OFF;                 -- prevents blocking readers during writes
```

**Schema:** Use `STRICT` tables (SQLite 3.37+). Separate hot data (`tabs` — read on every tab switch) from cold data (`unsaved_content`, `undo_history` — written only on hot exit). Compress large blobs with zstd (3-5x reduction).

**On connection close:** `PRAGMA optimize;` (runs ANALYZE on tables that benefit)

**On clean exit:**
```sql
PRAGMA incremental_vacuum;            -- reclaim free pages
PRAGMA wal_checkpoint(TRUNCATE);      -- minimize DB file size
```

**Crash recovery on startup:**
1. Check `PRAGMA application_id` = `0x4D4E5050` (is this our DB?)
2. Check `dirty_flag` in `app_state` table (1 = crashed, 0 = clean exit)
3. If crashed: run `PRAGMA quick_check` (NOT `integrity_check` — 10-100x faster)
4. If corrupt: try `.recover`, else restore from backup, else nuke and recreate
5. Set `dirty_flag = 1` (will be set to 0 on clean exit)

**Periodic backup:** Every 10 minutes, `VACUUM INTO 'sessions.db.backup'` (SQLite 3.27+). If primary DB is corrupt on startup, restore from backup.

**Never place session DB on network filesystem** — WAL requires shared memory (`-shm` file) which doesn't work over NFS/SMB. Session DB must be in local Application Support directory only.

**Monitoring (debug builds):** Log `DBSTATUS_CACHE_MISS` and WAL file size. Warn if cache miss rate >10% or WAL >10MB.

---

## RESOURCE LIFECYCLE & MEMORY MANAGEMENT

### Rust FFI Resource Cleanup
- Every FFI-allocated resource has a `*_free()` function (Rule #7). Swift `deinit` calls it.
- **Explicit `close()` method**: `EditorCore` Swift class MUST have an explicit `close()` method called from tab-close logic. `deinit` is a safety net, NOT the primary cleanup path. This prevents leaks from retain cycles where `deinit` never fires.
- **Debug leak counter**: in debug builds, Rust FFI maintains an atomic counter (increment on `*_create`, decrement on `*_free`). Assert counter is zero on app termination.

### Metal GPU Resource Cleanup
- **Glyph atlas**: `MTLTexture` released on font change, theme change, and when app enters background. 32MB LRU cap with page eviction. Separate RGBA atlas for color emoji.
- **Command buffers**: `MTLCommandBuffer` completion handler MUST release drawable references. Never hold `CAMetalDrawable` across frames (Apple limit: 3 drawables).
- **Display scale change**: invalidate glyph atlas when `contentsScale` changes (window dragged between Retina and non-Retina displays).

### File Handle Cleanup
- **FSEvents watcher**: remove file paths from watch list on tab close. Remove project root from watch list on project close. Call `FSEventStreamStop` + `FSEventStreamInvalidate` when stream is no longer needed.
- **Security-scoped bookmarks**: `startAccessingSecurityScopedResource()` balanced with `stopAccessingSecurityScopedResource()` on tab close. Reference-count per directory (start on first file in dir, stop on last file close). Kernel limit ~256 active accesses — exceed this and ALL file access fails.
- **Temp file cleanup**: on startup, scan recovery/backup dirs, delete temp files whose PID is not running. On successful atomic save, temp file is renamed (no orphan). On crash, temp files persist — cleaned up on next launch.

### Cache Purge Under Memory Pressure
- Register `DispatchSource.makeMemoryPressureSource()` on macOS.
- On `.warning`: purge shaped text cache, drop syntax trees for non-visible tabs, reduce undo history to last 10 operations for non-active tabs.
- On `.critical`: purge glyph atlas to visible glyphs only, release all non-essential caches.
- **No iOS-style `didReceiveMemoryWarning` on macOS** — use `DispatchSource` memory pressure instead.

### Thread Shutdown Ordering
On quit (Cmd+Q / SIGTERM):
1. Set `isShuttingDown = true` (global atomic flag)
2. Cancel all background operations via CancelToken
3. I/O threads: check `isShuttingDown` on each loop — use 1-second timeout (not 30s) during shutdown
4. SQLite writer: flush pending writes, `wal_checkpoint(TRUNCATE)`, close connection
5. rayon pools: do NOT join — let OS reclaim threads on process exit
6. If step 3-4 exceeds 3 seconds: force `std::process::exit(0)` — critical data is already saved

### Swift 6 Strict Concurrency
- Enable `SWIFT_STRICT_CONCURRENCY = complete` in Xcode project
- `EditorCore` (FFI bridge): mark as `@unchecked Sendable` (wraps thread-safe Rust pointer)
- All UI code: `@MainActor` isolated
- Services (`FileService`, `ThemeManager`, `AutoSaveService`): declare explicit actor isolation
- Set `SWIFT_DEFAULT_ACTOR_ISOLATION = nonisolated` to prevent C FFI functions from being implicitly `@MainActor`

### App Nap & Automatic Termination
- `ProcessInfo.disableSuddenTermination()` while any document has unsaved changes
- `ProcessInfo.disableAutomaticTermination("unsaved changes")` while any document is dirty (prevents macOS from auto-terminating the app)
- `ProcessInfo.beginActivity(options: [.userInitiated], reason: "auto-save")` during save operations (prevents App Nap throttling)
- Release all assertions when all documents are clean

---

## SECURITY RULES

### All Platforms
- Never execute file contents — this is an editor, not a runtime
- **Path sanitization**: canonicalize ALL file paths (`realpath`) before use. Reject any path containing `..` before canonicalization. Verify resolved path is within expected directory (`starts_with()` check). Applies to: plugin loading, theme loading, recovery files, URL scheme handler, temp files.
- **URL scheme handler security**: `mynotepadpp://open?file=...` MUST show confirmation dialog ("Website wants to open [filename]. Allow?"). Reject paths to system directories (`/etc/`, `/System/`, `/Library/`). Rate-limit to 1 request/second. Validate file is regular file (not device/pipe/symlink to device).
- **BiDi attack protection**: warn on bidirectional text override characters (U+202A-U+202E, U+2066-U+2069) in source code files. Display as visible escape sequences to prevent Trojan Source attacks.
- Validate encoding detection before converting — malformed input must not crash
- Sandbox where platform supports it (macOS App Sandbox, Android scoped storage)
- No network access by default — editor works fully offline
- Plugin system (future): sandboxed execution, explicit permission grants
- **Recovery/backup file security**: set file permissions to `0600` on all recovery, backup, and SQLite files immediately after creation. Consider encrypting recovery files with AES-256 key stored in Keychain.
- **Clipboard history security**: respect `org.nspasteboard.TransientType` and `org.nspasteboard.ConcealedType` pasteboard flags. Do NOT store clipboard entries with these flags. Auto-expire clipboard history after 5 minutes. Never persist to disk. Plugins MUST NOT have access to clipboard history.

### Rust Core
- `#[forbid(unsafe_code)]` at crate root — lift to `#[allow(unsafe_code)]` only in specific modules with justification
- Fuzzing CI for all parser/input code
- No `std::process::Command` — core must never spawn processes
- Dependency audit: `cargo audit` in CI, deny `unmaintained` advisories

### Platform Specific
- macOS: Hardened Runtime enabled, notarized for distribution. **No `com.apple.security.cs.allow-jit` entitlement** — use wasmtime Pulley interpreter for plugins.
- macOS App Sandbox: **Integrated terminal (v1.1) is incompatible with App Sandbox.** Terminal must be excluded from Mac App Store build. Offer terminal only in direct-download (Homebrew/DMG) distribution, or use "Open in Terminal.app" as a safe alternative.
- macOS security-scoped bookmarks: implement reference counting for directory access (start on first file open in dir, stop on last close). Kernel limit of ~1000-2500 active accesses — exceeding this loses ALL file access until restart.
- macOS Xcode 26+: set `SWIFT_DEFAULT_ACTOR_ISOLATION=nonisolated` to prevent C FFI functions from being implicitly `@MainActor`-isolated (breaks background thread FFI calls).
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
- **CHANGELOG.md**: maintained in repo root using [Keep a Changelog](https://keepachangelog.com/) format. Sections: `Added`, `Changed`, `Fixed`, `Removed`, `Security`. Drives App Store "What's New" text.

### Store-Specific Version Formats

| Platform | Marketing Version | Build Number | Format | Source |
|----------|------------------|-------------|--------|--------|
| macOS | `CFBundleShortVersionString` | `CFBundleVersion` | `1.1.0` / `42` | Git tag / `${{ github.run_number }}` |
| iOS | `CFBundleShortVersionString` | `CFBundleVersion` | `1.1.0` / `42` | Same as macOS |
| Android | `versionName` | `versionCode` | `"1.1.0"` / `10100` | Git tag / `MAJOR*10000 + MINOR*100 + PATCH` |
| Windows | MSIX `Version` | — | `1.1.0.0` | Four-part (fourth always 0 for Store) |
| Linux | Flatpak release tag | — | `1.1.0` | Git tag |

**Rules:**
- `CFBundleVersion` (Apple) MUST strictly increase for every build uploaded to App Store Connect / TestFlight. Use CI build number.
- Android `versionCode` MUST strictly increase. Encoding: `MAJOR*10000 + MINOR*100 + PATCH` gives room for 99 minor and 99 patch per major.
- Git tag MUST match the marketing version exactly. The source code at the tag MUST correspond to the binary shipped to stores (GPL v3 compliance).
- CHANGELOG entry MUST exist for every version shipped to any store.

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
| Play last macro | `Cmd+Shift+E` | Replay most recent macro |
| Play N times | Command Palette → "Run Macro N Times" | Repeat macro with count |
| Manage macros | Preferences → Macros tab | List, rename, delete, re-assign shortcuts |

All platforms must expose equivalent functionality via their native UI patterns (menus, Command Palette, preferences).

---

## PLUGIN SYSTEM

### High-Level Architecture
Plugins extend editor functionality without modifying core code. The plugin system prioritizes **security** and **stability** over flexibility.

### Sandboxing
- Plugins run in a **WebAssembly (WASM) sandbox** using `wasmtime` with the **Pulley interpreter** backend (NOT Cranelift JIT). Pulley avoids the `com.apple.security.cs.allow-jit` entitlement required by JIT, which weakens Hardened Runtime. Pulley is ~10x slower than native compilation but acceptable given the 100ms CPU budget per event handler. This also eliminates the entire class of Cranelift miscompilation vulnerabilities (e.g., CVE-2026-34971 aarch64 sandbox escape).
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

### Connection Management
- **Connection pooling**: maintain 1-3 SSH sessions per host:port. Reuse sessions for multiple file operations.
- **Keep-alive**: send SSH keepalive packets every 30 seconds. Without this, firewalls/NATs silently drop idle connections after 60-120s.
- **SSH multiplexing**: one TCP connection, multiple SFTP channels. Avoids repeated key exchange overhead.
- **Reconnection**: on connection drop, queue pending operations. Auto-reconnect with exponential backoff (1s, 2s, 4s, max 30s).
- **Buffer tuning**: SFTP read/write buffer 256KB-1MB, request pipeline depth 64-128 for high-latency connections.
- **Connection lifecycle**: close pooled connections after 15 minutes of inactivity (configurable).

### Security
- All connections use TLS 1.2+ (FTPS) or SSH v2 (SFTP) — no fallback to insecure protocols
- Host key verification: first-connection prompt to trust, then pin in known_hosts
- Session timeout: configurable idle disconnect (default 15 minutes)

---

## DIFF / FILE COMPARISON VIEW

### Architecture
Diff computation runs in the Rust core using the `similar` crate. Algorithm selection:
- **Default**: Histogram (best for code — aligns function boundaries, handles repeated lines well)
- **User toggle**: Myers (fastest for common case), Patience (most human-readable for code)
- **Word-level**: Myers (fast enough for single lines)
- **Large files (>10MB)**: chunked approach — diff line hashes first (O(n)), then compute content diff only for changed regions

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
    <!-- v1.1 only: <key>com.apple.security.network.client</key><true/> (SFTP) -->
    <!-- v1.1 only: <key>com.apple.security.print</key><true/> (Print support) -->
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
| macOS | GitHub Releases (`.dmg`) + Homebrew Cask + Mac App Store | Free + Apple Developer $99/yr |
| Linux | Flathub + GitHub Releases (AppImage) | Free |
| Windows | GitHub Releases (`.msix`) + winget + Microsoft Store | Free |
| iOS | App Store (free) + TestFlight | Apple Developer $99/yr |
| Android | Google Play (free) + F-Droid + GitHub APK | Google Play $25 one-time |

### App Store Metadata (required for submission)

| Field | Apple (macOS/iOS) | Google Play | Notes |
|-------|-------------------|-------------|-------|
| App Name | 30 chars max | 30 chars max | "MYNOTEPAD++" (12 chars) |
| Description | 4000 chars | 4000 chars | Feature highlights, GPL notice, source link |
| Category | Developer Tools (primary), Productivity (secondary) | Productivity / Tools | — |
| Age Rating | 4+ (no objectionable content) | Everyone / PEGI 3 | IARC questionnaire |
| Privacy Policy URL | **Required** (even with zero data collection) | **Required** | Host at stable URL |
| Support URL | **Required** | Recommended | GitHub issues page is acceptable |
| Screenshots | macOS: 1280x800+. iOS: 6.7" + 5.5". iPad: 12.9" | Phone (required), tablet | Up to 10 per size |
| Copyright | "© 2026 [Your Name]" | — | Required by Apple |
| What's New | 4000 chars (from CHANGELOG) | 500 chars | Required for updates |

### Apple Privacy Manifest (PrivacyInfo.xcprivacy)

**Required since May 2024** for all App Store submissions. File location:
- macOS: `platforms/macos/MyNotepadPP/Resources/PrivacyInfo.xcprivacy`
- iOS: `platforms/ios/MyNotepadPP/Resources/PrivacyInfo.xcprivacy`

| Setting | Value | Reason |
|---------|-------|--------|
| `NSPrivacyTracking` | `false` | No tracking |
| `NSPrivacyCollectedDataTypes` | Empty array | No data collection |
| Required Reason: File timestamps | `C617.1` | File watcher, auto-save conflict detection |
| Required Reason: Disk space | `E174.1` | Auto-save disk-full handling |
| Required Reason: Boot time | `35F9.1` | Performance timing (frame timing, debounce) |
| Required Reason: UserDefaults | `CA92.1` | User preferences storage |

### Privacy Policy

Required by Apple and Google even with zero data collection. Minimal policy hosted at stable URL stating:
- No data collection, no telemetry, no analytics, no accounts
- SFTP credentials stored in device keychain only, never transmitted
- All files remain on user's device (or user-specified remote server)

### F-Droid Requirements

- Must build entirely from source (including Rust core cross-compilation)
- No proprietary dependencies (no Google Play Services, no Firebase)
- Metadata in `metadata/com.mynotepadpp.android.yml` or `fastlane` structure
- License declared as SPDX: `GPL-3.0-only`
- `AutoUpdateMode: Version %v` + `UpdateCheckMode: Tags`

---

## GPL v3 + APP STORE COMPLIANCE

### The Legal Issue

GPL v3 Section 6 requires "Installation Information" — users must be able to install modified versions. Apple's DRM and code signing prevent this on iOS. The FSF's position is that GPL v3 is incompatible with App Store terms.

### Our Compliance Strategy (Two-Layer Protection)

**Layer 1 — GPL v3 Section 7 App Store Exception:**

Add to the LICENSE file:
```
Additional permission under GNU GPL version 3 section 7:

You have permission to convey this work through the Apple App Store,
Google Play Store, Microsoft Store, or any other digital distribution
platform, even though such distribution involves the addition of terms
that restrict users' ability to install modified versions of the work
on their devices.
```

This is explicitly allowed by GPL v3 Section 7 (additional permissions). It makes the license unambiguously App Store-compatible while preserving full copyleft for source code.

**Layer 2 — CLA (Contributor License Agreement):**

All contributors MUST sign a CLA that either:
- Assigns copyright to the project maintainer, OR
- Grants the maintainer unlimited relicensing rights

This prevents the VLC scenario (a single contributor forced App Store removal by objecting to DRM incompatibility). As sole effective copyright holder, you cannot violate your own license.

### Source Code Availability (GPL v3 Section 6 compliance)

- **In-app About screen**: display GPL v3 notice + link to source repository
- **In-app Help menu → "License"**: display full GPL v3 text
- **App Store description**: include "Open source (GPL v3): [repo URL]"
- **Git tag MUST match** the exact version shipped to each store — source at the tag corresponds to the shipped binary
- **License text bundled** in every app package:
  - macOS: `MyNotepadPP.app/Contents/Resources/LICENSE`
  - iOS: Settings.bundle + in-app About screen
  - Android: `assets/LICENSE` + About screen
  - Windows: `Assets/LICENSE` in MSIX + About dialog
  - Linux: `/usr/share/licenses/mynotepadpp/LICENSE` or Flatpak data dir

### Google Play — No Conflict

Google Play does not apply DRM that prevents modification. GPL v3 is fully compatible. Include source link in Play Store description and in-app About screen.

### F-Droid — Ideal

F-Droid is designed for GPL apps. They build from source. No issues.

---

## CONTRIBUTING

- All contributions must be GPL v3 compatible
- **CLA required**: contributors must sign a Contributor License Agreement granting copyright or relicensing rights (enables App Store distribution)
- DCO (Developer Certificate of Origin) sign-off on commits
- PR template: what, why, which platforms affected, how to test
- One feature = one PR. Don't bundle unrelated changes.
