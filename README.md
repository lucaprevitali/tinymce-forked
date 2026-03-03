# WFP Rich Text Editor

A WFP-branded fork of **TinyMCE 6.7.3** — the last version released under the MIT licence.

This fork ships updated dependencies, the WFP colour scheme, and removes all Tiny Cloud promotion and branding from the default editor UI.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [First-time setup](#first-time-setup)
3. [Repository structure](#repository-structure)
4. [Build commands](#build-commands)
5. [Development server](#development-server)
6. [Running tests](#running-tests)
7. [WFP branding](#wfp-branding)
8. [Dependency management](#dependency-management)

---

## Prerequisites

| Tool | Version | Notes |
|---|---|---|
| **Node.js** | **22** | Pinned in `.nvmrc`. |
| **Yarn** | **1.22.22** | Classic Yarn (v1). Do **not** use npm or Yarn v2+. |
| **nvm** | any | Used to switch Node versions. |
| **ChromeDriver** | 145+ | Required for headless browser tests. Install globally: `npm install -g chromedriver` |

Switch to the required Node version before doing anything:

```bash
nvm use        # reads .nvmrc → 22
```

---

## First-time setup

```bash
nvm use
yarn install   # installs all workspaces; patch-package runs automatically via postinstall
```

`yarn install` is intentionally run with `--frozen-lockfile` (enforced via `.yarnrc`).  
To add or upgrade a dependency you must temporarily remove that flag — see [Dependency management](#dependency-management).

---

## Repository structure

This is a **Yarn Workspaces + Lerna 8** monorepo. Every folder under `modules/` is an `@ephox/*` package symlinked automatically into `node_modules`.

| Module | Package | Purpose |
|---|---|---|
| `tinymce` | `tinymce` | **The editor** — core, plugins, silver theme |
| `alloy` | `@ephox/alloy` | UI component framework (toolbar, menus, dialogs) |
| `bridge` | `@ephox/bridge` | Public UI API consumed by host applications |
| `oxide` | `@ephox/oxide` | LESS-based skin (compiled to CSS) |
| `oxide-icons-default` | `@ephox/oxide-icons-default` | Default SVG icon pack |
| `sugar` | `@ephox/sugar` | Typed DOM manipulation helpers |
| `katamari` | `@ephox/katamari` | Core FP data types (`Option`, `Result`, `Arr`, …) |
| `boulder` | `@ephox/boulder` | Runtime object schema validation |
| `sand` | `@ephox/sand` | Platform / device detection |
| `darwin` | `@ephox/darwin` | Selection and cursor navigation |
| `snooker` | `@ephox/snooker` | Table model (merge, split, resize) |
| `dragster` | `@ephox/dragster` | Drag-and-drop |
| `phoenix` | `@ephox/phoenix` | DOM text gathering |
| `robin` | `@ephox/robin` | Sibling node grouping by boundary points |
| `polaris` | `@ephox/polaris` | Array and string utilities |
| `porkbun` | `@ephox/porkbun` | JavaScript event framework |
| `jax` | `@ephox/jax` | AJAX library |
| `acid` | `@ephox/acid` | Colour picker (LESS + Alloy component) |
| `boss` | `@ephox/boss` | DocumentModel / TestUniverse wrapper |
| `agar` | `@ephox/agar` | Test infrastructure (async actions, input simulation) |
| `mcagar` | `@ephox/mcagar` | TinyMCE-specific Agar helpers |
| `katamari-assertions` | `@ephox/katamari-assertions` | Bedrock/Chai assertions for Katamari types |

---

## Build commands

All commands must be run from the **repository root** (not inside individual modules).

```bash
# Full development build (oxide icons → oxide CSS → TypeScript → webpack bundles)
yarn dev

# TypeScript type-check only (all packages, composite project references)
yarn tsc

# Compile oxide LESS → CSS only
yarn oxide-build

# Compile oxide in watch mode (BrowserSync on port 3000)
yarn oxide-start

# Compile oxide icons SVG pack
yarn oxide-icons-build

# Webpack bundles for tinymce only (assumes tsc already ran)
yarn tinymce-grunt dev
```

---

## Production build

Produces a fully minified, self-contained distribution in `modules/tinymce/js/tinymce/` (~8 MB):

```bash
yarn prod-build
```

This runs in sequence: `oxide-icons-build` → `oxide-build` → `tsc` → `rollup` (IIFE bundles for core, all plugins, themes, models) → `concat` (prepends license header) → `copy` (skins, icons, langs) → `terser` (minifies to `.min.js`).

Outputs:

| Path | Contents |
|---|---|
| `js/tinymce/tinymce.js` | Unminified editor core |
| `js/tinymce/tinymce.min.js` | Minified editor core (what you ship) |
| `js/tinymce/tinymce.d.ts` | TypeScript type definitions |
| `js/tinymce/plugins/*/plugin.min.js` | All plugins, minified |
| `js/tinymce/themes/silver/theme.min.js` | Silver theme, minified |
| `js/tinymce/skins/` | CSS skins (oxide, oxide-dark, …) |
| `js/tinymce/icons/default/` | Default icon pack |

The `js/` directory is symlinked from `modules/tinymce/js/` to the repo root and is already excluded by `.gitignore`.

> **Note**: `js/` is a build artifact and must **not** be committed to git.

---

## Development server

```bash
yarn start
# → http://localhost:3000/src/core/demo/html/full_demo.html
```

This starts a webpack-dev-server (via `grunt start`) that watches TypeScript sources and serves the full editor demo.  
The oxide CSS is **not** auto-recompiled by this server. If you change LESS files, run `yarn oxide-build` separately first.

webpack 5 filesystem cache is enabled — cache is stored in `modules/tinymce/node_modules/.cache/webpack/` (already in `.gitignore`).  
To force a cold rebuild: `rm -rf modules/tinymce/node_modules/.cache/webpack/`

---

## Running tests

### Standard headless test suite (Chrome headless)

This is the main test command. It runs all TinyMCE unit and browser integration tests:

```bash
yarn tinymce-grunt bedrock-auto:standard
```

Exit code `0` = all pass. Exit code `3` = at least one failure (check output for details).

Results are written to:
```
modules/tinymce/scratch/TEST-chrome-headless.xml
```

To parse failures quickly:
```bash
python3 -c "
import xml.etree.ElementTree as ET
tree = ET.parse('modules/tinymce/scratch/TEST-chrome-headless.xml')
ts = tree.getroot()
print(f'Tests: {ts.get(\"tests\")}, Failures: {ts.get(\"failures\")}')
for tc in ts.findall('testcase'):
    f = tc.find('failure')
    if f is not None:
        print('FAIL:', tc.get('name'))
"
```

### Known pre-existing flaky failures

The following **5 tests** fail intermittently due to timing or clipboard API limitations in headless Chrome. They are **not regressions** — confirmed across multiple consecutive runs against the unmodified upstream 6.7.3 codebase:

- 2 × clipboard timing tests
- 3 × selection/focus timing tests

If the test run reports only these 5 failures (or fewer), the build is healthy.

### Other test targets

```bash
# Silver theme tests only
yarn workspace @ephox/tinymce exec grunt bedrock-auto:silver

# Headless (alias, runs standard suite via root Gruntfile)
yarn headless-test

# Firefox headless
yarn headless-test-firefox

# Manual (opens browser, keeps server running for interactive inspection)
yarn headless-test-manual

# Single file (fast iteration — replace <path> with relative test file path)
yarn test-one <path>
```

### ChromeDriver requirement

Tests require ChromeDriver matching your installed Chrome version:

```bash
npm install -g chromedriver   # installs the version matching your local Chrome
chromedriver --version        # verify
```

---

## WFP branding

Both `branding` and `promotion` can still be re-enabled per-instance via the editor `init` config:

```js
tinymce.init({
  selector: '#editor',
  branding: true,    // re-enables "Powered by Tiny" logo
  promotion: true,   // re-enables upgrade link
});
```

After changing any LESS variable, rebuild the skin:

```bash
yarn oxide-build
```

---

## Dependency management

The lockfile is frozen (`--frozen-lockfile true` in `.yarnrc`). To upgrade a dependency:

```bash
# 1. Temporarily disable frozen lockfile
sed -i.bak 's/--install.frozen-lockfile true//' .yarnrc

# 2. Upgrade
yarn upgrade <package-name>

# 3. Re-enable frozen lockfile
echo '--install.frozen-lockfile true' >> .yarnrc

# 4. Verify
yarn tsc && yarn tinymce-grunt dev && yarn tinymce-grunt bedrock-auto:standard
```

### Never run `yarn install` inside a module subfolder

Always install from the root. Yarn Workspaces resolves cross-module dependencies via symlinks in the root `node_modules`; installing inside a single module breaks the graph.
