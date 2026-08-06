# AGENTS.md

## Overview
Obsidian plugin ("ToWord") that exports the active markdown note to a `.docx` file saved inside the vault. Desktop **and mobile** (iOS/Android) are supported, which constrains every API choice (see hard rules below).

## Commands
- `npm run dev` — esbuild watch build → `main.js`; reload Obsidian (Ctrl+R) to test
- `npm run build` — `tsc -noEmit -skipLibCheck` (typecheck) then production esbuild bundle
- `npm run version` — bumps version in `manifest.json` and `versions.json` (reads `npm_package_version`)
- Lint: `npx eslint .` — `.eslintrc` exists but there is **no npm lint script**
- No test framework or test suite. Verification is manual: build, export a note in Obsidian, open the `.docx` in Word/LibreOffice.

## Build/source gotchas
- `main.js` is gitignored build output — never commit it.
- esbuild treats `obsidian` (and codemirror/electron) as external; Obsidian API must be imported from the `obsidian` module.
- The real conversion source is `converter-mobile.ts` (NOT `converter.ts`). README, DEVELOPMENT.md, PROJECT_STRUCTURE.md, INSTALL.md and GITHUB_FILES.md all reference a stale `converter.ts`, an `examples/` dir, and a `docx` library that no longer exists. Trust the code over those docs.
- `node_modules` is not currently installed; run `npm install` before building.

## Architecture
- `main.ts` (~470 lines): plugin lifecycle, settings tab, ribbon/command/context-menu entries. Reads the note via `vault.read`, extracts Obsidian fonts from computed CSS, saves output via `vault.adapter.writeBinary`.
- `converter-mobile.ts` (~2700 lines): `MarkdownToDocxConverter`. Parses markdown with `markdown-it` (+ emoji/mark plugins), then builds the `.docx` **by hand**: OOXML XML strings for `word/document.xml`, `styles.xml`, `numbering.xml`, content types and relationships, zipped with `fflate`'s `zipSync`. There is no `docx` npm library.
- Export flow: `convert()` → strip frontmatter (if disabled) → convert math → optional preprocess → `processChunkedConversion` or `processNormalConversion` → `parseMarkdownToElements` → `generateDocx` → `getDocumentXml` → `zipSync` → `Blob`.

## Mobile-compatibility hard rules
- No Node/Buffer APIs. Use `fflate` (not JSZip), native `TextEncoder`, and Obsidian's `requestUrl` (never `fetch`).
- Use `activeWindow` / `activeDocument` and `.instanceOf(HTMLElement)` for popout-window compatibility — never bare `window` / `document` / `instanceof HTMLElement`.

## Conversion quirks (recurring bug areas)
- Bold/italic parsing uses internal placeholder markers (`|||ITALIC|||`, `|||BOLD|||`). A recurring bug is these markers leaking into the exported document (see CHANGELOG 1.4.4, 1.4.7) — new formatting logic must guarantee they are fully consumed or stripped.
- Math `$...$` / `$$...$$` is ALWAYS converted to Unicode plain text before anything else, regardless of the "Enable markdown preprocessing" toggle. Keep it that way so plain dollar amounts like `$20` don't get mangled.
- DOCX generation is synchronous. `convert()` is async, but do not `await` `generateDocx()` / `getDocumentXml()` — that caused lint warnings in 1.4.5/1.4.6.
- Notes larger than `chunkingThreshold` (default 100000 chars) are split on headings; per-chunk parse errors are logged and swallowed, processing continues.
- Images: embedded/local via `resourceLoader` (metadataCache `getFirstLinkpathDest` + `vault.readBinary`), remote via `requestUrl`. Route new image code through these paths, not new HTTP code.
- Font handling: sanitize font names against literal `??` / `undefined` (avoids "??" glyphs in Word); px→pt conversion is ×0.75.

## Settings tab
`ToWordSettingTab` renders via a private `renderSettings()` helper; don't reintroduce the deprecated `display()` self-call pattern.

## Release process (from git history)
- All work commits directly to `main`; no feature branches, no CI, no git tags.
- Release commits are named after the version (e.g. `1.4.7`). On release: bump version in `package.json`, run `npm run version` to sync `manifest.json` / `versions.json`, update `CHANGELOG.md`, then commit.
- `manifest.json`: `minAppVersion` is 1.4.0 and `isDesktopOnly` is `false` (mobile support is a selling point — never flip it).
