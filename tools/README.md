# `tools/` — local dev tools

Node ESM scripts (no deps) for running a Windows build locally instead of
waiting on/consuming a GitHub Actions run.

## local builds

- **[`build-windows-local.js`](build-windows-local.js)** -
  Windows only. Mirrors `build.yml`'s `build-windows` job step-for-step,
  including the real `BUILD.gn` source patches that job needs (`v8_monolith`
  is a `static_library`, which Chromium's automatic libc++ dependency
  injection never covers - without these, final link fails with dozens of
  real `LNK2019`/`LNK2001`s for libc++ runtime symbols). Requires Visual
  Studio 2026 (or compatible) with the "Desktop development with C++"
  workload already installed - nothing here installs Visual Studio itself,
  too heavy/varied to script reliably. Also builds `d8` and regenerates a
  repo-root `compile_commands.json` (editor tooling) each run.
  **Unverified end to end** - written directly from `build.yml`'s real
  steps, but there's no Windows machine anywhere this was tested against.
  ```
  node tools/build-windows-local.js x64
  node tools/build-windows-local.js x64 15.3
  ```
  Re-running is safe (resets, not re-clones, depot_tools; force-checks-out
  `v8` back to a clean `branch-heads/<version>` before reapplying patches).

## miscellanous

- **[`check-patches.js`](check-patches.js)** - before bumping
  `V8_VERSION`, checks whether every patch in [`../patches/`](../patches/)
  still applies cleanly against a target version's real source (fetched
  straight from `raw.githubusercontent.com/v8/v8`, no full V8 clone) -
  catches a patch going obsolete/context-drifted in seconds instead of a
  full CI round-trip.
  ```
  node tools/check-patches.js 15.3
  ```
