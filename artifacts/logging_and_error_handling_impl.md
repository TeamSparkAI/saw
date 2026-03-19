# Logging and error handling in TUI (implementation)

---

**User**

OK, implement the plan in the spec

---

**Cursor**


Implementing the spec: reading the spec and locating all affected files.



Creating the core logging/node files and applying the rest of the changes.



Implementing TUI logger init and runTui await:



Implementing web: async config and file logger.



Building and checking for issues:



Verifying launcher build and running core tests:



Summary of what was implemented:

**1. Core logging under `core/logging/node/`**
- **`core/logging/node/logger.ts`** – `silentLogger` (moved from `core/logging/logger.ts`).
- **`core/logging/node/fileLogger.ts`** – `createFileLogger(options)` that creates a pino file destination, awaits its `ready` event, then returns the logger as a `Promise<Logger>`.
- **`core/logging/node/index.ts`** – Exports `silentLogger`, `createFileLogger`, and `CreateFileLoggerOptions`.
- **Removed** `core/logging/logger.ts` and `core/logging/index.ts`.
- **`core/package.json`** – Replaced `"./logging"` and `"./logging/*"` with `"./logging/node"` and `"./logging/node/*"`.

**2. Imports updated to `logging/node`**
- **`core/auth/index.ts`** – Re-exports `silentLogger` from `../logging/node/index.js`.
- **`core/mcp/inspectorClient.ts`** – Imports `silentLogger` from `../logging/node/logger.js`.
- **`clients/web`** – `AuthDebugger.tsx` and `OAuthCallback.tsx` import from `@modelcontextprotocol/inspector-core/logging/node`.

**3. TUI**
- **`clients/tui/src/logger.ts`** – `tuiLogger` is a mutable export (starts as `silentLogger`). `initTuiLogger()` uses `createFileLogger` from core, awaits it, and assigns the result to `tuiLogger`.
- **`clients/tui/tui.tsx`** – Calls `await initTuiLogger()` at the start of `runTui()` before any logic that can call `program.error()`.

**4. Web**
- **`clients/web/src/web-server-config.ts`** – `buildWebServerConfigFromEnv()` is async; when `MCP_LOG_FILE` is set it uses `createFileLogger` and awaits it.
- **`clients/web/src/server.ts`** – `runStandalone()` uses `await buildWebServerConfigFromEnv()`.
- **`clients/web/src/vite-hono-plugin.ts`** – Plugin takes `WebServerConfig | Promise<WebServerConfig>`, awaits it in `configureServer`, and uses `resolvedConfig` for the rest of the callback.

**Verification**
- `npm run build-core`, `build-tui`, `build-web`, `build-launcher` all succeed.
- Core tests: 486 passed.
- TUI with no target exits with the expected message and no “sonic boom is not ready yet” stack trace.

---

**User**

For the web app, how is the file logger created when we don't use buildWebServerConfigFrom Env (or do we always use that)?

---

**Cursor**


We don’t always use `buildWebServerConfigFromEnv`. There are two separate paths:

**1. Paths that use `buildWebServerConfigFromEnv()` (file logger when `MCP_LOG_FILE` is set)**  
- **Standalone server**: `node dist/server.js` (or similar) → `server.ts` calls `await buildWebServerConfigFromEnv()`.  
- **Vite dev**: `vite.config.ts` passes `buildWebServerConfigFromEnv()` into `honoMiddlewarePlugin()`, so dev mode also builds config from env and gets a file logger when `MCP_LOG_FILE` is set.

**2. Path that does not use it (no file logger)**  
- **Runner**: `runWeb()` in `web.ts` (e.g. `npm run inspector -- --web` or with server args) builds config with **`buildWebServerConfig()`**, which takes parsed argv and always sets **`logger: undefined`** (see `web.ts` around 137). So when the web app is started via the launcher/runner, the file logger is never created and `MCP_LOG_FILE` is ignored.

**Summary**

| Entry point              | Config builder                  | File logger |
|--------------------------|----------------------------------|------------|
| `node dist/server.js`   | `buildWebServerConfigFromEnv()` | Yes if `MCP_LOG_FILE` set |
| Vite dev (`npm run dev`) | `buildWebServerConfigFromEnv()` | Yes if `MCP_LOG_FILE` set |
| Runner (`inspector --web`) | `buildWebServerConfig()`       | No (`logger: undefined`) |

So the file logger is only created when config comes from env (standalone server or Vite dev). To support file logging when starting via the runner, you’d need to have `runWeb()` (or a helper it calls) check `process.env.MCP_LOG_FILE`, call `createFileLogger` when set, and pass that logger into the config built by `buildWebServerConfig()`.

---

**User**

That seems like a problem.

---

**Cursor**


Adding file-logger support for the runner path so `MCP_LOG_FILE` is respected when starting the web app via the launcher.



Summary of changes:

**`clients/web/src/web.ts`**

1. **Import** – `createFileLogger` from `@modelcontextprotocol/inspector-core/logging/node`.

2. **`buildWebServerConfig`** – Added an optional final argument `logger?: WebServerConfig["logger"]` and use it in the returned config instead of always `undefined`.

3. **`runWeb()`** – Before calling `buildWebServerConfig`, if `process.env.MCP_LOG_FILE` is set, `await createFileLogger({ dest, append: true, mkdir: true, level: "info" })` and pass that as the logger; otherwise pass `undefined`. Then pass this logger into `buildWebServerConfig`.

With this, `MCP_LOG_FILE` is used for file logging in all three web entry points:

- Runner (`inspector --web`)
- Standalone server (`node dist/server.js`)
- Vite dev (via `buildWebServerConfigFromEnv()` in vite.config)

Web build completes successfully.

---

**User**

why would you type file logger as webserverconfig["logger"] - isn't there an SDK type that would work better there?  Or am I missing something?

---

**Cursor**


Using the pino `Logger` type instead of `WebServerConfig["logger"]`. Checking the code and fixing it.



Using pino's `Logger` type directly.



Updated. The file logger is now typed as **`Logger | undefined`** from **pino** (with `import type { Logger } from "pino"`), and `buildWebServerConfig`’s optional parameter uses the same type.

So the type comes from the logging library we’re using, not from `WebServerConfig`. That keeps the code about “a pino logger” rather than “whatever the config expects.”

---

**User**

ok, so in any useage of the web app, it will only log to a file logger of MCP_LOG_FILE is set in the env.  And if so, is that the full path to the log file?

---

**Cursor**


**1. File logging only when `MCP_LOG_FILE` is set**

Yes. In every web entry point we only create a file logger when `process.env.MCP_LOG_FILE` is set:

- **Runner** (`web.ts`): `if (process.env.MCP_LOG_FILE) { fileLogger = await createFileLogger({ dest: process.env.MCP_LOG_FILE, ... }); }` else `fileLogger = undefined`.
- **Standalone server** (`web-server-config.ts` `buildWebServerConfigFromEnv`): `if (process.env.MCP_LOG_FILE) { logger = await createFileLogger({ dest: process.env.MCP_LOG_FILE, ... }); }`.
- **Vite dev**: same `buildWebServerConfigFromEnv()`, so again only when `MCP_LOG_FILE` is set.

If `MCP_LOG_FILE` is unset, `config.logger` is `undefined` and the server doesn’t use a file logger.

**2. Is it the full path to the log file?**

We use the value of `MCP_LOG_FILE` as the `dest` for pino’s destination with no extra path logic. So:

- You can set it to a **full path**, e.g. `MCP_LOG_FILE=/var/log/mcp-inspector.log` or `MCP_LOG_FILE=/home/user/.mcp-inspector/web.log`.
- You can also set it to a **relative path** (e.g. `./logs/app.log`); it will be resolved relative to the process cwd when the server starts.

So it’s “the path you want for the log file” (usually a full path). We don’t add a default directory or filename; whatever you set is what’s used.

---

**User**

how would i set an env var for the context of one npm run

---

**Cursor**


Set it only for that one command by putting it in front of the command. The variable applies only to that run.

**Unix/macOS (bash/zsh):**
```bash
MCP_LOG_FILE=/path/to/app.log npm run inspector -- --web
```
or with a space before the command (same effect):
```bash
MCP_LOG_FILE=/path/to/app.log npm run web
```

**Windows (cmd):**
```cmd
set MCP_LOG_FILE=C:\path\to\app.log && npm run inspector -- --web
```

**Windows (PowerShell):**
```powershell
$env:MCP_LOG_FILE="C:\path\to\app.log"; npm run inspector -- --web
```

**Multiple vars (Unix):**
```bash
MCP_LOG_FILE=./web.log CLIENT_PORT=3000 npm run inspector -- --web
```

No need to export; the inline `VAR=value` (or Windows equivalent) only affects that single process.

---

**User**

For both tui and web lets log a startup event so we can test logging

---

**Cursor**


Adding a startup log for the TUI (right after init) and for the web app when a file logger is present.



Summary of changes:

**TUI** (`clients/tui/tui.tsx`):
- After `await initTuiLogger()`, added `tuiLogger.info("TUI starting")`.
- Runs on every TUI start; output goes to `~/.mcp-inspector/auth.log` (or `MCP_INSPECTOR_LOG_DIR` if set).

**Web**:
- **Prod** (`clients/web/src/server.ts`): At the start of `startHonoServer()`, if `config.logger` is set, call `config.logger.info("Web server starting")`.
- **Dev** (`clients/web/src/vite-hono-plugin.ts`): After resolving config, if `resolvedConfig.logger` is set, call `resolvedConfig.logger.info("Web server starting (dev)")`.

So:
- **TUI**: Always logs a startup line to the TUI log file.
- **Web**: Logs a startup line only when `MCP_LOG_FILE` is set (file logger enabled). Use e.g. `MCP_LOG_FILE=./web.log npm run inspector -- --web` and then check `web.log` for the startup line.

---

**User**

I don't love "when the logger is present".  Does it make sense to use silentLogger so a logger is always present and we don't have to check every single time we want to log something?

---

**Cursor**


Using the null-object pattern: always set `config.logger` (use `silentLogger` when not file-logging) so we can call `config.logger.info(...)` without checks.



Summary of changes:

**1. `WebServerConfig`**
- `logger` is now required and typed as `Logger` (no longer `Logger | undefined`).

**2. Config builders always set a logger**
- **`buildWebServerConfigFromEnv()`** (web-server-config.ts): Uses `process.env.MCP_LOG_FILE ? await createFileLogger(...) : silentLogger`, so the config always has a `Logger`. Imports `silentLogger` from `@modelcontextprotocol/inspector-core/logging/node`.
- **`buildWebServerConfig()`** (web.ts): Parameter is now `logger: Logger` (required). Call site uses `logger = process.env.MCP_LOG_FILE ? await createFileLogger(...) : silentLogger` and passes that in.

**3. Call sites no longer guard on logger**
- **server.ts**: Replaced `if (config.logger) { config.logger.info(...) }` with `config.logger.info("Web server starting")`.
- **vite-hono-plugin.ts**: Replaced `if (resolvedConfig.logger) { ... }` with `resolvedConfig.logger.info("Web server starting (dev)")`.

So the web app always has a logger (file logger when `MCP_LOG_FILE` is set, otherwise `silentLogger`), and code can call `config.logger.info(...)` without checks. `createRemoteApp` still receives a `Logger` and continues to accept it as before.

---

**User**

does the cli use the pino / file logger

---

**Cursor**


**No.** The CLI does not use a pino file logger (or any custom logger).

In `clients/cli/src/cli.ts`, `InspectorClient` is created with:

```61:64:clients/cli/src/cli.ts
  const inspectorClient = new InspectorClient(serverConfig, {
    environment: {
      transport: createTransportNode,
    },
```

There is no `logger` on `environment`, so `InspectorClient` uses its default: **`silentLogger`** from core (pino with `level: "silent"`), so those log calls don’t write anywhere useful.

The CLI also doesn’t reference `pino`, `createFileLogger`, or `MCP_LOG_FILE`. File logging is only wired up for **TUI** and **web** today.

---

**User**

it's odd that the tui uses a different log param than the web and that the cli doesn't use a file logger at all.  Let's make all three apps use the MCP_LOG_FILE env var (as web does now), and if specified use a file logger at that path, else silent logger.

---

**Cursor**


Unifying logging: TUI and CLI will use `MCP_LOG_FILE` + `createFileLogger` / `silentLogger` like the web app.




Implementing TUI and CLI changes and updating READMEs.




Here’s what changed:

### TUI (`clients/tui/src/logger.ts`)
- **Removed** `MCP_INSPECTOR_LOG_DIR` / fixed `~/.mcp-inspector/tui.log` behavior.
- **`MCP_LOG_FILE` set** → `createFileLogger` at that path (same options as web: append, mkdir, `LOG_LEVEL` default `info`, name `mcp-inspector-tui`).
- **Unset** → `silentLogger` (no file; `initTuiLogger()` is still cheap).

### CLI (`clients/cli/src/cli.ts`)
- Before `InspectorClient`, builds **`logger`** the same way: `MCP_LOG_FILE` → `createFileLogger` (`name: mcp-inspector-cli`, level `info`), else `silentLogger`.
- Passes **`logger`** in `environment` with `createTransportNode`.
- **`logger.info("CLI starting")`** so file logs show a startup line when `MCP_LOG_FILE` is set (silent logger discards it).

### Docs
- **TUI README** – env table for `MCP_LOG_FILE` and `LOG_LEVEL`.
- **CLI README** – env table for `MCP_LOG_FILE`.
- **Web README** – log file row now says the same env is used for web, TUI, and CLI.

Web behavior is unchanged: `MCP_LOG_FILE` + file logger, otherwise `silentLogger` on the server config.

**Note:** TUI no longer writes to `~/.mcp-inspector/auth.log` unless you set **`MCP_LOG_FILE`** to that path (or any path you want).

---

**User**

We don't need to reference "same as xxxx" in the client readmes (they are supposed to be standalone clients)

---

**Cursor**


Updates made:

- **Web README** — Dropped “(same env for web, TUI, and CLI)” from the log file row.
- **TUI README** — Dropped “(same as web/CLI)” from the `MCP_LOG_FILE` row.
- **CLI README** — Dropped “(same as web/TUI)” from the `MCP_LOG_FILE` row.

The MCP server option blurbs that say options are documented in `mcp-server-configuration.md` are unchanged; say if you want those rewritten so they don’t mention other clients either.

---

**User**

let's remove LOG_LEVEL support from the TUI for now (since it's the only client that supports it and we don't really have a level stragy yet) - just use "info" like the other loggers.

---

**Cursor**


Summary of changes:

- **`clients/tui/src/logger.ts`** — File logger now always uses `level: "info"` (no `LOG_LEVEL` env).
- **`clients/tui/README.md`** — Removed the `LOG_LEVEL` row from the logging env table.

There are no remaining `LOG_LEVEL` references in the repo.

---

**User**

When the CLI run fails because it has no config (or possibly when invalid config), it fails with a stack trace instead of just an error message, as in: 

```
Error running MCP Inspector: Error: Target is required. Specify a URL or a command to execute.
    at buildConfigFromOptions (file:///Users/bob/Documents/GitHub/inspector/core/build/mcp/node/config.js:134:15)
    at resolveServerConfigs (file:///Users/bob/Documents/GitHub/inspector/core/build/mcp/node/config.js:210:17)
    at parseArgs (file:///Users/bob/Documents/GitHub/inspector/clients/cli/build/cli.js:239:21)
    at runCli (file:///Users/bob/Documents/GitHub/inspector/clients/cli/build/cli.js:271:42)
    at run (file:///Users/bob/Documents/GitHub/inspector/clients/launcher/build/index.js:31:15)
```

---

**Cursor**


Tracing how the CLI handles parse errors and how the launcher reports them.




Wrapping `runCli` in try/catch and using the CLI's `handleError` (prints only the message, then exits). The launcher is invoked with `runCli` directly, so errors from `parseArgs` currently propagate and the launcher prints the full Error (including stack).




Here’s what changed.

**Cause:** `runCli()` didn’t catch failures from `parseArgs()` (e.g. missing target). That error bubbled to the launcher, which does `console.error("Error running MCP Inspector:", err)` and Node prints the full `Error` (message + stack).

**Change:** Wrap the body of `runCli()` in `try/catch` and call existing **`handleError(error)`**, which only prints **`error.message`** on stderr and **`process.exit(1)`**.

So for bad/missing config you get a single line, e.g.  
`Target is required. Specify a URL or a command to execute.`  
with no stack, and the process exits before the launcher’s catch runs (so no `"Error running MCP Inspector:"` line either).

The same path applies to other CLI failures (missing `--method`, invalid transport, MCP errors from `callMethod`, etc.): one message on stderr, exit 1, aligned with how **`main()`** already used `handleError` for uncaught rejections.

