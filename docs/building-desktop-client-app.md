# Building AFFiNE Desktop Client App

> **Warning**:
>
> This document is not guaranteed to be up-to-date.
> If you find any outdated information, please feel free to open an issue or submit a PR.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Development](#development)
- [Build](#build)
- [CI](#ci)

## Things you may need to know before getting started

Building the desktop client app for the moment is a bit more complicated than building the web app. The client right now is an Electron app that wraps the prebuilt web app, with parts of the native modules written in Rust, which means we have the following source modules to build a desktop client app:

1. `packages/frontend/core`: the web app
2. `packages/frontend/native`: the native modules written in Rust (mostly the sqlite bindings)
3. `packages/frontend/apps/electron`: the Electron app (containing main & helper process, and the electron entry point in `packages/frontend/apps/electron-renderer`)

#3 is dependent on #1 and #2, and relies on electron-forge to make the final app & installer. To get a deep understanding of how the desktop client app is built, you may want to read the workflow file in [release-desktop.yml](/.github/workflows/release-desktop.yml).

Due to [some limitations of Electron builder](https://github.com/yarnpkg/berry/issues/4804), you may need to have two separate yarn config for building the core and the desktop client app:

1. build frontend (with default yarn settings)
2. build electron (reinstall with hoisting off)

We will explain the steps in the following sections.

## Prerequisites

Before you start building AFFiNE Desktop Client Application, please following the same steps in [BUILDING#Prerequisites](./BUILDING.md#prerequisites) to install Node.js and Rust.

On Windows, you must enable symbolic links this code repo. See [#### Windows](./BUILDING.md#Windows).

## Build, package & make the desktop client app

> repos/AFFiNE/.github/workflows/release-desktop.yml contains real order to build the desktop client app, but here we will explain the steps in a more detailed way. Up-to date.

### 0. Build the native modules

Please refer to `Build Native Dependencies` section in [BUILDING.md](./BUILDING.md#Build-Native-Dependencies) to build the native modules.

### 1. Build the core

On Mac & Linux

```shell
BUILD_TYPE=canary yarn affine @affine/electron build

BUILD_TYPE=canary yarn affine @affine/electron generate-assets
```

On Windows (powershell)

```powershell
$env:BUILD_TYPE="canary"
$env:DISTRIBUTION="desktop"
$env:NODE_OPTIONS="--max-old-space-size=14384"
Remove-Item Env:SKIP_WEB_BUILD -ErrorAction SilentlyContinue
yarn affine @affine/electron generate-assets

# Both files must exist before continuing to the packaging install.
Test-Path packages/frontend/apps/electron/resources/web-static/index.html
Test-Path packages/frontend/apps/electron/resources/web-static/shell.html
```

Do not set `SKIP_WEB_BUILD` during this step. It tells
`generate-assets.ts` to reuse an existing renderer build, so using it before
`resources/web-static` has been populated produces an Electron window that
cannot load `assets://./` or `assets://./shell.html`.

### 2. Re-config yarn, clean up the node_modules and reinstall the dependencies

As we said before, you need to reinstall the dependencies with hoisting off. You can do this by running the following command:

```shell
yarn config set nmMode classic
yarn config set nmHoistingLimits workspaces
```

Then, clean up all node_modules and reinstall the dependencies:

On Mac & Linux

```shell
find . -name 'node_modules' -type d -prune -exec rm -rf '{}' +
yarn install
```

On Windows (powershell)

```powershell
yarn affine clean --node-modules
yarn install
```

### 3. Build the desktop client app installer

#### Mac & Linux

Note: you need to comment out `osxSign` and `osxNotarize` in `forge.config.mjs` to skip signing and notarizing the app.

```shell
BUILD_TYPE=canary SKIP_WEB_BUILD=1 HOIST_NODE_MODULES=1 yarn affine @affine/electron make
```

#### Windows

Making the windows installer is a bit different. Right now we provide two installer options: squirrel and nsis.

```powershell
$env:BUILD_TYPE="canary"
$env:DISTRIBUTION="desktop"
$env:SKIP_WEB_BUILD=1
$env:HOIST_NODE_MODULES=1
$env:NODE_OPTIONS="--max-old-space-size=14384"

# Close a previously launched canary build before replacing its output files.
Get-Process -Name "AFFiNE-canary" -ErrorAction SilentlyContinue | Stop-Process

yarn affine @affine/electron package --platform=win32 --arch=x64
yarn affine @affine/electron make-squirrel
yarn affine @affine/electron make-nsis
```

`SKIP_WEB_BUILD=1` is correct here because step 1 already generated the
renderer assets. `HOIST_NODE_MODULES=1` preserves the workspace-local
`node_modules` produced in step 2 so runtime dependencies such as
`electron-updater` are included in the package.

Once the build is complete, you can find the paths to the binaries in the terminal output.

```
Finished 2 bundles at:
  › Artifacts available at: <affine-repo>/packages/frontend/apps/electron/out/canary/make
```

The unpacked Windows executable is written below `out/canary`, for example:

```text
packages/frontend/apps/electron/out/canary/AFFiNE-canary-win32-x64/AFFiNE-canary.exe
```

The installer artifacts are written below `out/canary/make`.

### Restore the development dependency layout

Workspace-local hoisting is needed for packaging but can prevent development
tools such as `swc-loader` from resolving from other workspaces. Restore the
repository defaults before running `yarn dev` again:

```powershell
yarn config set nmMode hardlinks-local
yarn config unset nmHoistingLimits
yarn affine clean --node-modules
yarn install
```

You do not need to create `dist`, `resources/web-static`, or `out` manually;
the build and packaging commands create them.

## CI

Please refer to `.github/workflows/release-desktop.yml` and
`.github/workflows/release-desktop-platform.yml` for the CI workflow. They
will:

- build the app for all supported platforms
- upload the artifacts to GitHub Actions
