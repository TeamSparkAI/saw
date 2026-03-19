# Logging and error handling in TUI (specification)

---

**User**

On starting the TUI with no parameters, it exits with expected error, but then dumps the following stack trace to the console.  I don't want hacks to defer initializing logging until after we've decided if we're exiting early.  I want logging enabled (so that I could log the exit condition, if I wanted) and for it to terminate cleanly when exiting at any point, and to the extent that this same issue applies with other clients (cli or web) we need a solution for all apps.

```
bob@Bobs-MacBook-2450 inspector % npm run inspector -- --tui                          
npm info using npm@10.9.4
npm info using node@v22.22.0

> @modelcontextprotocol/inspector@0.20.0 inspector
> node clients/launcher/build/index.js --tui

Target is required. Specify a URL or a command to execute.
Error running MCP Inspector: Error: sonic boom is not ready yet
    at SonicBoom.flushSync (/Users/bob/Documents/GitHub/inspector/node_modules/sonic-boom/index.js:551:11)
    at autoEnd (/Users/bob/Documents/GitHub/inspector/node_modules/pino/lib/tools.js:328:12)
    at callRefs (/Users/bob/Documents/GitHub/inspector/node_modules/on-exit-leak-free/index.js:55:7)
    at process.onExit (/Users/bob/Documents/GitHub/inspector/node_modules/on-exit-leak-free/index.js:39:3)
    at process.emit (node:events:531:35)
    at process.exit (node:internal/process/per_thread:240:15)
    at Command._exit (/Users/bob/Documents/GitHub/inspector/node_modules/commander/lib/command.js:522:13)
    at Command.error (/Users/bob/Documents/GitHub/inspector/node_modules/commander/lib/command.js:1917:10)
    at runTui (file:///Users/bob/Documents/GitHub/inspector/clients/tui/build/tui.js:50:21)
    at run (file:///Users/bob/Documents/GitHub/inspector/clients/launcher/build/index.js:35:15)
```

---

**Cursor**


Tracing how logging is initialized and where the early exit occurs so we can fix the SonicBoom flush-on-exit error.



Checking where "Target is required" is thrown and how the CLI and web app create file loggers:



Checking how `resolveServerConfigs` is used in the TUI so we see the full flow:



Checking pino's types for the destination:



Implementing the fix: adding a sync file destination in core and using it in the TUI and web app.


---

**User**

let's make a new branch for this fix - this should be a branch on my fork that is based on upstream v1.5 called "logging-cleanup".  Then do not implement a solution, but instead explain your approach.

---

**Cursor**


Creating the branch, then explaining the approach.



Creating branch `logging-cleanup` from `upstream/v1.5/main`.



Branch **logging-cleanup** is created on your fork from **upstream/v1.5/main** and is checked out. Pushing it to your fork: `git push -u origin logging-cleanup` (when you’re ready).

---

## Approach (no code changes)

**What’s going wrong**  
When the TUI (or any client) exits early (e.g. “Target is required”), the process calls `process.exit()`. Pino registers an exit handler that flushes its destination. The TUI (and web when `MCP_LOG_FILE` is set) use `pino.destination({ dest: logPath, append: true, mkdir: true })`, which opens the file **asynchronously** via SonicBoom. If we exit before that open finishes, the exit handler runs `flushSync()` on a SonicBoom that isn’t ready yet → “sonic boom is not ready yet”. So the fix is to make the file destination **ready before** any exit can happen, without deferring logger creation.

**Direction**  
Use a **synchronously opened file descriptor** as the pino destination so the stream is ready from the first use and the exit handler can always flush safely.

1. **Shared helper in core**  
   Add something like `createSyncFileDestination(logPath: string)` under `core/logging` that:
   - Creates the directory with `fs.mkdirSync(..., { recursive: true })`
   - Opens the file with `fs.openSync(logPath, 'a')`
   - Returns `pino.destination(fd)`  
   Pino then wraps an already-open fd; SonicBoom treats that as ready, so exit-time flush no longer hits “not ready”.

2. **TUI**  
   In `clients/tui/src/logger.ts`, replace `pino.destination({ dest: logPath, append: true, mkdir: true })` with this helper (same path). Logging stays enabled from load; early exit still flushes cleanly.

3. **Web**  
   In `clients/web/src/web-server-config.ts`, when `MCP_LOG_FILE` is set, create the file logger using the same helper instead of `pino.destination({ dest: ..., append: true, mkdir: true })`, so web server exits (e.g. bad config) don’t trigger the same SonicBoom error.

4. **CLI**  
   From the earlier search, the CLI doesn’t create a file logger in the same way; only TUI and web do. So no change there unless a file logger is added later—in that case use the same helper.

**Why this fits your constraints**  
- Logging is enabled from the start (no “defer init until after we know we’re exiting”).  
- Exit is clean: the fd is open before any exit handler runs.  
- One shared helper keeps behavior consistent across TUI and web and any future file loggers.  
- No change to when or how we call `process.exit()`; we only change how the log destination is created.

---

**User**

can't we just wait on the pino init somehow?  This seems like a pretty customized solution (and/or hack) for something that must be a pretty common issue for people using this tooling.  If there is no init to await, would awaiting a startup log event output do the trick?

---

**Cursor**


Checking whether pino or sonic-boom expose a way to wait for the destination to be ready.



Checking the pino issue and whether `logger.flush()` returns a promise:



Checking whether pino's `flush` returns a promise we can await:



Yes. We can wait, and that’s the approach the pino maintainers recommend.

---

**Wait for the destination `ready` event**

Sonic-boom (used by `pino.destination()`) emits a **`ready`** event when the file descriptor is open. The maintainers explicitly recommend using it so you don’t hit “sonic boom is not ready yet” when exiting early:

- Create the **destination** with `pino.destination({ dest: logPath, append: true, mkdir: true })`.
- Wait for **ready** before using it:  
  `await new Promise(resolve => dest.once('ready', resolve))`.
- Then create the logger with that destination:  
  `pino(options, dest)`.

So we’re not adding a custom sync-open; we’re using the supported “wait until the destination is ready” pattern.

**Where to wait**

- **TUI**: In `runTui()`, await a “logger ready” step (e.g. a small helper that creates the dest, awaits `ready`, then returns the logger) **before** any logic that can call `program.error()` / `process.exit()` (e.g. before `resolveServerConfigs`). Logging stays enabled from the start; we only delay “doing things that might exit” until after the destination is ready.
- **Web**: Same idea where the file logger is created (e.g. in `web-server-config` when `MCP_LOG_FILE` is set): create dest, await `ready`, then create the logger and use it. No need to change when the server starts; we just make “logger creation” async and await it before continuing.

**Awaiting a startup log instead?**

In theory, if we could await “first write finished”, that would also imply the destination is ready. In practice, pino’s API doesn’t give you a promise that a specific log line has been written; `flush` is the only way to wait for writes, and the maintainers point to the **`ready`** event as the solution rather than “log then flush then exit.” So the clean, supported approach is: **wait on `ready`**, not on a startup log. If we wanted, we could still do an initial `logger.info('…')` after `ready` for a startup line, but the thing we await should be `ready`, not that log.

---

**User**

Show me how we create the destination currently

---

**Cursor**


We create the destination in two places:

**1. TUI — `clients/tui/src/logger.ts`**

```17:24:clients/tui/src/logger.ts
export const tuiLogger = pino(
  {
    name: "mcp-inspector-tui",
    level: process.env.LOG_LEVEL ?? "info",
  },
  pino.destination({ dest: logPath, append: true, mkdir: true }),
);
```

So the destination is `pino.destination({ dest: logPath, append: true, mkdir: true })` with `logPath` = `~/.mcp-inspector/auth.log` (or `MCP_INSPECTOR_LOG_DIR` + `auth.log`).

**2. Web — `clients/web/src/web-server-config.ts`** (only when `MCP_LOG_FILE` is set)

```168:176:clients/web/src/web-server-config.ts
  let logger: Logger | undefined;
  if (process.env.MCP_LOG_FILE) {
    logger = pino(
      { level: "info" },
      pino.destination({
        dest: process.env.MCP_LOG_FILE,
        append: true,
        mkdir: true,
      }),
    );
  }
```

So in both cases the destination is created with **`pino.destination({ dest: <path>, append: true, mkdir: true })`** and passed straight into `pino(...)`. There’s no `ready` wait or separate dest variable; that’s what leads to the “sonic boom is not ready yet” error when the process exits before the async open finishes.

---

**User**

are both of those in async contexts where we can await 

---

**Cursor**


Checking the surrounding context for both:



**Short answer: no. One place is sync-only; the other is sync too, but the caller is async.**

**1. TUI — `clients/tui/src/logger.ts`**  
The destination is created at **top-level** when the module loads (no function, no `async`). So that code is **fully synchronous** and there’s nowhere to `await`.  
The only async context is **`runTui()`** in `tui.tsx` (`export async function runTui`). So we *can* await there, but only if the logger is provided in an awaitable way (e.g. a helper that returns a Promise that resolves to the logger after `ready`), and we call it at the start of `runTui` before any code that might call `program.error()`.

**2. Web — `clients/web/src/web-server-config.ts`**  
The destination is created inside **`buildWebServerConfigFromEnv()`**, which is a **synchronous** function (lines 151–194). It’s not `async` and it returns `WebServerConfig` directly. So that code is **not** in an async context; we can’t `await` there without changing the design.  
To use `await` we’d have to make `buildWebServerConfigFromEnv()` `async`, have it return `Promise<WebServerConfig>`, and update every caller to `await buildWebServerConfigFromEnv()`.

**Summary**

| Place   | Where destination is created              | In async code? |
|--------|--------------------------------------------|----------------|
| TUI    | Top-level in `logger.ts` (module load)     | No             |
| Web    | Inside `buildWebServerConfigFromEnv()`     | No (sync fn)   |

So neither creation site is currently in async code where we can `await`. For TUI we can keep creation in `logger.ts` but expose an awaitable “logger ready” and `await` it in `runTui()`. For web we’d need to make config building async (async function + awaited callers) to await destination readiness there.

---

**User**

it seems like initializing the logger in module init is not a good design anyway.  Would it make more sense to make an async logger init method in the tui logger, and then call that early in runTui and await it?

---

**Cursor**


Yes, that makes sense.

**Why init-at-module-load is a bad fit**
- File open is I/O and can fail; doing it at import time is hard to handle and can break the app before the entrypoint runs.
- It forces the logger to be ready (or not) before you know if you’ll even need it (e.g. before `runTui` decides to exit early).
- It’s harder to test (you can’t easily swap or delay init).

**Async init called from `runTui`**
- **`logger.ts`**: Export something like `initTuiLogger(): Promise<pino.Logger>` that:
  - Creates the destination,
  - Awaits `ready` (e.g. `new Promise(res => dest.once('ready', res))`),
  - Builds the logger with that dest,
  - Stores it in module scope (or a small holder),
  - Returns (and/or exposes) the logger.
  Optionally also export `getTuiLogger(): pino.Logger` that returns the stored instance and throws if init hasn’t run, so existing `tuiLogger` call sites can become `getTuiLogger()` if you don’t want to pass the logger everywhere.
- **`tui.tsx`**: At the very start of `runTui()`, before parsing/validation that might call `program.error()`, do `await initTuiLogger()` (and use the returned logger or `getTuiLogger()` when building the app / server config).

Then the logger is created and ready only when the TUI actually runs, in an async context where you can await `ready`, and you avoid both the “sonic boom is not ready” issue and the brittle module-init I/O pattern.

---

**User**

is the tui logger actually used in the tui app?

---

**Cursor**


Yes. It’s used in **`clients/tui/src/App.tsx`** in several places:

- **Line 54**: `import { tuiLogger } from "./logger.js";`
- **Line 151**: `tuiLogger.info({ serverNames: Object.keys(mcpServers) }, "TUI started");`
- **Line 277**: passed as `logger: tuiLogger` (into the InspectorClient/environment)
- **Lines 544, 621, 683, 712**: `tuiLogger.info(...)` for OAuth/auth and related events

So the TUI logger is used for startup logging, as the InspectorClient logger, and for auth/OAuth logging.

---

**User**

And these assume module scope (and that the logger is initialized)?  So can we init/await the logger on runTui and leave the rest of the code as is using the module-level tui logger?

---

**Cursor**


Yes. You can do it that way and keep all existing `tuiLogger` usage as-is.

**1. Module-level export that gets replaced after init**

In `logger.ts`:

- Keep a single exported binding: `export let tuiLogger: pino.Logger`.
- Give it an initial value so the module is valid at load time, e.g. a silent logger:  
  `tuiLogger = pino({ level: "silent" })`.
- Export `initTuiLogger(): Promise<void>` that: creates the destination, awaits `ready`, builds the real file logger, then does **`tuiLogger = realLogger`**.

**2. runTui**

At the very start of `runTui()`, before any parsing/validation that might call `program.error()`:

- `await initTuiLogger();`
- Then the rest of the function runs as it does today.

**3. Rest of the app**

- App (and anything else) keeps using `import { tuiLogger } from "./logger.js"` and calling `tuiLogger.info(...)` and `logger: tuiLogger`.
- No need to pass the logger around or change call sites.

Because ES module bindings are live, once `initTuiLogger()` has run and assigned the real logger to `tuiLogger`, every import of `tuiLogger` sees that same instance. By the time any TUI code (including App) runs, `runTui` has already awaited init, so they always see the real logger. The initial silent logger is only there so the export exists before init runs; nothing in the TUI runs before that first `await initTuiLogger()` in `runTui`.

---

**User**

ok, is there a good place to put shared code between the TUI and web ux? 

---

**Cursor**


Yes. The right place is **`core`** (`@modelcontextprotocol/inspector-core`). Both TUI and web already depend on it, and it’s the shared layer for “web, CLI, and TUI” (per AGENTS.md).

For **logging** in particular, use **`core/logging`**. It already has `silentLogger` and is exported as `@modelcontextprotocol/inspector-core/logging`. You can add a shared helper there (e.g. “create file logger that waits for destination `ready`”) and have both clients use it:

- **TUI**: `clients/tui` → `import { createFileLogger } from "@modelcontextprotocol/inspector-core/logging";` (or a similar export).
- **Web**: `clients/web` → same import from core when building the server config file logger.

For other **TUI + web shared UX** code (not just logging), still put it in **core**, under a subpath that matches the concern, for example:

- **`core/logging/*`** – logger creation, file logger helpers.
- **`core/react/*`** – already used for React hooks and state used by the web app (and potentially by TUI if it shares any React/Ink patterns).
- A new subpath (e.g. **`core/clients/*`** or **`core/ui/*`**) only if you need a dedicated namespace for “shared client/UX” code that isn’t logging or react.

Constraint: shared code in core must be **Node-safe**, because TUI runs only in Node and the web app has a Node server; browser-only code can’t live in a TUI+web shared layer. The existing `core/react`, `core/auth`, `core/mcp` are already used from both sides, so that pattern is established.

---

**User**

In core we have tried to separate the pure javascript shared code from the node code and the react code, using node and react subdirs for that.  Is pino considered "node" (versus pure js), and if not, is the current silentLogger implementation considered node code?

---

**Cursor**


**Is pino “Node” vs pure JS?**  
Yes. The default `pino` entry is the **Node** build (Node streams, stdout, etc.). The **browser** build is a separate entry: `pino/browser` (and that’s what `createRemoteLogger.ts` uses for the browser/remote logger). So in this repo, “pino” without “/browser” means the Node build.

**Is the current `silentLogger` implementation Node code?**  
Yes. It does `import pino from "pino"`, so it uses the Node pino build. With no destination, that logger still uses Node’s default (e.g. stdout). So `core/logging/logger.ts` is Node-dependent, not pure JS.

Given your split (pure JS vs `node/` vs `react/`), that file is **Node** code. To match the pattern used in `core/auth` and `core/mcp` (e.g. `auth/node`, `mcp/node`), you could:

- Move Node-specific logging under **`core/logging/node/`** (e.g. `core/logging/node/logger.ts` exporting `silentLogger`, and any future file-destination helpers there), and
- Keep **`core/logging/index.ts`** as a thin re-export from `./node/...` (or only export pure/type-only things there if you add them later).

That way “logging” stays the top-level domain, and the fact that the default logger and file logging are Node-specific is clear from the `node/` path.

---

**User**

I think the consistent design would be to move core/logging to core/logging/node and add the new file logger creation helper there.  Does that make sense?

---

**Cursor**


Yes. That matches the pattern used elsewhere in core (e.g. `auth/node`, `mcp/node`).

**Layout:**

- **`core/logging/node/`** – all Node logging:
  - Current code: e.g. **`core/logging/node/logger.ts`** (or keep the name and move the file) with `silentLogger`.
  - New helper: e.g. **`core/logging/node/createFileLogger.ts`** (or similar) that creates a pino file destination, awaits `ready`, and returns the logger (or a Promise of it).
  - **`core/logging/node/index.ts`** – exports `silentLogger` and the new file-logger helper.

- **`core/logging/index.ts`** – stays at the top level and re-exports from `./node` (e.g. `export { silentLogger, createFileLogger } from "./node/index.js";`). Then existing imports from `@modelcontextprotocol/inspector-core/logging` keep working without changing every caller.

**Package exports:**  
In `core/package.json` you already have `"./logging": "..."` and `"./logging/*": "..."`. Add an explicit Node entry if you want direct imports, e.g. `"./logging/node": "./build/logging/node/index.js"` (and optionally `"./logging/node/*"`), and have `./logging` point at `build/logging/index.js` so the re-export works.

So: move current logging implementation into **`core/logging/node/`**, add the file-logger helper there, and keep **`core/logging`** as a thin re-export from `node` for a consistent, backward-compatible design.

---

**User**

Lets call it fileLogger.ts.  Also, exporting from the directory above is a bad idea.  The entire point is to keep the node and pure JS code separate, so we want to make the fact that node code is being imported explicit.  

Create a markdown document explaining the problem and our proposed solution

---

**Cursor**


Created **`docs/logging-file-logger-spec.md`** that covers:

1. **Problem** – “sonic boom is not ready yet” on early exit, why it happens (async destination + exit flush), and why module-init file logging is a bad fit.
2. **Proposed solution** –  
   - Node-only code under **`core/logging/node/`**; no re-export from the top-level logging entry so Node usage stays explicit.  
   - **`core/logging/node/fileLogger.ts`** as the shared helper that waits for the destination `ready` event.  
   - **`core/logging/node/logger.ts`** for `silentLogger`, **`core/logging/node/index.ts`** exporting both.  
   - TUI: **`initTuiLogger()`** plus module-level **`tuiLogger`** (silent placeholder until init), **`await initTuiLogger()`** at the start of **`runTui()`**.  
   - Web: use the same file-logger helper and make config building async where a file logger is created.  
   - Package exports: explicit **`./logging/node`** (and **`./logging/node/*`**), no Node re-exports from **`./logging`**; callers import from **`@modelcontextprotocol/inspector-core/logging/node`**.

No code or exports were changed; only the markdown doc was added.

---

**User**

is the remote logging in core or somewhere else?

---

**Cursor**


It’s in **core**, under the remote MCP layer:

- **`core/mcp/remote/createRemoteLogger.ts`** – Builds a pino logger that sends log events to the remote **`/api/log`** endpoint (used by the web client so browser logs go to the server).
- **`core/mcp/remote/node/server.ts`** – Defines the **`POST /api/log`** handler and can forward received events to an optional pino file logger.

So “remote logging” is the **mcp/remote** feature in core (client-side `createRemoteLogger` + server-side `/api/log`), not under **core/logging**. The **core/logging** (and **core/logging/node**) work is only for Node file loggers (TUI and web server when `MCP_LOG_FILE` is set).

---

**User**

ok, the design looks fine, except for one thing.  If we move the existing logging code to node logging, then there is nothing left in logging to export (it is empty, and we don't want to export the node code from it, we only export that form the explicit logging/node dir).  Update the spec.

---

**Cursor**


Updated the spec to state that top-level `core/logging` has no exports (only `core/logging/node` is exposed), and removing `core/logging/README.md`.

