# CODE_STYLE.md (myNotesApp)

> **Version:** v1
> **Audience:** agents and humans contributing to myNotesApp.
> **Core principle:** As modular as possible. It is better to have more small, focused files than one large file.
>
> This guide is the myNotesApp-specific flavour of the generic `codeStyleGuide.md` template.
> It assumes **Electrobun.js + React + TypeScript**, **plain CSS**, **plain local state + controllers**,
> and **Zustand** for shared state. It does **not** assume TanStack Query, Jotai, or Tailwind.

---

## Table of contents

1. How to use this guide
2. Assumed stack
3. Core principles
4. TypeScript conventions
5. Components: definition & structure
6. The controller pattern
7. Hooks
8. Editor & the markdown layer
9. Data access & persistence (RPC)
10. State — Zustand
11. Styling — plain CSS
12. File & directory conventions
13. Error handling & edge cases
14. Testing & Storybook
15. Tooling / lint expectations
16. Appendices

---

## 1. How to use this guide

This guide is a **palette of defaults, not dogma**. Follow it unless a specific reason to deviate
exists. When you deviate from a `MUST` or `SHOULD` rule, **do it deliberately and leave a brief
note explaining why** (a code comment, or a one-line note in your response).

### 1.1 Precedence (tie-break order)

When two sources conflict, the higher one wins:

1. **SPEC.md** — the product requirements for this repo. It is the source of truth for what the
   app *does* and how its data is shaped.
2. **Explicit repo conventions** — anything else this repo documents (references, README).
3. **This guide** — the design defaults.
4. **Neighbouring code** — how the files around the one you are editing happen to be written today.

Neighbouring code is the *weakest* signal: it may predate the current conventions. Match it only
when no higher source applies. **SPEC.md beats this guide** when the two ever disagree.

### 1.2 Soft-adapt rule

This guide **assumes a stack** (section 2). Before writing code, check that the stack applies.

- **If the stack is present** → follow this guide.
- **If a part of the stack is absent or materially different** → **adapt** the affected rule to the
  equivalent in the target repo, and **flag the deviation**. Don't silently ignore the rule.
- **If a core assumption cannot be mapped at all** (see 1.4) → **pause and ask for guidance**
  rather than improvising an architecture.

Adaptation means: keep the *intent* of the rule, express it with the repo's actual tools.

### 1.3 RFC keywords (defined once, used everywhere)

- **MUST** / **MUST NOT** / **NEVER** — firm default. Overridable only by a higher-precedence source
  (1.1) or a clearly stated, deliberate reason.
- **SHOULD** / **SHOULD NOT** — strong recommendation. Deviate with a reason.
- **MAY** / **MAY NOT** — optional; context decides.
- **PREFER (aspirational)** — the better pattern; weaker choices acceptable when real constraints
  make it impractical. Every `PREFER` rule states its escape hatch.

### 1.4 What counts as "core" (hard to adapt)

If one of these is missing or fundamentally different, **ask for guidance**:

- The **editor stack** (CodeMirror + `codemirror-markdown-hybrid`). It is the heart of this app.
- The **data layer**: RPC via `BrowserView.defineRPC()` against a synchronous local filesystem.
- The **state layer**: Zustand + controllers.
- The **styling system**: plain CSS with semantic tokens.

Everything else (unit kit, formatter utils) is peripheral — adapt and flag.

### 1.5 Out of scope

This guide does **not** cover, and agents MUST NOT over-apply it to:

- Backend / server architecture (the Bun main process touches the filesystem, but that is not a
  server API surface).
- Git workflow, branching, commit and PR conventions.
- Internationalisation (i18n).
- Security, auth architecture, and secrets handling (incl. the Bun home env file — never commit it).
- CI/CD and release automation.

---

## 2. Assumed stack

This guide assumes the following. If any item is absent or different, apply the soft-adapt rule (1.2).

| Area | Assumed |
|------|---------|
| Framework | **Electrobun.js** (`Bun` runtime + `BrowserWindow` / `BrowserView`) |
| Components | **React function components** only (no class components) |
| Language | **TypeScript**, strict-ish |
| State | **Zustand** for cross-component client state |
| Logic | **Controllers** (`use<X>Controller`) + local `useState` |
| Data access | **RPC via `BrowserView.defineRPC()`** — synchronous, local filesystem, **no TanStack Query** |
| Storage | **`.md` files only** — never JSON |
| Editor | **CodeMirror** (`@codemirror/state`, `@codemirror/view`) + `codemirror-markdown-hybrid` |
| Styling | **Plain CSS** (semantic classes + CSS variables), *not* Tailwind or `cn()` |
| Imports | A **path alias** for app-internal imports (e.g. `@/*`) |

**Deliberately NOT used:** TanStack Query, Jotai, Tailwind, CSS module build tooling, `motion`,
Next.js route conventions, `use client` directives.

---

## 3. Core principles

These drive every specific rule. When this guide is silent on a case, decide using these.

1. **Separate logic from rendering.** A component renders; a controller/hook decides.
   Non-trivial logic goes to a controller (section 6). Trivial rendering stays inline.
2. **Store the right state in the right place.**
   App-wide shared state → Zustand (section 10). Cross-UI state that doesn't need to be shared →
   local `useState`/`useRef`. Server-owned data that must round-trip → hold it in one domain store or
   a controller. Everything else → local `useState`.
3. **Guard early, render the happy path.** Prefer early returns over nesting (section 13).
4. **Style with semantic classes/tokens, not ad-hoc values.** Meaningful class names and CSS
   variables — never raw colours baked into components (section 11).
5. **Make types describe reality.** Closed sets as unions with `type`; derive types from a single
   source of truth; keep `any` and `!` out of the happy path (section 4, 9).
6. **Co-locate by feature.** Keep a component, its logic, its types, and its test/story together,
   with a strict naming vocabulary (section 12).
7. **One obvious way.** Consistency with the surrounding repo beats personal preference. Reach for
   the pattern this guide prescribes before inventing a new one.

---

## 4. TypeScript conventions

### 4.1 `interface` for shapes, `type` for the rest — MUST

- **`interface`** for object shapes, props, and anything intended to be extended (interface `extends`).
- **`type`** for unions, intersections, mapped/utility types, tuples, and function signatures.

```ts
export interface Note {
  id: string;
  filename: string;
  title: string;
  date: number;
  snippet: string;
  folder: string;
}

export type NoteFolderDepth = number | null; // null = unlimited (recursive tree)

export type NoteChangeHandler = (note: Note, patch: Partial<Note>) => void;
```

### 4.2 Prefer string-literal unions over `enum` — PREFER (aspirational)

Unions are lighter, infer better, and need no runtime object. Use them for closed value sets.

- **Escape hatch:** use an `enum` (string-valued) when the codebase already standardizes on enums
  or when a runtime iteration over the values is genuinely needed. If you need a runtime map but
  not an enum, use a `const` object + `as const`.
- **NEVER** use numeric or auto-incremented enums.

```ts
// PREFER: union
export type NoteView = "hybrid" | "raw" | "preview";

// Acceptable: const object when a runtime map is needed
export const NoteTheme = {
  Light: "light",
  Dark: "dark",
} as const;
export type NoteTheme = (typeof NoteTheme)[keyof typeof NoteTheme];
```

### 4.3 Derive types from a single source of truth — MUST

NEVER hand-duplicate a set of keys or values.

```ts
// RPC returns notes in this shape; types are derived, not retyped by hand.
export type RpcNotesParams = { folderId?: string };

// Build a union of folder-structure fields from one place:
export const NOTE_FIELD_KEYS = ["id", "title", "folderId", "createdAt", "updatedAt"] as const;
export type NoteField = (typeof NOTE_FIELD_KEYS)[number];
```

### 4.4 `as const` and `satisfies` — SHOULD

Use `as const` for constant tuples/objects, and `satisfies` to validate a value against a type
without widening it.

```ts
export const NOTE_SIZES = [0.85, 8, 14, 32] as const satisfies readonly number[];

const defaultToolbar = {
  bold: true,
  italic: false,
} satisfies ToolbarConfig;
```

### 4.5 `unknown` over `any`; isolate unavoidable `any` — PREFER (aspirational)

At external boundaries (RPC boundary, DOM parsing, third-party libs) type as **`unknown`** and
narrow immediately.

- **Escape hatch:** `any` is acceptable only when a value is irreducibly dynamic, it is isolated to
  the smallest scope, and the reason is obvious from context. Never let `any` propagate through
  public types.

```ts
function parseNote(raw: unknown): Note {
  if (!isNote(raw)) throw new Error("Invalid note payload from RPC");
  return raw;
}
```

### 4.6 Avoid non-null assertions as the default — PREFER (aspirational)

Reach for narrowing/guards before `!` or `as`.

- **Escape hatch:** `!`/`as` is acceptable when the invariant is proven by construction (e.g. a
  value validated one line above). Keep it local, and prefer narrowing when it's just as cheap.

```ts
if (!note) return;
useNoteController(note); // note is narrowed

// Acceptable: invariant proven immediately above
const el = editorContainerRef.current!; // ref we just confirmed in a layout effect
```

### 4.7 Nullability — PREFER (aspirational)

Default convention: **`undefined` for absent optional data**, **`null` for React render guards**.
Optional fields use `?` (which yields `undefined`), not `field: T | null`.

- **Escape hatch:** if a specific module already standardizes on `null`, follow it consistently.
  What matters is being *consistent within a module*.

### 4.8 Exported functions — PREFER (aspirational): declare return types

Add explicit return types on **exported** functions where inference isn't obvious — it documents the
public shape and catches accidental drift.

- **Escape hatch:** omit for trivial one-liners and self-evident getters.

```ts
export const truncateSnippet = (text: string, max = 60): string => {
  if (text.length <= max) return text;
  return `${text.slice(0, max).trimEnd()} …`;
};
```

### 4.9 Exhaustive switches — PREFER (aspirational): `never` guards

When switching on a union, add a default that assigns to `never` so *new* members break the build.

- **Escape hatch:** skip when the switch is intentionally partial.

```ts
function noteViewIcon(view: NoteView): string {
  switch (view) {
    case "hybrid":
      return "edit";
    case "raw":
      return "code";
    case "preview":
      return "eye";
    default: {
      const _exhaustive: never = view;
      return _exhaustive;
    }
  }
}
```

### 4.10 Naming — MUST

- **Types / interfaces / components:** `PascalCase`.
- **Values / functions / variables:** `camelCase`.
- **Module-level constants:** `SCREAMING_SNAKE_CASE`.
- **Booleans:** prefix with `is` / `has` / `should` / `can`.
- **Hooks:** prefix with `use`.
- **Local event handlers:** `handle*`. **Handler props:** `on*`.
- **Store modules:** name the state (`selectedFolder`), not the action (`setSelectedFolder`).

### 4.11 `import type` — MUST

Use `import type { ... }` for type-only imports.

```ts
import type { Note } from "@/types/note";
import { useNoteController } from "@/features/note/note.controller";
```

**Anti-patterns:** numeric/auto enums; hand-repeating a key list; `any` in a public signature;
`field: T | null` where `field?: T` was meant; mixing `null` and `undefined` for the same concept.

---

## 5. Components: definition & structure

### 5.1 Functional components only — MUST

NEVER use class components or `PureComponent`. All components are functions.

### 5.2 Named exports — MUST

Components are **named exports**. `export default` is reserved for the SPA entry (`App.tsx`/
`main.tsx`) and Storybook `meta` only.

### 5.3 Component typing — MUST

- Components with props: define and export a `<Name>Props` **interface** and type the component as
  `React.FC<NameProps>`.
- Components without props: `React.FC` with no generic.
- Default prop values go in the destructuring signature.

```tsx
export interface NoteRowProps {
  note: Note;
  active?: boolean;
  readonly className?: string;
}

export const NoteRow: React.FC<NoteRowProps> = ({ note, active = false, className, ...props }) => {
  const label = useNoteRowController(note, active);
  return <ListItem className={cn("sidebar__row", { sidebar__row_active: active }, className)} {...props}>...</ListItem>;
};
```

### 5.4 The component stays presentational — MUST

The `.tsx` reads as declarative structure. It calls **one** controller hook at the top (section 6)
and destructures. No business logic, no derivations beyond trivial, no side effects.

### 5.5 Pass-through props and `className` — MUST

- To forward DOM attributes, extend the relevant `HTMLAttributes<T>` and spread `{ ...props }`.
- Always accept `className` and merge it **last** with `cn(...)` so callers can override.

### 5.6 JSX rules — MUST

- **Conditional rendering** uses `&&` for optional elements; wrap with `Boolean(...)` when the left
  side could render a value.

  ```tsx
  {folder.isExpanded && <FolderChildren folder={folder} />}
  {Boolean(note.snippetLength) && <PreviewSnippet text={note.snippet} />}
  ```

- **Lists:** always a stable `key` (a real id, never an array index).
- **Booleans:** use shorthand when `true`; not `={true}`.
- **Entities:** escape quotes/apostrophes in JSX text (`&apos;`, `&quot;`).
- **Prop order:** shorthand props → props → multiline props → `on*`/`set*`/`get*`/`render*`
  handlers last (section 15).
- **Accessibility:** interactive elements are real `<button>`/`<a>` (not clickable `<div>`),
  images have `alt`, external links use `rel="noopener noreferrer"`.

### 5.7 Loading state — SHOULD: a shared load-or-render primitive

Wrap async/loading content in a single shared **load-or-render** component rather than repeating
`isLoading ? <Something /> : children` at every call site.

```tsx
// Loading.tsx
export const Loading = ({ children, isLoading, className }: LoadingProps) =>
  isLoading ? <Divider className={cn("load__skeleton", className)} /> : <>{children}</>;
```

**Anti-patterns:** a component that fetches, transforms, formats, *and* renders; nested ternaries;
conditionals deeper than one level.

### 5.8 Animation — PREFER (aspirational)

Keep it minimal. A few short CSS transitions (0.15–0.3s) are fine for open/close of the folder tree
and toolbar. No heavyweight animation library is assumed — reach for a couple of CSS rules before
adding code.

---

## 6. The controller pattern

**Trigger:** a component needs logic beyond trivial prop reading and markup — derived values,
handlers, formatted data, side effects, branching on props/store.

**Rule:** put that logic in a **controller hook** in a sibling `<Name>.controller.ts(x)` file. The
component imports *only* the controller and renders.

### 6.1 Shape — MUST

- Name it `use<Name>Controller`.
- Return a **flat object** of derived state + handlers. NEVER a tuple.
- Booleans are named positively (`isLoading`, `hasUnsavedChanges`, `isCollapsed`).
- Handlers are named `handle*` or as plain verbs returned to the view.

```tsx
export const useNoteRowController = (note: Note, active: boolean) => {
  const activeClass = active ? "sidebar__row_active" : "";
  const preview = useMemo(() => truncateSnippet(note.snippet, 60), [note.snippet]);
  const openHandler = (e: MouseEvent) => { /* … */ };
  return { preview, activeClass, openHandler, label: titleFromNote(note) };
};
```

```tsx
export const NoteRow: React.FC<NoteRowProps> = (props) => {
  const { preview, activeClass, openHandler, label } = useNoteRowController(props);
  if (activeClass.includes("active")) return null; // guard clause — section 13
  return <ListItem onOpen={openHandler} className={cn("sidebar__row", activeClass)}>…</ListItem>;
};
```

### 6.2 Composition & props — SHOULD

- A controller MAY take the component's props and forward/destructure them.
- A controller MAY compose other controllers (a parent controller calling child controllers).
- If the controller returns JSX (e.g. a formatted node), use a `.controller.tsx` file.

### 6.3 When to skip — MAY

Trivial components (icons, layout wrappers, a single static element) MAY inline everything. The
controller exists to isolate *non-trivial* logic — do not create one for a two-line render.

**Anti-patterns:** a controller returning a tuple; business logic in the `.tsx` while the controller
only re-exports store values; a controller for a component that has no logic.

---

## 7. Hooks

### 7.1 Home for hooks — MUST

- **Feature-scoped** hooks live beside their feature (inside the component's co-located folder).
- **App-wide reusable** hooks live in the shared `hooks/` directory (flat, `useX.ts`).
- **RPC/data hooks** live beside the service layer (section 9).

### 7.2 Memoization — SHOULD (not reflexive)

- `useMemo` for genuinely expensive derivations, or to produce a stable reference passed to a child
  or effect.
- `useCallback` for handlers passed as props or used in effect deps.
- **Do not** wrap every value or trivial handler. Inline functions are fine when not passed down.

```ts
// Stable callback used as a dep
const commit = useCallback((next: string) => { saveNoteControl(commit), []); // …
```

### 7.3 Effects — MUST

Effects are for **synchronizing with external systems only** (timers, DOM measurement, the RPC
bridge wiring). They are NOT for deriving state from state.

- Always return a **cleanup** for timers and listeners.
- Keep dependency arrays honest. When intentionally omitting a dep, add a *described* disable
  comment (section 15).
- Prefer **functional updates** (`setX((previous) => …)`).
- Mirror a value you need inside an interval/effect with a `useRef` to avoid re-subscribing.

### 7.4 Headless updater components — MAY

An updater renders `null` (or one child) purely to sync an external system. Prefer rendering them
**as JSX**, not calling them as plain functions, so each keeps its own hook scope and render
boundary.

**Anti-patterns:** an effect that copies one piece of state into another; effects without cleanup
for intervals/subscriptions; blanket `useMemo`/`useCallback`; calling a component as a plain function.

---

## 8. Editor & the markdown layer

The editor is the heart of the app. Keep its glue in one place.

### 8.1 One composition per editor module — MUST

All CodeMirror extensions are assembled in **one** place (e.g. `editor/extensions.ts`), then a single
`createEditor()` factory hands components a ready-to-mount editor. Components do **not** reach into
extension config.

### 8.2 The hybrid plugin — MUST

- Configure `@codemirror/markdown` **and** `codemirror-markdown-hybrid` in the same factory.
- Render the hybrid instance via a single `<HybridEditor … />` wrapper. Unfocused lines show
  rendered markdown + KaTeX + mermaid; the focused line stays editable and raw. This is the documented
  hybrid behaviour in SPEC — do not reinvent it per usage site.

### 8.3 KaTeX & mermaid config — SHOULD

Enable both plugins' optional features (MathJax/KaTeX output, diagram syntax) **once**, in the
extension factory, so every editor in the app renders identically. Keep the config in `constants`.

### 8.4 Editing actions go through controllers — MUST

Per SPEC, editing is **toolbar-driven**; there are no editor keyboard shortcuts in the MVP. Formatting
from the toolbar (Bold, List, Heading, Table, …) comes from a controller that applies the change via
the editor view dispatch, not from keybinds.

```tsx
// EditorToolbar.controller.ts – one function per toolbar action
export const useEditorToolbarController = (view: EditorView) => {
  const setHeading = (level: 1 | 2 | 3 | 4 | 5 | 6) => {
    view.dispatch({ changes: { insert: newLine(), from: view.state.selection.main.from, to: view.state.selection.main.to } });
  };
  return { setHeading, setCodeBlock, toggleTaskList, ... };
};
```

### 8.5 Active editor state is local — MUST

The focused document content lives in the editor's own state, not in a component or store. On blur
and on a debounced autosave (SPEC: 500 ms) a controller calls the save RPC; until then the editor is
the source of truth for what's typed.

### 8.6 Theme toggling — SHOULD

The theme is driven by the hybrid plugin (`hybridMarkdown({ theme })`) **and** by the shell CSS
(section 11). Keep them in sync: the theme Zustand selector feeds both the plugin and the `<html>`
attribute (see section 11).

### 8.7 Markdown model — MUST

- Notes are stored as **`.md` files, never JSON** (SPEC).
- **One file per note** at `$userData/notebook/<folder>/<note>.md`; folders are recursive with no
  depth limit.
- Keep the FEATURE MATRIX from SPEC as the source of truth for what markdown is implemented (`#`
  headers, bold/italic/strikethrough, links, images, ` ``` ` code blocks, tables, task lists,
  blockquotes, ordered/unordered lists, `$$...$$` math, mermaid). Wikilinks `[[links]]` and backlinks
  are post-MVP (render as plain text for now).

> Note: the editor is hard to unit-test. Test controller logic with unit tests (section 14); verify
> rendering and interactions with **end-to-end** tests on the actual hybrid editor.

---

## 9. Data access & persistence (RPC)

### 9.1 The data is sync & local — the governing fact

`BrowserView.defineRPC()` bridges Bun filesystem calls from the webview. Reads and writes on the file
system are **synchronous and local**, in the normal case effectively instant — there is *no* network
client, no cache, and **no TanStack Query** to wrap them.

**Guiding rule from the old TanStack section (kept as intent):** don't duplicate state. Hold each piece
of server-owned data in **one** place and give consumers access to it. Here "one place" is either a
controller or a domain Zustand store.

### 9.2 Thin service wrappers — MUST

Wrap the RPC calls in framework-agnostic service functions under `src/services/*`. Controllers call
the service; components **never** call RPC directly.

```ts
// services/notes.service.ts (framework-agnostic — a plain function file)
export const getAllNotes = (params: NotesParams) => BrowserView.rpc<Note[]>("getAllNotes", params);
export const getFolders = () => BrowserView.rpc<Folder[]>("getFolders");
export const openNote = (filename: string) => BrowserView.rpc<string>("openNote", { filename });
export const saveNote = (filename: string, content: string) =>
  BrowserView.rpc<void>("saveNote", { filename, content });
export const deleteNote = (filename: string) => BrowserView.rpc<void>("deleteNote", { filename });
export const createNote = (title: string, folderId: string) =>
  BrowserView.rpc<Note>("createNote", { title, folderId });
```

The return shapes in the SPEC are the contract for these services:

| RPC | Signature | Returned shape |
|-----|-----------|----------------|
| `getAllNotes` | `() => object` | `{ id, filename, title, date, snippet, folder }[]` |
| `getFolders` | `() => object` | `{ path, depth, isRoot }[]` |
| `openNote` | `(filename) => string` | file content |
| `saveNote` | `(filename, content) => object` | void |
| `deleteNote` | `(filename) => object` | void |
| `createNote` | `(title, folder) => object` | `{ id, filename }` |
| `createFolder` | `(path) => object` | void |
| `deleteFolder` | `(path) => object` | void |
| `moveNote` | `(srcFilename, destFolder) => object` | void |

### 9.3 Hold notes/folders in one domain place — MUST

Keep the folder tree and the notes list in **one** store/controller so the sidebar and the editor
always subscribe to the same source. When a note is saved or deleted, mutate that one source and
every subscriber (sidebar row, editor) reflects it — no mirroring.

- If the tree is stable and the whole thing fits trivially, a single store module (section 10) is
  enough.
- If the app grows per-note ephemeral state (e.g. dirty flags), keep that **local** to the note
  controller and still keep the canonical notes/folders list authoritative.

### 9.4 RPC type safety — SHOULD

- Define `Note`, `Folder`, and the RPC param/return types **once** in `src/types/`, derived from the
  SPEC contract. Components import these — they never re-declare them.
- If you later add genuinely remote/async endpoints, introduce a thin query layer **behind** the
  service interface (section 9.2). Flipping from "call the service" to "wrap the service in a query"
  stays cheap because consumers already depend on `useX` hooks, not on `fetch`.

### 9.5 Service-layer anti-patterns

- Calling RPC/`fetch`/the API client from a component or a controller controller.
- Mirroring server-owned data into `useState` or a local variable that can drift from the canonical copy.
- Hand-typing `Note`/`Folder` shapes in multiple files (violates §4.3 / §9.4).

---

## 10. State — Zustand

### 10.1 When to use it — MUST

Cross-component **client** state lives in Zustand **only when it must actually be shared** between
components that don't have a clean parent/child or prop-passing relationship. If a value doesn't need
to be shared, keep it local (`useState`/`useRef`).

### 10.2 Domain-first stores — MUST

One store module per **domain**, not one giant store and not per-page stores. Group by domain
(`store/notebook/…`, `store/editor/…`, `store/ui/…`).

### 10.3 Shape — MUST

Each module exports the store **and** a `useX()` selector hook, so consumers never touch the raw
store API directly.

```ts
// store/selectedFolder.ts
import { create } from "zustand";

export interface SelectedFolderState {
  folderId: string | null;
  selectFolder: (folderId: string | null) => void;
}

export const useSelectedFolder = create<SelectedFolderState>((set) => ({
  folderId: null,
  selectFolder: (folderId) => set({ folderId }),
}));

// consumers use the selector, never the raw store
export const useSelectedFolderId = () => useSelectedFolder((s) => s.folderId);
```

### 10.4 Persisted UI preferences — MUST (via middleware)

Persisted preferences (theme, selected folder, open panes, note view mode) are a **distinct concern**
from transient UI state. Use the Zustand `persist` middleware so they survive restarts:

```ts
import { create, StateCreator } from "zustand";
import { persist } from "zustand/middleware";

export const useSettings = create<SettingsState>()(
  persist(
    (set) => ({
      theme: "light" satisfies NoteTheme,
      setTheme: (theme) => set({ theme }),
    }),
    { name: "mynotes.settings" }
  )
);
```

### 10.5 Anti-patterns

- Server/data-layer state living in the Zustand store (that belongs in controllers / section 9).
- Consumers importing raw `create`/`useStore`s everywhere instead of the `useX()` selectors.
- One giant store instead of domain modules.
- Persisting server-owned or derived data.

---

## 11. Styling — plain CSS

### 11.1 No utility classes — MUST

Keep the styling system simple: **plain CSS**. No Tailwind, no `cn()` class-merging helper, no
utility-scatter. Components carry **semantic** class names (`sidebar__row`, `toolbar__button`,
`note__title`, `load__skeleton`).

### 11.2 Semantic design tokens via CSS variables — SHOULD

Centralise colours, radii, and spacing in CSS custom properties on `:root` (and in editor-specific
blocks for the hybrid preview). Reference them with `var(--token)` rather than hard-coded colours.

```css
:root {
  --color-text: #1b1f23;
  --color-text-muted: #6a737d;
  --color-surface: #ffffff;
  --color-surface-hover: #f6f8fa;
  --color-border-divider: #eaecef;
  --color-accent: #2f81f7;
  --radius-note: 6px;

  --font-editor: "SFMono-Regular", "JetBrains Mono", ui-monospace, monospace;
  --font-ui: system-ui, -apple-system, "Segoe UI", sans-serif;

  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
}
```

Then in CSS (and, if you must, inline style):

```css
.sidebar {
  background: var(--color-surface);
  border-right: 1px solid var(--color-border-divider);
  color: var(--color-text);
  font-family: var(--font-ui);
}
```

### 11.3 Dark mode — MUST be theme-provider driven

Dark mode is driven by a theme provider (the Zustand theme selector, section 10) setting a flag on
the **element**, not by a JS theme check in components. Example:

```css
/* :root holds the light theme */
root[theme="dark"] {
  --color-text: #e6edf3;
  --color-surface: #161b22;
  /* … override each token … */
}
```

```tsx
// theme is applied to <html>/<body>; no per-component runtime theme check
document.documentElement.setAttribute("theme", theme);
```

### 11.4 Where styles live

- `src/styles.css` — global app shell + utility classes.
- `src/editor/editor.css` — hybrid preview, toolbars, KaTeX/mermaid layout.
- No per-component stylesheets.

### 11.5 Editor styling — SHOULD

Edit theming for the hybrid preview lives with the plugin config (section 8.3 + 8.6), and its layout
lives in `editor/editor.css`. Keep the two (JS plugin + CSS) in sync through the `:root[data-theme]`
flag.

### 11.6 Anti-patterns

- Utility/class strings built by template literals or string concatenation.
- Hex/rgb values sprinkled in JSX, or `dark:` classes with no build step.
- Baking a raw theme colour into a component instead of a token.

### 11.7 Numbers, still apply — SHOULD (cross-cutting; not finance-specific)

- Don't render raw `NaN` / `Infinity`. Guard them and render a placeholder — `""`, `"-"`.

```tsx
{Number.isNaN(snippetLength) ? "" : `${snippetLength} chars`}
```

- Format derived numbers through a small helper (`truncateSnippet` in sections 4/6) rather than
  re-implementing truncation inline. The finance-flavoured examples in the template
  (`formatCurrency`, `formatPercent`) **don't apply** here; keep only the "never render `NaN` raw"
  principle.

---

## 12. File & directory conventions

### 12.1 Co-locate by feature — MUST

A component, its logic, its types, its constants, and its test/story live together.

```
src/
├── components/            # presentational UI, one folder per feature
│   └── noteRow/
│       ├── noteRow.tsx            # the component (named export)
│       ├── noteRow.controller.ts  # logic hook (only if non-trivial)
│       ├── noteRow.constants.ts   # static labels / config (only if needed)
│       ├── noteRow.utils.ts       # pure helpers
│       └── noteRow.test.ts        # unit tests
├── hooks/                 # app-wide hooks (flat: useX.ts)
├── services/              # framework-agnostic RPC wrappers (sections 9.2)
├── store/                 # Zustand domain modules (section 10)
│   └── selectedFolder.ts
├── types/                 # shared interfaces & derived types (single source)
│   ├── note.ts
│   └── folder.ts
├── editor/                # editor glue & config (section 8)
└── feature-scope hooks     # feature-specific (live beside their feature)
```

### 12.2 Suffix vocabulary

Use the suffix for its meaning: `.controller.ts(x)`, `.constants.ts(x)`, `.config.tsx`, `.types.ts`,
`.utils.ts`, `.helpers.ts`, `.hooks.ts`, `.stories.tsx`, `.test.ts`.

### 12.3 Imports — MUST

- Use the alias `@/*` for app-internal imports; avoid deep `../../../` where an alias exists.
- `@/*` resolves under `/src/`.
- Order-enforced groups (section 15):
  1. External packages.
  2. Path-alias / internal packages (`@/*`).
  3. App-internal aliased imports (`@/*`).
  4. Relative.

### 12.4 Folder & component names — MUST

- Folders and component files: `PascalCase`.
- Non-component modules: match the file to its exported name (`useSelectedFolder.ts`, `truncateSnippet.ts`).

**Anti-patterns:** a grab-bag `utils.ts` shared across features; types duplicated in `.types.ts`
and inline in the component; `export default` outside `App.tsx`/`main.tsx`/Storybook `meta`.

---

## 13. Error handling & edge cases

### 13.1 Guard clauses over nesting — MUST

Keep the happy path un-indented.

**In views** — decide visibility up front, then render:

```tsx
if (isCollapsed) return null;
if (!note) return <EmptyState />;
return <NoteBody note={note} />;
```

**In controllers/utils** — return the empty/default shape early:

```ts
if (!note || !note.snippet) {
  return { label: "", snippetLength: 0, preview: "" };
}
```

**In callbacks** — guard per branch:

```ts
onRowClick: (note) => {
  if (!note) return;
  openNote(note);
},
```

### 13.2 Sentinel values — SHOULD

Use them consistently: `null` for a render guard, `undefined` for absent optional data, `"-"`/`""`
for display, `[]` for empty lists. (The finance-specific `NaN` sentinel in the template is dropped
here — the general "consistent empties" principle remains; `NaN` guard is covered in section 11.7.)

### 13.3 User-facing failures — SHOULD

- The Bun main process owns filesystem access. Failures (missing file, write errors) surface as
  RPC rejections that controllers catch — catch at the boundary that makes sense, and surface the
  failure via a shared toast/notification.
- Do **not** blanket `try/catch` everywhere; swallow nothing silently.

**Anti-patterns:** nested `if` pyramids in render; leaking raw system errors to the UI unadorned;
swallowing errors silently.

---

## 14. Testing & Storybook

### 14.1 Maturity gate — MUST

- **Prototype / spike phase** → tests and stories are **not required**. Focus on shipping the shape.
- **Beyond prototype (production-bound code)** → the conventions below become **required** for new
  and changed code.

If unsure which phase the project is in, **ask**. Don't assume "no tests" is the norm once the
project is production-bound.

### 14.2 Unit tests — MUST (once mature), except the editor

- Target **pure logic**: controllers, utils, helpers, formatters, reducers, derived types.
- Co-locate as `<Name>.test.ts` next to the thing under test.
- Use **Vitest** with `describe` / `it` / `expect`.
- Test edge cases explicitly: empty, zero, negative, `undefined`, boundary sizes, no folder selected,
  duplicate titles, recursion depth in the folder tree.
- The hybrid editor itself is not a unit-test target — drive it with **end-to-end** tests instead.

```ts
describe("truncateSnippet", () => {
  it("keeps short text intact", () => {
    expect(truncateSnippet("hi", 60)).toBe("hi");
  });
  it("truncates and marks the cut", () => {
    expect(truncateSnippet("a".repeat(100), 60)).toBe(`${"a".repeat(60).trimEnd()} …`);
  });
});
```

### 14.3 Stories — SHOULD (once mature)

- Add a `<Name>.stories.tsx` for reusable presentational components.
- Use `satisfies Meta<typeof Component>`; `title` follows `Features/<Name>` or `Components/<Name>`.
- Cover the default case plus meaningful variants and edge cases (empty, no folder, active, etc.).

**Anti-patterns:** testing implementation details instead of behaviour; snapshot tests as the
primary assertion; requiring stories/tests during an explicit prototype phase.

### 14.4 What to test (myNotesApp-specific)

- Folder-tree recursion, collapse/expand, count of notes per folder.
- All-notes view (unified, no folder selected) vs. a single-folder view.
- Search/filter across all notes.
- Autosave (500 ms debounce) and save-on-blur round-trip through the RPC save layer.
- Truncation (first 60 chars of snippet), empty states, duplicate titles, empty-folder deletion.

---

## 15. Tooling / lint expectations

These are enforced (or expected to be). Write code that satisfies them by construction.

- **Linting + formatting** — ESLint + Prettier. Code must pass `lint` and be formatted.
- **Import sorting** — automated (`simple-import-sort`). Write imports in the correct groups (section 12.3).
- **JSX prop ordering** — automated (`perfectionist`): shorthand → props → multiline → `on*`/`set*`/
  `get*`/`render*` handlers last. Don't hand-order against it.
- **`no-console`** — only `console.error` is allowed.
- **Described disables** — every eslint-disable MUST have a reason:
  ```ts
  // eslint-disable-next-line react-hooks/exhaustive-deps -- save-on-blur fires once
  ```
- **Unused vars** — prefix intentionally-unused bindings with `_` (`_exhaustive`, `_event`).
- **Type-checking** — the project MUST pass `type-check` (`tsc --noEmit`).
- **Circular dependencies** — break them by extracting shared code into a lower-level module rather
  than importing across the loop.
- **Component definition** — named function components; arrow for unnamed.

> Styling note: because styling is plain CSS (section 11), there is no utility-class or Tailwind
> lint rule. Prop and import ordering still apply.

---

## 16. Appendices

### Appendix A — Defaults quick reference

| Concern | Default |
|---------|---------|
| Component | Function, named export, `React.FC<Props>` when it has props |
| Logic | `use<X>Controller()` in `<X>.controller.ts(x)` |
| Client boundary | none — the whole app is client-side (Electrobun renderer) |
| Data access | RPC via `BrowserView.defineRPC()`; synchronous; **no TanStack Query** |
| Held state | one domain: controller or domain Zustand store |
| Shared client state | Zustand module + `useX()` selector |
| Local state | `useState` |
| Editor | CodeMirror + hybrid plugin, composed once (`editor/extensions.ts`) |
| Styling | Plain CSS, semantic classes + CSS variables |
| Types | `interface` for shapes, `type` for unions/utilities |
| "Absent" | `undefined` (data) / `null` (render guard) |
| Empty render | `return null` |
| Numbers | `format*`/helper utils; never render raw `NaN`/`Infinity` |
| Control flow | Guard clauses / early returns |
| Files | Co-located, `<Name>.<role>.<ext>` |
| Tests | Co-located; pure logic; once past prototype |

### Appendix B — Anti-patterns (consolidated)

- Class components / `PureComponent`.
- `export default` outside `App.tsx`/`main.tsx`/Storybook `meta`.
- Business logic in the `.tsx`; a component that fetches/transforms/formats/renders.
- Controllers returning tuples; controllers with no logic.
- `useEffect` to fetch (`useX` hook instead); effect copying state into state.
- Effects without cleanup; blanket `useMemo`/`useCallback`.
- Calling a component as a plain function.
- Server/RPC data in the Zustand store; duplicating canonical data into local state.
- Calling RPC/`fetch` from a component or controller.
- Utility-class scatter / Tailwind / `cn()` (this repo is plain CSS).
- Raw `dark:` classes with no build step; hex colours in components; JS theme checks.
- Nesting ternaries / `if` pyramids instead of guard clauses.
- Hand-duplicated unions; numeric enums; `any` in public types.
- `field: T | null` where `field?: T` was meant; mixing `null`/`undefined`.
- Grab-bag `utils.ts`; types duplicated across `.types.ts` and inline.
- Editing via manual keybinds (this app is toolbar-based — controllers own text actions).

### Appendix C — Readiness checklist

- [ ] Component is a function, named export, `React.FC<Props>` when it has props.
- [ ] Props interface exported; default values in the signature.
- [ ] Non-trivial logic extracted to `use<X>Controller()`.
- [ ] `React.FC` with no generic for propless components.
- [ ] Imports grouped/sorted; `@/*` alias for app-internal imports.
- [ ] Guard clauses instead of nesting; sentinels used consistently.
- [ ] Styling via plain CSS; semantic tokens via `var(-- …)`; no raw colours in components.
- [ ] Text via helper utils; no raw `NaN`/`Infinity` rendered.
- [ ] Server/RPC data accessed only through the service layer + a domain store/controller; no raw RPC in components.
- [ ] Shared state via a Zustand module + `useX()` selector.
- [ ] Effects synchronise external systems only, with cleanup.
- [ ] Editor content composed once via the extension factory; actions via controllers.
- [ ] Types: `interface` for shapes, unions for closed sets, derived from one source.
- [ ] Tests/stories added per maturity gate.
- [ ] Editor behaviour verified end-to-end (not unit-tested).
- [ ] Passes `lint` and `type-check`.

### Appendix D — Glossary

- **Controller** — a `use<X>Controller()` hook holding a component's logic; the view renders only.
- **Service layer** — framework-agnostic RPC wrappers (`services/*.service.ts`), never called by components.
- **Store** — Zustand cross-component client-state modules (`store/<domain>/`), exposed via `useX()`.
- **Hybrid editor** — CodeMirror + `codemirror-markdown-hybrid`: unfocused lines render markdown,
  the focused line stays raw.
- **Guard clause** — an early `return` that handles an edge case before the happy path.
- **Sentinel** — the conventional "empty" value for a type (`null`, `undefined`, `"-"`, `[]`).
- **Token** — a semantic design value (colour/surface/border) referenced via `var(--token)`, never a literal.
- **Soft-adapt** — adapting a rule to the repo's equivalent tool while keeping the rule's intent (1.2).

### Appendix E — Escape hatches (consolidated)

- `enum` when the repo already standardizes on it or a runtime map is genuinely needed (4.2).
- `any` only when irreducibly dynamic, isolated, and obvious (4.5).
- `!`/`as` when the invariant is provably true (4.6).
- Return types skipped for trivial one-liners (4.8).
- `useQuery`/query layer only if you later add remote/async endpoints — behind the service interface (9.4).

### Appendix F — Escalation triggers

Pause and ask for guidance when:

- The **editor stack** (CodeMirror + hybrid), RPC data layer, Zustand controllers, or CSS styling
  system is missing or fundamentally different (1.4).
- A rule's intent cannot be expressed with the repo's tools (1.2).
- SPEC.md, repo conventions, and this guide conflict and the resolution is unclear (1.1).
- You're unsure whether the project is prototype or production-bound (14.1).
