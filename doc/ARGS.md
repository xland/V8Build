# GN build args — what `args.win.x64.gn` sets, and why

A field guide to every GN arg in this repo's single build file
(`args.win.x64.gn` — Windows/x64 only). Very brief by design — for the
flags that have a real, deeper investigation behind them (pointer
compression, cppgc's caged heap, `symbol_level`/`strip_debug_info`), this
doc gives the short version and links out to
[`V8-BUILD-OPTS.md`](V8-BUILD-OPTS.md) in this same directory rather than
duplicating it.

Two families of flag here: plain Chromium/GN build-system args (no
prefix — `is_debug`, `use_rtti`, etc., defined in `build/` — a
`gclient`-fetched dependency, not part of this repo's own source) and
`v8_*`/`cppgc_*` args (V8's own, defined in V8's `BUILD.gn`/`gni/v8.gni`).
Descriptions are anchored to the V8 release currently pinned by this
repo (see `.github/workflows/build.yml`'s `V8_VERSION`) — a flag's
default or even its existence can drift across V8 versions, so treat
line-number citations here as approximate for any other version.

## 1. Every flag `args.win.x64.gn` sets

| Flag | Value | What it does | Alternative |
|---|---|---|---|
| `is_clang` | `true` | Compile with clang (clang-cl on Windows), not MSVC. | `false` — MSVC is a documented unsupported combo for V8+libc++ (see `use_custom_libcxx` below). |
| `is_debug` | `false` | Release vs. debug build (optimizations on, assertions off). | `true` — much slower/larger, not what a shipped `v8_monolith` wants. |
| `is_component_build` | `false` | One static/monolithic output vs. every target as its own shared library. | `true` — only useful for Chromium-style incremental-link dev iteration; irrelevant once `v8_monolithic=true` is the actual goal. |
| `symbol_level` | `0` | How many debug symbols to embed (`0` none, `1` backtrace-only, `2` full). | `1`/`2`. **Real finding:** *load-bearing* on Windows — the real default here is `2` (full symbols) regardless of release/debug. Full derivation: [V8-BUILD-OPTS.md](V8-BUILD-OPTS.md). |
| `strip_debug_info` | `true` | Per its own GN comment: **Android-only** — strips `lib.unstripped/*.so` after link while keeping backtraces symbolizable via `symbol_level>0`. | `false`. **Real finding:** does nothing on Windows — no consumer of the flag found outside Android APK packaging. Harmless, likely inherited unexamined from an early build-feasibility reference. Same writeup as above. |
| `treat_warnings_as_errors` | `false` | Abort compilation on any compiler warning. | `true` (Chromium's real default) — risky here since the build compiles against whatever toolchain the CI runner image has, not one pinned/vendored copy. |
| `use_sysroot` | `false` | Use Chromium's bundled/pinned sysroot instead of the host's own libc/headers. | `true` (real default) — would need Chromium's sysroot-fetch machinery; instead the build compiles against the runner's real system headers, which is *why* [`../patches/`](../patches/) exists (real source bugs some V8 branch-heads have that Google's own CI, using the pinned sysroot, never hits). |
| `dcheck_always_on` | `false` | Force `DCHECK()`-style asserts on even in a release build. | `true` — extra runtime safety checks, slower. Already GN's own default; this line is documentation only. |
| `enable_rust` | `false` | Enable the Chromium build's Rust toolchain integration. | `true`, needs a Rust toolchain present; not applicable to V8 itself. Already GN's own default here too. |
| `clang_use_chrome_plugins` | `false` | Chromium's custom clang static-analysis plugins (coding-style/lifetime checks). | `true` — only usable with Chromium's own clang build, not a standalone toolchain. |
| `v8_deprecation_warnings` | `false` | Compiler warnings on `V8_DEPRECATED`-annotated embedder APIs. | `true` (real default) — useful when actively chasing deprecations ahead of a version bump, just noise for a release build. |
| `v8_imminent_deprecation_warnings` | `false` | Same, for `V8_DEPRECATE_SOON`. | `true` (real default). |
| `v8_enable_i18n_support` | `false` | ECMAScript `Intl.*` API. | `true` (real default) — pulls in a real ICU dependency; bigger binary. |
| `v8_enable_pointer_compression` | `true` | Compress the JS heap's tagged pointers to 32-bit offsets into a 4GB cage. | `false` — real, direct memory-footprint win when on; flips several derived defaults (`v8_enable_external_code_space`, `v8_enable_static_roots`, `v8_enable_short_builtin_calls`). Full cascade: [V8-BUILD-OPTS.md](V8-BUILD-OPTS.md). |
| `v8_enable_sandbox` | `false` | V8's heap "sandbox" — confines all heap-object pointers to a smaller address range so a JS/Wasm memory-corruption bug can't be turned into an arbitrary read/write of host process memory. | `true` — real security hardening, aimed at the untrusted-JS-in-browser threat model. Worth turning on if your embedding runs untrusted script; off here since this build targets a trusted-script embedding use case, where the extra indirection/overhead buys nothing. |
| `v8_enable_temporal_support` | `false` | The `Temporal` date/time API. | `true` — off by default anywhere not building for Node; pulls in `temporal_rs`. |
| `v8_enable_test_features` | `false` | Various V8-internal testing-only features/flags. | `true` — only relevant to V8's own test suite. |
| `v8_enable_webassembly` | `false` | The WebAssembly engine and the `WebAssembly.*` API. | `true` (real default). Genuinely removed at compile time (`V8_ENABLE_WEBASSEMBLY`), not just linker-pruned — measured at ~9.3MB of a 103.6MB archive. No JS execution cost either way; the only tradeoff is losing `WebAssembly.*`. Full writeup: [V8-BUILD-OPTS.md](V8-BUILD-OPTS.md). |
| `v8_monolithic` | `true` | Build one big static `v8_monolith.lib` for embedders, instead of V8's normal many-small-libraries layout. | `false` (real default) — this is the entire reason this repo exists: one linkable archive for embedders, rather than V8's normal Chromium-oriented many-small-libraries layout. |
| `v8_use_external_startup_data` | `false` | Keep builtins/startup snapshot embedded in the binary vs. a separate blob file loaded at runtime. | `true` — an extra file to ship/load, no benefit for a static-lib embedder. Confirmed correct already: [V8-BUILD-OPTS.md](V8-BUILD-OPTS.md). |
| `v8_enable_backtrace` | `false` | V8's own compiled-in crash-backtrace support (distinct from `symbol_level`/`strip_debug_info` above — this is V8-internal, not toolchain-level). | `true` — extra code/size; worth turning on if your embedder wants V8-native crash backtraces rather than relying on the OS/toolchain's own symbolication. |
| `v8_enable_gdbjit` | `false` | Emit a GDB JIT interface so `gdb`/`perf` can symbolicate V8's JIT-generated code. | `true` — already GN's own default; this line is documentation only. |
| `cppgc_enable_caged_heap` | `false` | cppgc's (C++-GC'd heap, e.g. for Blink DOM wrappers) own separate 16GB `PROT_NONE` virtual reservation. | `true` (real default). Off here because this build has no embedder-side `CppHeap` usage today — confirmed pure unused overhead in that case. **If your embedder constructs a `CppHeap`, re-enable this.** Full reasoning: [V8-BUILD-OPTS.md](V8-BUILD-OPTS.md). |
| `cppgc_enable_pointer_compression` | `false` | Pointer compression for the above (moot with it disabled). | `true` — only meaningful if the caged heap above is re-enabled. |
| `v8_enable_partition_alloc` | `false` | Use Chromium's PartitionAlloc as `d8`'s allocator, to mimic in-browser allocator behavior for benchmarking/testing. | `true` (real default once pointer compression is on) — pinned off because it conflicts with the Windows SDK's `operator new`/`delete` visibility declarations (real CI failure class seen on this build). |

## 2. Platform/arch identity

`target_os="win"`, `target_cpu="x64"`, and `v8_target_cpu="x64"`
(mirroring `target_cpu`). `v8_target_cpu` exists as its *own* flag
separate from `target_cpu` specifically to support cross-compiling V8's
JIT for an architecture different from the host's own — not a mode this
repo uses.

## 3. Two flags Windows is forced into

| Flag | Value | Why it differs from a Linux/mac build |
|---|---|---|
| `use_custom_libcxx` | `true` | Use Chromium's in-tree/vendored libc++ instead of the system C++ standard library. Forced `true` here because MSVC's own STL is a documented unsupported/broken combo with clang-cl for building V8 (confirmed: a real `v8-users` thread hit the identical failure, and a V8 team reply says MSVC support "will soon be deprecated altogether"). **Consequence for consumers:** an embedder must compile against the same vendored libc++ headers, which is why CI publishes `libcxx-headers-win-x64.zip` alongside `v8_monolith.lib`. |
| `use_rtti` | `true` | Emit C++ RTTI (`dynamic_cast`/`typeid`) info. Off by default to save size; forced on as a paired requirement of `use_custom_libcxx=true`. |

Two more are set explicitly but are already GN's own defaults on this
platform — kept only as documentation: `v8_enable_builtins_optimization`
(`true`; per [V8-BUILD-OPTS.md](V8-BUILD-OPTS.md) very likely inert
either way, since it needs a profile file this workflow never fetches)
and `v8_enable_snapshot_compression` (`false`, which is the real default
everywhere anyway).

## 4. Why there is no debug variant

This repo ships one args file. It briefly had a debug counterpart; that
was removed, and the two reasons are worth recording.

**The obvious one: "debug JS" needs no debug build.** V8's Inspector —
the CDP/DevTools-protocol debugger, i.e. breakpoints, stepping, scopes —
is not behind any GN flag. There is no `v8_enable_inspector`;
`src/inspector` sits in BUILD.gn's source list unconditionally, so it is
compiled into this release lib too. Debugging JavaScript works either way.

**The decisive one: on Windows, `is_debug` changes the CRT, and a static
library drags that choice into its consumer.** build/config/win/BUILD.gn's
`static_crt` config:

```gn
config("static_crt") {
  if (is_debug) {
    # This pulls in the static debug CRT and defines _DEBUG
    cflags = [ "/MTd" ]
    if (use_custom_libcxx) {
      ldflags = [ "/DEFAULTLIB:libcpmtd.lib" ]
    }
  } else {
    cflags = [ "/MT" ]
```

`/DEFAULTLIB` is recorded inside the object files, so a debug
`v8_monolith.lib` forces every embedder onto `/MTd` too: link it into a
`/MT` program and you get `LNK4098: defaultlib 'libcmtd.lib' conflicts
with use of other libs`, and pushing through anyway with `/NODEFAULTLIB`
leaves two CRTs with two separate heaps — a cross-module
`malloc`/`free` mismatch. And because `/MTd` *implicitly defines `_DEBUG`*,
it changes libc++ as well here (`use_custom_libcxx=true`), not just V8.
Chromium carries a dedicated `release_crt` config for exactly this class
of problem, commented "This also avoids *implicitly* defining `_DEBUG`".

What `is_debug` would otherwise buy, and how to get each piece without
switching CRTs:

| Want | How, without `is_debug=true` |
|---|---|
| Symbols / readable stack traces | `symbol_level=2` — no runtime cost, no CRT change. Note the debug info lands in a `.pdb` beside `obj/v8_monolith.lib`, which the workflow does not currently publish. |
| V8's debug features (`--print-code` and friends) and its self-checks | `v8_enable_debugging_features=true` **plus** `v8_enable_verification_features=true` — gni/v8.gni asserts the pair. gni/v8.gni's own comment calls this "running V8 in a pseudo debug mode within a release Chrome", i.e. the supported way to get debug capability without a debug build. It does re-define `DEBUG` and turn every `DCHECK` on (`src/base/logging.h` wraps them in `#ifdef DEBUG`), so expect a real speed cost. |
| Not slowing down to O0 | `is_debug` drops V8 to `:no_optimize`; `v8_optimized_debug=true` is what puts `optimize_speed` (O3) back, and is already its real default. |

**The two ETW flags cost nothing at steady state**, which is why turning
them off in release is a pure size win rather than a performance
trade-off: `ETWJITInterface::EventHandler`'s first statement is
`if (!isolate->IsETWTracingEnabled()) return;`, and
`src/libplatform/tracing/recorder.h`'s own comment says system
instrumentation needs the build flag *and* the
`--enable-system-instrumentation` runtime flag. Neither emits anything
unless a trace session is actively attached.

**Not available in this configuration**:
`v8_enable_generated_code_validator` (BUILD.gn asserts it requires
`v8_enable_sandbox`, which is off) and `v8_enable_torque_dwarf` (`is_debug
&& is_linux`). `v8_enable_gdbjit` is the GDB JIT interface — posix-only,
no effect on Windows.

## 5. Source patches CI applies that aren't GN args

Not expressible as `args.gn` lines or as a `patches/*.patch` (they touch
files `gclient sync` fetches fresh, not V8's own source) — applied as
inline build steps in `.github/workflows/build.yml`, each with its own
long comment there:

- `build/config/win/BUILD.gn`: `NTDDI_VERSION` downgraded from
  `NTDDI_WIN11_BR` to `NTDDI_WIN10_VB` — the former needs a newer Windows
  SDK than the one installed on the runner, and an undefined
  `NTDDI_VERSION` silently zeroes out every version-gated declaration in
  the SDK headers.
- V8's `BUILD.gn`: `v8_monolith` made to depend on
  `//buildtools/third_party/libc++` explicitly, because Chromium's
  automatic libc++ injection never covers `static_library` targets —
  without it, final link fails with dozens of real `LNK2019`/`LNK2001`s.
- `buildtools/third_party/libc++/BUILD.gn`: that target's `visibility`
  list extended to allow `//:v8_monolith` to depend on it.

## External links

- [`v8.dev/docs/build-gn`](https://v8.dev/docs/build-gn) — V8's own GN build doc.
- [`v8.dev/docs/build`](https://v8.dev/docs/build) — general V8 build overview.
- [`GN reference`](https://gn.googlesource.com/gn/+/main/docs/reference.md) — the GN language/tool itself, not V8-specific.
- [`v8.dev/blog/sandbox`](https://v8.dev/blog/sandbox) — the V8 heap sandbox (`v8_enable_sandbox`), why it exists and its threat model.
- [`chromium/chromium` GitHub mirror: `build/config/compiler/compiler.gni`](https://github.com/chromium/chromium/blob/main/build/config/compiler/compiler.gni) — real source for `symbol_level`, `strip_debug_info`, `treat_warnings_as_errors`, `use_thin_lto`.
- [`chromium/chromium` GitHub mirror: `build/config/c++/c++.gni`](https://github.com/chromium/chromium/blob/main/build/config/c%2B%2B/c%2B%2B.gni) — `use_custom_libcxx`.
- [`v8/v8` GitHub: top-level `BUILD.gn`](https://github.com/v8/v8/blob/main/BUILD.gn) — most `v8_*` args not in `gni/v8.gni` (`v8_enable_sandbox`, `v8_monolithic`, deprecation-warning flags, `v8_enable_partition_alloc`, `v8_enable_builtins_optimization`, `v8_enable_snapshot_compression`, `v8_enable_test_features`, `cppgc_enable_caged_heap`).
- [`v8/v8` GitHub: `gni/v8.gni`](https://github.com/v8/v8/blob/main/gni/v8.gni) — `v8_enable_i18n_support`, `v8_enable_temporal_support`, `v8_use_external_startup_data`, `v8_enable_backtrace`, `v8_advanced_bigint_algorithms`, `v8_enable_pointer_compression`, `v8_target_cpu`.

See also [`V8-BUILD-OPTS.md`](V8-BUILD-OPTS.md) in this directory for the
deeper performance/startup-memory tradeoffs behind a subset of the flags
above.
