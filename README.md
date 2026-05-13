# Superset Windows

Automated nightly Windows builds of [Superset](https://github.com/superset-sh/superset) — The Terminal for Coding Agents.

## How it works

1. A GitHub Actions workflow runs nightly at 12:00 AM UTC
2. Checks if `superset-sh/superset` has a new release we haven't built yet
3. Clones the upstream release
4. Uses GitHub Copilot CLI to apply Windows compatibility patches from [`PATCHES.md`](PATCHES.md)
5. Builds the Electron app and NSIS installer
6. Publishes the `.exe` installer as a GitHub Release

## Download

Go to [Releases](../../releases) to download the latest Windows installer.

## Manual build

If you want to build locally instead of waiting for the nightly workflow:

```bash
# 1. Clone upstream Superset (sibling to this repo)
git clone https://github.com/superset-sh/superset.git
cd superset
git config core.longpaths true   # required: node_modules paths exceed 260 chars

# 2. Apply patches.
#    Open this superset/ checkout in Claude Code / Cursor / Copilot CLI and say:
#    "Read ../superset-windows/PATCHES.md and apply all patches to this repo"

# 3. Full build pipeline
bun install

# Rebuild better-sqlite3 against Electron's Node ABI before packaging.
# Replace 40.8.5 with the Electron version from apps/desktop/package.json.
node node_modules/.bun/@electron+rebuild@4.0.3/node_modules/@electron/rebuild/lib/cli.js \
  --version 40.8.5 --module-dir apps/desktop --only better-sqlite3 --force

cd apps/desktop
bun run generate:icons
bun run compile:app
bun run copy:native-modules
CSC_IDENTITY_AUTO_DISCOVERY=false npx electron-builder --win \
  --publish never --config electron-builder.ts
# Installer will be at apps/desktop/release/Superset-<version>-x64.exe
```

### Rebuild after tweaking

Once patches are applied, you usually only need the incremental desktop build
steps:

```powershell
cd <path>\superset
# edit files, then:
bun install                  # only if dependencies changed
cd apps\desktop
bun run compile:app
bun run copy:native-modules  # only if native deps changed
# If better-sqlite3 was refreshed or Electron changed, rebuild it from repo root first:
# node node_modules\.bun\@electron+rebuild@4.0.3\node_modules\@electron\rebuild\lib\cli.js --version 40.8.5 --module-dir apps\desktop --only better-sqlite3 --force
$env:CSC_IDENTITY_AUTO_DISCOVERY = "false"
npx electron-builder --win --publish never --config electron-builder.ts
```

Skip `bun install` and `copy:native-modules` if you only changed TypeScript
or renderer code — `compile:app` + `electron-builder` is usually enough in
that case.

See [`PATCHES.md`](PATCHES.md) for the current Windows V2 compatibility notes,
including the named-pipe terminal daemon, fd-handoff guard, PowerShell 7
selection, worktree-path handling, and recent Windows UX fixes.

## Patches

See [`PATCHES.md`](PATCHES.md) for the full list of Windows compatibility patches and their rationale.

## Requirements

- Windows 10/11 x64
- [Bun](https://bun.sh) 1.0+
- [Node.js](https://nodejs.org) 22+
- [Git](https://git-scm.com) 2.20+ with `core.longpaths=true`
- [GitHub CLI](https://cli.github.com) (`gh`)
- [Visual Studio Build Tools 2022](https://visualstudio.microsoft.com/visual-cpp-build-tools/) with the "Desktop development with C++" workload (only needed if a native module ships without a prebuilt Windows binary)
