# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies (requires Bun)
bun install

# Development server
bunx vite

# Production build
bun run build

# Run tests
bun test

# Run a single test file
bun test test/TraversionGraph.test.ts

# Generate format cache (run after build; uses puppeteer/Chromium — slow first run)
bun run cache:build

# Type-check only (no emit)
bunx tsc --noEmit

# Docker (prebuilt image)
docker compose -f docker/docker-compose.yml up -d

# Docker (local build)
docker compose -f docker/docker-compose.yml -f docker/docker-compose.override.yml up --build -d
```

## Architecture

The app is a **browser-based, privacy-first file converter** that runs all conversions client-side. It supports cross-medium conversion (e.g. AVI → PDF) by chaining multiple handlers together.

### Core abstractions (`src/`)

- **`FormatHandler.ts`** — Defines the key interfaces:
  - `FileFormat` — a format with MIME, extension, `from`/`to` booleans, `internal` ref, `category`, and `lossless` flag.
  - `FileData` — `{ name: string, bytes: Uint8Array }`. Handlers own the buffer's lifetime; clone with `new Uint8Array()` if mutation is possible.
  - `FormatHandler` — interface all handlers implement: `name`, `supportedFormats`, `ready`, `init()`, `doConvert()`. Optional `supportAnyInput` flag marks generic "catch-all" handlers (e.g. archive wrappers).
  - `FormatDefinition` / `builder()` — fluent API for defining formats in handlers (`.allowFrom()`, `.allowTo()`, `.markLossless()`, etc.).

- **`TraversionGraph.ts`** — Dijkstra's algorithm over the format graph. Nodes are unique formats (identified by `mime(format)`); edges are handler-provided conversions. Cost factors: depth, category-change penalties, lossy multiplier, handler/format priority index. `searchPath()` is an async generator yielding candidate paths. Handlers registered with `supportAnyInput` get edges from every known node to their output formats (with an `ANY_INPUT_COST` penalty to prefer specific paths).

- **`CommonFormats.ts`** — Pre-built `FormatDefinition` instances (PNG, JPEG, MP4, MP3, PDF, etc.) and the `Category` enum (`image`, `video`, `audio`, `text`, `document`, `archive`, `data`, `vector`, `font`, `code`, `spreadsheet`, `presentation`). Reuse these in handlers to avoid duplicate format definitions.

- **`PriorityQueue.ts`** — Min-heap used by `TraversionGraph`.

### Handlers (`src/handlers/`)

Each handler wraps one conversion tool. Naming convention: file `foo.ts`, class `fooHandler`, registered in `src/handlers/index.ts`. Handler order in `index.ts` matters — it affects the `handlerIndex` cost term in pathfinding (earlier = slightly preferred).

Key handlers: `FFmpeg.ts` (video/audio), `ImageMagick.ts` (images), `pandoc.ts` (documents), `font.ts`, `threejs.ts` (3D), `sevenZip.ts`, `sqlite.ts`, and ~60 others.

**WASM dependencies** must be declared in `vite.config.js` under `viteStaticCopy` targets, served from `/convert/wasm/`. Never link directly to `node_modules` for WASM. External git-repo dependencies are added as submodules under `src/handlers/`.

### Format cache

On first load, the app builds the supported-format map by calling `init()` on every handler. This is slow (~seconds). The result can be serialized to `cache.json` via `printSupportedFormatCache()` in the browser console, then rebuilt offline with `bun run cache:build`. Disable the cache when testing handler changes.

### Build targets

- **Web** — Vite SPA served under `/convert/` base path.
- **Electron** — `src/electron.cjs` is the main process; built with `IS_DESKTOP=true`.
- **Docker** — nginx serves the built output; Dockerfile installs Chromium for `buildCache.js`.

### Testing

- `test/TraversionGraph.test.ts` — graph traversal unit tests.
- `test/commonFormats.test.ts` — format definition tests.
- `test/handlers/<handlerName>.test.ts` — optional per-handler unit tests (add for handlers with non-trivial parsing/serialization logic).
- `test/MockedHandler.ts` — test utility for creating stub handlers.
- Test resources live in `test/resources/`.
