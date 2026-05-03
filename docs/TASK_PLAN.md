# MYNOTEPAD++ — Implementation Task Plan

**Version:** 1.0
**Date:** 2026-05-03
**Covers:** v1.0 MVP (macOS only) — 65 features, all P0 + P1
**Source of truth:** `docs/FUNCTIONAL_SPECIFICATION.md` v1.2 + `CLAUDE.md`

---

## How to Read This Document

Each task has:
- **ID**: `P{phase}.T{task}` (e.g., P1.T3)
- **Purpose**: WHY this task exists — which user-facing feature or architectural requirement it satisfies
- **Scope**: EXACTLY what files/modules are created or modified
- **Acceptance criteria**: Measurable conditions that MUST be true before the task is done
- **Depends on**: Which prior tasks MUST be complete (by ID)
- **Integrates with**: Which features this task connects to (cross-references)
- **Test requirements**: What tests must pass

**Rule: Every task at the end of its phase produces a buildable, testable artifact. No phase leaves the app in a broken state.**

---

## Phase Overview

| Phase | Name | Goal | Deliverable |
|-------|------|------|-------------|
| **1** | Rust Core Foundation | Text buffer engine that can open, edit, save files | `core/` crate with passing `cargo test` |
| **2** | macOS App Shell | Native window that renders text from the core | Running `.app` that opens/saves files |
| **3** | Editor Essentials | Multi-tab, split view, multi-cursor, find/replace, auto-save | Usable editor for daily coding |
| **4** | Intelligence Layer | Syntax highlighting, code folding, autocomplete, tree-sitter | Professional editing experience |
| **5** | Power Features (P1) | Minimap, sidebar, snippets, git gutter, project support | Feature-complete v1.0 |
| **6** | Polish & Robustness | Performance tuning, accessibility, themes, zero-hang hardening | Ship-ready v1.0 |
| **7** | Distribution & Legal | App Store prep, signing, notarization, GPL compliance | Published v1.0 |

---

## PHASE 1 — Rust Core Foundation

**Goal:** Build the Rust core engine that handles ALL text manipulation, file I/O, encoding, undo/redo, and search. This is the engine that every platform consumes. Nothing renders to screen yet — all validation is via `cargo test` and benchmarks.

**Exit criteria:** `cargo test`, `cargo clippy -- -D warnings`, `cargo fmt --check` all pass. Benchmarks establish baselines for rope ops, search, file loading.

---

### P1.T1 — Project Scaffolding & Build System

**Purpose:** Create the Cargo workspace, crate structure, CI pipeline, and foundational files so every subsequent task has a place to land. Without this, nothing can be built.

**Scope:**
- Create `core/Cargo.toml` (workspace root) with edition 2024, `rust-version` MSRV
- Create `core/src/lib.rs` with `#![forbid(unsafe_code)]` at crate root
- Create `core/src/error.rs` — `EditorError` enum with `thiserror` (exactly as specified in CLAUDE.md ERROR HANDLING section: `Io`, `Encoding`, `Rope`, `Syntax`, `Diff`, `Macro`, `Plugin`, `InvalidArg`)
- Create `core/ffi/Cargo.toml` and `core/ffi/src/lib.rs` — FFI crate skeleton
- Create `core/ffi/src/error.rs` — `FfiResult` enum (`Ok=0`, `ErrIo=-1`, ... `ErrNullPointer=-100`)
- Create `core/ffi/cbindgen.toml` — cbindgen config for C header generation
- Set global allocator to `mimalloc` in `core/src/lib.rs`
- Add `Cargo.toml` dependencies: `thiserror`, `mimalloc`, `tracing`, `tracing-subscriber`
- Create `.github/workflows/ci.yml` — Rust CI job (fmt, clippy, test, test --release, audit)
- Create `LICENSE` file with GPL v3 text + Section 7 App Store exception
- Create `CHANGELOG.md` with Keep a Changelog format
- Create `CONTRIBUTING.md` with CLA requirement

**Acceptance criteria:**
- [ ] `cargo build` succeeds with zero warnings
- [ ] `cargo test` passes (even if only a trivial test exists)
- [ ] `cargo clippy -- -D warnings` passes
- [ ] `cargo fmt --check` passes
- [ ] `#![forbid(unsafe_code)]` is the first attribute in `core/src/lib.rs`
- [ ] `EditorError` has exactly 8 variants matching CLAUDE.md
- [ ] `FfiResult` has exactly 8 variants matching CLAUDE.md
- [ ] Global allocator is mimalloc
- [ ] CI runs on push and PR
- [ ] `cargo audit` runs in CI and denies `unmaintained` advisories
- [ ] `cargo deny` configured for license compatibility (GPL v3 / MIT / Apache 2.0 only)

**Depends on:** Nothing (first task)

**Integrates with:** Every subsequent task builds on this structure

**Test requirements:** `cargo test` — at minimum, `EditorError` round-trip test (create each variant, verify Display output)

---

### P1.T2 — Copy-on-Write Rope Buffer

**Purpose:** The rope is the foundational data structure for ALL text editing. Every character typed, every file loaded, every search performed operates on the rope. This must be correct, fast, and thread-safe via copy-on-write snapshots (NOT `RwLock` — per CLAUDE.md section 3).

**Scope:**
- Create `core/src/buffer/mod.rs` — public module
- Create `core/src/buffer/rope.rs` — `Document` struct wrapping a persistent rope
  - `Document` holds `Arc<Rope>` (current snapshot)
  - `Document::edit(range, text) -> Arc<Rope>` — returns new snapshot, old snapshot remains valid
  - `Document::snapshot() -> Arc<Rope>` — cheap clone of Arc for background threads
  - `Document::line(n) -> Option<RopeSlice>` — get line by index
  - `Document::line_count() -> usize`
  - `Document::byte_to_line(byte) -> usize`
  - `Document::line_to_byte(line) -> usize`
  - `Document::char_count() -> usize`
  - `Document::slice(range) -> RopeSlice`
- Use `ropey` crate with `simd` feature enabled for SIMD-accelerated operations
- Create `core/src/buffer/cursor.rs` — `Cursor { line: usize, col: usize, byte_offset: usize }` + movement methods (`move_right`, `move_left`, `move_up`, `move_down`, `move_to_line_start`, `move_to_line_end`, `move_word_left`, `move_word_right`)
- Create `core/src/buffer/selection.rs` — `Selection { anchor: Position, head: Position }` + multi-selection container `Selections(Vec<Selection>)` with merge/sort/dedup for overlapping selections
- Line-offset index: leverage ropey's built-in O(log n) line/byte conversion (augmented B+ tree)

**Acceptance criteria:**
- [ ] `Document::edit` produces a new `Arc<Rope>` — old snapshot is NOT modified (verified by test: snapshot before edit, edit, verify snapshot content unchanged)
- [ ] `Document::snapshot()` is O(1) — benchmark confirms < 100ns
- [ ] Multiple threads can hold snapshots simultaneously without contention (test: spawn 10 threads each holding a snapshot, main thread edits, no panic/deadlock)
- [ ] Line lookup is O(log n) — benchmark on 1M-line document
- [ ] Cursor movement is correct at document boundaries (line 0, last line, col 0, end of line)
- [ ] Multi-selection correctly merges overlapping selections
- [ ] Zero use of `RwLock`, `Mutex`, or any locking primitive in this module
- [ ] `#[cfg(test)] mod tests` in each file with comprehensive tests

**Depends on:** P1.T1

**Integrates with:** Every editing operation (P1.T4 undo, P1.T5 search, P2 rendering, P3 multi-cursor, P3 auto-save)

**Test requirements:**
- Unit tests: insert at start/middle/end, delete range, multi-line edit, empty document edge case
- Snapshot isolation test (concurrent reads + writes)
- Benchmark: `criterion` — insert, delete, line lookup on 100K-line document
- Fuzz test: random edit sequences on random documents

---

### P1.T3 — File I/O Pipeline (Load, Save, Encoding)

**Purpose:** Users open and save files. This task handles the entire file lifecycle: binary detection, encoding detection, chunked loading for large files, atomic save with three-tier fallback, and line ending detection. Per CLAUDE.md, file I/O runs on the I/O thread pool — never on the main thread.

**Scope:**
- Create `core/src/io/mod.rs` — public module
- Create `core/src/io/loader.rs` — file loading pipeline:
  1. Read first 8KB: detect binary (null bytes > 0.1% → `EditorError::InvalidArg("binary file")`)
  2. Detect BOM → XML/HTML charset → `encoding_rs` statistical detection → user locale fallback
  3. Detect line endings (LF/CRLF/CR) from first chunk
  4. For files < 1MB: `BufReader` in 64KB chunks → build rope
  5. For ALL files: `BufReader` in 64KB chunks → build rope (NO mmap for user files — SIGBUS risk on truncated files/network FS, per CLAUDE.md)
  6. Return `LoadResult { document: Document, encoding: Encoding, line_ending: LineEnding, was_binary: bool }`
- Create `core/src/io/saver.rs` — atomic save with three-tier fallback:
  - Tier 1: Write to `{dir}/{name}.mynotepadpp-{pid}-{timestamp}.tmp`, fsync, rename
  - Tier 2: Copy original to `.bak`, direct overwrite, fsync, delete backup
  - Tier 3: Write to recovery directory, return `SaveResult::RecoveryOnly(path)`
  - fsync timeout: 10 seconds (configurable). On timeout, proceed without sync.
  - Temp file naming includes PID + timestamp for orphan detection
- Create `core/src/io/encoding.rs` — `Encoding` enum, detection via `encoding_rs`, conversion
- Create `core/src/io/line_ending.rs` — `LineEnding { Lf, Crlf, Cr }`, detection, normalization
- Create `core/src/io/backup.rs` — continuous backup writer (500ms debounce, writes to `backups/{doc_id}/{timestamp}.backup`)
- Create `core/src/io/recovery.rs` — on-startup recovery scan: find backups newer than originals, validate encoding, return list of recoverable documents
- Use `bytecount` crate for SIMD-accelerated newline counting
- Use `simdutf8` crate for fast UTF-8 validation
- Add `encoding_rs` dependency

**Acceptance criteria:**
- [ ] Binary file detection: file with 2% null bytes → detected as binary
- [ ] Encoding detection: UTF-8, UTF-8 BOM, UTF-16 LE, UTF-16 BE, Shift-JIS → all detected correctly from test fixtures
- [ ] Line ending detection: pure LF file → `Lf`, pure CRLF → `Crlf`, mixed → `Crlf` (dominant wins)
- [ ] Atomic save Tier 1: write + fsync + rename succeeds; original file content matches
- [ ] Atomic save Tier 2: if rename returns EXDEV (simulated), falls back to direct overwrite
- [ ] Atomic save Tier 3: if write fails with EACCES (simulated), writes to recovery dir
- [ ] Large file (100MB test fixture): loads in < 2 seconds on M4
- [ ] Continuous backup: after edit, backup file appears within 600ms
- [ ] Recovery scan: detects backup newer than original, returns it in recovery list
- [ ] Orphaned temp file cleanup: temp files with dead PIDs detected and reported
- [ ] No `unwrap()` or `expect()` in any file (only `Result` propagation)
- [ ] No `println!` or `eprintln!` — only `tracing` macros

**Depends on:** P1.T1, P1.T2

**Integrates with:** P1.T5 (search uses loaded documents), P3.T2 (auto-save), P3.T3 (session restore), P6.T1 (performance hardening)

**Test requirements:**
- Unit tests: load UTF-8, UTF-16 LE, Shift-JIS, empty file, binary file, 0-byte file
- Integration tests in `core/tests/`: load real files of various encodings
- Benchmark: load 1MB, 10MB, 100MB files
- Fuzz test: random byte sequences → encoding detection must not panic

---

### P1.T4 — Undo/Redo Engine

**Purpose:** Users expect `Cmd+Z` to undo and `Cmd+Shift+Z` to redo. With the copy-on-write rope, undo is a stack of rope snapshots (each shares >99% of nodes via Arc). This must support operation grouping (macro playback = single undo group, replace-all = single undo group).

**Scope:**
- Create `core/src/undo/mod.rs`
- Create `core/src/undo/history.rs` — `UndoHistory` struct:
  - `undo_stack: Vec<UndoEntry>` — each entry holds `Arc<Rope>` snapshot + cursor state + group_id
  - `redo_stack: Vec<UndoEntry>` — cleared on new edit
  - `undo() -> Option<Arc<Rope>>` — pop undo stack, push to redo, return previous rope snapshot
  - `redo() -> Option<Arc<Rope>>` — pop redo stack, push to undo, return next rope snapshot
  - `push(snapshot, cursor, group_id)` — push new state. If group_id matches top of stack, merge (compaction).
  - `begin_group(name: &str) -> GroupId` — for macro/replace-all grouping
  - `end_group(id: GroupId)` — close the group
  - Memory budget: 100MB max per document (configurable). Oldest entries pruned when exceeded.
  - Undo entry memory = overhead of new nodes only (shared nodes don't count) — measure via `Arc` strong_count
- Operation compaction: consecutive single-char inserts at same position → merged into one entry after 500ms pause (compaction timer)

**Acceptance criteria:**
- [ ] Undo reverses the last edit exactly (insert → undo → original content)
- [ ] Redo re-applies after undo (undo → redo → edited content)
- [ ] New edit after undo clears redo stack
- [ ] Group: begin_group → 5 edits → end_group → single undo undoes all 5
- [ ] Compaction: type "hello" one char at a time → single undo undoes "hello"
- [ ] Memory budget: after exceeding 100MB, oldest entries are pruned
- [ ] Empty document: undo on empty history returns None (no panic)

**Depends on:** P1.T2

**Integrates with:** P3.T1 (multi-cursor — all cursors in one undo group), P3.T5 (find-replace-all as undo group), P4.T6 (macro playback as undo group)

**Test requirements:**
- Unit tests: single undo, single redo, undo after redo, group undo, compaction
- Memory test: create 10,000 undo entries, verify memory < budget
- Edge cases: undo on fresh document, redo without prior undo

---

### P1.T5 — Search Engine

**Purpose:** Find & Replace is a P0 feature. Search must be fast (< 2s for 10K files), support regex, support literal, and stream results. Per CLAUDE.md, search uses `memchr` for literals, `regex-automata` for regex with lazy DFA, and runs on the rayon thread pool.

**Scope:**
- Create `core/src/search/mod.rs`
- Create `core/src/search/in_buffer.rs` — search within a single document:
  - `search(rope: &Rope, query: &SearchQuery) -> Vec<Match>` — returns all matches
  - `search_incremental(rope: &Rope, query: &SearchQuery, prev_results: &[Match]) -> Vec<Match>` — refine previous results as user types
  - `SearchQuery { pattern: String, is_regex: bool, case_sensitive: bool, whole_word: bool }`
  - `Match { start: usize, end: usize, line: usize }`
- Create `core/src/search/in_files.rs` — multi-file search:
  - `search_files(paths: Vec<PathBuf>, query: &SearchQuery, cancel: CancelToken, progress: Sender<Progress>) -> Vec<FileMatch>`
  - Uses `rayon::par_iter` for parallel file scanning
  - Uses `ignore` crate for `.gitignore`-aware file walking
  - Respects `CancelToken` — checks between files
  - Streams results to UI via progress channel (batched per-file)
- Create `core/src/search/replace.rs` — replace logic:
  - `replace_all(rope: &Rope, query: &SearchQuery, replacement: &str) -> (Rope, usize)` — returns new rope + count
  - Regex backreferences: `$1`, `$2`, `$0` supported via `regex-automata` capture groups
- Create `core/src/cancel.rs` — `CancelToken(Arc<AtomicBool>)` with `cancel()` and `is_cancelled()`
- Create `core/src/progress.rs` — `Progress` enum (Indeterminate, Determinate, Complete, Error, Cancelled)
- Add dependencies: `memchr`, `regex-automata`, `ignore`, `rayon`
- Cache compiled regex patterns: LRU cache of 50 entries

**Acceptance criteria:**
- [ ] Literal search: finds all occurrences of "hello" in a 1MB document in < 10ms
- [ ] Regex search: `\b\w+_test\b` finds matches correctly
- [ ] Case-insensitive search: "Hello" matches "hello", "HELLO", "Hello"
- [ ] Whole-word search: "test" does NOT match "testing"
- [ ] Replace all: replaces all occurrences, returns correct count
- [ ] Regex backreferences: `(\w+)_(\w+)` replaced with `$2_$1` works
- [ ] Multi-file search: 10K files completes in < 2 seconds (benchmark)
- [ ] Cancellation: cancel mid-search, operation stops within 100ms
- [ ] `.gitignore` respected: files in `node_modules/` are not searched
- [ ] Incremental search: typing additional characters narrows previous results

**Depends on:** P1.T1, P1.T2

**Integrates with:** P3.T5 (Find & Replace UI), P3.T6 (Find in Files UI), P4.T8 (search in scope/selection)

**Test requirements:**
- Unit tests: literal, regex, case, whole-word, backreferences, empty pattern, empty document
- Integration tests: multi-file search on a real project directory
- Benchmark: literal and regex on 1MB, 10MB documents; multi-file on 10K files
- Fuzz test: random regex patterns must not panic (invalid regex → `EditorError`)

---

### P1.T6 — Cancellation Token & Progress Reporting

**Purpose:** Every long-running background operation (search, file loading, syntax parsing, diff) MUST be cancellable and report progress. Without this, the UI freezes on large operations. Per CLAUDE.md section 4 and 5.

**Scope:**
- Already created in P1.T5 (`cancel.rs`, `progress.rs`). This task adds comprehensive standalone tests, documentation, and the `TaskDispatcher` utility (per CLAUDE.md thread architecture). **This is a test + documentation + dispatcher task, not new file creation.**
- `CancelToken` — `cancel()`, `is_cancelled()`, `clone()`. Uses `Arc<AtomicBool>` with `Ordering::Release` / `Ordering::Acquire` (NOT `Relaxed` — per CLAUDE.md CancelToken spec).
- Create `core/src/dispatch.rs` — `TaskDispatcher` with `High`, `Normal`, `Low` priorities routing to the correct rayon pool instance (per CLAUDE.md thread architecture).
- `Progress` — `Indeterminate(String)`, `Determinate(f64, String)`, `Complete(String)`, `Error(String)`, `Cancelled`
- Ensure both are `Send + Sync` (required for cross-thread use)
- Document usage pattern in doc comments: "Check `is_cancelled()` between loop iterations"

**Acceptance criteria:**
- [ ] `CancelToken` is `Send + Sync`
- [ ] `Progress` is `Send + Sync`
- [ ] Cancel from one thread, `is_cancelled()` returns true in another thread within 1 microsecond
- [ ] Clone produces a shared token (cancel one, all clones see it)
- [ ] `TaskDispatcher` routes `High` priority to the HIGH rayon pool and `Normal`/`Low` to the LOW rayon pool
- [ ] Direct `rayon::spawn()` call (global pool) is NOT used anywhere — only `TaskDispatcher::dispatch()`
- [ ] Both rayon pool instances have correct QoS via `pthread_set_qos_class_self_np` in their `spawn_handler`

**Depends on:** P1.T1

**Integrates with:** P1.T5 (search), P1.T3 (file loading), P4.T1 (syntax parsing), P4.T5 (diff)

**Test requirements:** Unit tests for cross-thread cancellation, clone sharing. Integration test for dispatcher routing (submit to HIGH pool, verify QoS class of executing thread).

---

### P1.T7 — EditorConfig Parser

**Purpose:** `.editorconfig` support (Feature #33) is P0. The parser reads `.editorconfig` files and resolves properties for a given file path. This runs in the core so all platforms share the same logic.

**Scope:**
- Create `core/src/config/mod.rs`
- Create `core/src/config/editorconfig.rs`:
  - `parse_editorconfig(path: &Path) -> Result<EditorConfig, EditorError>` — parse a single `.editorconfig` file
  - `resolve_config(file_path: &Path) -> EditorConfigResolved` — walk upward from file directory, merge configs (closest wins), stop at `root = true`
  - `EditorConfigResolved` struct with: `indent_style: Option<IndentStyle>`, `indent_size: Option<u32>`, `tab_width: Option<u32>`, `end_of_line: Option<LineEnding>`, `charset: Option<Encoding>`, `trim_trailing_whitespace: Option<bool>`, `insert_final_newline: Option<bool>`, `max_line_length: Option<u32>`
  - Glob matching for section headers: `[*.rs]`, `[{*.js,*.ts}]`, `[Makefile]`

**Acceptance criteria:**
- [ ] Parses standard `.editorconfig` files per editorconfig.org spec
- [ ] Closest file wins per property (parent `.editorconfig` provides fallback)
- [ ] `root = true` stops upward search
- [ ] Glob patterns: `*.rs`, `*.{js,ts}`, `Makefile`, `lib/**.js` all match correctly
- [ ] Unknown properties are ignored (forward compatible)
- [ ] Missing `.editorconfig` → all `None` (no error)

**Depends on:** P1.T1

**Integrates with:** P1.T3 (encoding/line-ending from editorconfig), P3.T4 (indent behavior), P4.T3 (tab size detection)

**Test requirements:** Unit tests with fixture `.editorconfig` files covering all properties, glob patterns, multi-level inheritance

---

### P1.T8 — FFI Bridge (Core → Platform)

**Purpose:** The macOS app (Swift) needs to call Rust core functions. This task creates the C-compatible FFI layer with opaque pointers, `catch_unwind` on every export, and `*_free()` for every allocation. Per CLAUDE.md FFI Rules.

**Scope:**
- Create `core/ffi/src/document.rs`:
  - `document_open(path_ptr: *const u8, path_len: usize) -> *mut DocumentHandle` — opens file, returns opaque pointer. **Length-delimited strings only — no `*const c_char`** (per CLAUDE.md FFI Rules).
  - `document_create() -> *mut DocumentHandle` — creates empty document
  - `document_free(handle: *mut DocumentHandle)` — frees the document
  - `document_get_line(handle: *mut DocumentHandle, line: usize, out_ptr: *mut *const u8, out_len: *mut usize) -> FfiResult` — **Rust allocates output buffer**, returns pointer + length. Caller must call `string_free(ptr, len)` after use. No caller-allocated buffers (per CLAUDE.md FFI Rules — prevents buffer overflow on encoding expansion).
  - `string_free(ptr: *const u8, len: usize)` — frees a Rust-allocated string buffer
  - `document_line_count(handle: *mut DocumentHandle) -> usize`
  - `document_edit(handle: *mut DocumentHandle, start_byte: usize, end_byte: usize, text_ptr: *const u8, text_len: usize) -> FfiResult`
  - `document_save(handle: *mut DocumentHandle, path_ptr: *const u8, path_len: usize) -> FfiResult`
  - `document_undo(handle: *mut DocumentHandle) -> FfiResult`
  - `document_redo(handle: *mut DocumentHandle) -> FfiResult`
- Every `pub extern "C"` function wrapped in `std::panic::catch_unwind` returning `FfiResult::ErrPanic` on panic
- Every function validates null pointers → `FfiResult::ErrNullPointer`
- Generate C header via `cbindgen`: `core/ffi/include/mynotepadpp_core.h`
- Create `core/ffi/src/search.rs` — search FFI functions
- Create `core/ffi/src/version.rs` — `mynotepadpp_core_api_version() -> u32`

**Acceptance criteria:**
- [ ] Every `pub extern "C"` function has `catch_unwind`
- [ ] Every `pub extern "C"` function checks for null pointers
- [ ] Every `*_create` / `*_open` has a matching `*_free`
- [ ] `cbindgen` generates valid C header without errors
- [ ] FFI types are `#[repr(C)]` or opaque pointers
- [ ] No `unsafe` block without `// SAFETY:` comment
- [ ] `document_open` → `document_get_line` → verify content → `document_free` — no leak (test with ASAN)

**Depends on:** P1.T2, P1.T3, P1.T4, P1.T5

**Integrates with:** P2.T2 (Swift FFI bridge consumes these C functions)

**Test requirements:**
- Integration tests calling FFI functions from Rust (simulating what Swift would do)
- Null pointer tests — every function returns `ErrNullPointer` when given null
- Panic test — trigger a panic inside an FFI function, verify `ErrPanic` returned (not UB)
- Leak test — open/close 1000 documents, verify no memory growth

---

### P1.T9 — SQLite Session Database

**Purpose:** Session persistence (open tabs, cursor positions, scroll positions, window geometry) survives app restarts and crashes. Per CLAUDE.md SQLite section: WAL mode, single writer + multiple readers, 5-second batch flush, STRICT tables, zstd compression for blobs.

**Scope:**
- Add dependencies: `rusqlite` (with `bundled`, `modern_sqlite`, `backup`, `blob`, `trace` features), `zstd`
- Create `core/src/session/mod.rs`
- Create `core/src/session/db.rs` — `SessionDb` struct:
  - `open(path: &Path) -> Result<SessionDb>` — opens/creates DB, runs PRAGMA init, checks application_id, runs migrations
  - Full PRAGMA initialization (all 15 PRAGMAs from CLAUDE.md: 14 on every connection + 1 write-only `cache_spill`)
  - Schema: `windows`, `tabs`, `unsaved_content`, `undo_history`, `plugin_state`, `app_state`, `split_views` — all STRICT tables
  - `application_id = 0x4D4E5050` ('MNPP')
- Create `core/src/session/writer.rs` — `SessionWriter` (runs on dedicated SQLite writer thread):
  - In-memory dirty map: `HashMap<TabId, TabState>`
  - `mark_dirty(tab_id, state)` — updates in-memory map
  - `flush(conn)` — writes all dirty entries in single `BEGIN IMMEDIATE` transaction with `prepare_cached`
  - Flush interval: every 5 seconds
  - `hot_exit(conn, app_state)` — single bulk transaction: tab states + compressed unsaved content + compressed undo history + window geometry. Target: < 500ms for 50 tabs.
- Create `core/src/session/reader.rs` — `SessionReader` (read-only connections for UI thread):
  - `get_tabs_for_window(window_id) -> Vec<TabState>`
  - `get_window_state(window_id) -> WindowState`
- Create `core/src/session/recovery.rs` — crash detection and recovery:
  - Check `dirty_flag` in `app_state` table
  - If crashed: run `PRAGMA quick_check`, restore from backup if corrupt, nuke and recreate as last resort
  - Set `dirty_flag = 1` on open, `0` on clean exit
- Create `core/src/session/backup.rs` — periodic DB backup via `VACUUM INTO` every 10 minutes
- Clean exit: `PRAGMA optimize`, `PRAGMA incremental_vacuum`, `PRAGMA wal_checkpoint(TRUNCATE)`

**Acceptance criteria:**
- [ ] All 15 PRAGMAs from CLAUDE.md are set (14 on every connection + 1 write-only: `cache_spill`)
- [ ] `application_id` is checked on open — wrong ID → recreate DB
- [ ] `dirty_flag` set to 1 on open, 0 on clean exit
- [ ] Crash simulation: kill process mid-write, reopen → DB is consistent (WAL recovery)
- [ ] `quick_check` runs on crash-detected startup, not `integrity_check`
- [ ] Write batching: 50 tab state updates in single transaction < 5ms
- [ ] Hot exit: 50 tabs with compressed unsaved content < 500ms
- [ ] `prepare_cached` used for all repeated queries (no `prepare()`)
- [ ] All tables are STRICT
- [ ] Schema migration: increment `user_version`, migration code runs on version mismatch
- [ ] DB is never placed on network filesystem (path validation)
- [ ] Connection architecture: 1 writer (IMMEDIATE transactions) + read-only pool
- [ ] `zstd` compression of unsaved_content and undo_history blobs (3-5x reduction verified)

**Depends on:** P1.T1

**Integrates with:** P3.T3 (session restore), P3.T2 (auto-save metadata), P6.T2 (shutdown hardening)

**Test requirements:**
- Unit tests: open, write tab state, read back, verify
- Crash recovery test: set dirty_flag=1, reopen, verify recovery flow
- Performance benchmark: hot_exit with 50 synthetic tab states
- Concurrency test: writer + reader simultaneously (no blocking)
- Corruption test: corrupt DB file, verify graceful recreation

---

### PHASE 1 INTEGRATION CHECKPOINT

After all Phase 1 tasks are complete, verify:

1. **`cargo test` passes** — all unit tests across all modules
2. **`cargo test --release` passes** — no debug-only behavior
3. **`cargo clippy -- -D warnings` passes** — no warnings
4. **`cargo fmt --check` passes** — consistent formatting
5. **`cargo bench` runs** — baselines established for rope ops, search, file loading
6. **Fuzz targets created** — `cargo fuzz` for encoding detection and search input
7. **No `unwrap()`/`expect()` in library code** — grep verification
8. **No `println!`/`eprintln!` in library code** — grep verification
9. **All `pub` items have `///` doc comments** — grep verification
10. **FFI header generates cleanly** — `cbindgen` produces valid `mynotepadpp_core.h`

**The core is a standalone, testable library. No GUI exists yet. Phase 2 connects it to macOS.**

---

## PHASE 2 — macOS App Shell

**Goal:** A running macOS app that opens a window, loads a file via the Rust core FFI, renders text using CoreText, and saves changes. Single tab, single pane. The minimum viable rendering pipeline.

**Exit criteria:** App launches in < 500ms, opens a 1MB file in < 200ms, renders text at 60 FPS, saves files atomically. `xcodebuild test` passes.

---

### P2.T1 — Xcode Project Setup

**Purpose:** Create the Xcode project with correct build settings, entitlements, and Rust core integration. Per CLAUDE.md Build & CI section — static library linking, bridging header, build phases for Cargo.

**Scope:**
- Create `platforms/macos/MyNotepadPP.xcodeproj/` with the exact directory structure from CLAUDE.md
- Create `platforms/macos/MyNotepadPP/App/AppDelegate.swift` — `NSApplication` delegate
- Create `platforms/macos/MyNotepadPP/App/main.swift` — entry point
- Create `platforms/macos/MyNotepadPP/Core/BridgingHeader.h` — `#include "mynotepadpp_core.h"`
- Configure build settings per CLAUDE.md table: deployment target macOS 14.0, arm64, Hardened Runtime, App Sandbox, linker flags `-lmynotepadpp_core`, header/library search paths
- Create entitlements file with: App Sandbox, user-selected files R/W, app-scope bookmarks. **Do NOT include `network.client` in v1.0** (SFTP is v1.1). **Do NOT include `print` in v1.0** (Print is v1.1). Add these in v1.1 only.
- Create build phases: Run Script (Cargo build), Run Script (cbindgen), Compile Sources, Link Binary, Copy Resources
- Create `platforms/macos/MyNotepadPP/Resources/PrivacyInfo.xcprivacy` with Required Reason API declarations (file timestamps C617.1, disk space E174.1, boot time 35F9.1, UserDefaults CA92.1)
- Create `platforms/macos/MyNotepadPPTests/` and `platforms/macos/MyNotepadPPUITests/` directories

**Acceptance criteria:**
- [ ] `xcodebuild build` succeeds — Rust core is compiled and linked
- [ ] App launches and shows an empty window (no crash)
- [ ] Hardened Runtime is enabled
- [ ] App Sandbox is enabled
- [ ] PrivacyInfo.xcprivacy is included in bundle
- [ ] Bridging header imports `mynotepadpp_core.h`
- [ ] Build phases execute in correct order (Cargo before Compile)
- [ ] Deployment target is macOS 14.0 (Sonoma)
- [ ] Architecture is arm64 (dev) / arm64+x86_64 (release)

**Depends on:** P1.T8 (FFI bridge must exist)

**Integrates with:** Every subsequent macOS task

**Test requirements:** Build succeeds, app launches without crash

---

### P2.T2 — Swift FFI Bridge (EditorCore)

**Purpose:** Swift wrapper around the C FFI that provides safe, idiomatic Swift access to the Rust core. Per CLAUDE.md: `EditorCore` class manages opaque pointer lifecycle, `deinit` calls `*_free()`.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Core/EditorCore.swift`:
  - `class EditorCore` — wraps `OpaquePointer` to Rust `DocumentHandle`
  - `init(path: URL?)` — calls `document_open` or `document_create`
  - `deinit` — calls `document_free` (CRITICAL — prevents memory leak)
  - `func getLine(_ index: Int) -> String?` — calls FFI, converts C string to Swift String
  - `func lineCount() -> Int`
  - `func edit(range: NSRange, text: String)` — calls FFI edit
  - `func save(to url: URL) throws` — calls FFI save, maps FfiResult to Swift error
  - `func undo() -> Bool` / `func redo() -> Bool`
  - `func search(query: String, options: SearchOptions) -> [SearchMatch]`
  - All methods use `async/await` — FFI calls dispatched to background `TaskGroup`, results posted back to main actor
- Create `platforms/macos/MyNotepadPP/Core/EditorError.swift`:
  - `enum EditorCoreError: Error` mapping `FfiResult` codes to Swift errors
  - User-facing messages: actionable ("Could not open file: permission denied"), not technical

**Acceptance criteria:**
- [ ] `EditorCore` instance created and destroyed without leak (test with Instruments Allocations)
- [ ] `deinit` always calls `document_free` — verified via print in debug builds
- [ ] No force unwraps (`!`) — all optionals handled with `guard let` / `if let`
- [ ] `[weak self]` in all closures that outlive scope
- [ ] Errors mapped to user-friendly messages
- [ ] FFI calls never run on main thread — always dispatched via `async`
- [ ] Logging uses `OSLog`, not `print()` or `NSLog()`

**Depends on:** P1.T8, P2.T1

**Integrates with:** P2.T3 (EditorView uses EditorCore to get text), P3 (all editing operations)

**Test requirements:** XCTest: create EditorCore, open file, read line, verify content, close

---

### P2.T3 — Custom Text Rendering View (EditorView)

**Purpose:** Render text from the Rust core onto screen using CoreText and Metal. Per CLAUDE.md section 1: custom `NSView` (NOT `NSTextView`), glyph atlas, dirty rect rendering, viewport culling, overdraw buffer.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Views/EditorView.swift` — `// CANONICAL REFERENCE` tag at top:
  - Subclass `NSView`, override `draw(_ dirtyRect:)`
  - **NSTextInputClient (CRITICAL — Feature #59)**: MUST implement `NSTextInputClient` protocol for IME composition (CJK), emoji picker (`Cmd+Ctrl+Space`), dictation, system text replacement. Methods: `setMarkedText`, `markedRange`, `selectedRange`, `attributedSubstring`, `insertText`, `firstRect(forCharacterRange:)`, `characterIndex(for:)`. Call `inputContext?.invalidateCharacterCoordinates()` on scroll/layout change. Without this, CJK input, emoji, and dictation are completely broken.
  - **NSWritingToolsCoordinator (macOS 15+)**: implement delegate for Writing Tools integration (AI proofreading). Provide text context, handle replacements.
  - **Emoji glyph atlas**: separate RGBA texture atlas for Apple Color Emoji (`sbix` format bitmaps). Standard grayscale atlas does NOT work for emoji. Use `CTFontDrawGlyphs()`.
  - **Metal drawable lifecycle**: release `CAMetalDrawable` in `MTLCommandBuffer` completion handler. Never hold across frames. Handle `contentsScale` change on display switch.
  - **Explicit `close()` method**: called from tab-close logic to free Rust resources immediately. `deinit` is safety net only (prevents leaks from retain cycles).
  - **Text shaping**: CoreText (`CTLine`, `CTFrame`) for text shaping and glyph extraction. Handles ligatures, kerning, combining characters.
  - **Glyph atlas**: `MTLTexture`-backed atlas. Cache key: `(glyph_id, font_id, size, subpixel_offset)`. Color applied via Metal shader, not baked.
  - **Shaped text cache**: LRU cache of 10,000 entries per (content, style, font)
  - **Line layout cache**: per-line layout (character positions, advances). Invalidate only on that line's content change.
  - **Font metric cache**: ascent, descent, line height per (font, size) — cached at font load time
  - **Dirty rect rendering**: on edit, only re-render changed lines + cursor line. On scroll, only render newly visible lines.
  - **Overdraw buffer**: pre-render 2x viewport height (1 screen above + 1 below)
  - **Viewport culling**: only compute layout for visible lines + overdraw. Line-offset index (from rope) for O(log n) "which line is at pixel Y"
  - **Gutter**: line numbers column, 50px default width, right-aligned numbers, current line number bold
  - **Current line highlight**: subtle background color per theme
  - **Cursor**: blinking line cursor (1px width), drawn via Metal. Block cursor option.
  - **Scroll**: `NSScrollView` wrapping EditorView. Smooth scrolling with momentum. `scrollRectToVisible` for cursor following.
  - Request text from `EditorCore` by line range (visible lines + overdraw)
  - Metal fallback: if Metal unavailable, use CoreGraphics (no GPU acceleration)
- Font: SF Mono as default, configurable. Variable font support via CoreText axis API.
- **Performance target**: 60 FPS scrolling on 100MB file on M4

**Acceptance criteria:**
- [ ] Text renders correctly for ASCII, Unicode (emoji, CJK, Arabic)
- [ ] Ligatures render (Fira Code: `->`, `=>`, `!=`)
- [ ] Line numbers visible and correct
- [ ] Current line highlighted
- [ ] Cursor visible and blinking
- [ ] Scroll works with momentum (native macOS feel)
- [ ] 60 FPS when scrolling through a 100MB file (Instruments verification)
- [ ] Dirty rect: editing one line does NOT re-render entire viewport (Instruments verification)
- [ ] Glyph atlas: same glyph rendered twice uses cached texture (GPU trace verification)
- [ ] Long lines (10K+ chars): only visible columns are laid out, no hang
- [ ] Empty file renders correctly (blank content area with cursor at line 1)

**Depends on:** P2.T1, P2.T2

**Integrates with:** P3 (multi-cursor rendering, selection rendering), P4 (syntax highlighting colors), P5 (minimap)

**Test requirements:**
- XCTest: create EditorView, load document, verify line count rendered
- Performance: Instruments Time Profiler on 100MB file scroll
- Manual: VoiceOver announces cursor position when moving (accessibility foundation)

---

### P2.T4 — Window, Menu Bar & File Dialogs

**Purpose:** Complete macOS chrome: menu bar, file open/save dialogs, window management, Finder integration. Users need to open, save, and create files via standard macOS UI.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Resources/MainMenu.xib` — menu bar:
  - MYNOTEPAD++ (About, Preferences, Quit)
  - File (New, Open, Open Folder, Save, Save As, Save All, Revert, Close Tab, Close Window, Print)
  - Edit (Undo, Redo, Cut, Copy, Paste, Select All, Find...)
  - Selection, Find, View, Go, Preferences — menu items per FUNCTIONAL_SPECIFICATION.md section 4.4
  - All menu items wired to First Responder actions
  - Keyboard equivalents match specification exactly
- Create `platforms/macos/MyNotepadPP/Services/FileService.swift` — `// CANONICAL REFERENCE`:
  - `openFile()` — `NSOpenPanel` with multiple selection, any file type
  - `saveFile(document:)` — calls `EditorCore.save()`, handles errors via `NSAlert`
  - `saveFileAs(document:)` — `NSSavePanel`
  - `revertToSaved(document:)` — confirmation alert, then reload from disk
  - File type associations via `Info.plist` (UTI declarations for common text files)
  - Drag-and-drop: register window as drop target for file URLs
  - URL handler: `mynotepadpp://open?file=/path&line=42&col=15` — MUST show confirmation dialog ("Website wants to open [filename]. Allow?"). Canonicalize path, reject system dirs (`/etc/`, `/System/`), reject `..` traversal, rate-limit 1/second.
  - Recent files: `NSDocumentController.shared.noteNewRecentDocumentURL()`
- Window title: file name (with modified indicator ●), or "Untitled"
- Finder integration: Open With context menu, Spotlight metadata (deferred to P6)
- Status bar: bottom bar showing `Ln 1, Col 1 | Spaces: 4 | UTF-8 | LF | Plain Text`
- **Required NSApplicationDelegate methods:**
  - `applicationShouldHandleReopen(_:hasVisibleWindows:)` — when user clicks Dock icon with no windows open, restore session from SQLite (same as "On reopen" in CLAUDE.md section 7). Return `true` if windows were restored, `false` to let AppKit create a default window.
  - `application(_:openFiles:)` — Finder "Open With" for multiple files. Open each file in a new tab. Call `NSApp.reply(toOpenOrPrint: .success)` on completion.
  - `application(_:open:)` — handle `mynotepadpp://` URL scheme (already covered by URL handler above).
  - `applicationSupportsSecureRestorableState(_:) -> Bool` — return `true`. Required since macOS 12 to suppress console warnings and enable state restoration.

**Acceptance criteria:**
- [ ] Every menu item has correct keyboard shortcut per spec
- [ ] Open file dialog works — selected file opens in editor
- [ ] Save works — file written to disk, modified indicator clears
- [ ] Save As works — file written to new path
- [ ] Revert to Saved: shows confirmation, reloads from disk
- [ ] Drag file from Finder onto window → opens file
- [ ] Recent Files menu populated
- [ ] Window title shows filename
- [ ] Status bar shows cursor position, encoding, line ending
- [ ] `Cmd+N` creates new empty document
- [ ] `Cmd+W` closes (auto-save if enabled, no prompt)
- [ ] `Cmd+Q` quits (hot exit, no prompt)
- [ ] CLI: `./mynotepadpp file.rs:42` opens file at line 42
- [ ] Click Dock icon with no windows → session restored from SQLite
- [ ] Finder "Open With" on 5 files → all 5 open as tabs
- [ ] `applicationSupportsSecureRestorableState` returns `true` — no console warnings on macOS 12+

**Depends on:** P2.T2, P2.T3

**Integrates with:** P3.T2 (auto-save wires into FileService), P3.T3 (session restore uses FileService), P3.T5 (Find UI)

**Test requirements:**
- XCTest: programmatically trigger Open/Save, verify file operations
- XCUITest: click File > New, verify new tab appears
- Manual: drag file from Finder, verify it opens

---

### PHASE 2 INTEGRATION CHECKPOINT

After all Phase 2 tasks are complete, verify:

1. **App launches in < 500ms** — `time open MyNotepadPP.app`
2. **Opens 1MB file in < 200ms** — benchmark
3. **60 FPS scroll** on 100MB file — Instruments
4. **Memory < 50MB idle** with one file open — Activity Monitor
5. **File open/save/create works** — manual verification
6. **Cmd+Q quits immediately** — no hang, no prompt
7. **Menu bar complete** — all items present with correct shortcuts
8. **CLI invocation works** — `mynotepadpp file.rs:42`
9. **`xcodebuild test` passes** — all XCTests
10. **No force unwraps, no print(), no DispatchQueue** — grep verification

**At this point: a working single-tab text editor that opens, edits, and saves files.**

---

## PHASE 3 — Editor Essentials

**Goal:** Transform the single-file viewer into a real editor: multi-tab, split views, multi-cursor, find/replace, auto-save with hot exit, session restore. After this phase, the app is usable for daily coding.

**Exit criteria:** All editing features work correctly. Auto-save never hangs. Close/quit never prompts (with auto-save on). Session restores on relaunch. Find & Replace works with regex.

---

### P3.T1 — Multi-Cursor Editing

**Purpose:** Feature #8 (P0). Users add cursors at multiple positions and type at all simultaneously. Per spec: `Option+Click`, `Cmd+Option+Up/Down`, `Cmd+D` select next occurrence, `Cmd+Ctrl+G` select all occurrences, `Cmd+Shift+L` split selection into lines, `Option+Shift+Drag` column select.

**Scope:**
- Extend `EditorView` to render multiple cursors and selections
- Integrate with `core/src/buffer/selection.rs` — `Selections` container:
  - `add_cursor(position)` — add cursor, merge if overlapping
  - `select_next_occurrence(rope)` — find next match of current selection text, add cursor
  - `select_all_occurrences(rope)` — add cursor at every occurrence
  - `split_into_lines(rope)` — one cursor per line in current selection
  - `column_select(start, end)` — rectangular selection
- All edit operations (`insert`, `delete`, `indent`) apply to ALL cursors in a single undo group
- Cursor rendering: each cursor has its own blink state, drawn as 1px line
- Selection rendering: semi-transparent background highlight per selection. Batch into single GPU draw call for performance (no per-selection draw).
- `Esc` exits multi-cursor → single cursor at primary position

**Acceptance criteria:**
- [ ] `Option+Click` adds a cursor at click position
- [ ] `Cmd+Option+Up/Down` adds cursor above/below
- [ ] `Cmd+D` selects next occurrence, adds cursor
- [ ] `Cmd+K, Cmd+D` skips current, selects next
- [ ] `Cmd+Ctrl+G` adds cursors at ALL occurrences
- [ ] `Cmd+Shift+L` splits selection into per-line cursors
- [ ] `Option+Shift+Drag` creates column/rectangular selection
- [ ] Typing at all cursors simultaneously — text appears at each
- [ ] Backspace at all cursors — deletes at each
- [ ] `Cmd+Z` undoes the entire multi-cursor operation as one group
- [ ] `Esc` returns to single cursor
- [ ] 100 cursors: no visible lag when typing (< 16ms frame time)
- [ ] Overlapping selections auto-merged

**Depends on:** P2.T3 (EditorView rendering), P1.T2 (selections), P1.T4 (undo grouping)

**Integrates with:** P3.T5 (find adds cursors), P4.T7 (bracket selection), P4.T8 (expand/shrink selection)

**Test requirements:**
- XCTest: add 3 cursors, type "hello", verify 3 "hello" insertions
- XCTest: Cmd+D on word, verify next occurrence selected
- Performance: 100 cursors, type character, verify < 16ms frame time

---

### P3.T2 — Auto-Save & Continuous Backup

**Purpose:** Feature #6 (P0). NEVER lose data. Per CLAUDE.md section 6: debounce 1s after typing, throttle 30s, on focus lost. Three-tier atomic save. Continuous backup at 500ms. Per spec section 7.2: auto-save never hangs, close tab within 50ms, no dialog.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Services/AutoSaveService.swift`:
  - Debounce timer: 1s after last keystroke (resets on each key)
  - Throttle timer: 30s regardless of activity
  - Focus lost: immediate save on `NSApplication.didResignActiveNotification`
  - All three OR'd — whichever fires first wins, all reset after save
  - Save dispatched to background task (never blocks UI)
  - Calls `EditorCore.save()` which calls Rust core's three-tier save
- Create `platforms/macos/MyNotepadPP/Services/BackupService.swift`:
  - Continuous backup: 500ms debounce after any edit
  - Writes to `~/Library/Application Support/mynotepadpp/backups/{doc_id}/`
  - Independent of auto-save — runs even if auto-save is off
  - Cleanup: delete backup on successful save to original location
  - Startup: scan backups/, offer recovery for entries newer than original
- App Nap prevention: `ProcessInfo.processInfo.beginActivity(options: [.userInitiated])` while documents are dirty, release when clean
- File watcher suppression: after auto-save write, suppress FSEvents for that path for 500ms
- Modified indicator: tab shows ● when dirty, clears on save
- Disk full handling: if all tiers fail, set `DIRTY_CRITICAL` state, show persistent banner, poll every 10s

**Acceptance criteria:**
- [ ] Type in editor, wait 1.5s → file saved silently (modified indicator clears)
- [ ] Type continuously for 35s → file saved at 30s mark (throttle)
- [ ] Switch to another app → file saved immediately (focus lost)
- [ ] Close tab (`Cmd+W`) with auto-save on → closes within 50ms, no dialog
- [ ] Kill app (SIGKILL), relaunch → backup detected, recovery offered
- [ ] Max data loss after SIGKILL: 500ms of typing (continuous backup)
- [ ] File watcher does NOT trigger "external change" prompt after auto-save
- [ ] App Nap does NOT delay auto-save timers (test: background app, verify save timing)
- [ ] Untitled files: auto-saved to `recovery/` directory
- [ ] Disk full: banner shown, buffer preserved in memory, retry on space available

**Depends on:** P1.T3 (file save), P1.T9 (session DB), P2.T4 (file service)

**Integrates with:** P3.T3 (session restore reads what auto-save wrote), P3.T7 (tab modified indicator)

**Test requirements:**
- XCTest: modify document, wait 1.5s, verify file written
- XCTest: modify, then close tab → verify file saved + tab closed < 50ms
- XCTest: simulate disk full (mock), verify banner + memory preservation
- Manual: kill app from Activity Monitor, relaunch → verify recovery prompt

---

### P3.T3 — Session Restore & Hot Exit

**Purpose:** Feature #26 (P0). When the app quits (Cmd+Q, system shutdown, crash), ALL state is preserved. On next launch, restore exactly where the user left off. Per CLAUDE.md section 7 and FUNCTIONAL_SPECIFICATION section 4.47.

**Scope:**
- Integrate `core/src/session/` (P1.T9) into app lifecycle:
  - On `Cmd+Q` / `applicationShouldTerminate`:
    - Set `isShuttingDown = true` guard flag
    - Call `session_writer.hot_exit()` — single SQLite transaction: all tab states + compressed unsaved content + undo history
    - For named files with unsaved changes: delta-based — store edit operations (not full buffer)
    - Target: < 500ms for 50 tabs
    - No confirmation dialog. No waiting.
  - On `windowWillClose` (every window close, not just quit):
    - Save THIS window's session state to SQLite immediately
  - On `NSApplicationWillTerminate` (system shutdown / SIGTERM):
    - Same hot_exit sequence, 5-second budget
    - `ProcessInfo.disableSuddenTermination()` while documents dirty
  - On launch:
    - Check `dirty_flag` — if crashed, run recovery
    - Show recovery prompt: "Restore previous session?" [Restore] [Start Fresh]
    - If clean: restore session silently (no prompt)
    - Startup waterfall: show window < 50ms, active tab < 200ms, remaining tabs progressive
  - Safe mode: 3 crashes in 60 seconds → disable plugins, minimal theme
- State machine for each document: `Clean | Dirty | Saving { cancel } | HotExiting`
- Hot exit cancels any in-progress auto-save (CancelToken), takes over

**Acceptance criteria:**
- [ ] Cmd+Q: all state saved, app terminates within 3 seconds, no dialog
- [ ] Relaunch after Cmd+Q: previous session restored exactly (tabs, cursors, scroll positions)
- [ ] SIGTERM (system shutdown): all state saved within 5 seconds
- [ ] Crash recovery: dirty_flag detected, recovery prompt shown, user can restore or start fresh
- [ ] 50 tabs: hot exit < 500ms (benchmark)
- [ ] Startup waterfall: window visible < 50ms, active tab rendered < 200ms
- [ ] Shutdown guard: drag-and-drop during shutdown → file queued for next launch
- [ ] State machine: hot exit cancels in-progress auto-save without corruption
- [ ] Delta-based: named file with 1GB unsaved changes → hot exit stores edit ops (< 10KB), not full buffer

**Depends on:** P1.T9 (session DB), P3.T2 (auto-save), P3.T7 (tabs exist)

**EXECUTION ORDER NOTE:** P3.T3 depends on P3.T7 (tabs). Despite numbering, **implement P3.T7 BEFORE P3.T3**. Recommended Phase 3 execution order: P3.T1 → P3.T4 → P3.T5 → P3.T6 → P3.T7 → P3.T2 → P3.T3 → P3.T8 → P3.T9.

**Integrates with:** P6.T2 (shutdown hardening — iOS lifecycle, multi-window)

**Test requirements:**
- XCTest: open 5 files, move cursors, quit, relaunch → verify positions
- Performance benchmark: hot_exit with 50 synthetic tabs
- XCTest: simulate crash (set dirty_flag=1), verify recovery flow
- Manual: Cmd+Q, relaunch → all tabs restored

---

### P3.T4 — Keyboard Input & Editing Operations

**Purpose:** Handle all keyboard input: character insertion, deletion, line operations (move, duplicate, delete, join), indent/outdent, comment toggle, undo/redo shortcuts. Auto-closing brackets. Auto-indent. Per FUNCTIONAL_SPECIFICATION sections 4.4.2, 4.20, 4.21.

**Scope:**
- Implement `keyDown(_ event:)` in EditorView:
  - Character insertion at cursor(s)
  - Backspace, Delete (forward delete)
  - Enter/Return (with auto-indent: new line at correct indent level)
  - Tab/Shift+Tab (indent/outdent)
  - All shortcuts from section 4.4.2 (Cut, Copy, Paste, Select All, Select Line, Duplicate Line, Delete Line, Move Line Up/Down, Join Lines, Uppercase/Lowercase)
  - Chord shortcuts: `Cmd+K, Cmd+U` (uppercase), `Cmd+K, Cmd+L` (lowercase)
- Auto-closing brackets (section 4.20):
  - Type `(` → insert `)` with cursor between
  - Type `{` at end of line → insert `}` + newline + indent
  - Type `"` → insert `"` (not inside existing string — tree-sitter aware once P4 is done; heuristic for now)
  - Selection + `(` → wrap selection: `(selection)`
  - Backspace on empty pair `()` → delete both
  - Overtype: type `)` when already before `)` → skip over
- Auto-indent (section 4.21):
  - Enter after `{` → indent one level
  - Enter after `:` (Python) → indent one level
  - Type `}` → outdent to matching level
  - Paste code → adjust indent to match context
- Tab size auto-detection (section 4.22):
  - On file open: scan first 100 lines, detect tabs vs spaces and size
  - `.editorconfig` overrides detection (uses P1.T7)
- Comment toggle (`Cmd+/`): detect language comment style, toggle line comment
- Block comment toggle (`Cmd+Shift+/`): toggle block comment `/* ... */`
- Transpose: `Ctrl+T` (characters), `Ctrl+Option+T` (words)

**Acceptance criteria:**
- [ ] All character keys insert at cursor
- [ ] All editing shortcuts from section 4.4.2 work correctly
- [ ] Chord shortcut `Cmd+K, Cmd+U` uppercases selection
- [ ] Auto-close: type `(` → `()` with cursor between
- [ ] Auto-close: type `{` at end of line → `{` + newline + indent + `}`
- [ ] Auto-close: select "hello", type `(` → `(hello)`
- [ ] Overtype: cursor before `)`, type `)` → cursor moves past `)`, no double `)`
- [ ] Auto-indent: Enter after `{` → new line indented
- [ ] Tab detection: file with 2-space indent → status bar shows "Spaces: 2"
- [ ] `.editorconfig` overrides detection (test with fixture)
- [ ] Comment toggle: in `.rs` file, `Cmd+/` adds `//` prefix
- [ ] Transpose: `Ctrl+T` swaps characters correctly
- [ ] All operations work with multi-cursor (from P3.T1)
- [ ] Undo reverses each operation correctly

**Depends on:** P2.T3 (EditorView), P2.T2 (EditorCore), P1.T4 (undo), P1.T7 (editorconfig)

**Integrates with:** P4.T2 (tree-sitter aware auto-close), P4.T3 (tree-sitter aware indent)

**Test requirements:**
- XCTest: each editing shortcut verified programmatically
- XCTest: auto-close bracket pairs (all 6 pair types)
- XCTest: auto-indent after `{` in Rust/JS/Python
- XCTest: tab detection on fixture files

---

### P3.T5 — Find & Replace UI

**Purpose:** Feature #11 (P0). Find bar at top of editor pane. Find in selection (Feature #49). Replace. Regex support. Case, whole-word toggles. Per FUNCTIONAL_SPECIFICATION sections 4.10, 4.42.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Views/FindBarView.swift`:
  - Find field + Replace field + toggle buttons [Aa] [.*] [W] + navigation arrows [↑][↓] + close [×]
  - Find in Selection toggle button
  - Match count display: "14 matches" / "3 of 14 (in selection)"
  - `Cmd+F` opens find bar (focused)
  - `Cmd+Option+F` opens find + replace
  - `Cmd+G` / `Enter` → find next
  - `Cmd+Shift+G` / `Shift+Enter` → find previous
  - `Esc` closes find bar
  - `Cmd+E` → use selection for find (puts selected text in find field)
  - Toggle shortcuts: `Option+Cmd+R` (regex), `Option+Cmd+C` (case), `Option+Cmd+W` (whole word)
  - Replace: `Cmd+Shift+1` replace one, `Cmd+Shift+Enter` replace all
  - Replace All → single undo group
- Incremental search: as user types in find field, highlights update live
- Match highlighting in editor: orange background for matches, current match highlighted differently
- Search history: last 50 searches in dropdown
- Scroll annotations: orange ticks in scrollbar for match positions (integrates with P5.T6 scroll annotations)
- Calls `core/src/search/` via FFI

**Acceptance criteria:**
- [ ] `Cmd+F` opens find bar, cursor in find field
- [ ] Type text → matches highlight live in editor
- [ ] `Cmd+G` jumps to next match, `Cmd+Shift+G` to previous
- [ ] Regex mode: `\b\w+\b` highlights all words
- [ ] Case toggle: "Hello" matches/doesn't match "hello" based on toggle
- [ ] Whole word: "test" doesn't match "testing" when enabled
- [ ] Replace: single replace works, Replace All replaces all
- [ ] Replace All is single undo group (`Cmd+Z` undoes all replacements)
- [ ] Find in Selection: only matches within selected text
- [ ] Match count accurate
- [ ] `Esc` closes find bar, focus returns to editor
- [ ] `Cmd+E` fills find field with current selection
- [ ] Search history dropdown accessible via arrow key in find field

**Depends on:** P1.T5 (search engine), P2.T3 (EditorView for highlights), P3.T1 (multi-cursor)

**Integrates with:** P3.T6 (Find in Files), P4.T8 (highlight integration)

**Test requirements:**
- XCTest: open find bar, type query, verify match count
- XCTest: replace all, verify undo restores original
- XCUITest: full find/replace workflow via keyboard

---

### P3.T6 — Find in Files (Multi-File Search)

**Purpose:** Feature #19 (P0). Search across all files in a project/folder. Results shown in a panel. Per FUNCTIONAL_SPECIFICATION section 4.10 (Find in Files).

**Scope:**
- Create `platforms/macos/MyNotepadPP/Views/SearchResultsPanel.swift`:
  - Bottom panel showing results grouped by file
  - Each result: file name, line number, line preview with match highlighted
  - Click result → opens file at that line
  - Double-click → opens file and closes results panel
  - Tree view: file → matches under it
- Create `platforms/macos/MyNotepadPP/Services/SearchService.swift`:
  - Calls `core/src/search/in_files.rs` via FFI
  - Progress bar in status bar during search
  - `Esc` cancels search (sends CancelToken)
  - File filter: include/exclude by glob (`*.rs`, `!node_modules`)
  - Search scope: current file, open files, folder, project
  - Replace in Files: replace across multiple files with preview
- `Cmd+Shift+F` opens Find in Files panel
- Results stream in as they arrive (per-file batching)
- `.gitignore` respected (via `ignore` crate in core)

**Acceptance criteria:**
- [ ] `Cmd+Shift+F` opens Find in Files panel
- [ ] Search 10K files completes in < 2 seconds (benchmark)
- [ ] Results grouped by file with line previews
- [ ] Click result → file opens at correct line
- [ ] Cancel: `Esc` stops search within 100ms
- [ ] File filter: `*.rs` only searches Rust files
- [ ] `.gitignore` respected: `node_modules/` not searched
- [ ] Replace in Files: preview shown before applying
- [ ] Progress bar updates during search
- [ ] Empty search term → no results (no crash)

**Depends on:** P1.T5 (multi-file search), P3.T5 (find UI patterns)

**Integrates with:** P5.T2 (project/workspace — search scope defaults to project root)

**Test requirements:**
- XCTest: search in test fixture directory, verify results
- Performance: benchmark on 10K-file project
- XCUITest: open Find in Files, type query, click result

---

### P3.T7 — Multi-Tab Editing

**Purpose:** Feature #1 (P0). Tab bar at top, multiple documents open simultaneously. Per FUNCTIONAL_SPECIFICATION section 4.1: tab display, overflow, reorder, tear-off, context menu, close tab behavior, MRU cycling, pin tabs.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Views/TabBarView.swift`:
  - Horizontal tab strip above editor
  - Each tab: file name + close button (×) + modified indicator (●)
  - Tab overflow: scroll arrows + dropdown for all tabs
  - Drag to reorder tabs
  - Drag tab out of window → create new window (tab tear-off)
  - Right-click context menu: Close, Close Others, Close All, Close to the Right, Copy Path, Reveal in Finder
  - `Cmd+N` creates new empty tab
  - `Cmd+W` closes current tab (auto-save → close instantly)
  - `Ctrl+Tab` / `Ctrl+Shift+Tab` cycle tabs (MRU order)
  - `Cmd+1` through `Cmd+9` jump to tab by position
  - Pin tabs: right-click > Pin (smaller, stay left)
  - `Cmd+Shift+T` reopens last closed tab
- Tab state synced to SQLite session (P1.T9)
- Each tab holds its own `EditorCore` instance
- Switching tabs: restore scroll position and cursor from session state (< 30ms)

**Acceptance criteria:**
- [ ] Multiple files open in separate tabs
- [ ] Tab shows filename + modified indicator
- [ ] Click tab → switches to that document
- [ ] `Cmd+W` closes tab instantly (no prompt if auto-save on)
- [ ] `Cmd+N` creates new untitled tab
- [ ] `Ctrl+Tab` cycles through tabs in MRU order
- [ ] `Cmd+1` through `Cmd+9` jump to correct tab
- [ ] Tab reorder via drag
- [ ] Tab tear-off → new window
- [ ] Right-click context menu works
- [ ] Tab overflow → scroll arrows appear
- [ ] Pin tab → tab shrinks and stays left
- [ ] `Cmd+Shift+T` reopens closed tab
- [ ] Tab switch < 30ms (scroll + cursor restored from session)

**Depends on:** P2.T3 (EditorView), P2.T4 (window), P1.T9 (session state)

**Integrates with:** P3.T2 (auto-save per tab), P3.T3 (session restore restores all tabs)

**Test requirements:**
- XCTest: open 3 tabs, switch, verify content
- XCTest: close tab, verify auto-save called
- XCUITest: Cmd+N, Cmd+W, Ctrl+Tab
- Performance: tab switch < 30ms benchmark

---

### P3.T8 — Split View

**Purpose:** Features #3, #4 (P0). Vertical and horizontal split. Per FUNCTIONAL_SPECIFICATION section 4.2: `Cmd+\` vertical, `Cmd+Shift+\` horizontal, `Cmd+Option+\` 2x2 grid, resizable, independent scroll/cursor per pane, same file in multiple panes syncs edits.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Views/SplitContainerView.swift`:
  - `NSSplitView` wrapper managing multiple EditorView panes
  - Each pane has its own tab bar and EditorView
  - Vertical split: `Cmd+\` — adds pane to the right
  - Horizontal split: `Cmd+Shift+\` — adds pane below
  - 2x2 grid: `Cmd+Option+\`
  - 3-column: via Command Palette
  - Resize: drag divider, minimum pane width 200px
  - Move file between panes: drag tab from one pane to another
  - Same file in two panes: shared `Document` (same `Arc<Rope>`), each pane has own cursor + scroll
  - Close pane: `Cmd+K, Cmd+W` (chord)
  - Focus pane: `Cmd+Option+Arrow` to move focus between panes

**Acceptance criteria:**
- [ ] `Cmd+\` creates vertical split
- [ ] `Cmd+Shift+\` creates horizontal split
- [ ] `Cmd+Option+\` creates 2x2 grid
- [ ] Each pane scrolls independently
- [ ] Each pane has its own cursor
- [ ] Same file in two panes: edit in one → reflected in other immediately
- [ ] Drag tab between panes works
- [ ] `Cmd+K, Cmd+W` closes focused pane
- [ ] `Cmd+Option+Arrow` moves focus between panes
- [ ] Resize divider works, minimum 200px enforced
- [ ] Closing all panes returns to single pane view

**Depends on:** P3.T7 (tabs), P2.T3 (EditorView), P1.T2 (shared rope via Arc)

**Integrates with:** P3.T1 (multi-cursor per pane), P3.T5 (find per pane)

**Test requirements:**
- XCTest: create split, verify two panes exist
- XCTest: same file in two panes, edit in one, verify other updated
- XCUITest: split via keyboard, navigate between panes

---

### P3.T9 — Scroll Navigation & Keyboard Shortcuts

**Purpose:** Feature #7 (P0) + Feature #5 (P0). All scroll commands from section 4.3 + all remaining shortcuts from 4.4 that aren't covered by other tasks.

**Scope:**
- Implement all scroll commands from section 4.3:
  - `Cmd+Home/Up` → top, `Cmd+End/Down` → bottom
  - Scroll to middle via Command Palette
  - `Ctrl+L` center current line
  - `Ctrl+Up/Down` scroll without moving cursor
  - Page Up/Down
  - `Ctrl+G` → Goto Line dialog (input field for line number)
- Implement all navigation shortcuts from section 4.4.4:
  - `Cmd+P` → Goto Anything (file/symbol/line) — this is a complex feature, create skeleton here, full implementation in P4.T9
  - `Cmd+Shift+P` → Command Palette — create skeleton here, full in P5.T3
  - `Cmd+R` → Goto Symbol — skeleton here, full in P4.T9
  - `Ctrl+-` / `Ctrl+Shift+-` → Go back / Go forward (navigation stack, depth 100)
- Implement keybinding engine:
  - `platforms/macos/MyNotepadPP/Services/KeyBindingManager.swift`:
    - Load from `~/.mynotepadpp/keybindings.json`
    - Default keybindings (all from section 4.4)
    - Chord support (`Cmd+K, Cmd+U` = two-step)
    - Context-aware: `editorFocus`, `sidebarFocus`, `findBarFocus`
    - All shortcuts remappable
    - Import/export keybinding profiles
    - **Note:** Preset profiles (compatibility mappings for other editor shortcut layouts) are deferred to v1.1 per the release roadmap (section 12). The engine supports loading any JSON profile — presets are just bundled JSON files.

**Acceptance criteria:**
- [ ] All scroll shortcuts from section 4.3 work
- [ ] `Ctrl+G` opens Goto Line, typing line number jumps there
- [ ] Navigation stack: Go to Definition (P4) pushes, `Ctrl+-` pops back
- [ ] Keybinding engine resolves all default shortcuts correctly
- [ ] Chord shortcuts work (Cmd+K, Cmd+U)
- [ ] Custom keybindings from JSON override defaults
- [ ] Context-aware: `Cmd+L` = Select Line in editor, but not in find bar

**Depends on:** P2.T3 (EditorView scroll), P3.T7 (tabs for Goto Anything)

**Integrates with:** P4.T9 (Goto Anything, Command Palette), P5.T3 (full Command Palette)

**Test requirements:**
- XCTest: each scroll command verified
- XCTest: chord shortcut resolves correctly
- XCTest: custom keybinding overrides default

---

### PHASE 3 INTEGRATION CHECKPOINT

After all Phase 3 tasks are complete, verify:

1. **Multi-tab editing works** — open 10 files, switch between them
2. **Split view works** — vertical, horizontal, 2x2, same file in two panes
3. **Multi-cursor works** — add 5 cursors, type, undo, verify
4. **Find & Replace works** — literal, regex, case, whole word, replace all
5. **Find in Files works** — search project, click result, opens correct file
6. **Auto-save works** — edit file, wait, verify saved (no prompt)
7. **Hot exit works** — Cmd+Q, relaunch, all tabs restored
8. **Session restore works** — cursor positions, scroll positions, window geometry
9. **All keyboard shortcuts work** — no conflicts, no dead shortcuts
10. **Zero-hang verified** — close tab < 50ms, Cmd+Q < 3s, no dialog

**At this point: a usable multi-tab code editor with find/replace, split views, and rock-solid auto-save.**

---

## PHASE 4 — Intelligence Layer

**Goal:** Add syntax highlighting (50+ languages), code folding, autocomplete, bracket matching/colorization, tree-sitter powered expand/shrink selection, Go to Definition, sticky scroll. After this phase, the editor feels intelligent and professional.

**Exit criteria:** Syntax highlighting for 50+ languages. Code folding works. Autocomplete popup appears. Go to Definition works within same file. Expand/shrink selection follows syntax tree.

---

### P4.T1 — Tree-Sitter Integration & Syntax Highlighting

**Purpose:** Feature #2 (P0). Accurate, incremental syntax highlighting for 50+ languages. Per CLAUDE.md: tree-sitter for parsing, 30ms debounce after edit, revision tracking, viewport priority, embedded language support (HTML+JS+CSS).

**Scope:**
- Add `tree-sitter` and language grammar crates to `core/Cargo.toml` (precompiled native `.so`/`.dylib`, not WASM)
- Create `core/src/syntax/mod.rs`
- Create `core/src/syntax/parser.rs`:
  - `SyntaxParser` manages tree-sitter `Parser` instances per language
  - Lazy grammar loading: grammar loaded on first use, not at startup
  - `parse(rope: &Rope, old_tree: Option<&Tree>) -> Tree` — incremental parse via `tree_sitter::Parser::parse_with`
  - Revision tracking: each parse result carries rope generation number. If rope has advanced, discard stale result.
  - 30ms debounce after edit before re-parse
  - Cancel-and-restart: if new edit arrives during parse, cancel via CancelToken
  - Parse on rayon pool (HIGH priority lane)
- Create `core/src/syntax/highlighter.rs`:
  - `highlight(tree: &Tree, rope: &Rope, theme: &Theme, visible_range: Range<usize>) -> Vec<HighlightSpan>`
  - `HighlightSpan { start: usize, end: usize, style: StyleId }` — maps to theme colors
  - Priority: visible viewport → ±1 screen overdraw → rest of file
  - Long line limit: highlight only visible column range + 500 chars
- Create `core/src/syntax/languages.rs`:
  - Language registry mapping file extensions to grammars
  - 50+ languages per FUNCTIONAL_SPECIFICATION section 4.11
  - File type detection: extension → shebang → modeline → content heuristic (P1.T3 loader + section 4.36)
- Embedded language parsing:
  - HTML: detect `<script>` → JS grammar, `<style>` → CSS grammar
  - Markdown: detect fenced code blocks → appropriate grammar
  - Use tree-sitter language injection
- Theme integration:
  - Create `core/src/syntax/theme.rs`:
    - `Theme` struct loaded from JSON theme files
    - Maps tree-sitter highlight names (`keyword`, `string`, `comment`, etc.) to colors
    - 10 bundled themes (5 dark + 5 light) per section 11

**Acceptance criteria:**
- [ ] All 50+ languages from section 4.11 have working syntax highlighting
- [ ] Incremental: edit one line, only that region re-parsed (verify via parse tree comparison)
- [ ] Visible viewport highlighted first (verify: open 100MB file, visible lines colored before scrolling down)
- [ ] 30ms debounce: rapid typing doesn't cause parse storm
- [ ] Stale parse discarded: verify old parse result not applied to newer rope version
- [ ] Embedded: HTML with `<script>` → JS highlighted correctly inside script tags
- [ ] Embedded: Markdown with ```rust``` → Rust highlighted inside code block
- [ ] Long lines (100K chars): only visible portion highlighted, no hang
- [ ] Theme applies: keywords one color, strings another, comments another
- [ ] Syntax highlight after edit < 50ms for visible viewport (benchmark)
- [ ] Grammar loading lazy: opening first `.rs` file loads Rust grammar, not all 50+

**Depends on:** P1.T2 (rope), P2.T3 (EditorView to render colors)

**Integrates with:** P4.T2 (code folding uses syntax tree), P4.T3 (indent uses tree), P4.T5 (Go to Definition uses tree), P4.T7 (bracket matching uses tree), P4.T8 (expand/shrink uses tree)

**Test requirements:**
- Unit tests: parse Rust/Python/JS/HTML, verify token types
- Integration test: incremental parse after edit, verify tree updated
- Benchmark: parse 10K-line file, highlight visible viewport
- Manual: open mixed HTML file, verify JS/CSS highlighted in embedded regions

---

### P4.T2 — Code Folding

**Purpose:** Feature #14/#29 (P0). Collapse code regions. Per FUNCTIONAL_SPECIFICATION section 4.19: tree-sitter scopes, fold gutter, fold all, fold at level N, region markers, find auto-unfolds.

**Scope:**
- Create `core/src/syntax/folding.rs`:
  - `compute_fold_ranges(tree: &Tree, rope: &Rope) -> Vec<FoldRange>` — extract foldable regions from syntax tree
  - `FoldRange { start_line: usize, end_line: usize, kind: FoldKind }` — function, class, block, import, comment, region marker
  - Region markers: `// #region Name` / `// #endregion`
- Extend `EditorView`:
  - Fold gutter: triangles ▶ (folded) / ▼ (expanded), click to toggle
  - Folded region: show first line + `... (N lines)` indicator
  - Folded lines hidden from viewport (not rendered, not in scroll calculation)
  - Line numbers skip folded range
  - Keyboard: `Cmd+Option+[` fold, `Cmd+Option+]` unfold at cursor
  - `Cmd+K, Cmd+0` fold all, `Cmd+K, Cmd+J` unfold all
  - `Cmd+K, Cmd+1-5` fold at level N
  - Find interaction: if match is in folded region, auto-unfold

**Acceptance criteria:**
- [ ] Fold gutter shows triangles for foldable regions
- [ ] Click triangle → region folds/unfolds
- [ ] Folded region shows `... (N lines)` indicator
- [ ] Line numbers skip folded lines
- [ ] `Cmd+K, Cmd+0` folds everything
- [ ] `Cmd+K, Cmd+J` unfolds everything
- [ ] `Cmd+K, Cmd+2` folds at indent level 2
- [ ] Region markers: `// #region` / `// #endregion` creates foldable region
- [ ] Find match in folded region → region auto-unfolds
- [ ] Scroll position stable after fold/unfold

**Depends on:** P4.T1 (syntax tree), P2.T3 (EditorView gutter)

**Integrates with:** P3.T5 (find auto-unfolds), P4.T8 (expand selection respects folds)

**Test requirements:**
- XCTest: fold function in Rust file, verify lines hidden
- XCTest: find text in folded region, verify unfold
- XCTest: fold all, unfold all

---

### P4.T3 — Smart Indent & Tab Size Detection

**Purpose:** Feature #24/#31/#32 (P0). Tree-sitter aware indentation. Tab size auto-detection. Per sections 4.21, 4.22.

**Scope:**
- Extend auto-indent from P3.T4 with tree-sitter awareness:
  - Use syntax tree to determine indent level for new lines
  - Type `}` → outdent to matching `{` level (tree-sitter scope)
  - Paste code → adjust indent based on tree context
- Tab size detection enhanced:
  - On file open: scan first 100 lines, detect tabs/spaces and size
  - `.editorconfig` overrides (from P1.T7)
  - Status bar shows detected style (click to change manually)

**Acceptance criteria:**
- [ ] Enter after `fn main() {` → new line indented (Rust)
- [ ] Enter after `def foo():` → new line indented (Python)
- [ ] Type `}` → auto-outdents to correct level
- [ ] Paste indented code → adjusted to match context
- [ ] Tab detection: 2-space file shows "Spaces: 2", 4-space shows "Spaces: 4", tab file shows "Tabs: 4"

**Depends on:** P4.T1 (syntax tree), P3.T4 (basic indent)

**Integrates with:** P1.T7 (editorconfig), P5.T2 (project settings override)

**Test requirements:** XCTest: indent behavior in Rust, Python, JS with tree-sitter

---

### P4.T4 — Bracket Matching & Colorization

**Purpose:** Feature #20/#36 (P0). Highlight matching brackets. Color nested brackets. Per sections 4.20 (matching), 4.26 (colorization).

**Scope:**
- Create `core/src/syntax/brackets.rs`:
  - `find_matching_bracket(tree: &Tree, rope: &Rope, position: usize) -> Option<usize>` — find matching bracket using tree-sitter
  - Bracket pair colorization: assign colors by nesting depth (3-color cycle per theme)
  - Applies to `()`, `[]`, `{}`, `<>` (in languages where `<>` is a bracket)
- Extend EditorView:
  - Highlight matching bracket pair when cursor is adjacent
  - `Ctrl+M` → jump to matching bracket (section 4.39)
  - `Ctrl+Shift+M` → select content between brackets (section 4.39)
  - Bracket pair colors rendered inline (not just gutter)

**Acceptance criteria:**
- [ ] Cursor next to `(` → matching `)` highlighted
- [ ] Nested brackets: 3 distinct colors cycling
- [ ] `Ctrl+M` jumps to matching bracket
- [ ] `Ctrl+Shift+M` selects inner content, press again → includes brackets
- [ ] Unmatched bracket → no highlight (no crash)
- [ ] Configurable: on/off toggle, custom colors in theme

**Depends on:** P4.T1 (tree-sitter for accurate bracket matching)

**Integrates with:** P3.T1 (multi-cursor bracket operations), P4.T8 (expand selection)

**Test requirements:** XCTest: bracket matching in nested code, colorization verification

---

### P4.T5 — Go to Definition & Navigation Stack

**Purpose:** Feature #34 (P0). Navigate to symbol definition. Per section 4.24: tree-sitter scope analysis for same-file, project-wide heuristic for cross-file.

**Scope:**
- Create `core/src/navigation/mod.rs`
- Create `core/src/navigation/definition.rs`:
  - `find_definition(tree: &Tree, rope: &Rope, position: usize) -> Option<Position>` — same-file via tree-sitter
  - `find_definition_project(symbol: &str, project_root: &Path) -> Vec<FileMatch>` — cross-file via grep for `fn/def/class/function SYMBOL` patterns
  - Navigation stack: push current position, depth 100
- Extend EditorView:
  - `Cmd+Click` or `F12` → go to definition
  - `Ctrl+-` → go back (pop stack)
  - `Ctrl+Shift+-` → go forward

**Acceptance criteria:**
- [ ] Cursor on function call → `F12` → jumps to function definition (same file)
- [ ] Cross-file: cursor on imported symbol → jumps to definition in other file
- [ ] `Ctrl+-` returns to previous position
- [ ] `Ctrl+Shift+-` goes forward
- [ ] Navigation stack: 100 entries deep, oldest pruned
- [ ] No definition found → status bar shows "No definition found"

**Depends on:** P4.T1 (tree-sitter), P1.T5 (search for cross-file)

**Integrates with:** P4.T9 (Goto Symbol), P3.T9 (navigation stack)

**Test requirements:** XCTest: go to definition within Rust file, verify position. Back/forward.

---

### P4.T6 — Word-Based Autocomplete

**Purpose:** Feature #39 (P0). Suggest completions as user types. Per section 4.32: buffer words + language keywords, debounced 50ms, popup below cursor.

**Scope:**
- Create `core/src/completion/mod.rs`:
  - `complete(rope: &Rope, cursor: Position, tree: Option<&Tree>) -> Vec<CompletionItem>`
  - Word completion: scan buffer for words matching prefix
  - Keyword completion: language keywords from tree-sitter grammar
  - Path completion: detect string literal context, complete file paths
  - Ranking: exact prefix > fuzzy > distance from cursor
- Create `platforms/macos/MyNotepadPP/Views/CompletionPopupView.swift`:
  - Popup below cursor, max 10 visible items, scrollable
  - Arrow keys to navigate, Tab/Enter to accept, Esc to dismiss
  - Show on 3+ chars typed or `Ctrl+Space` manual trigger
  - Debounced 50ms — typing rapidly doesn't spam completions
  - Computed on rayon pool, never blocks typing

**Acceptance criteria:**
- [ ] Type "pri" in Rust file → shows "println", "print", "private" etc.
- [ ] Language keywords: type "fn" → shows "fn" keyword
- [ ] Path completion: inside string `"./src/` → shows files in src/
- [ ] Tab/Enter accepts completion
- [ ] Esc dismisses popup
- [ ] `Ctrl+Space` triggers manually
- [ ] Popup appears < 50ms after debounce
- [ ] 10,000 unique words in buffer → no lag (computed on background thread)

**Depends on:** P4.T1 (tree-sitter for keywords), P2.T3 (popup rendering)

**Integrates with:** P5.T2 (project-wide word index in v1.1)

**Test requirements:** XCTest: type prefix, verify completion items, accept, verify inserted

---

### P4.T7 — Indent Guides & Current Line Highlight

**Purpose:** Feature #20 (P0, indent guides) + Feature #40 (P0, current line highlight). Visual aids for code structure.

**Scope:**
- Extend EditorView rendering:
  - Indent guides: thin vertical lines at each indent level, subtle color per theme
  - Current line highlight: background color change on the line with the primary cursor
  - Multiple cursors: each cursor's line highlighted
  - Reduced motion: respect OS setting (`isReduceMotionEnabled`) — disable cursor blink animation

**Acceptance criteria:**
- [ ] Indent guides visible at each indent level
- [ ] Current line has subtle background highlight
- [ ] Multi-cursor: each line highlighted
- [ ] Guides and highlight theme-aware (change with dark/light mode)

**Depends on:** P2.T3 (EditorView rendering)

**Integrates with:** P4.T1 (indent level from syntax tree for accurate guides)

**Test requirements:** Manual visual verification. XCTest: verify highlight position changes with cursor.

---

### P4.T8 — Expand/Shrink Selection & Select to Brackets

**Purpose:** Feature #45 (P0) + Feature #46 (P0). Tree-sitter aware selection expansion. Per sections 4.38, 4.39.

**Scope:**
- Create `core/src/navigation/selection.rs`:
  - `expand_selection(tree: &Tree, rope: &Rope, selection: Selection) -> Selection` — expand to parent syntax node
  - `shrink_selection(tree: &Tree, rope: &Rope, selection: Selection, history: &[Selection]) -> Selection` — shrink back
  - Selection history stack per expand sequence
- `Ctrl+Shift+Space` → expand selection: word → token → expression → statement → block → function → class → file
- `Ctrl+Shift+Backspace` → shrink selection (reverse)
- `Ctrl+Shift+M` → select to brackets (reuses P4.T4 bracket matching)

**Acceptance criteria:**
- [ ] Cursor on variable name → expand → selects variable → expand → selects expression → expand → selects statement → etc.
- [ ] Shrink reverses each expansion step
- [ ] Works across all 50+ languages (tree-sitter powered)
- [ ] Select to brackets: selects inner content, again → includes brackets

**Depends on:** P4.T1 (tree-sitter), P4.T4 (bracket matching)

**Integrates with:** P3.T1 (multi-cursor expand/shrink)

**Test requirements:** XCTest: expand sequence in Rust/Python, verify each step

---

### P4.T9 — Goto Anything & Goto Symbol

**Purpose:** Feature #10 (P0). Unified fuzzy navigator: `Cmd+P` for files, `:` for line, `@` for symbol, `#` for search. Per section 4.7.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Views/GotoAnythingView.swift`:
  - Overlay popup (similar to Command Palette)
  - `Cmd+P` opens with file search mode
  - Type `:42` → go to line 42
  - Type `@handleClick` → go to symbol in current file
  - Type `#TODO` → search in current file
  - Chained: `main.rs:42`, `main.rs@main`
  - Fuzzy matching: "mnrs" matches "main.rs"
  - Preview: as user navigates list, show file preview without opening
  - Recently opened files appear first
  - File icons per language
- `Cmd+R` → Goto Symbol in current file (shortcut to `@` prefix)
- `Cmd+T` → Goto Symbol in project (uses `core/src/search/` for cross-file symbol search via tree-sitter `symbols` query)
- Create `core/src/syntax/symbols.rs`:
  - `extract_symbols(tree: &Tree, rope: &Rope) -> Vec<Symbol>` — functions, classes, structs, enums, methods
  - Symbol index built lazily per-file on first Goto Symbol request
  - Incremental update on file edit

**Acceptance criteria:**
- [ ] `Cmd+P` opens Goto Anything
- [ ] Type file name → fuzzy matches shown
- [ ] Type `:42` → jumps to line 42
- [ ] Type `@main` → jumps to `main` function
- [ ] `Cmd+R` opens with `@` prefix
- [ ] `Cmd+T` searches symbols across project
- [ ] Preview: navigating list shows file preview
- [ ] Esc cancels, Enter opens
- [ ] Recently opened files ranked first

**Depends on:** P4.T1 (tree-sitter for symbols), P3.T7 (tabs for file opening), P3.T9 (keybinding)

**Integrates with:** P5.T3 (Command Palette uses similar UI)

**Test requirements:** XCTest: open Goto Anything, type filename, verify match. Symbol search.

---

### PHASE 4 INTEGRATION CHECKPOINT

1. **50+ languages highlighted** — open sample files for each language
2. **Code folding works** — fold/unfold functions, classes, regions
3. **Autocomplete works** — type prefix, see suggestions, accept
4. **Bracket matching and colorization** — matching pairs highlighted, nested colors
5. **Go to Definition** — same-file and cross-file
6. **Expand/shrink selection** — follows syntax tree correctly
7. **Goto Anything** — files, lines, symbols all work
8. **Indent guides visible** — correct at each level
9. **Syntax highlight < 50ms** for visible viewport after edit
10. **No performance regression** — still 60 FPS scroll on 100MB file

---

## PHASE 5 — Power Features (P1)

**Goal:** Add P1 features that complete the v1.0 experience: minimap, file tree sidebar, project/workspace, snippets, distraction-free mode, git gutter, sticky scroll, column editing, line operations, Command Palette, scroll annotations, encoding/line-ending UI, preferences, themes.

**Exit criteria:** All 65 features from the v1.0 feature matrix work. Feature-complete v1.0.

---

### P5.T1 — File Tree Sidebar

**Purpose:** Feature #28 (P1). Explorer panel showing project files. Per section 4.12.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Views/SidebarView.swift`:
  - `NSOutlineView`-based file tree
  - Toggle: `Cmd+B`
  - File operations: new file, new folder, rename, delete, duplicate
  - Context menu: Open, Open to Side, Reveal in Finder, Copy Path, Copy Relative Path
  - Search filter: type to filter files
  - File icons per language (from icon set in section 10.3)
  - Git status colors: green (new), yellow (modified), red (deleted) — executes `git status` via platform layer (`Process` in Swift), NOT via Rust core (Rule #6 forbids `std::process::Command` in core). Git calls are platform-side only.
  - Drag and drop: reorder and move files/folders
  - Multi-select: `Cmd+Click`
- File system watcher: FSEvents for project root (debounced 200ms)
- Excluded patterns from preferences: `node_modules`, `.git`, `.DS_Store`

**Acceptance criteria:**
- [ ] `Cmd+B` toggles sidebar
- [ ] File tree shows project structure
- [ ] Click file → opens in editor
- [ ] Right-click context menu works
- [ ] File icons per language type
- [ ] Git status colors shown
- [ ] Type to filter works
- [ ] New file/folder/rename/delete work
- [ ] FSEvents: external file change updates tree

**Depends on:** P2.T4 (window layout), P3.T7 (tabs)

**Integrates with:** P5.T2 (project workspace), P3.T6 (search scope)

---

### P5.T2 — Project/Workspace Support

**Purpose:** Feature #27 (P1). Per-project settings, multi-root workspaces. Per section 4.30.

**Scope:**
- Open folder → creates `.mynotepadpp/project.json` in project root
- Per-project settings: indent style, tab size, theme, excluded patterns
- Multi-root: workspace can include multiple folders
- Search scope defaults to project root
- Recent projects: File → Open Recent → Projects section

**Acceptance criteria:**
- [ ] Open folder → sidebar shows project tree
- [ ] `.mynotepadpp/project.json` created
- [ ] Per-project settings override global
- [ ] `.editorconfig` overrides project settings for matching files
- [ ] Recent projects accessible from menu

**Depends on:** P5.T1 (sidebar), P1.T7 (editorconfig)

---

### P5.T3 — Command Palette

**Purpose:** Feature #9 (P0). Universal command executor. Per section 4.6: `Cmd+Shift+P`, fuzzy matching, shortcut display, categories.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Views/CommandPaletteView.swift`:
  - Overlay popup with text input
  - Lists all registered commands with fuzzy match
  - Shows keyboard shortcut next to each command
  - Recently used commands first
  - Categories: File, Edit, View, Selection, Find, Go, Preferences
  - All menu items also available as commands
  - Extensible: plugins can register commands (v1.1)

**Acceptance criteria:**
- [ ] `Cmd+Shift+P` opens Command Palette
- [ ] Type "tww" → matches "Toggle Word Wrap"
- [ ] Shows shortcut next to command
- [ ] Enter executes command
- [ ] Esc dismisses
- [ ] Recently used commands appear first

**Depends on:** P3.T9 (keybinding engine for shortcut display)

---

### P5.T4 — Minimap

**Purpose:** Feature #12 (P1). Zoomed-out file preview. Per section 4.9.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Views/MinimapView.swift`:
  - Right side panel, 80px width (configurable 60-120)
  - Rendered as scaled-down bitmap on background thread (debounced 200ms)
  - GPU-composited onto main view
  - NOT per-character rendering
  - Shows syntax colors, search highlights, modified lines
  - Click to scroll, drag viewport highlight to scroll
  - Toggle: `Cmd+Shift+M`

**Acceptance criteria:**
- [ ] `Cmd+Shift+M` toggles minimap
- [ ] Viewport highlight shows current position
- [ ] Click jumps to that position
- [ ] Drag scrolls
- [ ] Syntax colors visible in minimap
- [ ] Search matches shown as colored markers

**Depends on:** P4.T1 (syntax colors), P2.T3 (scroll integration)

---

### P5.T5 — Distraction-Free Mode

**Purpose:** Feature #16 (P1). Per section 4.13: hide all chrome, center editor, `Cmd+Ctrl+F`.

**Scope:**
- Hide: tab bar, sidebar, minimap, status bar
- Menu bar auto-hides (appears on hover)
- Editor centered with configurable max width (80/100/120/full)
- Background fades to solid color
- `Esc` or `Cmd+Ctrl+F` exits

**Acceptance criteria:**
- [ ] `Cmd+Ctrl+F` enters distraction-free mode
- [ ] All chrome hidden
- [ ] Editor centered
- [ ] `Esc` exits
- [ ] Menu bar appears on hover at top

**Depends on:** P3.T7 (tabs), P5.T1 (sidebar), P5.T4 (minimap)

---

### P5.T6 — Remaining v1.0 Features

**Purpose:** Bundle all remaining features that don't warrant their own large task. Each is small but must be implemented.

**Scope:** (each item has specific acceptance criteria inline)

1. **Snippet system** (Feature #17, section 4.29): Tab triggers, tabstops, placeholders, variables. Storage in `~/.mynotepadpp/snippets/`. Built-in snippets per language.
2. **Column/block editing** (Feature #18, section 4.31): `Option+Shift+Drag`, rectangular selection, paste N lines → N cursors.
3. **Line operations** (Feature #23): Sort ascending/descending, deduplicate, join. Via Command Palette.
4. **Git gutter** (Feature #37, section 4.27): Green/blue/red markers in gutter for added/modified/deleted lines vs git HEAD. Hover shows original. Revert hunk. Debounced 500ms.
5. **Sticky scroll** (Feature #38, section 4.28): Scope headers pinned at top when scrolling through long functions/classes. Max 5 sticky lines. Click → scroll to scope start.
6. **Whitespace visualization** (Feature #41, section 4.34): Render spaces as `·`, tabs as `→`, newlines as `¶`. Options: none/selection/trailing/boundary/all.
7. **Wrap guides/rulers** (Feature #42, section 4.35): Vertical lines at column 80, 120 (configurable array). Wrap modes: off/viewport/column.
8. **URL detection** (Feature #48, section 4.41): Detect URLs, underline on hover, `Cmd+Click` opens browser.
9. **Read-only mode** (Feature #50, section 4.43): Toggle via Command Palette. Lock icon on tab. All edits rejected.
10. **Scroll annotations** (Feature #53, section 4.46): Ticks on scrollbar for search matches, modified lines, cursor position.
11. **Encoding/line ending UI**: Status bar segments clickable to change encoding/line ending.
12. **Zoom**: `Cmd+=` / `Cmd+-` / `Cmd+Shift+0` for font size.
13. **Word wrap toggle**: `Option+Z`.
14. **Drag-and-drop file opening** (Feature #25): Register as drop target.
15. **Open file at line from terminal** (Feature #35, section 4.25): CLI `mynotepadpp file:42:15`, URL scheme handler.
16. **Binary file detection** (Feature #51, section 4.44): Scan first 8KB, warn user.
17. **Long line handling** (Feature #52, section 4.45): Column-level viewport virtualization for lines > 10K chars.
18. **Transpose** (Feature #47, section 4.40): `Ctrl+T` characters, `Ctrl+Option+T` words.
19. **Revert to saved** (Feature #44, section 4.37): File > Revert File, confirmation dialog, undo-able.
20. **File type auto-detection** (Feature #43, section 4.36): Shebang, modelines, content heuristics.
21. **Smart highlighting** (Feature #54, section 4.50): Auto-highlight all occurrences of selected word. Debounced 100ms, viewport only. Distinct from Find highlights. Toggle in Settings.
22. **Case conversion commands** (Feature #55, section 4.51): 8 commands (UPPER, lower, Title, camelCase, snake_case, PascalCase, kebab-case, CONSTANT_CASE) via Command Palette + chord shortcuts. Multi-cursor aware. Single undo group.
23. **Document statistics** (Feature #56, section 4.52): Word/char/line count in status bar. Selection stats on selection. Full stats dialog via Command Palette.
24. **Convert indentation** (Feature #58, section 4.53): Tabs-to-spaces, spaces-to-tabs, detect indentation. Via Command Palette.
25. **Trim whitespace** (Feature #64): Trim trailing, trim leading, trim both. Via Command Palette and Edit > Blank Operations menu.
26. **Insert date/time** (Feature #65): Insert short/long date-time at cursor. Via Edit > Insert menu.
27. **File handling edge cases** (section 4.49): All 12 edge cases — duplicate open (switch to existing tab), same-name disambiguation (`filename — parent_dir/`), file deleted on disk, file becomes read-only, symlinks (follow target), permissions/ownership/xattrs preservation on atomic save, max file size warning > 10GB, large paste (>10MB) on background thread, auto-reload preference (`ask`/`always`/`never`), .git directory edit warning (auto-save disabled for .git files).
28. **Save behavior details** (section 4.48): NSSavePanel default extension per syntax (30+ mappings), extension enforcement, Save All behavior (skip read-only, untitled to recovery), Save Copy As (no path change).
29. **Editing edge cases** (section 4.55): Wrapped line cursor (visual movement with arrows, logical with Cmd), selection across folds (includes hidden content), CJK word boundaries (ICU segmentation for Option+Left/Right).

**Acceptance criteria per item:** Each must match its FUNCTIONAL_SPECIFICATION section exactly. All shortcuts must work. All preferences must be persisted. Key measurable criteria for complex sub-items:
- [ ] **Snippets (P5.T6.1):** Tab trigger expands snippet, tabstops navigable, placeholders editable, 5+ built-in snippets per major language
- [ ] **Column editing (P5.T6.2):** `Option+Shift+Drag` creates rectangular selection, paste N lines → N cursors, entire operation = single undo group
- [ ] **Git gutter (P5.T6.4):** Green/blue/red markers in gutter, hover shows original, revert hunk works, debounced 500ms
- [ ] **Sticky scroll (P5.T6.5):** Scope headers pinned at top (max 5), click → scroll to scope start, toggle via View menu
- [ ] **Smart highlighting (P5.T6.21):** Select word → all visible occurrences highlighted within 100ms, distinct from Find highlights
- [ ] **File handling edge cases (P5.T6.27):** All 12 edge cases from section 4.49 verified (including .git directory warning)
- [ ] **Save behavior (P5.T6.28):** NSSavePanel default extension per syntax for all 30+ mappings, Save Copy As does not change active tab path

**Depends on:** Phases 1-4

---

### P5.T7 — Preferences System

**Purpose:** Section 6 (Preferences & Settings). Dual-mode: GUI panel + JSON editor. All settings from `preferences.json` must be live.

**Scope:**
- Create `platforms/macos/MyNotepadPP/Views/PreferencesView.swift`:
  - NSWindow with categories: Editor, Auto-Save, Theme, Files, Search, Window
  - Each setting maps to `preferences.json` key
  - Changes saved immediately to JSON
  - JSON editor: button to "Edit JSON" opens preferences.json in the editor itself
- File watcher on preferences.json: live reload
- Default preferences: exactly as specified in section 6.1

**Acceptance criteria:**
- [ ] Preferences window opens from menu
- [ ] All settings from section 6.1 present
- [ ] Changing font size → editor updates immediately
- [ ] Changing theme → editor updates immediately
- [ ] "Edit JSON" opens file in editor
- [ ] External edit of JSON → settings reload live

**Depends on:** P2.T4 (window management), P5.T6 (feature integration), P5.T8 (theme system — must be implemented BEFORE preferences so theme switching works).

**EXECUTION ORDER NOTE:** Implement P5.T8 (themes) BEFORE P5.T7 (preferences) despite numbering. Recommended Phase 5 execution order: P5.T1 → P5.T2 → P5.T3 → P5.T4 → P5.T5 → P5.T6 → P5.T8 → P5.T7.

---

### P5.T8 — Dark/Light Mode & Theme System

**Purpose:** Feature #13 (P0). System appearance respect + manual override. 10 bundled themes. Per section 11, CLAUDE.md DARK MODE section.

**Scope:**
- Theme loading: read JSON theme files from `themes/` directory
- 5 dark + 5 light themes per section 11 (MYNOTEPAD++ Dark, Midnight Ocean, Carbon, Forest Night, Slate, MYNOTEPAD++ Light, Paper, Arctic, Meadow, Sand)
- System appearance: respect `NSAppearance` — auto-switch between light/dark variants
- Manual override via Preferences
- Hot-reload: watch `themes/` directory, reload without restart
- All themes are original (not ported from any editor)

**Acceptance criteria:**
- [ ] All 10 themes load and apply correctly
- [ ] System dark mode → editor switches to dark variant
- [ ] Manual override works
- [ ] Hot-reload: modify theme JSON → editor updates live
- [ ] WCAG AAA high-contrast theme included
- [ ] Color-blind friendly theme included

**Depends on:** P4.T1 (syntax highlighting uses theme colors)

---

### PHASE 5 INTEGRATION CHECKPOINT

1. **All 65 features from v1.0 matrix work** — manual verification of each
2. **File tree sidebar** — shows project, opens files
3. **Command Palette** — fuzzy search, execute commands
4. **Minimap** — visible, clickable, shows syntax colors
5. **Distraction-free mode** — all chrome hidden, exit works
6. **Git gutter** — shows changes vs HEAD
7. **Snippets** — tab triggers work
8. **Preferences** — all settings saved and applied
9. **10 themes** — all load, dark/light auto-switch
10. **All P0 features verified** against section 3.1 feature matrix

---

## PHASE 6 — Polish & Robustness

**Goal:** Performance tuning, accessibility (VoiceOver), zero-hang hardening, edge case handling. Make the app ship-ready.

**Exit criteria:** 60 FPS on M4, < 500ms startup, VoiceOver complete, zero-hang scenarios verified, all tests passing.

---

### P6.T1 — Performance Tuning

**Purpose:** Meet all performance targets from section 7.1. Profile and optimize hot paths.

**Scope:**
- Profile with Instruments (Time Profiler, Allocations, Core Animation, Energy)
- Ensure: cold startup < 500ms, 1MB open < 200ms, 100MB open < 2s, 60 FPS scroll, keystroke < 16ms
- Memory: < 50MB idle, < 300MB with 50 tabs
- Optimize glyph atlas hit rate, layout cache effectiveness
- Verify SIMD paths active (simdutf8, bytecount)
- Verify mimalloc vs system allocator (benchmark, keep whichever is faster on M4)
- Verify thermal throttling behavior (ProcessInfo.thermalState)

**Acceptance criteria — ALL 17 metrics from section 7.1 must pass:**
- [ ] Cold startup < 500ms
- [ ] Warm startup: active tab < 200ms
- [ ] Open 1MB file < 200ms
- [ ] Open 100MB file < 2s (first screen < 200ms)
- [ ] Open 1GB file < 10s (first screen < 200ms)
- [ ] Open 1MB single line (minified JS) < 3s, no hang
- [ ] Keystroke latency < 16ms
- [ ] Scroll FPS >= 60 on M4
- [ ] Search 10K files (literal) < 2s
- [ ] Search 10K files (regex) < 5s
- [ ] Autocomplete popup < 50ms after debounce
- [ ] Memory idle < 50MB
- [ ] Memory 50 tabs < 300MB
- [ ] Auto-save < 50ms (< 100KB) / < 100ms (> 1MB)
- [ ] Hot exit 50 tabs < 500ms
- [ ] Tab switch < 30ms
- [ ] Syntax highlight after edit < 50ms for viewport
- [ ] File watcher response < 500ms
- [ ] Benchmark results for ALL metrics documented in a results file

**Depends on:** All previous phases

---

### P6.T2 — Shutdown Hardening, Watchdog & QoS

**Purpose:** Verify zero-hang guarantees from section 7.2. Handle every edge case from CLAUDE.md section 7. Implement the I/O watchdog and QoS thread assignments mandated by CLAUDE.md thread architecture.

**Scope:**
- Verify: Cmd+Q < 3s, Cmd+W < 50ms, SIGTERM < 5s
- NFS/SMB stall: I/O timeout at 30s, UI never freezes
- Disk full: banner shown, buffer preserved, retry works
- iCloud/Dropbox: cloud-aware save behavior
- Close last window: session saved (not just on quit)
- APFS clonefile() for fast backups (detect APFS at runtime)
- Multi-window: shared rope via Arc, session per window
- **I/O watchdog thread**: lightweight check every 5 seconds for stuck I/O operations. If an operation exceeds its timeout, post warning to UI: "Saving to [path] is slow — network may be unreachable."
- **QoS class assignment** (per CLAUDE.md thread architecture):
  - Main thread: `.userInteractive` (implicit via AppKit)
  - Rayon pool HIGH priority lane: `.userInitiated` (visible viewport syntax highlighting)
  - Rayon pool NORMAL lane: `.utility` (search, diff)
  - Rayon pool LOW lane: `.background` (file indexing)
  - I/O thread pool: `.utility`
  - SQLite writer thread: `.utility`
  - Plugin threads: `.background`
  - Verify QoS assignments in Instruments Energy Log

**Acceptance criteria:**
- [ ] All scenarios from section 7.2 verified
- [ ] I/O watchdog: simulate 35s stall → warning posted to UI after 30s
- [ ] QoS: Instruments Energy Log shows correct QoS per thread type
- [ ] Edge cases tested: disk full, NFS stall, iCloud conflict, APFS clonefile

**Depends on:** P3.T2, P3.T3

---

### P6.T3 — VoiceOver Accessibility

**Purpose:** Feature #58 (P0 — launch requirement). Per section 8: all UI elements labeled, editor navigable by line/word/character, announcements for status changes.

**Scope:**
- All views: accessibility labels and roles
- EditorView: line/word/character navigation via VoiceOver
- Cursor position announced on move
- Selection range announced
- Status bar changes: live regions for auto-announce
- Keyboard-only operation: every feature reachable without mouse
- Focus management: logical tab order, no focus traps
- Reduced motion: respect OS setting

**Acceptance criteria:**
- [ ] VoiceOver: navigate editor by line → reads line content
- [ ] VoiceOver: navigate by word/character
- [ ] Cursor move → position announced
- [ ] File saved → "File saved" announced
- [ ] Encoding changed → announced
- [ ] Tab switching → new tab name announced
- [ ] All buttons/controls labeled
- [ ] Full keyboard navigation without mouse

**Depends on:** All UI tasks

---

### P6.T4 — Testing Suite

**Purpose:** Comprehensive test coverage: XCTest, XCUITest, benchmark suite. Per CLAUDE.md TESTING section.

**Scope:**
- Layer 1: `cargo test` — all Rust modules (already from Phases 1-4)
- Layer 2: XCTest — EditorCore bridge, ThemeManager, FileService, KeyBindingManager
- Layer 3: XCUITest — Tab operations, Find & Replace, Command Palette, multi-cursor, dark mode toggle, split view
- Layer 4: Manual test checklist: VoiceOver, performance (60 FPS, < 500ms startup), large files, crash recovery
- CI: GitHub Actions `macos-14` runner — `cargo test` + `xcodebuild test`

**Acceptance criteria:**
- [ ] All cargo tests pass
- [ ] All XCTests pass
- [ ] All XCUITests pass
- [ ] CI pipeline green on push and PR

**Depends on:** All previous phases

---

### PHASE 6 INTEGRATION CHECKPOINT

1. All performance targets met (section 7.1)
2. All zero-hang guarantees verified (section 7.2)
3. VoiceOver tested manually — all criteria from section 8
4. All tests passing (cargo + XCTest + XCUITest)
5. CI pipeline green

---

## PHASE 7 — Distribution & Legal

**Goal:** Prepare for App Store submission: signing, notarization, store metadata, privacy manifest, GPL compliance. Ship v1.0.

---

### P7.T1 — Code Signing, Notarization & Packaging

**Purpose:** Section 13.3 App Store compliance. Package the app for distribution.

**Scope:**
- Hardened Runtime: verify enabled
- Code signing: development cert (local), distribution cert (release)
- Notarization: submit to Apple for notarization
- Package: `.dmg` for direct distribution, `.app` for Mac App Store
- `Info.plist`: correct `CFBundleShortVersionString`, `CFBundleVersion`, UTI declarations
- PrivacyInfo.xcprivacy: verify all Required Reason APIs declared
- GPL v3 license: bundled in app bundle + exposed via Help menu
- About screen: GPL notice + source code link
- Privacy policy: hosted at stable URL, linked in app

**Acceptance criteria:**
- [ ] App notarized by Apple
- [ ] `.dmg` installer works on clean macOS 14.0 Sonoma
- [ ] About screen shows GPL notice + source link
- [ ] Help → License shows full GPL v3 text
- [ ] PrivacyInfo.xcprivacy validates in Xcode

---

### P7.T2 — Store Metadata & Submission

**Purpose:** Section 13.4. Prepare all assets and metadata for App Store submission.

**Scope:**
- App icon: 1024x1024 `.icns`
- Screenshots: at least 3 at 1280x800 or 1440x900
- Description: 4000 chars, highlighting key features
- Keywords: 100 chars
- Category: Developer Tools
- Age rating: 4+
- Privacy policy URL
- Support URL (GitHub issues)
- What's New text (from CHANGELOG.md)
- Submit to App Store Connect
- Homebrew Cask formula for direct distribution

**Acceptance criteria:**
- [ ] All store metadata prepared per section 13.4
- [ ] Submitted to App Store Connect
- [ ] Homebrew Cask formula published

---

### P7.T3 — CI/CD Pipeline for Releases

**Purpose:** Section 13.5. Automated build + sign + notarize + upload on git tag.

**Scope:**
- GitHub Actions workflow: on tag push `macos-v*`
  - Build Rust core (release, aarch64-apple-darwin + x86_64-apple-darwin)
  - Build universal binary
  - `xcodebuild archive`
  - Notarize
  - Create `.dmg`
  - Upload to GitHub Releases
  - Upload to App Store Connect via `altool` / `notarytool`
- Version: `CFBundleShortVersionString` from git tag, `CFBundleVersion` from `github.run_number`
- CHANGELOG entry required (CI validates)

**Acceptance criteria:**
- [ ] Push `macos-v1.0.0` tag → CI produces notarized `.dmg` + uploads
- [ ] Version numbers match tag
- [ ] CHANGELOG entry exists for this version

---

### PHASE 7 INTEGRATION CHECKPOINT — v1.0 SHIP

1. App notarized and signed
2. Available on Mac App Store (or submitted for review)
3. Available via Homebrew Cask + GitHub Releases
4. GPL v3 license visible in app and store listing
5. Source code tagged and matches shipped binary
6. CHANGELOG.md updated
7. All 65 features working
8. All tests passing
9. All performance targets met
10. VoiceOver accessible

**v1.0 is shipped. v1.1 development begins (Phases 8+).**

---

## TASK CROSS-REFERENCE MATRIX

Every feature from the v1.0 matrix (65 features) maps to at least one task:

| Feature # | Feature | Task(s) |
|-----------|---------|---------|
| 1 | Multi-tab editing | P3.T7 |
| 2 | Syntax highlighting | P4.T1 |
| 3 | Vertical split | P3.T8 |
| 4 | Horizontal split | P3.T8 |
| 5 | Keyboard shortcuts | P3.T4, P3.T9 |
| 6 | Auto-save | P3.T2 |
| 7 | Scroll navigation | P3.T9 |
| 8 | Multi-cursor | P3.T1 |
| 9 | Command Palette | P5.T3 |
| 10 | Goto Anything | P4.T9 |
| 11 | Find & Replace | P3.T5 |
| 12 | Minimap | P5.T4 |
| 13 | Dark/Light mode | P5.T8 |
| 14 | Line numbers + folding | P2.T3 (numbers), P4.T2 (folding) |
| 15 | Encoding support | P1.T3 |
| 16 | Distraction-free mode | P5.T5 |
| 17 | Snippets | P5.T6.1 |
| 18 | Column editing | P5.T6.2 |
| 19 | Find in files | P3.T6 |
| 20 | Indent guides + brackets | P4.T7 (guides), P4.T4 (brackets) |
| 21 | Word wrap | P5.T6.13 |
| 22 | Zoom | P5.T6.12 |
| 23 | Line operations | P5.T6.3 |
| 24 | Auto-indent | P3.T4, P4.T3 |
| 25 | Drag-and-drop | P2.T4 (window drop target) — P5.T6.14 removed (already implemented in P2.T4) |
| 26 | Session restore | P3.T3 |
| 27 | Project support | P5.T2 |
| 28 | File tree sidebar | P5.T1 |
| 29 | Code folding | P4.T2 |
| 30 | Auto-close brackets | P3.T4 |
| 31 | Smart indent | P4.T3 |
| 32 | Tab size detection | P4.T3 |
| 33 | .editorconfig | P1.T7 |
| 34 | Go to Definition | P4.T5 |
| 35 | CLI open at line | P5.T6.15 |
| 36 | Bracket colorization | P4.T4 |
| 37 | Git gutter | P5.T6.4 |
| 38 | Sticky scroll | P5.T6.5 |
| 39 | Autocomplete | P4.T6 |
| 40 | Current line highlight | P4.T7 |
| 41 | Whitespace visualization | P5.T6.6 |
| 42 | Wrap guides | P5.T6.7 |
| 43 | File type detection | P5.T6.20 |
| 44 | Revert to saved | P5.T6.19 |
| 45 | Expand/shrink selection | P4.T8 |
| 46 | Select to brackets | P4.T8 |
| 47 | Transpose | P5.T6.18 |
| 48 | URL detection | P5.T6.8 |
| 49 | Find in selection | P3.T5 |
| 50 | Read-only mode | P5.T6.9 |
| 51 | Binary detection | P5.T6.16 |
| 52 | Long line handling | P5.T6.17 |
| 53 | Scroll annotations | P5.T6.10 |
| 54 | Smart highlighting (auto-highlight selected word) | P5.T6.21 |
| 55 | Case conversion commands | P5.T6.22 |
| 56 | Document statistics | P5.T6.23 |
| 57 | Save Copy As | P5.T6.28 (Save behavior details — implements saveCopyAs in FileService) |
| 58 | Convert indentation | P5.T6.24 |
| 59 | NSTextInputClient (IME, emoji, dictation) | P2.T3 (EditorView — CRITICAL) |
| 60 | Paste and Indent | P3.T4 (keyboard input) |
| 61 | Open Recent submenu | P2.T4 (FileService + menu) |
| 62 | New Window | P2.T4 (window management) |
| 63 | Rename file | P5.T1 (sidebar) + P5.T3 (command palette) |
| 64 | Trim trailing/leading whitespace | P5.T6.25 |
| 65 | Insert date/time | P5.T6.26 |

**All 65 v1.0 features are accounted for. Zero gaps.**

---

*This task plan is derived from FUNCTIONAL_SPECIFICATION.md v1.2 and CLAUDE.md. Any contradiction should be resolved in favor of CLAUDE.md for architecture decisions and FUNCTIONAL_SPECIFICATION.md for feature behavior.*
