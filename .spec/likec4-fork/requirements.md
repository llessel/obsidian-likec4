# Requirements — obsidian-likec4 (M2C fork)

## Introduction

This fork of `alisoliman/obsidian-likec4` renders LikeC4 diagrams inline in
Obsidian's Reading view. Upstream supports inline DSL in a ` ```likec4 ` fence
plus `view=` / `height=` options. This fork adds the ability to reference an
external `.c4` file via `src=` (so notes share one source-of-truth model instead
of duplicating DSL), keeping the fence option names aligned with the M2C
exporter. A later phase adds in-Obsidian **layout-saving editing**.

The driving use case: LikeC4 models live in plain `.c4` files (editable in VS
Code / `likec4 start`, consumable by the Astro web-component codegen). A note
previews a model the same way the Excalidraw plugin previews `.excalidraw.md`.

## Glossary

- **Model file**: a `.c4` file in the vault containing LikeC4 DSL (and possibly
  LikeC4's manual-layout comment block).
- **View**: a named view inside a model (`view index { … }`).
- **Fence options**: `key=value` tokens on the first line of the code block,
  e.g. `src=Diagrams/payments.c4 view=index height=500`.

## Requirements

### 1. `src=` external model reference (primary new feature)

**User story:** As an author, I want a note to render a view from a shared `.c4`
file, so I don't duplicate DSL across notes.

- 1.1 WHEN a ` ```likec4 ` fence's first line contains `src=<vault-path>` THEN
  the plugin SHALL read that `.c4` file from the vault and render it.
- 1.2 WHEN `src=` is combined with `view=<id>` THEN the plugin SHALL render the
  named view from the referenced file.
- 1.3 WHEN `src=` is given without `view=` THEN the plugin SHALL render the
  model's default/first view (same default behavior as inline DSL).
- 1.4 `src=` SHALL accept vault-relative paths; resolution SHALL use the
  Obsidian vault API (not Node `fs`) so it respects the vault root.
- 1.5 IF the `src=` file does not exist or is not readable THEN the plugin SHALL
  render a clear inline error (reusing the existing `likec4-error` element).
- 1.6 The existing `height=<px>` option SHALL continue to work with `src=`.
- 1.7 Inline DSL (a fence with no `src=`) SHALL continue to work unchanged
  (backward compatible).

### 2. Option parsing alignment

- 2.1 The fence option parser SHALL accept `src`, `view`, `height` in any order
  on the first line.
- 2.2 The option names SHALL match exactly what the M2C exporter consumes
  (`src`, `view`, `height`) so vault authoring and exported output stay in sync.
- 2.3 IF an unknown option is present THEN the plugin SHALL ignore it gracefully
  (no crash) and still render.

### 3. Reactivity & caching

- 3.1 WHEN the referenced `.c4` file changes on disk THEN an open preview SHOULD
  update (or at minimum update on note re-render) — stale cached models SHALL be
  invalidated when the source file's mtime/content changes.
- 3.2 The existing model cache SHALL key entries so that inline DSL and
  `src=`-referenced models do not collide.

### 4. Live-layout editing (phase 2 — specified, not required for first release)

**User story:** As an author, I want to drag nodes/edges in Obsidian and persist
the layout back into the `.c4` file, like the VS Code extension.

- 4.1 WHEN an edit-mode toggle is enabled THEN the plugin SHALL mount an editable
  LikeC4 view (`LikeC4EditorProvider` / `createLikeC4Editor`).
- 4.2 WHEN the user changes layout (move node, re-route edge, change autolayout)
  THEN the plugin SHALL persist the change back into the referenced `.c4` file
  in LikeC4's manual-layout comment format.
- 4.3 Editing SHALL only be available for `src=`-referenced files (a concrete
  writable target); inline DSL SHALL remain read-only.
- 4.4 Persisting SHALL write via the vault API and SHALL be debounced to avoid
  excessive writes.
- 4.5 IF a write fails THEN the plugin SHALL surface an error and SHALL NOT lose
  the in-memory layout.

### 5. Compatibility & upstream hygiene

- 5.1 The fork SHALL keep `manifest.json` `id` = `likec4` (do not rebrand; it is
  stable API for existing installs).
- 5.2 The fork SHALL remain mergeable with upstream where practical — new logic
  SHALL live in new files or clearly isolated functions rather than rewriting
  upstream files wholesale.
- 5.3 The plugin SHALL unload cleanly (no leaked render children, watchers, or
  listeners) — preserve the upstream `activeChildren`/`dispose()` pattern.

### 6. Non-functional

- 6.1 Local/offline only; no telemetry; no network requests.
- 6.2 Everything bundled into `main.js`; npm + esbuild toolchain unchanged.
