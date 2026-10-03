# SPEC: Obsidian-like Note-Taking App (Electrobun.js)

## Overview
Desktop note-taking app built on **Electrobun.js** using the `notes-app` template. Features an Obsidian-style hybrid editor where typed markdown renders inline as you move to the next line — powered by `codemirror-markdown-hybrid`. Multi-folder support is included in MVP.

---

## Tech Stack

| Layer | Choice |
|-------|--------|
| Desktop framework | Electrobun.js (Bun runtime, BrowserWindow + RPC) |
| Frontend framework | React.js + TypeScript |
| Editor engine | CodeMirror (`@codemirror/state`, `@codemirror/view`, `codemirror-markdown-hybrid`) |
| Base markdown lang | `@codemirror/lang-markdown` (GFM + commonmark) |
| Math rendering | KaTeX (bundled with hybrid plugin) |
| Diagrams | Mermaid.js (bundled with hybrid plugin) |
| Storage | `.md` files, `$userData/notebook/<folder>/<note>.md` |

---

## Architecture

```
Bun Main Process                          Webview Renderer
┌──────────────┐    RPC calls     ┌──────────────────┐
│  FileSystem    │◄──────────────►│  CodeMirror editor │
│   (fs module)  │                │  (hybrid plugin)   │
│               │                │                    │
│  Note Manager  │                │   Markdown preview │
│  (crud ops)    │                │ (unfocused lines)  │
└──────────────┘                 └──────────────────┘
        │
        ▼
$userData/notebook/<folder>
  ├── getting-started.md
  └── projects/
      └── q4-planning.md    ← persistent storage (tree structure)
```

**RPC Pattern:** `BrowserView.defineRPC()` bridges Bun filesystem calls from the webview UI process. Mirrors the Electrobun `notes-app` template exactly.

---

## Core Features (MVP)

### Editor (Webview)
- **Hybrid rendering:** unfocused lines show rendered markdown, focused line stays editable raw markdown
- Inline syntax highlighting in code blocks (Prism.js via hybrid plugin)
- Task lists (`- [ ]` / `- [x]`) with clickable toggles
- Collapsible headings (click chevron to collapse/expand sections)
- Math rendering via KaTeX (`$$...$$` blocks)
- Mermaid diagrams (flowcharts, sequence diagrams) inline in preview
- **Toolbar-only editing** — no keyboard shortcuts. Formatting via floating toolbar: Bold, Italic, Link, Code, Image, Task List, Heading 1/2/3/4/5/6, Quote, Code Block, Table, Bullet List, Numbered List, Horizontal Rule, Emoji

### Filesystem Layer
- Notes stored as `.md` files — **not JSON**
- One file per note in `$userData/notebook/<folder>/<note>.md`
- Save on blur + debounced autosave (500ms after last keystroke)
- Folder support: folders contain subfolders recursively; depth is unlimited

### Sidebar (Left Panel, ~280px)
- **Folder tree** on left edge (~140px): collapse/expand hierarchy
- **Notes panel** next to folder tree — shows .md files from selected folder (or "All Notes" virtual view)
- Each note row shows: **bold title** + date (small, gray) + 1-line preview snippet (first 60 chars)
- Active note highlighted with accent color
- Search/filter bar at top of notes panel (searches across all folders and their .md files by title+snippet)
- "+" button to create new note (appears in selected folder; if no folder selected, creates in root)

### Multi-Folder Support
- Create subfolders recursively (`folder/subfolder/deeper.md`)
- Drag-and-drop notes between folders
- Context menu on folders: "New Subfolder", "Rename Folder", "Delete Empty Folder"
- All-search mode shows notes from every folder in one unified list — search queries apply across all

---

## File Structure

```
app/
├── bun.ts                 # Electrobun main process — RPC handlers, file I/O, folder management
├── bun.html               # Electron window shell (minimal)
├── src/
│   ├── components/        # React component hierarchy
│   ├── hooks/             # Custom React hooks
│   ├── types/             # TypeScript interfaces and type definitions
│   ├── App.tsx            # Main application component
│   ├── main.tsx           # Entry point for React rendering
│   ├── styles.css         # Global styles
│   └── editor/            # Editor-specific logic and components
├── package.json           # NPM deps: react, typescript, codemirror, etc.
└── README.md
```

---

## RPC Interface (Bun ↔ Webview)

| Method | Direction | Signature | Purpose |
|--------|-----------|-----------|---------|
| `getAllNotes()` | Bun → View | `() => {id, filename, title, date, snippet, folder}[]` | Notes list (all or filtered by selected folder) |
| `getFolders()` | Bun → View | `() => {path, depth, isRoot}[]` | Build sidebar folder tree |
| `openNote(filename)` | View → Bun | `(f: string) => content: string` | Read .md file |
| `saveNote(filename, content)` | View → Bun | `(f: string, c: string) => void` | Write .md file (creates parent dirs if needed) |
| `deleteNote(filename)` | View → Bun | `(f: string) => void` | Remove note from notebook |
| `createNote(title, folder)` | View → Bun | `(t: string, f: string) => {id, filename}` | Create new .md in target folder |
| `createFolder(path)` | View → Bun | `(p: string) => void` | Make subfolder (creates ancestors if missing) |
| `deleteFolder(path)` | View → Bun | `(p: string) => void` | Delete empty subfolder |
| `moveNote(srcFilename, destFolder)` | View → Bun | `(src: string, dst: string) => void` | Move note file to new folder |

---

## UI Design

### Sidebar — Left Panel (~280px)
- **Folder tree** (left margin, ~140px): hierarchical, collapsible/expandable folders
  - Root = `$userData/notebook/`
  - Nodes have chevron icons (+/-), folder name text, count of notes in parenthesis `(3)`
  - Click to select folder into the "notes panel"
- **Notes panel** (right margin, ~140px): lists .md files per folder or "All Notes" view
  - Search input at top: filters across selected folder (or all notes if search mode)
  - Each row: `[icon] `**Title` + date` → subtitle snippet (first 60 chars trimmed to "...")
  - Click row → open in main editor
- **"+" button:** create new note. Folder context-aware — creates in selected folder or root.

### Main Editor (~1200px or expandable)
- Full-code-markdown hybrid mode by default
- Floating toolbar (top-right): Bold, Italic, Strikethrough, Link, Image, Headings 1–6, Quote, Code Block, Table, Bullet List, Numbered List, Task List, Horizontal Rule, Emoji
- Status bar (bottom): word count, line count, current note filename + folder path
- "Preview" mode toggle: switches between hybrid (default) and raw-only or preview-only

---

## Markdown Feature Matrix

| Feature | Implementation | MVP? |
|---------|---------------|------|
| Headers (# through ######) | Native in hybrid rendering | ✅ |
| Bold / Italic / Strikethrough | Syntax highlight + render | ✅ |
| Links (inline + autolink) | Clickable in preview, editable raw | ✅ |
| Images `![](src)` | Renders in preview, editable raw | ✅ |
| Code blocks (` ``` `) | Syntax highlighting via Prism.js | ✅ |
| Tables | Rendered with basic styling | ✅ |
| Task lists `- [ ]` / `- [x]` | Clickable toggle checkboxes | ✅ |
| Blockquotes `>` | Indented rendering | ✅ |
| Lists (ordered/unordered) | Numbered/bulleted + Enter continuation* | ✅† |
| Math `$$...$$` | KaTeX rendered inline | ✅ |
| Mermaid diagrams | Flowchart/sequence in hybrid view | ✅ |
| Obsidian links `[[wikilinks]]` | **Post-MVP #1** — plain text rendering for now | ❌ |

*Editor disables keyboard shortcuts by default. Enter continuation handled only via toolbar actions.
†Keyboard shortcut list is explicitly excluded from MVP (see Post-MVP backlog).

---

## Post-MVP Backlog (v1.0+)

| # | Feature | Details |
|---|---------|---------|
| 1 | **Wikilinks `[[links]]` + backlinks** | Parse `[[Note Title]]` syntax in markdown, resolve links → clickable to other notes; show "Backlinks" panel listing all notes that mention the current note. Link resolution handles missing files (show "not found" on hover). |
| 2 | **Tag system** | YAML frontmatter: `--- tags: design, code ---`. Search sidebar gains tag filter pane; per-note view shows clickable tag pills; click a tag to see all notes with that tag. Tag cloud or frequency histogram. |
| 3 | **Keyboard shortcuts** | Restore standard Markdown keybinds (Ctrl+B for bold, Ctrl+I for italic, Ctrl+K for link, Enter continuation in lists). Customize to Obsidian-like patterns (e.g., Cmd+Shift+M → insert math block). Per-note customizable bindings. |

---

## Notes
- **No keyboard shortcuts in MVP.** All editing is toolbar-driven as specified above.
- **No wikilinks (`[[links]]`) in MVP.** They render as plain text. Parsed and resolved post-MVP #1.
- **Folder structure is recursive.** No depth limit beyond what the filesystem permits.
- **Theme support:** Dark/light themes via `hybridMarkdown({ theme: 'dark' })`. Toggle available in toolbar or status bar.
- **All file operations are synchronous** (fs module is local). No async wait needed for typical note sizes.
