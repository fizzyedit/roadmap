# Zig 0.17

Status: proposed. ZLS 0.17 is no longer a gate (the user, 2026-10-14): we move when it is the
efficient thing to do, and the first reason is dvui (the dvui upstreaming plan (kept private)).

Zig 0.17.0 was released at the start of October 2026, and dvui merged its `0.17.x` branch
(david-vanderson/dvui#1012), so every dependency fizzy draws with can now move. This plan is the
order we move in, what we check before we start, and which change in the release notes becomes
which migration in fizzy. The goal is a proper migration: where the notes name a better way to do
something, we adopt it rather than writing a shim that keeps the 0.16 way compiling.

Source: the [0.17.0 release notes](https://ziglang.org/download/0.17.0/release-notes.html), plus
the 0.17 standard library itself where the notes stop short (custom build steps, below). Counts
below are from a grep of fizzy at `36cd31c` (`main`, 2026-10-13) and will drift.

## What 0.17 does not give us

**Fast incremental compilation on macOS arm64.** The notes say `-fincremental --watch` now works
for "most projects targeting x86_64-linux", on the back of the new ELF linker. A new Mach-O
linker and a self-hosted aarch64 backend are future work; the aarch64 backend section is one line,
"blocked on Linker enhancements". The roadmap promises aarch64 as the Debug default without a
date. On the Mac we stay on LLVM and full rebuilds. Linux x86_64 (CI, the Ubuntu VM) is the only
place incremental could help, and that is not a reason to move.

**Loop vectorization.** Still disabled in 0.17's LLVM 22, as it was in 0.16; LLVM 23 in 0.18
brings it back. Not a regression, but nor is it the win a toolchain bump usually brings.

## The gate

We start step 1 when the checks below hold, and say in the step 1 PR how each was checked.

**Why now.** dvui's `main` has required 0.17 since 2026-10-13 (david-vanderson/dvui#1012). Every
upstream PR fizzy makes to dvui is now a 0.17 branch, and fizzy can't pin dvui's `main` until fizzy
is on 0.17 too (the dvui upstreaming plan (kept private)). Rebasing `fizzy-dev` onto a 0.16 base in the meantime
would be work we throw away. So the move is timed by dvui, not by the editor tooling.

**ZLS is a known cost, not a gate** (the user, 2026-10-14). As of 2026-10-13 the latest ZLS is
0.16.0. The 0.17 work is draft zigtools/zls#3270, with open regressions: `@import` resolution in
build files (ziglang/zig#36387), and in modules reached only through `std.Build.Dependency.module`
(ziglang/zig#36388). The second is how every plugin reaches `fizzy_sdk` and `dvui`, so completions
and go-to-definition across that boundary will be degraded until a fixed ZLS ships. Record in the
step 1 PR:

1. **ZLS, as it stands.** Which ZLS build was tried, and whether completions and go-to-definition
   across `fizzy_sdk` work from a bundled plugin, from pixi, and in fizzy's own `build.zig`. When
   a ZLS 0.17 release fixes ziglang/zig#36388, move to it; until then, note what is broken.
2. **The `zig` plugin.** `fizzyedit/zig` drives ZLS (`src/Lsp.zig`). Run it on a 0.17 project in
   fizzy. If the plugin is broken, that does not block the move, but it gets a repin PR of its own
   with the SDK release (step 3).
3. **Nothing lost.** On a branch build, before step 2 merges:
   - Every target still builds and runs: macOS, Windows, Linux, the web (wasm32 + emscripten), and
     the six cross-compiled plugin targets in `fizzyedit/plugin-build-action`. 0.17's std needs
     Darwin 15+, Windows 10+, Linux 5.10+; compare with what fizzy promises
     (`assets/macos/info.plist` `LSMinimumSystemVersion`) and raise ours or say why not.
   - Frame time and startup measured the same way as on 0.16 (`core.profiler`, the `sample`
     recipe), Debug and ReleaseFast. Performance is fizzy's edge; a toolchain bump that costs
     frames waits.
   - Build times: cold and warm `zig build`, a plugin build, `zig build test-integration`. The
     maker process is new and built `-Osafe` once; configure caching (below) should make warm
     builds faster, so measure it rather than assume it.
   - The known regressions in the notes don't touch us: Run-step response files
     (ziglang/zig#36444), `std.debug.simple_panic` (#36986), weak libc symbols (#36986).
   - The Velopack updater still updates (it is the one `@cImport` user).
   - Debug-build leak reports still print (the allocator changes, below).

## Order

Each step is its own PR in its own repo, saying `Part of fizzyedit/plans#N`.

0. **Forward-compatible changes, on 0.16.** Anything that compiles on both versions lands first,
   in small PRs, so the switch itself is smaller: `@import("builtin").os` → `.target.os` (and
   `cpu`, `abi`, `object_format`), and the `@hasDecl` audit. This step may start before the
   gate.
1. **Forks and vendored libraries.** Each moves to 0.17 in its own repo and is tagged, so fizzy
   can pin it by URL. Upstream first: where a third-party library has no 0.17 release, open a PR
   there, and fork only if it stalls (record the fork in `fizzy/docs/DEPENDENCIES.md`).
   - `foxnne/dvui-dev`: rebase the fizzy-dev patches onto upstream `main` (now 0.17), dropping
     those upstream has merged (the dvui upstreaming plan (kept private)). dvui stays pinned only in
     `sdk/build.zig.zon`.
   - `fizzyedit/sdl_zig` and `backend/patches/sdl`: upstream dvui bumped its SDL dependency for
     0.17; take what it needs.
   - `foxnne/zig-objc`, `fizzyedit/md4zig`, `fizzyedit/icons`, `foxnne/assetpack`.
   - Vendored: `libs/dvui-singleton-app`.
   - Third-party, check for 0.17 releases: `natecraddock/zf`, `neurocyte/nightwatch`,
     `marlersoft/zigwin32`, `graphl-tech/velopack-zig`.
   - New: the official `translate-c` package (codeberg ziglang/translate-c), see below.
2. **fizzy and its bundled plugins.** One switch PR: `minimum_zig_version` flips every file at
   once, so it cannot be stacked. It will pass the 800-line guideline; keep it reviewable by
   landing the mechanical part (`zig fmt`'s automatic upgrades, pure renames) as its own commit
   and saying so, with the hand migrations below in commits of their own. This moves the SDK
   fingerprint (dvui's shape, and `@typeInfo` itself changes, below), so step 2 is followed at
   once by an SDK release PR.
   It also puts fizzy's `backend/` package on 0.17, which is what dvui's `-Dbackend=fizzy` will
   fetch (the dvui upstreaming plan (kept private)): that dvui PR follows this step.
3. **External plugins, prepared in parallel with step 2, landed with the release.** A 0.17 fizzy
   refuses every plugin built against the old fingerprint, and fizzyed.it/app is built from
   `main`, so the plugins can't trail by days. Port each on a branch against step 2's `sdk/` by
   path, then repin to the released SDK.
   - `fizzyedit/example-app` and `fizzyedit/example-plugin` first: CI builds both against every
     fizzy PR (`scripts/check-examples.sh`), so step 2 cannot go green without their 0.17 PRs.
   - `fizzyedit/plugin-build-action`: the Zig version it installs; the repin automation.
   - Store plugins: pixi, zig, ghostty, drive, atlas. Then agent, chat, and the rest of
     `fizzyedit`.

## The spike (2026-10-15)

A throwaway branch got fizzy and its four bundled plugins building and running on 0.17, on macOS
arm64, against `fizzy-dev` rebased onto dvui's `main` (`41f6703` plus 8 patches). It touched 122
files (+626/−624). The window was up 1.1 s after `main`. `zig build test` passed 546/547 (1
skipped), and `zig build test-integration` passed 522/522. The only failures were leak reports
caused by the zon stand-in below.

Not exercised: Windows, Linux, the web build, a standalone plugin build (`DevInstall`),
packaging, cross builds, store plugins, ZLS. The display was asleep, so there is no screenshot.

**Step 0.** fizzyedit/fizzy#435 lands the `builtin.target` rename (235 sites). The `@hasDecl`
audit needs no change: every target fizzy, a store plugin or an example looks for is already
`pub`.

**What `zig fmt` did on its own:** about 280 `@intFromEnum`/`@enumFromInt` → `@backingInt`/
`@fromBackingInt`. Still compiling as deprecated aliases, so they can go in the rewrite script
later: `std.builtin` (141), `allocPrint` (123), `std.mem.indexOf*` → `find*`, and the
`OptimizeMode` decl names (but not the enum literals, below).

**Stand-ins: the step 2 work, by size.** 57 `SPIKE` markers in 31 files, each a placeholder for a
real migration:

1. **The build system: largest.** 0.17 runs `build.zig` in a configurer and the graph in a
   separate maker.
   - Three custom `makeFn` steps. `DevInstall`, which every plugin installs through, compiled
     only because no app build reaches it; a plugin build will hit it. "Set aside a running
     executable" is Windows only. The `update` step is stale (it pulls dvui-dev `main` and a
     zigwin32 0.16 branch) and should be deleted, not ported.
   - Install paths are not known at configure time (`b.getInstallPath`, 3 sites: Velopack
     packaging, set-aside).
   - There is no global `--libc` (`b.libc_file`). Cross-building to Windows with MSVC takes it as
     a `-D` option.
   - The MSVC translate-c shim (`build/msvc.zig`) walks steps by `Step.id` and reads generated
     files' steps, and both are gone. It was stubbed out.
   - Removed: `Run.addPathDir` (Velopack's squashfs), `LazyPath.getPath3` (7 configure-time path
     reads, now `b.root.joinString`), `b.build_root` (now `b.root`, a `Cache.Path`; the spike
     assumed an empty `sub_path`) and `b.pathFromRoot`.
   - A package may not add the same public module twice: `b.addModule("App")` per target became
     `createModule`.
   - `Subsystem` tags are lowercase.
2. **`std.zon.parse`** (about 30 parse sites, 20 `free`s). Results and diagnostics now come from a
   caller's arena, and there is no `free`. The stand-in parsed into the old allocator and dropped
   the frees, hence the leaks. The real work is the arena-per-owner design this plan already
   describes. `Diagnostics` is now `{ errors }`, formatted with `.fmt(path)`. Zoir lookups take
   `*const Zoir`. `Ast.parse` takes `.{ .mode = .zon }`.
3. **`@typeInfo`**: about 30 field loops became struct-of-arrays (`field_names`, `field_types`,
   `field_attrs`; enums `field_values`; fns `param_types`, `attrs.@"callconv"`; pointers
   `attrs.@"const"`; error sets `error_names`). Most were scripted.
   - `sdk/src/fingerprint.zig` was translated loop for loop to hash the same facts in the same
     order. That needs proof, not trust: a test hashing a fixed set of types under 0.16 and 0.17.
   - `sdk/src/settings.zig` and `sdk/src/Command.zig` index field records (`struct_fields[i]`).
     The spike gave them a compat helper. The real change reworks the schema builders.
4. **`@cImport` is gone, even in a branch never taken.** Velopack (`app/update/*`, 2 sites) is
   stubbed and needs a translate-c module from `build/velopack.zig`.
5. **dvui's own changes since 2026-10-02.** Per-pointer mouse capture (dvui#1014):
   `Window.capture` became `captures[slot]`, at 2 sites (the chooser and the workbench's drag).
   More may surface at run time.

**Forks (step 1), all small.**

| Fork | Change |
|---|---|
| `foxnne/dvui-dev` | One-line conflict (`@backingInt` beside precision); stack passes dvui's tests on 0.17 |
| `foxnne/zig-objc` | `@typeInfo` (`BlockContext` rebuilt), `**`: about 60 lines. Upstream mitchellh/zig-objc has no 0.17 yet |
| `fizzyedit/md4zig` | `@cImport` → a translate-c module (`include/md4zig_c.h`) |
| `foxnne/assetpack` | One line: `Ast.parse(…, .{ .mode = .zig })` |
| `fizzyedit/sdl_zig` | 5 config values 0.17 rejects as unused (`HAVE_FREE`, `CALLOC`, `REALLOC`, `MPROTECT`, `SYSCTL`), on SDL 3.4.16. Upstream sdl_zig is on 3.4.18 and restructured (prefetch, system paths): moving to it means porting our two patches, and checking SDL's patches against 3.4.18 |
| zf, nightwatch | Repin to their 0.17 commits (zf `main`, nightwatch `master` `023e79f`) |

**What this plan's migration list missed:**
- The array-repeat operator `**`. The list says no uses; there are 13, mostly tests. A
  multi-character string repeat needs a comptime loop.
- `std.meta.Int` → `@Int`.
- `Allocator.dupeZ` → `dupeSentinel(T, x, 0)` (61 sites). A local helper named `dupeZ` must not
  be caught by the rewrite.
- `fmt.bufPrintZ` → `bufPrintSentinel(…, 0)` (9).
- `EnumSet.initEmpty()` → `.empty`.
- `ascii.indexOfIgnoreCase` → `findIgnoreCase`.
- `@bitCast` to an `extern struct`: 4 sites hit (an image fill, the zip reader's headers), fixed
  with `bytesToValue` or field-wise init. The other ~50 `@bitCast`s compiled, so only a review
  finds the silent array/vector changes.
- Optimize modes as enum literals (`== .Debug`, `.ReleaseFast =>`), about 20. The deprecated
  decls don't make the literals compile.

**Suggested cut of step 2 into PRs on `zig/0.17`:**
- (a) build system;
- (b) zon arenas;
- (c) `@typeInfo`, with the fingerprint-equality test first;
- (d) the small std renames, scripted;
- (e) Velopack translate-c;
- (f) dvui API drift.

The rewrite script carries `zig fmt`'s upgrades and the deprecated-alias renames. Then Windows,
Linux and the web, and a standalone plugin build to reach `DevInstall`.

## Migrations, from the release notes

### Build system

- **Custom build steps are gone.** Not in the notes, but in the source: 0.17 runs `build.zig` in a
  configurer process and the graph in a separate, prebuilt maker, and `Step.Tag` is now a closed
  list (`compile`, `run`, `install_*`, `write_file`, …) with no `makeFn`. fizzy has three:
  `sdk/plugin_sdk.zig` `DevInstall` (every plugin's install into the plugins directory, with the
  temp-file + rename that keeps a running fizzy safe; third-party plugins use it too),
  `build/app.zig` "set aside a running executable", and `build/common.zig` `updateStep`. Each
  becomes a small helper executable run by a Run step, the way `plugin_sdk_meta.zig` and
  `plugin_sdk_check.zig` already work, or a plain install step where that is all it was. This is
  the largest design item here; prototype `DevInstall` early.
- **Configure-time purity.** The configure result is now cached and skipped when nothing it read
  changed; anything read untracked "poisons" the cache and forces a reconfigure every time. Our
  configure phase reads `VERSION` (`build/app.zig`), each plugin's `plugin.zig.zon`
  (`sdk/plugin_sdk.zig`), `web/index.html` (`build/web.zig`), the MSVC `libc.ini`
  (`build/msvc.zig`). Declare each with `b.dependOnFileContents`. Find the rest with
  `--cache-poison=disallowed`, which panics on any poisoning. Environment variables read at
  configure time (`FIZZY_STATIC_<NAME>`, `FIZZY_PLUGIN_USE_LLVM`, `SDKROOT`, the macOS signing
  variables): find out whether 0.17 keys the configure cache on them; if it does not, a changed
  variable silently reuses the old graph, and they should become `-D` options.
- **API renames.** `b.build_root` (Directory) → `b.root` (Path): 15 sites. `b.args` +
  `addArgs` → `run.addPassthruArgs()`: 11. Run-step argument helpers → their `*Arg2` forms: 9.
  `LazyPath.getDisplayName` → `{f}`: 1. `findProgram` and `addOptionPath`: no uses.
- **Lazy dependencies.** `b.dependencyLazy` returns `error.LazyDependencyNeeded`, so the 16
  `lazyDependency` sites can `try` instead of carrying optionals. Velopack stays lazy, and
  `ls zig-pkg | grep -i velo` must still be empty after a clean plugin build.
- **No build-runner override.** Nothing in fizzy uses `--build-runner`; tools that need the graph
  use the Build Server Protocol (`--listen=-`).
- **`addWin32ResourceFile` is deprecated** (`build/exe.zig`) and moves to an external package in
  0.18. Keep it for now; the seam is that one call.

### Language

- **`@cImport` is removed and `addTranslateC` deprecated**, in favour of the `translate-c`
  package. `@cImport`: `app/update/update_install.zig`, `app/update/auto_update.zig` (Velopack.h);
  `addTranslateC`: `backend/build.zig` (SDL). The translated header becomes a module from the
  package's `Translator`, imported by name.
- **`@hasDecl` is true only for `pub` declarations**, now in every file (27 sites in 17 files).
  This one fails silently: a non-`pub` decl in the same file used to count and now reads as
  absent, so an optional hook is just skipped. Audit every site (`sdk/src/Host.zig`
  `service_version`, `sdk/src/settings.zig` `is_setting_cell`, `app/App.zig` `static_<id>`, the
  backend's `appFrameOpen`/`callbackFrames`/`setWindowVSync`, …) and make the target `pub`. Can
  land in step 0.
- **`@typeInfo` returns structs and unions struct-of-arrays style** (`field_names`,
  `field_types`, …): about 45 `.fields` uses across 20 files, the weight in
  `sdk/src/fingerprint.zig` and `sdk/src/settings.zig`. The fingerprint must keep hashing the same
  facts in the same order. `std.meta.fieldNames`/`fieldInfo`/`fieldTypes` are deprecated in
  favour of this (no uses).
- **`@bitCast` is redefined for arrays and vectors**, which can break without a compile error,
  and no longer accepts `extern struct`/`union` (use `@ptrCast` or an `extern union`). Audit the
  55 sites in 19 files; the renderer and image code are the likely ones.
- **`@intFromEnum`/`@enumFromInt` → `@backingInt`/`@fromBackingInt`**: 300 sites; `zig fmt` does
  it. **`std.builtin` → `std.lang`**: 142. **`OptimizeMode` → `Optimize`, tags lowercased**
  (`.ReleaseFast` → `.fast`): 43 + 30, and `==` against the old names stops compiling. Check
  whether the `-Doptimize=ReleaseFast` CLI spelling in CI and `plugin-build-action` changes too.
- Removed and unused here: array `**`, `void{}`, `errdefer |err|`, `i0`, `internal`/`link_once`
  linkage.

### Standard library

- **`std.zon.parse` is reworked** (47 sites in 23 files): struct arguments, and results and
  diagnostics allocated from an arena. Give each parse the arena of whatever owns the result, and
  drop the `std.zon.parse.free` calls. The new `updateFrom*` variants overwrite an existing value
  with only the fields a source names; the notes' own example is an editor's global and
  per-project config. Evaluate them for settings reconciliation (`Access.applyBlob`) and for
  plugin defaults under user values, rather than porting our current merge as-is.
- **`DebugAllocator` → `SafeAllocator`** (`src/Entry.zig`): thread-safe, never reuses memory,
  detects cross-allocator frees. Use it in Debug, and check startup and frame time in Debug with
  it, since no reuse means more mapping.
- **`std.fmt.allocPrint(a, …)` → `a.print(…)`**: 115 sites.
- **`@import("builtin").os`/`cpu`/`abi`/`object_format`** are deprecated, removed in 0.18 → the
  `target` fields: 243 sites in 47 files. Step 0.
- `ArrayList.getLast*`, the bit-set renames, `StackFallbackAllocator`, `containsAtLeastScalar`'s
  swapped arguments, `Uri.getHost`: no uses.

## Graphl

Graphl is working on moving to Zig 0.17 now (status from the maintainer, 2026-10-20). It has its
own `foxnne/dvui-dev` pin from the dvui 0.4.0 era. The coupling that matters is dvui, not Zig: a joint move
is both projects agreeing on one `dvui-dev` tag built for 0.17 and pinning it (here that is the
single pin in `sdk/build.zig.zon`). Bring graphl in when the dvui rebase of
the dvui upstreaming plan (kept private) picks that tag. Its `dvui_unreal` and `blender_dvui` backends move with
it. This is for graphl to do; nothing here waits on it.

## SDK and plugin authors

- The fingerprint moves, and a 0.17 host refuses plugins built against the old one (the Installed
  pane says why, fizzyedit/fizzy#368) until the repin automation delivers rebuilt ones.
- Plugin authors see: `minimum_zig_version = "0.17.0"`, a new `fizzy-sdk-v*` tarball, and the
  build-side changes they share with us (their `build.zig` calls `fizzy.plugin.create` /
  `.install`, so the `DevInstall` rework reaches them only through the SDK). The SDK release PR
  says so in plain words, and `fizzy/docs/PLUGINS.md` gets the new minimum.
- A plugin still on 0.16 cannot build against the 0.17 SDK; there is no overlap window. That is
  why step 3 lands with the release.
