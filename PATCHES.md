# Windows Patches for Superset

## Instructions for AI Agent

You are patching a fresh clone of [superset-sh/superset](https://github.com/superset-sh/superset) for Windows compatibility. Apply each patch in order. **Read the target file first**, then make the described edit. If upstream has changed the code since these instructions were written, **adapt the patch to fit** — the intent and rationale for each patch are described so you can make intelligent adjustments.

After all patches are applied, run:
```bash
bun install
cd apps/desktop
bun run generate:icons
bun run compile:app
bun run copy:native-modules
CSC_IDENTITY_AUTO_DISCOVERY=false npx electron-builder --win --publish never --config electron-builder.ts
```

The installer will be at `apps/desktop/release/Superset-<version>-x64.exe`.

---

## Patch 1: Cross-platform postinstall script

**Why:** The default `postinstall.sh` is bash-only and fails on Windows.

**File: `package.json` (root)**
Find the `"postinstall"` script line. Change its value from the bash script (e.g. `"./scripts/postinstall.sh"`) to:
```json
"postinstall": "node scripts/postinstall.mjs"
```

**New file: `scripts/postinstall.mjs`**
Create this file with the following contents:
```javascript
/**
 * Cross-platform postinstall script.
 *
 * Replaces the bash-only postinstall.sh so that `bun install` works on
 * Windows, macOS and Linux without special flags.
 *
 * Steps:
 *  1. Guard against infinite recursion (electron-builder install-app-deps
 *     can trigger nested bun installs which would re-run this script).
 *  2. Run sherif for workspace validation.
 *  3. Install native dependencies for the desktop app.
 */

import { execSync } from "node:child_process";

// Prevent infinite recursion during postinstall
if (process.env.SUPERSET_POSTINSTALL_RUNNING) {
	process.exit(0);
}
process.env.SUPERSET_POSTINSTALL_RUNNING = "1";

const env = { ...process.env, SUPERSET_POSTINSTALL_RUNNING: "1" };

/** Run a command, inheriting stdio so output is visible. */
function run(cmd) {
	execSync(cmd, { stdio: "inherit", env });
}

/** Run a command but don't fail if it errors (for optional native deps on Windows). */
function tryRun(cmd, label) {
	try {
		execSync(cmd, { stdio: "inherit", env });
	} catch {
		console.warn(`[postinstall] ${label} failed (non-fatal on Windows) — continuing`);
	}
}

// Run sherif for workspace validation
run("sherif");

// Install native dependencies for desktop app.
// On Windows, native module compilation may fail if Visual Studio Build Tools
// are not installed. This is non-fatal — prebuilt binaries will be used when available.
if (process.platform === "win32") {
	tryRun("bun run --filter=@superset/desktop install:deps", "install:deps");
} else {
	run("bun run --filter=@superset/desktop install:deps");
}
```

---

## Patch 2: Fix TDZ in Rollup banner + strip crossorigin + defineEnv fix

**Why:** Three related Vite/Rollup fixes:
1. The ELECTRON_RUN_AS_NODE banner must use `globalThis.process` to avoid a Temporal Dead Zone error in chunks that declare `const process = require("node:process")`.
2. Vite's `crossorigin` attribute on script/link tags breaks Electron's ASAR file:// handler on Windows.
3. `defineEnv` should use `||` instead of `??` so empty strings from unresolved CI secrets fall back to defaults.

**File: `apps/desktop/electron.vite.config.ts`**

1. Add `stripCrossOriginPlugin` to the imports from `./vite/helpers`:
   ```typescript
   import {
     defineEnv,
     devPath,
     htmlEnvTransformPlugin,
     stripCrossOriginPlugin,  // ADD THIS
   } from "./vite/helpers";
   ```

2. Find the `output:` object inside the main process `rollupOptions`. Add a `banner` property inside it:
   ```typescript
   output: {
     dir: resolve(devPath, "main"),
     // VS Code and other Electron hosts set ELECTRON_RUN_AS_NODE=1 which
     // prevents Electron from entering browser mode. Clear it before any
     // require("electron") call — must be the very first statement.
     banner:
       'delete globalThis.process.env.ELECTRON_RUN_AS_NODE;',
   },
   ```

3. In the renderer `plugins` array, add `stripCrossOriginPlugin()` after `htmlEnvTransformPlugin()`:
   ```typescript
   reactPlugin(),
   htmlEnvTransformPlugin(),
   stripCrossOriginPlugin(),  // ADD THIS
   ```

**File: `apps/desktop/vite/helpers.ts`**

1. In the `defineEnv` function, change `value ?? fallback` to `value || fallback`:
   ```typescript
   return JSON.stringify(value || fallback);
   ```

2. Add the `stripCrossOriginPlugin` function after the `copyResourcesPlugin` function:
   ```typescript
   /**
    * Strips the `crossorigin` attribute that Vite adds to script/link tags.
    * Electron's ASAR file:// handler doesn't support CORS on Windows,
    * so crossorigin causes scripts/styles to silently fail to load (black screen).
    */
   export function stripCrossOriginPlugin(): Plugin {
     return {
       name: "strip-crossorigin",
       transformIndexHtml: {
         order: "post",
         handler(html) {
           if (process.platform !== "win32") return html;
           return html.replace(/ crossorigin(?:="[^"]*")?/g, "");
         },
       },
     };
   }
   ```

---

## Patch 3: Disable GPU hardware acceleration on Windows

**Why:** Prevents black/blank screens caused by GPU driver incompatibilities with Chromium's compositor.

**File: `apps/desktop/src/lib/electron-app/factories/app/setup.ts`**

Find the line that disables hardware acceleration for Linux:
```typescript
PLATFORM.IS_LINUX && app.disableHardwareAcceleration();
```

Change it to also include Windows:
```typescript
// Disable GPU hardware acceleration on Linux and Windows to prevent black/blank
// screens caused by GPU driver incompatibilities with Chromium's compositor.
(PLATFORM.IS_LINUX || PLATFORM.IS_WINDOWS) && app.disableHardwareAcceleration();
```

---

## Patch 4: Fix Windows junction removal in copy-native-modules

**Why:** Windows uses NTFS junctions (directory-like) instead of symlinks. `rmSync` needs `{ recursive: true }` to remove them.

**File: `apps/desktop/scripts/copy-native-modules.ts`**

Find the symlink removal line inside the `copyModuleIfSymlink` function:
```typescript
rmSync(modulePath);
```

Replace with:
```typescript
// Windows uses junctions (directory-like) instead of symlinks;
// rmSync needs { recursive: true } to remove them.
if (process.platform === "win32") {
  rmSync(modulePath, { recursive: true, force: true });
} else {
  rmSync(modulePath);
}
```

---

## Patch 5: Custom protocol + CORS bypass for Windows

**Why:** `file://` protocol breaks ES module dynamic imports (code-split chunks) on Windows. A custom `superset-app://` protocol serves renderer files properly. CORS bypass headers are needed because the API server doesn't recognize the custom protocol origin.

**File: `apps/desktop/src/lib/window-loader.ts`**

Find the production branch that loads from file (the `else` clause after the development URL branch). Add a Windows-specific case before it:

```typescript
} else if (process.platform === "win32") {
  // Production (Windows): use custom protocol for proper dynamic import support.
  // file:// protocol breaks ES module dynamic imports (code-split chunks) on Windows.
  const url = "superset-app://app/index.html#/";
  console.log("[window-loader] Loading custom protocol URL:", url);
  props.browserWindow.loadURL(url);
} else {
  // Production (macOS/Linux): load from file with hash routing
```

**File: `apps/desktop/src/main/index.ts`**

1. Find the `protocol.registerSchemesAsPrivileged([...])` call. Add a new scheme entry to the array:
   ```typescript
   {
     scheme: "superset-app",
     privileges: {
       standard: true,
       secure: true,
       supportFetchAPI: true,
       corsEnabled: true,
     },
   },
   ```

2. Inside the `app.whenReady()` callback, after the `superset-icon` protocol handler registration, add the following block:
   ```typescript
   // Register custom protocol for serving renderer files.
   // Dynamic imports (code-split chunks) fail on file:// protocol in Electron on Windows.
   const rendererDir = path.join(__dirname, "../renderer");
   const appProtocolHandler = (request: Request) => {
     let urlPath = new URL(request.url).pathname;
     if (urlPath.startsWith("/")) urlPath = urlPath.slice(1);
     const filePath = path.join(rendererDir, urlPath);
     return net.fetch(pathToFileURL(filePath).toString());
   };
   protocol.handle("superset-app", appProtocolHandler);
   session
     .fromPartition("persist:superset")
     .protocol.handle("superset-app", appProtocolHandler);

   // On Windows, the custom superset-app:// protocol origin is not recognized by
   // the API server's CORS policy. Bypass CORS for API requests by modifying headers.
   if (PLATFORM.IS_WINDOWS) {
     const appSession = session.fromPartition("persist:superset");
     appSession.webRequest.onBeforeSendHeaders(
       { urls: ["https://api.superset.sh/*", "https://*.posthog.com/*", "https://*.sentry.io/*", "https://app.outlit.ai/*"] },
       (details, callback) => {
         if (details.requestHeaders.Origin === "superset-app://app") {
           delete details.requestHeaders.Origin;
         }
         callback({ requestHeaders: details.requestHeaders });
       },
     );
     appSession.webRequest.onHeadersReceived(
       { urls: ["https://api.superset.sh/*"] },
       (details, callback) => {
         const headers = details.responseHeaders ?? {};
         headers["access-control-allow-origin"] = ["superset-app://app"];
         headers["access-control-allow-credentials"] = ["true"];
         callback({ responseHeaders: headers });
       },
     );
   }
   ```

   Make sure `net` and `pathToFileURL` are imported. `net` comes from `electron`, `pathToFileURL` from `node:url`. Check the existing imports and add if missing.

---

## Patch 6: Feature flag default to prevent infinite render block

**Why:** When PostHog is not configured (no key), feature flags stay `undefined` forever. The app blocks rendering waiting for them, causing a permanent blank/white screen.

**File: `apps/desktop/src/renderer/routes/_authenticated/providers/CollectionsProvider/CollectionsProvider.tsx`**

Find the block that returns `null` when the feature flag is undefined. It looks like this (may include an `env.SKIP_ENV_VALIDATION` guard):
```typescript
if (useElectricCloud === undefined && !env.SKIP_ENV_VALIDATION) {
  return null;
}
```

**Delete that entire `if` block** (remove all 3 lines). Then, immediately before the `setElectricUrl(...)` call, add:
```typescript
// When PostHog is not configured (no key), feature flags stay undefined forever.
// Default to false (use proxy) so the app doesn't block rendering.
const isElectricCloud = useElectricCloud ?? false;
```

Then change the `setElectricUrl` call to use `isElectricCloud` instead of `useElectricCloud`:
```typescript
setElectricUrl(
  isElectricCloud
    ? env.NEXT_PUBLIC_ELECTRIC_URL
    : env.NEXT_PUBLIC_ELECTRIC_PROXY_URL,
);
```

---

## Patch 7: Forward renderer console messages on Windows

**Why:** On Windows, renderer warnings/errors aren't visible in the terminal. This forwards them to stdout for debugging.

**File: `apps/desktop/src/main/windows/main.ts`**

Find the `MainWindow()` function. After the existing event handlers (look for window bounds persistence or similar), add:

```typescript
// Forward renderer warning/error messages to main process stdout for Windows debugging.
if (PLATFORM.IS_WINDOWS) {
  window.webContents.on(
    "console-message",
    (_event, level, message, line, sourceId) => {
      if (level < 2) return;
      const levelStr =
        ["verbose", "info", "warning", "error"][level] ?? "unknown";
      const source = sourceId ? ` (${sourceId}:${line})` : "";
      const formatted = `[renderer:${levelStr}] ${message}${source}`;
      if (level === 3) console.error(formatted);
      else console.warn(formatted);
    },
  );
}
```

Make sure `PLATFORM` is imported from `shared/constants` (it likely already is).

---

## Patch 8: Terminal named pipes for Windows

**Why:** Windows doesn't support Unix domain sockets. The terminal host daemon must use Windows named pipes (`\\.\pipe\superset-terminal-host-<user>`) instead.

**New file: `apps/desktop/src/main/lib/terminal-host/paths.ts`**

Create this file:
```typescript
import { homedir } from "node:os";
import { join } from "node:path";
import { SUPERSET_DIR_NAME } from "shared/constants";

const IS_WINDOWS = process.platform === "win32";

const SUPERSET_HOME_DIR = join(homedir(), SUPERSET_DIR_NAME);

const PIPE_SUFFIX = (
  process.env.USERNAME ?? process.env.USER ?? "user"
).replace(/[^a-zA-Z0-9_.-]/g, "_");

const SOCKET_PATH = IS_WINDOWS
  ? `\\\\.\\pipe\\superset-terminal-host-${PIPE_SUFFIX}`
  : join(SUPERSET_HOME_DIR, "terminal-host.sock");

export const TERMINAL_HOST_PATHS = {
  IS_WINDOWS,
  SUPERSET_DIR_NAME,
  SUPERSET_HOME_DIR,
  SOCKET_PATH,
  TOKEN_PATH: join(SUPERSET_HOME_DIR, "terminal-host.token"),
  PID_PATH: join(SUPERSET_HOME_DIR, "terminal-host.pid"),
  SPAWN_LOCK_PATH: join(SUPERSET_HOME_DIR, "terminal-host.spawn.lock"),
  SCRIPT_MTIME_PATH: join(SUPERSET_HOME_DIR, "terminal-host.mtime"),
};
```

**File: `apps/desktop/src/main/lib/terminal-host/client.ts`**

1. Remove `import { homedir } from "node:os";` (if `homedir` is no longer used elsewhere in the file).
2. Replace the import of `SUPERSET_DIR_NAME` from `shared/constants` with:
   ```typescript
   import { TERMINAL_HOST_PATHS } from "./paths";
   ```
3. Replace the hardcoded path constants block (SUPERSET_HOME_DIR, SOCKET_PATH, TOKEN_PATH, PID_PATH, SPAWN_LOCK_PATH, SCRIPT_MTIME_PATH) with:
   ```typescript
   const {
     IS_WINDOWS,
     SUPERSET_DIR_NAME,
     SUPERSET_HOME_DIR,
     SOCKET_PATH,
     TOKEN_PATH,
     PID_PATH,
     SPAWN_LOCK_PATH,
     SCRIPT_MTIME_PATH,
   } = TERMINAL_HOST_PATHS;
   ```
4. Find every `!existsSync(SOCKET_PATH)` check and prepend `!IS_WINDOWS &&`:
   ```typescript
   if (!IS_WINDOWS && !existsSync(SOCKET_PATH)) {
   ```
   There should be approximately 5 occurrences.
5. Find `if (existsSync(SOCKET_PATH))` in the `spawnDaemon` method. Change to:
   ```typescript
   if (IS_WINDOWS || existsSync(SOCKET_PATH)) {
   ```
6. Wrap the stale socket `unlinkSync(SOCKET_PATH)` block in `!IS_WINDOWS`:
   ```typescript
   if (!IS_WINDOWS) {
     if (DEBUG_CLIENT) {
       console.log("[TerminalHostClient] Removing stale socket file");
     }
     try {
       unlinkSync(SOCKET_PATH);
     } catch {
       // Ignore
     }
   }
   ```
7. In `waitForDaemon`, replace `if (existsSync(SOCKET_PATH))` with:
   ```typescript
   const live = await this.isSocketLive();
   if (live) {
   ```

**File: `apps/desktop/src/main/terminal-host/index.ts`**

1. Remove `import { homedir } from "node:os";` and the `join` import from `node:path` (if no longer used elsewhere).
2. Replace the import of `SUPERSET_DIR_NAME` with:
   ```typescript
   import { TERMINAL_HOST_PATHS } from "../lib/terminal-host/paths";
   ```
3. Replace hardcoded path constants with:
   ```typescript
   const {
     IS_WINDOWS,
     SUPERSET_HOME_DIR,
     SOCKET_PATH,
     TOKEN_PATH,
     PID_PATH,
   } = TERMINAL_HOST_PATHS;
   ```
4. In `isSocketLive()`, add `!IS_WINDOWS &&` before `!existsSync(SOCKET_PATH)`.
5. In `startServer()`, change `if (existsSync(SOCKET_PATH))` to `if (IS_WINDOWS || existsSync(SOCKET_PATH))`.
6. Wrap the stale socket `unlinkSync` in `startServer` with `if (!IS_WINDOWS)`.

**File: `apps/desktop/src/main/lib/terminal/dev-reset.ts`**

1. Add: `import { TERMINAL_HOST_PATHS } from "main/lib/terminal-host/paths";`
2. Remove `"terminal-host.sock"` from the `TERMINAL_STATE_PATHS` array.
3. Add after the array:
   ```typescript
   const TERMINAL_STATE_PATHS_WITH_SOCKET = TERMINAL_HOST_PATHS.IS_WINDOWS
     ? TERMINAL_STATE_PATHS
     : (["terminal-host.sock", ...TERMINAL_STATE_PATHS] as const);
   ```
4. In the cleanup loop, change `TERMINAL_STATE_PATHS` to `TERMINAL_STATE_PATHS_WITH_SOCKET`.

---

## Patch 9: Fix double-nested dist/main path for daemon and worker scripts

**Why:** When running `electron dist/main/index.js` directly, `app.getAppPath()` returns `dist/main/`. Joining with `dist/main/terminal-host.js` produces `dist/main/dist/main/terminal-host.js`.

**File: `apps/desktop/src/main/lib/terminal-host/client.ts`**

Find `getDaemonScriptPath()`. Replace it with:
```typescript
private getDaemonScriptPath(): string {
  const appPath = app.getAppPath();
  // When running `electron dist/main/index.js` directly, appPath is already
  // the dist/main directory. Check for the script there first to avoid
  // double-nesting (dist/main/dist/main/terminal-host.js).
  const direct = join(appPath, "terminal-host.js");
  if (existsSync(direct)) {
    return direct;
  }
  // Packaged app or running from project root
  return join(appPath, "dist", "main", "terminal-host.js");
}
```

**File: `apps/desktop/src/lib/trpc/routers/changes/workers/git-task-runner.ts`**

Find `getWorkerScriptPath()`. Inside the `try` block, after `const appPath = ...`, add:
```typescript
// When running `electron dist/main/index.js` directly, appPath is already
// the dist/main directory. Check for the script there first to avoid
// double-nesting (dist/main/dist/main/git-task-worker.js).
const { existsSync } = require("node:fs") as typeof import("node:fs");
const direct = join(appPath, "git-task-worker.js");
if (existsSync(direct)) {
  return direct;
}
```

---

## Patch 10: Switch node-pty to @lydell/node-pty for Windows

**Why:** `@lydell/node-pty` provides prebuilt Windows native binaries (conpty.node) that don't require Visual Studio Build Tools.

**File: `apps/desktop/package.json`**

Find the `"node-pty"` dependency line. Change it from whatever version it is to:
```json
"node-pty": "npm:@lydell/node-pty@^1.0.1",
```

---

## Patch 11: Add missing Windows env vars to PTY allowlist

**Why:** Without `SYSTEMDRIVE` in the terminal environment, tools like .NET/NuGet that reference `%SystemDrive%` create files in a literal `%SystemDrive%` directory.

**File: `apps/desktop/src/main/lib/terminal/env.ts`**

Find the `ALLOWED_ENV_VARS` set. In the Windows-specific section (look for `COMSPEC`, `USERPROFILE`, etc.), add these entries:
```typescript
"PROGRAMDATA",
"SYSTEMDRIVE",
```
after `"PROGRAMFILES(X86)"`.

Also add after `"PATHEXT"`:
```typescript
"NUMBER_OF_PROCESSORS", // Used by MSBuild for parallel builds
"PROCESSOR_ARCHITECTURE", // Used by native toolchains (x86/AMD64/ARM64)
```

---

## Patch 12: Windows NSIS installer configuration

**Why:** Configures electron-builder for Windows with proper icons, NSIS installer options, native module handling, and code signing toggle.

**File: `apps/desktop/electron-builder.ts`**

1. Near the top, after the `productName` declaration, add:
   ```typescript
   const disableWinSigning = process.env.SUPERSET_DISABLE_WIN_SIGNING === "1";
   ```

2. In `extraResources`, add an entry for icons:
   ```typescript
   {
     from: join(pkg.resources, "build/icons"),
     to: "build/icons",
     filter: ["**/*"],
   },
   ```

3. Replace the `files` array. Change:
   ```typescript
   files: [
     "dist/**/*",
     "package.json",
     {
       from: pkg.resources,
       to: "resources",
       filter: ["**/*"],
     },
   ```
   To:
   ```typescript
   files: [
     {
       filter: ["dist/**/*", "!dist/resources/migrations/**", "package.json"],
     },
     {
       from: pkg.resources,
       to: "resources",
       filter: ["**/*", "!build/**"],
     },
   ```

4. Change `npmRebuild: true` to:
   ```typescript
   npmRebuild: process.platform !== "win32",
   ```

5. In the `win` section, add after `artifactName`:
   ```typescript
   asarUnpack: ["**/node_modules/@lydell/node-pty-win32-x64/**/*"],
   files: [
     {
       from: "node_modules/@lydell/node-pty-win32-x64",
       to: "node_modules/@lydell/node-pty-win32-x64",
       filter: ["**/*"],
     },
   ],
   ```

6. In the `nsis` section, add:
   ```typescript
   createDesktopShortcut: true,
   createStartMenuShortcut: true,
   shortcutName: productName,
   installerIcon: join(pkg.resources, "build/icons/icon.ico"),
   uninstallerIcon: join(pkg.resources, "build/icons/icon.ico"),
   ```

**File: `apps/desktop/package.json`**

Find the `"build"` script. Add `--config electron-builder.ts` to the electron-builder command:
```json
"build": "cross-env CSC_IDENTITY_AUTO_DISCOVERY=false electron-builder --publish never --config electron-builder.ts",
```

---

## Patch 13: Windows app resolution for "Open in" actions

**Why:** The "Open in VS Code/Terminal/Finder" feature only handles macOS and Linux. Windows needs exe path auto-detection.

**File: `apps/desktop/src/lib/trpc/routers/external/helpers.ts`**

1. Add `import fs from "node:fs";` at the top (alongside existing imports).

2. After the `LINUX_CLI_CANDIDATES` object, add the entire Windows app resolution system. This includes:
   - A `WindowsAppConfig` type
   - A `WINDOWS_APP_CONFIG` record mapping every `ExternalApp` to Windows executable info (cli name, exe names, install directories, JetBrains exe names, custom args)
   - Helper functions: `resolveTerminalTarget`, `getWindowsProgramRoots`, `findExistingPath`, `buildWindowsExeCandidates`, `findJetBrainsExeInRoot`, `findJetBrainsToolboxExe`, `findJetBrainsExe`
   - A `getWindowsAppCommand` function that tries: full exe path → JetBrains resolution → CLI fallback

   Key Windows app configs:
   - **vscode**: cli `code`, exe `Code.exe`, install dir `Microsoft VS Code`
   - **cursor**: cli `cursor`, exe `Cursor.exe`, install dir `Cursor`
   - **terminal**: cli `wt`, exe `wt.exe`/`WindowsTerminal.exe`, args `["-d", targetDir]`
   - **JetBrains IDEs**: search `Program Files/JetBrains/<product>/bin/<exe>` and Toolbox paths
   - **macOS-only apps** (xcode, iterm, appcode): empty config `{}`

3. In `getAppCommand()`, add a `win32` check at the top before the `darwin` check:
   ```typescript
   if (platform === "win32") {
     return getWindowsAppCommand(app, targetPath);
   }
   ```

---

## Patch 14: Materialize @lydell/node-pty platform binary from Bun store

**Why:** `@lydell/node-pty` loads its native binary via `require("@lydell/node-pty-win32-x64/conpty.node")`. Bun keeps optional dependencies in its internal `.bun/` store, so they're not resolvable from the desktop workspace's `node_modules`. Without this, PTY spawn fails with "PTY not spawned".

**File: `apps/desktop/scripts/copy-native-modules.ts`**

1. After the `NATIVE_MODULE_DEPS` array, add a new array for platform-specific optional modules:
   ```typescript
   // Platform-specific optional native packages that must be materialized from Bun's store.
   // @lydell/node-pty uses optionalDependencies for platform binaries, but Bun keeps them
   // in .bun/ and they aren't resolvable from the desktop workspace without explicit copying.
   const OPTIONAL_PLATFORM_MODULES = [
     ...(process.platform === "win32" ? ["@lydell/node-pty-win32-x64"] : []),
     ...(process.platform === "darwin" && process.arch === "arm64" ? ["@lydell/node-pty-darwin-arm64"] : []),
     ...(process.platform === "darwin" && process.arch === "x64" ? ["@lydell/node-pty-darwin-x64"] : []),
     ...(process.platform === "linux" && process.arch === "x64" ? ["@lydell/node-pty-linux-x64"] : []),
     ...(process.platform === "linux" && process.arch === "arm64" ? ["@lydell/node-pty-linux-arm64"] : []),
   ] as const;
   ```

2. In the `prepareNativeModules()` function, before `console.log("\nDone!");`, add a block to copy these platform modules from Bun's store:
   ```typescript
   if (OPTIONAL_PLATFORM_MODULES.length > 0) {
     console.log("\nPreparing platform-specific optional modules...");
     const bunStoreDir = getBunStoreDir(nodeModulesDir);
     for (const moduleName of OPTIONAL_PLATFORM_MODULES) {
       const destPath = join(nodeModulesDir, moduleName);
       if (existsSync(destPath)) {
         console.log(`  ${moduleName}: already exists`);
         continue;
       }
       // Search Bun store for the package
       const bunPrefix = moduleName.startsWith("@")
         ? moduleName.replace("/", "+")
         : moduleName;
       const bunStoreEntries = existsSync(bunStoreDir)
         ? readdirSync(bunStoreDir).filter((e) => e.startsWith(`${bunPrefix}@`))
         : [];
       if (bunStoreEntries.length === 0) {
         console.warn(`  ${moduleName}: not found in Bun store (skipping)`);
         continue;
       }
       const sourcePath = join(
         bunStoreDir,
         bunStoreEntries.sort().reverse()[0],
         "node_modules",
         moduleName,
       );
       if (!existsSync(sourcePath)) {
         console.warn(`  ${moduleName}: Bun store path missing (${sourcePath})`);
         continue;
       }
       console.log(`  ${moduleName}: copying from Bun store`);
       mkdirSync(dirname(destPath), { recursive: true });
       cpSync(sourcePath, destPath, { recursive: true });
     }
   }
   ```

   Note: `getBunStoreDir`, `mkdirSync`, `dirname`, `readdirSync`, `cpSync`, `existsSync` should already be imported/available in this file. Verify before adding.

---

## Patch 15: Use \r instead of \n for terminal command execution on Windows

**Why:** Windows ConPTY expects `\r` (carriage return) to trigger command execution, not `\n` (linefeed). Without this, agent launch commands are typed into the terminal but not executed — the user has to manually press Enter.

**File: `apps/desktop/src/renderer/lib/terminal/launch-command.ts`**

Find the `normalizeTerminalCommand` function:
```typescript
function normalizeTerminalCommand(command: string): string {
  return command.endsWith("\n") ? command : `${command}\n`;
}
```

Replace with:
```typescript
function normalizeTerminalCommand(command: string): string {
  // Windows ConPTY expects \r (carriage return) to execute a command,
  // while Unix terminals use \n (newline). Use \r for cross-platform compat
  // as most Unix terminal emulators also accept \r.
  const eol = "\r";
  return command.endsWith("\n") || command.endsWith("\r")
    ? command
    : `${command}${eol}`;
}
```

---

## Patch 16: Fix sound playback on Windows (MP3 support)

**Why:** The Windows sound implementation uses `System.Media.SoundPlayer` which **only supports WAV format**. All ringtones are MP3 files, so playback silently fails. The fix switches to `System.Windows.Media.MediaPlayer` (WPF/PresentationCore) which supports MP3. This affects both the notification sound and the ringtone preview.

**File: `apps/desktop/src/main/lib/notification-sound.ts`**

Find the Windows branch inside `playSoundFile`:
```typescript
} else if (process.platform === "win32") {
  execFile("powershell", [
    "-c",
    `(New-Object Media.SoundPlayer '${soundPath}').PlaySync()`,
  ]);
}
```

Replace with:
```typescript
} else if (process.platform === "win32") {
  // System.Media.SoundPlayer only supports WAV. Use WPF MediaPlayer for MP3 support.
  const psScript = `
Add-Type -AssemblyName PresentationCore
$p = New-Object System.Windows.Media.MediaPlayer
$p.Open([Uri]::new("${soundPath.replace(/\\/g, "/")}"))
$p.Play()
Start-Sleep -Milliseconds 100
while ($p.Position -lt $p.NaturalDuration.TimeSpan) { Start-Sleep -Milliseconds 200 }
$p.Close()
`.trim();
  execFile("powershell", ["-NoProfile", "-c", psScript]);
}
```

**File: `apps/desktop/src/lib/trpc/routers/ringtone/index.ts`**

Find the Windows branch inside `playSoundFile`:
```typescript
} else if (process.platform === "win32") {
  currentSession.process = execFile(
    "powershell",
    ["-c", `(New-Object Media.SoundPlayer '${soundPath}').PlaySync()`],
    () => {
      if (currentSession?.id === sessionId) {
        currentSession = null;
      }
    },
  );
}
```

Replace with:
```typescript
} else if (process.platform === "win32") {
  // System.Media.SoundPlayer only supports WAV. Use WPF MediaPlayer for MP3 support.
  const psScript = `
Add-Type -AssemblyName PresentationCore
$p = New-Object System.Windows.Media.MediaPlayer
$p.Open([Uri]::new("${soundPath.replace(/\\/g, "/")}"))
$p.Play()
Start-Sleep -Milliseconds 100
while ($p.Position -lt $p.NaturalDuration.TimeSpan) { Start-Sleep -Milliseconds 200 }
$p.Close()
`.trim();
  currentSession.process = execFile(
    "powershell",
    ["-NoProfile", "-c", psScript],
    () => {
      if (currentSession?.id === sessionId) {
        currentSession = null;
      }
    },
  );
}
```

---

## Patch 17: Fix workspace switching shortcut display on Windows

**Why:** The sidebar tooltip hardcodes the `⌘` symbol on all platforms. On Windows/Linux the actual keybinding is `Ctrl+Shift+1-9` (correctly derived by `deriveNonMacDefault`), but the UI still shows `⌘1`. The display should match the actual shortcut.

**File: `apps/desktop/src/renderer/screens/main/components/WorkspaceSidebar/WorkspaceListItem/WorkspaceListItem.tsx`**

1. Add import for the hotkeys store at the top of the file:
   ```typescript
   import { useHotkeysStore } from "renderer/stores/hotkeys";
   ```

2. Inside the `WorkspaceListItem` component function body (near the top where other hooks are called), add:
   ```typescript
   const platform = useHotkeysStore((state) => state.platform);
   ```

3. Find the hardcoded shortcut display:
   ```tsx
   <span className="text-[10px] text-muted-foreground font-mono tabular-nums shrink-0">
     ⌘{shortcutIndex + 1}
   </span>
   ```

   Replace with:
   ```tsx
   <span className="text-[10px] text-muted-foreground font-mono tabular-nums shrink-0">
     {platform === "darwin" ? "⌘" : "Ctrl+Shift+"}{shortcutIndex + 1}
   </span>
   ```

---

## Patch 18: Windows Ctrl+C (copy) and Ctrl+V (paste) in terminal

**Why:** xterm.js is a terminal emulator, so it follows Unix terminal conventions:
- **Ctrl+C** always sends SIGINT (interrupt) to the process, even when text is selected. On Windows, users expect Ctrl+C to copy selected text (and only interrupt when nothing is selected).
- **Ctrl+V** sends the literal `\x16` character (ASCII SYN) to the terminal instead of pasting. On Windows, users expect Ctrl+V to paste from clipboard. (Ctrl+Shift+V works because xterm.js has built-in support for that as the "terminal paste" shortcut.)

**File: `apps/desktop/src/renderer/screens/main/components/WorkspaceView/ContentView/TabsContent/Terminal/helpers.ts`**

Find the `setupKeyboardHandler()` function. Inside the `handler` function, find the line:
```typescript
if (isTerminalReservedEvent(event)) return true;
```

**Immediately before** that line (after the Ctrl+Right word navigation block), add the following two blocks:

```typescript
// Windows: Ctrl+C copies selected text to clipboard; if nothing is
// selected, fall through to send the normal interrupt signal.
if (
  isWindows &&
  event.type === "keydown" &&
  event.ctrlKey &&
  !event.shiftKey &&
  !event.altKey &&
  event.key === "c"
) {
  if (xterm.hasSelection()) {
    navigator.clipboard.writeText(xterm.getSelection());
    xterm.clearSelection();
    return false; // Copied — don't send interrupt
  }
  // No selection — fall through to terminal reserved handler (interrupt)
}

// Windows: Ctrl+V pastes from clipboard.
// xterm.js defaults to sending \x16 for Ctrl+V (Unix "quoted insert").
// Intercept it and write clipboard text to the PTY instead.
if (
  isWindows &&
  event.type === "keydown" &&
  event.ctrlKey &&
  !event.shiftKey &&
  !event.altKey &&
  event.key === "v"
) {
  navigator.clipboard.readText().then((text) => {
    if (text && options.onWrite) {
      options.onWrite(text.replace(/\r?\n/g, "\r"));
    }
  });
  return false; // Don't send \x16 to terminal
}
```

The result should look like:
```typescript
    // ... Ctrl+Right word navigation block above ...

    // Windows: Ctrl+C copies selected text ...
    if (isWindows && ...) { ... }

    // Windows: Ctrl+V pastes from clipboard ...
    if (isWindows && ...) { ... }

    if (isTerminalReservedEvent(event)) return true;
```

---

## Verification Checklist

After applying all patches, verify:
- [ ] `bun install` completes (bufferutil warning is expected and non-fatal)
- [ ] `bun run compile:app` builds with 0 TypeScript errors
- [ ] `electron dist/main/index.js` launches without TDZ or path errors
- [ ] Terminal opens without "Connection lost" or "PTY not spawned" errors
- [ ] Agent launch (click Claude/Codex) auto-executes the command (no manual Enter needed)
- [ ] Changes panel loads (no "Unable to load changes")
- [ ] "Open in VS Code" works
- [ ] Notification sounds and ringtone preview play correctly on Windows
- [ ] Ctrl+Shift+1/2/3 switches workspaces on Windows (⌘+1/2/3 on macOS)
- [ ] Sidebar tooltip shows "Ctrl+Shift+1" on Windows instead of "⌘1"
- [ ] Ctrl+C copies selected text in terminal, sends interrupt when nothing selected (Windows)
- [ ] Ctrl+V pastes from clipboard in terminal (Windows)
- [ ] NSIS installer builds successfully
