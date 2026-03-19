# Logging file logger – problem and solution

## Problem

When the TUI (or web server, when `MCP_LOG_FILE` is set) uses a pino file logger and the process exits early—for example, due to invalid arguments or missing required options—the process can crash with:

```text
Error: sonic boom is not ready yet
    at SonicBoom.flushSync (...)
```

**Cause:** The file logger is created with `pino.destination({ dest: logPath, append: true, mkdir: true })`. That destination opens the file **asynchronously**. Pino registers an exit handler that calls `flushSync()` on the destination when the process exits. If `process.exit()` runs before the file has finished opening, the destination is not ready and `flushSync()` throws.

Additional downsides of the previous design:

- **Logger created at module load** – The TUI logger was created at top-level in `logger.ts`, so file I/O happened at import time. That is brittle (failures are hard to handle) and makes it impossible to wait for the destination to be ready before doing work that might exit.
- **No clear separation of Node vs pure JS** – Logging code that depends on Node (pino’s default build, file destinations) lived at the top level of `core/logging`, so it wasn’t obvious that callers were pulling in Node-specific code.

## Proposed solution

### 1. Node logging only under `core/logging/node/`; no top-level logging export

- **`core/logging/node/`** – All Node-specific logging (pino default build, file destinations). Callers must import from `@modelcontextprotocol/inspector-core/logging/node` so that use of Node code is explicit.

- **`core/logging/node/logger.ts`** – Current `silentLogger` (move from `core/logging/logger.ts`).

- **`core/logging/node/fileLogger.ts`** – New helper that creates a pino file logger whose destination is ready before use:
  - Create the destination with `pino.destination({ dest: logPath, append: true, mkdir: true })`.
  - Wait for the destination’s `ready` event (the approach recommended by pino maintainers for this issue).
  - Create and return the logger (or a Promise that resolves to it). Both TUI and web (when using a file log) use this helper so behaviour is consistent and exit-time flush is safe.

- **`core/logging/node/index.ts`** – Exports `silentLogger` and the file-logger API from `fileLogger.ts`.

- **Top-level `core/logging/`** – After moving the existing code into `node/`, there is nothing left to export. Do **not** expose a top-level `./logging` entry in `core/package.json`; the directory exists only as the parent of `logging/node`. All logging API is under `./logging/node`. Existing imports of `silentLogger` from `@modelcontextprotocol/inspector-core/logging` must be updated to `@modelcontextprotocol/inspector-core/logging/node`.

### 2. TUI: async logger init, no module-init I/O

- **`clients/tui/src/logger.ts`** – No longer creates the logger at module load. It exposes:
  - **`initTuiLogger(): Promise<void>`** – Creates the file destination (using the shared helper from `core/logging/node/fileLogger.ts`), awaits `ready`, assigns the resulting logger to a module-level variable.
  - **`tuiLogger`** – A module-level export (e.g. `let tuiLogger`) initially set to a silent no-op; after `initTuiLogger()` runs it is replaced with the real file logger. So existing code that imports `tuiLogger` and uses it later (after `runTui` has started) continues to work unchanged.
- **`clients/tui/tui.tsx`** – At the very start of `runTui()`, before any parsing or validation that might call `program.error()` / `process.exit()`, call **`await initTuiLogger()`**. Then the destination is ready for the lifetime of the process and exit handlers can flush cleanly.

### 3. Web (when `MCP_LOG_FILE` is set)

- Use the same **`core/logging/node/fileLogger.ts`** helper to create the file logger (e.g. in `buildWebServerConfigFromEnv` or its caller). That requires making config building async where a file logger is created, and awaiting the helper so the destination is ready before the server runs or exits.

### 4. Package exports

- In `core/package.json`, expose **only** the Node logging path: `"./logging/node": "./build/logging/node/index.js"` and `"./logging/node/*": "./build/logging/node/*"`. Remove the top-level `"./logging"` and `"./logging/*"` exports; there is no public API at `core/logging` anymore.

## Summary

- **Problem:** Early process exit triggers pino’s exit flush before the file destination is ready → “sonic boom is not ready yet”. Module-init file logging is also a poor fit.
- **Approach:** Move Node logging into `core/logging/node/` (explicit imports only). Add `fileLogger.ts` that creates a file logger after awaiting the destination’s `ready` event. TUI (and web where applicable) init the logger asynchronously and await it before doing any work that might exit. Top-level `core/logging` has no exports; only `core/logging/node` is exposed.
