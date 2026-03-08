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
# Clone upstream
git clone https://github.com/superset-sh/superset.git
cd superset
git config core.longpaths true

# Open in Claude Code / Cursor / Copilot and say:
# "Read ../superset-windows/PATCHES.md and apply all patches to this repo"

# Then build
bun install
cd apps/desktop
bun run compile:app
CSC_IDENTITY_AUTO_DISCOVERY=false npx electron-builder --win --config electron-builder.ts
# Installer will be at apps/desktop/release/Superset-<version>-x64.exe
```

## Patches

See [`PATCHES.md`](PATCHES.md) for the full list of Windows compatibility patches and their rationale.

## Requirements

- Windows 10/11 x64
- [Bun](https://bun.sh) 1.0+
- [Git](https://git-scm.com) 2.20+
- [GitHub CLI](https://cli.github.com) (`gh`)
