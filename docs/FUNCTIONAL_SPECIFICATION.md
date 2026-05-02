# MYNOTEPAD++ — Functional Specification Document

**Version:** 1.1  
**Date:** 2026-05-03  
**Status:** Draft  
**License:** GPL v3  

---

## 1. PROJECT OVERVIEW

### 1.1 Vision

MYNOTEPAD++ is a free, open-source, native text and code editor built from scratch for macOS (primary), with planned expansion to Linux, Windows, iOS, and Android. It combines the speed and simplicity of Notepad++ with the modern editing power of Sublime Text — filling the long-standing gap of a lightweight, feature-rich, free native code editor on Mac.

### 1.2 What This Is NOT

- NOT a fork or copy of Notepad++ or Sublime Text
- NOT an IDE (no built-in compiler, debugger, or project management)
- NOT an Electron app — fully native on every platform
- NO paid assets, watermarks, or copied icons/branding

### 1.3 Target Users

- Developers who want a fast, lightweight editor on Mac
- Former Notepad++ users who switched to macOS
- Students and hobbyists who need a free, capable code editor
- System administrators editing config files and logs
- Writers who need a distraction-free plain text environment

### 1.4 MVP Platform

**macOS (Apple Silicon native)** — Swift + AppKit + Rust core engine

---

## 2. BRANDING & IDENTITY

### 2.1 Name

**MYNOTEPAD++**

- Unique name, not infringing on "Notepad++" trademark (different prefix, different identity)
- The "++" signifies "enhanced" — a common programming convention, not owned by anyone

### 2.2 Icon Design Guidelines

All icons and visual assets MUST be:

| Rule | Requirement |
|------|-------------|
| Original | Designed from scratch, not derived from any existing editor icon |
| License-free | No stock photos, no paid icon packs, no watermarked assets |
| Self-created | Made using free tools (SF Symbols, Figma free tier, Inkscape, GIMP) |
| Vector-first | All icons as SVG source, export to PNG/ICNS/ICO for platforms |
| Consistent | Unified design language across all platforms |

### 2.3 App Icon Concept

**Design:** A stylized document/page with two "+" symbols — representing the "++" in the name.

**Color palette (original, not copied from any editor):**

| Element | Color | Hex |
|---------|-------|-----|
| Primary | Deep Teal | `#0D7377` |
| Accent | Warm Amber | `#F0A500` |
| Background | Soft Dark | `#1A1A2E` |
| Text/Light | Off-White | `#EAEAEA` |
| Success | Muted Green | `#2ECC71` |
| Warning | Soft Orange | `#E67E22` |
| Error | Calm Red | `#E74C3C` |

**Icon specifications:**

| Platform | Size | Format |
|----------|------|--------|
| macOS | 1024x1024 (auto-scaled) | `.icns` |
| iOS | 1024x1024 | `.png` (Asset Catalog) |
| Windows | 256x256 | `.ico` |
| Linux | 512x512 | `.svg` + `.png` |
| Android | 512x512 (adaptive) | `.xml` + `.png` |

### 2.4 Toolbar & UI Icon Set

All toolbar icons will use:

1. **SF Symbols** (Apple platforms) — free, built into macOS/iOS
2. **Custom SVG icons** (cross-platform) — hand-drawn in Inkscape or Figma (free tier)
3. **No icon fonts from paid libraries**

**Toolbar icon style:**
- Monochrome line icons (1.5pt stroke)
- 24x24dp base size, scalable
- Respects light/dark mode (invert or use semantic colors)

### 2.5 Trademark

- File for trademark on "MYNOTEPAD++" name and logo
- The name and icon are original creations, not derivative works
- No use of any other editor's name, logo, or branding anywhere in the app

---

## 3. CORE FEATURES — MVP (v1.0)

### 3.1 Feature Matrix

Features are categorized by source inspiration and priority.

| # | Feature | Inspired By | Priority | MVP |
|---|---------|-------------|----------|-----|
| 1 | Multi-tab editing | Both | P0 | Yes |
| 2 | Syntax highlighting (50+ languages) | Both | P0 | Yes |
| 3 | Vertical split view | Sublime | P0 | Yes |
| 4 | Horizontal split view | Sublime | P0 | Yes |
| 5 | Smart keyboard shortcuts | Sublime | P0 | Yes |
| 6 | Auto-save | Original | P0 | Yes |
| 7 | Scroll to top / bottom / middle | Original | P0 | Yes |
| 8 | Multi-cursor editing | Sublime | P0 | Yes |
| 9 | Command Palette | Sublime | P0 | Yes |
| 10 | Goto Anything (file/symbol/line) | Sublime | P0 | Yes |
| 11 | Find & Replace (regex) | Both | P0 | Yes |
| 12 | Minimap (code preview sidebar) | Sublime | P1 | Yes |
| 13 | Dark/Light mode | Both | P0 | Yes |
| 14 | Line numbers + code folding | Both | P0 | Yes |
| 15 | Encoding support (UTF-8, UTF-16, etc.) | Notepad++ | P0 | Yes |
| 16 | Distraction-free mode | Sublime | P1 | Yes |
| 17 | Snippet system with tab triggers | Sublime | P1 | Yes |
| 18 | Column/block editing | Both | P1 | Yes |
| 19 | Find in files (multi-file search) | Both | P0 | Yes |
| 20 | Indent guides + bracket matching | Both | P0 | Yes |
| 21 | Word wrap toggle | Both | P0 | Yes |
| 22 | Zoom in/out (font size) | Both | P0 | Yes |
| 23 | Line operations (sort, deduplicate, join) | Notepad++ | P1 | Yes |
| 24 | Auto-indent + smart indent | Both | P0 | Yes |
| 25 | Drag-and-drop file opening | Both | P0 | Yes |
| 26 | Session restore (reopen last files) | Both | P0 | Yes |
| 27 | Project/workspace support | Sublime | P1 | Yes |
| 28 | File tree sidebar | Sublime | P1 | Yes |
| 29 | Code folding (tree-sitter based) | Both | P0 | Yes |
| 30 | Auto-closing brackets & quotes | Both | P0 | Yes |
| 31 | Auto-indent & smart indent (tree-sitter) | Both | P0 | Yes |
| 32 | Tab size auto-detection | Sublime | P0 | Yes |
| 33 | .editorconfig support | VS Code | P0 | Yes |
| 34 | Go to Definition (tree-sitter + heuristic) | Both | P0 | Yes |
| 35 | Open file at line from terminal | Both | P0 | Yes |
| 36 | Bracket pair colorization | VS Code | P0 | Yes |
| 37 | Git gutter (inline diff markers) | Sublime | P1 | Yes |
| 38 | Sticky scroll (scope headers) | VS Code | P1 | Yes |
| 39 | Snippet system with tab triggers | Sublime | P1 | Yes |
| 40 | Project/workspace support | Sublime | P1 | Yes |
| 41 | Column/block editing (detailed) | Both | P1 | Yes |

**v1.1 — Power Features:**

| # | Feature | Inspired By | Priority | v1.1 |
|---|---------|-------------|----------|------|
| 29 | Macro recording & playback | Notepad++ | P1 | Yes |
| 30 | Plugin/extension system (WASM sandbox) | Both | P1 | Yes |
| 31 | SFTP/FTPS remote file editing | Notepad++ | P1 | Yes |
| 32 | Diff / file comparison view | Notepad++ | P1 | Yes |
| 33 | Accessibility (VoiceOver, TalkBack, Narrator, Orca) | Original | P0 | Yes |

**Deferred to v2.0+:**

| Feature | Reason |
|---------|--------|
| Collaborative editing | Requires server infrastructure |
| Remote file browser (SFTP directory listing) | v1.1 supports open-by-path; browser is a UX enhancement |
| Scriptable macros (Lua/JS) | v1.1 has keystroke recording; scripting requires embedding a runtime |
| Plugin UI injection | v1.1 plugins register commands only; custom UI requires layout negotiation |

---

## 4. DETAILED FEATURE SPECIFICATIONS

### 4.1 Multi-Tab Editing

The tab bar is the primary navigation surface for open documents.

**Behavior:**

```
┌──────────────────────────────────────────────────────────────┐
│ [main.rs ×] [config.toml ×] [README.md ●] [+ New Tab]      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  (editor content)                                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

| Feature | Specification |
|---------|--------------|
| Tab display | File name + close button (×); modified indicator (●) |
| Tab overflow | Scroll arrows when tabs exceed width; dropdown menu for all tabs |
| Tab reorder | Drag-and-drop to reorder tabs |
| Tab tear-off | Drag tab out to create new window |
| Tab context menu | Close, Close Others, Close All, Close to the Right, Copy Path, Reveal in Finder |
| Tab duplicate | Open same file in new tab (linked or independent) |
| New tab | `Cmd+N` creates empty tab; `Cmd+T` opens Goto Anything |
| Close tab | `Cmd+W`; if auto-save enabled (default): save silently + close instantly. If auto-save disabled + untitled file: prompt Save As. Never prompt for named files. |
| Cycle tabs | `Ctrl+Tab` / `Ctrl+Shift+Tab` (MRU order) |
| Jump to tab | `Cmd+1` through `Cmd+9` for first 9 tabs |
| Pin tabs | Right-click > Pin (pinned tabs are smaller, stay left) |
| Max tabs | No hard limit; memory is the only constraint |

### 4.2 Split View (Vertical & Horizontal)

Split view allows viewing multiple files or multiple locations in the same file simultaneously.

**Layouts:**

```
SINGLE (default)         VERTICAL SPLIT           HORIZONTAL SPLIT
┌──────────────┐        ┌───────┬───────┐        ┌──────────────┐
│              │        │       │       │        │              │
│              │        │       │       │        │    Pane 1    │
│   Pane 1     │        │ Pane1 │ Pane2 │        ├──────────────┤
│              │        │       │       │        │              │
│              │        │       │       │        │    Pane 2    │
└──────────────┘        └───────┴───────┘        └──────────────┘

2x2 GRID                 3-COLUMN
┌───────┬───────┐        ┌─────┬─────┬─────┐
│       │       │        │     │     │     │
│ Pane1 │ Pane2 │        │  1  │  2  │  3  │
├───────┼───────┤        │     │     │     │
│       │       │        │     │     │     │
│ Pane3 │ Pane4 │        │     │     │     │
└───────┴───────┘        └─────┴─────┴─────┘
```

| Feature | Specification |
|---------|--------------|
| Vertical split | `Cmd+\` — split editor right |
| Horizontal split | `Cmd+Shift+\` — split editor down |
| 2x2 grid | `Cmd+Option+\` — four equal panes |
| 3-column | Via View menu or Command Palette |
| Resize panes | Drag divider with mouse; minimum pane width 200px |
| Move file between panes | Drag tab from one pane to another |
| Same file in multiple panes | Supported — edits sync in real-time |
| Close pane | Close all tabs in pane, or `Cmd+Shift+W` |
| Focus pane | `Cmd+Option+Arrow` to move focus between panes |
| Independent scroll | Each pane scrolls independently |
| Independent cursor | Each pane has its own cursor position |
| Pane-specific tab bar | Each pane has its own tab bar |

### 4.3 Scroll Navigation (Top / Bottom / Middle)

Quick scroll commands for navigating large files without losing context.

**Scroll-To Commands:**

| Action | Shortcut | Behavior |
|--------|----------|----------|
| Scroll to top of file | `Cmd+Home` or `Cmd+Up` (with fn) | Cursor moves to line 1, column 1 |
| Scroll to bottom of file | `Cmd+End` or `Cmd+Down` (with fn) | Cursor moves to last line |
| Scroll to middle of file | `Cmd+Shift+M` | Cursor moves to line `totalLines / 2` |
| Center current line on screen | `Ctrl+L` | Scrolls view so cursor line is vertically centered |
| Scroll up (no cursor move) | `Ctrl+Up` | View scrolls up 1 line; cursor stays |
| Scroll down (no cursor move) | `Ctrl+Down` | View scrolls down 1 line; cursor stays |
| Page up | `Fn+Up` / `Page Up` | Scroll one viewport height up |
| Page down | `Fn+Down` / `Page Down` | Scroll one viewport height down |
| Scroll to specific line | `Ctrl+G` → enter line number | Jump to exact line |
| Scroll to percentage | Command Palette → "Go to 50%" | Jump to percentage of file |

**Scroll Behavior Rules:**
- Smooth scrolling with momentum (native macOS feel)
- Scroll position preserved per-tab (switching tabs restores scroll position)
- Minimap (if visible) highlights current viewport
- Scroll bar shows annotations: modified lines (yellow), errors (red), search matches (orange)

### 4.4 Smart Keyboard Shortcuts

All shortcuts follow macOS conventions (`Cmd` instead of `Ctrl`). Every shortcut is **remappable** via Preferences.

#### 4.4.1 File Operations

| Action | Shortcut |
|--------|----------|
| New file | `Cmd+N` |
| Open file | `Cmd+O` |
| Open folder | `Cmd+Shift+O` |
| Save | `Cmd+S` |
| Save As | `Cmd+Shift+S` |
| Save All | `Cmd+Option+S` |
| Close tab | `Cmd+W` |
| Close window | `Cmd+Shift+W` |
| Close all tabs | `Cmd+Option+W` |
| Reopen closed tab | `Cmd+Shift+T` |
| Print | `Cmd+Shift+P+R` (chord) or via File menu — `Cmd+P` is reserved for Goto Anything |

#### 4.4.2 Editing

| Action | Shortcut |
|--------|----------|
| Undo | `Cmd+Z` |
| Redo | `Cmd+Shift+Z` |
| Cut | `Cmd+X` |
| Copy | `Cmd+C` |
| Paste | `Cmd+V` |
| Select all | `Cmd+A` |
| Select line | `Cmd+L` |
| Select word | `Cmd+D` (also adds next occurrence to multi-cursor) |
| Select all occurrences | `Cmd+Shift+D` |
| Duplicate line | `Cmd+Shift+Down` |
| Delete line | `Cmd+Shift+K` |
| Move line up | `Option+Up` |
| Move line down | `Option+Down` |
| Indent | `Cmd+]` |
| Outdent | `Cmd+[` |
| Toggle comment | `Cmd+/` |
| Toggle block comment | `Cmd+Shift+/` |
| Join lines | `Cmd+J` |
| Uppercase | `Cmd+K, Cmd+U` (chord) |
| Lowercase | `Cmd+K, Cmd+L` (chord) |
| Sort lines | `Cmd+Shift+L` (via Command Palette) |

#### 4.4.3 Multi-Cursor & Selection

| Action | Shortcut |
|--------|----------|
| Add cursor above | `Cmd+Option+Up` |
| Add cursor below | `Cmd+Option+Down` |
| Add cursor at click | `Option+Click` |
| Select next occurrence | `Cmd+D` |
| Skip occurrence | `Cmd+K, Cmd+D` (chord) |
| Select all occurrences | `Cmd+Ctrl+G` |
| Column select | `Option+Shift+Drag` |
| Split selection into lines | `Cmd+Shift+L` |

#### 4.4.4 Navigation

| Action | Shortcut |
|--------|----------|
| Goto Anything | `Cmd+P` |
| Command Palette | `Cmd+Shift+P` |
| Goto line | `Ctrl+G` |
| Goto symbol | `Cmd+R` |
| Goto symbol in project | `Cmd+Shift+R` |
| Go to definition | `Cmd+Click` or `F12` |
| Go back | `Ctrl+-` |
| Go forward | `Ctrl+Shift+-` |
| Next tab | `Ctrl+Tab` |
| Previous tab | `Ctrl+Shift+Tab` |
| Focus sidebar | `Cmd+0` |
| Focus editor | `Cmd+1` through `Cmd+9` |

#### 4.4.5 View & Layout

| Action | Shortcut |
|--------|----------|
| Toggle sidebar | `Cmd+B` |
| Toggle minimap | `Cmd+Shift+M` |
| Vertical split | `Cmd+\` |
| Horizontal split | `Cmd+Shift+\` |
| Grid layout (2x2) | `Cmd+Option+\` |
| Single pane | `Cmd+Option+1` |
| Distraction-free mode | `Cmd+Ctrl+F` |
| Full screen | `Cmd+Ctrl+F` (native macOS) or `Fn+F` |
| Zoom in | `Cmd+=` |
| Zoom out | `Cmd+-` |
| Reset zoom | `Cmd+0` |
| Toggle word wrap | `Option+Z` |
| Toggle line numbers | Via Command Palette |
| Toggle invisible characters | Via Command Palette |

#### 4.4.6 Search & Replace

| Action | Shortcut |
|--------|----------|
| Find | `Cmd+F` |
| Find and replace | `Cmd+Option+F` |
| Find in files | `Cmd+Shift+F` |
| Find next | `Cmd+G` or `Enter` (in find bar) |
| Find previous | `Cmd+Shift+G` or `Shift+Enter` |
| Toggle regex mode | `Option+Cmd+R` (in find bar) |
| Toggle case sensitive | `Option+Cmd+C` (in find bar) |
| Toggle whole word | `Option+Cmd+W` (in find bar) |
| Use selection for find | `Cmd+E` |
| Replace | `Cmd+Shift+1` (in replace bar) |
| Replace all | `Cmd+Shift+Enter` (in replace bar) |

#### 4.4.7 Shortcut Customization

```json
// ~/.mynotepadpp/keybindings.json
{
  "bindings": [
    {
      "key": "cmd+shift+m",
      "command": "editor.scrollToMiddle",
      "when": "editorFocus"
    },
    {
      "key": "cmd+k cmd+u",
      "command": "editor.uppercaseSelection",
      "when": "editorHasSelection"
    }
  ]
}
```

- All shortcuts are remappable
- Supports chord shortcuts (two-key sequences like `Cmd+K, Cmd+U`)
- Context-aware: shortcuts can be scoped to `editorFocus`, `sidebarFocus`, `findBarFocus`, etc.
- Import/export keybinding profiles
- Preset profiles: "Sublime Text", "VS Code", "Notepad++", "Vim", "Emacs" mappings

### 4.5 Auto-Save

Auto-save eliminates data loss without user intervention.

**Behavior:**

| Setting | Default | Options |
|---------|---------|---------|
| Auto-save enabled | `true` | `true` / `false` |
| Save trigger | Focus lost + timer | `onFocusLost`, `onTimer`, `both` |
| Timer interval | 30 seconds | 5s — 300s (configurable) |
| Save delay after typing stops | 1 second | 0.5s — 10s (configurable) |
| Save on window close | `true` | `true` / `false` |
| Save unnamed files | To temp dir | Temp dir path configurable |
| Hot exit (save session on quit) | `true` | `true` / `false` |

**Auto-Save Rules:**

1. **Modified indicator**: Tab shows (●) when file has unsaved changes
2. **After auto-save**: Indicator clears, file is saved silently (no flash or notification)
3. **Conflict detection**: If file changes on disk while open, prompt user: "File changed externally. Reload?"
4. **Unnamed files**: Auto-saved to `~/.mynotepadpp/recovery/` with timestamp
5. **Crash recovery**: On next launch, offer to restore unsaved files from recovery directory
6. **Large files**: Auto-save writes to temp file first, then atomic rename (no partial writes)
7. **Read-only files**: Auto-save is disabled; show lock icon on tab

**Hot Exit:**
When the app quits (even without saving), all open tabs, cursor positions, scroll positions, undo history, and unsaved changes are preserved. On next launch, everything is restored exactly as it was.

```
Platform-specific config directory:
  macOS:   ~/Library/Application Support/mynotepadpp/
  Linux:   $XDG_CONFIG_HOME/mynotepadpp/ (~/.config/ fallback)
  Windows: %APPDATA%\mynotepadpp\
  iOS:     App container Documents/
  Android: App-internal storage

Directory structure (same on all platforms):
<config-dir>/
├── recovery/
│   ├── untitled-2026-05-03-143022.txt
│   └── untitled-2026-05-03-150145.rs
├── sessions/
│   └── last-session.json    ← tab state, cursor, scroll, undo
└── settings/
    └── preferences.json
```

### 4.6 Command Palette

The Command Palette is the universal entry point for every action in the editor.

**Activation:** `Cmd+Shift+P`

```
┌──────────────────────────────────────────────┐
│ > Toggle Word Wrap                        ▏  │
├──────────────────────────────────────────────┤
│   Toggle Word Wrap              Option+Z     │
│   Toggle Minimap            Cmd+Shift+M      │
│   Toggle Sidebar                  Cmd+B      │
│   Toggle Line Numbers                        │
│   Toggle Distraction Free     Cmd+Ctrl+F     │
│   Change Language Mode                       │
│   Change Encoding                            │
│   Change Line Endings (LF/CRLF)             │
│   Sort Lines Ascending                       │
│   Sort Lines Descending                      │
│   Remove Duplicate Lines                     │
│   Trim Trailing Whitespace                   │
│   Convert Indentation to Tabs                │
│   Convert Indentation to Spaces              │
│   Install Theme...                           │
└──────────────────────────────────────────────┘
```

| Feature | Specification |
|---------|--------------|
| Fuzzy matching | Type "tww" matches "Toggle Word Wrap" |
| Recently used | Most recent commands appear first |
| Shortcut display | Show bound shortcut next to each command |
| Categories | File, Edit, View, Selection, Find, Go, Preferences |
| Extensible | Plugins can register commands to the palette |

### 4.7 Goto Anything

**Activation:** `Cmd+P`

A unified fuzzy file/symbol/line navigator.

| Prefix | Action | Example |
|--------|--------|---------|
| (none) | Fuzzy file search | `main.rs` |
| `:` | Go to line | `:42` |
| `@` | Go to symbol | `@handleClick` |
| `#` | Search in file | `#TODO` |
| Chained | File + line | `main.rs:42` |
| Chained | File + symbol | `main.rs@main` |

**Behavior:**
- Shows preview of file as you navigate (peek without opening)
- Press `Enter` to open; press `Esc` to cancel
- Recently opened files appear first
- File icons indicate language/type

### 4.8 Multi-Cursor Editing

| Feature | Specification |
|---------|--------------|
| Add cursor | `Option+Click` anywhere |
| Add cursor above/below | `Cmd+Option+Up/Down` |
| Select next match | `Cmd+D` — selects next occurrence of current word/selection |
| Skip match | `Cmd+K, Cmd+D` — skip current, find next |
| Select all matches | `Cmd+Ctrl+G` — cursors on every occurrence |
| Column select | `Option+Shift+Drag` — rectangular selection |
| Type at all cursors | All cursors receive same input simultaneously |
| Undo per cursor | `Cmd+Z` undoes last action at all cursors |
| Exit multi-cursor | `Esc` — returns to single cursor |

### 4.9 Minimap

A zoomed-out preview of the entire file shown in the right margin.

```
┌────────────────────────────────────┬────┐
│                                    │▓▓▓▓│ ← current viewport (highlighted)
│                                    │░░░░│
│     (editor content)               │░░░░│
│                                    │░░░░│
│                                    │░░░░│
│                                    │░░░░│
└────────────────────────────────────┴────┘
```

| Feature | Specification |
|---------|--------------|
| Toggle | `Cmd+Shift+M` or View menu |
| Width | 80px default, resizable (60–120px) |
| Click to scroll | Click anywhere on minimap to jump there |
| Drag to scroll | Drag the viewport highlight to scroll |
| Syntax colors | Minimap shows syntax highlighting |
| Search highlights | Search matches shown as colored markers |
| Modified lines | Yellow markers for unsaved changes |
| Default state | Visible (can be set to hidden in preferences) |

### 4.10 Find & Replace

**Find Bar** (`Cmd+F`):

```
┌──────────────────────────────────────────────────────────────┐
│ Find: [search term              ] [Aa] [.*] [W] [↑] [↓] [×]│
│ Repl: [replacement              ] [Replace] [Replace All]    │
│ Found: 14 matches (3 in selection)                           │
└──────────────────────────────────────────────────────────────┘
```

| Toggle | Icon | Shortcut | Function |
|--------|------|----------|----------|
| `[Aa]` | Case | `Option+Cmd+C` | Case-sensitive matching |
| `[.*]` | Regex | `Option+Cmd+R` | Regular expression mode |
| `[W]` | Word | `Option+Cmd+W` | Whole word matching |

**Find in Files** (`Cmd+Shift+F`):

| Feature | Specification |
|---------|--------------|
| Search scope | Current file, open files, folder, project |
| File filter | Include/exclude by glob pattern (e.g., `*.rs`, `!node_modules`) |
| Results panel | Tree view grouped by file, with line previews |
| Replace in files | Replace across multiple files with confirmation |
| Regex groups | Support `$1`, `$2` backreferences in replacement |
| Live preview | Show replacement preview before applying |
| History | Search history dropdown (last 50 searches) |

### 4.11 Syntax Highlighting

Powered by **tree-sitter** for accurate, incremental parsing.

**MVP Languages (50+):**

| Category | Languages |
|----------|-----------|
| Systems | C, C++, Rust, Go, Zig |
| Web | HTML, CSS, JavaScript, TypeScript, JSX, TSX |
| Backend | Python, Ruby, Java, Kotlin, C#, PHP, Scala |
| Scripting | Bash, Zsh, PowerShell, Lua, Perl |
| Data | JSON, YAML, TOML, XML, CSV, SQL |
| Markup | Markdown, LaTeX, reStructuredText |
| Config | Dockerfile, Nginx, Apache, .env, .gitignore, Makefile |
| Mobile | Swift, Objective-C, Dart |
| Other | R, MATLAB, Haskell, Elixir, Clojure, Assembly |

**Theme system:**
- Bundled themes: 5 light + 5 dark (all original, GPL v3 licensed)
- Theme format: JSON (compatible structure, but original themes)
- Custom themes: user can create and import
- Live preview when browsing themes

### 4.12 File Tree Sidebar

```
┌─────────────────┬────────────────────────────────────────────┐
│ MYNOTEPAD++      │                                            │
│ ├── src/         │                                            │
│ │   ├── main.rs  │     (editor content)                       │
│ │   ├── lib.rs   │                                            │
│ │   └── utils/   │                                            │
│ ├── tests/       │                                            │
│ ├── Cargo.toml   │                                            │
│ └── README.md    │                                            │
└─────────────────┴────────────────────────────────────────────┘
```

| Feature | Specification |
|---------|--------------|
| Toggle | `Cmd+B` |
| File operations | New file, new folder, rename, delete, duplicate |
| Context menu | Open, Open to Side, Reveal in Finder, Copy Path, Copy Relative Path |
| Search filter | Type to filter files in sidebar |
| File icons | Language-specific icons (original SVG set) |
| Git status | Color-coded: green (new), yellow (modified), red (deleted) |
| Drag and drop | Reorder and move files/folders |
| Multi-select | `Cmd+Click` to select multiple files |

### 4.13 Distraction-Free Mode

**Activation:** `Cmd+Ctrl+F`

| Element | Behavior |
|---------|----------|
| Tab bar | Hidden |
| Sidebar | Hidden |
| Minimap | Hidden |
| Status bar | Hidden |
| Menu bar | Auto-hides (appears on mouse hover at top) |
| Editor | Centered with configurable max width (80/100/120 chars or full) |
| Background | Fades to solid color |
| Line numbers | Hidden (configurable to keep) |
| Exit | `Esc` or `Cmd+Ctrl+F` |

### 4.14 Encoding & Line Endings

| Feature | Specification |
|---------|--------------|
| Auto-detect | Detect encoding on file open (using `encoding_rs`) |
| Status bar | Current encoding shown in status bar (click to change) |
| Supported encodings | UTF-8, UTF-8 BOM, UTF-16 LE, UTF-16 BE, ASCII, ISO-8859-1 through -15, Windows-1250 through -1258, Shift-JIS, EUC-JP, EUC-KR, GB2312, Big5 |
| Convert encoding | Reopen with Encoding / Save with Encoding |
| Line endings | LF (Unix/Mac), CRLF (Windows), CR (legacy Mac) |
| Line ending display | Status bar shows current; click to change |
| Default | UTF-8, LF for new files |

### 4.15 Macro Recording & Playback

Record sequences of editor actions and replay them on demand.

**Recording Model:**

Macros record *editor commands* (e.g., `MoveCursorRight`, `InsertText("foo")`, `DeleteLine`), not raw keycodes. This makes macros portable across platforms and keybinding profiles.

```
┌──────────────────────────────────────────────────────────────┐
│  Ln 1, Col 1 │ UTF-8 │ LF │ Rust │ ● RECORDING MACRO       │  ← Status Bar
└──────────────────────────────────────────────────────────────┘
```

**Keyboard Shortcuts:**

| Action | Shortcut | Behavior |
|--------|----------|----------|
| Start/stop recording | `Cmd+Shift+R` | Toggle; status bar shows red recording indicator |
| Play last macro | `Cmd+Shift+E` | Replay the most recently recorded macro |
| Play macro N times | Command Palette → "Run Macro N Times" | Prompt for count, then replay |
| Save macro | Command Palette → "Save Macro" | Name it and optionally assign a shortcut |
| Edit macros | Preferences → Macros tab | List, rename, delete, re-assign shortcuts, export/import |

**Behavior Rules:**

| Rule | Specification |
|------|--------------|
| Undo integration | Entire macro playback = single undo group (`Cmd+Z` undoes all) |
| Nesting | Cannot record a macro while recording — starting new recording stops current |
| Infinite-loop guard | Max 10,000 commands per playback (configurable); halts with warning if exceeded |
| Scope | Macros operate on active buffer only — no file I/O, no network, no process execution |
| Persistence | Saved as JSON in `~/.mynotepadpp/macros/macro_name.json` |
| Portability | Macros are platform-independent (command-based, not keycode-based) |

**Macro File Format:**

```json
{
  "name": "Add semicolons to selection",
  "shortcut": "Cmd+Shift+;",
  "commands": [
    { "type": "MoveToEndOfLine" },
    { "type": "InsertText", "text": ";" },
    { "type": "MoveCursorDown", "count": 1 },
    { "type": "MoveToEndOfLine" },
    { "type": "InsertText", "text": ";" }
  ],
  "created": "2026-05-03T14:30:00Z",
  "modified": "2026-05-03T14:30:00Z"
}
```

### 4.16 Plugin / Extension System

Plugins extend editor functionality via a sandboxed WASM runtime. Security and stability are prioritized over flexibility.

**Architecture:**

```
┌─────────────────┐     ┌──────────────────────────────────┐
│  Plugin (WASM)  │────▶│  Plugin Host (wasmtime sandbox)  │
│  - plugin.wasm  │     │  - Memory limit: 64MB            │
│  - plugin.toml  │     │  - CPU limit: 100ms/event        │
│                 │     │  - No FS / network / process      │
└─────────────────┘     └──────────┬───────────────────────┘
                                   │ Host API
                        ┌──────────▼───────────────────────┐
                        │       Editor Core                 │
                        │  - Buffer read/write              │
                        │  - Command registration           │
                        │  - Event subscription             │
                        └───────────────────────────────────┘
```

**Plugin Manifest:**

```toml
# plugin.toml
[plugin]
name = "trailing-whitespace-trimmer"
version = "1.0.0"
author = "Author Name"
license = "GPL-3.0"
description = "Trims trailing whitespace on save"
entry = "plugin.wasm"

[permissions]
buffer_read = true
buffer_write = true
command_palette = true
events = ["on_save", "on_open"]
```

**Plugin Capabilities (v1.1):**

| Capability | Description |
|-----------|-------------|
| `buffer_read` | Read active buffer content |
| `buffer_write` | Modify active buffer content |
| `command_palette` | Register commands in the Command Palette |
| `events` | Subscribe to editor events (`on_open`, `on_save`, `on_close`, `on_cursor_move`, `on_text_change`) |
| `themes` | Register custom syntax themes |
| `status_bar` | Display text in a status bar segment |

**Not available in v1.1:** filesystem access, network access, custom UI panels, inter-plugin communication.

**Plugin Management UI:**

| Action | Access |
|--------|--------|
| Install plugin | Command Palette → "Install Plugin" (browse directory) |
| Enable/disable | Preferences → Plugins tab |
| View permissions | Preferences → Plugins → plugin detail |
| Uninstall | Preferences → Plugins → "Remove" |

**Safety Rules:**
- Crashing plugin is automatically unloaded; editor continues
- Plugin cannot block the UI thread (event handlers have CPU time limit)
- All plugins must declare GPL-v3-compatible license
- No native code — WASM only

### 4.17 SFTP / FTPS Remote File Editing

Open and edit files on remote servers via secure file transfer protocols.

**Supported Protocols:**

| Protocol | Security | Default Port |
|----------|----------|-------------|
| SFTP (SSH File Transfer) | SSH v2 | 22 |
| FTPS (FTP over TLS) | TLS 1.2+ | 990 (implicit) / 21 (explicit) |
| Plain FTP | **NOT SUPPORTED** | — |

**Connection UI:**

```
┌──────────────────────────────────────────────┐
│ Open Remote File                              │
├──────────────────────────────────────────────┤
│ Protocol:  [SFTP ▾]                           │
│ Host:      [example.com              ]        │
│ Port:      [22   ]                            │
│ Username:  [deploy                   ]        │
│ Auth:      [● SSH Key  ○ Password    ]        │
│ Key file:  [~/.ssh/id_ed25519   ] [Browse]    │
│ Remote path: [/var/www/app/config.toml]       │
│                                               │
│ [Save Connection] [Cancel] [Connect]          │
└──────────────────────────────────────────────┘
```

**Behavior:**

| Feature | Specification |
|---------|--------------|
| Activation | File → Open Remote (`Cmd+Shift+O+R`) or Command Palette |
| Authentication | SSH key (RSA, Ed25519) + ssh-agent; password (stored in native keychain) |
| Workflow | Download to temp file → open in editor → upload on save |
| Conflict detection | Check remote file mtime before upload; prompt if changed |
| Connection indicator | Tab title shows `[remote]` prefix; status bar shows connection icon |
| Saved connections | Preferences → Remote Connections; stored in config (no passwords in plaintext) |
| Offline resilience | If connection drops, local temp copy remains editable; retry upload when reconnected |
| Session timeout | Configurable idle disconnect (default 15 minutes) |
| Host key verification | First-connect prompt to trust; pinned in `known_hosts` thereafter |
| No remote browsing (v1.1) | User provides full path; directory browsing is a v2.0 feature |

**Security Rules:**
- No fallback to insecure protocols
- Credentials stored in platform-native secure storage only (Keychain, libsecret, Credential Manager, EncryptedSharedPreferences)
- No auto-connect on app launch — user must explicitly open a remote connection
- No network scanning or broadcast

### 4.18 Diff / File Comparison View

Compare two files or two versions of the same file side-by-side or inline.

**Activation:**

| Method | Access |
|--------|--------|
| Compare two open files | Right-click tab → "Compare with..." → select other tab |
| Compare with saved version | Command Palette → "Compare with Saved" (working buffer vs. disk) |
| Compare with clipboard | Command Palette → "Compare with Clipboard" |
| Compare two files from disk | File → Compare Files → select two files |

**View Modes:**

```
SIDE-BY-SIDE                                    INLINE (UNIFIED)
┌───────────────────┬───────────────────┐      ┌──────────────────────────────────┐
│ original.rs       │ modified.rs       │      │ original.rs ↔ modified.rs        │
├───────────────────┼───────────────────┤      ├──────────────────────────────────┤
│  1  fn main() {   │  1  fn main() {   │      │  1    fn main() {                │
│  2-   println!("h │  2+   println!("w │      │  2  - println!("hello");         │
│  3  }             │  3  }             │      │  2  + println!("world");         │
│                   │  4+ fn helper() { │      │  3    }                          │
│                   │  5+ }             │      │  4  + fn helper() {              │
└───────────────────┴───────────────────┘      │  5  + }                          │
                                                └──────────────────────────────────┘
```

**Features:**

| Feature | Specification |
|---------|--------------|
| Toggle view mode | Toolbar button or `Cmd+Shift+D` to switch side-by-side ↔ inline |
| Navigate hunks | `F7` (next diff) / `Shift+F7` (previous diff) |
| Merge direction | Click gutter arrow to copy hunk left→right or right→left |
| Synchronized scrolling | Both panes scroll together (toggle to unlock) |
| Syntax highlighting | Both sides retain full highlighting |
| Word-level diff | Changed words within a line are highlighted with a different background |
| Ignore whitespace | Toggle to suppress whitespace-only changes |
| Ignore case | Toggle for case-insensitive comparison |
| Diff summary | Status bar: "5 additions, 3 deletions, 2 modifications" |
| Large file support | Stream-diff for files > 10MB; show first N hunks with "Load More" |
| Gutter indicators | Green bar (added), red bar (removed), blue bar (modified) |
| Accessibility | Diff hunks navigable by keyboard; screen reader announces "Added 2 lines after line 3" |

### 4.19 Code Folding

Collapse regions of code to focus on structure. **P0 — must be in v1.0.**

| Feature | Specification |
|---------|--------------|
| Fold source | Tree-sitter syntax scopes (functions, classes, blocks, imports, comments) |
| Fold gutter | Triangles in gutter: ▶ (folded), ▼ (expanded). Click to toggle. |
| Fold preview | Folded region shows first line + `... (N lines)` indicator |
| Fold all | `Cmd+K, Cmd+0` — fold everything |
| Unfold all | `Cmd+K, Cmd+J` — unfold everything |
| Fold at level N | `Cmd+K, Cmd+1` through `Cmd+K, Cmd+5` — fold at indent level N |
| Region markers | `// #region Name` / `// #endregion` for manual fold regions |
| Find interaction | Finding text in a folded region auto-unfolds that region |
| Line number skip | Folded lines skipped in gutter numbers (shows line 10, then line 50 if 40 lines folded) |
| Keyboard | `Cmd+Option+[` fold, `Cmd+Option+]` unfold at cursor |

### 4.20 Auto-Closing Brackets & Quotes

Automatically insert matching closing characters. **P0 — fundamental editing feature.**

| Trigger | Result | Condition |
|---------|--------|-----------|
| Type `(` | Insert `)`, cursor between | Always in code files |
| Type `[` | Insert `]`, cursor between | Always |
| Type `{` | Insert `}`, cursor between. If at end of line, also add newline + indent. | Always |
| Type `"` | Insert `"`, cursor between | Not inside a string (tree-sitter aware) |
| Type `'` | Insert `'`, cursor between | Not inside a string; not in languages where `'` is an operator (Rust lifetimes) |
| Type `` ` `` | Insert `` ` ``, cursor between | Markdown, JS/TS template literals |
| Selection + `(` | Wrap selection: `(selection)` | When text is selected |
| Selection + `"` | Wrap selection: `"selection"` | When text is selected |
| Backspace on `()` | Delete both brackets if empty pair | Cursor between empty pair |
| Overtype `)` | Skip over existing `)` instead of inserting duplicate | Cursor before matching `)` |

### 4.21 Auto-Indent & Smart Indent

Language-aware automatic indentation. **P0.**

| Trigger | Behavior |
|---------|----------|
| Press Enter after `{` | New line indented one level; closing `}` on next line at original indent |
| Press Enter after `:` (Python) | New line indented one level |
| Type `}` / `end` / `fi` | Auto-outdent to matching open block level |
| Paste code | Auto-detect and adjust indent to match surrounding context |
| Tab at line start | Indent line/selection one level |
| Shift+Tab | Outdent line/selection one level |
| Indent source | Tree-sitter scope analysis (preferred) or heuristic (fallback) |

### 4.22 Tab Size Auto-Detection

On file open, detect the file's existing indent style and size. **P0.**

| Detection | Method |
|-----------|--------|
| Tabs vs spaces | Scan first 100 lines. If > 50% use tabs → tabs mode. Else spaces. |
| Tab size | For space-indented files: find most common indent delta (2, 4, 8). Default: 4. |
| Override | Status bar shows detected style (click to change). `.editorconfig` overrides detection. |
| New files | Use global preference (default: 4 spaces) |

### 4.23 .editorconfig Support

Read `.editorconfig` files per the [EditorConfig spec](https://editorconfig.org). **P0.**

| Property | Supported |
|----------|-----------|
| `indent_style` | `tab` / `space` |
| `indent_size` | Number |
| `tab_width` | Number |
| `end_of_line` | `lf` / `crlf` / `cr` |
| `charset` | `utf-8` / `utf-8-bom` / `latin1` / `utf-16be` / `utf-16le` |
| `trim_trailing_whitespace` | `true` / `false` |
| `insert_final_newline` | `true` / `false` |
| `max_line_length` | Number (for wrap guides) |
| `root` | `true` stops upward search |

- Search upward from file directory. Closest `.editorconfig` wins for each property.
- `.editorconfig` overrides global preferences but NOT user's per-file override.

### 4.24 Go to Definition

Navigate to where a symbol is defined. **P0 — has shortcut, needs implementation spec.**

| Method | How it works |
|--------|-------------|
| Tree-sitter scope analysis | Find the definition node for the symbol under cursor within the same file. Works for local variables, functions, classes. |
| Project-wide heuristic | `grep` for `fn SYMBOL`, `def SYMBOL`, `class SYMBOL`, `function SYMBOL` patterns across project files. Rank by exact match. |
| Navigation stack | Every Go-to-Definition pushes current position onto a stack. `Ctrl+-` pops back. `Ctrl+Shift+-` goes forward. Stack depth: 100. |

**NOT LSP-based** — this editor has no language server. Tree-sitter provides 80% accuracy for same-file definitions. Cross-file is heuristic.

### 4.25 Open File at Line from Terminal

```bash
# Open file
mynotepadpp file.rs

# Open file at line
mynotepadpp file.rs:42

# Open file at line and column
mynotepadpp file.rs:42:15

# Open multiple files
mynotepadpp file1.rs file2.rs file3.rs

# Open folder as project
mynotepadpp /path/to/project/
```

Register as macOS URL handler: `mynotepadpp://open?file=/path&line=42&col=15`

### 4.26 Bracket Pair Colorization

Nested brackets get distinct colors for visual clarity. **Enabled by default.**

| Level | Default colors (dark theme) | Default colors (light theme) |
|-------|-----------------------------|------------------------------|
| 1 | Gold `#FFD700` | Blue `#0000FF` |
| 2 | Violet `#DA70D6` | Green `#008000` |
| 3 | Cyan `#00CED1` | Red `#B22222` |
| 4+ | Cycle from level 1 | Cycle from level 1 |

- Powered by tree-sitter (accurate bracket matching, not regex)
- Configurable: on/off toggle, custom color cycle
- Applies to: `()`, `[]`, `{}`, `<>` (in languages where `<>` is a bracket, e.g., generics)

### 4.27 Git Gutter (Inline Diff Markers)

Show live inline markers for changes vs. git HEAD. **Distinct from Diff View (4.18).**

```
  1  │ fn main() {                    ← unchanged (no marker)
  2  ┃ let x = 42;                    ← green bar: added line
  3  ┃ let y = x + 1;                 ← green bar: added line
  4  ▌ println!("{}", y);             ← blue bar: modified line
  5  │ }                               ← unchanged
     ▸                                 ← red triangle: line(s) deleted here
```

| Feature | Specification |
|---------|--------------|
| Gutter markers | Green bar (added), blue bar (modified), red triangle (deleted) |
| Hover | Hover on marker shows original content in popup |
| Revert hunk | Click gutter marker → "Revert Change" option |
| Navigate | `Option+Cmd+F5` (next change), `Option+Cmd+Shift+F5` (previous change) |
| Computation | Diff current buffer vs. git HEAD on background thread. Debounced: re-diff 500ms after last edit. |
| Requirement | Requires git repo; gracefully absent for non-git files |

### 4.28 Sticky Scroll (Scope Headers)

When scrolling through a long function/class, the enclosing scope headers stay pinned at the top.

```
┌──────────────────────────────────────┐
│ ░ class UserService {                │  ← sticky: class header
│ ░   fn validate_input(              │  ← sticky: function header
│ ░     &self, input: &str            │  ← sticky: continuation
│──────────────────────────────────────│
│  45      if input.is_empty() {      │  ← actual scroll position
│  46          return Err("empty");   │
│  47      }                          │
│  48      let trimmed = input.trim();│
```

| Feature | Specification |
|---------|--------------|
| Source | Tree-sitter scope nodes (function, class, module, impl, struct, enum) |
| Max lines | 5 sticky lines maximum (configurable) |
| Click | Click sticky header to scroll to that scope's start |
| Toggle | View menu → "Sticky Scroll" or Command Palette |
| Default | Enabled |

### 4.29 Snippet System

Tab-trigger expandable code templates. **P1.**

**Snippet format:**
```json
{
  "name": "Function",
  "prefix": "fn",
  "scope": "rust",
  "body": "fn ${1:name}(${2:params}) -> ${3:ReturnType} {\n\t${0}\n}"
}
```

| Feature | Specification |
|---------|--------------|
| Tab trigger | Type prefix, press Tab → expand to snippet body |
| Tabstops | `$1`, `$2`, `$0` (final position). Tab navigates between tabstops. |
| Placeholders | `${1:default_text}` — highlighted, replaced on type |
| Variables | `$TM_FILENAME`, `$TM_FILEPATH`, `$CURRENT_YEAR`, `$CLIPBOARD`, `$TM_SELECTED_TEXT` |
| Mirror | Same tabstop number = mirrored (type once, appears in multiple places) |
| Scope | Language-specific snippets (only show Rust snippets in `.rs` files) |
| Storage | `~/.mynotepadpp/snippets/{language}.json` |
| Built-in | Ship 5-10 snippets per major language (function, class, if/else, loop, import) |
| Custom | User-created via Preferences → Snippets |

### 4.30 Project / Workspace Support

**P1 — enables multi-root, per-project settings.**

| Feature | Specification |
|---------|--------------|
| Open folder | `Cmd+Shift+O` → folder picker → opens as project |
| Project file | `.mynotepadpp/project.json` in project root (auto-created on first project open) |
| Per-project settings | Override global preferences: indent style, tab size, theme, excluded patterns |
| Multi-root | A workspace can include multiple folders (configured in project file) |
| Sidebar root | File tree shows project root(s) at top level |
| Search scope | "Find in Files" defaults to project root |
| Recent projects | File → Open Recent → Projects section |
| `.editorconfig` interaction | `.editorconfig` overrides project settings for matching files |

### 4.31 Column / Block Editing

**P1 — detailed specification.**

| Feature | Specification |
|---------|--------------|
| Activate | `Option+Shift+Drag` for mouse; `Option+Shift+Arrow` for keyboard |
| Typing | Characters inserted at every cursor in the column selection |
| Short lines | If a line is shorter than the column cursor position, pad with spaces to reach the cursor column |
| Paste | Paste N lines into N column cursors (one line per cursor). If clipboard has 1 line, paste at all cursors. |
| Cut/Copy | Cut/copy extracts rectangular block. Paste inserts as rectangle. |
| Tab | Inserts tab/spaces at all column cursors |
| Undo | Entire column operation = single undo group |

### 4.32 Crash Recovery

**Detailed specification for what survives a crash.**

| Data | Survives crash? | How |
|------|----------------|-----|
| File contents (saved files) | Yes | On disk + auto-save |
| File contents (unsaved named files) | Yes | Auto-save writes to disk continuously |
| File contents (untitled files) | Yes | Written to `recovery/` directory by auto-save |
| Open tab list | Yes | SQLite WAL journal survives crash |
| Cursor positions | Yes | Written to SQLite on every significant cursor move (debounced 2s) |
| Scroll positions | Yes | Written to SQLite with cursor positions |
| Undo history | **No** | Undo history is in-memory only; too expensive to persist continuously |
| Unsaved preferences | **No** | Preferences are saved immediately on change, so this is rarely an issue |

**Crash detection:** SQLite `dirty_flag` column set to 1 on startup, 0 on clean exit. If 1 on next startup → previous session crashed → show recovery prompt.

**Recovery prompt:** "MyNotepad++ didn't shut down cleanly. Restore previous session?" → [Restore] [Start Fresh]

**Safe mode:** If the app crashes 3 times in a row within 60 seconds, launch with all plugins disabled and a minimal theme. Show: "Started in safe mode due to repeated crashes."

---

## 5. USER INTERFACE LAYOUT

### 5.1 Main Window

```
┌──────────────────────────────────────────────────────────────────────┐
│  MYNOTEPAD++    File  Edit  Selection  Find  View  Go  Preferences  │  ← Menu Bar
├────────────────┬─────────────────────────────────────────────────────┤
│                │ [main.rs ×] [lib.rs ●] [config.toml ×] [+]        │  ← Tab Bar
│  EXPLORER      ├─────────────────────────────────────────────┬──────┤
│                │                                             │▓▓▓▓▓▓│
│  ▼ src/        │  1  fn main() {                             │░░░░░░│
│    main.rs     │  2      let config = Config::load();        │░░░░░░│  ← Minimap
│    lib.rs      │  3      let editor = Editor::new(config);   │░░░░░░│
│  ▼ tests/      │  4      editor.run();                       │░░░░░░│
│    test_ed.rs  │  5  }                                       │░░░░░░│
│    Cargo.toml  │  6                                          │░░░░░░│
│                │  7                                          │░░░░░░│
│                │                                             │░░░░░░│
├────────────────┴─────────────────────────────────────────────┴──────┤
│  Ln 1, Col 1  │  Spaces: 4  │  UTF-8  │  LF  │  Rust  │  Cmd+S ✓  │  ← Status Bar
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 Status Bar

The status bar shows contextual information and quick-access controls.

| Segment | Content | Click Action |
|---------|---------|-------------|
| Cursor position | `Ln 42, Col 15` | Opens Goto Line |
| Selection info | `3 selected` / `5 lines selected` | — |
| Indentation | `Spaces: 4` or `Tabs: 4` | Toggle/change indent |
| Encoding | `UTF-8` | Change encoding |
| Line ending | `LF` / `CRLF` | Change line ending |
| Language | `Rust` / `Python` | Change language mode |
| Auto-save status | `Saved ✓` / `Saving...` | Toggle auto-save |
| Git branch | `main` | — (future: git integration) |

---

## 6. PREFERENCES & SETTINGS

### 6.1 Settings Structure

```json
// ~/.mynotepadpp/settings/preferences.json
{
  "editor": {
    "fontSize": 14,
    "fontFamily": "SF Mono, Menlo, Monaco, monospace",
    "tabSize": 4,
    "insertSpaces": true,
    "wordWrap": "off",
    "lineNumbers": true,
    "minimap": true,
    "scrollBeyondLastLine": true,
    "smoothScrolling": true,
    "cursorBlinking": "smooth",
    "cursorStyle": "line",
    "bracketMatching": true,
    "indentGuides": true,
    "renderWhitespace": "selection",
    "trimTrailingWhitespace": true,
    "insertFinalNewline": true
  },
  "autoSave": {
    "enabled": true,
    "trigger": "both",
    "timerInterval": 30,
    "delayAfterTyping": 1,
    "hotExit": true
  },
  "theme": {
    "name": "MYNOTEPAD++ Dark",
    "variant": "dark"
  },
  "files": {
    "defaultEncoding": "utf-8",
    "defaultLineEnding": "lf",
    "watchForExternalChanges": true,
    "excludePatterns": ["node_modules", ".git", ".DS_Store", "*.pyc"]
  },
  "search": {
    "defaultRegex": false,
    "defaultCaseSensitive": false,
    "defaultWholeWord": false,
    "searchHistorySize": 50
  },
  "window": {
    "restoreSession": true,
    "newWindowForFolder": true,
    "titleBarStyle": "native"
  }
}
```

### 6.2 Preferences UI

Dual-mode settings:
1. **GUI**: Settings panel with organized categories and toggles
2. **JSON**: Direct edit of `preferences.json` (for power users)

---

## 7. PERFORMANCE TARGETS

| Metric | Target |
|--------|--------|
| Cold startup | < 500ms to first editor frame |
| Open 1MB file | < 200ms |
| Open 100MB file | < 2 seconds (progressive rendering) |
| Open 1GB file | < 10 seconds (streaming, not full load) |
| Keystroke latency | < 16ms (60 FPS) |
| Scroll FPS | 60 FPS minimum |
| Search (10K files) | < 3 seconds |
| Memory (idle, 1 file) | < 50MB |
| Memory (50 tabs) | < 300MB |
| Auto-save write | < 50ms (background thread) |

---

## 8. ACCESSIBILITY (All Platforms — P0)

Accessibility is a launch requirement, not a post-launch enhancement. Every platform must pass testing with its native assistive technology.

### 8.1 Screen Reader Support

| Platform | Screen Reader | Minimum Requirements |
|----------|--------------|---------------------|
| macOS | VoiceOver | All UI elements have accessibility labels; editor content navigable by line, word, character; cursor position announced on every move; selection range announced; syntax token type at cursor announced ("keyword", "string", etc.) |
| iOS | VoiceOver | Same as macOS; rotor actions for symbol/heading navigation; adjustable trait on zoom and font size controls |
| Windows | Narrator + NVDA + JAWS | UIA (UI Automation) providers for all controls; live regions for status bar changes; caret tracking for text navigation |
| Linux | Orca (via ATK / AT-SPI2) | All GTK4 widgets have `GtkAccessible` roles and labels; text widget exposes AT-SPI2 Text interface for line/word navigation |
| Android | TalkBack | All composables have `contentDescription` or `semantics {}`; `LiveRegion` for status bar; touch exploration support for editor content |

### 8.2 Keyboard-Only Operation

Every feature must be reachable without a mouse or pointer:

| Area | Requirement |
|------|-------------|
| Menu bar | Arrow key navigation, Enter to activate |
| Tab bar | `Ctrl+Tab` / `Cmd+1-9` to switch; no mouse-only actions |
| Sidebar | Arrow keys to navigate tree; Enter to open; Tab to move focus |
| Find bar | Tab between fields; Enter to search; Esc to dismiss |
| Command Palette | Arrow keys to navigate; Enter to execute; Esc to dismiss |
| Split panes | `Cmd+Option+Arrow` to move focus between panes |
| Diff view | `F7` / `Shift+F7` to jump between hunks; Enter to apply merge |
| Macro management | All actions reachable via Command Palette |
| Preferences | Full Tab/Arrow/Enter navigation |

### 8.3 Visual Accessibility

| Feature | Specification |
|---------|--------------|
| High contrast theme | At least one bundled theme with WCAG AAA contrast ratios (7:1 for normal text, 4.5:1 for large text) |
| Color-blind support | At least one syntax theme designed for deuteranopia/protanopia; avoid red/green as sole differentiator; git status in sidebar uses icons + color, not color alone |
| Diff view colors | Use background tint + gutter icons, not color alone, to indicate added/removed/modified |
| Font scaling | Editor font size is independent; UI chrome respects OS text-size preferences |
| Reduced motion | Respect OS setting (`prefers-reduced-motion` / `isReduceMotionEnabled`); disable smooth scrolling, cursor blink, minimap animations |
| Cursor visibility | High-visibility cursor option (thicker line, block cursor, or highlight current line) |
| Focus indicators | Visible focus ring on all interactive elements (not just browser-default — styled to match theme) |

### 8.4 Announcements & Live Regions

Status bar changes must be announced to screen readers without requiring focus:

| Event | Announcement |
|-------|-------------|
| File saved | "File saved" |
| Encoding changed | "Encoding changed to UTF-16 LE" |
| Language mode changed | "Language mode: Python" |
| Search: N matches found | "14 matches found" |
| Macro recording started/stopped | "Macro recording started" / "Macro recording stopped" |
| Remote connection established/lost | "Connected to server" / "Connection lost" |
| Error | Error message text |

---

## 9. FILE FORMAT SUPPORT

### 9.1 Natively Handled

| Category | Extensions |
|----------|-----------|
| Plain text | `.txt`, `.text`, `.log` |
| Source code | `.rs`, `.swift`, `.py`, `.js`, `.ts`, `.java`, `.c`, `.cpp`, `.h`, `.go`, `.rb`, `.php`, `.cs`, `.kt`, `.scala`, `.dart`, `.lua`, `.pl`, `.r`, `.m`, `.zig`, `.ex`, `.exs`, `.hs`, `.clj` |
| Web | `.html`, `.htm`, `.css`, `.scss`, `.less`, `.jsx`, `.tsx`, `.vue`, `.svelte` |
| Data/Config | `.json`, `.yaml`, `.yml`, `.toml`, `.xml`, `.csv`, `.tsv`, `.ini`, `.cfg`, `.conf`, `.env`, `.properties` |
| Shell | `.sh`, `.bash`, `.zsh`, `.fish`, `.ps1`, `.bat`, `.cmd` |
| Build | `Makefile`, `CMakeLists.txt`, `Cargo.toml`, `package.json`, `build.gradle`, `.sln`, `.csproj` |
| Docs | `.md`, `.rst`, `.tex`, `.adoc` |
| DevOps | `Dockerfile`, `.dockerignore`, `.gitignore`, `.gitattributes`, `Jenkinsfile`, `.github/workflows/*.yml` |

### 9.2 Binary Files

Binary files (images, executables, archives) show a message: **"This file is binary and cannot be displayed as text."** with option to open in system default app.

---

## 10. ICON ASSET LIST

All icons below will be custom-designed SVG originals. No paid or third-party icons.

### 10.1 App Icons

| Icon | Description | Used In |
|------|-------------|---------|
| `app-icon.svg` | Main application icon (document with ++ symbol) | Dock, Finder, About |
| `app-icon-dark.svg` | Dark variant for dark menu bars | Status menu |
| `document-generic.svg` | Generic file icon | Untitled tabs |

### 10.2 Toolbar Icons (24x24, monochrome line style)

| Icon | File | Action |
|------|------|--------|
| New file | `icon-new-file.svg` | New tab |
| Open file | `icon-open-file.svg` | Open file dialog |
| Open folder | `icon-open-folder.svg` | Open folder |
| Save | `icon-save.svg` | Save current file |
| Save all | `icon-save-all.svg` | Save all open files |
| Undo | `icon-undo.svg` | Undo last action |
| Redo | `icon-redo.svg` | Redo last action |
| Cut | `icon-cut.svg` | Cut selection |
| Copy | `icon-copy.svg` | Copy selection |
| Paste | `icon-paste.svg` | Paste from clipboard |
| Find | `icon-find.svg` | Open find bar |
| Replace | `icon-replace.svg` | Open find & replace |
| Split vertical | `icon-split-v.svg` | Split editor vertically |
| Split horizontal | `icon-split-h.svg` | Split editor horizontally |
| Sidebar toggle | `icon-sidebar.svg` | Toggle file explorer |
| Minimap toggle | `icon-minimap.svg` | Toggle minimap |
| Settings | `icon-settings.svg` | Open preferences |
| Command palette | `icon-command.svg` | Open command palette |
| Close | `icon-close.svg` | Close tab |
| Zoom in | `icon-zoom-in.svg` | Increase font |
| Zoom out | `icon-zoom-out.svg` | Decrease font |

### 10.3 File Type Icons (16x16, colored)

| Icon | File Types |
|------|-----------|
| `lang-rust.svg` | `.rs` |
| `lang-swift.svg` | `.swift` |
| `lang-python.svg` | `.py` |
| `lang-javascript.svg` | `.js`, `.jsx` |
| `lang-typescript.svg` | `.ts`, `.tsx` |
| `lang-html.svg` | `.html`, `.htm` |
| `lang-css.svg` | `.css`, `.scss`, `.less` |
| `lang-java.svg` | `.java` |
| `lang-kotlin.svg` | `.kt` |
| `lang-csharp.svg` | `.cs` |
| `lang-go.svg` | `.go` |
| `lang-cpp.svg` | `.c`, `.cpp`, `.h` |
| `lang-ruby.svg` | `.rb` |
| `lang-php.svg` | `.php` |
| `lang-shell.svg` | `.sh`, `.bash`, `.zsh` |
| `lang-json.svg` | `.json` |
| `lang-yaml.svg` | `.yaml`, `.yml` |
| `lang-xml.svg` | `.xml` |
| `lang-markdown.svg` | `.md` |
| `lang-sql.svg` | `.sql` |
| `lang-docker.svg` | `Dockerfile` |
| `lang-config.svg` | `.toml`, `.ini`, `.cfg`, `.env` |
| `lang-git.svg` | `.gitignore`, `.gitattributes` |
| `lang-generic.svg` | All other / unknown |

**Design rules for file icons:**
- Each icon uses a distinct shape + the primary color from the language's unofficial color identity (e.g., Rust = orange, Python = blue/yellow) but redrawn originally
- All icons are original vector art — NOT copied from VS Code, JetBrains, or any icon pack
- Licensed under GPL v3 as part of the project

---

## 11. ORIGINAL THEMES (Bundled)

### 11.1 Dark Themes

| Theme Name | Base | Accent | Description |
|------------|------|--------|-------------|
| MYNOTEPAD++ Dark | `#1A1A2E` | `#0D7377` | Default dark theme, teal accents |
| Midnight Ocean | `#0B1120` | `#4A9EFF` | Deep blue, ocean-inspired |
| Carbon | `#191919` | `#FF6B35` | Pure dark, warm orange highlights |
| Forest Night | `#1B2A1B` | `#7BC67E` | Dark green, nature-inspired |
| Slate | `#2D2D3A` | `#B8A9C9` | Muted purple, easy on eyes |

### 11.2 Light Themes

| Theme Name | Base | Accent | Description |
|------------|------|--------|-------------|
| MYNOTEPAD++ Light | `#FAFAFA` | `#0D7377` | Default light theme, teal accents |
| Paper | `#FFF8F0` | `#D4764E` | Warm off-white, like paper |
| Arctic | `#F0F4F8` | `#2E86AB` | Cool blue-white |
| Meadow | `#F5F8F0` | `#4A7C59` | Soft green-white |
| Sand | `#FAF3E8` | `#8B6914` | Warm beige, golden accents |

All themes are original creations, not ported from other editors.

---

## 12. RELEASE ROADMAP

### v1.0 — MVP (macOS ONLY — 100% complete + tested before moving to any other platform)

**Platform**: macOS (Swift + AppKit + Rust core) — Apple Silicon M4 primary target

**Scope**: All P0 features from Section 3.1:
- Rust core engine (rope buffer, tree-sitter syntax, encoding detection, undo/redo, search)
- Multi-tab editing, split views, scroll navigation
- Command Palette + Goto Anything
- Multi-cursor editing
- Find & Replace with regex + Find in Files
- Auto-save + hot exit + session restore
- 50+ language syntax highlighting
- 5 dark + 5 light original themes
- Original branding and icons
- Full VoiceOver accessibility (macOS)
- High-contrast + color-blind themes
- macOS native: menu bar, keyboard shortcuts (Cmd), Finder integration, Spotlight, dark mode
- Hardened Runtime + App Sandbox + notarization-ready

**Exit criteria**: All P0 features working, all `cargo test` + XCTest + XCUITest passing, VoiceOver tested manually, 60 FPS scrolling on M4, < 500ms cold startup. **No other platform work begins until v1.0 is shipped.**

### v1.1 — Power Features (macOS ONLY — same "ship then move on" rule)

- P1 features: minimap, distraction-free mode, snippets, file tree sidebar, project/workspace support
- **Macro recording & playback** (keystroke recording, save, assign shortcut) — Sections 4.15
- **Plugin system** (WASM sandbox, command registration, event hooks) — Section 4.16
- **SFTP/FTPS remote file editing** (secure protocols only, opt-in, no remote browser) — Section 4.17
- **Diff / file comparison view** (side-by-side, inline, word-level) — Section 4.18
- Performance optimization for very large files (1GB+)
- Additional themes + theme editor
- Keybinding preset profiles (Sublime, VS Code, Notepad++, Vim, Emacs)

**Exit criteria**: All v1.1 features working on macOS, tests passing, performance benchmarks met. Only THEN begin v2.0.

**Clarification on CLAUDE.md feature specs**: CLAUDE.md contains full architecture specs for Macros, Plugins, SFTP, and Diff because the Rust core is cross-platform by design — the core code written in v1.1 will be reused by all platforms in v2.0. But platform UI for these features is macOS-only in v1.1.

### v2.0 — Cross-Platform (one platform at a time, fully tested before next)

**Order**: Linux → Windows → iOS → Android (same as MVP priority order minus Mac)

Each platform release includes:
- All v1.0 + v1.1 features ported to the new platform
- Platform-native UI (GTK4, WinUI 3, SwiftUI, Jetpack Compose)
- Platform-native accessibility (Orca, Narrator, VoiceOver iOS, TalkBack)
- Platform-native SFTP libraries
- Git integration in sidebar
- Remote file browser (SFTP directory listing UI)
- iCloud / Google Drive sync for mobile

**Exit criteria per platform**: all features working, all platform tests passing, accessibility tested with native screen reader.

### v3.0 — Ecosystem

- Scriptable macros (Lua or JS runtime)
- Plugin UI injection (custom panels, sidebars)
- Collaborative editing (optional, server-based)
- Plugin registry / marketplace

---

## 13. LEGAL CHECKLIST

| Item | Status | Notes |
|------|--------|-------|
| Name "MYNOTEPAD++" is original | Verified | Does not infringe on "Notepad++" trademark |
| All code written from scratch | Required | No code copied from Notepad++, Sublime, VS Code, or any editor |
| All icons are original SVG art | Required | Created in Inkscape/Figma free tier |
| No paid stock images | Required | Zero watermarked or licensed assets |
| No copied UI designs | Required | Inspired by, not copied from |
| GPL v3 license | Chosen | All contributors must agree |
| Font usage | System fonts only | SF Mono (Apple), no bundled proprietary fonts |
| Tree-sitter grammars | MIT licensed | Compatible with GPL v3 |
| Rust crates | Audit for license compatibility | All must be GPL v3 / MIT / Apache 2.0 compatible |
| App Store guidelines | Review before submission | iOS/macOS App Store allow GPL v3 with caveats |

---

*This document is the source of truth for MYNOTEPAD++ features and behavior. All implementation must conform to these specifications.*
