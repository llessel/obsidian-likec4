# obsidian-likec4 (M2C fork) — Obsidian community plugin

## What this is

A **fork** of [`alisoliman/obsidian-likec4`](https://github.com/alisoliman/obsidian-likec4)
(MIT) that renders [LikeC4](https://likec4.dev) architecture diagrams inline in
Obsidian's Reading view. Upstream bundles the full LikeC4 engine
(`@likec4/core`, `@likec4/diagram`, `@likec4/language-services`) and renders a
` ```likec4 ` fence via a `MarkdownCodeBlockProcessor`.

This fork exists to support the **M2C publishing pipeline** authoring workflow:
LikeC4 models live in plain `.c4` files (single source of truth, VS Code + Astro
codegen compatible), and notes reference a model + view rather than inlining the
DSL. The Obsidian plugin must be able to preview those referenced models the
same way the Excalidraw plugin previews its `.excalidraw.md` → SVG.

Remotes: `origin` = `llessel/obsidian-likec4` (push here); `upstream` =
`alisoliman/obsidian-likec4` (pull updates, do not push).

## The spec drives implementation

The planned changes are specified in `.spec/likec4-fork/`:
- `requirements.md` — the fork's added capabilities (EARS acceptance criteria).
- `design.md` — how to add `src=` file reading + the later layout-saving editor.
- `tasks.md` — ordered implementation checklist.

## Toolchain (real, verified)

- **Package manager: npm** (`package-lock.json` is committed). Do **not**
  introduce pnpm — see "Stray lockfiles" below.
- **Bundler: esbuild** via `esbuild.config.mjs` (`npm run dev` = watch,
  `npm run build` = production).
- **TypeScript** strict, `jsx: react-jsx` (renderer is React:
  `LikeC4ReactRenderer.tsx`), `moduleResolution: bundler`, target ES2022.
- `isDesktopOnly: true` (upstream choice; the LikeC4 engine is heavy).
- No ESLint/Prettier/Biome configured upstream — keep edits consistent with the
  existing 2-space style; do not add a linter unless the spec calls for it.

```bash
npm install
npm run dev     # esbuild watch -> main.js
npm run build   # production bundle
```

## Source map (small surface — keep it that way)

```
src/
  main.ts                     # lifecycle; registers the "likec4" code-block processor
  LikeC4CodeBlockProcessor.ts # parses fence options (view=, height=) + inline DSL
  LikeC4RenderChild.ts        # MarkdownRenderChild wrapper for a rendered diagram
  LikeC4ReactRenderer.tsx     # mounts the @likec4/diagram React view
  cache.ts                    # LRU cache of parsed models
  settings.ts                 # settings tab (diagramHeight, colorScheme, showBrowser, cacheSize)
```

Current fence options already parsed: `view=<id>`, `height=<px>`. Inline DSL
(no options) renders directly. **The fork adds `src=<vault-path>`** to read a
`.c4` file from the vault and render its named view.

## The M2C relationship (important)

The **Astro** side renders LikeC4 via a web-component bundle
(`<likec4-<slug>-view>` + `/likec4/<key>.js`), and the **exporter** plugin
(`obsidian-m2c-exporter`) turns a ` ```likec4 src=... view=... ` fence into that
HTML on export. This fork's job is only the **in-Obsidian preview** of the same
`src=`/`view=` fence, so authoring and published output stay in sync. Keep the
fence option names (`src`, `view`, `height`) identical to what the exporter
expects.

## Stray lockfiles

If `pnpm-lock.yaml` / `pnpm-workspace.yaml` appear, delete them — this repo is
npm-based and dual lockfiles cause drift.

## Versioning & releases (BRAT)

- Bump `version` in `manifest.json` (SemVer, no leading `v`) and add the
  matching entry to `versions.json` (`"<version>": "<minAppVersion>"`).
- Keep `manifest.json` `id` stable (`likec4`). Changing it breaks existing
  installs.
- Create a GitHub release tagged exactly `<version>`; attach `main.js`,
  `manifest.json`, `styles.css`. BRAT installs from `llessel/obsidian-likec4`.

## Obsidian plugin rules (summary)

- Browser environment, not Node — avoid Node/Electron APIs in rendering paths.
- Bundle everything into `main.js`; no unbundled runtime deps.
- Register listeners/children with `this.register*` / `ctx.addChild` so unload
  is clean (upstream already does this via `activeChildren` + `dispose()`).
- Follow Obsidian Developer Policies: local/offline by default, no telemetry,
  no remote code execution.

## References

- Upstream: https://github.com/alisoliman/obsidian-likec4
- LikeC4: https://likec4.dev - https://github.com/likec4/likec4
- Obsidian API: https://docs.obsidian.md
- M2C pipeline doc (in the Astro repo): `docs/obsidian-export-research.md`
