# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Zen Browser is a Firefox-based browser built on Firefox 148.0.2, managed by the [Surfer](https://github.com/zen-browser/surfer) build tool. The codebase layers Zen-specific features on top of Firefox's source tree via patches. Two release channels exist: `release` (stable) and `twilight` (RC/feature preview).

## Initial Setup

```bash
npm ci                  # Install Node dependencies (requires Node 22)
npm run download        # Download Firefox engine via Surfer
npm run import          # Import patches + preferences into engine/
npm run bootstrap       # Set up the build environment
```

Python 3.11 and Rust 1.89 are also required (see `.python-version`, `.rust-toolchain`).

## Common Commands

| Task | Command |
|------|---------|
| Run browser | `npm run start` |
| Build | `npm run build` |
| Build UI only | `npm run build:ui` |
| Lint | `npm run lint` |
| Lint with fixes | `npm run lint:fix` |
| Run all tests | `npm run test` |
| Run single test | `python3 scripts/run_tests.py <test-subdir>` |
| Debug tests | `npm run test:dbg` |
| Rebuild Firefox prefs | `npm run ffprefs` |
| Sync Firefox version | `npm run sync` |
| Check licenses | `npm run lc` |

### Running a single test

Test paths are relative to `src/zen/tests/` (which maps to `engine/zen/tests/`):

```bash
python3 scripts/run_tests.py mochitests/compact_mode
python3 scripts/run_tests.py folders
```

Pass extra mach flags after the path:

```bash
python3 scripts/run_tests.py mochitests/glance --jsdebugger
```

## Architecture

### How Zen extends Firefox

Surfer manages a set of patches applied to the Firefox source tree. After `npm run import`, the patched Firefox source lives in `engine/`. Zen's own source files live in `src/` and are copied into the appropriate places within `engine/` at build time.

### Source layout

```
src/zen/                    # All Zen-specific JS/CSS/XUL code
  common/modules/           # Core manager classes (ZenStartup, ZenUIManager, etc.)
  spaces/                   # Workspaces (tab isolation with separate histories)
  tabs/                     # Tab management, pinned tabs, essential tabs
  folders/                  # Zen Folders (tab groups distinct from Firefox tab groups)
  split-view/               # Side-by-side tab viewing
  compact-mode/             # Reduced UI density mode
  live-folders/             # Dynamic folders from RSS/GitHub/REST APIs
  glance/                   # Quick-preview overlay
  mods/                     # User CSS/stylesheet injection system
  urlbar/                   # URL bar customizations
  kbs/                      # Keyboard shortcut bindings
  sessionstore/             # Session persistence overrides
  tests/                    # Mochitests and unit tests
  zen.globals.mjs           # Declares all global names exposed to the browser window
src/browser/                # Overrides/patches to Firefox browser UI components
configs/                    # Branding and platform build configs
scripts/                    # Python utilities (test runner, FF sync, etc.)
build/                      # Platform-specific packaging configs
```

### Global manager pattern

Zen uses named singletons (prefixed `gZen`) registered on the browser window. All globals must be listed in `src/zen/zen.globals.mjs` so ESLint recognises them. Key managers:

- `gZenStartup` — orchestrates browser initialisation order
- `gZenUIManager` — top-level UI state
- `gZenWorkspaces` / `ZenWorkspacesEngine` — workspace logic
- `gZenVerticalTabsManager` — sidebar tab strip
- `gZenFolders` — tab folder management
- `gZenViewSplitter` — split view
- `gZenCompactModeManager` — compact mode toggle
- `gZenKeyboardShortcutsManager` — keybinding system
- `gZenSessionStore` — session persistence

### Tests

Tests live under `src/zen/tests/` and are standard Firefox mochitests. The test runner script (`scripts/run_tests.py`) copies them into `engine/zen/tests/` and delegates to `./mach test`.

## Commit Message Format

Commits follow formal-git format (`.formal-git/template`):

```
{bugId}: {message}
```

Examples from recent history:
- `no-bug: Start using a different commit standard`
- `fix: Keyboard shortcut assignment issue caused by modifier, b=closes #10686, p=#12776`
- `feat: Improve accessibility for split view buttons, b=no-bug, c=split-view, tabs`

## Branch Structure

```
dev (main)  ←──  feature branches
  │
  └──→  stable  ←──  hotfixes (security patches applied directly)
```

PRs should target `dev`. The `twilight` branch tracks RC builds.

## Linting

ESLint is configured in `eslint.config.mjs` and lints only `src/zen/` (not the full Firefox engine). Pre-commit hooks (Husky + lint-staged) run automatically. The CI linter can be bypassed on a commit by including `[no-lint]` in the commit message.
