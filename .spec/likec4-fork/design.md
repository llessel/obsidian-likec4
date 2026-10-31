# Design — obsidian-likec4 (M2C fork)

## Starting point (upstream, verified)

```
src/
  main.ts                     # registers "likec4" code-block processor
  LikeC4CodeBlockProcessor.ts # parseOptions(firstLine) -> {view, height}; inline DSL
  LikeC4RenderChild.ts        # MarkdownRenderChild; owns a mounted view
  LikeC4ReactRenderer.tsx     # mounts @likec4/diagram
  cache.ts                    # modelCache (LRU of parsed models)
  settings.ts                 # diagramHeight, colorScheme, showBrowser, cacheSize
```

`parseOptions` already recognizes a first line like `view=index height=500` and
returns `{ options, remainingSource, hasOptions }`. Inline DSL = the remaining
source. We extend this, we don't rewrite it.

## Phase 1 — `src=` support

### Option parsing (`LikeC4CodeBlockProcessor.ts`)

Extend `CodeBlockOptions` with `src?: string`. In `parseOptions`, add `src` to
the recognized keys. Keep the same regex gate (first line is only options).

```ts
interface CodeBlockOptions { viewId?: string; height?: number; src?: string; }
// in the parse loop:
else if (key === "src" && value) options.src = value;
```

### Source resolution + render flow

In `process(source, el, ctx)`:
1. Parse options as today.
2. IF `options.src` is set:
   - Resolve the file via the vault: `this.plugin.app.metadataCache
     .getFirstLinkpathDest(src, ctx.sourcePath)` (respects links + relative
     paths) with a fallback to `app.vault.getAbstractFileByPath(src)`.
   - IF not a readable `TFile` → `renderError(el, "LikeC4 src not found: <src>")`.
   - Read with `await app.vault.cachedRead(file)` → `dslSource`.
   - Record the resolved `TFile` on the render child for reactivity (phase 1.3).
3. ELSE use the existing inline `remainingSource` / `source`.
4. Continue into `LikeC4RenderChild` with `dslSource` + `viewId` + `height`.

Because `process` is currently synchronous and reading is async, make the read
path async: render a lightweight placeholder, then populate once the file is
read (or make a small async helper and have the render child await it). Keep the
inline path synchronous to avoid a flash for existing notes.

### Cache keying (`cache.ts`)

Key models by a discriminator so inline and file models don't collide:
- inline: `inline:<hash(dsl)>`
- file:   `file:<path>:<mtime-or-hash>`

Invalidate the `file:` entry when the `TFile` mtime/content changes. Optionally
register `app.vault.on("modify", …)` (via `this.registerEvent` in `main.ts`) to
drop cache entries for a changed path and trigger a re-render of open previews.

### Render child reactivity (`LikeC4RenderChild.ts`)

Store the resolved source path on the child. On a vault "modify" event matching
that path, re-read and re-mount the view. Ensure the watcher is registered/freed
with the child lifecycle so unload stays clean.

## Phase 2 — layout-saving editing

> Specified for a later release; not required for the first BRAT build.

### Toggle & editor mount

- Add a setting `enableEditing: boolean` (default false) and/or a per-diagram
  affordance (a pencil button in the diagram chrome) — editing only offered when
  the diagram came from a `src=` file (a writable target).
- Swap the read-only `@likec4/diagram` view for the editable variant
  (`LikeC4EditorProvider` + `createLikeC4Editor`). Keep this in a new file
  (`LikeC4Editor.tsx`) so upstream merges stay clean.

### Persisting layout

LikeC4 persists manual layout **inline in the `.c4` source** as a structured
comment block next to the view (not a sidecar). The editor's `handleChange`
emits events: `save-view-snapshot`, `change-element-style`,
`change-autolayout`, `change-routing`. Map these to an updated `.c4` text:
- Use LikeC4's own serialization to produce the updated source (do NOT
  hand-roll the manual-layout comment format).
- Write back via `app.vault.modify(file, newText)`.
- Debounce writes (e.g. 500 ms trailing) to coalesce drags.
- On write failure: Notice + keep in-memory state.

### Concurrency

Guard against the "modify" watcher (phase 1) reacting to our own write (feedback
loop): set an "ignore next modify for this path" flag around our `vault.modify`.

## Settings additions

```ts
interface LikeC4PluginSettings {
  diagramHeight: number;
  colorScheme: "light" | "dark" | "auto";
  showBrowser: boolean;
  cacheSize: number;
  enableEditing: boolean;   // phase 2
}
```

## Alignment with the M2C exporter (must-hold invariant)

The exporter turns ` ```likec4 src=<path> view=<id> height=N ` into the Astro
web-component HTML. This plugin must preview the **same fence** so what the
author sees equals what gets published. Therefore:
- Option names are exactly `src`, `view`, `height`.
- `src=` points at the `.c4` file that the exporter will copy into the Astro
  repo's isolated `src/likec4/<key>/` root.
- Rendering here uses the LikeC4 engine directly (React); the Astro side uses the
  codegen'd web component — both read the same `.c4`, so visuals match.

## Testing

- Unit: option parser accepts `src`/`view`/`height` in any order; unknown option
  ignored; inline DSL unchanged.
- Manual (Reading view): a note with `src=Diagrams/x.c4 view=index` renders the
  view; editing a `src=` file updates the preview; a bad `src=` shows the error.
- Phase 2 manual: drag a node, confirm the `.c4` file gains/updates its layout
  comment and the VS Code extension reads it back identically.

## Non-goals

- Changing the rendered output format (stays the LikeC4 React view in Reading
  view).
- Editing inline-DSL diagrams (no writable target).
- Mobile support.
