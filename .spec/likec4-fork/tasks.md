# Tasks — obsidian-likec4 (M2C fork)

Implement in order. Phase 1 (`src=`) is the first release. Phase 2 (editing) is
a later release. Reference requirement ids. Keep new logic in new files where
practical so upstream merges stay clean.

## 0. Groundwork

- [ ] 0.1 Delete stray `pnpm-lock.yaml` / `pnpm-workspace.yaml`; confirm
  `npm install` + `npm run build` produce `main.js`.
- [ ] 0.2 Confirm `upstream` remote points at `alisoliman/obsidian-likec4` and a
  `git fetch upstream` works (for later merges).

## Phase 1 — `src=` external model reference (first release)

- [ ] 1.1 (Req 2) Extend `CodeBlockOptions` with `src?: string` and recognize
  `src=` in `parseOptions`; ignore unknown options. Unit-test the parser.
- [ ] 1.2 (Req 1.1–1.5) In `LikeC4CodeBlockProcessor.process`, when `src=` is
  set, resolve the `.c4` via `metadataCache.getFirstLinkpathDest` /
  `vault.getAbstractFileByPath`, read via `vault.cachedRead`, and feed the DSL
  into the render child. Render the existing `likec4-error` on miss.
- [ ] 1.3 (Req 1.3, 1.6) Honor `view=` and `height=` together with `src=`;
  default view when `view=` absent.
- [ ] 1.4 (Req 1.7) Verify inline-DSL fences still render unchanged (regression).
- [ ] 1.5 (Req 3.2) Update `cache.ts` keying: `inline:<hash>` vs
  `file:<path>:<mtime|hash>` so the two never collide.
- [ ] 1.6 (Req 3.1) Register a vault `modify` watcher in `main.ts` (via
  `this.registerEvent`) that invalidates the `file:` cache entry and re-renders
  open previews for the changed path. Free it on unload.
- [ ] 1.7 Update README: document `src=` usage and that option names match the
  M2C exporter.
- [ ] 1.8 (Release) Bump `manifest.json` version + `versions.json`; `npm run
  build`; tag release; verify BRAT install from `llessel/obsidian-likec4`.

## Phase 2 — layout-saving editing (later release)

- [ ] 2.1 (Req 4.1) Add `enableEditing` setting + a per-diagram edit affordance,
  offered only for `src=`-backed diagrams.
- [ ] 2.2 (Req 4.1) New `src/LikeC4Editor.tsx` mounting the editable view
  (`LikeC4EditorProvider` / `createLikeC4Editor`).
- [ ] 2.3 (Req 4.2) Map editor `handleChange` events (`save-view-snapshot`,
  `change-element-style`, `change-autolayout`, `change-routing`) to updated
  `.c4` text using LikeC4's own serialization (do not hand-roll the layout
  comment).
- [ ] 2.4 (Req 4.4) Debounced `vault.modify` write-back with an
  ignore-own-modify guard to avoid a watcher feedback loop.
- [ ] 2.5 (Req 4.5) Error handling: Notice on write failure, keep in-memory
  layout.
- [ ] 2.6 Manual verification: drag a node in Obsidian, confirm the `.c4` gains a
  layout comment that the VS Code LikeC4 extension reads back identically.

## Verification checklist (definition of done, phase 1)

- `src=Diagrams/x.c4 view=index height=N` renders the named view in Reading view.
- Missing/invalid `src=` shows the inline error, no crash.
- Inline-DSL diagrams unchanged.
- Editing the referenced `.c4` updates the preview.
- Option names `src`/`view`/`height` match the M2C exporter exactly.
- `npm run build` clean; unloads without leaks.
