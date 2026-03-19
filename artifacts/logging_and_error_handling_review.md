# Logging and error handling in TUI (Implement PR Review Feedback)

---

**User**

The silentLogger is not node-specific and is used by web components.  We need to move it back up to core/logger and update it's consumers (specifically so that web app components aren't importing it or anything from a node dir)

---

**Cursor**


Moving `silentLogger` to `core/logging/logger.ts` (browser-safe) and updating consumers. Searching the codebase for imports.




Creating `core/logging/logger.ts` using `pino/browser.js` (matches `createRemoteLogger`, works in browser and Node tests). Updating exports and consumers.




Here’s what changed:

### `core/logging/logger.ts` (new)
- Defines **`silentLogger`** with **`pino/browser.js`** (same idea as `createRemoteLogger`) so it’s safe in **browser bundles** and **Node tests**, without using the Node `pino` entry.

### `core/logging/node/`
- **Removed** `node/logger.ts`.
- **`node/index.ts`** only exports **`createFileLogger`** / **`CreateFileLoggerOptions`**.

### Package export
- **`@modelcontextprotocol/inspector-core/logging/logger`** → `core/logging/logger.js`

### Imports
| Consumer | Import |
|----------|--------|
| `AuthDebugger.tsx`, `OAuthCallback.tsx` | `@modelcontextprotocol/inspector-core/logging/logger` |
| `web-server-config.ts`, `web.ts` | `silentLogger` from `logging/logger`, `createFileLogger` from `logging/node` |
| `cli.ts`, `tui/src/logger.ts` | same split |
| `core/auth/index.ts`, `inspectorClient.ts` | `../logging/logger.js` |

**Build:** core, web, cli, tui all compile. **Core tests:** 486 passed.

---

**User**

What I wanted was a pure JS implementation that could be used in a browser or node context, and you just replaced all usage of silentLogger with a browser version, even in places like core, where it's almost certainly going to be used in a node environment.  Consider this and advise (don't code until I'm convinced you have a workable plan)

---

**Cursor**


You’re right: the goal is a **single, pure-JS** silent logger that works in both browser and Node, not “use the browser build everywhere.”

**What went wrong**

- `pino/browser.js` is still a pino build (browser-specific). Using it in core/CLI/TUI/server means Node code paths depend on the browser build and we have two pino worlds.
- That’s not “pure JS” and not “one implementation for both environments.”

**What you actually want**

- **One implementation**: a small, **pino-free** object that **implements the Logger contract** (only what’s used: e.g. `level`, `info`, `warn`, `error`, `debug`, `trace`, `fatal`, `child`, and any other method callers use).
- **Pure JS**: no `import pino from "pino"` or `import pino from "pino/browser.js"` in that file. Only `import type { Logger } from "pino"` so we can type the object as `Logger`.
- **Same code path everywhere**: core (Node), CLI, TUI, web server, and **browser** all import this same module; no Node-only or browser-only variant.

**Concrete plan**

1. **Implement a no-op logger in `core/logging/logger.ts`**
   - Plain object (or a tiny class): `level: "silent"`, and every method callers use implemented as no-ops (e.g. `info`, `warn`, `error`, `debug`, `trace`, `fatal` do nothing; `child()` returns the same object or an equivalent no-op logger).
   - Type it as `Logger` with `as Logger` (or satisfy the `Logger` type explicitly) so it type-checks wherever a `Logger` is expected.
   - No runtime import from `"pino"` or `"pino/browser"` in this file.

2. **Confirm the surface area**
   - Grep / read how `logger` is used in InspectorClient and any other consumer (e.g. `logger.info`, `logger.child`, `logger.flush`) and implement exactly those. If the pino `Logger` type has many methods, we can either implement each as no-op or use a **Proxy** that returns a no-op function for any method and `"silent"` for `level` (and `child()` returning the same proxy). That way we’re future-proof without hand-coding every pino method.

3. **Keep location and exports**
   - Keep `silentLogger` in `core/logging/logger.ts` and the existing package export `@modelcontextprotocol/inspector-core/logging/logger`. No new “browser vs node” split; one entry point.

4. **Leave consumers as they are**
   - Core, CLI, TUI, web server, and web components keep importing `silentLogger` from `logging/logger`. No switching to “browser version” anywhere; everyone gets the same pure-JS implementation.

**Result**

- **Browser**: no pino in the default-logger path; only the small no-op object.
- **Node**: same object; no `pino/browser` in core/CLI/TUI/server.
- **Single implementation**, pure JS, works in both environments. If this matches what you want, next step is to implement the no-op logger (and optionally the Proxy for full `Logger` compatibility) and then we can adjust types/details if needed.

---

**User**

So that's your "let's pretend we don't care about TypeScript and just do a giant hack and hope we get the type right and pray it never changes" approach.  Maybe think about how someone who cared about TypeScript would do it.  Consider how users of Pino with this likely very common problem handle it.  Again, don't code until I approve the direction.

---

**Cursor**


The hacky part was relying on a cast or a Proxy and hoping it matched `Logger` and wouldn’t break when pino’s types change. A TypeScript-minded approach is to **own the contract** and implement it properly.

**1. Define the contract the app actually needs**

- Don’t treat “must be exactly pino.Logger” as the goal. Instead, define an **interface that describes what Inspector and the rest of the app use** (e.g. `level`, `info`, `warn`, `error`, `debug`, `trace`, `child`, maybe `flush`).
- Put that in core (e.g. `core/logging/types.ts` or next to the logger), e.g. `InspectorLogger` or `LoggerLike`.
- Have **InspectorClient and all call sites** use this type (or `pino.Logger` only where you truly need full pino).

**2. Implement the silent logger against that contract**

- In `core/logging/logger.ts`, implement a **single object/class that explicitly implements that interface**: every method and property in the type is implemented (no-ops, `child()` returns the same or equivalent no-op).
- No `as Logger`, no Proxy. TypeScript then **enforces** that the implementation matches the interface. If the interface grows or changes, you get compile errors and fix them.

**3. How this fits with pino**

- Real loggers (from `createFileLogger`, `createRemoteLogger`, etc.) are still `pino.Logger`. They have `.info`, `.warn`, `.child`, etc., so they **structurally satisfy** your interface as long as it’s a subset of what pino exposes.
- So: **your code depends on your interface**; the silent logger is one implementation, pino loggers are another. No pretending about types.

**4. Handling pino’s own types**

- If you still need to pass this into APIs that are typed as `pino.Logger`, you have two paths:
  - **Narrow the dependency**: Type those APIs as taking your `InspectorLogger` (or whatever you name it) instead of `pino.Logger` where possible. Then the silent logger and pino loggers both satisfy the same, explicit contract.
  - **Or** keep `pino.Logger` at the boundary but ensure your interface is a **strict subset** of what pino’s type defines, and your implementation is typed as implementing that subset. Then the only “bridge” is at the boundary (e.g. one typed parameter), not a blanket cast on the whole object.

**5. How pino users usually handle “silent”**

- The standard pino way is `pino({ level: 'silent' })` — a real Logger instance. The twist here is you want **one** implementation that works in both Node and browser **without** pulling in either pino runtime. So the type-safe way is: **treat “logger” as an abstraction your app owns**, implement a silent version that satisfies that abstraction, and let pino loggers satisfy the same abstraction. That’s the approach above: own the interface, implement it properly, no giant hack, and TypeScript stays in the driver’s seat.

---

**User**

OK, let's back up.  There are only two modules in the web app that import silentLogger.  Do these modules even need to import silentLogger?  It seems odd that we'd have UX code creating / assinging a logger when that code isn't cabable of talking to node (if there is a logger they can use it will be a remote logger) and if not, it's not clear why they'd need to set the logger.  

---

**Cursor**


Checking how those two components use the logger and where it comes from:



Here's what the code is doing:

**Where the logger comes from**
- **createWebEnvironment()** always creates a **remote logger** (createRemoteLogger) and returns it with the environment. So whenever the app has an environment, it has a logger.
- **App** passes that as `inspectorLogger` into **AuthDebugger** and (in the main flow) **OAuthCallback**.
- The only place **logger** is explicitly **null** is in **App.tsx** around line 1720, where the **guided OAuth callback** route is rendered with `logger={null}` (lazy-loaded OAuthCallback path where there’s no client yet).

**How the components use it**
- Both do `const log = logger ?? silentLogger` and then call `log.info(...)`, `log.error(...)`, etc.
- So **silentLogger** is only used when the caller passes **null/undefined**, so those calls don’t throw.

**Do they need to import silentLogger?**
- **No.** In the web app, the only real logger they’ll ever get is the **remote logger** from the environment. When App passes `null`, the component is in a “no client / no env” situation and doesn’t need to log to the server anyway.
- So instead of “if no logger, use silentLogger”, the components can treat “no logger” as “don’t log”: use **optional chaining** like `logger?.info(...)` (and similarly for `.error`, `.warn`, etc.). When `logger` is null/undefined, those calls are no-ops and the components don’t need a fallback logger at all.

**Concrete direction**
- **Remove** the `silentLogger` import from **AuthDebugger** and **OAuthCallback**.
- Use **`logger`** only when it’s present: e.g. `logger?.info(...)`, `logger?.error(...)`, etc., and drop the `const log = logger ?? silentLogger` pattern.
- Type the prop as **`logger?: WebEnvironmentResult["logger"] | null`** (or equivalent); it stays optional.
- Then **no UX code creates or assigns a logger**, and **no UX code imports from core logging**. The only logger they use is the one the app gives them (the remote logger when it exists); when it doesn’t, they simply don’t log. No silentLogger, no node/browser logger split in the UI layer.

---

**User**

Ok, I have reverted all of your changes.  Let's make a core/logging/browser with a logger.ts that implements the browser-based silent logger, and use it from the two web contexts.

---

**Cursor**


Adding core/logging/browser with a browser silent logger and switching the two web components to use it.



Summary of changes:

**1. `core/logging/browser/logger.ts`**
- Defines **`silentLogger`** using **`pino/browser.js`** with `level: "silent"` (same approach as `createRemoteLogger`).
- JSDoc notes it’s for browser use so web code doesn’t depend on Node logging.

**2. `core/logging/browser/index.ts`**
- Re-exports **`silentLogger`** from `./logger.js`.

**3. `core/package.json`**
- New exports: **`"./logging/browser"`** and **`"./logging/browser/*"`** pointing at the new build files.

**4. Web components**
- **AuthDebugger.tsx**: `silentLogger` is now imported from **`@modelcontextprotocol/inspector-core/logging/browser`**.
- **OAuthCallback.tsx**: same import.

**Unchanged**
- **core/logging/node/logger.ts** still has the Node **`silentLogger`** for InspectorClient, auth, CLI, TUI, and web server config. Only the two web UI components use the browser logger.

Core and web builds complete successfully.

---

**User**

2. Missing logger name in web.ts

cli.ts passes name: "mcp-inspector-cli" to createFileLogger, but web.ts (inside runWeb()) does not:

// clients/web/src/web.ts:260
const logger = process.env.MCP_LOG_FILE
  ? await createFileLogger({
      dest: process.env.MCP_LOG_FILE,
      append: true,
      mkdir: true,
      level: "info",
      // missing: name: "mcp-inspector-web"
    })
  : silentLogger;
Log lines from the web runner won't have a name field, making it harder to distinguish sources when multiple clients log to the same file.

---

**Cursor**


Update applied in both places that create the web file logger:

1. **`clients/web/src/web.ts`** (`runWeb`) – `name: "mcp-inspector-web"`.
2. **`clients/web/src/web-server-config.ts`** (`buildWebServerConfigFromEnv` – standalone server + Vite dev) – same `name`.

Runner and server/dev paths will now emit the same `name` in shared log files.

