# MYNOTEPAD++ — Functional Specification Document

**Version:** 1.2  
**Date:** 2026-05-03  
**Status:** Draft  
**License:** GPL v3  

---

## 1. PROJECT OVERVIEW

### 1.1 Vision

MYNOTEPAD++ is a free, open-source, native text and code editor built from scratch for macOS (primary), with planned expansion to Linux, Windows, iOS, and Android. It delivers the speed and simplicity of a lightweight editor with the modern editing power of a professional code editor — filling the long-standing gap of a fast, feature-rich, free native code editor on Mac.

### 1.2 What This Is NOT

- NOT a fork or copy of any existing editor
- NOT an IDE (no built-in compiler, debugger, or project management)
- NOT an Electron app — fully native on every platform
- NO paid assets, watermarks, or copied icons/branding

### 1.3 Target Users

- Developers who want a fast, lightweight editor on Mac
- Users migrating from Windows/Linux who need a powerful Mac-native editor
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

Features are categorized by priority. All features are original implementations designed from scratch.

| # | Feature | Priority | MVP |
|---|---------|----------|-----|
| 1 | Multi-tab editing | P0 | Yes |
| 2 | Syntax highlighting (50+ languages) | P0 | Yes |
| 3 | Vertical split view | P0 | Yes |
| 4 | Horizontal split view | P0 | Yes |
| 5 | Smart keyboard shortcuts (fully remappable) | P0 | Yes |
| 6 | Auto-save (never lose data) | P0 | Yes |
| 7 | Scroll to top / bottom / middle | P0 | Yes |
| 8 | Multi-cursor editing | P0 | Yes |
| 9 | Command Palette | P0 | Yes |
| 10 | Goto Anything (file/symbol/line) | P0 | Yes |
| 11 | Find & Replace (regex) | P0 | Yes |
| 12 | Minimap (code preview sidebar) | P1 | Yes |
| 13 | Dark/Light mode | P0 | Yes |
| 14 | Line numbers + code folding | P0 | Yes |
| 15 | Encoding support (UTF-8, UTF-16, etc.) | P0 | Yes |
| 16 | Distraction-free mode | P1 | Yes |
| 17 | Snippet system with tab triggers | P1 | Yes |
| 18 | Column/block editing | P1 | Yes |
| 19 | Find in files (multi-file search) | P0 | Yes |
| 20 | Indent guides + bracket matching | P0 | Yes |
| 21 | Word wrap toggle | P0 | Yes |
| 22 | Zoom in/out (font size) | P0 | Yes |
| 23 | Line operations (sort, deduplicate, join) | P1 | Yes |
| 24 | Auto-indent + smart indent | P0 | Yes |
| 25 | Drag-and-drop file opening | P0 | Yes |
| 26 | Session restore (reopen last files) | P0 | Yes |
| 27 | Project/workspace support | P1 | Yes |
| 28 | File tree sidebar | P1 | Yes |
| 29 | Code folding (tree-sitter based) | P0 | Yes |
| 30 | Auto-closing brackets & quotes | P0 | Yes |
| 31 | Auto-indent & smart indent (tree-sitter) | P0 | Yes |
| 32 | Tab size auto-detection | P0 | Yes |
| 33 | .editorconfig support | P0 | Yes |
| 34 | Go to Definition (tree-sitter + heuristic) | P0 | Yes |
| 35 | Open file at line from terminal | P0 | Yes |
| 36 | Bracket pair colorization | P0 | Yes |
| 37 | Git gutter (inline diff markers) | P1 | Yes |
| 38 | Sticky scroll (scope headers) | P1 | Yes |
| 39 | Word-based autocomplete (buffer + keywords) | P0 | Yes |
| 40 | Current line highlight | P0 | Yes |
| 41 | Whitespace visualization (spaces/tabs/newlines) | P0 | Yes |
| 42 | Wrap guides / rulers at configurable columns | P0 | Yes |
| 43 | File type auto-detection (shebang, modelines) | P0 | Yes |
| 44 | Revert file to saved | P0 | Yes |
| 45 | Expand/shrink selection (tree-sitter aware) | P0 | Yes |
| 46 | Select to brackets / matching pair | P0 | Yes |
| 47 | Transpose characters/words | P0 | Yes |
| 48 | URL detection + Cmd+click to open | P0 | Yes |
| 49 | Find in selection toggle | P0 | Yes |
| 50 | Read-only / lock mode | P0 | Yes |
| 51 | Binary file detection + warning | P0 | Yes |
| 52 | Long line handling (column virtualization) | P0 | Yes |
| 53 | Scroll annotations / overview ruler | P0 | Yes |
| 54 | Smart highlighting (auto-highlight selected word occurrences) | P0 | Yes |
| 55 | Case conversion (upper, lower, title, camelCase, snake_case) | P0 | Yes |
| 56 | Document statistics (word/char/line count in status bar) | P0 | Yes |
| 57 | Save Copy As (save to different path without changing active file) | P0 | Yes |
| 58 | Convert indentation (tabs-to-spaces, spaces-to-tabs) | P0 | Yes |
| 59 | NSTextInputClient (IME, emoji picker, dictation, CJK input) | P0 | Yes |
| 60 | Paste and Indent (auto-adjust indent on paste) | P0 | Yes |
| 61 | Open Recent submenu (recent files from previous sessions) | P0 | Yes |
| 62 | New Window (`Cmd+Shift+N`) | P0 | Yes |
| 63 | Rename file (from sidebar or Command Palette) | P0 | Yes |
| 64 | Trim trailing/leading whitespace (manual command) | P0 | Yes |
| 65 | Insert date/time | P0 | Yes |

**v1.1 — Power Features:**

| # | Feature | Priority | v1.1 |
|---|---------|----------|------|
| 66 | Macro recording & playback | P1 | Yes |
| 67 | Plugin/extension system (WASM sandbox) | P1 | Yes |
| 68 | SFTP/FTPS remote file editing | P1 | Yes |
| 69 | Diff / file comparison view (+ three-way merge) | P1 | Yes |
| 70 | Accessibility (VoiceOver, TalkBack, Narrator, Orca) | P0 | Yes |
| 71 | Integrated terminal panel (non-App-Store build only) | P1 | Yes |
| 72 | Line bookmarks (toggle, navigate, persist) | P1 | Yes |
| 73 | Clipboard history / ring | P1 | Yes |
| 74 | Outline view / symbol tree sidebar | P1 | Yes |
| 75 | Breadcrumbs bar (file path + symbol) | P1 | Yes |
| 76 | Markdown preview (side-by-side, live) | P1 | Yes |
| 77 | Git blame / inline annotate | P1 | Yes |
| 78 | Compare with any git commit | P1 | Yes |
| 79 | Hex editor mode (read-only) | P1 | Yes |
| 80 | Print support (syntax highlighted) | P1 | Yes |
| 81 | Character inspector / Unicode info | P1 | Yes |
| 82 | Spell checking (dictionary-based, squiggly underlines) | P1 | Yes |
| 83 | File monitoring / tail mode (live-watch log files) | P1 | Yes |
| 84 | Always-on-top window toggle | P1 | Yes |
| 85 | Plugin repository browser (search, install, update from editor) | P1 | Yes |

**Deferred to v2.0+:**

| Feature | Reason |
|---------|--------|
| Collaborative editing | Requires server infrastructure |
| Remote file browser (SFTP directory listing) | v1.1 supports open-by-path; browser is a UX enhancement |
| Scriptable macros (Lua/JS) | v1.1 has keystroke recording; scripting requires embedding a runtime |
| Plugin UI injection | v1.1 plugins register commands only; custom UI requires layout negotiation |
| LSP support (optional language servers) | Tree-sitter heuristic sufficient for v1; LSP adds hover docs, real go-to-def, rename, code actions |
| Emmet (HTML/CSS expansion) | Can be delivered as plugin |
| Color picker / inline color preview | CSS color swatches and native color picker |
| Auto-update mechanism (Sparkle on macOS) | Requires update server infrastructure |
| Integrated source control panel | Full stage/commit/push/pull UI beyond gutter markers |
| Task runner / build system integration | Run build/test commands with error parsing |
| Language-specific formatters | JSON pretty-print, XML prettify, SQL format (or plugin) |
| Editable hex editor | Read-only hex view in v1.1; edit mode in v2.0 |

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
| New tab | `Cmd+N` creates empty tab named "Untitled-N"; `Cmd+T` opens Goto Anything |
| Untitled naming | Name is "Untitled-N" where N is the **first available number** starting from 1. If Untitled-1 and Untitled-3 exist, next new tab is Untitled-2 (fills the gap). If 1,2,3 all exist, next is Untitled-4. Numbering persists across session restore. |
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
| Close pane | Close all tabs in focused pane: `Cmd+K, Cmd+W` (chord) |
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
| Scroll to middle of file | Via Command Palette "Go to Middle" | Cursor moves to line `totalLines / 2` |
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
| Select all occurrences | `Cmd+Ctrl+G` |
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
| Sort lines | Via Command Palette only |

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
| Goto symbol in project | `Cmd+T` |
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
| Full screen | `Fn+F` (native macOS globe key) |
| Zoom in | `Cmd+=` |
| Zoom out | `Cmd+-` |
| Reset zoom | `Cmd+Shift+0` |
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
      "command": "editor.toggleMinimap",
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

#### 4.4.8 Complete Menu Bar Structure

The menu bar follows Apple HIG: app menu, standard menus, Window (required), Help (required, rightmost).

**MYNOTEPAD++ (App Menu):**
- About MYNOTEPAD++
- Check for Updates...
- ---
- Settings... (`Cmd+,`)
- ---
- Services (macOS auto-injected)
- Hide MYNOTEPAD++ (`Cmd+H`)
- Hide Others (`Cmd+Option+H`)
- Show All
- ---
- Quit MYNOTEPAD++ (`Cmd+Q`)

**File:**
- New (`Cmd+N`)
- New Window (`Cmd+Shift+N`)
- ---
- Open... (`Cmd+O`)
- Open Folder... (`Cmd+Shift+O`)
- Open Recent > (recent files list + "Clear Menu")
- ---
- Reopen with Encoding > (UTF-8, UTF-8 BOM, UTF-16 LE, UTF-16 BE, Shift-JIS, ISO-8859-1, Windows-1252, ...)
- ---
- Save (`Cmd+S`)
- Save As... (`Cmd+Shift+S`)
- Save Copy As...
- Save All (`Cmd+Option+S`)
- Save with Encoding > (same list as Reopen)
- ---
- Revert to Saved
- Rename...
- ---
- Close Tab (`Cmd+W`)
- Close Window (`Cmd+Shift+W`)
- Close All Tabs (`Cmd+Option+W`)
- Reopen Closed Tab (`Cmd+Shift+T`)
- ---
- Print... (`Cmd+P` — only when print panel is available; otherwise reserved for Goto Anything)

**Edit:**
- Undo (`Cmd+Z`)
- Redo (`Cmd+Shift+Z`)
- ---
- Cut (`Cmd+X`)
- Copy (`Cmd+C`)
- Paste (`Cmd+V`)
- Paste and Indent (`Cmd+Shift+V`)
- Delete
- Select All (`Cmd+A`)
- ---
- Convert Case > UPPERCASE (`Cmd+K, Cmd+U`) | lowercase (`Cmd+K, Cmd+L`) | Title Case | camelCase | snake_case | PascalCase | kebab-case | CONSTANT_CASE
- ---
- Line Operations > Duplicate Line (`Cmd+Shift+Down`) | Delete Line (`Cmd+Shift+K`) | Move Up (`Option+Up`) | Move Down (`Option+Down`) | Join Lines (`Cmd+J`) | Sort Lines Ascending | Sort Lines Descending | Remove Duplicate Lines | Reverse Lines | Shuffle Lines
- ---
- Comment > Toggle Comment (`Cmd+/`) | Toggle Block Comment (`Cmd+Shift+/`)
- ---
- Indent > Indent (`Cmd+]`) | Outdent (`Cmd+[`) | Convert Indentation to Spaces | Convert Indentation to Tabs
- ---
- Blank Operations > Trim Trailing Whitespace | Trim Leading Whitespace | Trim Both
- ---
- Insert > Date/Time Short | Date/Time Long
- ---
- Transpose Characters (`Ctrl+T`)
- Transpose Words (`Ctrl+Option+T`)

**Selection:**
- Select Line (`Cmd+L`)
- Select Word / Next Occurrence (`Cmd+D`)
- Skip Occurrence (`Cmd+K, Cmd+D`)
- Select All Occurrences (`Cmd+Ctrl+G`)
- ---
- Expand Selection (`Ctrl+Shift+Space`)
- Shrink Selection (`Ctrl+Shift+Backspace`)
- Select to Brackets (`Ctrl+Shift+M`)
- ---
- Split Selection into Lines (`Cmd+Shift+L`)
- Add Cursor Above (`Cmd+Option+Up`)
- Add Cursor Below (`Cmd+Option+Down`)
- ---
- Single Cursor (`Esc`)

**Find:**
- Find... (`Cmd+F`)
- Find and Replace... (`Cmd+Option+F`)
- Find in Files... (`Cmd+Shift+F`)
- ---
- Find Next (`Cmd+G`)
- Find Previous (`Cmd+Shift+G`)
- Use Selection for Find (`Cmd+E`)
- ---
- Next Search Result (`F4`)
- Previous Search Result (`Shift+F4`)
- ---
- Go to Matching Bracket (`Ctrl+M`)

**View:**
- Sidebar (`Cmd+B`)
- Minimap (`Cmd+Shift+M`)
- Breadcrumbs (toggle)
- ---
- Syntax > (submenu: Plain Text, C, C++, Python, Rust, JavaScript, TypeScript, ... all supported languages alphabetically)
- ---
- Line Endings > Unix (LF) | Windows (CRLF) | Legacy Mac (CR)
- ---
- Indentation > Indent Using Spaces | Indent Using Tabs | Tab Width: 2 | 4 | 8 | Detect Indentation
- ---
- Code Folding > Fold (`Cmd+Option+[`) | Unfold (`Cmd+Option+]`) | Fold All (`Cmd+K, Cmd+0`) | Unfold All (`Cmd+K, Cmd+J`) | Fold Level 1-5
- ---
- Word Wrap (`Option+Z`)
- Word Wrap Column > Off | Viewport | 80 | 100 | 120 | Custom...
- ---
- Show Whitespace > None | Selection | Trailing | All
- Show Line Numbers (toggle)
- Show Indent Guides (toggle)
- Sticky Scroll (toggle)
- ---
- Layout > Single (`Cmd+Option+1`) | 2 Columns (`Cmd+\`) | 2 Rows (`Cmd+Shift+\`) | Grid (`Cmd+Option+\`) | 3 Columns
- ---
- Distraction-Free Mode (`Cmd+Ctrl+F`)
- Full Screen (`Fn+F`)
- Always on Top (toggle)
- ---
- Zoom In (`Cmd+=`) | Zoom Out (`Cmd+-`) | Reset Zoom (`Cmd+Shift+0`)

**Go:**
- Goto Anything... (`Cmd+P`)
- Command Palette... (`Cmd+Shift+P`)
- Goto Line... (`Ctrl+G`)
- Goto Symbol... (`Cmd+R`)
- Goto Symbol in Project... (`Cmd+T`)
- Go to Definition (`F12`)
- ---
- Go Back (`Ctrl+-`)
- Go Forward (`Ctrl+Shift+-`)
- ---
- Bookmarks > Toggle Bookmark (`Cmd+F2`) | Next Bookmark (`F2`) | Previous Bookmark (`Shift+F2`) | Clear All Bookmarks (`Cmd+Shift+F2`)
- ---
- Next Tab (`Ctrl+Tab`)
- Previous Tab (`Ctrl+Shift+Tab`)
- ---
- Scroll > Scroll to Top | Scroll to Bottom | Center Current Line (`Ctrl+L`)

**Macro:**
- Start/Stop Recording (`Cmd+Shift+R`)
- Play Last Macro (`Cmd+Shift+E`)
- Play Macro N Times...
- Save Macro...
- ---
- (List of saved macros with assigned shortcuts)

**Window:**
- Minimize (`Cmd+M`)
- Zoom
- ---
- Bring All to Front
- ---
- (List of open windows — macOS auto-managed)

**Help:**
- MYNOTEPAD++ Help (opens documentation)
- Keyboard Shortcuts Reference
- ---
- Report Issue... (opens GitHub issues)
- ---
- Release Notes (opens CHANGELOG)
- ---
- About MYNOTEPAD++ (also in app menu)

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
| Save unnamed files | To recovery directory | `<config-dir>/recovery/` (auto-managed) |
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
  Linux:   $XDG_DATA_HOME/mynotepadpp/ (~/.local/share/ fallback) for sessions/backups
           $XDG_CONFIG_HOME/mynotepadpp/ (~/.config/ fallback) for settings/macros/snippets
  Windows: %APPDATA%\mynotepadpp\
  iOS:     App container Documents/
  Android: App-internal storage

Directory structure (same on all platforms):
<config-dir>/
├── backups/                  ← Continuous backup (500ms debounce, survives SIGKILL)
│   └── {doc_id}/
│       └── {timestamp}.backup
├── recovery/                 ← Tier-3 auto-save fallback (disk full, permission denied)
│   ├── untitled-2026-05-03-143022.txt
│   └── untitled-2026-05-03-150145.rs
├── sessions/
│   └── sessions.db          ← SQLite (WAL mode): tab state, cursor, scroll, window layout
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

### 4.32 Word-Based Autocomplete

Basic completion without LSP. **P0 — every editor has this.**

| Feature | Specification |
|---------|--------------|
| Word completion | Scan current buffer for words; suggest as user types (3+ chars or `Ctrl+Space`) |
| Keyword completion | Language keywords from tree-sitter grammar (`fn`, `struct`, `impl` for Rust, `def`, `class` for Python) |
| Path completion | Complete file paths when typing inside string literals or import statements |
| Popup | Inline dropdown below cursor; arrow keys to navigate, Tab/Enter to accept, Esc to dismiss |
| Ranking | Exact prefix > fuzzy match > distance from cursor. Recently used words ranked higher. |
| Performance | Completion list computed on rayon pool, debounced 50ms. Never blocks typing. |
| Scope | Current file only for v1.0. Project-wide word index for v1.1. |

### 4.33 Current Line Highlight

Subtle background highlight on the active cursor line. **P0 — table stakes.**

| Feature | Specification |
|---------|--------------|
| Default | Enabled |
| Style | Subtle background color change (theme-defined). Not a border. |
| Multi-cursor | Each cursor's line is highlighted |
| Toggle | Preferences → Editor → Highlight Active Line |

### 4.34 Whitespace Visualization

Render invisible characters visually. **P0 — setting exists in preferences but needs rendering spec.**

| Character | Glyph | When Shown |
|-----------|-------|------------|
| Space | Center dot `·` | Based on `renderWhitespace` setting |
| Tab | Right arrow `→` spanning tab width | Based on setting |
| Newline | Pilcrow `¶` or return symbol `↵` | Only in `all` mode |
| Trailing whitespace | Red/pink background highlight | Always in `trailing` and `all` modes |

**`renderWhitespace` options:** `none`, `selection` (default), `trailing`, `boundary` (leading/trailing only), `all`

### 4.35 Wrap Guides / Rulers

Visual vertical lines at configurable column positions. **P0.**

| Feature | Specification |
|---------|--------------|
| Default | One ruler at column 80 (configurable) |
| Multiple rulers | Support array: `[80, 120]` draws two lines |
| Soft wrap at column | `wordWrap: "column"` wraps at the first ruler position |
| Wrap modes | `off` (no wrap), `viewport` (wrap at window edge), `column` (wrap at ruler) |
| Appearance | Thin vertical line, semi-transparent, theme-colored |

### 4.36 File Type Auto-Detection

Detect language from file content when extension is missing or ambiguous. **P0.**

| Method | Priority | Example |
|--------|----------|---------|
| File extension | 1 (highest) | `.rs` → Rust |
| Shebang line | 2 | `#!/usr/bin/env python3` → Python |
| Vim modeline | 3 | `# vim: set ft=python` → Python |
| Emacs modeline | 3 | `-*- mode: python -*-` → Python |
| Content heuristic | 4 | `<?xml` → XML, `<!DOCTYPE html>` → HTML |
| `.editorconfig` | 5 | Not for language, but for indent/encoding |

### 4.37 Revert File to Saved

Discard all unsaved changes and reload from disk. **P0.**

| Feature | Specification |
|---------|--------------|
| Access | `File > Revert File` or Command Palette "Revert File to Saved" |
| Confirmation | If buffer is modified: "Revert [filename]? All unsaved changes will be lost." [Revert] [Cancel] |
| Undo | Revert is itself an undo group — `Cmd+Z` restores the pre-revert state |
| Unmodified files | No-op (greyed out in menu) |

### 4.38 Expand / Shrink Selection (Tree-Sitter Aware)

Progressively expand selection along syntax tree nodes. **P0 — killer feature with tree-sitter.**

| Action | Shortcut | Behavior |
|--------|----------|----------|
| Expand | `Ctrl+Shift+Space` | word → token → expression → statement → block → function → class → file |
| Shrink | `Ctrl+Shift+Backspace` | Reverse of expand |
| Source | Tree-sitter syntax tree node hierarchy — accurate for all 50+ languages |

### 4.39 Select to Brackets / Matching Pair

Select content between matching brackets/quotes. **P0.**

| Action | Shortcut | Behavior |
|--------|----------|----------|
| Select inner | `Ctrl+Shift+M` | Select content between nearest enclosing `()`, `[]`, `{}`, or quotes |
| Press again | `Ctrl+Shift+M` | Expand to include the brackets/quotes themselves |
| Go to matching | `Ctrl+M` | Jump cursor to the matching bracket |

### 4.40 Transpose

Swap characters, words, or lines. **P0.**

| Action | Shortcut |
|--------|----------|
| Transpose characters | `Ctrl+T` — swap char before and after cursor |
| Transpose words | `Ctrl+Option+T` — swap word before and after cursor |
| Transpose lines | Already covered by Move Line Up/Down |

### 4.41 URL Detection & Clickable Links

Detect URLs in text and make them interactive. **P0.**

| Feature | Specification |
|---------|--------------|
| Detection | Regex for `http://`, `https://`, `ftp://`, `file://` URLs |
| Visual | Underline on hover (not always) |
| Action | `Cmd+Click` opens URL in default browser |
| File paths | Detect absolute file paths; `Cmd+Click` opens in editor |
| Toggle | Preferences → Editor → Clickable URLs (default: on) |

### 4.42 Find in Selection

Constrain find/replace to the current selection. **P0.**

| Feature | Specification |
|---------|--------------|
| Toggle | Button in find bar (icon: selection with magnifying glass) |
| Shortcut | `Cmd+L` while find bar is open |
| Behavior | When enabled, Find/Replace only operates within the selected region |
| Counter | "3 of 14 matches (in selection)" |

### 4.43 Read-Only / Lock Mode

Prevent accidental edits. **P0.**

| Feature | Specification |
|---------|--------------|
| Toggle | Command Palette → "Toggle Read-Only Mode" |
| Visual | Lock icon (🔒) on tab; status bar shows "READ-ONLY" |
| Behavior | All edit operations rejected (insert, delete, paste). Navigation works. |
| Auto-detect | Files without write permission open as read-only automatically |

### 4.44 Binary File Detection

Prevent crashes when opening non-text files. **P0.**

| Feature | Specification |
|---------|--------------|
| Detection | Scan first 8KB for null bytes (`0x00`) and magic byte signatures |
| Threshold | If > 0.1% null bytes in first 8KB → binary |
| Action | Show message: "This file appears to be binary." [Open as Text] [Open in Default App] [Cancel] |
| Magic bytes | Detect: PNG, JPEG, GIF, PDF, ZIP, ELF, Mach-O, PE/COFF |

### 4.45 Long Line Handling

Prevent hangs when opening files with extremely long lines (minified JS/CSS). **P0.**

| Feature | Specification |
|---------|--------------|
| Detection | Lines > 10,000 characters detected at load time |
| Warning | Status bar: "File contains very long lines. Some features may be slower." |
| Viewport culling | Only compute layout for visible columns ± overdraw (not entire line) |
| Syntax limit | Syntax highlighting limited to visible column range + 500 chars each side |
| Word wrap | Lazy wrap computation — only for visible viewport region |
| Soft limit | Lines > 500,000 chars: disable syntax highlighting for that line |

### 4.46 Scroll Annotations / Overview Ruler

Visual markers on the scrollbar track. **P0 (spec existed in 4.3 but not formalized).**

| Marker | Color | Source |
|--------|-------|--------|
| Search matches | Orange ticks | Active find operation |
| Modified lines | Yellow ticks | Unsaved changes vs. last save |
| Git changes | Green/red ticks | Added/deleted vs. git HEAD |
| Errors/warnings | Red/yellow ticks | (Future: diagnostics) |
| Current cursor | Blue indicator | Always visible |
| Bookmarks | Blue ticks | (v1.1: line bookmarks) |

### 4.47 Crash Recovery

**Detailed specification for what survives a crash.**

| Data | Survives crash? | How |
|------|----------------|-----|
| File contents (saved files) | Yes | On disk + auto-save |
| File contents (unsaved named files) | Yes | Auto-save writes to disk continuously |
| File contents (untitled files) | Yes | Written to `recovery/` directory by auto-save |
| Open tab list | Yes | SQLite WAL journal survives crash |
| Cursor positions | Yes | Batched in-memory, flushed to SQLite every 5 seconds |
| Scroll positions | Yes | Batched with cursor positions (flushed every 5 seconds) |
| Undo history | **No** | Undo history is in-memory only; too expensive to persist continuously |
| Unsaved preferences | **No** | Preferences are saved immediately on change, so this is rarely an issue |

**Crash detection:** SQLite `dirty_flag` column set to 1 on startup, 0 on clean exit. If 1 on next startup → previous session crashed → show recovery prompt.

**Recovery prompt:** "MyNotepad++ didn't shut down cleanly. Restore previous session?" → [Restore] [Start Fresh]

**Safe mode:** If the app crashes 3 times in a row within 60 seconds, launch with all plugins disabled and a minimal theme. Show: "Started in safe mode due to repeated crashes."

### 4.48 Save Behavior (Detailed)

**`Cmd+S` (Save):**

| Scenario | Behavior |
|----------|----------|
| Named file (already saved once) | Save silently to same path. No dialog. No confirmation. Modified indicator (●) clears. |
| Untitled file (never saved) | Opens `NSSavePanel` (Save As dialog). Default filename: the tab's current name (e.g., "Untitled-3"). Default extension: based on current syntax (`.py` for Python, `.rs` for Rust, no extension for Plain Text). Default location: last-used directory, or project root if in project, or `~/Documents/` as fallback. |
| Read-only file | Shows error: "Cannot save: file is read-only." Option to Save As to a different location. |

**`Cmd+Shift+S` (Save As):**
- Always opens `NSSavePanel` regardless of file state.
- Pre-fills current filename and extension.
- After saving: the tab SWITCHES to the new path. The old path is no longer associated with this tab. The old file retains its last-saved content on disk.

**`Cmd+Option+S` (Save All):**
- Saves ALL modified tabs silently.
- Skips read-only tabs (no error, just skip).
- Untitled tabs: auto-saved to recovery directory (NOT prompted for Save As — that would interrupt the user).
- Feedback: brief status bar message "All files saved" for 2 seconds.

**Save Copy As** (Feature #57):
- `File > Save Copy As...` — opens `NSSavePanel`.
- Saves to new path WITHOUT changing the active tab's association. The original file remains open and active.
- Use case: create a backup or copy without disrupting the current editing session.

**NSSavePanel File Type Filter:**
- macOS: no file type dropdown filter (native macOS convention — user types full filename with extension).
- If current syntax is set, the panel suggests the corresponding default extension when the user has not typed one.
- Extension enforcement: if user types a filename without extension, append the default extension from current syntax. If syntax is Plain Text, append `.txt` (safe default — user can delete it if they want no extension).
- If user explicitly types a different extension (e.g., types `data.csv` while syntax is Plain Text), respect the user's choice — do NOT override.

**Default extension per syntax (Save As dialog):**

| Syntax | Default Extension | Notes |
|--------|------------------|-------|
| Plain Text | `.txt` | Safe default for new files |
| C | `.c` | |
| C++ | `.cpp` | |
| Rust | `.rs` | |
| Python | `.py` | |
| JavaScript | `.js` | |
| TypeScript | `.ts` | |
| HTML | `.html` | |
| CSS | `.css` | |
| JSON | `.json` | |
| XML | `.xml` | |
| YAML | `.yaml` | |
| TOML | `.toml` | |
| SQL | `.sql` | |
| CSV | `.csv` | User must set syntax to CSV first |
| Markdown | `.md` | |
| Shell (Bash) | `.sh` | |
| Zsh | `.zsh` | |
| Fish | `.fish` | |
| PowerShell | `.ps1` | |
| Batch | `.bat` | |
| Ruby | `.rb` | |
| PHP | `.php` | |
| Go | `.go` | |
| Java | `.java` | |
| Kotlin | `.kt` | |
| Swift | `.swift` | |
| Scala | `.scala` | |
| Lua | `.lua` | |
| Perl | `.pl` | |
| R | `.r` | |
| LaTeX | `.tex` | |
| Dockerfile | `Dockerfile` | No extension |
| Makefile | `Makefile` | No extension |
| (all other syntaxes) | (first extension from that syntax's definition) | |

**macOS-specific shell scripts:** `.command` files (macOS double-clickable shell scripts) are recognized as Shell syntax when opened. To save as `.command`, the user types the full filename with `.command` extension in the Save dialog.

### 4.49 File Handling Edge Cases

| Scenario | Behavior |
|----------|----------|
| **Open file already open in another tab** | Switch to the existing tab. Do NOT open a duplicate. |
| **Open file already open in another window** | Open in the requesting window (files can be open in multiple windows simultaneously — they share the same buffer via the core). |
| **Two files with same name in different directories** | Tab displays: `filename.ext — parent_dir/`. E.g., `main.rs — src/` and `main.rs — tests/`. Show enough path to disambiguate. |
| **File deleted on disk while open** | Show non-blocking banner: "File has been deleted from disk." Tab title adds "(deleted)" suffix. The buffer remains editable. Save will recreate the file. |
| **File becomes read-only on disk while open** | On next save attempt: show error with option to Save As. If auto-save is running, auto-save skips this file and shows status bar warning. |
| **Symbolic links** | Follow symlinks — edit the target file, not the link. For duplicate-tab detection, resolve symlinks before comparing paths. |
| **File permissions on save** | Preserve original file permissions (mode bits) on atomic save. Copy permissions from original before rename. |
| **File ownership on save** | Atomic rename preserves ownership (same filesystem). Cross-filesystem save may change ownership — document this in user-facing error. |
| **Extended attributes (xattrs) on save** | Preserve extended attributes (Finder tags, quarantine flags) from original file. Copy xattrs before atomic rename. |
| **Maximum file size** | No hard limit. Files > 1GB: progressive loading with warning in status bar. Files > 10GB: show confirmation "This file is very large. Opening may use significant memory." |
| **Very large paste (>10MB clipboard)** | Paste on background thread with progress indicator. Never block UI. Entire paste = single undo group. |
| **Auto-reload preference** | Settings → Files → "Auto-reload externally changed files": `ask` (default, shows prompt), `always` (reload silently), `never` (ignore external changes). |

### 4.50 Smart Highlighting (Auto-Highlight Selected Word)

Automatically highlight all occurrences of the selected word/text in the document without opening Find. **P0.**

| Feature | Specification |
|---------|--------------|
| Trigger | Select a word (double-click or `Cmd+D`) or make a text selection |
| Behavior | All matching occurrences in the visible viewport highlighted with subtle background color (distinct from Find highlight) |
| Match mode | Case-sensitive, whole-word when a single word is selected. Exact substring when multi-word selection. |
| Performance | Computed on main thread for visible viewport only (debounced 100ms). Not a full-document search. |
| Scroll annotations | Ticks on scrollbar for all matches (same as search matches but different color) |
| Toggle | Settings → Editor → Smart Highlighting (default: on) |
| Relationship to Find | Independent of Find & Replace. Find highlights are orange; smart highlights are a softer color per theme. Both can coexist. |

### 4.51 Case Conversion Commands

Text transformation commands accessible via Command Palette and keyboard. **P0.**

| Command | Access |
|---------|--------|
| UPPERCASE | `Cmd+K, Cmd+U` (chord) |
| lowercase | `Cmd+K, Cmd+L` (chord) |
| Title Case | Command Palette → "Transform to Title Case" |
| camelCase | Command Palette → "Transform to camelCase" |
| snake_case | Command Palette → "Transform to snake_case" |
| PascalCase | Command Palette → "Transform to PascalCase" |
| kebab-case | Command Palette → "Transform to kebab-case" |
| CONSTANT_CASE | Command Palette → "Transform to CONSTANT_CASE" |

All commands operate on the current selection. With multi-cursor, applies to each selection independently. Single undo group.

### 4.52 Document Statistics

Display document metrics in the status bar. **P0.**

| Metric | Where | When |
|--------|-------|------|
| Line count | Status bar (always) | `Ln X of Y` format |
| Word count | Status bar (on selection) | "N words selected" or total word count via Command Palette |
| Character count | Status bar (on selection) | "N chars selected" |
| Selection count | Status bar (when multi-cursor) | "N selections" |
| Full statistics | Command Palette → "Document Statistics" | Dialog showing: total lines, total words, total characters, total characters (no spaces), file size on disk |

### 4.53 Convert Indentation

Bulk convert existing indentation in a document. **P0.**

| Command | Access | Behavior |
|---------|--------|----------|
| Convert Indentation to Spaces | Command Palette | Replace all leading tabs with spaces (using current tab size) |
| Convert Indentation to Tabs | Command Palette | Replace all leading spaces (in multiples of tab size) with tabs |
| Detect Indentation | Command Palette | Re-run tab size auto-detection (section 4.22) |

### 4.54 NSTextInputClient (Input Method Support)

The custom EditorView MUST support all macOS input methods. **P0 — without this, CJK input, emoji, and dictation are completely broken.**

| Feature | Requirement |
|---------|-------------|
| IME composition (CJK) | Implement `NSTextInputClient` protocol: `setMarkedText`, `markedRange`, `selectedRange`, `attributedSubstring`, `insertText`, `firstRect(forCharacterRange:)`, `characterIndex(for:)` |
| Emoji picker | `Cmd+Ctrl+Space` must work — opens system emoji picker, inserts selected emoji at cursor |
| Dictation | System dictation (Fn Fn or menu) must insert dictated text at cursor |
| Text replacement | System text replacement (Settings → Keyboard → Text) must expand abbreviations |
| Marked text rendering | IME composition text shown inline with underline, distinct from committed text |
| Candidate window positioning | `firstRect(forCharacterRange:)` must return accurate screen coordinates so IME candidate window appears next to cursor |

### 4.55 Editing Behavior Edge Cases

| Scenario | Behavior |
|----------|----------|
| **Cursor at wrapped line boundary** | Down/Up arrow moves to the next/previous **visual** line (within the same logical line). `Cmd+Down/Up` moves to next/previous logical line. This follows macOS text editing convention. |
| **Selection across folded regions** | Folded content IS included in the selection. Copy/cut includes the hidden lines. The fold indicator shows selection extends through it. |
| **Type while folded region is selected** | Replaces the entire folded region (including hidden content) with typed text. Same as typing over any selection. |
| **CJK word boundaries** | Use ICU word boundary detection for `Option+Left/Right` (word movement) and `Cmd+D` (word selection). ICU handles Chinese/Japanese segmentation correctly. CoreText provides this via `CTTypesetterSuggestLineBreak`. |

---

## 5. USER INTERFACE LAYOUT

### 5.1 Main Window

```
┌──────────────────────────────────────────────────────────────────────┐
│  MYNOTEPAD++  File  Edit  Selection  Find  View  Go  Macro  Window  Help │ ← Menu Bar
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

### 7.1 Core Metrics

| Metric | Target | Verification |
|--------|--------|-------------|
| Cold startup | < 500ms to first editor frame | `time open MyNotepadPP.app` |
| Warm startup (session restore) | Active tab visible < 200ms, remaining tabs progressive | Instruments Time Profiler |
| Open 1MB file | < 200ms to first render | Benchmark |
| Open 100MB file | < 2s (progressive: first screen < 200ms) | Benchmark |
| Open 1GB file | < 10s (streaming, first screen < 200ms) | Benchmark |
| Open file with 1MB single line (minified JS) | < 3s, no hang | Benchmark with minified React bundle |
| Keystroke latency | < 16ms (60 FPS) | Instruments |
| Scroll FPS | 60 FPS minimum on M4 | Instruments Core Animation |
| Search (10K files, literal) | < 2 seconds | Benchmark on real project |
| Search (10K files, regex) | < 5 seconds | Benchmark |
| Autocomplete popup | < 50ms after debounce | Instruments |
| Memory (idle, 1 file) | < 50MB | Activity Monitor |
| Memory (50 tabs, mixed files) | < 300MB (global budget enforced) | Activity Monitor |
| Auto-save write (< 100KB file) | < 50ms (background, never blocks UI) | Benchmark |
| Auto-save write (> 1MB file) | < 100ms (F_FULLFSYNC on macOS adds 10-50ms) | Benchmark |
| Hot exit (50 tabs) | < 500ms total | Benchmark |
| Tab switch | < 30ms (including session state read) | Instruments |
| Syntax highlight after edit | < 50ms for visible viewport | Instruments |
| File watcher response | < 500ms from disk change to UI update | Manual test |

### 7.2 Zero-Hang Guarantees

These are **hard requirements** — violations are P0 bugs:

| Scenario | Guarantee |
|----------|-----------|
| Cmd+Q (quit) | Terminate within 3 seconds. No dialog. No hang. |
| Cmd+W (close tab) | Close within 50ms. No dialog (auto-save enabled). |
| System shutdown / SIGTERM | Save + terminate within 5 seconds. |
| SIGKILL / Force Quit | Max 500ms of data loss (continuous backup covers rest). |
| Network filesystem stall | I/O operations timeout at 30 seconds. UI never freezes. |
| Opening 1GB file | UI responsive immediately. Loading in background. |
| Find in 100K files | Cancellable. UI responsive during search. |
| Disk full during save | Non-blocking notification. Buffer preserved in memory. |
| Auto-save on focus lost | Complete within 100ms or defer to background. |

### 7.3 Startup Waterfall

```
T+0ms     Show window frame (empty, from cached XIB)
T+50ms    Load last active tab metadata from SQLite
T+100ms   Begin loading last active tab's file content
T+200ms   First screen of active tab rendered (plain text)
T+300ms   Syntax highlighting applied to visible viewport
T+500ms   Remaining tab metadata loaded, tab bar populated
T+1000ms  Background: index remaining open files, build syntax trees
```

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
| Plain text | `.txt`, `.text`, `.log`, `.err` |
| C/C++ | `.c`, `.cc`, `.cpp`, `.cxx`, `.h`, `.hh`, `.hpp`, `.hxx`, `.ino` |
| Rust | `.rs` |
| Go | `.go` |
| Zig | `.zig` |
| Assembly | `.asm`, `.s`, `.S` |
| D | `.d` |
| Swift | `.swift` |
| Objective-C | `.m`, `.mm` |
| Java/JVM | `.java`, `.jsp`, `.groovy`, `.gradle` |
| Kotlin | `.kt`, `.kts` |
| Scala | `.scala`, `.sc` |
| C# | `.cs` |
| Python | `.py`, `.pyw`, `.pyx`, `.pxd`, `.pxi`, `.pyi` |
| Ruby | `.rb`, `.rbw`, `.rake`, `.gemspec`, `Rakefile`, `Gemfile` |
| PHP | `.php`, `.php3`, `.php4`, `.php5`, `.phtml` |
| Perl | `.pl`, `.pm`, `.plx`, `.t` |
| Lua | `.lua` |
| JavaScript | `.js`, `.mjs`, `.cjs`, `.jsx` |
| TypeScript | `.ts`, `.tsx` |
| CoffeeScript | `.coffee`, `.litcoffee` |
| Web markup | `.html`, `.htm`, `.shtml`, `.xhtml`, `.xht`, `.hta` |
| CSS | `.css`, `.scss`, `.less` |
| Frameworks | `.vue`, `.svelte` |
| JSON | `.json`, `.json5`, `.jsonc` |
| YAML | `.yaml`, `.yml` |
| TOML | `.toml` |
| XML | `.xml`, `.xsl`, `.xslt`, `.xsd`, `.svg`, `.kml`, `.plist`, `.xaml`, `.wsdl` |
| SQL | `.sql`, `.tsql` |
| Data | `.csv`, `.tsv` |
| Config | `.ini`, `.cfg`, `.conf`, `.env`, `.properties`, `.editorconfig`, `.gitconfig` |
| Shell | `.sh`, `.bash`, `.zsh`, `.fish`, `.command` |
| PowerShell | `.ps1`, `.psm1`, `.psd1` |
| Batch | `.bat`, `.cmd` |
| Build | `Makefile`, `.mak`, `.mk`, `CMakeLists.txt`, `Cargo.toml`, `package.json`, `build.gradle`, `.sln`, `.csproj`, `.vcxproj` |
| Docs | `.md`, `.markdown`, `.rst`, `.tex`, `.adoc`, `.textile` |
| DevOps | `Dockerfile`, `.dockerignore`, `.gitignore`, `.gitattributes`, `Jenkinsfile`, `.github/workflows/*.yml` |
| IaC / Cloud | `.tf`, `.tfvars`, `.proto`, `.graphql`, `.gql` |
| Haskell | `.hs`, `.lhs` |
| Elixir | `.ex`, `.exs` |
| Clojure | `.clj`, `.cljs`, `.cljc`, `.edn` |
| Erlang | `.erl`, `.hrl` |
| OCaml | `.ml`, `.mli` |
| Lisp/Scheme | `.lisp`, `.lsp`, `.el`, `.scm`, `.ss` |
| R | `.r`, `.R` |
| Julia | `.jl` |
| Nim | `.nim` |
| Dart | `.dart` |
| Pascal | `.pas`, `.pp`, `.dpr` |
| Fortran | `.f`, `.f90`, `.f95`, `.for` |
| TCL | `.tcl` |
| Ada | `.ada`, `.ads`, `.adb` |
| HDL | `.v`, `.sv`, `.vh`, `.svh`, `.vhd`, `.vhdl` |
| Visual Basic | `.vb`, `.vbs` |
| AppleScript | `.applescript`, `.scpt` |
| Diff/Patch | `.diff`, `.patch` |
| Regex | `.regexp` |
| Registry | `.reg` |

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
- All icons are original vector art — NOT copied from any existing editor or icon pack
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
- Find & Replace with regex + Find in Files + Find in Selection
- Auto-save + hot exit + session restore + continuous backup
- 50+ language syntax highlighting with embedded language support
- Word-based autocomplete (buffer + keywords + path completion)
- Current line highlight, whitespace visualization, wrap guides/rulers
- File type auto-detection (shebang, modelines, content heuristics)
- Expand/shrink selection, select to brackets, transpose, Go to Matching Bracket
- URL detection + clickable links, read-only mode, revert to saved
- Binary file detection, long line handling (column virtualization)
- Scroll annotations / overview ruler
- 5 dark + 5 light original themes
- Original branding and icons
- Full VoiceOver accessibility (macOS)
- High-contrast + color-blind themes
- macOS native: menu bar, keyboard shortcuts (Cmd), Finder integration, Spotlight, dark mode
- Hardened Runtime + App Sandbox + notarization-ready
- Zero-hang guarantees: no prompt on close/quit, no freeze on large files or network FS

**Exit criteria**: All P0 features working, all `cargo test` + XCTest + XCUITest passing, VoiceOver tested manually, 60 FPS scrolling on M4, < 500ms cold startup, zero-hang scenarios verified. **No other platform work begins until v1.0 is shipped.**

### v1.1 — Power Features (macOS ONLY — same "ship then move on" rule)

- P1 features: minimap, distraction-free mode, snippets, file tree sidebar, project/workspace support
- **Macro recording & playback** (keystroke recording, save, assign shortcut) — Section 4.15
- **Plugin system** (WASM sandbox, command registration, event hooks) — Section 4.16
- **SFTP/FTPS remote file editing** (secure protocols only, opt-in, no remote browser) — Section 4.17
- **Diff / file comparison view** (side-by-side, inline, word-level, three-way merge) — Section 4.18
- **Integrated terminal** panel (bottom, multiple instances, shell integration)
- **Line bookmarks** (toggle, navigate, persist in session)
- **Clipboard history / ring** (last 20 entries, in-memory)
- **Outline view / symbol tree** sidebar panel (tree-sitter powered)
- **Breadcrumbs bar** (file path + symbol hierarchy)
- **Markdown preview** (side-by-side, live, GFM support)
- **Git blame / annotate** (inline per-line, hover for commit details)
- **Compare with any git commit** (not just HEAD)
- **Hex editor mode** (read-only view of binary files)
- **Print support** (syntax highlighted, headers, line numbers)
- **Character inspector** (Unicode codepoint, UTF-8 bytes at cursor)
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

### 13.1 Intellectual Property

| Item | Status | Notes |
|------|--------|-------|
| Name "MYNOTEPAD++" is original | Verified | Does not infringe on "Notepad++" trademark |
| All code written from scratch | Required | No code copied from any existing editor — fully original |
| All icons are original SVG art | Required | Created in Inkscape/Figma free tier |
| No paid stock images | Required | Zero watermarked or licensed assets |
| No copied UI designs | Required | Inspired by, not copied from |
| Font usage | System fonts only | SF Mono (Apple), no bundled proprietary fonts |
| Trademark filing | Planned | File for "MYNOTEPAD++" name and logo |

### 13.2 Licensing

| Item | Status | Notes |
|------|--------|-------|
| GPL v3 license | Chosen | All source code, all platforms |
| GPL v3 Section 7 App Store exception | **Required** | Additional permission allowing distribution via Apple/Google/Microsoft stores despite DRM restrictions |
| CLA (Contributor License Agreement) | **Required** | All contributors assign copyright or grant relicensing rights — prevents VLC-scenario App Store removal |
| License text bundled in app | **Required** | In-app About/Help menu displays full GPL v3 text. LICENSE file in every app bundle/package. |
| Source code link in app | **Required** | About screen shows repo URL. App Store descriptions include repo link. |
| Git tag = shipped version | **Required** | Source at each git tag MUST correspond to the binary shipped to stores |
| Tree-sitter grammars | MIT licensed | Compatible with GPL v3 |
| Rust crates | Audit for compatibility | All must be GPL v3 / MIT / Apache 2.0 compatible. `cargo deny` in CI. |
| Plugin license enforcement | Required | Plugins must declare GPL-v3-compatible license in manifest; non-compatible rejected at install |

### 13.3 App Store Compliance

| Item | Platform | Status | Notes |
|------|----------|--------|-------|
| Apple Developer Program enrollment | macOS/iOS | Required | $99/yr for notarization + App Store |
| Google Play Developer enrollment | Android | Required | $25 one-time |
| Privacy manifest (`PrivacyInfo.xcprivacy`) | macOS/iOS | **Required since May 2024** | Declare Required Reason APIs (file timestamps, disk space, boot time, UserDefaults) |
| Privacy policy URL | All stores | **Required** | Even with zero data collection. Host at stable URL. |
| Age rating (IARC questionnaire) | All stores | Required | Text editor = 4+ (Apple) / Everyone (Google) |
| App Sandbox enabled | macOS | Required for Mac App Store | File access via Open/Save panels + security-scoped bookmarks |
| Hardened Runtime + notarization | macOS | Required | For both App Store and direct distribution (.dmg) |
| Code signing | macOS/iOS/Windows | Required | Development cert (local), distribution cert (release) |
| MSIX signing | Windows | Required for Microsoft Store | Code signing certificate |
| F-Droid reproducible build | Android | Required for F-Droid | Entire build (including Rust core) from source |
| No proprietary dependencies for F-Droid | Android | Required | No Google Play Services, no Firebase |
| Document browser integration | iOS | Required | `UIDocumentPickerViewController` for file access |
| `LSSupportsOpeningDocumentsInPlace` | iOS | Required | Edit files in-place from Files app |

### 13.4 Store Metadata Checklist (per platform)

| Asset | macOS App Store | iOS App Store | Google Play | Microsoft Store |
|-------|----------------|---------------|-------------|-----------------|
| App icon | 1024x1024 `.icns` | 1024x1024 PNG (no alpha) | 512x512 PNG | 300x300 PNG |
| Screenshots | 1280x800 or 1440x900 | 6.7" + 5.5" + iPad 12.9" | Phone (required) + tablet | 1366x768 minimum |
| Feature graphic | — | — | 1024x500 PNG | — |
| App preview (video) | Optional (15-30s) | Optional | Optional | Optional |
| Description | 4000 chars | 4000 chars | 4000 chars | 10000 chars |
| Keywords | 100 chars total | 100 chars total | — (indexed from description) | — |
| Category | Developer Tools | Developer Tools | Productivity | Developer Tools |
| Copyright notice | Required | Required | — | — |
| Support URL | Required | Required | Email required | Required |
| "What's New" text | Required for updates | Required for updates | 500 chars | Required |

### 13.5 Version Management for Stores

| Platform | Marketing Version | Build Number | CI Automation |
|----------|------------------|-------------|---------------|
| macOS/iOS | `CFBundleShortVersionString` = git tag (`1.1.0`) | `CFBundleVersion` = `${{ github.run_number }}` (must strictly increase) | Xcode build settings from CI env |
| Android | `versionName` = git tag (`"1.1.0"`) | `versionCode` = `MAJOR*10000 + MINOR*100 + PATCH` (must strictly increase) | `build.gradle.kts` computes from tag |
| Windows | MSIX `Version` = `1.1.0.0` (fourth part always 0) | — | `.csproj` reads from CI |
| Linux | Flatpak release tag = git tag | — | Flathub manifest auto-updates |

**Release process:**
1. Update `CHANGELOG.md` with version entry
2. Bump version in `core/Cargo.toml` and platform version files
3. Create git tag (e.g., `macos-v1.1.0`)
4. CI builds, signs, notarizes, and uploads to stores
5. Git tag source MUST match shipped binary (GPL v3 compliance)

---

*This document is the source of truth for MYNOTEPAD++ features and behavior. All implementation must conform to these specifications.*
