# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BentoPDF is a privacy-first, client-side PDF toolkit (50+ tools) built with Vite + TypeScript. All PDF processing happens in the browser (no server upload) using a mix of pure-JS libraries (pdf-lib, pdfjs-dist, pdfkit, jspdf) and WASM modules (PyMuPDF, Ghostscript, CoherentPDF/qpdf, LibreOffice converter, vips, Tesseract.js for OCR). It is dual-licensed (AGPL-3.0 / Commercial) and supports white-label builds with feature gating.

## Commands

```bash
npm run dev              # vite dev server
npm run build             # tsc + vite build + i18n page generation + sitemap + security headers + SEO audit
npm run build:docker      # vite build only (used by Dockerfile, skips the extra scripts)
npm run preview           # preview a production build
npm run serve             # build + preview on port 3000
npm run serve:simple      # SIMPLE_MODE=true build + preview (marketing-stripped build)

npm run lint               # eslint .
npm run lint:fix
npm run lint:security      # strict pass: no-unsanitized + security/detect-eval-with-expression, no inline disables
npm run format              # prettier --write .

npm test                   # vitest (watch mode)
npm run test:run            # vitest run (single pass, CI mode)
npm run test:coverage        # vitest run --coverage (80% threshold on lines/functions/branches/statements)
npx vitest run src/tests/<file>.test.ts   # run a single test file
npx vitest run -t "<test name>"            # run a single test by name

npm run docs:dev            # vitepress dev server for docs/ (VitePress site)
```

Pre-commit hook (husky) runs `lint-staged`: eslint --fix + prettier on staged JS/TS, prettier on staged JSON/MD/HTML/CSS.

## Architecture

### Tool pattern: page → handler → logic

Every tool follows the same three-layer flow:

1. **Page** — a standalone HTML file in `src/pages/` (e.g. `merge-pdf.html`), templated with `vite-plugin-handlebars` and statically generated per-locale at build time.
2. **Handler** — `src/js/handlers/fileHandler.ts` is the central dispatcher: it inspects the tool ID (derived from the page's URL/path) against the tool-type registries in `src/js/config/pdf-tools.ts` (`multiFileTools`, `simpleTools`, `singlePdfLoadTools`) to decide upload behavior, PDF loading, encryption prompts, and which logic module's process function to wire up.
3. **Logic** — `src/js/logic/*-page.ts` (100+ files), one module per tool, doing the actual transformation. Heavier operations (merge, compress) offload to Web Workers (`public/workers/*.worker.js`) or WASM bindings rather than blocking the main thread.

The master tool registry — display name, icon, category, href, auto-derived id — lives in `src/js/config/tools.ts`. Categories: Popular, Edit & Annotate, Convert to PDF, Convert from PDF, Organize, Optimize & Repair, Secure.

### WASM loading: CDN by default, overridable for offline use

`src/js/config/wasm-cdn-config.ts` and `src/js/utils/wasm-provider.ts` resolve WASM module URLs in two tiers:

- **Build-time default**: `.env` vars `VITE_WASM_PYMUPDF_URL`, `VITE_WASM_GS_URL`, `VITE_WASM_CPDF_URL` point at jsDelivr CDN packages by default.
- **Runtime override**: `WasmProviderManager` checks `localStorage` (`bentopdf:wasm-providers`) for user-configured URLs (validated against a trusted-host allowlist), falling back to the build-time env default. This is how air-gapped/self-hosted deployments repoint WASM fetches without rebuilding — see `STATIC-HOSTING.md` and the README's "WASM Configuration" section.

Per-module loaders (`*-loader.ts` under `src/js/utils/`) lazy-load each WASM bundle only when a tool that needs it is actually used.

### Feature gating / white-label builds

`src/js/utils/disabled-tools.ts` supports disabling tools two ways: a build-time `__DISABLED_TOOLS__` define in `vite.config.ts`, and a runtime `config.json` (served from `BASE_URL`) with a `disabledTools` array — letting a single commercial build serve a feature-limited instance without rebuilding. `SIMPLE_MODE` (env var, see `SIMPLE_MODE.md`) swaps in `simple-index.html` for a marketing-stripped landing page while keeping all tool pages.

### Workflow builder (Rete.js node editor)

`src/js/workflow/` implements a visual, node-based pipeline editor on top of `rete` + `rete-area-plugin`/`rete-connection-plugin`/`rete-engine`. Each node (`src/js/workflow/nodes/*.ts`) extends a `BaseWorkflowNode` and wraps the same standalone tool logic used by the page-based tools (e.g. the merge node calls into the same merge logic as `merge-pdf.html`), so adding a tool to the workflow editor means writing a thin node wrapper, not duplicating logic.

### Compare engine

`src/js/compare/engine/` diffs two PDFs (text runs, images, annotations) using a Web Worker (`compare.worker.ts`) for the CPU-heavy diffing, with OCR fallback via Tesseract.js for image-based PDFs. `src/js/compare/reporting/` generates a highlighted PDF report from the diff result.

### i18n

`scripts/generate-i18n-pages.mjs` statically generates a full set of localized HTML pages per supported locale at build time (19 languages, see `VITE_DEFAULT_LANGUAGE` in `.env.example` for the list) — this is a build step, not client-side routing. Runtime string lookups use i18next (`src/js/i18n/`), loading `common.json`/`tools.json` per locale via `i18next-http-backend`.

### Build (`vite.config.ts`)

This is a multi-page app: `rollupOptions.input` enumerates every page in `src/pages/` plus the generated per-locale variants. Notable behaviors controlled by env/flags:

- `SIMPLE_MODE` — marketing-stripped build (see above).
- `COMPRESSION_MODE` (`g`/`b`/`o`/`all`) — gzip/brotli/original/all asset compression via `vite-plugin-compression`.
- `VITE_USE_CDN` — used by `build:production` to control CDN vs bundled asset references.
- WASM URL env vars described above.
- `vite-plugin-static-copy` copies workers, WASM assets, and fonts into `dist/`.

### Security-relevant linting

`eslint.config.mjs` runs `eslint-plugin-security` (eval, timing attacks, unsafe regex) and `eslint-plugin-no-unsanitized` (flags `innerHTML`/`outerHTML` assignment patterns) as part of normal `npm run lint`; `npm run lint:security` runs a stricter pass with inline rule-disable comments forbidden, used in CI/security auditing alongside `npm audit` (`npm run security:audit`) and CodeQL (`npm run security:codeql`).

## Testing notes

Vitest runs in `jsdom` with `globals: true` (no per-file `describe`/`it` imports needed) and a 10s default timeout. Setup file: `src/tests/setup.ts`. Tests are colocated under `src/tests/` (not next to source files), mirroring the `src/js/logic/`, `src/js/compare/`, etc. structure with `*.test.ts` naming.
